# /akka:review — Review Akka SDK Code

Review Akka SDK code at the specified path (or current file if no path given).
Apply all rules from `.claude/skills/akka-sdk/SKILL.md` strictly.

## Instructions

1. Read `.claude/skills/akka-sdk/SKILL.md` before starting.
2. Read the target file(s). If no path is given, ask which file to review.
3. Check architecture layers first — domain importing Akka is an instant CRITICAL.
4. Check for agent-to-agent direct calls — instant CRITICAL.
5. Check every `applyEvent()` for purity — no I/O, no ComponentClient.
6. Check every `@Component` id for stability risk.
7. Output the review in the structure below.

## Output Structure

### Summary
One sentence describing overall quality and the main concern.

### Issues

Group by severity. For each issue:

```
[SEVERITY] Short title
Location: ClassName.java:methodName
Problem:  What is wrong and why it matters for the Akka runtime.
Fix:      Corrected code snippet.
```

Severity levels:
- `[CRITICAL]` — causes runtime failure, data loss, broken resilience, or lost state
- `[WARNING]`  — performance degradation, anti-pattern, or scalability issue
- `[SUGGESTION]` — style, idiomatic SDK usage, or readability improvement

### Verdict
- ✅ Approved
- ⚠️ Approved with warnings
- ❌ Blocked — critical issues must be fixed before deploy

## Arguments

Usage: `/akka:review [path/to/File.java]`

Examples:
- `/akka:review` → reviews the file currently open in context
- `/akka:review src/main/java/com/example/application/OrderWorkflow.java`
- `/akka:review src/main/java/com/example/application/` → reviews all files in package
