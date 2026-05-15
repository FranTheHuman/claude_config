# /akka:diagnose — Diagnose an Akka Error or Unexpected Behavior

Analyze an error, exception, or unexpected behavior in this Akka application.
Identify the root cause, explain why it happened, and provide a concrete fix.

## Instructions

1. Read `.claude/skills/akka-sdk/SKILL.md` before diagnosing.
2. If no stacktrace or error message is provided, ask for it before proceeding.
3. Read the files mentioned in the stacktrace or provided by the user.
4. Classify the root cause into one of the categories below.
5. Output the diagnosis in the structure defined below.

## Root Cause Categories

| Category | Typical Symptoms |
|----------|-----------------|
| **Agent calling Agent** | Timeout, nested session confusion, broken resilience |
| **Side effect in applyEvent** | Non-deterministic state, test failures on replay |
| **Missing @TypeName** | `ClassNotFoundException` or deserialization failure after rename |
| **Changed @Component id** | Empty entity state in production, view needs full rebuild |
| **Step timeout too short** | `TimeoutException` on LLM calls, workflow stuck |
| **No @Acl on endpoint** | HTTP 403 for all callers including legitimate ones |
| **Domain importing Akka** | Circular dependencies, test isolation broken |
| **Missing error forwarding** | Unhandled exceptions crash the actor, no client response |
| **applyEvent blocking** | Non-deterministic replay, intermittent state corruption |
| **Hotspot entity** | Single entity serializing all writes, throughput bottleneck |

## Output Structure

### Diagnosis

**Root cause category**: [category from table above]

**What happened**: Clear explanation of the failure mechanism tied to the Akka
runtime model — not just "the session was null" but WHY in the context of
actors, persistence, and routing.

**Affected code**:
```java
// The problematic snippet
```

### Fix

```java
// The corrected code, complete and compilable
```

**Why this fixes it**: One paragraph tying the fix to the Akka execution model.

### Prevention

The rule or pattern to apply going forward. Reference the relevant skill section.

## Arguments

Usage: `/akka:diagnose [error description or paste stacktrace directly]`

Examples:
- `/akka:diagnose` → agent asks for the error
- `/akka:diagnose TimeoutException in PaymentWorkflow step charge`
- `/akka:diagnose entity state is empty after renaming the class`
- `/akka:diagnose HTTP 403 on all requests to /api/orders`
- `/akka:diagnose workflow step retries indefinitely without failing over`
- `/akka:diagnose src/main/java/com/example/application/OrderAgent.java`
