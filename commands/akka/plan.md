# /akka:plan — Create Technical Implementation Plan (SDD)

Produce the technical architecture and implementation plan for the current feature spec.
This is the HOW — component selection, data models, endpoints, and step sequencing.

## Instructions

1. Read `.claude/skills/akka-sdk/SKILL.md` before proceeding.
2. Read the current feature spec at `specs/<current-branch>/spec.md`.
3. Read `constitution.md` if it exists — plan must comply with all rules.
4. Parse the user's technical prompt for architecture decisions.
5. Generate the plan at `specs/<current-branch>/plan.md`.
6. Also generate `research.md` (key decisions and trade-offs) and
   `data-model.md` (domain objects, API records, entity state).

## Plan must include

- **Components**: which Akka components are needed and why (Agent, Workflow, ESE, KVE, View, Endpoint)
- **Data model**: entity state types, API request/response records, domain objects
- **HTTP API contract**: routes, methods, request/response shapes, error codes
- **Step sequencing**: for Workflows, list all steps and transitions
- **Session strategy**: for Agents, define session ID and memory scope
- **Error handling**: recovery strategies, compensation steps, failover transitions
- **Test plan**: which unit tests and integration tests are needed

## Output Structure

```
Branch:  NNN-<feature-name>

Generated Artifacts:
  specs/NNN-.../plan.md
  specs/NNN-.../research.md
  specs/NNN-.../data-model.md
  specs/NNN-.../contracts/http-api.md

Architecture Summary:
  Components: ...
  Key decisions: ...

Constitution Check: PASS / FAIL (list violations)

Next step: /akka:tasks to generate the implementation task list.
```

## Arguments

Usage: `/akka:plan <technical architecture description>`

Examples:
- `/akka:plan Implement as a Workflow with two steps: charge and notify.
   Use an EventSourcedEntity for payment records. Expose a POST /payments endpoint.
   Retry failed charges up to 3 times with exponential backoff.`
- `/akka:plan Single Agent backed by OpenAI GPT-4o. Session per user ID.
   Tools: check inventory, get pricing. HTTP endpoint POST /recommendations.
   Return structured JSON with product list.`
