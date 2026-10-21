# Session Feedback Model

Status: written (Session 0); §4 (signal mapping) is finalized in Session 1. Decisions referenced: D-011, D-012, D-021. Requirements: FR-070 to FR-074, FR-080 to FR-082.

## 1. Principle

Feedback must be fast enough that every user submits it after every session, and rich enough that the engine can adapt. Those pull in opposite directions; the three-tier model resolves it: the required tier is tiny, the optional tiers add depth only when the user or the engine wants it.

## 2. Three tiers

| Tier | Who fills it | When | Content |
|---|---|---|---|
| 1 Required | Every user, every session | Immediately after the session, two taps and one rating | Completed: yes / partially / no. Success rating (§3). |
| 2 Optional notes | Engaged users | Same screen, after Tier 1 | Free text, one field, placeholder prompts ("What was hard? What distracted them?") |
| 3 Adaptive follow-up | AI, when useful | After Tier 1 (and Tier 2 if provided) | At most one question, dismissible. Triggered by a low rating, a problem mentioned in notes, or an engine signal needing disambiguation. Answer is structured (chips) where possible. |

Timing target: Tier 1 under 30 seconds on a phone (FR-072). Tier 3 MUST NOT appear on every session; the engine MUST have a reason (`../architecture/adaptive-engine-spec.md` defines the triggers).

## 3. The success rating

The original framework thresholds are written in success-rate terms (for example "80% over the last 3 sessions"), while the original feedback design was a 1–5 star rating. A star rating is a feeling; a success rate is a count. They are not interchangeable, and this gap is risk R-06.

**The Tier 1 question is defined by the skill's framework measurement type (D-033).** A single global widget cannot serve "out of ten tries" and "accidents today" at once.

| Measurement type | Skills | Tier 1 question | Stored |
|---|---|---|---|
| `cue_progression` | recall, sit/down/stay, tricks, foundations components | "Out of N tries, how many did {dog} get?" N = the exercise's repetitions; quick-pick row 0..N (one tap) | success = k/N, precision `counted` |
| `interval_progression` | leash walking | "How much of the walk was the leash loose?" hardly any / under half / about half / most / almost all | 0.1 / 0.3 / 0.5 / 0.75 / 0.95, precision `banded` |
| `management_count` | potty, jumping, barking | The skill's own count: "Accidents today?", "Jumps per greeting?", "Attention barks today?" | framework score function over count vs baseline/target, precision `counted` |
| `exposure_checklist` | socialization | "How did {dog} react?" relaxed / neutral / worried / fearful | 1.0 / 0.75 / 0.3 / 0.0; `fearful` also raises a fear signal |

The stored value is always a success proportion (0–1) with a `precision` flag so the engine can weight banded data below counted data (`../architecture/engine-config.md` `signals.bandedWeight`). "Completed: no" MUST NOT produce a success value; it is a skip signal with a reason.

An optional **difficulty tap** (too hard / about right / too easy) sits beside the rating (D-038). It never changes the success value; it adjusts the active D within the stage and can shorten the advance window (`../architecture/adaptive-engine-spec.md` ENG-053).

## 4. Signals produced per session

| Signal | Source | Consumer |
|---|---|---|
| completed (yes / partial / no) | Tier 1 | Progression: session counts toward thresholds only if yes or partial |
| success proportion + precision | Tier 1 | Progression: advance / maintain / simplify |
| duration actual vs planned | Timer or self-report (optional) | Fatigue detection, agenda sizing |
| skip reason | Skip action (FR-055) | Decay, overload, health flag |
| structured note signals | AI interpretation of Tier 2 (FR-074): distraction type, severity, environment, owner-execution issue, health flag, fear/stress indicator | Secondary adaptation signals, coaching guidance lookup |
| follow-up answer | Tier 3 | Same as note signals, higher confidence |

Environment count feeds the confidence score (`priority-engine.md` §4); in MVP, environment is taken from the exercise's planned environment plus any note or follow-up signal. Latency (response speed) is weighted to zero in MVP config (D-034) because no reliable signal measures it; the term stays in the formula for when one exists.

## 5. Coaching response contract

The response shown after feedback (FR-080) MUST follow this shape:

1. **Acknowledge** the result in one sentence, matched to the rating (no false praise for a 10% session).
2. **Observe** one specific thing, drawn from the framework's coaching guidance for the matched problem, or from the reason object (e.g., "Three sessions at 80% or better means Bailey is ready for more distance").
3. **Next** what changes tomorrow and why, in one sentence, from the reason object.
4. Optionally the Tier 3 question.

The AI phrases; the engine decides. If the AI is unavailable, the same four parts are rendered from templates keyed by the reason object (FR-082).

---
**Why**: The chat's PM role said the biggest product risk is whether people will log feedback at all, and that the right answer is a five-second required tier with depth layered on top. Reconciling that with the progression framework exposed that a star rating cannot drive an 80% threshold, so the rating is redefined here as a success proportion. Which input design wins is a Session 1 decision because it depends on how the engine weights banded versus counted data.
