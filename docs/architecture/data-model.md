# Data Model

Status: written (Session 2). Owner: engineering. Decisions: D-023, D-024, D-027, D-040 to D-043. Rule IDs: `DM-`. Database: Supabase Postgres. Migrations live in `supabase/migrations/` and are the executable truth; this document is the design and the invariants a migration must preserve.

## 1. Principles

| ID | Rule |
|---|---|
| DM-001 | Training history is the source of truth. Per-skill progress is materialized state that MUST be rebuildable from history (D-042). |
| DM-002 | Frameworks and engine config are immutable once published; new behavior is a new version (D-024, CFG-001). |
| DM-003 | Every user-owned row is reachable from a `dogs` row or a `profiles` row and is protected by row-level security on the owner (NFR-006). The engine and AI layer write through the service role only. |
| DM-004 | Age is never stored as a number; it is computed from `dogs.dob` at read time (FR-011). |
| DM-005 | Physical quantities in framework bodies and resolved parameters carry explicit units (`{ "value": 10, "unit": "ft" }`); storage does not assume a locale (NFR-008). |
| DM-006 | Soft delete on `dogs` and `profiles` (`deleted_at`); hard purge by a scheduled job after 30 days (NFR-007). Catalog and framework tables are never user-deletable. |
| DM-007 | `dog_id` is on every training row even where derivable, so multi-dog accounts (FR-004) and RLS need no schema change. |

## 2. Entity overview

```
auth.users ─1:1─ profiles ─1:N─ dogs ─1:N─ dog_goals
                                 │  ├─1:N─ assessments
                                 │  ├─1:N─ skill_progress ──N:1─ framework_versions ──N:1─ skills ──N:1─ skill_groups
                                 │  ├─1:N─ agendas ─1:N─ agenda_items ──N:1─ framework_versions
                                 │  └─1:N─ dog_events
profiles ─1:N─ push_subscriptions, ai_calls, subscriptions
engine_config_versions   breeds   admin_users
```

## 3. Tables

Types are Postgres. `pk` = primary key, `fk` = foreign key. Enums are Postgres enums unless noted.

### 3.1 Identity and owner

**profiles** (pk `id` = `auth.users.id`)

| Column | Type | Notes |
|---|---|---|
| display_name | text | |
| email | text | mirrored from auth for queries |
| timezone | text | IANA; drives local-midnight runs (ENG-012) |
| owner_tier | enum first_time, intermediate, experienced | prior from onboarding, revised by engine (ENG-051) |
| minutes_per_day | int | FR-020 |
| sessions_per_day | int | FR-020 |
| equipment | text[] | ids from a static equipment list |
| tier_evidence | jsonb | rolling outcome stats used by ENG-051/052 |
| notification_prefs | jsonb | reminder time, weekly summary opt-in |
| onboarding_state | jsonb | resumable step pointer (FR-035) |
| created_at, updated_at, deleted_at | timestamptz | |

**admin_users** (pk `user_id` fk profiles) — membership grants admin RLS policies. Admin UI also checks a custom JWT claim set by a database hook.

### 3.2 Dog

**dogs** (pk `id` uuid)

| Column | Type | Notes |
|---|---|---|
| owner_id | fk profiles | |
| name | text | |
| breed_id | fk breeds, nullable | null = mixed/unknown |
| size_class | enum toy, small, medium, large, giant | from breed or user when mixed |
| dob | date | |
| dob_estimated | bool | |
| sex | enum female, male | |
| neutered | bool, nullable | |
| acquired_at | date, nullable | |
| training_history | enum none, some, formal | |
| household | jsonb | children_under_12, other_dogs, other_pets, home_type, yard |
| excluded_concerns | text[] | screened exclusions (D-004); never trained; drives escalation UI |
| created_at, updated_at, deleted_at | timestamptz | |

**breeds** (pk `id` text) — name, size_class, notes. Static catalog, seeded.

**assessments** (pk `id`)

| Column | Type | Notes |
|---|---|---|
| dog_id | fk dogs | |
| questionnaire_version | int | the branching definition used |
| answers | jsonb | structured answers keyed by question id, incl. AI follow-ups (FR-033) |
| screening | jsonb | per excluded-case screen: result and the triggering answers |
| status | enum in_progress, completed | FR-035 |
| completed_at | timestamptz, nullable | |

**dog_goals** (pk (`dog_id`, `skill_id`))

| Column | Type | Notes |
|---|---|---|
| rank | enum primary, secondary | user intent, FR-032 |
| source | enum user, recommended | engine-recommended goals the user did not pick (PE userGoal 0.2) |
| selected_at | timestamptz | |

### 3.3 Catalog and frameworks

**skill_groups** (pk `id` text) — label, display_order. E.g., `puppy_foundations` (D-035).

**skills** (pk `id` text, e.g., `recall`)

| Column | Type | Notes |
|---|---|---|
| name | text | |
| category | enum foundation, challenge, engagement | |
| measurement_type | enum cue_progression, interval_progression, management_count, exposure_checklist | D-033 |
| group_id | fk skill_groups, nullable | |
| display_order | int | |
| active | bool | hidden skills cannot be selected but existing progress continues |

Skills are system-defined (FR-032); rows are created by migration or admin, never by users.

**framework_versions** (pk `id` uuid; unique (`skill_id`, `version`))

| Column | Type | Notes |
|---|---|---|
| skill_id | fk skills | |
| version | int | monotonically increasing per skill |
| status | enum draft, published, retired | exactly one `published` per skill at a time |
| schema_version | int | `framework-schema.md` version the body conforms to |
| body | jsonb | the framework document (D-040) |
| change_notes | text | |
| created_by | fk profiles | |
| created_at, published_at, retired_at | timestamptz | |

Retired versions remain readable forever because dogs may still be locked to them (D-024).

**engine_config_versions** (pk `id`; unique `version`) — version int, status draft/published/retired, body jsonb, created_by, created_at, published_at. See CFG-001.

### 3.4 Training state

**skill_progress** (pk (`dog_id`, `skill_id`))

| Column | Type | Notes |
|---|---|---|
| framework_version_id | fk framework_versions | **locked** at activation (D-024, ENG-003) |
| status | enum deferred, active_advancing, active_maintenance, graduated, frozen | ENG §7.4 |
| current_stage | int | |
| stage_entered_at | timestamptz | |
| counted_sessions_at_stage | int | |
| confidence | int | 0–100, internal (D-019) |
| confidence_components | jsonb | success, consistency, latency, environments, recency |
| last_trained_at | timestamptz, nullable | |
| flags | jsonb | fear_until, fatigue, plateau, biology_gated, comeback_pending, execution_diagnostic |
| environment_history | jsonb | per environment class: best window success, sessions |
| owner_overrides | jsonb | swap records within window (ENG-074) |
| updated_at | timestamptz | |

Written only by the engine (service role). Rebuilt by `rebuild_progress(dog_id)` (§5).

**agendas** (pk `id`; unique (`dog_id`, `local_date`, `horizon_offset`))

| Column | Type | Notes |
|---|---|---|
| dog_id | fk dogs | |
| local_date | date | in owner timezone |
| horizon_offset | int 0..2 | 0 = today, 1–2 projected (ENG-094) |
| projected | bool | true for offsets 1–2 |
| generation | int | incremented on each regeneration of this date |
| engine_config_version | int | CFG-002 |
| trace | jsonb | rule ids fired, priority ranking with terms (ENG-060) |
| decisions | jsonb | per-skill decision + reason objects |
| planned_minutes | int | |
| generated_at | timestamptz | |

**agenda_items** (pk `id`)

| Column | Type | Notes |
|---|---|---|
| agenda_id | fk agendas | |
| dog_id | fk dogs | DM-007 |
| skill_id | fk skills | |
| framework_version_id | fk framework_versions | |
| kind | enum session, routine | ENG-092 |
| position | int | order within the day |
| stage | int | |
| exercise_id | text | template id within the framework body |
| params | jsonb | resolved parameters with units (distance, repetitions, duration, distraction, environment) |
| estimated_minutes | int | |
| locked | bool | entitlement gate (ENG-096) |
| reason | jsonb | reason object (ENG-100) |
| rendered | jsonb, nullable | AI-personalized text cache keyed by prompt version; null = render from template |
| status | enum pending, started, completed, partial, skipped | ENG-093 |
| started_at, completed_at | timestamptz, nullable | |
| success | numeric(4,3), nullable | 0–1 (D-033) |
| precision | enum counted, banded, nullable | |
| raw_answer | jsonb, nullable | the Tier 1 answer as given (k of N, chip, count, reaction) |
| difficulty | enum too_hard, about_right, too_easy, nullable | FR-075 |
| actual_minutes | int, nullable | |
| skip_reason | enum no_time, dog_unwell, other, nullable | FR-055 |
| notes | text, nullable | raw Tier 2 text; never parsed by the engine (FR-074) |
| note_signals | jsonb, nullable | AI-structured signals (ENG-023) |
| followup_trigger | text, nullable | Tier 3 trigger id (ENG §11) |
| followup_question, followup_answer | text / jsonb, nullable | |
| coaching | jsonb, nullable | the shown coaching response (acknowledge, observe, next) and its source (ai or template) |
| created_at, updated_at | timestamptz | |

A completed `agenda_items` row **is** the training record (D-041). Regeneration deletes `pending` items for today and inserts new ones; `started`, `completed`, `partial`, `skipped` rows are never deleted (ENG-093). Projected days are fully replaced.

**dog_events** (pk `id`) — dog_id, type (enum: goal_added, goal_removed, skill_activated, skill_swapped, tier_promoted, execution_diagnostic, comeback, profile_changed, framework_locked, escalation_shown), payload jsonb, created_at. Append-only audit trail; used by rebuild and analytics.

### 3.5 Platform

**subscriptions** (pk `user_id`) — provider, provider_customer_id, provider_subscription_id, status (trialing, active, past_due, canceled, none), plan, current_period_end, updated_at. Entitlement is derived: `premium` iff status in (trialing, active). Provider specifics in `tech-stack.md`.

**push_subscriptions** (pk `id`) — user_id, endpoint, keys jsonb, user_agent, created_at, last_success_at.

**ai_calls** (pk `id`) — user_id, dog_id, agenda_item_id nullable, purpose (enum questionnaire_followup, exercise_render, coaching, note_interpretation), model, prompt_version, input_tokens, output_tokens, cost_usd, latency_ms, status (ok, fallback, error), created_at. Feeds the per-user daily budget (NFR-005) and observability (NFR-010).

## 4. Row-level security

| Table | Authenticated user | Admin | Service role |
|---|---|---|---|
| profiles | read/write own | read all | all |
| dogs, assessments, dog_goals, dog_events | read/write own (via owner_id) | read all | all |
| skill_progress, agendas | read own | read all | all (only writer) |
| agenda_items | read own; **update own** limited to feedback columns (status, success, precision, raw_answer, difficulty, actual_minutes, skip_reason, notes, started_at, completed_at) via a security-definer function that validates transitions | read all | all |
| skills, skill_groups, breeds | read | read/write | all |
| framework_versions | read `published` and `retired` | read/write all | all |
| engine_config_versions | none | read/write | all |
| subscriptions | read own | read all | all (webhook writer) |
| push_subscriptions | read/write own | none | all |
| ai_calls | none | read all | all |

| ID | Rule |
|---|---|
| DM-010 | Users MUST NOT be able to write `skill_progress`, `agendas`, or non-feedback columns of `agenda_items`; those are engine outputs. |
| DM-011 | Feedback writes go through `submit_feedback(agenda_item_id, payload)` (security definer) which validates the state transition and then enqueues the engine run; clients never update rows directly. |
| DM-012 | Draft frameworks and engine config are invisible to non-admins. |

## 5. Rebuild and audit

| ID | Rule |
|---|---|
| DM-020 | `rebuild_progress(dog_id)` MUST exist: it replays `agenda_items` with feedback in `completed_at` order through the engine's progression and confidence steps using each item's locked framework version and the engine config version recorded on its agenda, and overwrites `skill_progress`. It is used after config changes, bug fixes, and in tests. |
| DM-021 | Rebuild MUST NOT regenerate agendas or alter history. |
| DM-022 | A nightly job compares a sample of materialized `skill_progress` rows with a rebuild and alerts on divergence. |

## 6. Indexes and performance

- `agenda_items (dog_id, skill_id, completed_at desc)` for recent-window queries.
- `agendas (dog_id, local_date desc)` for today lookup.
- `framework_versions (skill_id, status)` partial index on `status = 'published'`.
- `ai_calls (user_id, created_at)` for daily budget.
- Today's agenda for one dog MUST resolve in one query on `agendas` + one on `agenda_items` (NFR-002).

## 7. Seed data

Migrations seed `breeds`, `skill_groups`, `skills`, the first `engine_config_versions` (equal to `config/engine.default.json`, CFG-004), and the v0 `framework_versions` as `published` version 1 for each MVP skill. Research-backed frameworks (D-030) become version 2+.

---
**Why**: The chat's architect listed six entities (User, Dog, Skill, SkillProgress, Session, DailyAgenda) and said SkillProgress "becomes one of the most important tables in the system." Session 2 made three choices on top of that: frameworks are one JSON document per version because the engine reads them whole and nothing queries across stages (D-040); the agenda item doubles as the session record so plan and actual never drift (D-041); progress is materialized but always rebuildable from history so a progression bug is recoverable (D-042). The RLS split, where users can only touch feedback columns through a validated function, is what keeps "engine outputs are engine-owned" true at the database level rather than by convention.

## 8. Additions from Session 5 (entitlement)

- `profiles.trial_started_at`, `profiles.trial_ends_at` (timestamptz): app-managed 14-day trial (FM-002). Set by the profile-creation trigger.
- `trial_ledger` (pk `email_hash` text, `trial_started_at`): survives account purge so a re-created account within 90 days does not restart the trial (FM-003). Contains no reversible PII.
- `getEntitlement(profile, subscription, now)` is the single derivation (FM-001); `subscriptions.status` gains `past_due` handling with `entitlement.graceDays`.
