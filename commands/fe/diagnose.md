# /fe:diagnose — Diagnose a Frontend Architecture or UX Problem

Analyze a specific frontend problem: poor performance, low conversion,
accessibility failure, wrong rendering strategy, or architectural mismatch.
Identify the root cause, explain the impact, and provide a concrete fix.

## Instructions

1. Read `.claude/skills/fe-architecture/SKILL.md` before diagnosing.
2. If insufficient context is provided, ask for the specific symptom, metric,
   or user behavior before proceeding.
3. Classify the root cause into one of the categories below.
4. Output the diagnosis in the structure below.

## Root Cause Categories

| Category | Typical Symptoms |
|----------|-----------------|
| **Wrong rendering strategy** | Poor SEO despite content-heavy page; slow LCP on dashboard |
| **Poor LCP** | LCP > 2.5s — hero image not preloaded, unoptimized, or render-blocked |
| **Layout shift (CLS)** | Buttons jump when page loads, images without dimensions |
| **Low conversion** | High bounce, form abandonment, CTA ignored |
| **Accessibility failure** | Users cannot tab through form, screen reader reads nothing useful |
| **State management mismatch** | API data in Redux causing stale/sync issues |
| **Component coupling** | Changing one component breaks unrelated UI |
| **Micro-frontend premature** | Teams blocked on shared dependencies, DX degraded |
| **Design inconsistency** | Different button styles, spacing, or colors across pages |
| **Mobile UX failure** | High mobile bounce, low mobile conversion vs desktop |

## Output Structure

### Diagnosis

**Root cause category**: [from table above]

**What is happening**: Clear explanation of the problem in terms of user impact —
not just the technical description but what users experience and how it affects
business metrics (conversion, retention, SEO ranking, time-on-page).

**Evidence**: What specific signal points to this root cause
(Lighthouse score, heatmap data, conversion metric, user report, code pattern).

### Fix

Concrete corrective action — code, configuration change, or design decision.

**Why this fixes it**: One paragraph connecting the fix to the user outcome,
not just the technical resolution.

### Prevention

The architectural rule, design principle, or monitoring practice that prevents
this problem from recurring. Reference the relevant skill section.

## Arguments

Usage: `/fe:diagnose [description of problem, symptom, or metric]`

Examples:
- `/fe:diagnose our LCP is 4.2 seconds on the product page`
- `/fe:diagnose form completion rate is 28%, users drop at the email field`
- `/fe:diagnose our homepage has a 75% bounce rate on mobile`
- `/fe:diagnose we have CLS score of 0.35 and content shifts on load`
- `/fe:diagnose our micro-frontend setup is causing 3MB bundle on every page`
- `/fe:diagnose the checkout CTA has a 2% click rate despite high traffic`
- `/fe:diagnose accessibility audit found 47 issues, where do we start`
- `/fe:diagnose src/features/catalog/ProductPage.tsx — SSR vs SSG decision`
