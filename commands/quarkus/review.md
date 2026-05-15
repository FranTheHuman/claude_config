# /review — Quarkus Code Review

Review the Quarkus code at the specified path (or the current file if no path is given).
Apply the rules from `.claude/skills/quarkus-reactive/SKILL.md` strictly.

## Instructions

1. Read `.claude/skills/quarkus-reactive/SKILL.md` before starting.
2. Read the target file(s). If no path is given, ask the user which file to review.
3. Scan imports first — wrong imports (ORM vs Reactive) are an instant CRITICAL.
4. Check every method for thread-safety and transaction correctness.
5. Output the review in the structure below.

## Output Structure

### Summary
One sentence describing the overall quality and main concern.

### Issues

Group by severity. For each issue:

```
[SEVERITY] Short title
Location: ClassName.java:lineNumber (or method name)
Problem:  What is wrong and why it matters.
Fix:      Corrected code snippet.
```

Severity levels:
- `[CRITICAL]` — will cause runtime failures, data loss, or thread starvation
- `[WARNING]`  — will cause performance degradation or subtle bugs under load
- `[SUGGESTION]` — style, readability, or idiomatic Quarkus improvements

### Verdict
- ✅ Approved — no critical or warning issues
- ⚠️ Approved with warnings — warnings present, no criticals
- ❌ Blocked — one or more critical issues must be fixed before merge

## Arguments

Usage: `/review [path/to/File.java]`

Examples:
- `/review` → reviews the file currently open in context
- `/review src/main/java/org/acme/BookResource.java`
- `/review src/main/java/org/acme/` → reviews all Java files in the package
