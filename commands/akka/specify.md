# /akka:specify — Specify a New Feature (SDD)

Start the Spec-Driven Development cycle for a new feature.
Produces a formal specification from natural language — no code yet.

## Instructions

1. Read `.claude/skills/akka-sdk/SKILL.md` section 13 (SDD) before proceeding.
2. Parse the arguments: feature name + feature description.
3. The spec must describe WHAT and WHY only — zero technical implementation details.
4. If the user included technical details (component names, entity types, endpoint paths),
   flag them and ask whether they belong in the spec or the plan.
5. Create the spec file at `specs/<NNN>-<feature-name>/spec.md`.
6. Include: user stories with priority, functional requirements, success criteria,
   edge cases, and assumptions made.
7. After generating, summarize and confirm with the user before moving to clarify.

## Output Structure

```
Branch:    NNN-<feature-name>
Spec file: specs/NNN-<feature-name>/spec.md

User Stories (P1/P2):
- US1 (P1): ...
- US2 (P2): ...

Functional Requirements: N items
Edge Cases: N items
Assumptions: N items

Next step: /akka:clarify to fill gaps before planning.
```

## Arguments

Usage: `/akka:specify <feature-name> - <what and why description>`

Examples:
- `/akka:specify payment-processing - Users can pay for orders using credit card.
   Payments must be idempotent. Failed payments retry automatically up to 3 times.`
- `/akka:specify user-notifications - The system must notify users about order status
   changes in real time. Users can opt out of specific notification types.`

## Rules

- Spec prompt = WHAT the system does + WHY it matters. Never HOW it works.
- If a requirement implies a specific component (e.g. "must be durable"), document
  the requirement, not the implementation choice.
- Keep each user story small enough to be implemented and verified independently.
