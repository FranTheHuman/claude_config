# /fe:ux-audit — UX and Conversion Audit

Audit a page, flow, or feature for UX quality, conversion potential, and usability.
Apply Nielsen's 10 heuristics and conversion-focused UX principles from the skill.
Output actionable findings ordered by business impact.

## Instructions

1. Read `.claude/skills/fe-architecture/SKILL.md` sections 7 (Nielsen) and 8
   (Conversion UX) before auditing.
2. Parse the input: page description, user flow, component code, or URL.
3. Check the 4-element hero formula if this is a conversion surface.
4. Apply all 10 Nielsen heuristics systematically.
5. Check the conversion checklist from the skill.
6. Output findings ordered by business impact (revenue/retention impact first).

## Audit Checklist

### Above the fold
- [ ] Clear value proposition (not a slogan)
- [ ] Supporting line with who it's for
- [ ] Primary CTA with outcome-based copy
- [ ] Trust cue visible without scrolling

### Navigation
- [ ] Fewer than 7 top-level menu items
- [ ] Primary CTA in header
- [ ] Labels are descriptive, not vague
- [ ] Consistent across all pages

### Trust signals
- [ ] Client logos near CTAs (not just footer)
- [ ] Specific social proof with numbers and context
- [ ] Security/compliance badges (only accurate ones)
- [ ] Clear policies (refund, delivery, cancellation)

### Forms
- [ ] Labels on all inputs (not placeholder-only)
- [ ] Error messages are specific and instructive
- [ ] Minimum required fields only
- [ ] Success state clearly communicated

### Mobile
- [ ] Tap targets ≥ 44×44px
- [ ] Primary CTA reachable with one thumb
- [ ] Forms have fewer fields than desktop
- [ ] Load time acceptable on 4G

### Feedback and states
- [ ] Loading states for actions > 1 second
- [ ] Error states for failed operations
- [ ] Empty states for no-data scenarios
- [ ] Success confirmation after key actions

## Output Structure

### Conversion Score
X/10 — one sentence summary of the most critical gap.

### Critical Issues (fix immediately — direct revenue impact)

```
[CRITICAL] Issue title
Where:   Page section or component
Problem: What users experience and why it kills conversions
Fix:     Specific, actionable corrective action
```

### Warnings (fix soon — friction affecting retention)

```
[WARNING] Issue title
Where:   ...
Problem: ...
Fix:     ...
```

### Suggestions (improve when time allows)

### Quick Wins
3 highest-impact changes that can be shipped in under a day.

## Arguments

Usage: `/fe:ux-audit [page description or path]`

Examples:
- `/fe:ux-audit our checkout flow`
- `/fe:ux-audit src/features/landing/HeroSection.tsx`
- `/fe:ux-audit the pricing page — we have a high bounce rate`
- `/fe:ux-audit the sign-up form — completion rate is only 30%`
- `/fe:ux-audit the product detail page for conversion issues`
