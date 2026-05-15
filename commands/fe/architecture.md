# /fe:architecture — Design or Evaluate a Frontend Architecture

Design a complete frontend architecture or evaluate an existing one.
Cover rendering strategy, component structure, state management, data fetching,
performance targets, and reliability patterns.
Apply all rules from `.claude/skills/fe-architecture/SKILL.md`.

## Instructions

1. Read `.claude/skills/fe-architecture/SKILL.md` before starting.
2. Clarify requirements before proposing any solution:
   - Product type (marketing, SaaS, e-commerce, dashboard, content)?
   - Team size and structure?
   - SEO requirements per surface?
   - Performance targets?
   - Accessibility requirements?
   - Real-time or offline needs?
3. Apply the rendering strategy decision table per surface.
4. Identify the application architecture pattern (monolith vs micro-frontends).
5. Define component architecture using atomic design levels.
6. Define state management by state type (client vs server state).
7. Output the architecture in the structure below.

## Output Structure

### Requirements Summary
Functional requirements + non-functional requirements explicitly listed.

### Rendering Strategy (per surface)

| Surface | Strategy | Justification |
|---------|---------|--------------|
| Homepage | SSG | ... |
| Product pages | ISR | ... |
| Dashboard | CSR | ... |

### Application Architecture
- Overall pattern (monolith / micro-frontends / Jamstack / hybrid)
- Folder structure
- Team ownership model

### Component Architecture
- Atomic design level mapping for key components
- Shared component library strategy
- Container queries vs breakpoint approach

### State Management
- Client state: tool + scope per state type
- Server state: TanStack Query setup
- Global state: Zustand stores if needed

### Data Fetching
- Fetch timing (route-level vs component-level)
- Caching strategy
- Error handling and partial failure approach

### Performance Budget
- LCP / CLS / FID targets
- Bundle size budget
- Key optimization techniques

### Trade-off Table

| Decision | Chosen approach | Alternative | Reason |
|----------|----------------|------------|--------|

## Arguments

Usage: `/fe:architecture [description of product or problem]`

Examples:
- `/fe:architecture an e-commerce platform with 500k products, SEO critical,
  team of 6 frontend engineers`
- `/fe:architecture a SaaS analytics dashboard with real-time data, authenticated users`
- `/fe:architecture evaluate our current architecture in src/`
- `/fe:architecture should we migrate from monolith to micro-frontends?`
