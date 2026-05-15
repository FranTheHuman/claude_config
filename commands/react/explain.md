# /react:explain — Explain a React Concept or Code

Explain a React concept, hook, pattern, API, or specific file from this project.
Default to React 19 explanations. Flag if a concept is deprecated or legacy.
Always include runnable TypeScript examples and a "when to use / when NOT to use" section.

## Instructions

1. Read `.claude/skills/react-best-practices/SKILL.md` before answering.
2. Parse the argument: concept name, hook, pattern, or file path.
3. If a file path is given, read the file and explain THAT specific code in context.
4. If the concept involves a deprecated API, explain both the old and new approach.
5. Structure the answer as below.
6. Never skip "When NOT to use" — it prevents the most common misapplication mistakes.

## Output Structure

### What it is
One clear paragraph. Define any React-specific jargon.

### How it works in React's rendering model
3–5 bullet points on what React does internally when this concept is used.
Focus on the reconciler, re-render behavior, or Fiber scheduling as relevant.

### Code example — minimal
Simplest possible working TypeScript + React example.

### Code example — realistic
Closer to what this project would actually use in production.

### Common mistakes
The 2–3 most frequent errors with this concept.
Format: ❌ Wrong → ✅ Correct with brief explanation.

### When to use / When NOT to use
Concise decision table with alternatives.

## Arguments

Usage: `/react:explain <concept|path>`

Examples:
- `/react:explain useActionState`
- `/react:explain useOptimistic`
- `/react:explain Server Components vs Client Components`
- `/react:explain useEffect`
- `/react:explain useState vs useReducer`
- `/react:explain Zustand vs TanStack Query`
- `/react:explain React Compiler`
- `/react:explain useTransition vs useDeferredValue`
- `/react:explain hydration mismatch`
- `/react:explain src/features/auth/components/LoginForm.tsx`
- `/react:explain why forwardRef is deprecated in React 19`

## Concepts Fully Covered by the Skill

`useState`, `useReducer`, `useEffect`, `useRef`, `useContext`, `useMemo`,
`useCallback`, `useTransition`, `useDeferredValue`, `useActionState`,
`useOptimistic`, `useFormStatus`, `use()`, `Suspense`, `lazy()`,
`React.memo`, `React Compiler`, `Server Components`, `Client Components`,
`Server Actions`, `Zustand`, `TanStack Query`, `forwardRef` deprecation,
`createRoot`, `hydrateRoot`, `key` prop, `ref` as plain prop,
native document metadata (`<title>/<meta>`), `@testing-library/react`,
`renderHook`, `userEvent`, Vite setup, Next.js App Router, feature-based structure
