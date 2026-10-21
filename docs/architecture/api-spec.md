# API Specification

Status: written (Session 4). Owner: engineering. Rule IDs: `API-`. Transport: Next.js server actions for UI mutations and reads inside the app; route handlers under `/api` for anything with a non-browser caller (cron, webhooks), for streaming, and for admin tooling. The operations below are the contract regardless of transport; payload shapes are Zod schemas in `src/app/.../schemas.ts` and are the source of truth once code exists.

## 1. Conventions

| ID | Rule |
|---|---|
| API-001 | All user operations require a Supabase session; the acting user is taken from the session, never from the payload. Dog-scoped operations verify `dogs.owner_id = auth.uid()` (RLS does this for reads; writes through functions re-check). |
| API-002 | Mutations that can be retried (feedback, skip, start, follow-up) carry a client-generated `idempotencyKey` (UUID). Replays return the original result. |
| API-003 | Errors use one shape: `{ error: { code: string; message: string; details?: unknown } }`. Codes: `unauthorized`, `forbidden`, `not_found`, `invalid`, `conflict`, `rate_limited`, `unavailable`. User-facing messages are plain language; details are for logs. |
| API-004 | Route handlers that are called by cron require `Authorization: Bearer ${CRON_SECRET}`; webhooks verify provider signatures; admin routes require the admin claim. |
| API-005 | Every response that includes an agenda includes its `generation` and `engineConfigVersion` so the client can detect staleness. |
| API-006 | Timestamps are ISO 8601 with offset; "today" is computed server-side from the owner's timezone. |

## 2. Onboarding

| Operation | Transport | Input | Output | Notes |
|---|---|---|---|---|
| `saveOnboardingStep` | action | `{ step, data }` | `{ nextStep }` | persists to `profiles.onboarding_state`, `dogs`, `assessments` (FR-035) |
| `requestFollowupQuestion` | action | `{ branchId, answers }` | `{ question?, options? }` | AI job 1; empty when not needed (FR-033) |
| `completeOnboarding` | action | `{ dogId, goals: [{skillId, rank}] }` | `TrainingPath` | runs screening, locks frameworks, first engine run, returns the FR-040 payload |

`TrainingPath`: `{ dogName, primaryGoals: SkillSummary[], secondaryGoals: SkillSummary[], deferredGoals: SkillSummary[], dailyMinutes, stage, explanation: string, reasons: ReasonObject[] }`.

## 3. Agenda and training

| Operation | Transport | Input | Output | Notes |
|---|---|---|---|---|
| `getToday` | server component / action | `{ dogId }` | `Agenda` | generates on demand if missing (ARC 3.2) |
| `getProjected` | action | `{ dogId }` | `Agenda[]` (offsets 1–2) | labeled projected (FR-051); triggers render on preview (AI-023) |
| `startItem` | action | `{ itemId, idempotencyKey }` | `{ status, startedAt }` | marks immutable for regeneration (ENG-093) |
| `submitFeedback` | action | `{ itemId, idempotencyKey, completed: yes/partial/no, tier1: Tier1Answer, difficulty?, notes?, actualMinutes? }` | `{ agenda: Agenda, coachingToken: string, followup?: { trigger, question?, options? } }` | DM-011 → AI job 4 (bounded) → engine → persist; coaching streamed separately |
| `streamCoaching` | `GET /api/coaching/{itemId}?token=` | — | SSE / streamed structured object `{ acknowledge, observe, next, followupQuestion?, followupOptions? }` | AI job 3; writes final to `agenda_items.coaching`; falls back to template inline |
| `submitFollowup` | action | `{ itemId, idempotencyKey, answer: chip \| text }` | `{ noted: true }` | stored; may produce note signals (AI job 4) for the next run |
| `skipItem` | action | `{ itemId, idempotencyKey, reason }` | `{ agenda }` | FR-055; engine run |
| `getExercise` | server component | `{ itemId }` | `RenderedExercise` | `rendered` or template substitution |

`Tier1Answer` is a discriminated union by measurement type: `{ type: "cue_progression", successes: number, attempts: number }`, `{ type: "interval_progression", band: 1..5 }`, `{ type: "management_count", count: number }`, `{ type: "exposure_checklist", reaction: "relaxed"|"neutral"|"worried"|"fearful" }`.

`Agenda`: `{ localDate, generation, engineConfigVersion, projected, plannedMinutes, routine: RoutineItem[], sessions: AgendaItem[] }`. `AgendaItem`: `{ id, position, skillId, skillName, stage, stageName, estimatedMinutes, status, locked, params, reason: ReasonObject, exercise?: RenderedExercise }`.

## 4. Progress and goals

| Operation | Transport | Input | Output | Notes |
|---|---|---|---|---|
| `getProgress` | server component | `{ dogId }` | `{ skills: [{ skillId, name, groupId?, status, stage, stageName, statusText, milestones }] }` | `statusText` is plain language (FR-100); no confidence number |
| `getHistory` | server component | `{ dogId, cursor? }` | `{ items: HistoryItem[], next? }` | FR-101 |
| `updateGoals` | action | `{ dogId, add?: [{skillId, rank}], remove?: [skillId], swap?: { fromSkillId, toSkillId } }` | `{ agenda, deferred: SkillSummary[] }` | swap honors ENG-074 window; engine run |

## 5. Profile and dog

| Operation | Transport | Input | Output | Notes |
|---|---|---|---|---|
| `getProfile` / `updateProfile` | server component / action | profile fields incl. `minutesPerDay`, `sessionsPerDay`, `equipment`, `timezone`, `notificationPrefs` | profile | capacity changes trigger an engine run (ENG-011) |
| `updateDog` | action | dog fields | dog | DOB or household change triggers an engine run (FR-012) |
| `deleteAccount` | action | `{ confirm: true }` | `{ scheduledPurgeAt }` | soft delete now, purge in 30 days (NFR-007) |

## 6. Billing

| Operation | Transport | Input | Output | Notes |
|---|---|---|---|---|
| `createCheckoutSession` | action | `{ price: "monthly"\|"annual" }` | `{ url }` | Stripe Checkout with Stripe Tax; success/cancel URLs from `APP_URL` |
| `createPortalSession` | action | — | `{ url }` | Customer Portal |
| Stripe webhook | `POST /api/webhooks/stripe` | Stripe event | 200 | verifies signature; handles `checkout.session.completed`, `customer.subscription.*`, `invoice.payment_failed`; upserts `subscriptions`; idempotent on event id |
| `getEntitlement` | server component | — | `{ tier: "free"\|"premium", renewsAt?, status }` | derived (TS-003) |

## 7. Notifications

| Operation | Transport | Input | Output |
|---|---|---|---|
| `subscribePush` / `unsubscribePush` | action | PushSubscription JSON | `{ ok }` |
| Reminders cron | `POST /api/cron/reminders` | — | `{ sent, removed }` |

## 8. Cron routes

| Route | Schedule | Behavior |
|---|---|---|
| `POST /api/cron/agendas` | hourly | ARC 3.4; cursor-resumable; idempotent per dog per local date |
| `POST /api/cron/reminders` | every 15 min | ARC 3.5 |
| `POST /api/cron/purge` | daily | hard-delete soft-deleted accounts older than 30 days (DM-006) |
| `POST /api/cron/progress-audit` | nightly | DM-022 sample rebuild comparison |

## 9. Admin

| Operation | Transport | Input | Output | Notes |
|---|---|---|---|---|
| list frameworks | `GET /api/admin/frameworks` | — | per skill: published version, drafts | |
| get version | `GET /api/admin/frameworks/{skillId}/versions/{version}` | — | body + status | |
| save draft | `PUT /api/admin/frameworks/{skillId}/versions/{version}` | body | validation result (FS- ids) | draft only |
| validate | `POST .../validate` | body | `{ ok, errors: [{ rule, path, message }] }` | |
| publish | `POST .../publish` | — | `{ version, publishedAt }` | FS-011..014; retires the previous published version |
| preview | `POST .../preview` | body + sample dog | rendered stages and exercises (template substitution, no AI) | |
| config | `GET/PUT /api/admin/config/versions/{version}`, `POST .../publish` | config body | validation (CFG- ids) | |
| rebuild progress | `POST /api/admin/dogs/{dogId}/rebuild` | — | diff summary | DM-020 |

## 10. Rate limits

| Scope | Limit (config) |
|---|---|
| AI-backed operations per user | budgets in `ai.budgets` (AI-050) |
| `submitFeedback` per item | one accepted result; replays idempotent |
| Auth endpoints | Supabase defaults |
| Admin routes | admin only; no public exposure |

---
**Why**: The contract is written per operation rather than per HTTP route because the app uses server actions for most UI work and route handlers only where a browser is not the caller or a stream is needed. Idempotency keys on every retryable mutation are what make the offline feedback queue (ARC-021) safe. The Tier 1 answer union mirrors D-033 so the client cannot submit a rating shape the framework did not ask for.
