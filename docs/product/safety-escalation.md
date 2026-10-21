# Safety Screening and Escalation

Status: written (Session 5). Owner: product + founder. Decisions: D-004, D-008, D-054, D-055. Requirement IDs: `SAF-`. Enforced in code per `../architecture/security-and-privacy.md` §5 and `../architecture/ai-orchestration.md` AI-032.

Biscuit trains in-scope behaviors with positive reinforcement. It does not treat aggression, resource guarding, bite history, clinical separation anxiety, or medically driven behavior. This document defines how those cases are detected, what the app does, and where it sends people.

## 1. Excluded concerns and severity tiers (D-054)

| Concern id | Definition | Tier | Effect on training |
|---|---|---|---|
| `human_aggression` | Has bitten a person (skin broken or not), or growls, snaps, or lunges at people | **A** | All training pauses behind the escalation screen; owner may resume in-scope skills only after explicit acknowledgment (SAF-030) |
| `dog_aggression` | Growls, snaps, or fights with other dogs beyond normal play | B | `socialization` dog-exposure items excluded; leash walking stages with dog distraction gated; escalation card; other skills continue |
| `resource_guarding` | Growls or snaps when someone approaches food, toys, chews, or resting spots | B | Impulse-control component of Puppy Foundations (food bowl, trade games) excluded; escalation card; other skills continue |
| `separation_distress` | Self-injury, destruction of doors or crates, or elimination from distress when alone | B | `home_alone_confidence` excluded; escalation card recommends veterinary behaviorist; other skills continue |
| `severe_fear` | Intense fear in everyday situations (hiding, trembling, refusing to move, not eating) | B | Socialization exposures excluded; `fear` flag set on all skills (ENG-033); escalation card recommends professional help; other skills continue at reduced D |
| `vet_behavioral` | A vet has referred the dog to a behaviorist, or a medical condition or medication affects behavior | B | No exclusion by itself; escalation card advises coordinating with the vet; the `health` flag applies (ENG-036) |

| ID | Rule |
|---|---|
| SAF-001 | Concern ids are stored in `dogs.excluded_concerns`; the exclusions above are implemented as engine state (skills set to `excluded`, components gated) and MUST NOT depend on UI alone. |
| SAF-002 | An excluded skill MUST NOT appear in goal selection, the Training Path, or any agenda (ENG-031). |
| SAF-003 | Concern ids are never shown to the user as labels; the UI uses plain language ("we don't train this in the app"). |

## 2. Intake screening (onboarding)

Asked after the dog profile and before goal selection, in this order, one per screen, yes/no with a short "why we ask" line. All use the dog's name.

| # | Question | Maps to |
|---|---|---|
| S1 | Has {dog} ever bitten a person, even without breaking skin? | `human_aggression` (A) |
| S2 | Does {dog} growl, snap, or lunge at people? | `human_aggression` (A) |
| S3 | Does {dog} growl, snap, or fight with other dogs, beyond normal rough play? | `dog_aggression` (B) |
| S4 | Does {dog} growl or snap when someone comes near food, toys, chews, or a resting spot? | `resource_guarding` (B) |
| S5 | When left alone, does {dog} hurt themselves, destroy doors or crates, or pee or poop from distress? | `separation_distress` (B) |
| S6 | Does {dog} show intense fear in everyday situations, like hiding, trembling, or refusing to move? | `severe_fear` (B) |
| S7 | Has a vet referred {dog} to a behaviorist, or is {dog} on medication that affects behavior? | `vet_behavioral` (B) |

| ID | Rule |
|---|---|
| SAF-010 | Screening MUST be completed before goal selection and MUST be re-askable from settings ("Something changed about {dog}"). |
| SAF-011 | A "yes" to S1 or S2 MUST show the Tier A escalation screen immediately (§4) before anything else. |
| SAF-012 | A "yes" to S3–S7 MUST show the Tier B card at the end of screening, listing in plain language what the app will not train and why, and then continue to goal selection with the affected skills absent. |
| SAF-013 | Screening answers and the shown escalation are recorded in `assessments.screening` and `dog_events` (`escalation_shown`) with timestamps. |
| SAF-014 | Age-appropriate phrasing: for puppies under 16 weeks, S1/S2 include "Puppy mouthing and nipping during play is normal and is not what we mean here" so mouthing is not screened as aggression. |

## 3. Mid-training detection

| ID | Rule |
|---|---|
| SAF-020 | Note interpretation (AI-032) sets `safety` with a concern id when free text describes biting a person, aggression toward people or dogs, guarding with growling or snapping, self-injury when alone, or severe fear. The engine treats it exactly like a screening "yes" for that tier. |
| SAF-021 | When `safety` is set, the coaching response for that session is replaced by the escalation card (AI-061); no training advice is generated for that note. |
| SAF-022 | The owner can correct a false positive ("that's not what I meant") from the card; the correction is recorded, the concern is removed, and training resumes. Two corrections for the same concern within 30 days disable automatic detection for that concern and leave a settings prompt instead. |
| SAF-023 | Exposure checklist answers of `fearful` on three separate exposures within 7 days MUST raise a `severe_fear` check-in (Tier 3 question) rather than an automatic exclusion. |

## 4. Escalation UX

### Tier A screen (full pause)

1. Calm, non-judgmental headline: "This needs more than an app."
2. One paragraph: aggression toward people is outside what Biscuit can safely help with; a certified professional who can see {dog} in person is the right next step; this is common and workable with the right help.
3. Referral block (§5).
4. Two actions: "Find a professional" (primary) and "Continue with the basics anyway" (secondary), which opens the acknowledgment (SAF-030).
5. Training stays paused until acknowledgment.

### Tier B card (skill excluded)

Shown inline (end of screening, or in place of coaching): what is excluded ("We won't work on being home alone in the app"), why in one sentence, the referral block, and "Got it". Other skills continue.

| ID | Rule |
|---|---|
| SAF-030 | Resuming after Tier A requires the owner to tick: "I understand Biscuit does not address aggression and I will seek professional help for it" and to confirm which in-scope goals to continue. Recorded in `dog_events` with the text version. Human-safety exclusions remain. |
| SAF-031 | Escalation copy MUST avoid blame, diagnosis, and breed generalizations. It MUST NOT suggest management techniques for the excluded behavior beyond "keep everyone safe: avoid the situations that trigger it until you have help". |
| SAF-032 | The escalation card is reachable later from the dog's settings so the links are not lost. |

## 5. Referral block (D-055)

Static, no endorsement, three lines with links to the organizations' own directories (verify URLs at launch):

- **Certified trainers**: Certification Council for Professional Dog Trainers (ccpdt.org) — for obedience and most behavior problems.
- **Behavior consultants**: International Association of Animal Behavior Consultants (iaabc.org) — for aggression, guarding, and fear.
- **Veterinary behaviorists**: American College of Veterinary Behaviorists (dacvb.org) — when medication or a medical cause may be involved. Also: talk to your vet first to rule out pain or illness.

| ID | Rule |
|---|---|
| SAF-040 | Referral text MUST state that Biscuit does not vet or endorse individual professionals. |
| SAF-041 | Links open externally; no affiliate tracking in MVP. |

## 6. Disclaimers and consent

| ID | Rule |
|---|---|
| SAF-050 | At onboarding completion, before the Training Path, the user accepts: Biscuit provides general training guidance, is not a veterinarian or behaviorist, and is not a substitute for professional help; stop any exercise if the dog shows fear, pain, or aggression; children must be supervised around dogs during training. Acceptance is stored with the disclaimer version (SEC-032). |
| SAF-051 | Every exercise screen carries a one-line footer: "Stop if {dog} seems scared, in pain, or growls. Ask us or a professional." |
| SAF-052 | Coaching responses MUST never contradict the disclaimers (AI-031 post-check includes the banned-terms list; D-008). |

## 7. Launch gate tests

Personas for the excluded cases (`personas.md` anti-persona) MUST be walked through onboarding and through a mid-training note in staging; each MUST produce the correct tier, exclusion, card, event log, and resume behavior (R-03).

---
**Why**: The chat's analyst said bad advice on "sit" annoys a user and bad advice on aggression gets someone bitten, and the PRD excluded those cases without saying how the app would notice them. Session 5 chose severity tiers because a single "pause everything" rule would make a resource-guarding flag stop potty training and be ignored, while "never pause" leaves person-bite cases running recall drills under a disclaimer. Static directory links are the referral because vetting trainers is a separate business (the Phase 2+ marketplace) and an endorsement is a liability.
