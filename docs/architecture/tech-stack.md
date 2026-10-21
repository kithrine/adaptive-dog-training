# Tech Stack

Status: written (Session 4). Owner: engineering. Decisions: D-026, D-027, D-028, D-048 to D-051. Rule IDs: `TS-`. Versions are "current stable at scaffold time"; pin exact versions in `package.json` and record them in `../ops/environments-and-config.md` when the repo is scaffolded.

## 1. Stack table

| Layer | Choice | Notes |
|---|---|---|
| Language | TypeScript, `strict: true` | No `any` in `src/engine`, `src/ai`, `src/db` |
| Framework | Next.js, App Router | Server components by default; client components only for interaction |
| Runtime | Node.js (current LTS) on Vercel Fluid Compute | No Edge runtime (ARC-004) |
| Hosting | Vercel | `vercel.ts` typed config for crons and headers |
| UI | Tailwind CSS + shadcn/ui (D-050) | Components copied into `src/components/ui`, themed per `../ux/visual-design-reference.md` |
| Icons | Lucide | Ships with shadcn |
| Forms / validation | Zod everywhere (schemas shared by server actions, AI outputs, framework and config documents) | One validation vocabulary |
| Database | Supabase Postgres | Migrations via Supabase CLI; RLS on every user table (`data-model.md` §4) |
| Auth | Supabase Auth: email + password, Google OAuth; `@supabase/ssr` for cookies | Password reset by email (FR-002) |
| Storage | Supabase Storage | Unused in MVP; reserved for Phase 3 photos |
| Data access | Supabase JS client with generated types (`supabase gen types`) | Three clients: browser (anon), server (user session), service (engine writes, webhooks, crons) |
| AI | Vercel AI SDK via AI Gateway, `provider/model` strings from config (`anthropic/claude-sonnet-5-5`, `anthropic/claude-haiku-5-5` by default) | Structured output for every job; streaming for coaching |
| Billing | Stripe: Checkout (subscription), Customer Portal, Stripe Tax, webhooks (D-048) | Products and prices created in Stripe dashboard; ids in env |
| PWA | Serwist service worker; `manifest.webmanifest`; `web-push` (VAPID) for notifications (D-051) | Offline queue in IndexedDB |
| Scheduling | Vercel Cron (hourly agendas, 15-min reminders, daily purge, nightly progress audit) | Vercel Queues reserved for Phase 2 fan-out |
| Email | Supabase Auth emails for auth; transactional (weekly summary, P1) via Resend | One provider; templates in repo |
| Testing | Vitest (unit, engine fixtures, AI post-checks), Playwright (critical flows), Supabase local for DB tests | `../ops/testing-strategy.md` |
| Lint / format | ESLint (next config + engine import restriction), Prettier | Pre-commit via simple-git-hooks |
| Package manager | pnpm | Lockfile committed |
| Observability | Vercel logs + Sentry (errors, traces), `ai_calls` and `agendas.trace` tables | `../ops/observability.md` |
| Analytics | PostHog (product events; free tier) | Events named in `../ux/ux-spec.md`; no PII beyond user id |
| Feature flags | Config-driven for engine; Vercel Flags SDK optional for UI experiments | Not needed for MVP launch |

## 2. Rules

| ID | Rule |
|---|---|
| TS-001 | New dependencies in `src/engine` are forbidden without a decision-log entry; the engine depends on nothing but the TypeScript standard library and Zod for schema types. |
| TS-002 | Supabase service-role credentials exist only in server environment variables and are read only inside `src/db/service-client.ts`. |
| TS-003 | Stripe is the only billing integration; entitlement is read from `subscriptions`, never by calling Stripe at request time. |
| TS-004 | Any LLM call goes through `src/ai/run-job.ts` (AI-010); direct SDK calls elsewhere fail lint. |
| TS-005 | Secrets and environment variables are declared in `src/config/env.ts` with Zod and validated at boot; a missing variable fails the build, not a request. |
| TS-006 | Distances in UI are displayed in feet for the US locale from unit-tagged values (DM-005); unit conversion lives in `src/lib/units.ts`. |
| TS-007 | Timezone handling uses the owner's IANA timezone from `profiles` for every "today" computation; the server never assumes UTC is local. |

## 3. Environment variables (initial)

| Variable | Scope | Purpose |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` | client + server | Supabase |
| `SUPABASE_SERVICE_ROLE_KEY` | server only | engine writes, crons, webhooks |
| `AI_GATEWAY_API_KEY` (or Vercel OIDC) | server only | AI Gateway |
| `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_PREMIUM_MONTHLY`, `STRIPE_PRICE_PREMIUM_ANNUAL` | server only | billing |
| `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` | server (public key exposed) | web push |
| `CRON_SECRET` | server only | authorizes cron routes |
| `RESEND_API_KEY` | server only | transactional email (P1) |
| `SENTRY_DSN`, `NEXT_PUBLIC_POSTHOG_KEY` | mixed | observability, analytics |
| `APP_URL` | server | absolute URLs in emails and Stripe redirects |

Managed with `vercel env` per environment (`../ops/environments-and-config.md`).

## 4. Explicitly not used

- Edge runtime, Edge middleware for auth (use server-side checks in layouts and route handlers).
- A separate backend service, GraphQL, or an ORM beyond the Supabase client (Drizzle/Prisma add a second schema source of truth; migrations are SQL).
- Client-side state libraries beyond React state and server components; today's agenda is server-rendered.
- Native app frameworks (deferred, D-026).

---
**Why**: Every row is the option with the broadest documentation and the smallest operational footprint for one founder plus a coding agent. Supabase and Stripe are chosen for what they remove (auth, RLS, tax, portal), not for what they add. The engine's dependency rule (TS-001) exists because the engine is the asset; keeping it free of framework code is what makes it portable to a future native app or a different host without rewriting the moat.
