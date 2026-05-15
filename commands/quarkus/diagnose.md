# /diagnose — Diagnose a Quarkus Error or Unexpected Behavior

Analyze an error, exception, or unexpected behavior in this Quarkus application.
Identify the root cause category, explain why it happened, and provide a concrete fix.

## Instructions

1. Read `.claude/skills/quarkus-reactive/SKILL.md` before diagnosing.
2. If no stacktrace or error message is provided, ask for it before proceeding.
3. Read the files mentioned in the stacktrace or provided by the user.
4. Classify the root cause into one of the categories below.
5. Output the diagnosis in the structure defined below.

## Root Cause Categories

| Category | Typical Symptoms |
|----------|-----------------|
| **Blocked I/O thread** | `BlockingOperationNotAllowedException`, high latency under load, thread pool exhaustion |
| **Missing session** | `LazyInitializationException`, `No current Mutiny.Session`, session closed errors |
| **Missing transaction** | `TransactionRequiredException`, rollback on commit, data not persisted |
| **Mixed ORM/Reactive APIs** | `ClassCastException` on session, wrong `PanacheEntity` import |
| **Broken reactive chain** | Operation runs but result is ignored, `Uni` created but never subscribed |
| **Nested subscribe** | Callbacks fire out of order, partial results, hard-to-reproduce bugs |
| **Configuration error** | `CDIException`, extension not found, datasource not configured, wrong `db-kind` |
| **XA / multi-datasource** | `Failed to enlist`, connection rollback on second datasource |
| **Native image** | `ClassNotFoundException` in native, reflection errors, missing resources |

## Output Structure

### Diagnosis

**Root cause category**: [category name from table above]

**What happened**: A clear explanation of the failure mechanism — not just "the session was null"
but *why* the session was null at that point in the reactive pipeline.

**Affected code**:
```java
// The problematic snippet, highlighted
```

### Fix

```java
// The corrected code, complete and compilable
```

**Why this fixes it**: One paragraph explaining the correction in terms of the execution model.

### Prevention

The rule or pattern to apply going forward to avoid this class of error. Reference the
relevant section of `.claude/skills/quarkus-reactive/SKILL.md` by number.

## Arguments

Usage: `/diagnose [error description or paste stacktrace directly]`

Examples:
- `/diagnose` → the agent will ask for the error
- `/diagnose No current Mutiny.Session found`
- `/diagnose BlockingOperationNotAllowedException at BookResource.java:42`
- `/diagnose my endpoint returns 200 but nothing is saved to the database`
- `/diagnose src/main/java/org/acme/BookResource.java` → diagnose that file proactively
