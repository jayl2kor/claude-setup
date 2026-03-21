# Language-Specific and Architecture Antipatterns — Full Reference

Frontend patterns and architecture-level antipatterns with complete examples.

---

## Frontend Antipatterns

### Prop Drilling

Passing a value through multiple intermediate components that don't use it, just to reach a deeply nested consumer.

```tsx
// BAD — currentUser passed through three unrelated intermediate components
function App() {
  const currentUser = useCurrentUser()
  return <Dashboard currentUser={currentUser} />
}
function Dashboard({ currentUser }) {
  return <Sidebar currentUser={currentUser} />
}
function Sidebar({ currentUser }) {
  return <UserAvatar currentUser={currentUser} />
}
```

```tsx
// GOOD — context for cross-cutting data; components only consume what they need
const UserContext = createContext<User | null>(null)

function App() {
  const currentUser = useCurrentUser()
  return (
    <UserContext.Provider value={currentUser}>
      <Dashboard />
    </UserContext.Provider>
  )
}

function UserAvatar() {
  const user = useContext(UserContext)  // consumes directly, no prop threading
  return <img src={user?.avatarUrl} alt={user?.name} />
}
```

**When context is overkill**: For 2-level drilling, props are fine. Reach for context only when the same value crosses 3+ component boundaries or is needed in truly unrelated subtrees.

**Alternatives**: Zustand, Jotai, or Redux for global state with complex update logic; React Query for server state.

---

### Over-Fetching (Loading Everything Upfront)

Requesting more data than the view needs wastes bandwidth, slows initial render, and inflates memory usage.

```typescript
// BAD — fetches all user fields even though the component only shows name + avatar
const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchFullUser })
// fetchFullUser returns 40 fields including billing history, permissions, preferences
```

```typescript
// GOOD — fetch only what the view needs
const { data: user } = useQuery({
  queryKey: ['user', 'summary'],
  queryFn: () => fetchUserSummary(),  // returns { id, name, avatarUrl } only
})

// Or with GraphQL: select only the fields the component needs
const USER_SUMMARY = gql`
  query UserSummary {
    me { id name avatarUrl }
  }
`
```

**Remediation strategies**:
- REST APIs: add view-specific endpoints (e.g., `/users/me/summary`) or support `?fields=id,name,avatarUrl`
- GraphQL: use fragment colocation — each component declares its own data needs
- tRPC: define separate procedures for list views vs. detail views

---

### Premature Optimization with useMemo/useCallback

Wrapping trivial computations in `useMemo` or `useCallback` adds overhead (closure allocation, dependency array comparison) that exceeds any savings for cheap operations.

```tsx
// BAD — memoizing a trivial value; the overhead of useMemo exceeds the savings
function PriceTag({ price }: { price: number }) {
  const formatted = useMemo(() => `$${price.toFixed(2)}`, [price])
  return <span>{formatted}</span>
}
```

```tsx
// GOOD — inline the trivial computation; use useMemo only after profiling shows a problem
function PriceTag({ price }: { price: number }) {
  return <span>${price.toFixed(2)}</span>
}
```

Reserve `useMemo`/`useCallback` for:
1. Expensive computations confirmed by the React DevTools Profiler (> 1 ms per render)
2. Referentially stable values passed as props to `React.memo`-wrapped children
3. Values used as dependencies in other hooks where reference equality matters

---

## Architecture Antipatterns

### Circular Dependencies

Module A imports B, B imports C, and C imports A. Results in import errors at startup, Jest "Cannot access X before initialization" errors, and modules that are impossible to test in isolation.

```
// BAD — module A imports B, B imports C, C imports A

order-service.ts  → imports → payment-service.ts
payment-service.ts → imports → notification-service.ts
notification-service.ts → imports → order-service.ts  ← cycle
```

```
// GOOD — dependency flows in one direction; shared types extracted
//        to a dependency-free shared module

shared/types.ts          (no imports from this project)
      ↑
notification-service.ts  (imports from shared only)
      ↑
payment-service.ts       (imports from shared + notification)
      ↑
order-service.ts         (imports from all below)
```

**Detection tools**:
- `eslint-plugin-import/no-cycle` — catches cycles at lint time
- `madge --circular src/` — visualizes the full import graph
- `depcruise --validate .dependency-cruiser.cjs src/` — enforces architectural rules as code

**Root cause**: Circular deps usually mean two modules have too tight a coupling. The fix is almost always to extract the shared concept into a third module that both can import.

---

### Anemic Domain Model

A domain model where classes are pure data bags (getters/setters only) and all business logic lives in service classes. Invariants cannot be enforced because any code can mutate any field.

```typescript
// BAD — Order is a data bag; nothing prevents illegal state transitions
class Order {
  public status: string
  public items: OrderItem[]
  public total: number
}

class OrderService {
  cancelOrder(order: Order) {
    order.status = 'cancelled'   // nothing prevents cancelling a delivered order
  }
}
```

```typescript
// GOOD — Order enforces its own invariants
class Order {
  private constructor(
    private readonly id: string,
    private items: OrderItem[],
    private status: 'pending' | 'confirmed' | 'shipped' | 'cancelled',
  ) {}

  static create(id: string, items: OrderItem[]): Order {
    if (items.length === 0) throw new Error('Order must have at least one item')
    return new Order(id, items, 'pending')
  }

  cancel(): void {
    if (this.status === 'shipped') throw new Error('Cannot cancel a shipped order')
    if (this.status === 'cancelled') throw new Error('Order is already cancelled')
    this.status = 'cancelled'
  }

  get total(): number {
    return this.items.reduce((sum, i) => sum + i.price * i.quantity, 0)
  }
}
```

**Detection signal**: Domain classes with only public fields and no methods, paired with large service classes that contain all the `if` statements for state transitions.

---

### Big Ball of Mud

A codebase with no recognizable architecture: modules import freely from one another, business logic is scattered across HTTP handlers, there are no enforced layers, and every change breaks something unexpected.

**Detection signals:**
- Changing a database column requires modifying 10+ files
- Tests require the full application to be running
- New developers cannot add a feature without asking where it goes
- Import graph is a dense mesh with no discernible hierarchy

**Remediation path** (do not attempt a full rewrite):

1. **Map the current import graph** with `madge` or `depcruise`
2. **Identify the core domain** — the nouns and verbs of the business
3. **Draw the target layer boundary** (domain → application → infrastructure → interface)
4. **Enforce it incrementally**: move one class per PR; add an architecture test (ArchUnit, dependency-cruiser) that fails on violations
5. **Strangle the old structure** — new code goes in the new structure; old code migrates when touched

**Target layer rules:**
- `domain/` — pure business logic; no framework imports, no database imports
- `application/` — use cases; orchestrates domain objects; no HTTP types
- `infrastructure/` — database, external APIs, file system
- `interface/` — HTTP handlers, CLI, gRPC; thin; delegates to application layer
