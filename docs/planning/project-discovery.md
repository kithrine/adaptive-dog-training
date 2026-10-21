# Project Discovery

Status: written (Session 0). Owner: product. Decisions referenced: D-001 to D-004, D-025, D-029.

## 1. Problem

Dog owners, especially first-time and new-puppy owners, rely on books, YouTube, blogs, and social media. Those sources are generic, contradictory, and disconnected from the individual dog's progress. Owners do not know where to start, what to train next, whether they are making progress, or how to adjust when something stalls. Many abandon training or accidentally reinforce the behaviors they wanted to stop.

The immediate trigger is not "I want a well-behaved dog in two years." It is "my puppy peed on the carpet again," "I have no idea what I should be teaching him," or "I'm overwhelmed."

## 2. Opportunity

An adaptive training system that delivers personalized, day-by-day instruction and continuously adjusts based on the dog's results and the owner's feedback. The product answers one question, every day: **"What should I do with my dog today, and why?"**

## 3. Jobs to be done

| Layer | Job |
|---|---|
| Emotional | Raise a happy, well-behaved dog with confidence. |
| Functional | Know exactly what to train next and how to do it. |

The functional job delivers the emotional one. Every feature is tested against: *does this help the owner know what to do next?* (Questionnaire yes, daily agenda yes, coaching yes, notes yes, progress yes; community maybe; social network no; trainer marketplace not initially.)

## 4. Target users

See `../product/personas.md` for full personas.

- **Primary**: first-time dog owners and new puppy owners seeking structured guidance for foundational training.
- **Secondary**: owners of adolescent or adult dogs with common challenges (leash pulling, recall, barking, jumping, potty setbacks, home-alone confidence).
- **Future**: owners facing complex behavioral challenges that need advanced coaching or professional involvement.

"Owners with obedience issues" is a problem state that occurs inside the first two segments, not a separate segment.

## 5. Why not just ChatGPT?

ChatGPT can write a decent general training plan. It cannot:

1. Run a structured progression system with stage thresholds, biology gates, and age windows.
2. Remember and score every session over months and adapt tomorrow's agenda accordingly.
3. Prioritize across a portfolio of simultaneous skills and cap workload.
4. Coach from a consistent methodology instead of whatever the prompt happens to elicit.

"We do what ChatGPT does, more thoroughly" is an improvement, not a moat. The moat (section 7) is the progression engine plus the data it accumulates.

## 6. Product lifecycle

| Stage | Owner goals | Typical duration |
|---|---|---|
| 1 Foundations | Potty, crate, basic obedience, leash, recall | 1–6 months |
| 2 Reliability | Distraction work, public manners, reliable obedience | 3–12 months |
| 3 Advanced (optional) | Off-leash reliability, advanced obedience, tricks, sports prep | Ongoing |
| 4 Lifetime companion | New goals, refreshers, new dog, senior adaptations, challenges | Ongoing |

Some owners finish Stage 1 and leave. That is a success, not churn to be fought. Others continue for years. The product serves both; retention is measured honestly against the user's own goal.

## 7. Moat

In order of durability:

1. **Progression engine**: trainer-designed skill trees, advancement/regression rules, priority formula, owner model, confidence scoring. Hard to copy because it is expertise encoded as data and tuned on outcomes.
2. **Adaptive feedback loop**: "log how today went, tomorrow changes" creates retention, personalization, and subscription value.
3. **Training session data** (long term): millions of sessions, notes, and outcomes enable learning which protocols work for which dogs and owners. Becomes very hard to copy.
4. **Media-assisted coaching** (Phase 3–4): photo and video analysis of owner technique and dog response. Not MVP (D-025).

The AI layer is explicitly *not* the moat. It is replaceable.

## 8. Biggest risks (summary)

See `risks.md` for the register.

- **User compliance**: the loop dies if owners stop logging feedback. Mitigated by the 30-second feedback design (D-012).
- **Training expertise acquisition**: the app is only as good as its frameworks (D-030).
- **Scope creep**: enough ideas exist to build a 12-month product instead of a 3-month MVP.

## 9. Provenance

This document consolidates the BMAD-style planning chat (May 19 – Jun 23 2026, business analyst and product manager roles) and the founder's two framework documents (`dog_training_progression_framework.html`, `priority_engine_owner_skill_confidence.html`). Those sources are superseded by this folder.

---
**Why**: The discovery phase moved the concept from "AI dog training plans" (easy to copy, weak) to "adaptive coaching built on expert frameworks" (defensible). The key realizations were that the product is a decision engine rather than a plan generator, that owners buy a transformation rather than features, and that the progression system is where the intelligence actually lives.
