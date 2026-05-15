# /akka:explain — Explain an Akka Concept or Code

Explain an Akka SDK concept, component, pattern, or specific file from this project.
Always include a runnable Java code example and a "when to use / when NOT to use" section.

## Instructions

1. Read `.claude/skills/akka-sdk/SKILL.md` before answering.
2. Parse the argument: concept name, API method, pattern, or file path.
3. If a file path is given, read the file and explain THAT specific code in context.
4. Structure the answer as below.
5. Never skip the "When NOT to use" guidance — it is the most valuable part.

## Output Structure

### What it is
One clear paragraph. No jargon without definition.

### How it works at runtime
3–5 bullet points on what the Akka runtime does under the hood.
Focus on persistence, routing, recovery, and actor behavior.

### Minimal code example
Simplest possible working Java example.

### Realistic code example
A more complete example close to production usage in this project.

### Common mistakes
The 2–3 most frequent errors with this concept.
Format: ❌ Wrong → ✅ Correct

### When to use / When NOT to use
Concise decision table.

## Arguments

Usage: `/akka:explain <concept|path>`

Examples:
- `/akka:explain EventSourcedEntity`
- `/akka:explain Workflow`
- `/akka:explain applyEvent`
- `/akka:explain Agent vs Workflow`
- `/akka:explain session memory`
- `/akka:explain @TypeName`
- `/akka:explain src/main/java/com/example/application/PaymentWorkflow.java`
- `/akka:explain replicated reads vs CRDT`
- `/akka:explain how to avoid a hotspot entity`

## Concepts Fully Covered by the Skill

`Agent`, `Workflow`, `EventSourcedEntity`, `KeyValueEntity`, `View`, `Consumer`,
`HttpEndpoint`, `GrpcEndpoint`, `McpEndpoint`, `ComponentClient`, `Effect`, `StepEffect`,
`applyEvent`, `emptyState`, `@TypeName`, `@Component`, `ReadOnlyEffect`, `WorkflowSettings`,
`RecoverStrategy`, `thenPause`, `thenEnd`, `thenTransitionTo`, `session memory`,
`MemoryProvider`, `AgentRegistry`, `@AgentRole`, `dynamicCall`, `@FunctionTool`,
`@Consume.FromTopic`, `@Produce.ServiceStream`, `replicated reads`, `CRDT`,
`TestModelProvider`, `EventSourcedEntityTestKit`, `KeyValueEntityTestKit`
