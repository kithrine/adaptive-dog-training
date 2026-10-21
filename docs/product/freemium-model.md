# Freemium Model

Status: written (Session 5). Owner: product + founder. Decisions: D-005, D-047, D-052, D-053. Requirement IDs: `FM-`. Config keys: `entitlement.*`, `ai.budgets.free` (`../architecture/engine-config.md`).

## 1. Entitlement states

| State | How entered | Experience |
|---|---|---|
| `trial` | Account creation; `profiles.trial_ends_at = now + entitlement.trialDays` (14). No card. | Full premium experience |
| `premium` | Active or trialing Stripe subscription (`subscriptions.status` in active, trialing) | Full |
| `free` | Trial ended and no active subscription | Limited daily tier (§3) |
| `past_due` | Subscription payment failed | Premium for `entitlement.graceDays` (7), then `free`; banner to update payment |

| ID | Rule |
|---|---|
| FM-001 | Entitlement is computed by one function `getEntitlement(profile, subscription, now)` and read by the engine (`state.entitlement`), the UI, and the AI budget selector. No other code compares dates or statuses. |
| FM-002 | The app-managed trial does not use Stripe trials (no card is collected). Subscribing during the trial starts billing immediately; the remaining trial days are not credited (keeps the rule simple; revisit if conversion data argues otherwise). |
| FM-003 | One trial per account; account deletion and re-creation with the same email within 90 days does not restart it (`profiles.trial_started_at` is retained in the purge-safe `trial_ledger` table keyed by email hash). |

## 2. What premium includes

Everything: multi-skill agenda up to the tier cap, all AI coaching within `ai.budgets.premium`, projected days, Tier 3 follow-ups, milestones, history, weekly summary (P1), all skill categories.

## 3. Free tier (after trial) (D-052)

| Area | Free | Premium |
|---|---|---|
| Today's agenda | Routine strip + **1 numbered session** (the engine's top-priority skill); remaining sessions shown locked (ENG-096, `entitlement.freeDailyItems = 1`) | All sessions |
| Projected days | Hidden | Shown |
| AI coaching | 1 AI coaching response per day (`ai.budgets.free.perDay.coaching = 1`); template coaching after | Full budget |
| Exercise rendering | AI-rendered for the unlocked item (`exerciseRender = 1`); templates otherwise | All |
| Tier 3 follow-ups | Off (`questionnaireFollowup = 0`; note interpretation 1/day so the engine still learns) | On |
| Progress and history | Visible | Visible |
| Goals | Can select; engine still prioritizes across all; only the top one is trained | All |
| Skills in advancement | Effectively 1 | Tier cap |

| ID | Rule |
|---|---|
| FM-010 | Locked items show skill, stage name, and minutes, with the exercise content behind the paywall; the owner sees what they are missing, not a blank. |
| FM-011 | The free user's single daily session is always the engine's top-priority item, so safety-critical skills (recall) still get trained. |
| FM-012 | Progression continues on the trained skill; nothing is "paused" in the free tier, only narrowed. |

## 4. Pricing hypothesis (D-053)

| Plan | Price | Notes |
|---|---|---|
| Monthly | $9.99 | |
| Annual | $59.99 | 50% off monthly; default-selected on the paywall |

Stripe Tax enabled; prices are tax-exclusive; US only. Prices live in Stripe; price ids in env (`tech-stack.md` §3). Changing prices is a Stripe change plus a decision-log entry.

## 5. Paywall moments

| Moment | Trigger | Copy angle |
|---|---|---|
| Trial countdown | Days 10–14 of trial, once per day at most, on the Today screen | "4 days left of full coaching for {dog}" |
| Trial end | First open after `trial_ends_at` | What stays, what narrows, annual price with monthly alternative |
| Locked item tap | Free user taps a locked session | "This is what {dog} would train next" + unlock |
| Coaching cap | Free user submits a second session's feedback | Template coaching shown with a one-line "AI coaching is part of premium" |
| Settings | Always | Manage subscription (Customer Portal) |

| ID | Rule |
|---|---|
| FM-020 | No paywall during days 1–9 of the trial. |
| FM-021 | The paywall never blocks feedback submission or progress viewing; it only gates content and AI depth. |
| FM-022 | All paywall copy is in `../ux/content-and-tone.md`; it never shames the user or the dog. |

## 6. Funnel events (analytics)

`trial_started`, `onboarding_completed`, `first_session_completed`, `first_feedback_submitted`, `paywall_viewed {moment}`, `checkout_started {plan}`, `subscription_activated {plan}`, `trial_ended`, `locked_item_tapped`, `subscription_canceled`. Conversion is measured trial → premium within 30 days of trial end; the brief's KPIs (`../planning/project-brief.md` §7) reference these.

## 7. Config keys

| Key | Default |
|---|---|
| `entitlement.trialDays` | 14 |
| `entitlement.graceDays` | 7 |
| `entitlement.freeDailyItems` | 1 |
| `ai.budgets.free.perDay` | coaching 1, exerciseRender 1, noteInterpretation 1, questionnaireFollowup 0 |
| `ai.budgets.free.usdPerDay` | 0.03 |

---
**Why**: The chat agreed on freemium because the value is ongoing adaptation, and that users must be able to try the loop before paying. Session 5 resolved the shape: a no-card trial lets the free user experience the multi-skill agenda and the coaching voice (the two differentiators), and the post-trial tier keeps the loop alive on one skill so retention and the hypothesis can still be measured. The free tier is narrowed, never paused, because a stalled puppy owner churns and a progressing one converts.
