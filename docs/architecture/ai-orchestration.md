# AI Orchestration

Status: written (Session 3). Owner: engineering. Decisions: D-007, D-020, D-021, D-022, D-044 to D-047. Rule IDs: `AI-`. Config keys live in `engine-config.md` §6 (`ai.*`).

The AI layer has exactly four jobs (D-021). It sits at the end of the pipeline, after every decision has been made deterministically (`adaptive-engine-spec.md`). It phrases, personalizes, asks, and interprets. It never decides.

## 1. Principles

| ID | Rule |
|---|---|
| AI-001 | The LLM MUST NOT be the source of truth for any training decision. Inputs to the LLM are reason objects, framework guidance, and user text; outputs are text or structured signals that the engine treats as advisory. |
| AI-002 | Every AI job MUST have a deterministic fallback that produces an acceptable result when the model is unavailable, over budget, refuses, times out, or returns invalid output (FR-063, FR-082). A user is never blocked by AI. |
| AI-003 | Every call MUST be validated against a schema (structured output) and against job-specific post-checks (§5). Invalid output MUST fall back, never be shown. |
| AI-004 | Models, prompt versions, timeouts, and budgets are configuration (`ai.*`), never literals in code. Model identifiers are routed through the AI Gateway as `provider/model` strings (D-027). |
| AI-005 | Prompts MUST NOT contain owner PII beyond a first name, and MUST NOT contain other users' data. Dog name, age, breed, and the current reason object are allowed. |
| AI-006 | Every call is recorded in `ai_calls` (`data-model.md` §3.5) with purpose, model, prompt version, tokens, cost, latency, and status (`ok`, `fallback`, `error`). |

## 2. The four jobs

| Job | Trigger | Model class (config key) | Input | Output | Latency target | Fallback |
|---|---|---|---|---|---|---|
| **Questionnaire follow-up** | Branch answer is ambiguous (FR-033) | strong (`ai.models.questionnaireFollowup`, default `anthropic/claude-sonnet-5-5`) | branch id, structured answers so far, the ambiguity reason | one question + 2–5 chip options, each mapping to a structured field value | < 2.5 s | skip the follow-up; proceed with the structured answers |
| **Exercise rendering** | Agenda generation for today's items (ENG-095); on open for projected days | fast (`ai.models.exerciseRender`, default `anthropic/claude-haiku-5-5`) | exercise template, resolved params, dog name/age, owner tier, stage intro | the same seven-part template structure, rephrased and personalized | pre-rendered; < 3 s when on demand | placeholder substitution into the template (`{dog}` → name, etc.) |
| **Coaching response** | Feedback submitted (FR-080) | strong (`ai.models.coaching`, default `anthropic/claude-sonnet-5-5`) | reason object, Tier 1 result, difficulty tap, note signals, engine-matched coaching guidance (≤ 2 problems), encouragement band, tier coaching depth, Tier 3 trigger | acknowledge, observe, next, optional follow-up question with chips | first token < 1.5 s, streamed | template assembled from the matched problem's `response`, `encouragement[band]`, and the reason object's next action |
| **Note interpretation** | Tier 2 note or Tier 3 free-text answer present | fast (`ai.models.noteInterpretation`, default `anthropic/claude-haiku-5-5`) | raw note, skill, stage, planned environment | `noteSignals` (ENG-023) with per-field confidence | < 2 s, before the engine run | no signals; retry asynchronously for history (§4) |

Model defaults are the current generation at the time of writing; the config decides (D-044). The default model family is Claude (D-027).

## 3. Call contract

Every call goes through one internal function:

```ts
runAiJob<TOut>(job: AiJob, payload: JobPayload, opts): Promise<{ result: TOut; source: "ai" | "fallback"; callId: string }>
```

| ID | Rule |
|---|---|
| AI-010 | The call uses structured output (`output_config.format`) with the job's Zod schema; free-form text is parsed only inside the schema's string fields. Forced tool choice is not used (current models reject it). |
| AI-011 | `max_tokens` per job from `ai.limits[job].maxOutputTokens`; effort `low` for all four jobs (`output_config.effort`), since none benefits from deep reasoning. |
| AI-012 | Timeout from `ai.limits[job].timeoutMs`; one retry on transient errors (429, 5xx, network) within the timeout budget; then fallback. |
| AI-013 | A `refusal` stop reason, a schema failure, a post-check failure, or an exhausted budget MUST all route to the same fallback path and record `status = fallback` with the cause. |
| AI-014 | Prompt structure for caching (§6): stable system layer first, framework-version layer second (its own cache breakpoint), per-call payload last. |
| AI-015 | Prompt text lives in versioned files under `src/ai/prompts/<job>/v<N>.md`; the active version per job is `ai.promptVersions[job]`. `ai_calls.prompt_version` records it. |

## 4. Sequencing on feedback submit

```
client: submit_feedback(item, tier1, difficulty?, notes?)
  → persist feedback row (status completed/partial, success, precision, raw_answer)          sync
  → if notes: note interpretation (timeout ai.limits.noteInterpretation.timeoutMs)              sync, bounded
  → engine run (ENG-010): decisions, agenda regen, Tier 3 trigger, coaching hooks               sync, < 1 s
  → persist agenda, skill_progress, reason objects
  → respond: updated agenda + coaching stream handle
  → coaching response streams to the UI (AI-02x)                                                async stream
  → pre-render tomorrow's items for this dog                                                     async, fire-and-forget
  → if note interpretation timed out: retry once asynchronously; store note_signals for history
```

| ID | Rule |
|---|---|
| AI-020 | The engine run MUST complete and persist before any coaching text is generated (ENG-101). The coaching prompt is built from the persisted reason object. |
| AI-021 | If note interpretation misses its timeout, the engine runs without note signals. The late result updates `note_signals` on the item but does not trigger a second engine run; the next run consumes it. |
| AI-022 | Coaching is streamed token by token into the coaching card (D-046). The UI MUST render the structured parts (acknowledge, observe, next) as they complete, in that order. |
| AI-023 | Pre-rendering for tomorrow happens after the response is sent and MUST NOT delay it. Projected-day items (offsets 1–2) are rendered only when previewed. |

## 5. Guardrails and post-checks

### 5.1 Allowed and forbidden content

| Allowed | Forbidden |
|---|---|
| Rephrase the exercise template in plain, warm language for a first-time owner | Add, remove, or reorder exercise steps; change any parameter (distance, reps, duration, distraction, environment) |
| Explain the reason object ("three sessions at 85% means we add distance tomorrow") | Decide or imply a different action than the reason object states |
| Select from and rephrase the framework's `commonProblems[].response` and `encouragement` | Recommend techniques, tools, or equipment not in the framework; any aversive tool (D-008) |
| Ask one follow-up question from the trigger list (ENG §11) | Diagnose medical or behavioral conditions; give advice on excluded concerns (D-004) |
| Normalize setbacks (fear periods, adolescence, biology gates) using framework copy | Promise outcomes or timelines; mention the confidence score or internal numbers (D-019) |

### 5.2 Post-checks (deterministic, per job)

| ID | Job | Check |
|---|---|---|
| AI-030 | Exercise rendering | Output has exactly the template's number of steps, setup items, mistakes, and troubleshooting items; every resolved parameter value (number + unit) appears verbatim in the output where the template used its placeholder; no banned terms (§5.3). |
| AI-031 | Coaching | `next` mentions the reason object's action (advance/maintain/simplify/freeze/activate) via a required enum field echoed in the schema; the follow-up question, if present, carries the engine's trigger id; no banned terms; no numeric confidence. |
| AI-032 | Note interpretation | All enum fields within the allowed sets; confidence in [0,1]; a `safety` field set when the text mentions biting, aggression, growling at people, or self-injury, which routes to the escalation copy (`../product/safety-escalation.md`) instead of coaching. |
| AI-033 | Questionnaire follow-up | Each option maps to a known field and value in the questionnaire schema; at most 5 options; question under 140 characters. |

### 5.3 Banned terms

Config list `ai.bannedTerms` (initial: prong, pinch collar, e-collar, shock, alpha roll, dominance, pack leader, choke chain, scruff, "show him who's boss"). Any match fails the post-check and falls back; the call is flagged for prompt review.

## 6. Prompt architecture and caching

```
[system L1: global role + guardrails + output contract]          static per prompt version   ← cache breakpoint
[system L2: framework coaching guidance for this skill version]  static per framework version ← cache breakpoint
[user: JSON payload for this call]                               volatile
```

| ID | Rule |
|---|---|
| AI-040 | L1 and L2 MUST contain no per-user or per-call data so the prefix cache hits across users of the same framework version. |
| AI-041 | L2 includes `coaching.stageIntro`, the `commonProblems` entries the engine matched (not the whole list), `encouragement`, and `biologyExplanation`. Because the matched subset varies, L2 is split: the full static guidance block (cached) and a short "matched for this call" block in the user payload. |
| AI-042 | Cache effectiveness is monitored via `cache_read_input_tokens` in `ai_calls`; a sustained hit rate below `ai.cacheHitAlertBelow` alerts. |

## 7. Budgets and cost control

| ID | Rule |
|---|---|
| AI-050 | Per-user daily budgets by entitlement from `ai.budgets[tier]`: a count cap per job and a cost cap in USD. Counters derive from `ai_calls` for the user's local day. |
| AI-051 | Degradation order when a budget is exhausted: exercise rendering → placeholder templates; questionnaire follow-ups → skipped; coaching → template coaching. Note interpretation degrades last because it feeds the engine. |
| AI-052 | Free-tier users receive AI coaching within their budget (D-047); the exact counts are set in `../product/freemium-model.md` and mirrored into `ai.budgets.free`. |
| AI-053 | A global daily spend ceiling (`ai.globalDailyUsd`) switches all jobs to fallback when exceeded and pages the operator (`../ops/observability.md`). |

### Cost model (hypothesis, to be measured)

Per active dog per day with defaults, assuming ~5 exercise renders, ~4 coaching responses, ~2 note interpretations, 0.3 follow-ups:

| Job | Model | Tokens in/out per call | Calls/day | Cost/day |
|---|---|---|---|---|
| Exercise rendering | Haiku 5.5 | 1,500 / 450 | 5 | ≈ $0.002 |
| Coaching | Sonnet 5.5 | 2,000 / 250 | 4 | ≈ $0.026 |
| Note interpretation | Haiku 5.5 | 600 / 120 | 2 | < $0.001 |
| Questionnaire follow-up | Sonnet 5.5 | 1,500 / 150 | 0.3 | ≈ $0.001 |
| **Total** | | | | **≈ $0.03/day, ≈ $0.90/month for a daily user** before caching; coaching dominates |

If the coaching job were moved to the fast model, the total drops below $0.01/day at a quality cost on the one surface the user feels most. Measure with an eval before changing (§9).

## 8. Privacy and safety

| ID | Rule |
|---|---|
| AI-060 | Raw notes are stored under RLS and sent to the model only for interpretation; the model never receives email, full name, or location. |
| AI-061 | Safety routing (AI-032) has priority over coaching: when a safety flag is set, the coaching card shows the escalation copy and the engine marks the concern for the screening flow; no training advice is generated for that note. |
| AI-062 | Transcripts are sampled for quality review (`ai.sampleRate`) and stripped of dog name before review storage. |

## 9. Evaluation

| ID | Rule |
|---|---|
| AI-070 | A golden set of ≥ 50 (reason object, feedback, guidance) cases with expected properties (action echoed, tone band, no banned terms, follow-up only when triggered) MUST exist before launch and run on every prompt-version change. |
| AI-071 | Rendering has a property test: for every published framework, render every exercise with a sample dog and assert AI-030 passes. |
| AI-072 | Fault injection tests: timeout, refusal, malformed output, budget exhausted — each MUST produce the fallback path with `status = fallback` and no user-visible error. |

## 10. Config keys (added to `engine-config.md` §6)

`ai.models.{questionnaireFollowup, exerciseRender, coaching, noteInterpretation}`, `ai.promptVersions.{job}`, `ai.limits.{job}.{timeoutMs, maxOutputTokens}`, `ai.budgets.{free, premium}.{job}.perDay` and `.usdPerDay`, `ai.globalDailyUsd`, `ai.bannedTerms[]`, `ai.sampleRate`, `ai.cacheHitAlertBelow`.

---
**Why**: The chat's architect limited AI to four jobs and said the AI "explains the reason object; it does not invent it." Session 3 added the parts that make that enforceable rather than hoped for: structured output on every call, deterministic post-checks that catch parameter drift and banned techniques, a fallback for every job so the product never blocks on the model, a strict ordering so coaching is always derived from a persisted decision, and budgets as config so the free tier can demonstrate the coaching loop (the hypothesis under test) without unbounded cost. Model tiering (D-044) puts the expensive model only where the user feels it.
