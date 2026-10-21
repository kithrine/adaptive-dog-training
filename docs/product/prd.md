# Product Requirements Document — Biscuit

Status: written (Session 0), v2 of the BGlad PRD v1. Owner: product. Supersedes `Product Requirements Document.pdf` (Jun 2026).

Requirement IDs: `FR-` functional, `NFR-` non-functional. Stories (Session 7) MUST reference these IDs. Engine internals are specified in `../architecture/adaptive-engine-spec.md`; this document states *what* the product does.

## 1. Summary

Biscuit is an adaptive dog-training coach. An owner describes their dog and goals, receives a personalized training path, gets a daily agenda of fully specified exercises, logs how each session went in under 30 seconds, receives coaching, and sees tomorrow's agenda adapt. Methodology comes from expert, positive-reinforcement frameworks; a deterministic engine decides what to train; AI explains and coaches.

Target users, problem, vision, and hypothesis: see `../planning/project-brief.md`.

## 2. Core product loop

```
Assess → Training Path → Daily Agenda → Session → Feedback → Coaching → Adapt → repeat
```

## 3. Functional requirements

### 3.1 Accounts

| ID | Requirement |
|---|---|
| FR-001 | Users MUST be able to register with email + password and with at least one OAuth provider (Google). |
| FR-002 | Users MUST be able to log in, log out, and reset their password by email. |
| FR-003 | Users MUST be able to edit profile information (display name, email, notification preferences) and delete their account with all associated data. |
| FR-004 | An account has exactly one dog in MVP. The data model MUST allow multiple dogs per account without migration (Phase 2). |

### 3.2 Dog profile

| ID | Requirement |
|---|---|
| FR-010 | Users MUST be able to create a dog profile with: name, breed (from a canonical list, with "mixed/unknown"), date of birth or approximate age, sex, spay/neuter status, date acquired, prior training history (none / some / formal classes), household info (children under 12, other dogs, other pets, home type, yard access). |
| FR-011 | Dog age MUST be stored as a date (DOB or estimated DOB) so that age in weeks is computed, not stored. |
| FR-012 | Users MUST be able to edit the dog profile. Changes that affect prioritization (age correction, household change) MUST trigger agenda regeneration. |

### 3.3 Owner profile

| ID | Requirement |
|---|---|
| FR-020 | During onboarding the user MUST answer: prior dog-training experience (never / one or two dogs or a puppy class / multiple dogs, comfortable with shaping and distraction work), realistic minutes per day, sessions per day they can do, and training equipment on hand (treats, treat pouch, clicker or marker word, long line, crate, harness/collar). |
| FR-021 | The system MUST derive an owner skill tier (first-time / intermediate / experienced) from FR-020 and MUST allow the engine to revise it from outcomes (see `priority-engine.md` §3). |

### 3.4 Dynamic assessment (questionnaire)

| ID | Requirement |
|---|---|
| FR-030 | The questionnaire MUST branch on dog age (puppy under 6 months / adolescent 6–18 months / adult), selected goals, reported behavior concerns, owner experience, and available time. Irrelevant branches MUST NOT be shown. |
| FR-031 | The questionnaire MUST include safety screening (see `safety-escalation.md`). Any positive screen for an excluded case (D-004) MUST stop the standard flow and show the escalation path. The user MAY continue with in-scope goals only, with the excluded concern recorded and never trained. |
| FR-032 | Users MUST select training goals from the system-defined skill list (D-003). Users MUST NOT be able to create free-text skills. |
| FR-033 | The AI MAY generate at most two follow-up questions per branch when a structured answer is ambiguous (D-021). Follow-ups MUST produce structured answers, not free text stored as-is. |
| FR-034 | The full assessment MUST be completable on a phone in under 5 minutes for a typical puppy owner. |
| FR-035 | The assessment MUST be resumable if abandoned mid-way. |

### 3.5 Personalized training path

| ID | Requirement |
|---|---|
| FR-040 | After assessment the system MUST present a "Meet Your Dog's Training Path" screen before Day 1, containing: primary goals (the skills the engine prioritized), secondary goals, estimated daily commitment, current training stage, and a plain-language explanation of the strategy (why these first, what comes later). |
| FR-041 | The explanation MUST be generated from the engine's reason objects (D-022); it MUST NOT contain recommendations that are not in the computed path. |
| FR-042 | If the user selected more goals than the engine's active-skill cap allows, the screen MUST say which goals are deferred and that they will be introduced as progress allows. |

### 3.6 Daily agenda

| ID | Requirement |
|---|---|
| FR-050 | The home screen MUST be today's agenda (D-013): the list of training sessions for today, each with skill, stage name, estimated minutes, and a total for the day. |
| FR-051 | The engine MUST generate an agenda horizon of 3 days. Only today MUST be shown by default; the next two days MAY be previewed and MUST be labeled as subject to change. |
| FR-052 | The agenda MUST respect the owner's stated minutes per day and sessions per day, and the engine's per-session limits (`progression-methodology.md` §6). |
| FR-053 | The agenda MUST be regenerated after every submitted session feedback, after any profile change affecting priority, and at least once per calendar day in the user's timezone. |
| FR-054 | A regenerated agenda MUST NOT change a session the user has already started or completed today. |
| FR-055 | Users MUST be able to skip a session with a reason (no time / dog unwell / other). Skips are signals, not failures. |

### 3.7 Exercises

| ID | Requirement |
|---|---|
| FR-060 | Every exercise shown to a user MUST contain: goal (why it matters), setup (environment, equipment), step-by-step instructions, duration or repetitions, success criteria, common mistakes, and what to do if the dog struggles (D-015). |
| FR-061 | Exercise text MUST be written for an owner who has never trained a dog. Reading level target: grade 6–8. |
| FR-062 | Exercise text MUST be rendered from the framework's structured exercise template plus the dog's details (name, stage parameters). The AI MAY rephrase; it MUST NOT change parameters (distance, reps, duration, distraction level) or add techniques not in the framework (D-007). |
| FR-063 | When AI personalization is unavailable, the system MUST render exercises from deterministic templates so that training is never blocked. |

### 3.8 Session feedback

| ID | Requirement |
|---|---|
| FR-070 | After a session the user MUST be able to submit: completed (yes / partially / no) and a success rating. The structure of the rating and its mapping to engine signals is specified in `feedback-model.md`. |
| FR-071 | Notes MUST be optional and free text. |
| FR-072 | Required feedback MUST be submittable in under 30 seconds on a phone (two taps plus one rating). |
| FR-073 | The AI MAY ask one follow-up question when the rating is low, the notes mention a problem, or the engine detects a signal (fatigue, fear, plateau) that notes could disambiguate. The user MUST be able to dismiss it. |
| FR-074 | Free-text notes MUST be converted by the AI into structured signals (distraction type, severity, environment, owner-execution issue, health flag) before the engine consumes them (D-021). The raw note is stored; the engine never parses raw text. |

### 3.9 Coaching

| ID | Requirement |
|---|---|
| FR-080 | After feedback submission the system MUST show a coaching response within the same screen flow: acknowledgement of the result, one specific observation, what changes next and why. |
| FR-081 | Coaching responses MUST be grounded in the engine's reason object and the framework's coaching guidance. The AI MUST NOT recommend techniques, tools, or escalations outside the framework and `safety-escalation.md`. |
| FR-082 | If the LLM is unavailable, a deterministic coaching response keyed by reason object MUST be shown instead. |

### 3.10 Adaptation

| ID | Requirement |
|---|---|
| FR-090 | The engine MUST advance, maintain, or simplify each active skill according to the thresholds in the dog's locked framework version and the owner tier modifiers (`progression-methodology.md` §4, `priority-engine.md`). |
| FR-091 | The engine MUST apply age and biology gates regardless of performance (`progression-methodology.md` §5). |
| FR-092 | The engine MUST detect and respond to secondary signals: fatigue, fear/stress, plateau, regression after environment change. |
| FR-093 | Every adaptation MUST produce a reason object that is stored with the agenda and available to the coaching layer. |

### 3.11 Progress

| ID | Requirement |
|---|---|
| FR-100 | Users MUST be able to see, per active skill, the current stage name, a plain-language status (e.g., "progressing well, nearly ready for more distraction"), and stage history. Numeric confidence MUST NOT be shown (D-019). |
| FR-101 | Users MUST be able to see a session history with date, skills, completion, rating, and notes. |

### 3.12 Freemium

| ID | Requirement |
|---|---|
| FR-110 | Free and premium entitlements are defined in `freemium-model.md`. The engine MUST read entitlement state and MUST NOT depend on billing provider details. |
| FR-111 | Subscription purchase, renewal, cancellation, and restore MUST work on the web via the billing provider defined in `../architecture/tech-stack.md`. |

### 3.13 Safety and escalation

| ID | Requirement |
|---|---|
| FR-120 | Screening, exclusion handling, disclaimer presentation, and escalation copy are specified in `safety-escalation.md`. Those requirements are normative for this PRD. |

### 3.14 Framework administration

| ID | Requirement |
|---|---|
| FR-130 | An admin-only panel MUST allow creating, editing, validating, and publishing framework versions (D-024). Publishing a new version MUST NOT change the version any existing dog is locked to. |
| FR-131 | Engine configuration (weights, thresholds, caps) MUST be editable without a code deploy (D-017). |

### 3.15 Notifications (P1)

| ID | Requirement |
|---|---|
| FR-140 | Users SHOULD be able to opt into a daily training reminder via web push at a chosen time. |
| FR-141 | Users SHOULD receive a weekly summary (sessions completed, stages advanced, one coaching highlight). |

## 4. Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-001 | Mobile-first responsive web app, installable as a PWA, usable offline for reading today's agenda and exercise text (feedback submission MAY queue). |
| NFR-002 | Today's agenda MUST render within 2 seconds on a mid-range phone on 4G after login (p75). |
| NFR-003 | Agenda regeneration after feedback MUST complete within 5 seconds (p95); the coaching response MAY stream. |
| NFR-004 | All engine decisions (advancement, priority, confidence, gates) MUST be deterministic and covered by fixture-driven unit tests. Same inputs, same outputs. |
| NFR-005 | LLM cost MUST be bounded per user per day by configuration; exceeding the budget degrades to deterministic fallbacks, never to a blocked user. |
| NFR-006 | User data MUST be isolated per account (row-level security). Admin framework data is read-only to users. |
| NFR-007 | Account deletion MUST remove or anonymize all personal data within 30 days. |
| NFR-008 | The system MUST be built for US/English only but MUST NOT hardcode locale, currency, or units in a way that blocks later localization (distances stored in a canonical unit, displayed per locale). |
| NFR-009 | Accessibility: WCAG 2.1 AA for color contrast, focus order, and touch targets (44 px). |
| NFR-010 | Observability: every agenda generation, adaptation, and LLM call MUST be logged with the reason object and correlation ID. |

## 5. Out of scope for MVP

Video analysis, photo analysis, community features, trainer marketplace, family accounts, wearable integrations, multiple dogs (data model only), aggression, resource guarding, severe behavioral disorders, native apps.

## 6. Success metrics

See `../planning/project-brief.md` §7. The single most important MVP metric is **Tier 1 feedback submitted per completed session**, because the hypothesis cannot be tested without it.

## 7. Open items resolved in later sessions

| Item | Session | Doc |
|---|---|---|
| Exact rating-to-signal mapping | 1 | `feedback-model.md`, `../architecture/adaptive-engine-spec.md` |
| Framework JSON schema | 2 | `../architecture/framework-schema.md` |
| LLM prompts, schemas, budgets | 3 | `../architecture/ai-orchestration.md` |
| Billing provider, push, PWA details | 4 | `../architecture/tech-stack.md` |
| Screening questions, escalation copy, gating, pricing | 5 | `safety-escalation.md`, `freemium-model.md` |

---
**Why**: The PRD v1 from June was a solid vision statement but left the engine's behavior as "the AI adapts the training." v2 keeps the same scope and makes every behavior a numbered, testable requirement, pushes engine specifics into the architecture docs, and adds the requirements discovery implied but never wrote down (fallbacks when AI is unavailable, agenda immutability for started sessions, structured notes, skip reasons, deletion).
