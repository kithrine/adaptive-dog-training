# Progression Methodology

Status: written (Session 0) from the founder's progression framework (Jun 2026). Numeric values here are **starting defaults**; the authoritative, tunable values live in `../architecture/engine-config.md` (Session 1), and the per-skill stage tables live in `training-frameworks/` (v0 now, research-backed after D-030).

This document defines *when* to advance, repeat, or simplify. It does not define *how* to teach a cue; that is training methodology and belongs to the frameworks (D-016).

Requirement IDs: `PM-`.

## 1. The core question

Two 10-week-old puppies are both learning recall. Puppy A succeeds 90% of the time; Puppy B 20%. Tomorrow's agenda MUST differ for them: not different encouragement, a different plan with different exercises, difficulty, and goals. Everything below exists to compute that difference deterministically.

## 2. Skill trees and stages

| ID | Rule |
|---|---|
| PM-001 | Every skill is a tree of ordered **stages**. A dog has exactly one current stage per active skill. |
| PM-002 | Each stage defines its target values for the three D's (§3), its advancement threshold, and its exercise templates. |
| PM-003 | Stages are gated: a dog MUST NOT skip a stage. Advancement is one stage at a time; regression is one stage at a time unless a biology or fear signal forces a larger reset (§5). |
| PM-004 | Skills differ in shape. Cue-based skills (recall, sit, down, stay, leash) are progression trees. Management tracks (jumping, barking, potty) measure reduction in frequency of an unwanted behavior or absence of accidents, not response to a cue. The framework schema MUST support both measurement types. |

## 3. The three D's

Distance, Duration, Distraction are the universal advancement variables.

| ID | Rule |
|---|---|
| PM-010 | A stage transition MUST increase at most **one** D. |
| PM-011 | When a new D is introduced, the other D's MUST be reset to their baseline for that skill ("reset the other D's"). |
| PM-012 | When the dog moves to a new environment, all D's MUST reset to their baseline for that skill, regardless of stage (dogs do not generalize automatically). Specifically, distance is cut by at least 50% on first entering a more distracting environment. |
| PM-013 | Within a stage, exercises SHOULD vary the active D up and down (the "ping-pong" method: 2 s → 5 s → 3 s → 8 s) rather than only increasing it, to prevent pattern-based responding. |

## 4. Adaptation rules (advance / maintain / simplify)

Applied per skill after each counted session (feedback-model §4). "Recent window" = the last N counted sessions at the current stage; N defaults to 3.

| ID | Condition (defaults) | Action |
|---|---|---|
| PM-020 Advance | Success ≥ 80% across the recent window **and** window has ≥ 3 sessions **and** confidence ≥ 75 (`priority-engine.md` §4) **and** no biology gate blocks (§5) | Move to next stage. Increase one D; reset others (PM-010/011). |
| PM-021 Maintain | Success 50–79% in the recent window | Stay at stage. Vary exercises within the stage. MAY reduce one D for a session then return. Watch for plateau (§6). |
| PM-022 Simplify | Success < 50% across the last 2 counted sessions | Drop back one stage. Reduce all active D's. Check confound factors via Tier 3 (fatigue, distraction, hunger, health). Rebuild with high-success reps before re-attempting. |
| PM-023 | Thresholds are modified by owner tier (`priority-engine.md` §3): first-time 85%, intermediate 80%, experienced 75–80%. |
| PM-024 | Sessions marked "completed: no" MUST NOT count toward the window. "Partially" counts with its reported success value. |
| PM-025 | The engine MUST record the triggering rule ID and inputs in the reason object for every advance, maintain, or simplify decision (D-022). |

## 5. Gates and secondary signals

| ID | Signal | Detection (MVP sources) | Response |
|---|---|---|---|
| PM-030 Biology / age gate | Stage has a minimum age or physiological prerequisite (e.g., potty stages 3–4 require bladder maturity at ~4–6 months; off-leash reliability requires prior socialization) | Dog age in weeks from profile; framework stage `minimumAgeWeeks` | Stage MUST NOT be unlocked before the gate regardless of success. Failures before the gate MUST be labeled biological in the reason object, not behavioral, and coaching MUST say so. |
| PM-031 Fatigue | Within-session performance drops sharply (success falls > 30 points from start to end) | Tier 3 chip ("Did it get worse toward the end?"), or actual duration far over planned | End session guidance; next session shorter, D's reduced, start with an easy win. |
| PM-032 Fear / stress | Sudden drop in a previously mastered environment; note signals indicating fear; age in a known fear-period window (secondary fear period ~ 4 months, adolescence ~ 4–8 months) | Success history per environment; AI note interpretation; age | Freeze advancement for the skill. Return to the highest-confidence environment. Coaching explains fear periods. |
| PM-033 Plateau | Success 75–80% but unchanged for ≥ 7 sessions at a stage | Success history | Introduce novel versions of the same stage (new room, handler position, reward timing) before advancing. |
| PM-034 Regression after environment change | First sessions in a new environment score low | Exercise environment vs. history | Expected; apply PM-012 reset, do not simplify stage. |
| PM-035 Illness / stress flag | Skip reason "dog unwell" or note health signal | Skip reasons, note signals | Skip or light agenda; no threshold impact; prompt vet if persistent. |
| PM-036 Adolescent regression | Age ~ 4–8 months | Age | Previously mastered skills get extra maintenance priority (see priority Decay). Coaching normalizes regression. |

## 6. Learning phases

Each skill passes through four phases; the phase shapes reward schedule and success expectations. Phase is derived from stage position and confidence, not stored separately.

| Phase | Characteristics | Success expectation |
|---|---|---|
| Acquisition | Dog meets the cue. High reward frequency, low criteria, errors expected and informative. | 50% is acceptable; advance criterion is cue understanding, not reliability. |
| Fluency | Consistent in the original environment. Begin adding D's one at a time. Variable reinforcement may begin. | 80% before adding any D. |
| Generalization | Reliable across D levels in a familiar environment. Transfer to new environments with D reset (PM-012). | 80% in each new environment. |
| Maintenance | Reliable across environments. Intermittent reinforcement. | Scheduled maintenance reps 1–2× per week. |

## 7. Daily agenda constraints

Defaults; tunable in `../architecture/engine-config.md`.

| ID | Constraint | Default |
|---|---|---|
| PM-040 | Session length | 3–5 min |
| PM-041 | Sessions per day | 3–5 (bounded by owner's stated availability) |
| PM-042 | Reps per skill per session | 5–15 |
| PM-043 | Total daily training | 15–25 min (bounded by owner's stated minutes) |
| PM-044 | Skills per session | max 3 |
| PM-045 | Skills in active advancement | 1–2 |
| PM-046 | Maintenance skills per day | 1–2 |
| PM-047 | New skill introductions per day | ≤ 1 |
| PM-048 | Every session MUST end on a success rep (exercise templates end with an easy repetition). |
| PM-049 | Safety-critical skills (recall) appear daily once active; potty management is continuous; the current weakest skill is first in the day. |

Research basis: short, frequent sessions outperform long ones (Demant et al., 2011), to be re-cited in the research session.

## 8. Worked example

| | Puppy A (90%) | Puppy B (20%) |
|---|---|---|
| Rule fired | PM-020 Advance | PM-022 Simplify |
| Tomorrow | Recall Stage 2 (10 ft indoors), introduce Sit (acquisition), combine recall + leash intro in session 3, potty reinforcement evening | Recall Stage 1 simplified (2–3 ft, higher-value reward, no distractions, shorter), repeated across sessions, confidence games, no new skills, potty reinforcement evening |
| Reason object | `{skill: recall, stage: 1→2, success: 0.90, confidence: 81, rule: PM-020}` | `{skill: recall, stage: 1 (simplified), success: 0.20, rule: PM-022, confounds: [check_fatigue, check_distraction]}` |

---
**Why**: The chat's PM concluded the "intelligence of the product" is not how to teach recall but when to advance, repeat, or simplify, and that until this is specified the architect cannot design the adaptive engine. The founder's framework supplied the 3 D's, thresholds, biology gates, and learning phases. This document restates them as numbered rules so the engine spec can implement and test each one by ID. Numbers are defaults because the chat agreed they are hypotheses to calibrate on real cohorts, not validated constants.
