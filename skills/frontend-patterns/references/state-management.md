# State Management — Full Reference

Full examples for the state escalation ladder. See [../SKILL.md](../SKILL.md) for the decision summary.

---

## Local State — the right default

```tsx
// ✅ Use useState for UI-only state that nothing else cares about
function Accordion({ title, children }: AccordionProps) {
  const [isOpen, setIsOpen] = useState(false)
  return (
    <div>
      <button onClick={() => setIsOpen(v => !v)}>{title}</button>
      {isOpen && <div>{children}</div>}
    </div>
  )
}
```

---

## Lifting State

When two siblings need to share state, lift it to their nearest common ancestor and pass it down as props. Prop drilling is acceptable for 2–3 levels. Beyond that, reach for Context or a store.

```tsx
// Parent owns the state; both children receive it as props
function SearchPage() {
  const [query, setQuery] = useState('')
  return (
    <>
      <SearchBar query={query} onQueryChange={setQuery} />
      <SearchResults query={query} />
    </>
  )
}
```

---

## React Context — for low-frequency cross-cutting concerns

```tsx
// ✅ Theme, locale, and auth user change rarely — Context is appropriate
const ThemeContext = createContext<Theme>('light')

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light')
  // Memoize the value to avoid re-rendering all consumers on every parent render
  const value = useMemo(() => ({ theme, setTheme }), [theme])
  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
}

export function useTheme() {
  return useContext(ThemeContext)
}

// ❌ BAD — using Context for a frequently changing value (e.g., cursor position)
// Every consumer re-renders on every mouse move. Use a dedicated store instead.
```

---

## Zustand — for global client state with high update frequency

```tsx
// ✅ Cart state changes frequently and is needed in many unrelated components
import { create } from 'zustand'

interface CartStore {
  items: CartItem[]
  addItem: (item: CartItem) => void
  removeItem: (id: string) => void
  total: number
}

export const useCartStore = create<CartStore>((set, get) => ({
  items: [],
  total: 0,
  addItem: (item) => set(state => {
    const items = [...state.items, item]
    return { items, total: items.reduce((s, i) => s + i.price, 0) }
  }),
  removeItem: (id) => set(state => {
    const items = state.items.filter(i => i.id !== id)
    return { items, total: items.reduce((s, i) => s + i.price, 0) }
  }),
}))

// Consumers subscribe only to the slice they need — no unnecessary re-renders
function CartBadge() {
  const count = useCartStore(state => state.items.length)
  return <span>{count}</span>
}

function CartTotal() {
  const total = useCartStore(state => state.total)
  return <span>${total.toFixed(2)}</span>
}
```

---

## React Query — for server state

Server state (remote data) has fundamentally different requirements from client state: caching, background refetch, stale-while-revalidate. Do not manage it with `useState + useEffect`.

```tsx
// ❌ BAD — manual server state management
const [products, setProducts] = useState([])
const [loading, setLoading] = useState(true)
const [error, setError] = useState(null)
useEffect(() => {
  fetch('/api/products').then(r => r.json()).then(setProducts).catch(setError).finally(...)
}, [])

// ✅ GOOD — React Query handles caching, deduplication, background refresh
const { data: products, isLoading, error } = useQuery({
  queryKey: ['products', { category }],
  queryFn: () => fetchProducts(category),
  staleTime: 60_000,       // don't refetch if data is under 1 min old
})
```

---

## Optimistic Update Mutation Pattern

```tsx
// Mutations with optimistic updates
const mutation = useMutation({
  mutationFn: addToCart,
  onMutate: async (newItem) => {
    // Cancel any in-flight refetches so they don't overwrite our optimistic update
    await queryClient.cancelQueries({ queryKey: ['cart'] })
    // Snapshot the previous value for rollback
    const previous = queryClient.getQueryData(['cart'])
    // Optimistically update the cache
    queryClient.setQueryData(['cart'], old => [...old, newItem])
    return { previous }
  },
  onError: (_err, _newItem, context) => {
    // Roll back on error
    queryClient.setQueryData(['cart'], context?.previous)
  },
  onSettled: () => {
    // Always refetch after error or success to sync with the server
    queryClient.invalidateQueries({ queryKey: ['cart'] })
  },
})

// Usage
function AddToCartButton({ item }: { item: CartItem }) {
  return (
    <button
      onClick={() => mutation.mutate(item)}
      disabled={mutation.isPending}
    >
      {mutation.isPending ? 'Adding…' : 'Add to cart'}
    </button>
  )
}
```

---

## Custom Hook: useDebounce

Encapsulate debounce logic in a hook rather than duplicating timeouts in components:

```tsx
function useDebounce<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value)
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delayMs)
    return () => clearTimeout(timer)
  }, [value, delayMs])
  return debounced
}

// Usage — search fires only after 300 ms of inactivity
function SearchBar() {
  const [query, setQuery] = useState('')
  const debouncedQuery = useDebounce(query, 300)
  const { data } = useQuery({
    queryKey: ['search', debouncedQuery],
    queryFn: () => search(debouncedQuery),
    enabled: debouncedQuery.length > 1,
  })
  return <input value={query} onChange={e => setQuery(e.target.value)} />
}
```
