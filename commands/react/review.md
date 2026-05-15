# /react:review — Review React Code

Review React/TypeScript code at the specified path (or current file if no path given).
Apply all rules from `.claude/skills/react-best-practices/SKILL.md` strictly.
Default to React 19 standards.

## Instructions

1. Read `.claude/skills/react-best-practices/SKILL.md` before starting.
2. Read the target file(s). If no path is given, ask which file to review.
3. Check immediately for deprecated APIs — instant CRITICAL:
   - `class extends Component` → functional component
   - `ReactDOM.render()` → `createRoot().render()`
   - `forwardRef()` in React 19 context → plain ref prop
   - `react-test-renderer` imports → `@testing-library/react`
   - Index as list key → stable ID
4. Check all interactive elements for accessibility (keyboard + ARIA).
5. Check state mutations — `state.arr.push()` is always CRITICAL.
6. Check anti-patterns from skill section 13.
7. Output the review in the structure below.

## Output Structure

### Summary
One sentence describing overall quality and the main concern.

### Issues

Group by severity. For each issue:

```
[SEVERITY] Short title
Location: ComponentName.tsx:lineNumber (or function/hook name)
Problem:  What is wrong and why it matters in React's rendering model.
Fix:      Corrected TypeScript code snippet.
```

Severity levels:
- `[CRITICAL]` — deprecated API, state mutation, accessibility failure, or runtime bug
- `[WARNING]`  — performance problem, missing cleanup, stale closure, bad pattern
- `[SUGGESTION]` — style, idiomatic React 19, TypeScript improvement

### Verdict
- ✅ Approved
- ⚠️ Approved with warnings
- ❌ Blocked — critical issues must be fixed before merge

## Arguments

Usage: `/react:review [path/to/Component.tsx]`

Examples:
- `/react:review` → reviews the file currently open in context
- `/react:review src/features/auth/components/LoginForm.tsx`
- `/react:review src/features/users/` → reviews all files in the feature folder
- `/react:review src/` → full source tree review
