# /explain — Explain a Quarkus Concept or Code

Explain a Quarkus concept, pattern, API, or specific piece of code from this repository.
Prioritize practical clarity over theoretical depth. Always include runnable code examples.

## Instructions

1. Read `.claude/skills/quarkus-reactive/SKILL.md` before answering.
2. Parse the argument: is it a concept, an API method, a code snippet, or a file path?
3. Structure the explanation as described below.
4. If the argument is a file path, read the file and explain *that specific code* in context.
5. Always end with "When to use / When NOT to use" guidance.

## Output Structure

### What it is
One paragraph. No jargon without definition.

### How it works (the mental model)
The core mechanism in 3-5 bullet points. Focus on what happens at runtime, not just the API surface.

### Code example — minimal
The simplest possible working example. No noise.

### Code example — realistic
A more complete example close to what this codebase would need.

### Common mistakes
The 2-3 most frequent errors developers make with this concept, and how to avoid them.
Format: ❌ Wrong → ✅ Correct

### When to use / When NOT to use
A concise decision table.

## Arguments

Usage: `/explain <concept|path>`

Examples:
- `/explain Uni`
- `/explain @WithTransaction`
- `/explain chain vs transformToUni`
- `/explain @Blocking`
- `/explain src/main/java/org/acme/BookResource.java`
- `/explain the difference between active record and repository pattern`
- `/explain why my Multi is not emitting`

## Concepts Covered by the Skill

The following are fully covered — the agent will answer from the skill without additional lookup:

`Uni`, `Multi`, `@WithTransaction`, `@WithSession`, `@Blocking`, `@NonBlocking`, `PanacheEntity`,
`PanacheRepository`, `PanacheQuery`, `page()`, `chain()`, `map()`, `invoke()`, `call()`,
`onFailure()`, `retry()`, `Uni.combine()`, `EventBus`, `@ConsumeEvent`, `@Incoming`, `@Outgoing`,
`MutinyEmitter`, `@Channel`, `RemoteCache`, `@Remote`, `Stork`, `stork://`, `ReactiveMongoClient`,
`@RunOnVertxContext`, `UniAsserter`, `TransactionalUniAsserter`, `PanacheMock`
