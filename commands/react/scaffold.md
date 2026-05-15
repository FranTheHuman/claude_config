# /react:scaffold — Scaffold a New React Project

Generate a complete, production-ready React project from scratch.
Apply all rules from `.claude/skills/react-best-practices/SKILL.md`.
All code is TypeScript by default.

## Instructions

1. Read `.claude/skills/react-best-practices/SKILL.md` before generating.
2. Parse arguments: project type (Vite or Next.js), first feature name, addons.
3. Generate every file listed below — complete, not placeholder stubs.

## Output Files

### Both project types
- `tsconfig.json` — strict mode, path aliases
- `package.json` — scripts: dev, build, preview, lint, test
- `.eslintrc.json` — TypeScript + React hooks + jsx-a11y plugins
- `vitest.config.ts` or `jest.config.ts` — test runner setup
- `src/types/global.d.ts` — global TypeScript declarations

### Vite SPA output
- `vite.config.ts` — with React Compiler plugin if requested
- `src/main.tsx` — `createRoot`, QueryClientProvider, Router
- `src/App.tsx` — root component with routes
- `src/shared/components/` — Button, Input, Spinner base components
- `src/shared/hooks/` — useDebounce, useLocalStorage
- `src/features/<name>/` — full feature scaffold (see /react:new-feature)

### Next.js App Router output
- `app/layout.tsx` — root layout with providers
- `app/page.tsx` — home page (Server Component)
- `app/globals.css` — base styles
- `middleware.ts` — auth middleware if `--auth` flag given
- `lib/db.ts` — database client placeholder
- `actions/<name>.actions.ts` — Server Actions for the first feature
- `components/<name>/` — Server + Client Component pair as template

### Shared dev tooling
- `.husky/pre-commit` — runs lint + typecheck before commit
- `lint-staged.config.js` — lint only staged files

## Arguments

Usage: `/react:scaffold <project-name> [--type vite|next] [--feature <name>]
                         [--auth] [--compiler] [--db prisma|drizzle]`

Examples:
- `/react:scaffold my-dashboard --type vite --feature products --compiler`
- `/react:scaffold my-saas --type next --feature users --auth --db prisma`
- `/react:scaffold analytics-app --type vite --feature reports`
