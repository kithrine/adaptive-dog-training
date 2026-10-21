# Engine Configuration

Status: written (Session 1). Owner: engineering. Decisions: D-017, D-034. Rule IDs: `CFG-`.

Single source of truth for every tunable number the adaptive engine uses. `adaptive-engine-spec.md` references keys here by name; frameworks may override a subset per skill (`framework-schema.md`). If a number appears in code, it is a bug.

## 1. Storage and loading

| ID | Rule |
|---|---|
| CFG-001 | Configuration is stored as a versioned JSON document in the database (`engine_config` table: id, version, json, status draft/published, createdAt, createdBy). Exactly one version is `published` at a time. |
| CFG-002 | Every engine run records the config version it used in the agenda's metadata, so any decision can be reproduced. |
| CFG-003 | Config is validated against the schema below (Zod) on publish and on load; invalid config MUST fail closed (keep the previous published version) and alert. |
| CFG-004 | Local and test environments load config from `config/engine.default.json` in the repo, which MUST equal the initial published version. Fixture tests MAY pass overrides. |
| CFG-005 | Editing config is an admin action (FR-131). Remote adjustment without deploy is satisfied by CFG-001. Per-cohort or per-user overrides are out of MVP scope; the schema reserves `overrides` for Phase 2. |

## 2. Schema (TypeScript shape)

```ts
export interface EngineConfig {
  schemaVersion: 1;
  signals: {
    countedWeight: number;          // weight of precision=counted signals in window mean
    bandedWeight: number;           // weight of precision=banded signals
    fatigueDurationRatio: number;   // actual/planned minutes beyond which fatigue is suspected
    fatigueSuccessMax: number;      // ... only if success below this
    fearDropPoints: number;         // drop (0–100) in a mastered environment that flags fear
    fearDropPointsInPeriod: number; // smaller drop that flags fear during a fear period
    fearFreezeDays: number;
    plateauSessions: number;
    plateauMin: number;             // window success lower bound for plateau band (upper bound = tier threshold)
    newEnvGraceSessions: number;
    healthPersistDays: number;
  };
  progression: {
    windowSessions: number;
    simplifySessions: number;
    simplifyBelow: number;          // success proportion
    advanceConfidence: number;      // 0–100
  };
  confidence: {
    weights: { success: number; consistency: number; latency: number; environments: number; recency: number }; // sum = 1
    consistencyScale: number;
    recencyHorizonDays: number;
    latencyNeutral: number;
  };
  tiers: Record<"first_time" | "intermediate" | "experienced", {
    threshold: number;              // advance threshold (success proportion)
    minSessions: number;
    advancingCap: number;
    dStep: "small" | "standard" | "large";
    coachingDepth: "heavy" | "as_needed" | "minimal";
  }> & {
    promotion: { sessions: number; minSkills: number; meanSuccess: number };
    diagnostic: { simplifies: number; days: number };
  };
  priority: {
    weights: { safety: number; window: number; weakness: number; userGoal: number; decay: number; overload: number };
    recommendedGoalValue: number;
    recentSimplifyBoost: number;
    recentSimplifyDays: number;
    decayHorizonDays: number;
    adolescenceMaintenanceBoost: number;
  };
  windows: {
    socialization: { curve: Array<[ageWeeks: number, multiplier: number]> }; // piecewise linear
    fearPeriods: Array<[startWeeks: number, endWeeks: number]>;
    adolescence: [startWeeks: number, endWeeks: number];
  };
  agenda: {
    maxSessionsPerDay: number;
    sessionMinutes: [min: number, max: number];
    dailyMinutes: [min: number, max: number];
    maxSkillsPerSession: number;
    maintenanceSlots: number;
    maintenanceFrequencyPerWeek: number;
    maintenanceWeeks: number;       // weeks in maintenance before graduated
    newSkillsPerDay: number;
    groupMaxConcurrent: number;
    swapWindowDays: number;
    dailySafetyThreshold: number;   // safetyWeight at or above which a skill appears daily
    lowCapacityMinutes: number;
    capacityReduction: number;      // advancing-cap reduction when below lowCapacityMinutes
    combineUnderMinutes: number;
    repeatWeakestFromSessions: number;
    horizonDays: number;
  };
  comeback: { gapDays: number; minutesFactor: number; regressDays: number };
  entitlement: { freeDailyItems: number };
  overrides?: Record<string, Partial<EngineConfig>>; // reserved, Phase 2
}
```

## 3. Default values (initial published version)

These are the founder's starting defaults (June 2026) adjusted by Session 1 decisions. They are hypotheses to calibrate against real cohorts, not validated constants.

| Key | Default | Source / note |
|---|---|---|
| signals.countedWeight / bandedWeight | 1.0 / 0.8 | banded chips are coarser (D-033) |
| signals.fatigueDurationRatio / fatigueSuccessMax | 1.5 / 0.6 | PM-031 |
| signals.fearDropPoints / fearDropPointsInPeriod | 40 / 25 | PM-032 |
| signals.fearFreezeDays | 3 | |
| signals.plateauSessions / plateauMin | 7 / 0.70 | PM-033 |
| signals.newEnvGraceSessions | 2 | PM-034 |
| signals.healthPersistDays | 3 | PM-035 |
| progression.windowSessions | 3 | PM-020 |
| progression.simplifySessions / simplifyBelow | 2 / 0.50 | PM-022 |
| progression.advanceConfidence | 75 | PE-020 |
| confidence.weights | success 0.41, consistency 0.24, latency 0.00, environments 0.24, recency 0.11 | founder 35/20/15/20/10 with latency zeroed and renormalized (D-034) |
| confidence.consistencyScale | 200 | stddev 0.25 → consistency 50 |
| confidence.recencyHorizonDays | 10 | |
| confidence.latencyNeutral | 70 | unused while weight 0 |
| tiers.first_time | threshold 0.85, minSessions 3, cap 2, small, heavy | PE §3 |
| tiers.intermediate | 0.80, 3, 3, standard, as_needed | |
| tiers.experienced | 0.78, 2, 4, large, minimal | founder "75–80" resolved to 0.78 |
| tiers.promotion | sessions 10, minSkills 2, meanSuccess 0.90 | ENG-051 |
| tiers.diagnostic | simplifies 2, days 14 | ENG-052 |
| priority.weights | safety 30, window 30, weakness 20, userGoal 15, decay 15, overload 12 | founder defaults |
| priority.recommendedGoalValue | 0.2 | |
| priority.recentSimplifyBoost / recentSimplifyDays | 0.3 / 7 | |
| priority.decayHorizonDays | 7 | |
| priority.adolescenceMaintenanceBoost | 0.3 | PM-036 |
| windows.socialization.curve | [[8,0.4],[12,0.6],[14,1.6],[16,0.4],[17,0.05]] | founder piecewise; to be re-grounded in the research session |
| windows.fearPeriods | [[8,11],[26,60]] | first ~8–11 wks; secondary ~6–14 months, re-grounded later |
| windows.adolescence | [17,35] | ~4–8 months |
| agenda.maxSessionsPerDay | 5 | PM-041 |
| agenda.sessionMinutes | [3,5] | PM-040 |
| agenda.dailyMinutes | [15,25] | PM-043 |
| agenda.maxSkillsPerSession | 3 | PM-044 |
| agenda.maintenanceSlots / maintenanceFrequencyPerWeek | 2 / 2 | PM-046, learning phases |
| agenda.maintenanceWeeks | 4 | |
| agenda.newSkillsPerDay | 1 | PM-047 |
| agenda.groupMaxConcurrent | 2 | D-035 |
| agenda.swapWindowDays | 2 | D-037 |
| agenda.dailySafetyThreshold | 0.9 | recall = 1.0 |
| agenda.lowCapacityMinutes / capacityReduction | 10 / 1 | |
| agenda.combineUnderMinutes | 4 | |
| agenda.repeatWeakestFromSessions | 4 | |
| agenda.horizonDays | 3 | D-009 |
| comeback.gapDays / minutesFactor / regressDays | 5 / 0.6 / 14 | D-039 |
| entitlement.freeDailyItems | defined in `../product/freemium-model.md` (Session 5) | |

## 4. Validation rules

| ID | Rule |
|---|---|
| CFG-010 | `confidence.weights` MUST sum to 1 ± 0.001. |
| CFG-011 | Every tier `threshold` MUST be > `progression.simplifyBelow`. |
| CFG-012 | `agenda.sessionMinutes[1] × agenda.maxSessionsPerDay` MUST be ≥ `agenda.dailyMinutes[0]`. |
| CFG-013 | `windows.socialization.curve` MUST be sorted by age and have ≥ 2 points; multipliers ≥ 0. |
| CFG-014 | `comeback.regressDays` MUST be > `comeback.gapDays`. |
| CFG-015 | Publishing a config that fails validation MUST be rejected with the failing rule IDs. |

## 5. Change process

Changes are admin edits that create a new draft version; publishing is a second explicit action. Every published version is immutable. The decision log records config changes that alter product behavior materially (e.g., threshold shifts), referencing the config version.

---
**Why**: The chat's PM said he would "explicitly prohibit the architect from hardcoding these values anywhere in the application; the entire weighting system should live in configuration and eventually become remotely adjustable." This document makes that enforceable: one schema, one defaults table that is also the first published version, and validation that fails closed. Latency is kept in the schema at weight zero so that enabling it later is a config publish, not a release.

## 6. AI configuration block (added Session 3)

The same published config document carries an `ai` block consumed by `ai-orchestration.md`. It is versioned, validated, and published with the rest (CFG-001 to CFG-005).

```ts
interface EngineConfig {
  // ... sections above ...
  ai: {
    models: { questionnaireFollowup: string; exerciseRender: string; coaching: string; noteInterpretation: string }; // "provider/model" gateway strings
    promptVersions: { questionnaireFollowup: number; exerciseRender: number; coaching: number; noteInterpretation: number };
    limits: Record<AiJob, { timeoutMs: number; maxOutputTokens: number }>;
    budgets: Record<"free" | "premium", { perDay: Record<AiJob, number>; usdPerDay: number }>;
    globalDailyUsd: number;
    bannedTerms: string[];
    sampleRate: number;           // 0–1 share of calls kept for quality review
    cacheHitAlertBelow: number;   // 0–1
  };
}
```

| Key | Default | Note |
|---|---|---|
| ai.models.questionnaireFollowup / coaching | `anthropic/claude-sonnet-5-5` | strong tier (D-044) |
| ai.models.exerciseRender / noteInterpretation | `anthropic/claude-haiku-5-5` | fast tier (D-044) |
| ai.promptVersions.* | 1 | |
| ai.limits.questionnaireFollowup | 2500 ms / 300 tokens | |
| ai.limits.exerciseRender | 4000 ms / 800 tokens | |
| ai.limits.coaching | 6000 ms / 400 tokens | streamed; first token target 1.5 s |
| ai.limits.noteInterpretation | 2000 ms / 200 tokens | bounded before the engine run (AI-021) |
| ai.budgets.premium.perDay | coaching 12, exerciseRender 15, noteInterpretation 12, questionnaireFollowup 6 | |
| ai.budgets.premium.usdPerDay | 0.25 | |
| ai.budgets.free.* | set in `../product/freemium-model.md` (Session 5) | D-047 |
| ai.globalDailyUsd | 50 | raise with user count |
| ai.bannedTerms | prong, pinch collar, e-collar, shock, alpha roll, dominance, pack leader, choke chain, scruff | AI-032 |
| ai.sampleRate | 0.05 | |
| ai.cacheHitAlertBelow | 0.5 | |

Validation: `ai.models.*` MUST be non-empty strings; every `ai.limits` entry MUST have `timeoutMs` ≥ 500; `ai.budgets.free.usdPerDay` MUST be ≤ `ai.budgets.premium.usdPerDay` (CFG-016 to CFG-018).

## 7. Entitlement and free-tier defaults (added Session 5)

The `entitlement` block is extended to `{ trialDays: number; graceDays: number; freeDailyItems: number }`.

| Key | Default | Source |
|---|---|---|
| entitlement.trialDays | 14 | D-052 |
| entitlement.graceDays | 7 | FM past_due |
| entitlement.freeDailyItems | 1 | D-052 |
| ai.budgets.free.perDay | coaching 1, exerciseRender 1, noteInterpretation 1, questionnaireFollowup 0 | D-052, D-047 |
| ai.budgets.free.usdPerDay | 0.03 | |

Validation CFG-019: `entitlement.freeDailyItems` ≥ 1 (the free tier always trains something, FM-012).
