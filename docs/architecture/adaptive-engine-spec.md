# Adaptive Engine Specification

Status: written (Session 1). Owner: engineering + product. Decisions: D-009, D-010, D-016 to D-022, D-033 to D-039. Rule IDs: `ENG-`. Tunable values are named here and defined in `engine-config.md`; this document never states a number without naming the config key it comes from.

The adaptive engine is the product's intelligence: it turns a dog's state, an owner's capacity, and the locked frameworks into today's agenda and an explanation for every choice. It is deterministic, pure, and fully unit-testable. No LLM call occurs anywhere in this document.

## 1. Contract

```
generateAgenda(state: EngineState, config: EngineConfig, now: Instant): EngineOutput
```

| ID | Rule |
|---|---|
| ENG-001 | The engine MUST be a pure function: no I/O, no clock reads other than `now`, no randomness. Variation (exercise rotation, ping-pong values) MUST derive from deterministic seeds in state (e.g., counted-session index). |
| ENG-002 | Same `state`, `config`, and `now` MUST produce byte-identical `EngineOutput`. This is enforced by fixture tests (`../ops/testing-strategy.md`). |
| ENG-003 | The engine MUST NOT read framework data from anywhere except the versions referenced in `state.skillProgress[*].frameworkVersionId` (version locking, D-024). |
| ENG-004 | Every decision in `EngineOutput` MUST carry a reason object (§10). An agenda item without a reason is a bug. |

### 1.1 Inputs (`EngineState`)

| Field | Content |
|---|---|
| `dog` | id, ageWeeks (computed from DOB at `now`), sex, breedSize class, household flags (children, otherDogs), excluded concerns (never trained) |
| `owner` | tier (first_time, intermediate, experienced), minutesPerDay, sessionsPerDay, equipment set, timezone, tierEvidence (rolling outcomes) |
| `goals` | selected skills with rank (primary, secondary) and the engine's recommended-but-unselected skills |
| `frameworks` | the locked framework version per skill (see `framework-schema.md`), plus candidate frameworks for not-yet-activated skills (current published version) |
| `skillProgress[]` | per skill: status (deferred, active_advancing, active_maintenance, graduated, frozen), currentStage, stageEnteredAt, countedSessions at stage, confidence components, lastTrainedAt, flags (fear, plateau, fatigue, biologyGated), environment history |
| `sessions[]` | counted sessions (recent window plus history needed for signals): skill, stage, success proportion, precision, environment, actual vs planned minutes, difficulty tap, note signals, follow-up answers, completedAt |
| `today` | existing agenda for today (if any) with per-item status (pending, started, completed, skipped + reason) |
| `entitlement` | free or premium (affects only what is marked locked, §9) |

### 1.2 Output (`EngineOutput`)

| Field | Content |
|---|---|
| `agenda` | 3-day horizon (§8): day 1 ordered sessions + routine strip; days 2–3 projected |
| `decisions[]` | per-skill progression decision (advance, maintain, simplify, freeze, activate, defer, graduate) with reason |
| `signals[]` | detected secondary signals (fatigue, fear, plateau, env regression, comeback, overload) |
| `coachingHooks` | which reason objects the coaching layer should speak to, and whether a Tier 3 question is requested, with its trigger |
| `ownerTierUpdate` | optional tier promotion or execution-diagnostic flag |
| `trace` | ordered list of rule IDs fired, for debugging and tests |

## 2. When the engine runs

| ID | Trigger | Scope |
|---|---|---|
| ENG-010 | Session feedback submitted (Tier 1, plus Tier 2/3 if present) | Full run. Items already started or completed today are immutable (ENG-083). |
| ENG-011 | Profile or goal change affecting priority (age correction, household, goals, owner capacity) | Full run. |
| ENG-012 | Daily at local 00:00 for the dog's owner timezone (cron) | Full run; generates today's agenda if none exists. |
| ENG-013 | Comeback detection on first app open after a gap (§7.6) | Full run with comeback flag. |
| ENG-014 | Framework version publish | MUST NOT trigger a run for locked dogs. New activations use the new version. |

## 3. Pipeline

```
load state → 1 signal processing → 2 gates & secondary signals → 3 progression decisions
→ 4 confidence update → 5 owner tier revision → 6 priority scoring → 7 skill selection
→ 8 exercise selection → 9 agenda assembly → 10 reason objects → persist
```

Steps are executed in this order because later steps consume earlier outputs; each step is a separately testable module.

## 4. Step 1 — Signal processing

Converts raw feedback into normalized `SkillSignal` records. The Tier 1 question is defined by the framework's measurement type (D-033).

| Measurement type | Tier 1 question (framework-defined) | Stored signal |
|---|---|---|
| `cue_progression` | "Out of N tries, how many did {dog} get?" where N = exercise `repetitions`; quick-pick 0..N | success = k/N, precision `counted` |
| `interval_progression` (leash) | "How much of the walk was the leash loose?" chips: hardly any / under half / about half / most / almost all | success = 0.1 / 0.3 / 0.5 / 0.75 / 0.95, precision `banded` |
| `management_count` (potty, jumping, barking) | skill-specific count ("Accidents today?", "Jumps per greeting?") plus the framework's baseline | success = framework `scoreFn` over count vs. baseline/target (e.g., accidents 0 → 1.0, 1 → 0.5, ≥2 → 0.0), precision `counted` |
| `exposure_checklist` (socialization) | "How did {dog} react?" relaxed / neutral / worried / fearful per exposure | success = 1.0 / 0.75 / 0.3 / 0.0; `fearful` also raises a fear signal |

| ID | Rule |
|---|---|
| ENG-020 | `completed = no` MUST NOT produce a success signal; it is a skip with reason and feeds Decay and Overload only. `partially` produces a signal flagged `partial` and counts in the window (PM-024). |
| ENG-021 | Banded precision signals MUST be weighted by `config.signals.bandedWeight` relative to counted (`countedWeight`) when averaging. |
| ENG-022 | The optional difficulty tap (`too_hard`, `about_right`, `too_easy`) is stored on the session and consumed by §6.4 and §7. It MUST NOT change the success value. |
| ENG-023 | Free-text notes enter the engine only as structured `noteSignals` produced by the AI layer (`ai-orchestration.md`): `{distraction?: kind, severity?, environment?, ownerExecution?: kind, health?: bool, fear?: bool, fatigue?: bool}`. Missing or failed interpretation MUST be treated as no signal, never as an error. |
| ENG-024 | Environment for a session defaults to the exercise's planned environment and is overridden by a `noteSignals.environment` or Tier 3 answer. |

## 5. Step 2 — Gates and secondary signals

Evaluated per skill before any progression decision. Gates block; signals set flags that later steps honor.

| ID | Gate / signal | Condition (config keys) | Effect |
|---|---|---|---|
| ENG-030 Biology gate | `stage.minimumAgeWeeks > dog.ageWeeks` or stage `prerequisites` unmet | Stage locked. Decision cannot be `advance` into it. Reason `biology_gated`. Management/maintenance exercises for the current stage continue. |
| ENG-031 Exclusion gate | Skill in `dog.excludedConcerns` | Skill MUST NOT exist in state; if present, engine throws (data integrity). |
| ENG-032 Fatigue | Tier 3 `worseTowardEnd = true`, or `actualMinutes > plannedMinutes × config.signals.fatigueDurationRatio` with success < `config.signals.fatigueSuccessMax`, or `noteSignals.fatigue` | Flag `fatigue` for next agenda: shorter session, reduce active D one notch, lead with easy win. No threshold impact. |
| ENG-033 Fear / stress | Success in an environment previously ≥ tier threshold drops by ≥ `config.signals.fearDropPoints`; or `noteSignals.fear`; or exposure answer `fearful`; or age in `config.windows.fearPeriods` and drop ≥ `fearDropPointsInPeriod` | Flag `fear`: freeze advancement for `config.signals.fearFreezeDays`; next exercises use highest-confidence environment at reduced D. |
| ENG-034 Plateau | ≥ `config.signals.plateauSessions` counted sessions at stage with window success in `[plateauMin, tierThreshold)` | Flag `plateau`: exercise selection rotates to novel variants (§8.3) before any advance. |
| ENG-035 Environment regression | First `config.signals.newEnvGraceSessions` sessions in an environment not yet in the skill's environment history | Those sessions are excluded from simplify evaluation (PM-034) and D's are reset for that environment (PM-012). |
| ENG-036 Illness / stress | Skip reason `dog_unwell` or `noteSignals.health` | Flag `health`: today's agenda reduced to routine strip plus at most one light session; coaching prompts vet if ≥ `config.signals.healthPersistDays`. |
| ENG-037 Adolescence | `dog.ageWeeks` in `config.windows.adolescence` | Maintenance skills receive Decay bonus (§6.2); coaching normalizes regression. |
| ENG-038 Comeback | Days since last counted session ≥ `config.comeback.gapDays` | Flag `comeback` (§7.6). |

Precedence when flags conflict in later steps: `health` > `fear` > `fatigue` > `comeback` > `plateau`.

## 6. Steps 3–5 — Progression, confidence, owner tier

### 6.1 Recent window

For each active skill: the last `config.progression.windowSessions` counted sessions **at the current stage**, excluding sessions covered by ENG-035. Window success = precision-weighted mean (ENG-021).

### 6.2 Confidence (step 4)

```
confidence = Σ component_i × config.confidence.weights[i]   (weights sum to 1)
```

| Component | Computation |
|---|---|
| success | window success × 100 |
| consistency | 100 − clamp(stddev(window successes) × config.confidence.consistencyScale, 0, 100) |
| latency | neutral 70 unless a latency signal exists; **MVP weight 0** (D-034) |
| environments | 100 × min(1, distinct environments with window success ≥ tier threshold ÷ `stage.targetEnvironments`) |
| recency | 100 × max(0, 1 − daysSinceLastCounted ÷ config.confidence.recencyHorizonDays) |

Confidence is recomputed every run and stored with its components (for tuning and dashboards; never shown raw to users, D-019).

### 6.3 Progression decision (step 3)

Evaluated in this precedence; first match wins:

| ID | Decision | Condition |
|---|---|---|
| ENG-040 | `freeze` | Flag `fear` within freeze window, or `health`, or comeback not yet re-verified (§7.6) |
| ENG-041 | `simplify` | Last `config.progression.simplifySessions` counted sessions all < `config.progression.simplifyBelow`, and not in env grace (ENG-035). Effect: stage − 1 (floor 1), all active D's to stage baseline, `confoundChecks` added to coaching hooks. |
| ENG-042 | `advance` | window size ≥ `minSessions` (owner-tier modified, §6.4) **and** window success ≥ tier threshold **and** confidence ≥ `config.progression.advanceConfidence` **and** next stage not biology-gated **and** no `plateau` flag pending rotation. Effect: stage + 1, increase one D per stage definition, reset others (PM-010/011). |
| ENG-043 | `graduate` | `advance` from the final stage. Effect: status `active_maintenance`; frees an advancement slot (§7.4). |
| ENG-044 | `maintain` | Otherwise. Effect: stay; exercise selection varies within stage (§8). |

Advancement into a new environment (stage with a different `environment` class) applies PM-012 automatically: D's start at that stage's baseline, not at the previous stage's achieved values.

### 6.4 Owner tier modifiers and revision (step 5)

| Tier | advance threshold | minSessions | advancing cap | D step size | coaching depth |
|---|---|---|---|---|---|
| first_time | `config.tiers.first_time.threshold` (0.85) | 3 | 2 | small | heavy |
| intermediate | 0.80 | 3 | 3 | standard | as_needed |
| experienced | 0.78 | 2 | 4 | large | minimal |

| ID | Rule |
|---|---|
| ENG-050 | Tier modifiers are read from config per tier; code MUST NOT branch on tier names with literal numbers. |
| ENG-051 | Promotion: if the last `config.tiers.promotion.sessions` counted sessions across ≥ `promotion.minSkills` skills have mean success ≥ `promotion.meanSuccess`, promote one tier and emit `ownerTierUpdate` with reason `consistent_high_success`. Never promote past `experienced`. |
| ENG-052 | Execution diagnostic: if ≥ `config.tiers.diagnostic.simplifies` simplify decisions occur within `diagnostic.days`, emit flag `execution_diagnostic`. The coaching layer routes to technique checks (reward timing, cue repetition, criteria) before any further simplify. Tier is never demoted automatically. |
| ENG-053 | Difficulty tap: `too_hard` reduces the active D one notch for the next exercise of that skill (within stage). Two consecutive `too_hard` with success ≥ threshold → Tier 3 question (contradiction). `too_easy` with success ≥ threshold reduces `minSessions` for that skill by 1 (floor 2) for the current stage. |

## 7. Steps 6–7 — Priority scoring and skill selection

### 7.1 Candidate set

All skills in `goals` (selected and recommended) plus any skill with status `active_*`. Excluded concerns are never candidates (ENG-031). Components of a skill group (e.g., Puppy Foundations, D-035) are individual candidates.

### 7.2 Priority score

```
priority = safety×W.safety + window×W.window + weakness×W.weakness
         + userGoal×W.userGoal + decay×W.decay − overload×W.overload
```

All `W.*` from `config.priority.weights`. Term inputs are in [0, 1]:

| Term | Computation |
|---|---|
| safety | `framework.metadata.safetyWeight` |
| window | `framework.metadata.windowSensitivity × windowMultiplier(dog.ageWeeks)` using `config.windows.socialization.curve` (piecewise linear breakpoints) |
| weakness | `0.5 × (1 − stageProgress) + 0.5 × (1 − windowSuccess)`, where `stageProgress = (currentStage − 1) / (stageCount − 1)`; deferred skills use stage 1 and success 0.5; plus `config.priority.recentSimplifyBoost` if simplified within `recentSimplifyDays` |
| userGoal | primary 1.0, secondary 0.5, recommended-unselected `config.priority.recommendedGoalValue` (0.2) |
| decay | `min(1, daysSinceLastCounted / config.priority.decayHorizonDays)` + `config.priority.adolescenceMaintenanceBoost` when ENG-037 and status is maintenance, clamped to 1 |
| overload | 1.0 if the skill is not currently active and activating it would exceed the advancing cap or the owner's minutes; else 0 |

| ID | Rule |
|---|---|
| ENG-060 | The ranked list with per-term contributions MUST be included in the output trace and used by the Training Path explanation (FR-040, PE-002). |
| ENG-061 | A skill with `status = graduated` and no decay signal MUST score only its safety and decay terms (so recall keeps resurfacing for maintenance; sit does not). |

### 7.3 Selection

| ID | Rule |
|---|---|
| ENG-070 | Advancing slots = tier cap (§6.4), reduced by `config.agenda.capacityReduction` when `owner.minutesPerDay` < `config.agenda.lowCapacityMinutes`. Fill in priority order with skills whose decision is advance, maintain, or simplify. |
| ENG-071 | Maintenance slots = `config.agenda.maintenanceSlots` (1–2), filled in priority order with `active_maintenance` skills, each at most `config.agenda.maintenanceFrequencyPerWeek` times per week. |
| ENG-072 | Routine tracks (`framework.metadata.routine = true`, e.g., potty) are always included in the routine strip and do not consume slots or training minutes. |
| ENG-073 | At most `config.agenda.newSkillsPerDay` (1) skill may move from `deferred` to `active_advancing` per day, and only when a slot is free. Selection is the highest-priority deferred skill. The decision carries reason `activated_slot_freed`, and the coaching hook announces it (D-037). |
| ENG-074 | The owner MAY swap a newly activated skill for another deferred goal within `config.agenda.swapWindowDays`; the swap is recorded as an owner override in state and the engine honors it on the next run. Swaps do not bypass gates. |
| ENG-075 | Skill groups (D-035): at most `config.agenda.groupMaxConcurrent` (2) components of the same group may be advancing at once; the group is displayed under one label. |
| ENG-076 | Safety-critical skills (`safetyWeight ≥ config.agenda.dailySafetyThreshold`) with status advancing MUST appear every day once active (PM-049). |

### 7.4 Status transitions

```
deferred → active_advancing (ENG-073) → active_maintenance (graduate, ENG-043) → graduated (after config.agenda.maintenanceWeeks with confidence ≥ threshold)
active_* → frozen (ENG-040) → previous status when flag clears
```

### 7.5 Capacity fit

Planned minutes = Σ selected exercise `estimatedMinutes`. If planned > `owner.minutesPerDay`, drop the lowest-priority maintenance item, then the lowest-priority advancing item that is not safety-critical, until it fits. Never drop routine strip items. Reason `capacity_trimmed` is attached to dropped items so the Training Path can explain.

### 7.6 Comeback mode (D-039)

| ID | Rule |
|---|---|
| ENG-078 | Gap ≥ `config.comeback.gapDays` (5): first day back uses `config.comeback.minutesFactor` (0.6) of capacity, exercises at current stage with D reduced one notch and an easy-win opener; all skills are `freeze` for advancement until one fresh counted session at the stage confirms ≥ tier threshold; coaching normalizes the gap. |
| ENG-079 | Gap ≥ `config.comeback.regressDays` (14): skills with status `active_advancing` above stage 1 drop one stage before the comeback day; maintenance skills are unchanged. Confidence recency has already decayed naturally. |

## 8. Step 8 — Exercise selection

For each selected skill at its (possibly updated) stage:

| ID | Rule |
|---|---|
| ENG-080 | Candidate templates = `stage.exercises` filtered by owner equipment (`requiresEquipment ⊆ owner.equipment`) and by environment class allowed for the stage. If none match, fall back to the stage's `fallbackExercise` (every stage MUST define one; validated at framework publish). |
| ENG-081 | Rotation is deterministic: index = (countedSessionsAtStage + dayIndex) mod candidates. A `plateau` flag forces the `novelVariant` templates first. A `fatigue` or `fear` flag selects the lowest-difficulty candidate. |
| ENG-082 | Ping-pong (PM-013): the active D's value for this exercise is taken from `stage.pingPongSequence` by the same deterministic index, never monotonically increasing. |
| ENG-083 | Difficulty adjustments from ENG-053 modify D values within the stage's declared `range`; they MUST NOT exceed the stage's maximum or go below its minimum. |

## 9. Step 9 — Agenda assembly

| ID | Rule |
|---|---|
| ENG-090 | Day 1 agenda = ordered list of mini-sessions, no clock times (D-036). Session count = min(`owner.sessionsPerDay`, `config.agenda.maxSessionsPerDay`, number of selected items); one skill per session by default; two skills MAY share a session when both are ≤ `config.agenda.combineUnderMinutes` and sessions are fewer than selected skills. |
| ENG-091 | Order: health-flagged light session first if any; otherwise the weakest advancing skill first (PM-049), then remaining advancing by priority, then maintenance. The weakest advancing skill MAY repeat once in the day when session count ≥ `config.agenda.repeatWeakestFromSessions`. |
| ENG-092 | Routine strip: items from routine tracks for the current stage (e.g., potty schedule intervals), shown separately from numbered sessions. |
| ENG-093 | Immutability: on regeneration, day-1 items with status `started` or `completed` are copied unchanged; `skipped` items are removed; `pending` items are regenerated. |
| ENG-094 | Days 2 and 3 are projections computed by assuming `maintain` for every skill and the same capacity; they are labeled `projected` and fully regenerated on every run (FR-051). |
| ENG-095 | Every exercise item carries: skill, stage, template id, resolved parameters (distance, reps, duration, distraction, environment), estimatedMinutes, `endsOnSuccess = true` (PM-048), and its reason object. |
| ENG-096 | Entitlement: for `free` users, items beyond `config.entitlement.freeDailyItems` are included but marked `locked` so the UI can show the paywall without the engine knowing pricing (FR-110). Exact gating is defined in `../product/freemium-model.md`. |

## 10. Reason objects

Every decision and agenda item carries:

```json
{
  "skill": "recall",
  "frameworkVersion": "recall@3",
  "stage": { "from": 2, "to": 3 },
  "action": "advance",
  "rule": "ENG-042",
  "inputs": { "windowSuccess": 0.87, "windowSize": 3, "confidence": 81, "threshold": 0.85, "tier": "first_time" },
  "gates": [],
  "flags": [],
  "priority": { "total": 92, "terms": { "safety": 30, "window": 0, "weakness": 9, "userGoal": 15, "decay": 2, "overload": 0 } }
}
```

| ID | Rule |
|---|---|
| ENG-100 | Reason objects MUST be serializable, stable in shape, and versioned (`reasonSchemaVersion`) so the coaching layer and analytics can depend on them. |
| ENG-101 | The coaching layer (`ai-orchestration.md`) MUST derive its text from reason objects and the framework's coaching guidance only. It MUST NOT be given raw session history to re-decide anything. |

## 11. Tier 3 follow-up triggers

The engine requests at most one follow-up question per feedback submission, with a trigger id the AI layer uses to phrase it:

| Trigger | Condition |
|---|---|
| `low_success_unknown_cause` | success < `simplifyBelow` and no note signals |
| `fatigue_check` | duration ratio exceeded (ENG-032) without Tier 3 answer |
| `fear_check` | drop in mastered environment (ENG-033) without fear signal |
| `plateau_variation` | plateau flag first set |
| `contradiction` | difficulty tap contradicts success (ENG-053) |
| `comeback_checkin` | first counted session after comeback |
| `execution_check` | execution_diagnostic flag set (ENG-052) |

No trigger → no question (FR-073).

## 12. Testing contract

Fixture-driven tests MUST exist for every `ENG-` rule, plus the scenarios in `../product/progression-methodology.md` §8 (Puppy A / Puppy B), the four priority scenarios in `../product/priority-engine.md` §1, comeback at 5 and 14 days, capacity trimming, group cap, and version locking (publishing a new framework version changes nothing for a locked dog). Details in `../ops/testing-strategy.md`.

---
**Why**: The chat's architect stopped at "the next artifact is the adaptive engine specification: Priority Engine → Skill Selection → Stage Determination → Exercise Selection → Agenda Generation → AI Personalization." This document is that artifact. The additions relative to the chat are the ones that fell out of reconciling the pieces: a framework-defined Tier 1 question because management tracks cannot be scored out of ten, a zero-weighted latency term because nothing measures it yet, an explicit precedence order for conflicting signals, the comeback mode, and the immutability rule for started sessions. Every number is a config key so the "tunable, not hardcoded" decision (D-017) is structural rather than aspirational.
