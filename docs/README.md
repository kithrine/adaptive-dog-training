# Biscuit — Planning Documentation

Biscuit is an adaptive dog-training coach: a rules-driven progression engine built on expert-designed, positive-reinforcement frameworks, with an AI layer that explains, personalizes, and coaches. This folder is the single source of truth for what we are building and why. A coding agent should be able to implement the MVP from these documents without asking product questions.

## How to read these docs

1. Start with `planning/project-brief.md` (what and why), then `product/prd.md` (requirements).
2. `planning/decision-log.md` is authoritative. If a doc disagrees with the decision log, the decision log wins and the doc is a bug.
3. Requirement language: **MUST** / **MUST NOT** = non-negotiable; **SHOULD** = default unless a documented reason exists; **MAY** = optional. Each requirement has an ID (`FR-`, `NFR-`, `ENG-`, `UX-`, …) so stories and code can reference it.
4. Every section may end with a **Why** footer. The footer is rationale only. It is never a requirement.

## Index and status

| Folder | Doc | Status |
|---|---|---|
| planning | `project-discovery.md` — problem, market, moat, users | written (Session 0) |
| planning | `project-brief.md` — vision, scope, business model, success metrics | written (Session 0) |
| planning | `decision-log.md` — every product/architecture decision with ID and date | written, append-only |
| planning | `risks.md` — risk register | written (Session 0) |
| planning | `roadmap.md` — phases and launch gates | written (Session 0) |
| product | `prd.md` — product requirements | written (Session 0) |
| product | `personas.md` — primary/secondary users | written (Session 0) |
| product | `progression-methodology.md` — 3 D's, learning phases, adaptation rules, agenda constraints | written (Session 0) |
| product | `priority-engine.md` — priority formula, owner tiers, confidence score, age windows | written (Session 0) |
| product | `feedback-model.md` — three-tier session feedback | written (Session 0) |
| product | `training-frameworks/*.md` — one per skill (v0 founder drafts; deep research session replaces them) | v0 (Session 0) |
| product | `safety-escalation.md`, `freemium-model.md` | planned (Session 5) |
| architecture | `adaptive-engine-spec.md` — deterministic pipeline, rules ENG-, reason objects, Tier 3 triggers | written (Session 1) |
| architecture | `engine-config.md` — every tunable number, schema, defaults, validation | written (Session 1) |
| architecture | `data-model.md` — Supabase tables, RLS, rebuild, seeds | written (Session 2) |
| architecture | `framework-schema.md` — typed framework document, FS- validation, versioning, example | written (Session 2) |
| architecture | `ai-orchestration.md` — four AI jobs, call contract, sequencing, guardrails, budgets, evals | written (Session 3) |
| architecture | `architecture.md`, `tech-stack.md`, `api-spec.md`, `security-and-privacy.md` | planned (Session 4) |
| ux | `ux-spec.md`, `user-flows.md`, `wireframe-spec.md`, `wireframes-layouts.md`, `visual-design-reference.md`, `content-and-tone.md` | planned (Session 6) |
| stories | `epics.md`, `epic-NN-*.md` | planned (Session 7) |
| ops | `deployment.md`, `environments-and-config.md`, `observability.md`, `testing-strategy.md`, `runbook.md` | planned (Session 8) |
| root | `AGENTS.md`, `RULES.md`, `CLAUDE.md` | planned (Session 9) |

## Source material

The planning started in a BMAD-style chat (May–June 2026) and two founder-authored framework documents. Those sources are outside the repo and are superseded by the docs here; they are listed in `planning/project-discovery.md` for provenance only.
