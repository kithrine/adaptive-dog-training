# Training Frameworks

One document per MVP skill. Each describes the skill's progression tree in the shape the engine consumes: metadata, stages (with 3 D targets and advancement threshold), milestones, exercise templates, advancement and regression rules, and coaching guidance (common problem → response).

## Status

| File | Status |
|---|---|
| `recall.md`, `potty-training.md`, `leash-walking.md`, `sit-down-stay.md`, `jumping.md`, `barking.md` | **v0**: founder-authored from research notes (Jun 2026). Shape reference for the framework schema (Session 2). |
| `home-alone-confidence.md`, `trick-training.md`, `puppy-foundations.md`, `socialization.md` | **scope brief**: what the research session must produce. No stage data yet. |

Per D-030, every file is replaced by a research-backed v1 (cited sources, trainer review) after the engine spec and framework schema exist. Until then, treat numeric values as plausible defaults, not validated protocols.

## Measurement types

- **Cue progression** (recall, sit/down/stay, leash, tricks): success = correct response to cue; thresholds are success proportions.
- **Management track** (potty, jumping, barking, home-alone): success = reduction in unwanted behavior frequency or absence of incidents over a period; thresholds are expressed as "no accidents for N days" or "X% reduction from baseline".

The framework schema MUST support both (PM-004).

## Common structure of a v1 file

1. Metadata: skill id, category, measurement type, safety weight, minimum/maximum age weeks, prerequisites.
2. Stages: number, name, 3 D targets, environment, session guidance, advance-when, biology gate.
3. Milestones: name, stage, confidence required.
4. Exercise templates per stage: parameters (distance, reps, duration, distraction, environment), setup, steps, success criteria, common mistakes, troubleshooting.
5. Advancement and regression rules (overrides of the global defaults in `../progression-methodology.md`, if any).
6. Coaching guidance: common problems → recommended response, and owner-execution checks.
7. Sources.
