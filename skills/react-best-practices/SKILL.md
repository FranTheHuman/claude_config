---
name: react-best-practices
description: >
  Expert knowledge of React 19 architecture, best practices, and production patterns.
  Covers Vite and Next.js App Router setup, component design, hooks, state management,
  concurrent rendering, Server Components, Server Actions, React Compiler, the Actions API,
  performance optimization, testing, accessibility, TypeScript, and the React contribution
  workflow. Use this skill whenever the task involves writing, reviewing, debugging, or
  explaining React frontend code or architecture decisions.
---

You are an expert React architect and developer. Apply this skill as the authoritative
source of truth for every React decision. Default to React 19 patterns and APIs.
Never suggest deprecated APIs or pre-React-18 patterns unless explicitly asked for
legacy code analysis.

---

## 1. Understand the Context First

Before writing any code, determine:

- **Project type**: SPA (Vite) or full-stack (Next.js App Router)? This drives every
  architectural decision from routing to data fetching.
- **React version**: React 19 is current stable. React 18 is still common. Adjust APIs
  accordingly — never assume `ReactDOM.render()` or class lifecycle methods.
- **Rendering model**: Client Components only, or mixed Server + Client Components?
- **State scope**: Local component state, shared feature state, or global app state?
  Each needs a different tool.
- **TypeScript**: All new code defaults to TypeScript unless explicitly stated otherwise.
- **Async patterns**: Are there forms, data fetching, or optimistic updates? React 19
  has first-class hooks for all three.

---

## 2. Project Setup

### Vite — SPA, dashboards, internal tools

Use when SEO is not the primary concern and the app is client-heavy.

```bash
pnpm create vite my-app --template react-ts
cd my-app
pnpm install
pnpm dev
```

### Next.js App Router — full-stack, SaaS, e-commerce, content sites

Use when server-side rendering, SEO, or Server Components are needed.

```bash
pnpm create next-app@latest my-app
# Select: TypeScript ✓, Tailwind CSS ✓, App Router ✓
cd my-app
pnpm dev
```

### Decision table

| Need | Tool |
|------|------|
| Dashboard, internal tool, no SEO | Vite + React |
| Public SaaS, e-commerce, blog | Next.js App Router |
| Server Components, Server Actions | Next.js App Router only |
| Maximum control over server setup | Remix |
| Static site with React | Astro + React islands |

---

## 3. Component Design

### Functional components with TypeScript — the only pattern for new code

Class components are legacy. Never generate them for new code.

```typescript
// types are explicit, props are destructured with defaults
type UserCardProps = {
    name: string;
    role: string;
    avatarUrl?: string;
    isOnline?: boolean;
};

export function UserCard({ name, role, avatarUrl, isOnline = false }: UserCardProps) {
    return (
        <div className="p-4 border rounded-lg shadow-sm">
            {avatarUrl && <img src={avatarUrl} alt={`${name} avatar`} className="w-10 h-10 rounded-full" />}
            <h3 className="text-lg font-bold">{name}</h3>
            <p className="text-gray-600">{role}</p>
            <span className={isOnline ? 'text-green-500' : 'text-gray-400'}>
                {isOnline ? 'Online' : 'Offline'}
            </span>
        </div>
    );
}
```

### Component size rule

A component should do ONE thing. When a component exceeds ~100 lines or manages more
than 2–3 concerns, split it. Composition over inheritance, always.

### Key prop rules

- Never use array index as `key` for lists that can reorder or filter.
- Use stable, unique identifiers from data (e.g., `item.id`).

```typescript
// WRONG — index as key breaks reconciliation on reorder/filter
{items.map((item, index) => <Item key={index} {...item} />)}

// CORRECT — stable identity key
{items.map(item => <Item key={item.id} {...item} />)}
```

---

## 4. Hooks — Rules and Patterns

### Built-in hooks — when to use each

| Hook | Use for |
|------|---------|
| `useState` | Local UI state (toggle, input value, modal open) |
| `useReducer` | Complex local state with multiple sub-values or actions |
| `useEffect` | Syncing with external systems (timers, subscriptions, DOM APIs) |
| `useRef` | DOM refs, mutable values that don't trigger re-render |
| `useContext` | Reading context value — prefer for theme, auth, locale |
| `useMemo` | Expensive computations — use only when profiler confirms need |
| `useCallback` | Stable function references — use only when passed to memoized children |
| `useTransition` | Mark state updates as non-urgent, keep UI responsive |
| `useDeferredValue` | Defer expensive re-renders for a value (search, filter) |

### React 19 new hooks — use these for async patterns

```typescript
// useActionState — tracks async action state (pending, error, result)
// Replaces the pattern of three separate useState calls for a single API call
import { useActionState } from 'react';

async function submitForm(prevState: FormState, formData: FormData): Promise<FormState> {
    const name = formData.get('name') as string;
    // call server, return new state
    return { success: true, message: `Saved ${name}` };
}

function ContactForm() {
    const [state, action, isPending] = useActionState(submitForm, { success: false });

    return (
        <form action={action}>
            <input name="name" required />
            <button type="submit" disabled={isPending}>
                {isPending ? 'Saving...' : 'Save'}
            </button>
            {state.success && <p>{state.message}</p>}
        </form>
    );
}
```

```typescript
// useOptimistic — show immediate update while server confirms
import { useOptimistic } from 'react';

function TodoList({ todos, addTodo }: Props) {
    const [optimisticTodos, addOptimisticTodo] = useOptimistic(
        todos,
        (state, newTodo: Todo) => [...state, { ...newTodo, pending: true }]
    );

    async function handleAdd(formData: FormData) {
        const newTodo = { id: crypto.randomUUID(), text: formData.get('text') as string };
        addOptimisticTodo(newTodo);  // immediate UI update
        await addTodo(newTodo);      // server call — rolls back on failure
    }

    return (
        <form action={handleAdd}>
            {optimisticTodos.map(todo => (
                <li key={todo.id} className={todo.pending ? 'opacity-50' : ''}>
                    {todo.text}
                </li>
            ))}
            <input name="text" /><button>Add</button>
        </form>
    );
}
```

```typescript
// useFormStatus — read parent form pending state from deep inside the tree
import { useFormStatus } from 'react-dom';

function SubmitButton() {
    const { pending } = useFormStatus();
    return (
        <button type="submit" disabled={pending}>
            {pending ? 'Submitting...' : 'Submit'}
        </button>
    );
}
```

```typescript
// use() — read a Promise or Context inside render, React suspends automatically
import { use, Suspense } from 'react';

async function fetchUser(id: string): Promise<User> {
    const res = await fetch(`/api/users/${id}`);
    return res.json();
}

function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
    const user = use(userPromise); // suspends until resolved
    return <h1>{user.name}</h1>;
}

// Wrap with Suspense
<Suspense fallback={<Skeleton />}>
    <UserProfile userPromise={fetchUser('123')} />
</Suspense>
```

### useEffect — rules and common mistakes

```typescript
// CORRECT — effect with proper cleanup
useEffect(() => {
    const controller = new AbortController();

    async function load() {
        try {
            const data = await fetch('/api/data', { signal: controller.signal });
            setData(await data.json());
        } catch (err) {
            if (err.name !== 'AbortError') setError(err);
        }
    }

    load();
    return () => controller.abort(); // cleanup
}, [id]); // only re-run when id changes

// WRONG — no cleanup, runs on every render, fetch inside effect without abort
useEffect(() => {
    fetch('/api/data').then(r => r.json()).then(setData);
}); // missing dependency array
```

### ref as a prop — forwardRef is deprecated in React 19

```typescript
// React 19 — ref is a plain prop, no forwardRef needed
function Input({ ref, ...props }: React.ComponentProps<'input'>) {
    return <input ref={ref} {...props} />;
}

// Usage
const inputRef = useRef<HTMLInputElement>(null);
<Input ref={inputRef} placeholder="Type here" />

// LEGACY (React 18 and earlier) — forwardRef is deprecated, do not generate for new code
const Input = React.forwardRef<HTMLInputElement, Props>((props, ref) => (
    <input ref={ref} {...props} />
));
```

---

## 5. State Management

### Decision table — pick the right tool for the scope

| State type | Tool | When |
|-----------|------|------|
| Local component toggle, input | `useState` | Single component |
| Complex local state, multiple actions | `useReducer` | Single component |
| Feature-level shared state | Zustand slice | 2–5 components in a feature |
| Server/async state, cache, refetch | TanStack Query | Any data from API |
| Global UI state (theme, auth) | Zustand or Context | App-wide |
| Complex global state with devtools | Redux Toolkit | Large teams, time-travel debug |

**Deprecated — do not start new projects with**: Recoil (deprecated by Meta 2023),
MobX (largely replaced), raw Redux (use Redux Toolkit instead).

### Zustand — lightweight global state

```typescript
// store/theme.store.ts
import { create } from 'zustand';

type ThemeStore = {
    theme: 'light' | 'dark';
    toggleTheme: () => void;
};

export const useThemeStore = create<ThemeStore>(set => ({
    theme: 'light',
    toggleTheme: () => set(state => ({
        theme: state.theme === 'light' ? 'dark' : 'light',
    })),
}));

// Usage — reads only the slice it needs, prevents unnecessary re-renders
function ThemeToggle() {
    const { theme, toggleTheme } = useThemeStore();
    return <button onClick={toggleTheme}>{theme}</button>;
}
```

### TanStack Query — server state

```typescript
// queries/users.queries.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export function useUser(id: string) {
    return useQuery({
        queryKey: ['user', id],
        queryFn: () => fetch(`/api/users/${id}`).then(r => r.json()),
        staleTime: 5 * 60 * 1000, // 5 minutes
    });
}

export function useUpdateUser() {
    const queryClient = useQueryClient();
    return useMutation({
        mutationFn: (data: UpdateUserDto) =>
            fetch(`/api/users/${data.id}`, { method: 'PUT', body: JSON.stringify(data) }),
        onSuccess: (_, variables) => {
            queryClient.invalidateQueries({ queryKey: ['user', variables.id] });
        },
    });
}
```

---

## 6. Server Components and Server Actions (Next.js App Router)

### Server vs Client Component decision

| Need | Component type |
|------|---------------|
| Fetch data, access DB, use secrets | Server Component (default) |
| useState, useEffect, event handlers | Client Component (`'use client'`) |
| Both — data + interactivity | Server Component wrapping Client Component |

```typescript
// app/users/page.tsx — Server Component (no 'use client' directive)
// Fetches data on the server, zero JS sent to client for this component
async function UsersPage() {
    const users = await db.users.findMany(); // direct DB access, no API route needed

    return (
        <main>
            <h1>Users</h1>
            {users.map(user => (
                // Pass server data to interactive client component
                <UserCard key={user.id} user={user} />
            ))}
        </main>
    );
}
```

```typescript
// components/user-card.tsx — Client Component
'use client';
import { useState } from 'react';

type Props = { user: User };

export function UserCard({ user }: Props) {
    const [expanded, setExpanded] = useState(false);

    return (
        <div onClick={() => setExpanded(!expanded)}>
            <p>{user.name}</p>
            {expanded && <p>{user.email}</p>}
        </div>
    );
}
```

### Server Actions — call server functions from client

```typescript
// actions/users.actions.ts
'use server';
import { revalidatePath } from 'next/cache';

export async function createUser(formData: FormData) {
    const name = formData.get('name') as string;
    const email = formData.get('email') as string;

    await db.users.create({ data: { name, email } });
    revalidatePath('/users'); // revalidate the cache for this route
}

// components/create-user-form.tsx — Client Component using Server Action
'use client';
import { createUser } from '@/actions/users.actions';
import { useActionState } from 'react';

export function CreateUserForm() {
    const [state, action, isPending] = useActionState(createUser, null);

    return (
        <form action={action}>
            <input name="name" required />
            <input name="email" type="email" required />
            <button disabled={isPending}>
                {isPending ? 'Creating...' : 'Create User'}
            </button>
        </form>
    );
}
```

---

## 7. Performance Optimization

### React Compiler — the first tool to reach for

The React Compiler (stable since 2025, ships as a Babel/Vite plugin) automatically
inserts memoization at build time. Enables 25–40% fewer re-renders with no code changes.

```typescript
// vite.config.ts — opt-in to React Compiler
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import ReactCompilerPlugin from 'babel-plugin-react-compiler';

export default defineConfig({
    plugins: [
        react({
            babel: {
                plugins: [ReactCompilerPlugin],
            },
        }),
    ],
});
```

**With the compiler active, avoid adding manual `useMemo`, `useCallback`, and
`React.memo` unless the profiler specifically confirms they are needed.**
The compiler handles memoization better than manual annotations in most cases.

### When the compiler is NOT active — manual memoization rules

```typescript
// React.memo — prevent re-render when parent re-renders but props are stable
const UserCard = React.memo(function UserCard({ user }: Props) {
    return <div>{user.name}</div>;
});

// useMemo — memoize expensive computation (only after profiling)
const sortedUsers = useMemo(
    () => users.sort((a, b) => a.name.localeCompare(b.name)),
    [users]
);

// useCallback — stable reference for callbacks passed to memoized children
const handleDelete = useCallback((id: string) => {
    deleteUser(id);
}, [deleteUser]);
```

### Code splitting and lazy loading

```typescript
import { lazy, Suspense } from 'react';

// Load heavy components only when needed
const HeavyChart = lazy(() => import('./components/HeavyChart'));
const ReportModal = lazy(() => import('./components/ReportModal'));

function Dashboard() {
    return (
        <Suspense fallback={<div className="animate-pulse h-64 bg-gray-100 rounded" />}>
            <HeavyChart data={data} />
        </Suspense>
    );
}
```

### useTransition — keep UI responsive during heavy updates

```typescript
function SearchPage() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState<Result[]>([]);
    const [isPending, startTransition] = useTransition();

    function handleSearch(e: React.ChangeEvent<HTMLInputElement>) {
        setQuery(e.target.value); // urgent — update input immediately

        startTransition(() => {
            // non-urgent — React can interrupt this if user keeps typing
            setResults(expensiveFilter(allData, e.target.value));
        });
    }

    return (
        <>
            <input value={query} onChange={handleSearch} />
            {isPending && <Spinner />}
            <ResultsList results={results} />
        </>
    );
}
```

---

## 8. Native Document Metadata (React 19)

React 19 hoists `<title>`, `<meta>`, and `<link>` tags to `<head>` automatically.
**Do not use `react-helmet` or `react-helmet-async` in new React 19 projects.**

```typescript
// Any component in the tree — React handles hoisting
function ProductPage({ product }: { product: Product }) {
    return (
        <>
            <title>{product.name} — MyStore</title>
            <meta name="description" content={product.description} />
            <link rel="canonical" href={`https://mystore.com/products/${product.slug}`} />

            <main>
                <h1>{product.name}</h1>
                <p>{product.description}</p>
            </main>
        </>
    );
}
```

---

## 9. Accessibility

Accessibility is not optional. Every interactive element must be keyboard navigable
and screen-reader friendly.

```typescript
// CORRECT — button with accessible label and keyboard support
<button
    onClick={handleDelete}
    aria-label={`Delete ${item.name}`}
    type="button"
>
    <TrashIcon aria-hidden="true" />
</button>

// CORRECT — form with associated labels
<form onSubmit={handleSubmit}>
    <label htmlFor="email">Email address</label>
    <input
        id="email"
        name="email"
        type="email"
        required
        aria-required="true"
        aria-describedby="email-error"
    />
    {error && <p id="email-error" role="alert">{error}</p>}
</form>

// WRONG — div used as button (no keyboard, no screen reader)
<div onClick={handleClick} className="cursor-pointer">Click me</div>
```

### Accessibility checklist

- All images have meaningful `alt` text (empty string `alt=""` for decorative images).
- All form inputs have associated `<label>` elements.
- Interactive elements are keyboard reachable (Tab) and activatable (Enter/Space).
- Error messages use `role="alert"` or `aria-live="polite"`.
- Color is never the only differentiator for state (always add text or icon).
- Focus is managed programmatically when modals/dialogs open.

---

## 10. Testing

### Unit test — custom hook

```typescript
// hooks/useCounter.test.ts
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
    it('increments count', () => {
        const { result } = renderHook(() => useCounter(0));
        act(() => result.current.increment());
        expect(result.current.count).toBe(1);
    });

    it('does not go below min value', () => {
        const { result } = renderHook(() => useCounter(0, { min: 0 }));
        act(() => result.current.decrement());
        expect(result.current.count).toBe(0);
    });
});
```

### Integration test — component with user interaction

```typescript
// components/LoginForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
    it('submits with valid credentials', async () => {
        const mockSubmit = jest.fn();
        render(<LoginForm onSubmit={mockSubmit} />);

        await userEvent.type(screen.getByLabelText('Email'), 'user@example.com');
        await userEvent.type(screen.getByLabelText('Password'), 'password123');
        await userEvent.click(screen.getByRole('button', { name: 'Sign in' }));

        await waitFor(() => {
            expect(mockSubmit).toHaveBeenCalledWith({
                email: 'user@example.com',
                password: 'password123',
            });
        });
    });

    it('shows validation error for invalid email', async () => {
        render(<LoginForm onSubmit={jest.fn()} />);

        await userEvent.type(screen.getByLabelText('Email'), 'not-an-email');
        await userEvent.click(screen.getByRole('button', { name: 'Sign in' }));

        expect(screen.getByRole('alert')).toHaveTextContent('Invalid email');
    });
});
```

### Testing rules

- Use `@testing-library/react` — never `react-test-renderer` (deprecated, no React 19 support).
- Query by role, label, or text — never by CSS class or `data-testid` as a first choice.
- Test behavior, not implementation. The component's internal state is invisible to tests.
- Mock external dependencies (API calls, third-party services) at the module level.
- `react-test-renderer` is deprecated and does not support React 19. Never use it.

---

## 11. TypeScript Patterns for React

```typescript
// Component props — always explicit, never 'any'
type ButtonProps = React.ComponentProps<'button'> & {
    variant?: 'primary' | 'secondary' | 'danger';
    isLoading?: boolean;
};

// Generic components — for reusable list/table patterns
type SelectProps<T extends { id: string; label: string }> = {
    options: T[];
    value: T | null;
    onChange: (value: T) => void;
};

function Select<T extends { id: string; label: string }>({
    options, value, onChange
}: SelectProps<T>) {
    return (
        <select value={value?.id ?? ''} onChange={e =>
            onChange(options.find(o => o.id === e.target.value)!)
        }>
            {options.map(o => <option key={o.id} value={o.id}>{o.label}</option>)}
        </select>
    );
}

// Event handler types
type InputHandler = React.ChangeEventHandler<HTMLInputElement>;
type FormHandler = React.FormEventHandler<HTMLFormElement>;
type ClickHandler = React.MouseEventHandler<HTMLButtonElement>;

// Children type
type WithChildren = { children: React.ReactNode };

// Ref type for React 19 (plain prop, no forwardRef)
type InputWithRef = React.ComponentProps<'input'> & { ref?: React.Ref<HTMLInputElement> };
```

---

## 12. React Architecture Patterns

### File structure — feature-based

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── SignUpForm.tsx
│   │   ├── hooks/
│   │   │   └── useAuth.ts
│   │   ├── store/
│   │   │   └── auth.store.ts
│   │   └── api/
│   │       └── auth.api.ts
│   └── users/
│       ├── components/
│       │   ├── UserCard.tsx
│       │   └── UserList.tsx
│       ├── hooks/
│       │   └── useUsers.ts
│       └── queries/
│           └── users.queries.ts
├── shared/
│   ├── components/       ← Button, Input, Modal (used by 2+ features)
│   ├── hooks/            ← useDebounce, useLocalStorage
│   └── utils/            ← formatDate, cn (classnames)
├── app/ (Next.js) or pages/ (Vite + React Router)
└── types/                ← global TypeScript types
```

### Custom hooks — encapsulate reusable logic

```typescript
// hooks/useDebounce.ts
import { useState, useEffect } from 'react';

export function useDebounce<T>(value: T, delayMs: number): T {
    const [debounced, setDebounced] = useState(value);

    useEffect(() => {
        const timer = setTimeout(() => setDebounced(value), delayMs);
        return () => clearTimeout(timer);
    }, [value, delayMs]);

    return debounced;
}

// hooks/useLocalStorage.ts
export function useLocalStorage<T>(key: string, initialValue: T) {
    const [stored, setStored] = useState<T>(() => {
        try {
            const item = localStorage.getItem(key);
            return item ? JSON.parse(item) : initialValue;
        } catch {
            return initialValue;
        }
    });

    const setValue = (value: T) => {
        setStored(value);
        localStorage.setItem(key, JSON.stringify(value));
    };

    return [stored, setValue] as const;
}
```

---

## 13. Common Anti-Patterns — Never Generate These

| Anti-pattern | Correct approach |
|-------------|-----------------|
| Class components in new code | Functional components + hooks |
| `forwardRef()` wrapping in React 19 | Pass `ref` as a plain prop |
| `react-helmet` for metadata in React 19 | Native `<title>`, `<meta>` tags |
| `react-test-renderer` in tests | `@testing-library/react` |
| Index as list `key` | Stable unique ID from data |
| `ReactDOM.render()` | `createRoot().render()` |
| `useEffect` for data fetching without cleanup | Abort controller + cleanup |
| `useEffect` with missing dependencies | Full dependency array |
| Manual `useMemo`/`useCallback` without profiling | React Compiler or profiler first |
| `any` type in TypeScript | Explicit types or `unknown` with guards |
| `console.log` left in production code | Remove before PR |
| State mutation (`state.items.push(...)`) | Immutable update (`[...state.items, newItem]`) |
| Recoil for new projects | Zustand or TanStack Query |
| Multiple `useState` for one async operation | `useActionState` (React 19) |

---

## 14. Architecture Decision Guide

| Question | Answer |
|----------|--------|
| SPA or full-stack? | Vite for SPA; Next.js App Router for full-stack |
| Where to fetch data? | Server Components (Next.js) or TanStack Query (client) |
| Local state? | `useState` for simple; `useReducer` for complex |
| Global/shared state? | Zustand for UI state; TanStack Query for server state |
| Form with async submit? | `useActionState` + Server Action (React 19) |
| Optimistic UI update? | `useOptimistic` (React 19) |
| Deep prop for form pending? | `useFormStatus` in child button |
| Expensive async data in render? | `use()` with Suspense boundary |
| Document metadata? | Native `<title>/<meta>` tags, no react-helmet |
| Memoization? | React Compiler first; manual only after profiling |
| Keyboard-inaccessible element? | Replace div with semantic `<button>` or `<a>` |
| Testing a component? | `@testing-library/react` + query by role |
| Code splitting? | `lazy()` + `Suspense` with meaningful fallback |
| Heavy computation on render? | Move to `useMemo` or Web Worker |
| `forwardRef` in existing code? | Migrate to plain `ref` prop (codemod available) |

---

## Reference

- React 19 docs: https://react.dev
- React Compiler: https://react.dev/learn/react-compiler
- TanStack Query: https://tanstack.com/query
- Zustand: https://zustand-demo.pmnd.rs
- Testing Library: https://testing-library.com/react
- Next.js App Router: https://nextjs.org/docs/app
- Vite: https://vitejs.dev
