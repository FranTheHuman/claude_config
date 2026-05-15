# /react:diagnose — Diagnose a React Error or Unexpected Behavior

Analyze an error, unexpected behavior, or performance problem in this React application.
Identify the root cause, explain why it happened in React's rendering model,
and provide a concrete fix.

## Instructions

1. Read `.claude/skills/react-best-practices/SKILL.md` before diagnosing.
2. If no error message or component code is provided, ask for both before proceeding.
3. Read the files referenced in the error or provided by the user.
4. Classify root cause into one of the categories below.
5. Output the diagnosis in the structure below.

## Root Cause Categories

| Category | Typical Symptoms |
|----------|-----------------|
| **Stale closure** | `useEffect` or `useCallback` reading outdated state or props values |
| **Infinite re-render** | `useEffect` triggering state update every render, browser freezes |
| **Missing cleanup** | Memory leak, subscription still active after component unmounts |
| **Hydration mismatch** | Next.js error: "Text content did not match server/client" |
| **State mutation** | Array push/splice on state, UI does not update as expected |
| **Wrong key prop** | List items losing focus, animation glitches, incorrect component reuse |
| **Missing Suspense boundary** | Uncaught error: async component without Suspense fallback |
| **forwardRef in React 19** | Type error or lint warning about deprecated forwardRef usage |
| **Deprecated API** | `ReactDOM.render`, `react-test-renderer`, Recoil, react-helmet |
| **Missing dependency array** | `useEffect` runs on every render, causing loops or excessive fetching |

## Output Structure

### Diagnosis

**Root cause category**: [category from table above]

**What happened**: Clear explanation of the failure mechanism tied to React's
rendering and reconciliation model — not just "the effect ran again" but WHY
in terms of how React schedules and commits updates.

**Affected code**:
```typescript
// The problematic snippet
```

### Fix

```typescript
// The corrected code, complete with imports and types
```

**Why this fixes it**: One paragraph tying the fix to React's rendering model —
reconciliation, fiber scheduling, or hook execution order as relevant.

### Prevention

The ESLint rule, React 19 API, or architectural habit to prevent this class of bug.
Reference the relevant section of `.claude/skills/react-best-practices/SKILL.md`.

## Arguments

Usage: `/react:diagnose [error message or paste component code directly]`

Examples:
- `/react:diagnose` → agent asks for the error
- `/react:diagnose Maximum update depth exceeded in UserList`
- `/react:diagnose hydration mismatch on the product page`
- `/react:diagnose useEffect is firing on every render`
- `/react:diagnose list item loses input focus when I type`
- `/react:diagnose state update not reflected in the UI`
- `/react:diagnose src/features/auth/components/LoginForm.tsx`
