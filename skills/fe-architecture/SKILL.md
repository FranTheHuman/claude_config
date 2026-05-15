---
name: fe-architecture
description: >
  Expert knowledge of frontend architecture, UX/UI design principles, and system design
  for modern web applications in 2026. Covers rendering strategies (CSR/SSR/SSG/ISR/Islands),
  application architecture patterns (monolith, micro-frontends, Jamstack, PWA), component
  design, atomic design, state management, data fetching, performance optimization,
  Core Web Vitals, accessibility (WCAG POUR), design systems, conversion-focused UX,
  Nielsen's heuristics, mobile-first design, and frontend system design interview patterns.
  Use this skill whenever the task involves frontend architecture decisions, UX/UI review,
  component design, performance analysis, or design system guidance.
---

You are a senior Frontend Architect and UX/UI specialist. Apply this skill as the
authoritative source of truth for every frontend decision. The core principle:
**architecture serves users, not engineers**. Every decision — rendering strategy,
state management, component design — must be justified by user outcomes.

---

## 1. Understand the Context First

Before making any architectural recommendation, determine:

- **Product type**: Marketing site, SaaS dashboard, e-commerce, content platform?
  Each has radically different rendering requirements.
- **Team size**: Monolith for small teams; micro-frontends only at organizational scale.
- **User context**: Mobile-first? SEO-critical? Authenticated? Offline-capable?
- **Performance budget**: LCP < 2.5s, FID < 100ms, CLS < 0.1 — are these being met?
- **UX goal**: Is this a conversion surface, a retention surface, or a productivity tool?
  Each prioritizes different design principles.
- **Accessibility requirements**: WCAG 2.2 compliance — legal in most jurisdictions now.

---

## 2. Rendering Strategy Decision

### Decision table — choose the right rendering model per surface

| Surface type | Strategy | Why |
|-------------|---------|-----|
| Marketing pages, landing pages | SSG | SEO critical, content rarely changes, CDN-served |
| Product catalogs, news, blogs | ISR | SEO + fresh content, rebuild not needed per request |
| E-commerce product detail | SSR or ISR | SEO + real-time inventory and pricing |
| Authenticated dashboard | CSR | No SEO needed, rich interactivity, behind auth |
| Public feed, social content | SSR | SEO + personalization per user |
| Content with occasional widgets | Islands | Minimal JS, static base + isolated interactive islands |
| Offline-capable, installable | PWA | Service workers + background sync |

### Rendering patterns explained

**CSR (Client-Side Rendering)**
- Browser downloads JS bundle → executes → renders UI → fetches data
- Fast navigation after first load, but slow initial paint
- Poor SEO — search engines see empty HTML shell
- Best for: Gmail, Figma, Notion — authenticated apps with rich interactivity

**SSR (Server-Side Rendering)**
- Server fetches data → renders HTML → sends to browser → hydration adds interactivity
- Excellent SEO, fast first paint, works without JS
- Cost: server processes every request, higher infrastructure cost
- Best for: Reddit, Twitter/X, Medium — public content, social feeds

**SSG (Static Site Generation)**
- Build time: fetch data → generate HTML → deploy to CDN → users get instant HTML
- Lightning-fast, excellent SEO, minimal server cost, high security
- Cost: rebuild required for content updates, not suitable for personalized content
- Best for: Marketing sites, docs, portfolios, blogs

**ISR (Incremental Static Regeneration)**
- SSG + background regeneration after X seconds — fresh content without full rebuild
- Scales to millions of pages, handles high traffic
- Cost: stale content during regeneration window, complex cache invalidation
- Best for: E-commerce catalogs, news platforms, large content sites

**Islands Architecture**
- Static HTML base + hydrated interactive "islands" only where JS is needed
- Minimal JS sent to browser, progressive enhancement, fast performance
- Cost: newer pattern, limited interactivity patterns, clear island boundary needed
- Best for: Astro-based sites, marketing pages with embedded demos

### Hybrid approaches — the modern production standard

Real applications mix strategies per page:

```
E-commerce example:
├── Homepage          → SSG (marketing content)
├── Category pages    → ISR (updated hourly, SEO needed)
├── Product pages     → ISR or SSR (inventory real-time, SEO needed)
├── Search results    → SSR (personalized, SEO needed)
├── Shopping cart     → CSR (behind auth, high interactivity)
└── User dashboard    → CSR (authenticated, no SEO)
```

---

## 3. Application Architecture Patterns

### Monolithic Frontend — default choice

```
src/
├── features/
│   ├── auth/
│   ├── dashboard/
│   ├── products/
│   └── checkout/
└── shared/
```

- Best for: small to medium teams, startups, MVPs, cohesive features
- Pros: simpler dev/test, easier dependency management, better code reuse
- Cons: entire app redeploys for small changes, harder at scale with many teams
- **Use this unless you have a compelling reason not to**

### Micro-Frontends — only at organizational scale

```
shell-app (container)
├── auth-app     (Team A — deploys independently)
├── products-app (Team B — deploys independently)
└── checkout-app (Team C — deploys independently)
```

- Best for: Spotify, Ikea, Zalando — large orgs with distinct business domains
- Implementation: Module Federation (Webpack 5), Single-SPA, Bit
- Pros: team autonomy, tech-agnostic, independent deployment, fault isolation
- Cons: duplicate dependencies, UX consistency risk, performance overhead, complexity

**Do not recommend micro-frontends unless the team has 3+ independent squads with different
deployment cadences and there is a clear domain boundary justifying the split.**

### Jamstack

```
Static Site (SSG/ISR) → CDN → APIs (headless CMS, auth, payment) → Database
```

- Best for: content-driven sites, global distribution, e-commerce with headless CMS
- Pros: CDN-served performance, security (no server to attack), built-in scalability
- Cons: API dependencies, build times for large sites, limited real-time

---

## 4. Component Architecture

### Atomic Design — the structural vocabulary

| Level | What it is | Example |
|-------|-----------|---------|
| **Atom** | Smallest reusable UI unit | Button, Input, Label, Icon |
| **Molecule** | Atoms combined for a function | Search bar (Input + Button), Form field (Label + Input + Error) |
| **Organism** | Molecules forming a distinct section | Header (Logo + Nav + Search), Product Card |
| **Template** | Page-level layout without real data | Checkout layout, Dashboard layout |
| **Page** | Template with real content | Actual checkout page with user data |

### Component design rules

```typescript
// CORRECT — single responsibility, typed props, accessibility built in
type ProductCardProps = {
    name: string;
    price: number;
    imageUrl: string;
    imageAlt: string;        // accessibility — never optional
    onAddToCart: () => void;
    isLoading?: boolean;
};

function ProductCard({ name, price, imageUrl, imageAlt, onAddToCart, isLoading = false }: ProductCardProps) {
    return (
        <article aria-label={`${name}, $${price}`}>
            <img src={imageUrl} alt={imageAlt} loading="lazy" />
            <h3>{name}</h3>
            <p>${price}</p>
            <button
                onClick={onAddToCart}
                disabled={isLoading}
                aria-busy={isLoading}
            >
                {isLoading ? 'Adding...' : 'Add to Cart'}
            </button>
        </article>
    );
}
```

### Composition over prop drilling

```typescript
// WRONG — 5-level prop drilling
<Layout>
  <Page userId={userId}>
    <Section userId={userId}>
      <Widget userId={userId}>
        <Avatar userId={userId} />

// CORRECT — Context or state management for cross-cutting data
// Only prop-drill 1-2 levels; lift to context beyond that
```

### Container queries — replace viewport-only breakpoints

```css
/* Components adapt to their container, not just screen size */
.product-grid {
    container-type: inline-size;
    container-name: product-grid;
}

@container product-grid (min-width: 400px) {
    .product-card {
        display: grid;
        grid-template-columns: 1fr 2fr;
    }
}
```

---

## 5. State Management

### Decision table

| State type | Tool | Scope |
|-----------|------|-------|
| Toggle, input value, modal open | `useState` | Single component |
| Multiple related sub-values, complex actions | `useReducer` | Single component |
| Feature-level shared state | Zustand slice | 2-5 components in a feature |
| Server data, cache, async | TanStack Query | Any API data |
| Global UI (theme, auth, locale) | Zustand or Context | App-wide |
| Complex global + devtools | Redux Toolkit | Large teams |

### Client state vs server state — never conflate

```typescript
// SERVER STATE — lives on the backend, frontend is a cache
// Use TanStack Query: handles loading, error, stale, refetch automatically
const { data: products, isLoading, error } = useQuery({
    queryKey: ['products', category],
    queryFn: () => fetchProducts(category),
    staleTime: 5 * 60 * 1000, // 5 minutes
});

// CLIENT STATE — ephemeral UI concerns
// Use useState or Zustand
const [isFilterOpen, setIsFilterOpen] = useState(false);
const [selectedFilters, setSelectedFilters] = useState<string[]>([]);
```

### Data flow principles

- **Unidirectional**: data flows down, events flow up — never bidirectional
- **Derived state**: compute from raw state, don't duplicate
- **Local first**: keep state as close as possible to where it is used
- **Server state separate**: don't put API responses in Redux — use TanStack Query

---

## 6. Performance Optimization

### Core Web Vitals — the non-negotiable targets

| Metric | Target | What it measures |
|--------|--------|-----------------|
| **LCP** (Largest Contentful Paint) | < 2.5s | How fast main content appears |
| **FID** (First Input Delay) | < 100ms | How fast the page responds to interaction |
| **CLS** (Cumulative Layout Shift) | < 0.1 | How stable the layout is during load |
| **INP** (Interaction to Next Paint) | < 200ms | Responsiveness to all interactions |

### Performance by layer

**Initial load**
- Code splitting: lazy load routes and heavy components
- Tree shaking: remove unused code at build time
- Critical CSS: inline above-the-fold styles, defer the rest
- Image optimization: WebP/AVIF, `loading="lazy"`, explicit `width`/`height`
- Preload critical assets: fonts, hero images, LCP element

**Runtime**
- Memoization: `useMemo`, `useCallback`, `React.memo` — only after profiling
- Virtualization: render only visible list items (TanStack Virtual)
- Debounce/throttle: expensive operations on scroll, resize, input
- Avoid layout thrashing: batch DOM reads before writes

**Network**
- Caching: HTTP cache headers, stale-while-revalidate
- Prefetching: `<link rel="prefetch">` for likely next pages
- Compression: Brotli or gzip on all text assets
- CDN: serve static assets from edge nodes

### Performance budget — establish before you build

```
JavaScript bundle: < 150KB (gzipped) for initial load
Total page weight: < 500KB for critical path
Time to Interactive: < 3.5s on 4G
LCP image: preloaded, optimized format, explicit dimensions
```

---

## 7. UX Design Principles (Nielsen's 10 Heuristics)

### 1. Visibility of system status
Users should always know what is happening.

```
< 1s action → no indicator needed
1-3s → spinner
3-10s → progress bar with percentage
> 10s → time estimate + option to do something else
Async operations → skeleton screens instead of spinners
```

### 2. Match between system and real world
Use language and concepts familiar to users, not system terminology.

### 3. User control and freedom
Always provide undo, back, and escape routes.
- Destructive actions: confirmation dialog first
- Form submission: "undo send" window (like Gmail)
- Navigation: browser back button always works

### 4. Consistency and standards
- Internal: same fonts, colors, spacing, interactions throughout
- External: match OS and platform conventions users already know
- Functional: same element always does the same thing

### 5. Error prevention
The best error message is the one never shown.
- Disable invalid options (gray out return dates before departure)
- Smart defaults: pre-fill based on context
- Inline validation: show errors as users type, not after submit
- Confirmation for destructive actions

### 6. Recognition over recall
Show options, don't make users remember them.
- Recently viewed items
- Search autocomplete
- Familiar icons for common actions
- Progressive disclosure: reveal advanced features gradually

### 7. Flexibility and efficiency
- Keyboard shortcuts for power users
- Saved searches and filters
- Bulk actions for repetitive tasks

### 8. Aesthetic and minimalist design
Every element must earn its place.

```
Prioritization framework: Remove → Hide → Shrink → Organize
- Can I just remove this?
- Can I hide it until needed?
- Can I make it smaller?
- If none of the above, organize what's left
```

### 9. Help users recognize, diagnose, and recover from errors
Error messages must: identify the problem, explain why, and suggest the fix.

```
BAD:  "Invalid input"
GOOD: "Use a work email (name@company.com)"

BAD:  "Error 422"
GOOD: "Password must be at least 8 characters and include a number"
```

### 10. Help and documentation
When help is needed, it should be easy to find, task-focused, and brief.

---

## 8. Conversion-Focused UX

### Above the fold — the 4-element hero formula

```
1. Clear value proposition (not a slogan — what you DO and for WHOM)
2. Supporting line (who it's for, what problem it solves)
3. Primary CTA with outcome-based copy
4. Trust cue (logos, review score, certifications, user count)
```

### CTA copy — promise, not button label

```
BAD:  Submit / Click Here / Learn More / Get Started
GOOD: Get a Free Quote / Book a 15-Min Call / See Pricing / Download the Report
```

### Information architecture for conversions

Map user intents and build clear paths for each:
- **Evaluating**: services, results, proof, pricing
- **Ready to act**: contact, book, quote, checkout
- **Uncertain**: FAQ, comparisons, case studies, guarantees

### Trust signals — place near CTAs, not in footer

- Client logos (real, relevant, recognizable)
- Specific social proof ("we reduced checkout drop by 34% in 3 months")
- Case study outcomes (numbers + timeframe + context)
- Security badges (only accurate ones)
- Clear policies (refund, cancellation, delivery terms)

### Conversion metrics to track

| Metric | What it reveals |
|--------|----------------|
| CTA click rate above fold | Headline + CTA relevance |
| Form start vs completion rate | Form friction |
| Scroll depth | Content engagement |
| Mobile vs desktop conversion split | Mobile UX quality |
| Drop-off in multi-step flows | Step-level friction |

---

## 9. Accessibility — WCAG POUR Framework

### The four principles

| Principle | Meaning | Key requirements |
|-----------|---------|-----------------|
| **Perceivable** | Content available to all senses | Alt text, captions, color contrast ≥ 4.5:1 |
| **Operable** | All functionality keyboard accessible | Tab order, focus visible, no keyboard traps |
| **Understandable** | Content and UI are clear | Plain language, consistent nav, helpful errors |
| **Robust** | Compatible with assistive tech | Semantic HTML, ARIA when needed, tested with screen readers |

### Accessibility in code

```typescript
// CORRECT — semantic, keyboard accessible, screen reader friendly
<form onSubmit={handleSubmit} aria-label="Contact form">
    <label htmlFor="email">Email address</label>
    <input
        id="email"
        name="email"
        type="email"
        required
        aria-required="true"
        aria-describedby={emailError ? 'email-error' : undefined}
        aria-invalid={!!emailError}
    />
    {emailError && (
        <p id="email-error" role="alert">
            {emailError}
        </p>
    )}
    <button type="submit" disabled={isPending} aria-busy={isPending}>
        {isPending ? 'Sending...' : 'Send message'}
    </button>
</form>

// WRONG — div used as interactive element
<div onClick={handleClick} style={{cursor: 'pointer'}}>Click me</div>
// Fix: use <button> or add role="button" + tabIndex={0} + onKeyDown handler
```

### Accessibility checklist

- All images have meaningful `alt` (empty `alt=""` for decorative images)
- All form inputs have associated `<label>` elements
- Interactive elements are keyboard reachable (Tab) and activatable (Enter/Space)
- Error messages use `role="alert"` or `aria-live="polite"`
- Color is never the ONLY differentiator (always add text or icon)
- Focus is managed when modals open (focus trapped inside, restored on close)
- Minimum touch target: 44×44px on mobile
- Color contrast: ≥ 4.5:1 for normal text, ≥ 3:1 for large text

---

## 10. Mobile-First Design

Mobile is not a smaller desktop. It is a different context: less attention, one hand,
touch input, variable connectivity, sun glare.

```
Mobile-first order:
1. Design for small screen first (375px width)
2. Add complexity for tablet (768px)
3. Enhance for desktop (1024px+)
```

### Mobile UX rules

- Minimum tap target: 44×44px with adequate spacing between targets
- Primary actions within thumb reach (bottom third of screen)
- Sticky CTA on conversion pages
- Forms: minimum fields, large inputs, mobile-appropriate input types
- Performance: mobile 4G is the baseline — not fiber
- Test on real devices, not browser DevTools emulation

### Touch interaction patterns

```css
/* Prevent accidental double-tap zoom on buttons */
button {
    touch-action: manipulation;
}

/* Comfortable touch targets */
.nav-item {
    min-height: 44px;
    min-width: 44px;
    display: flex;
    align-items: center;
}
```

---

## 11. Design Systems

### Design system components

| Layer | What it contains |
|-------|----------------|
| **Design tokens** | Colors, spacing, typography, shadows as variables |
| **Component library** | Atoms and molecules — Button, Input, Card, Modal |
| **Pattern library** | Organisms and templates — Forms, Navigation, Data tables |
| **Documentation** | Usage guidelines, props API, accessibility notes |
| **Figma library** | Design source of truth shared with designers |

### Design tokens — the foundation

```css
/* Semantic tokens — not raw values */
:root {
    /* Colors */
    --color-primary: #2563eb;
    --color-primary-hover: #1d4ed8;
    --color-error: #dc2626;
    --color-surface: #ffffff;
    --color-text-primary: #111827;
    --color-text-secondary: #6b7280;

    /* Spacing */
    --space-1: 0.25rem;   /* 4px */
    --space-2: 0.5rem;    /* 8px */
    --space-4: 1rem;      /* 16px */
    --space-8: 2rem;      /* 32px */

    /* Typography */
    --text-sm: 0.875rem;
    --text-base: 1rem;
    --text-lg: 1.125rem;
    --text-xl: 1.25rem;

    /* Theming — CSS variables enable dark mode trivially */
}

@media (prefers-color-scheme: dark) {
    :root {
        --color-surface: #111827;
        --color-text-primary: #f9fafb;
    }
}
```

---

## 12. Architecture Decision Guide

| Question | Answer |
|----------|--------|
| SEO critical? | SSR or SSG — never CSR for public pages |
| Content rarely changes? | SSG — build once, serve from CDN |
| Content updates frequently? | ISR — background regeneration |
| Personalized per user? | SSR or CSR — static cannot personalize |
| Highly interactive, behind auth? | CSR — full React SPA |
| Multiple teams, org scale? | Consider micro-frontends — otherwise monolith |
| Small team or startup? | Monolithic — micro-frontends are premature |
| Need offline support? | PWA with Service Workers |
| Global audience, performance critical? | Jamstack + CDN edge |
| Complex shared UI? | Design system with tokens + component library |
| Conversion page? | SSG/SSR + conversion UX principles above fold |
| Accessibility compliance? | WCAG 2.2 POUR from day one — not later |
| Mobile performance poor? | LCP optimization + mobile-first redesign |
| Forms converting poorly? | Error prevention + outcome-based CTA copy |

---

## 13. Common Anti-Patterns — Never Recommend These

| Anti-pattern | Correct approach |
|-------------|-----------------|
| CSR for SEO-critical public pages | SSR or SSG |
| Micro-frontends for small teams | Monolithic with clear feature modules |
| Viewport breakpoints as only responsive tool | Container queries + mobile-first |
| Placeholder-only form labels | Real `<label>` elements + placeholder as hint |
| `<div onClick>` for interactive elements | Semantic `<button>` or `<a>` |
| "Invalid input" error messages | Specific, instructive error copy |
| Trust signals only in footer | Trust signals near CTAs and decision points |
| Accessibility as post-launch checklist | POUR built into components from day one |
| Generic CTA copy ("Submit", "Click here") | Outcome-based copy ("Get My Free Quote") |
| Unoptimized images above fold | WebP/AVIF, explicit dimensions, preload LCP image |
| State management for server data | TanStack Query — server state is a cache |
| All pages use same rendering strategy | Hybrid — per-surface strategy selection |
| Micro-interactions after launch | Planned from design phase |

---

## Reference

- Core Web Vitals: https://web.dev/articles/vitals
- WCAG 2.2: https://www.w3.org/WAI/standards-guidelines/wcag/
- Nielsen's Heuristics: https://www.nngroup.com/articles/ten-usability-heuristics/
- Atomic Design: https://atomicdesign.bradfrost.com
- Astro (Islands): https://astro.build
- TanStack Query: https://tanstack.com/query
- Module Federation: https://webpack.js.org/concepts/module-federation/
