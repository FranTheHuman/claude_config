# /fe:review — Frontend Architecture and UX/UI Review

Review frontend architecture decisions, UX flows, UI components, or design systems.
Apply all rules from `.claude/skills/fe-architecture/SKILL.md`.
Output issues with severity labels and concrete fixes.

## Instructions

1. Read `.claude/skills/fe-architecture/SKILL.md` before starting.
2. Read the target file(s), page description, or design spec provided.
3. Check immediately for hard rule violations:
   - CSR used for SEO-critical pages → CRITICAL
   - Generic CTA copy ("Submit", "Click here") → CRITICAL
   - `<div onClick>` without role/keyboard handler → CRITICAL
   - Trust signals only in footer → CRITICAL
   - No `<label>` on form inputs → CRITICAL
4. Apply Nielsen's 10 heuristics as the UX review framework.
5. Apply Core Web Vitals targets as the performance review framework.
6. Output the review in the structure below.

## Output Structure

### Summary
One sentence describing overall quality and the single most impactful issue.

### Issues

Group by severity. For each issue:

```
[SEVERITY] Short title
Location: component/page/section name
Problem:  What is wrong and why it hurts users or conversions.
Fix:      Concrete corrective action or code snippet.
```

Severity levels:
- `[CRITICAL]` — directly hurts conversions, blocks users, accessibility failure,
  or wrong rendering strategy for the surface type
- `[WARNING]`  — friction point, performance issue, missed trust opportunity
- `[SUGGESTION]` — improvement to clarity, consistency, or delight

### Verdict
- ✅ Approved
- ⚠️ Approved with warnings
- ❌ Blocked — critical issues must be resolved

## Arguments

Usage: `/fe:review [path or description]`

Examples:
- `/fe:review` → review whatever context is active
- `/fe:review src/features/checkout/CheckoutPage.tsx`
- `/fe:review src/features/landing/` → review entire landing feature folder
- `/fe:review the homepage hero section and its CTA`
- `/fe:review our rendering strategy for the product catalog`
