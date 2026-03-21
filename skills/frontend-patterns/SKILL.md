---
name: frontend-patterns
description: Activate when designing component architecture, choosing state management strategies, optimizing bundle size or runtime performance, implementing accessibility (ARIA, keyboard navigation, focus management), applying atomic design, or selecting between React/Vue/framework-agnostic patterns for UI development.
version: 1.0.0
---

# Frontend Development Patterns

Expert-level patterns for maintainable, performant, and accessible user interfaces. Patterns are illustrated in React/TypeScript but the principles apply to Vue, Svelte, and other component frameworks.

## When to Activate

- Designing component hierarchy and deciding composition strategies
- Choosing between local state, lifted state, context, or a global store
- Fixing performance regressions (excessive re-renders, large bundles, slow TTI)
- Implementing keyboard navigation, ARIA roles, or focus management
- Structuring forms with validation
- Applying atomic design to a component library
- Evaluating data-fetching strategies (SWR, React Query, server components)

---

## Component Design Principles

### Atomic Design

Organize components in five layers. The layer a component belongs to determines what it can depend on:

| Layer | Examples | Can depend on |
|---|---|---|
| **Atoms** | Button, Input, Badge, Icon | Nothing above itself |
| **Molecules** | SearchField, FormGroup, CardMedia | Atoms |
| **Organisms** | ProductCard, NavigationBar, CommentThread | Atoms, Molecules |
| **Templates** | DashboardLayout, CheckoutLayout | Organisms and below |
| **Pages** | HomePage, ProductPage | Templates, data-fetching hooks |

Never skip layers: an Organism must not be composed of Pages; an Atom must not import data-fetching hooks.

### Composition Over Configuration

Prefer composing small, single-purpose components over a single component with many boolean flags.

```tsx
// ❌ BAD — prop explosion, impossible to extend without changing the component
<Card showHeader showFooter headerTitle="Order #42" footerButtonLabel="Confirm" />

// ✅ GOOD — composable, each part is individually replaceable
<Card>
  <Card.Header>Order #42</Card.Header>
  <Card.Body>{/* content */}</Card.Body>
  <Card.Footer><Button variant="primary">Confirm</Button></Card.Footer>
</Card>
```

### Single Responsibility Per Component

Each component should answer one question: "What does this display?" If it also answers "Where does the data come from?" split it into a container (data) and a presentational component (UI).

---

## State Management: Escalation Ladder

Apply the simplest mechanism that solves the problem. Escalate only when simpler options fail.

```
Local useState
    ↓ state needs to be shared with sibling or distant child
Lift state / prop drilling (fine for 2–3 levels)
    ↓ prop drilling becomes painful (4+ levels, unrelated intermediaries)
React Context (for low-frequency updates: theme, locale, auth user)
    ↓ high-frequency updates or complex transitions cause too many re-renders
Zustand / Jotai / Redux Toolkit
    ↓ server state (async, cache invalidation, background refetch)
React Query / SWR / tRPC
```

Key rules:
- Use `useState` for UI-only state that nothing else cares about (toggle, accordion open/closed)
- Use Context only for low-frequency cross-cutting concerns; memoize its value to avoid cascading re-renders
- Use Zustand for global client state that changes frequently (cart, UI preferences)
- Never manage server state with `useState + useEffect` — use React Query or SWR instead

See [references/state-management.md](references/state-management.md) for full examples including Zustand store, React Query mutations, and optimistic updates.

---

## Performance Optimization

### Memoization Decision Table

| Situation | Tool | Notes |
|---|---|---|
| Expensive pure computation | `useMemo` | Only when profiling confirms cost |
| Stable function ref for memoized child | `useCallback` | Pair with `React.memo` on child |
| Skip re-render of expensive presentational component | `React.memo` | Add after profiling, not by default |
| Trivial computation (`count * 2`) | Nothing | Memoization adds overhead |

### Code Splitting — Every Route Must Be Lazy

```tsx
const CheckoutPage   = lazy(() => import('./pages/CheckoutPage'))
const AdminDashboard = lazy(() => import('./pages/AdminDashboard'))
```

Heavy widgets not needed on first paint should also be split at the component level.

### Virtualization for Long Lists

Never render thousands of DOM nodes. Use `@tanstack/react-virtual` to window large lists.

See [references/performance.md](references/performance.md) for full memoization examples, code splitting patterns, and the complete virtualization implementation.

---

## Accessibility — Key Rules

- Icon-only buttons require `aria-label`; decorative icons get `aria-hidden="true"`
- Async updates require `role="status" aria-live="polite"` so screen readers announce changes
- Error messages must be linked to their input via `aria-describedby` and `aria-invalid`
- Every interactive element must be keyboard-reachable with visible focus styles
- Custom widgets (listbox, combobox, tabs) must implement the ARIA keyboard pattern for that widget type
- Modals must trap focus while open and restore focus to the trigger on close
- Never use `onClick` on a `<div>` — use `<button>` which is keyboard-accessible by default

See [references/accessibility.md](references/accessibility.md) for full ARIA patterns, focus trap implementation, keyboard navigation, and form accessibility.

---

## Form Patterns

Use `react-hook-form` + `zod` for controlled forms with validation:

```tsx
const schema = z.object({
  email:    z.string().email('Enter a valid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
})
type FormValues = z.infer<typeof schema>

function LoginForm({ onSubmit }: { onSubmit: (v: FormValues) => Promise<void> }) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm<FormValues>({ resolver: zodResolver(schema) })

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" {...register('email')}
        aria-describedby={errors.email ? 'email-error' : undefined}
        aria-invalid={!!errors.email} />
      {errors.email && <span id="email-error" role="alert">{errors.email.message}</span>}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Signing in…' : 'Sign in'}
      </button>
    </form>
  )
}
```

Connect validation errors to inputs via `aria-describedby` and `role="alert"` on error spans. Use `noValidate` on the form element to disable native browser validation in favor of the schema. See [references/accessibility.md](references/accessibility.md) for the full form accessibility checklist.

---

## Custom Hook Patterns

Encapsulate complex logic in hooks, never in components:

- `useDebounce(value, delayMs)` — prevents firing on every keystroke; returns debounced value
- `useInView(ref, options)` — wraps `IntersectionObserver`; returns boolean; use for lazy-loading images

```tsx
function useDebounce<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value)
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delayMs)
    return () => clearTimeout(timer)
  }, [value, delayMs])
  return debounced
}
```

See [references/state-management.md](references/state-management.md) for the full `useDebounce` and `useInView` implementations with usage examples.

---

## Common Mistakes to Avoid

**State management:**
- Putting server state in a global store — use React Query/SWR instead
- Creating a single massive global store — split by domain (cart, auth, ui)
- Deriving state in render without memoization — use `useMemo`

**Components:**
- Key prop using array index for reorderable or filterable lists — use stable, unique IDs
- Spreading all props onto a DOM element (`<div {...props}>`) — unknown attributes cause warnings
- Putting business logic inside JSX expressions — extract to variables or hooks before the return

**Performance:**
- Wrapping every function in `useCallback` and every value in `useMemo` "just in case" — measure first
- Importing entire libraries (`import _ from 'lodash'`) — use named imports
- Loading all page JavaScript synchronously — every route must be lazy-loaded

**Accessibility:**
- Using `onClick` on a `<div>` without `role`, `tabIndex`, and keyboard handler
- Relying on color alone to convey meaning — always pair with text or shape
- Auto-playing audio or video without a pause control (violates WCAG 2.1 SC 1.4.2)
- Missing `aria-live` regions for async feedback

---

## References

- [references/state-management.md](references/state-management.md) — Full state escalation examples, Zustand store, React Query mutations, optimistic updates
- [references/performance.md](references/performance.md) — Full memoization examples, code splitting patterns, virtualization with @tanstack/react-virtual
- [references/accessibility.md](references/accessibility.md) — Full ARIA patterns, focus trap, keyboard navigation, form accessibility
