# Framework Schema

Status: written (Session 2). Owner: engineering + product. Decisions: D-015, D-023, D-024, D-033, D-035, D-040, D-043. Rule IDs: `FS-`. Schema version: 1.

A framework is the structured, versioned definition of one skill: how it is measured, its stages with 3 D targets, exercise templates, milestones, advancement and regression rules, and coaching guidance. It is a domain model, not content (D-023). The engine (`adaptive-engine-spec.md`) and the AI layer (`ai-orchestration.md`) consume it; the admin JSON editor (D-043) produces it; `framework_versions.body` stores it (`data-model.md` §3.3).

## 1. Document shape (TypeScript)

```ts
export interface Framework {
  schemaVersion: 1;
  metadata: Metadata;
  measurement: Measurement;
  stages: Stage[];                 // ordered, numbered 1..n
  milestones: Milestone[];
  rules?: RuleOverrides;           // optional per-skill overrides of engine-config progression values
  coaching: CoachingGuidance;
  routine?: RoutineSchedule;       // required when metadata.routine = true
  sources: Source[];
}

interface Metadata {
  skillId: string;                 // matches skills.id
  version: number;
  name: string;
  summary: string;                 // one paragraph for the Training Path screen
  category: "foundation" | "challenge" | "engagement";
  groupId?: string;                // skill_groups.id (D-035)
  safetyWeight: number;            // 0–1 (priority Safety term)
  windowSensitivity: number;       // 0–1 (priority Window term; 0 for most skills)
  routine: boolean;                // true = continuous track in the routine strip (ENG-072)
  minimumAgeWeeks: number;
  maximumAgeWeeks?: number;
  prerequisites: Array<{ skillId: string; stage: number }>;
  equipmentOptions: string[];      // equipment ids this skill can use
}

type Measurement =
  | { type: "cue_progression"; tier1: { prompt: string } }                       // "Out of {n} tries, how many did {dog} get?"
  | { type: "interval_progression"; tier1: { prompt: string; bands: Array<{ label: string; value: number }> } }
  | { type: "management_count"; tier1: { prompt: string; unit: string; scoreMap: Array<[count: number, score: number]>; baselineDays: number } }
  | { type: "exposure_checklist"; tier1: { prompt: string; reactions: Array<{ label: string; value: number; fear?: boolean }> } };

interface Stage {
  number: number;
  name: string;
  description: string;             // trainer-facing
  learningPhase: "acquisition" | "fluency" | "generalization" | "maintenance";
  environmentClass: "indoor_quiet" | "indoor_busy" | "yard" | "outdoor_quiet" | "outdoor_busy" | "public" | "any";
  targetEnvironments: number;      // distinct environments needed for full confidence (PE environments component)
  dTargets: {
    distance?: Quantity & { range: [number, number] };
    duration?: Quantity & { range: [number, number] };
    distraction?: { level: 0 | 1 | 2 | 3 | 4; range: [number, number] };
  };
  activeD?: "distance" | "duration" | "distraction";   // the D this stage increases vs the previous (PM-010); omitted for stage 1
  pingPongSequence?: number[];     // values for activeD across consecutive exercises (PM-013), within range
  minimumAgeWeeks?: number;        // biology gate (ENG-030)
  prerequisites?: Array<{ skillId: string; stage: number }>;
  advanceOverride?: { threshold?: number; minSessions?: number; confidence?: number };
  simplifyOverride?: { sessions?: number; below?: number };
  advanceWhen: string;             // human-readable criterion for admin preview and coaching
  exercises: Exercise[];
  fallbackExerciseId: string;      // must be an exercise in this stage with no equipment requirement
}

interface Quantity { value: number; unit: "ft" | "m" | "s" | "min" | "reps" }

interface Exercise {
  id: string;                      // unique within framework, stable across versions where possible
  name: string;
  kind: "drill" | "routine" | "exposure";
  difficulty: 1 | 2 | 3 | 4 | 5;
  novelVariant: boolean;           // used first when plateau flag is set (ENG-081)
  requiresEquipment: string[];
  params: {
    repetitions?: number;          // N for the cue_progression Tier 1 question
    distance?: Quantity;
    duration?: Quantity;
    distraction?: 0 | 1 | 2 | 3 | 4;
    environment: Stage["environmentClass"];
  };
  estimatedMinutes: number;
  template: ExerciseTemplate;
}

interface ExerciseTemplate {          // D-015: all seven parts required
  goal: string;
  setup: string[];
  steps: string[];                 // last step MUST be an easy-win repetition (PM-048)
  successCriteria: string;
  commonMistakes: string[];
  troubleshooting: string[];
  // Placeholders allowed in any string: {dog}, {distance}, {duration}, {repetitions}, {distraction}, {environment}, {owner}
}

interface Milestone { id: string; name: string; stage: number; confidenceRequired: number; celebrationCopy: string }

interface RuleOverrides {
  windowSessions?: number;
  advanceConfidence?: number;
  maintenanceFrequencyPerWeek?: number;
}

interface CoachingGuidance {
  stageIntro: Record<number, string>;            // what the owner is told when entering each stage
  commonProblems: Array<{
    id: string;
    match: {                                      // any listed condition matches (OR); omitted = always eligible
      distraction?: Array<"noise" | "people" | "dogs" | "animals" | "food" | "toys" | "environment" | "unknown">;
      ownerExecution?: Array<"cue_repeated" | "reward_late" | "reward_position" | "criteria_too_high" | "inconsistent_household" | "session_too_long">;
      successBelow?: number;
      flags?: Array<"fatigue" | "fear" | "plateau" | "comeback" | "biology_gated">;
    };
    problem: string;                              // trainer-facing label
    response: string;                             // the recommendation, owner-facing
    ownerExecutionCheck?: string;                 // question the coach can ask
  }>;
  encouragement: { high: string[]; mid: string[]; low: string[] };   // by success band; AI may paraphrase, must not invent
  biologyExplanation?: string;                    // shown when ENG-030 labels a failure biological
}

interface RoutineSchedule {
  byStage: Record<number, Array<{ id: string; label: string; cadence: string; instructions: string[] }>>;
}

interface Source { title: string; author?: string; year?: number; url?: string; note?: string }
```

## 2. Validation rules (enforced on publish, FS-)

| ID | Rule |
|---|---|
| FS-001 | `stages[].number` MUST be contiguous from 1. |
| FS-002 | Between consecutive stages at most one of distance, duration, distraction target MAY increase, and that D MUST equal `activeD` (PM-010). Resets of other D's are permitted. |
| FS-003 | Every stage MUST have ≥ 1 exercise and a `fallbackExerciseId` that exists in the stage and has `requiresEquipment = []`. |
| FS-004 | Every exercise template MUST have all seven parts non-empty (D-015); `steps` MUST have ≥ 3 entries. |
| FS-005 | Placeholders in templates MUST be from the allowed set; `{repetitions}` requires `params.repetitions`, `{distance}` requires `params.distance`, etc. |
| FS-006 | `measurement.type` MUST equal `skills.measurement_type` for the skill. `cue_progression` exercises MUST define `params.repetitions`. |
| FS-007 | `milestones[].stage` MUST reference an existing stage; `confidenceRequired` in 0–100. |
| FS-008 | `requiresEquipment` and `equipmentOptions` ids MUST exist in the static equipment list. |
| FS-009 | `pingPongSequence` values MUST lie within the active D's `range`. |
| FS-010 | `metadata.routine = true` requires `routine.byStage` for every stage. |
| FS-011 | `version` MUST equal the skill's latest version + 1; `metadata.skillId` MUST match the row. |
| FS-012 | Every `commonProblems[].id` and `exercises[].id` MUST be unique within the document. |
| FS-013 | `sources` MUST be non-empty for status `published` once research-backed versions exist (D-030); v0 (version 1) is exempt. |
| FS-014 | Publishing MUST fail with the list of violated FS- ids; a failed publish leaves the version in `draft`. |

## 3. Versioning and locking semantics

| ID | Rule |
|---|---|
| FS-020 | A published version is immutable. Corrections create a new draft from the published body. |
| FS-021 | A dog is locked to the version in `skill_progress.framework_version_id` from activation (D-024). Publishing a new version does not touch locked dogs (ENG-014). |
| FS-022 | Exercise and problem ids SHOULD be kept stable across versions so analytics can compare. Renamed ids are allowed; the admin diff highlights them. |
| FS-023 | Phase 2 MAY add an opt-in migration: a version declares `stageMap` from the previous version and locked dogs can be moved with their stage translated. Not in MVP; the schema reserves the key. |
| FS-024 | Retiring a version is allowed only when no `skill_progress` row references it, or as a soft retire that keeps it readable. |

## 4. Admin editor (D-043)

The MVP admin panel edits the JSON document directly with: schema-driven inline validation (FS- rules as you type), a rendered preview (stage cards with D targets and advance criteria, exercise cards with the seven template parts and placeholders resolved for a sample dog), a diff against the currently published version, and Publish / Save draft actions. Form-based editing is Phase 2 (D-024).

## 5. Example (recall, excerpt)

```json
{
  "schemaVersion": 1,
  "metadata": {
    "skillId": "recall", "version": 1, "name": "Recall (Come)",
    "summary": "Teach {dog} to come immediately when called, starting close and quiet and building distance and distraction one step at a time.",
    "category": "foundation", "safetyWeight": 1.0, "windowSensitivity": 0, "routine": false,
    "minimumAgeWeeks": 8, "prerequisites": [], "equipmentOptions": ["treats", "treat_pouch", "long_line"]
  },
  "measurement": { "type": "cue_progression", "tier1": { "prompt": "Out of {n} tries, how many times did {dog} come?" } },
  "stages": [
    {
      "number": 1, "name": "Close indoor", "description": "Charging the recall cue at 5 ft with zero distractions.",
      "learningPhase": "acquisition", "environmentClass": "indoor_quiet", "targetEnvironments": 1,
      "dTargets": { "distance": { "value": 5, "unit": "ft", "range": [3, 6] }, "distraction": { "level": 0, "range": [0, 0] } },
      "advanceWhen": "8 of 10 on the first cue across 3 sessions",
      "exercises": [
        {
          "id": "recall_s1_charge", "name": "Charge the cue", "kind": "drill", "difficulty": 1, "novelVariant": false,
          "requiresEquipment": [],
          "params": { "repetitions": 10, "distance": { "value": 5, "unit": "ft" }, "distraction": 0, "environment": "indoor_quiet" },
          "estimatedMinutes": 4,
          "template": {
            "goal": "{dog} learns that the recall word predicts something wonderful.",
            "setup": ["A quiet room", "Ten small, high-value treats in your pouch", "{dog} awake and not just fed"],
            "steps": ["Stand about {distance} from {dog}.", "Say {dog}'s name once, then your recall word once, in a happy voice.", "The moment {dog} reaches you, mark and give the treat right at your body.", "Release and let {dog} wander.", "Repeat {repetitions} times.", "Finish with one easy repetition from 2 ft so the session ends on a win."],
            "successCriteria": "{dog} arrives within about 3 seconds on the first cue in 8 of {repetitions} tries.",
            "commonMistakes": ["Repeating the cue", "Rewarding before {dog} reaches you", "Stepping toward {dog}"],
            "troubleshooting": ["Halve the distance", "Use a higher-value treat", "End early on a success"]
          }
        }
      ],
      "fallbackExerciseId": "recall_s1_charge"
    }
  ],
  "milestones": [{ "id": "reliable_indoor_recall", "name": "Reliable indoor recall", "stage": 3, "confidenceRequired": 75, "celebrationCopy": "{dog} comes when called indoors, every time it matters." }],
  "coaching": {
    "stageIntro": { "1": "We start close and quiet so the word always wins." },
    "commonProblems": [
      { "id": "distracted", "match": { "distraction": ["noise", "people", "dogs", "toys", "unknown"] }, "problem": "Dog distracted", "response": "Tomorrow we reduce distractions and shorten the distance so the cue wins again.", "ownerExecutionCheck": "Was anything else going on in the room when you called?" },
      { "id": "slow", "match": { "successBelow": 0.5 }, "problem": "Slow or hesitant response", "response": "Raise the reward value and call from closer; run backward as {dog} approaches to add excitement." }
    ],
    "encouragement": { "high": ["{dog} is nailing this."], "mid": ["Solid work; consistency is what builds reliability."], "low": ["Tough session, and completely normal at this stage."] }
  },
  "sources": []
}
```

---
**Why**: The chat's architect insisted the framework be stored as structured data the engine can reason about, listed the seven sections (metadata, stages, milestones, exercises, advancement rules, regression rules, coaching guidance), and showed that a trainer's "Stage 3 is too hard" should be a one-field data change. Session 2 turned that into a typed document with the parts later sessions had added: the measurement block that defines the Tier 1 question (D-033), the `activeD` and ping-pong fields the progression rules need, the fallback exercise so equipment never blocks training, the match conditions that let coaching guidance be selected by signal instead of by prose search, and publish-time validation so a bad framework cannot reach a dog.
