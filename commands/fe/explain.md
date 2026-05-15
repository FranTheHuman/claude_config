# /fe:explain — Explain a Frontend Architecture or UX Concept

Explain any frontend architecture pattern, UX principle, performance concept,
or design system topic. Always include real-world examples, a decision table,
and "when NOT to use" guidance.

## Instructions

1. Read `.claude/skills/fe-architecture/SKILL.md` before answering.
2. Parse the argument: pattern, concept, heuristic, or comparison.
3. If it involves a deprecated or legacy approach, flag it and show the modern alternative.
4. Structure the answer as below.
5. Never skip "When NOT to use" — it prevents the most common misapplications.

## Output Structure

### What it is
One clear paragraph. Define terms, no jargon without definition.

### How it works
3–5 bullet points on the mechanics or principles.
For architecture patterns: include a concrete structure diagram.
For UX principles: include a concrete before/after example.

### Real-world example
A recognizable product that applies this well (Netflix, Airbnb, Shopify, etc.)
and what specifically they do.

### Code or implementation example
Concrete snippet showing the concept in practice.

### Common mistakes
2–3 most frequent misapplications.
Format: ❌ Wrong → ✅ Correct

### When to use / When NOT to use
Decision table.

## Arguments

Usage: `/fe:explain <concept>`

Examples:
- `/fe:explain ISR vs SSR`
- `/fe:explain Islands Architecture`
- `/fe:explain atomic design`
- `/fe:explain container queries`
- `/fe:explain micro-frontends`
- `/fe:explain Core Web Vitals`
- `/fe:explain Nielsen heuristics`
- `/fe:explain conversion-focused CTA copy`
- `/fe:explain design tokens`
- `/fe:explain stale-while-revalidate`
- `/fe:explain when to use micro-frontends`
- `/fe:explain error prevention vs error recovery`

## Concepts Fully Covered by the Skill

CSR, SSR, SSG, ISR, Islands Architecture, PWA, Jamstack, Monolith, Micro-frontends,
Atomic Design (atoms/molecules/organisms/templates/pages), Container queries,
Core Web Vitals (LCP/FID/CLS/INP), Design tokens, CSS custom properties, Dark mode,
TanStack Query, Zustand, Client vs server state, Nielsen's 10 heuristics,
WCAG POUR principles, Conversion-focused UX, Above-the-fold formula,
CTA copy patterns, Trust signals, Information architecture, Mobile-first design,
Touch targets, Skeleton screens, Stale-while-revalidate, Module Federation,
Single-SPA, Progressive enhancement, Graceful degradation
