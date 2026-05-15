# /react:new-feature — Generate a Complete React Feature

Generate all layers of a new feature: components, hooks, queries/store, and types.
Always generates them together — never a single file in isolation.
Apply all rules from `.claude/skills/react-best-practices/SKILL.md`.
All code is React 19 + TypeScript.

## Instructions

1. Read `.claude/skills/react-best-practices/SKILL.md` before generating.
2. Parse: feature name, entities involved, interaction type, project type.
3. Scan `src/` for existing naming conventions and shared component usage.
4. For Next.js projects: include Server Component, Client Component, and Server Action.
5. For Vite/SPA: include TanStack Query hooks and Zustand slice if shared state needed.
6. Generate a test file alongside every component.

## Output Files

For a feature named `products`:

### 1. `src/features/products/types/product.types.ts`
- Domain types and DTOs: `Product`, `CreateProductDto`, `UpdateProductDto`

### 2. `src/features/products/api/products.api.ts` (Vite) or
   `src/features/products/actions/products.actions.ts` (Next.js)
- Vite: fetch functions returning typed data
- Next.js: Server Actions with `'use server'` directive + `revalidatePath`

### 3. `src/features/products/hooks/useProducts.ts` (Vite only)
- TanStack Query hooks: `useProducts`, `useProduct`, `useCreateProduct`
- Correct `queryKey` arrays, `staleTime`, `invalidateQueries` on mutation

### 4. `src/features/products/components/ProductList.tsx`
- List component with loading state (Suspense or `isPending`)
- Empty state, error state
- Uses `@testing-library/react` test

### 5. `src/features/products/components/ProductForm.tsx`
- Form using `useActionState` for async submit (React 19)
- `useOptimistic` if the list updates optimistically
- `SubmitButton` child using `useFormStatus`
- Full accessibility: labels, error messages with `role="alert"`, keyboard nav
- Test covering submit, validation error, and loading state

### 6. `src/features/products/components/ProductCard.tsx`
- Display component with typed props
- `aria-label` on interactive actions

## Arguments

Usage: `/react:new-feature <name> [--type vite|next] [--ops list,create,update,delete]
                             [--optimistic] [--no-test]`

Examples:
- `/react:new-feature products --type vite --ops list,create,update`
- `/react:new-feature orders --type next --ops list,create --optimistic`
- `/react:new-feature categories --type vite --ops list,create --no-test`

## Rules from Skill

- Every form uses `useActionState` — never three separate `useState` calls for
  one async operation
- Every list has a stable unique `key` — never array index
- Every interactive element is keyboard accessible
- State is never mutated — always return new object/array references
- `forwardRef` is never used — pass `ref` as a plain prop
