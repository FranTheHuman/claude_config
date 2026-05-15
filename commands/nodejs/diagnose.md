# /nodejs:diagnose — Diagnose a Node.js Error or Unexpected Behavior

Analyze an error, exception, or unexpected behavior in this Node.js application.
Identify the root cause, explain why it happened, and provide a concrete fix.

## Instructions

1. Read `.claude/skills/nodejs-best-practices/SKILL.md` before diagnosing.
2. If no error message or stacktrace is provided, ask for it before proceeding.
3. Read the files referenced in the error or provided by the user.
4. Classify the root cause into one of the categories below.
5. Output the diagnosis in the structure defined below.

## Root Cause Categories

| Category | Typical Symptoms |
|----------|-----------------|
| **Blocked event loop** | All requests slow, CPU at 100%, timeouts across the board |
| **Unhandled promise rejection** | Silent failures, partial data returned, crash in Node 15+ |
| **Missing await** | Race condition, undefined returned, stale data |
| **Layer violation** | DB query in controller, business logic in route handler |
| **Missing error forwarding** | Errors swallowed, client never gets a response |
| **Memory leak** | RSS growing over time, OOM kill, GC pressure |
| **Missing validation** | 500 errors from invalid input reaching business logic |
| **Hardcoded secret** | Credential exposure in logs or Git history |
| **Sync I/O in handler** | Occasional freezes under load, slow p99 latency |
| **Missing try/catch on async** | UnhandledPromiseRejectionWarning, server crash |

## Output Structure

### Diagnosis

**Root cause category**: [category from table above]

**What happened**: Clear explanation of the failure mechanism — not just "the DB query
failed" but WHY in terms of the Node.js event loop, async model, or layer contract.

**Affected code**:
```typescript
// The problematic snippet
```

### Fix

```typescript
// The corrected code, complete with imports
```

**Why this fixes it**: One paragraph explaining the correction in terms of the
Node.js execution model or architecture principle.

### Prevention

The ESLint rule, architecture rule, or coding habit to apply going forward.
Reference the relevant section of `.claude/skills/nodejs-best-practices/SKILL.md`.

## Arguments

Usage: `/nodejs:diagnose [error description or paste stacktrace directly]`

Examples:
- `/nodejs:diagnose` → agent asks for the error
- `/nodejs:diagnose UnhandledPromiseRejectionWarning in orders controller`
- `/nodejs:diagnose all requests are timing out after deploying to production`
- `/nodejs:diagnose TypeError: Cannot read property 'id' of undefined`
- `/nodejs:diagnose memory usage keeps growing and never drops`
- `/nodejs:diagnose endpoint returns 200 but data is not saved to database`
- `/nodejs:diagnose src/orders/orders.controller.ts`
