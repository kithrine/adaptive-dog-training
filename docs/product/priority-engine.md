# Priority Engine, Owner Model, Confidence Score, Age Windows

Status: written (Session 0) from the founder's "three gaps" document (Jun 2026). Numeric values are **starting defaults**; authoritative tunable values live in `../architecture/engine-config.md`. Decisions referenced: D-017, D-018, D-019. Requirement IDs: `PE-`.

Progression (`progression-methodology.md`) decides how a skill advances. This document decides **which skills get trained tomorrow** and **how confident the engine is** in a skill. Together they are the adaptive engine.

## 1. Priority formula

Every active skill gets a priority score recomputed on every agenda generation. The top N skills (N = active-skill cap, §3) form tomorrow's agenda.

```
Priority = Safety + Window + Weakness + UserGoal + Decay − Overload
```

| Term | Meaning | Input (0–1) × default weight |
|---|---|---|
| Safety | How much the skill prevents harm (recall, leave-it, potty hygiene) | per-skill constant in framework metadata × 30 |
| Window | Urgency from a closing developmental window (§5) | window multiplier for dog age × 30 |
| Weakness | How far the skill is from mastery and how recently it failed | derived from stage position and recent success × 20 |
| UserGoal | The owner selected or ranked this skill | 1.0 primary goal, 0.5 secondary, 0.2 not selected but recommended × 15 |
| Decay | Time since last trained, and regression risk (adolescence, recent simplify) | days since last session and regression flags × 15 |
| Overload | Penalty applied when the owner's capacity is exceeded or too many skills are already active | computed from capacity and active count × 12 |

| ID | Rule |
|---|---|
| PE-001 | All weights, multipliers, and the active-skill cap MUST be configuration (D-017). Code MUST NOT contain literal weight values. |
| PE-002 | The engine MUST output the ranked list with each term's contribution so the reason object and the Training Path screen can explain the ranking (D-022). |
| PE-003 | A skill blocked by a biology gate (PM-030) MUST NOT be scheduled for advancement but MAY be scheduled for management or maintenance exercises defined by its framework (e.g., potty management continues while stage 3 is gated). |
| PE-004 | Excluded concerns (D-004) MUST NOT exist as skills and therefore cannot be scored. |
| PE-005 | Selected goals above the cap MUST be deferred, not dropped; the Training Path screen says so (FR-042) and they enter the ranking when a slot frees. |

### Worked scenarios (defaults, 10-week puppy, cap 3)

| Scenario | Effect |
|---|---|
| Typical | Recall (safety 1.0) and potty (goal 0.9, weakness 0.7) lead; socialization rises with the window; sit trails. |
| Socialization window closing (13 wks) | Window multiplier peaks; socialization jumps to the top even if the dog is doing fine elsewhere. |
| Recall regressed | Weakness 0.95 and decay 0.9 push recall to the top with a simplify action. |
| First-time owner | Cap drops to 2; the third-ranked skill is deferred. |

## 2. Agenda builder (consumer of the ranking)

The ranking selects *which* skills; the progression stage selects *what* exercise; the daily constraints (PM-040 to PM-049) select *how much*. The detailed pipeline is specified in `../architecture/adaptive-engine-spec.md` (Session 1).

## 3. Owner skill model

Owner skill is a multiplier on progression, not a separate track. Training failure is frequently owner-execution failure; the engine must disambiguate dog-side, owner-side, and biology-side causes.

| Tier | Who | Stage step size | Advance threshold | Concurrent skills (cap) | Technique coaching |
|---|---|---|---|---|---|
| First-time | Never trained a dog | Smaller | 85% | 1–2 | Heavy |
| Intermediate | One or two dogs, or a puppy class | Standard | 80% | 2–3 | As needed |
| Experienced | Multiple dogs; comfortable with shaping, variable schedules, distraction work | Larger | 75–80% | 3–4 | Minimal |

| ID | Rule |
|---|---|
| PE-010 | Tier is set from onboarding answers (FR-020) as a prior. |
| PE-011 | The engine MUST revise tier from outcomes: consistent high success (e.g., ≥ 90% over 10 sessions) promotes; repeated regression on an "experienced" owner triggers an execution-diagnostic coaching path rather than demotion. Exact rules in Session 1. |
| PE-012 | A failed session has three candidate causes: stage too hard (dog), execution error (owner), not yet mature (biology). Lower owner tier raises the prior on execution error and routes the coaching response toward technique feedback (reward timing, criteria, consistency) before simplifying the dog's stage. |

## 4. Confidence score

Success rate alone lies: a dog at 90% in one quiet room is not as fluent as a dog at 85% across five environments. Confidence (0–100) blends fluency evidence into one number used for advancement gating and, later, dashboards.

```
Confidence = 0.35·success + 0.20·consistency + 0.15·latency + 0.20·environments + 0.10·recency
```

| Component | Meaning | MVP source |
|---|---|---|
| success | Recent success proportion | Tier 1 |
| consistency | Low variance across recent sessions | Tier 1 history |
| latency | Response speed (fast = fluent) | Tier 3 chip when asked; otherwise neutral default (Session 1 decides whether to zero-weight) |
| environments | Number of environments with success at this stage | Exercise environment + note signals |
| recency | Decays with days since last successful session | Session dates |

| ID | Rule |
|---|---|
| PE-020 | Advancement gate is **confidence ≥ 75**, not success ≥ 80 alone (PM-020 requires both). The two usually agree; confidence prevents advancing a brittle single-context behavior. |
| PE-021 | Confidence is internal only in MVP (D-019). User-facing status MUST be plain language: "Recall is progressing well and is nearly ready for more distractions" / "Recall still needs more practice before advancing". |
| PE-022 | Milestones in frameworks specify `confidenceRequired`; a milestone is achieved when confidence at its stage meets it. |

Worked example: Puppy A (success 85, consistency 88, latency 82, environments 80, recency 90) → 85. Puppy C (85, 55, 45, 20, 70) → 61. Same raw success; C does not advance.

## 5. Age windows

The priority engine needs a clock, not just a scoreboard.

| Window | Age | Engine behavior |
|---|---|---|
| Socialization | ~3–14 weeks, closes permanently | Socialization priority ramps toward maximum as 14 weeks approaches, then drops sharply. |
| Bladder control matures | ~12–16 weeks (full ~4–6 months) | Potty stages 3–4 unlock on maturity; failures before are labeled biological. |
| Adolescent regression | ~4–8 months | Previously mastered skills get extra maintenance priority; coaching normalizes regression. |

Default window multiplier for socialization (age in weeks): 0.4 + 0.05·(age−8) for ≤ 12; 0.7 + 0.45·(age−12) for 12–14 (peaks 1.6); 1.6 − 0.6·(age−14) for 14–16, floor 0.1; 0.05 after 16. Curve shape and breakpoints are configuration (PE-001) and will be re-grounded in the research session.

| ID | Rule |
|---|---|
| PE-030 | Window multipliers MUST be computed from the dog's age in weeks at agenda time, never stored. |
| PE-031 | Adult dogs (profile age beyond all windows) MUST get zero window contribution; the formula degrades gracefully to safety, weakness, goal, decay, overload. |

## 6. Composition

```
Priority Engine → select skills (top N within cap)
      ↓
Progression Engine → determine stage and action per skill (advance / maintain / simplify)
      ↓
Exercise Selection → pick templates for the stage and environment
      ↓
Agenda Generation → fit into sessions under PM-040..049 and owner capacity
      ↓
AI Personalization → render exercise text and coaching language
```

All steps above the last are deterministic (D-020).

---
**Why**: The chat's PM identified three gaps after the progression framework: no owner model, no confidence score, and no answer to "the dog has five skills at five stages, what gets trained tomorrow?" The founder's second document closed them with this formula, the tier table, and the five-component confidence score. The PM's only change, adopted here as D-019, was to keep confidence internal. The chat's architect then observed that skill *selection* (this doc) and skill *progression* (the other doc) are two different systems and that together they are the moat.
