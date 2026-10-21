# Risk Register

Status: written (Session 0). Review at the end of every planning session and every sprint.

Likelihood and impact: L / M / H. Owner is a role, not a person, until the team exists.

| ID | Risk | L | I | Mitigation | Owner |
|---|---|---|---|---|---|
| R-01 | **User compliance**: owners stop logging feedback, adaptation deteriorates, the core hypothesis cannot be validated. | H | H | Tier 1 feedback under 30 seconds (D-012). Coaching response immediately rewards logging. Track feedback-per-completed-session as a top KPI. Agenda still regenerates from completion data alone when notes are absent. | Product |
| R-02 | **Framework quality**: the engine is only as good as the stage definitions, thresholds, and coaching guidance. | M | H | Deep cited research per skill (D-030). Trainer review as a launch gate. Frameworks versioned so fixes ship without code (D-023, D-024). | Product |
| R-03 | **Safety**: a user with an excluded case (aggression, resource guarding, bite history) gets training advice instead of escalation. | M | H | Intake screening questions, hard exclusion in the engine, escalation UX, disclaimer copy (`../product/safety-escalation.md`, Session 5). AI prompts forbid advice on excluded cases. | Product + Eng |
| R-04 | **Scope creep**: building a 12-month product instead of a 3-month MVP. | H | M | MVP test: "if we removed this, would the core loop still work?" Stories reference requirement IDs; anything without an ID is out. | Product |
| R-05 | **LLM dependence**: outages, cost spikes, or quality drift in the AI layer. | M | M | AI is near the end of the pipeline (D-020). Deterministic fallbacks: template-rendered exercise text, canned coaching responses keyed by reason object. Per-user daily token budget. | Eng |
| R-06 | **Weak signal**: a 1–5 rating is too coarse to drive stage thresholds that were designed around success-rate percentages. | M | H | **Mitigated (D-033)**: Tier 1 asks a framework-defined question that yields a success proportion (successes out of N for cue skills, counts for management tracks). Residual risk: owners mis-count; banded data is down-weighted. | Product + Eng |
| R-07 | **Owner-execution errors masquerade as dog failure**, causing wrong adaptation (simplifying when the fix is technique coaching). | M | M | Owner skill tier as prior (D-018); coaching responses route to technique feedback for lower tiers. | Product |
| R-08 | **Trademark conflict** on "Biscuit" (Biscuit Pet Care, UK). | L | M | US trademark search before launch (D-029). Keep brand assets name-swappable until cleared. | Founder |
| R-09 | **Freemium gating too generous or too stingy**: either no conversion or no trial value. | M | M | Define gating around the loop, not around content; instrument conversion funnel from day one (`../product/freemium-model.md`). | Product |
| R-10 | **PWA limitations**: iOS web push and install friction reduce daily-return rates versus native. | M | M | Web push is supported on iOS 16.4+ for installed PWAs. Make install prompt part of onboarding. Revisit native after validation (D-026). | Eng |
| R-11 | **Biology misread**: engine treats a physical limit (bladder, fear period, adolescence) as a training failure. | M | M | Age and biology gates in the progression engine; potty stages account for bladder maturity; fear-period and adolescence signals freeze advancement. | Product |
| R-12 | **Single founder bandwidth**: planning, research, and review all route through one person. | H | M | Agent-automated workflow (D-032); docs structured so the coding agent needs no product clarification. | Founder |

---
**Why**: R-01 and R-06 are the two risks that can invalidate the product rather than merely delay it. R-06 was not identified in the original planning; it surfaced when reconciling the feedback model (a 1–5 rating) with the progression framework (80% success over 3 sessions), which measure different things.
