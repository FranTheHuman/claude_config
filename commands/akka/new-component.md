# /akka:new-component — Generate an Akka SDK Component

Generate a complete, production-ready Akka SDK component with its test class.
Apply all rules from `.claude/skills/akka-sdk/SKILL.md`.

## Instructions

1. Read `.claude/skills/akka-sdk/SKILL.md` before generating any code.
2. Parse the arguments: component type, name, package, and any domain context.
3. Scan `src/main/java/` for existing package structure and naming conventions.
4. Scan `src/main/resources/application.conf` for datasource or model configuration.
5. Generate the component in the correct package layer (application/ or api/).
6. Generate the corresponding test class using the correct TestKit.

## Output Files

For each component type:

**Agent** (`MyAgent.java` + `MyAgentTest.java`)
- `@Component(id = ...)` with stable id
- System message as constant
- Command handler returning `Effect<T>` or `StreamEffect`
- Memory and model configuration if specified
- Function tools if specified
- `TestKitSupport` test with `TestModelProvider`

**Workflow** (`MyWorkflow.java` + `MyWorkflowTest.java`)
- `@Component(id = ...)`, extends `Workflow<State>`
- State record with transition helpers
- `WorkflowSettings` with step timeouts ≥ 30s for agent steps
- `defaultStepRecovery` configured
- At least one public command handler and one private step
- Integration test with `componentClient`

**Event Sourced Entity** (`MyEntity.java` + `MyEntityTest.java`)
- `@Component(id = ...)`, extends `EventSourcedEntity<State, Event>`
- Sealed interface for events with `@TypeName` on every subtype
- `emptyState()` override
- Pure `applyEvent()` — no side effects
- `EventSourcedEntityTestKit` unit test

**Key Value Entity** (`MyEntity.java` + `MyEntityTest.java`)
- `@Component(id = ...)`, extends `KeyValueEntity<State>`
- `emptyState()` override
- `ReadOnlyEffect` for reads, `Effect` for writes
- `KeyValueEntityTestKit` unit test

**View** (`MyView.java`)
- `@Component(id = ...)`, stable id with comment warning
- `TableUpdater` with correct `@Consume.*` annotation
- `@TypeName` reminder if consuming an ESE
- `@Query` methods with correct return types

**HTTP Endpoint** (`MyEndpoint.java` + integration test)
- `@HttpEndpoint`, `@Acl` always present
- `ComponentClient` constructor injection
- Domain → API type mapping
- `HttpException` for error responses
- `supertest`-style integration test

## Arguments

Usage: `/akka:new-component <type> <Name> [--package com.example] [--session <strategy>]
                            [--tools <tool1,tool2>] [--steps <step1,step2>]`

Types: `agent` | `workflow` | `ese` | `kve` | `view` | `endpoint`

Examples:
- `/akka:new-component agent RecommendationAgent --tools inventory,pricing`
- `/akka:new-component workflow PaymentWorkflow --steps charge,notify,compensate`
- `/akka:new-component ese OrderEntity --package com.example.orders`
- `/akka:new-component endpoint OrderEndpoint --package com.example.api`
- `/akka:new-component view OrdersByStatusView`
