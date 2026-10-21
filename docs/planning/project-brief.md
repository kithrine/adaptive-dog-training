# Project Brief — Biscuit

Status: written (Session 0). Owner: product. Decisions referenced: D-001 to D-013, D-026, D-027, D-029.

## 1. Vision

Biscuit is an adaptive dog-training coach that helps owners raise well-behaved dogs through personalized daily guidance, progress tracking, coaching feedback, and continuously evolving recommendations. It is a personal dog trainer in your pocket, not a video library.

**Positioning statement**: an adaptive coaching system built on expert-designed, positive-reinforcement dog training frameworks, that prioritizes and personalizes daily training based on a dog's development stage, goals, progress, and owner feedback.

## 2. Core value proposition

> Raise a happy, well-behaved dog by knowing exactly what to train next and receiving adaptive coaching based on real-world progress.

Homepage-level promises that pass the test: "Tell us about your dog and we'll build your first week of training in under 5 minutes." "Know exactly what to train tomorrow."

## 3. Core loop

```
Assess → Personalized Training Path → Daily Agenda → Training Session
      → Session Feedback → AI Coaching → Adaptive Recommendations → repeat
```

Every subscription product needs a recurring value loop. This is Biscuit's. The product hypothesis (section 8) is that owners will run this loop repeatedly.

## 4. Scope

### 4.1 MVP (in scope)

| Area | Included |
|---|---|
| Accounts | Register, login, password reset, profile |
| Dog profile | Name, breed, age, sex, spay/neuter, training history, household info, owner experience |
| Dynamic assessment | Goal-specific questionnaire; adapts on age, goals, behavior concerns, owner experience, available time |
| Training path | Priorities, focus areas, daily commitment, strategy explanation (the "Meet Your Dog's Training Path" screen) |
| Daily agenda | Next ~3 days of exercises, regenerated from feedback; daily objectives, exercises, session durations |
| Exercises | Full structure per D-015 |
| Feedback | Required completed + rating; optional notes; adaptive AI follow-up |
| Coaching | Immediate AI feedback, encouragement, recommendation, clarification question |
| Adaptation | Future agenda updates automatically from feedback and performance |
| Freemium | Free tier and premium tier per `../product/freemium-model.md` (Session 5) |
| Admin | Simple panel to author/edit versioned frameworks (D-024) |

MVP skills: see D-003.

### 4.2 Out of scope for MVP

Video analysis, photo analysis, community, trainer marketplace, family/multi-user accounts, wearables, multiple dogs per account, progress analytics dashboards, and all excluded behavior cases (D-004). Multiple dogs and analytics are the first candidates for Phase 2.

### 4.3 Roadmap after MVP

See `roadmap.md`. Summary: Phase 2 analytics, weekly summaries, push, milestones, multiple dogs. Phase 3 photo uploads, environment and equipment checks. Phase 4 video uploads, technique coaching, behavioral observation.

## 5. Business model

Freemium subscription (D-005), US only (D-006). Detailed gating and pricing in `../product/freemium-model.md`.

## 6. Platform and stack

Mobile-first installable web app (D-026) on Next.js, Supabase, Vercel, with Claude via the AI SDK (D-027). Details in `../architecture/tech-stack.md`.

## 7. Goals and success metrics

### Business objectives

1. Help owners successfully complete personalized training programs.
2. Increase owner confidence through clear daily guidance and adaptive coaching.
3. Create a scalable alternative to generic training resources.
4. Build a sustainable freemium subscription business.
5. Establish the data foundation for future media-assisted coaching.

### KPIs

| Category | Metric |
|---|---|
| Activation | Account creation rate; questionnaire completion rate; training path generation rate |
| Engagement | DAU, WAU; sessions completed per week; feedback submitted per session (target: Tier 1 on >90% of completed sessions) |
| Outcome | Milestone completion rate; goal achievement rate; user-reported success score |
| Retention | Week 1, Week 4, monthly retention |
| Revenue | Free-to-paid conversion; MRR; subscriber retention |

## 8. Product hypothesis

> Dog owners will perceive adaptive AI coaching and personalized training progression as more valuable than static dog training content, and will consistently engage with the feedback loop to improve training outcomes.

The MVP exists to validate this and nothing else. "People want dog training content" is already proven and is not the hypothesis.

## 9. Constraints and principles

- Expert frameworks are the source of truth; AI never invents methodology (D-007).
- Positive reinforcement only (D-008).
- Deterministic engines, explainable recommendations (D-020, D-022).
- Scope discipline: if removing a feature leaves the core loop intact, it is not MVP.

---
**Why**: The brief exists so that every later document can point here for "what are we building and why" instead of restating it. The scope table is deliberately narrow because the founder and the planning agents both agreed there were enough ideas to accidentally build a 12-month product.
