# /nodejs:review — Review Node.js Code

Review Node.js / Express code at the specified path (or current file if no path given).
Apply all rules from `.claude/skills/nodejs-best-practices/SKILL.md` strictly.

## Instructions

1. Read `.claude/skills/nodejs-best-practices/SKILL.md` before starting.
2. Read the target file(s). If no path is given, ask which file to review.
3. Check architecture layers first — DB query in a controller is an instant CRITICAL.
4. Check for `console.log()` in production code — instant WARNING.
5. Check for hardcoded secrets — instant CRITICAL.
6. Check for missing `await` on async calls — instant CRITICAL.
7. Check anti-patterns table from skill section 18.
8. Output the review in the structure below.

## Output Structure

### Summary
One sentence describing overall quality and the main concern.

### Issues

Group by severity. For each issue:

```
[SEVERITY] Short title
Location: filename.ts:lineNumber (or function name)
Problem:  What is wrong and why it matters.
Fix:      Corrected TypeScript code snippet.
```

Severity levels:
- `[CRITICAL]` — security issue, will cause bugs, or production failure
- `[WARNING]`  — performance, bad practice, or maintainability issue
- `[SUGGESTION]` — style, idiomatic Node.js/TypeScript improvement

### Verdict
- ✅ Approved
- ⚠️ Approved with warnings
- ❌ Blocked — critical issues must be fixed before merge

## Arguments

Usage: `/nodejs:review [path/to/file.ts]`

Examples:
- `/nodejs:review` → reviews the file currently open in context
- `/nodejs:review src/users/users.controller.ts`
- `/nodejs:review src/users/` → reviews all files in the feature folder
- `/nodejs:review src/` → reviews the full source tree
