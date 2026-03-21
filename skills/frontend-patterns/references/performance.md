# Performance Optimization — Full Reference

Full examples for memoization, code splitting, and virtualization. See [../SKILL.md](../SKILL.md) for the decision summary.

---

## Memoization — Apply Judiciously, Not by Default

### useMemo for expensive pure computations

```tsx
// ✅ useMemo for expensive pure computations
const sortedAndFiltered = useMemo(
  () => products.filter(p => p.inStock).sort((a, b) => a.price - b.price),
  [products]   // only recompute when products changes
)

// ❌ BAD — memoizing a trivial computation wastes memory for no gain
const doubled = useMemo(() => count * 2, [count])
```

### useCallback to stabilize function references

```tsx
// ✅ useCallback to stabilize function references passed to memoized children
const handleAddToCart = useCallback((id: string) => {
  mutation.mutate(id)
}, [mutation])

// ❌ BAD — new function reference on every render defeats React.memo on the child
function Parent() {
  return <MemoizedChild onClick={() => doSomething()} />  // inline arrow = new ref always
}
```

### React.memo to skip re-renders

```tsx
// ✅ React.memo to skip re-render of expensive presentational components
const ProductCard = React.memo(function ProductCard({ product }: Props) {
  return (
    <div>
      <img src={product.imageUrl} alt={product.name} />
      <h3>{product.name}</h3>
      <span>${product.price}</span>
    </div>
  )
})
// Only add React.memo after profiling confirms unnecessary re-renders are a problem.
// Use React DevTools Profiler to measure before and after.
```

### Profiling workflow

1. Open React DevTools → Profiler tab
2. Record a user interaction that feels slow
3. Look for components with long render bars or repeated renders with no prop changes
4. Add `React.memo`, `useMemo`, or `useCallback` only where profiling confirms benefit
5. Re-profile to verify the improvement

---

## Code Splitting and Lazy Loading

### Route-level splitting — every route must be lazy

```tsx
// ✅ Route-level code splitting — load route bundle only when navigated to
const CheckoutPage   = lazy(() => import('./pages/CheckoutPage'))
const AdminDashboard = lazy(() => import('./pages/AdminDashboard'))

function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <Routes>
        <Route path="/checkout" element={<CheckoutPage />} />
        <Route path="/admin"    element={<AdminDashboard />} />
      </Routes>
    </Suspense>
  )
}
```

### Component-level splitting for heavy widgets

```tsx
// ✅ Component-level splitting for heavy widgets not needed on first paint
const RichTextEditor = lazy(() => import('./components/RichTextEditor'))
const HeavyChart     = lazy(() => import('./components/HeavyChart'))

function PostEditor() {
  const [showEditor, setShowEditor] = useState(false)
  return showEditor
    ? (
      <Suspense fallback={<div>Loading editor…</div>}>
        <RichTextEditor />
      </Suspense>
    )
    : <button onClick={() => setShowEditor(true)}>Write a post</button>
}
```

### Bundle analysis

```bash
# Vite
npx vite build --mode analyze   # generates stats.html

# Create React App / webpack
npx source-map-explorer 'build/static/js/*.js'

# Next.js
ANALYZE=true next build
```

Common bundle mistakes to catch with analysis:
- `import _ from 'lodash'` — ships entire lodash; use `import debounce from 'lodash/debounce'`
- `import * as Icons from '@heroicons/react'` — ships all icons; import individually
- Large polyfills for features already supported by your browser targets

---

## Virtualization for Long Lists

Never render thousands of DOM nodes. Use `@tanstack/react-virtual` to window large lists so only visible rows are mounted:

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualProductList({ products }: { products: Product[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: products.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80,   // estimated row height in px
    overscan: 5,              // render 5 extra rows above/below for smooth scroll
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflowY: 'auto' }}>
      {/* Total height spacer — keeps scrollbar accurate */}
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              transform: `translateY(${virtualRow.start}px)`,
              width: '100%',
            }}
          >
            <ProductRow product={products[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

### Variable-height rows

When row heights differ, measure them after render:

```tsx
const virtualizer = useVirtualizer({
  count: items.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 50,   // initial estimate
  measureElement: (el) => el?.getBoundingClientRect().height,   // actual measurement
})
```

### When to virtualize

- Lists with > 100 items that are visible simultaneously
- Infinite scroll feeds
- Data grids with many rows

Do not virtualize lists that are short, paginated, or where items have complex focus management requirements not yet supported by the virtualizer.

---

## Lazy Loading Images

```tsx
function useInView(ref: RefObject<Element>, options?: IntersectionObserverInit) {
  const [inView, setInView] = useState(false)
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => setInView(entry.isIntersecting),
      options
    )
    if (ref.current) observer.observe(ref.current)
    return () => observer.disconnect()
  }, [ref, options])
  return inView
}

function LazyImage({ src, alt }: { src: string; alt: string }) {
  const ref = useRef<HTMLImageElement>(null)
  const inView = useInView(ref, { rootMargin: '200px' })   // start loading 200px before visible
  return <img ref={ref} src={inView ? src : undefined} alt={alt} loading="lazy" />
}
```

For most cases, the native `loading="lazy"` attribute on `<img>` is sufficient without a custom hook. Use the hook when you need programmatic control or the browser support bar is lower.
