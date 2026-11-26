# Hermes Bot Fleet — Source of Truth (SoT)

**Status:** Authoritative design baseline. Derived from `hermes-bot-fleet-investigation.md` and official Hermes docs.
**Program:** Bot Fleet Governance, Role Alignment & Group Collaboration
**Mode:** Design / planning only. No implementation, no production writes, no commits/PRs/deployments.

## Document map

| # | Document | Answers | Critical element |
|---|---|---|---|
| 01 | [Context](./01-CONTEXT.md) | Why are we doing this? Who is it for? What is explicitly ignored? | **Non-Goals** — what the agent must ignore |
| 02 | [PRD](./02-PRD.md) | What features/edge cases/UX? How is done proven? | **Binary checklists** — Pass/Fail per requirement |
| 03 | [Architecture](./03-ARCHITECTURE.md) | What stack, data model, file layout, state rules? | **State & Schema** — complete entity relationship rules |
| 04 | [Sequential Plan](./04-SEQUENTIAL-PLAN.md) | In what order, with which gates and rollback? | **Dependencies** — data models before UI/rooms |

## Reading order

1. Context → agree motivation and non-goals.
2. Architecture → agree entities, invariants and minimum configuration.
3. PRD → agree features and binary acceptance.
4. Plan → execute phases in order, each gated by PRD checklists.

## Traceability

Findings (investigation) → Risks (R1–R10) → Functional Requirements (FR-1…FR-10) → Plan Phases (P0…P5) → Binary Tests (T-…). The matrix at the end of `04-SEQUENTIAL-PLAN.md` binds them.

## Sign-off authority

Per the standing lean-review rule, the executing agent is the reviewer; no separate reviewer appointment or independence declaration is required. Phase authorization is recorded in `01-CONTEXT.md` §8 and `04-SEQUENTIAL-PLAN.md` §3.
