# System Architecture

Status: written (Session 4). Owner: engineering. Decisions: D-020, D-026, D-027, D-048 to D-051. Rule IDs: `ARC-`. Companion docs: `tech-stack.md` (what), `api-spec.md` (interfaces), `data-model.md` (storage), `adaptive-engine-spec.md` (decisions), `ai-orchestration.md` (language), `security-and-privacy.md`.

## 1. Shape

Biscuit is a single Next.js application deployed on Vercel, backed by Supabase (Postgres, Auth, Storage), with Claude models reached through the Vercel AI Gateway and Stripe for subscriptions. The adaptive engine is a pure TypeScript module inside the app with no framework or I/O imports.

```
┌──────────────── Browser / installed PWA ────────────────┐
│ Next.js App Router UI (React, Tailwind, shadcn)         │
│ Service worker: app shell + today's agenda offline,     │
│ feedback queue, web push                                │
└───────────────┬───────────────────────────┬─────────────┘
                │ server actions / fetch    │ push (VAPID)
┌───────────────▼───────────────────────────▼─────────────┐
│ Vercel (Node runtime, Fluid Compute)                    │
│  app routes ─ server actions ─ route handlers           │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ src/engine  │  │ src/ai       │  │ src/billing    │  │
│  │ pure, no IO │  │ AI SDK jobs  │  │ Stripe         │  │
│  └─────────────┘  └──────────────┘  └────────────────┘  │
│  src/db (Supabase clients, queries)   src/admin         │
│  crons: agendas, reminders, purge, progress-audit       │
└──────┬───────────────────┬──────────────────┬───────────┘
       │ service role      │ gateway          │ webhooks
┌──────▼──────┐   ┌────────▼────────┐  ┌──────▼──────┐
│ Supabase    │   │ Vercel AI       │  │ Stripe      │
│ Postgres+RLS│   │ Gateway → Claude│  │ Checkout,   │
│ Auth, Storage│  │                 │  │ Portal, Tax │
└─────────────┘   └─────────────────┘  └─────────────┘
```

| ID | Rule |
|---|---|
| ARC-001 | One deployable: the Next.js app. No separate API service in MVP (D-049). |
| ARC-002 | The engine (`src/engine`) MUST NOT import from `next`, `react`, `@supabase/*`, `ai`, or any I/O library. Enforced by an ESLint `no-restricted-imports` rule scoped to that folder and by its own Vitest project. |
| ARC-003 | All database writes to engine-owned tables go through `src/db/engine-writes.ts` using the service role, called only from server code (DM-010). |
| ARC-004 | The default Node.js runtime (Fluid Compute) is used for every route; the Edge runtime is not used. |
| ARC-005 | Long or fan-out work (nightly agenda generation, purge) runs in Vercel Cron route handlers with a cursor so a run can resume within the function timeout; Vercel Queues are a Phase 2 option when fan-out exceeds what a cron loop handles. |

## 2. Source layout (D-049)

```
src/
  app/                      # routes (App Router)
    (marketing)/            # landing, pricing
    (auth)/                 # sign in, sign up, reset
    (app)/                  # authenticated: today, exercise, feedback, progress, goals, settings
    admin/                  # admin-only: frameworks, config, users (read-only)
    api/                    # route handlers: cron, webhooks, streams, admin api
  engine/                   # pure: signals, gates, progression, confidence, priority, selection, exercises, agenda, reasons
    __fixtures__/           # fixture dogs, frameworks, configs, expected outputs
  ai/                       # jobs, prompts/<job>/vN.md, schemas, post-checks, budgets, fallbacks
  db/                       # supabase clients (browser, server, service), typed queries, engine-writes, rebuild
  billing/                  # stripe client, checkout, portal, webhook handlers, entitlement
  frameworks/               # framework schema (zod), validation, seed v0 documents
  config/                   # engine config schema (zod), engine.default.json
  components/               # ui (shadcn), app components
  lib/                      # utils, dates/timezones, units
  pwa/                      # service worker (serwist), push registration, offline queue
supabase/
  migrations/               # SQL migrations
  seed.sql
docs/                       # this folder
```

## 3. Request flows

### 3.1 Onboarding

1. Sign up (Supabase Auth) → `profiles` row via DB trigger.
2. Dog profile and owner questions (server actions, resumable in `profiles.onboarding_state`).
3. Dynamic questionnaire: branching definition is a versioned TypeScript object in `src/app/(app)/onboarding/questionnaire.ts`; AI follow-up job (AI job 1) when a branch is flagged ambiguous; safety screening evaluated server-side (`security-and-privacy.md` §5, `../product/safety-escalation.md`).
4. Complete → server action: write `dog_goals`, lock framework versions into `skill_progress`, run engine (first agenda), pre-render today's items (AI job 2), return the Training Path payload (FR-040) built from the engine's priority trace.

### 3.2 Today

`GET` today's agenda (server component reads `agendas` + `agenda_items` for `local_date = today`); if none exists (user opened before the nightly cron), generate on demand. Projected days are read from offsets 1–2. Exercise screen reads `agenda_items.rendered` or falls back to template substitution.

### 3.3 Feedback → coaching (D-046)

Server action `submitFeedback(itemId, idempotencyKey, payload)` → `submit_feedback` DB function (DM-011) → note interpretation (bounded) → engine run → persist → returns new agenda + `coachingToken`. Client opens the streaming route `GET /api/coaching/{itemId}` which streams the structured coaching response (AI job 3) and writes the final response to `agenda_items.coaching`. Tomorrow's render (AI job 2) is kicked off after the response (`waitUntil`).

### 3.4 Nightly agenda generation

Vercel Cron hourly → `POST /api/cron/agendas` → for each timezone whose local time is 00:00–00:59, iterate dogs (cursor on `dogs.id`), run engine, persist, pre-render. Idempotent: if an agenda for `local_date` already exists with no feedback since, skip.

### 3.5 Reminders (P1)

Cron every 15 minutes → `POST /api/cron/reminders` → users whose reminder time falls in the window and have no completed item today → web push via VAPID. Failed endpoints (410) are removed.

### 3.6 Billing

Checkout: server action creates a Stripe Checkout Session (subscription, Stripe Tax enabled) → redirect. Webhook `POST /api/webhooks/stripe` (signature-verified) upserts `subscriptions`. Customer Portal for cancel/update. Entitlement is a derived value read by the engine (`free` / `premium`) and by UI gating (ENG-096).

### 3.7 Admin

`/admin` is server-gated by `admin_users` + JWT claim. Framework editor: load draft or published body → JSON editor with Zod validation (FS- rules) → preview → diff → save draft / publish (`POST /api/admin/frameworks/{skill}/versions`). Config editor: same pattern for `engine_config_versions`.

## 4. Engine integration

| ID | Rule |
|---|---|
| ARC-010 | `src/engine/index.ts` exports `generateAgenda(state, config, now)` and the step functions. `src/db/engine-state.ts` builds `EngineState` from the database; `src/db/engine-writes.ts` persists `EngineOutput`. No other module calls engine internals. |
| ARC-011 | The engine's fixtures (`__fixtures__`) are the shared language between product docs and code: every `ENG-` and `PM-` rule has at least one fixture case (`../ops/testing-strategy.md`). |
| ARC-012 | `rebuild_progress(dogId)` (DM-020) lives in `src/db/rebuild.ts` and reuses the engine's step functions; it is exposed as an admin action and used in tests. |

## 5. Offline and PWA (D-051)

| ID | Rule |
|---|---|
| ARC-020 | The service worker precaches the app shell and runtime-caches today's agenda payload and rendered exercise text (stale-while-revalidate). |
| ARC-021 | Feedback submissions made offline are queued in IndexedDB with their idempotency key and replayed on reconnect; the UI shows "saved, will sync". Coaching for queued items is fetched after sync. |
| ARC-022 | Install prompt is part of onboarding on supported browsers; on iOS the reminder opt-in explains that install is required for notifications. |

## 6. Environments

Local (Supabase CLI, `.env.local`, config from `config/engine.default.json`), Preview (Vercel preview deployments against a staging Supabase project), Production. Details and promotion rules in `../ops/environments-and-config.md` and `../ops/deployment.md`.

## 7. Non-functional mapping

| NFR | Mechanism |
|---|---|
| NFR-001 offline read | ARC-020/021 |
| NFR-002 today < 2 s | server component, two indexed queries, pre-rendered text |
| NFR-003 regen < 5 s | pure engine (< 1 s), bounded note interpretation, coaching streamed |
| NFR-004 determinism | ARC-002, fixtures |
| NFR-005 AI budget | `ai_calls` counters, AI-050..053 |
| NFR-006 isolation | RLS matrix, service role server-only |
| NFR-010 observability | `ai_calls`, `agendas.trace`, structured logs with correlation id (`../ops/observability.md`) |

---
**Why**: The chat's architect warned against "User Data → LLM Prompt → Training Plan" and placed the AI near the end of a deterministic pipeline. The architecture here is that pipeline realized in the fewest moving parts a solo founder can run: one Next.js app, one database with RLS doing the access control, a pure engine module whose boundary is enforced by lint and tests rather than by a separate service, and crons instead of queues until volume demands otherwise. Every choice in Session 4 (Stripe, single app, shadcn, full PWA) was the option with the most coding-agent coverage and the least operational surface.
