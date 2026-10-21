# Personas

Status: written (Session 0). Decisions referenced: D-002, D-018.

Personas are used to test every flow and every piece of copy. Each persona maps to an owner skill tier used by the engine (`priority-engine.md` §3).

## P1 — Sarah, first-time puppy owner (primary)

- 29, apartment, works hybrid. Just adopted Bailey, an 11-week-old Labrador. Never owned a dog.
- Woke up thinking "the puppy peed on the carpet again" and "I have no idea what I should be teaching him."
- Has read three contradictory articles and one Reddit thread. Overwhelmed. Highly motivated. Will spend 20 minutes a day if told exactly what to do.
- Owner tier: **first-time**. Needs heavy technique coaching, small stage steps, 1–2 skills at a time, clear "you are doing this right" signals.
- Wants to feel: "this app gets my dog." Converts to paid when she can see tomorrow changing based on today.
- Biggest risk of abandoning: a day with no visible progress and no explanation.

## P2 — Marcus, first-time adult-dog adopter (primary)

- 41, house with a yard, two kids (7 and 10). Adopted Pepper, a 2-year-old shepherd mix from a shelter three weeks ago.
- Pulls on leash, jumps on visitors, barks at the window. No bite history, no aggression; just untrained and excited.
- Owner tier: **first-time**. Age gates mostly do not apply (adult dog); household-consistency coaching matters (kids must follow the same jumping protocol).
- Wants structure and fast wins on the two behaviors that embarrass him. Likely to graduate after Foundations.

## P3 — Priya, second dog, some experience (secondary)

- 35, suburban, had a dog growing up and took one puppy class. New puppy Milo, 14 weeks, plus an older dog.
- Knows what a marker is. Struggles with proofing and generalization ("he does it in the kitchen and nowhere else").
- Owner tier: **intermediate**. Standard thresholds, 2–3 concurrent skills, coaching as needed.
- Wants efficiency and the 3 D's made explicit. Will stay long term for tricks and advanced obedience if the product keeps offering next steps.

## P4 — Dan, experienced owner (secondary, lower priority)

- 52, rural, has trained multiple dogs, comfortable with shaping and distraction work. New working-breed puppy.
- Owner tier: **experienced**. Larger steps, 3–4 skills, minimal hand-holding. Will be annoyed by over-explanation.
- Validates that coaching depth scales down. Not the design center of MVP copy.

## Anti-persona — Excluded cases (must be routed, not served)

- Owner whose dog has bitten a person, guards food or toys with growling and snapping, or shows aggression to dogs or people.
- Owner whose dog self-injures or destroys property when alone (clinical separation anxiety).
- The screening flow (`safety-escalation.md`) MUST identify these at intake and show professional-help guidance. In-scope goals MAY continue; the excluded concern is never trained.

---
**Why**: Discovery concluded the purchase trigger is emotional ("I just got my first puppy and have no idea what I'm doing"), so the two primary personas are both first-time owners and differ mainly by dog age, which drives different engine behavior (biology gates and socialization windows for Sarah, none for Marcus). P3 and P4 exist to make sure the owner-tier modifiers are exercised, not to shape MVP marketing.
