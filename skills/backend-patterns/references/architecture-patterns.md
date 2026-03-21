# Architecture Patterns — Full Reference

Full TypeScript examples for clean/hexagonal architecture, CQRS, event sourcing, and API versioning. See [../SKILL.md](../SKILL.md) for the decision summaries.

---

## Clean / Hexagonal Architecture — Full TypeScript Examples

### Domain entity with invariants

```typescript
// ✅ Domain layer — pure, testable, no framework imports
export class Order {
  private constructor(
    public readonly id: string,
    private items: OrderItem[],
    private status: OrderStatus
  ) {}

  static create(id: string, items: OrderItem[]): Order {
    if (items.length === 0) throw new Error('Order must have at least one item')
    return new Order(id, items, 'pending')
  }

  confirm(): void {
    if (this.status !== 'pending') throw new Error('Only pending orders can be confirmed')
    this.status = 'confirmed'
  }

  cancel(reason: string): void {
    if (this.status === 'cancelled') throw new Error('Order is already cancelled')
    this.status = 'cancelled'
  }

  get total(): number {
    return this.items.reduce((sum, i) => sum + i.price * i.quantity, 0)
  }
}
```

### Port (interface) — domain defines what it needs

```typescript
// ✅ Port lives in domain/ — domain dictates the contract
export interface OrderRepository {
  findById(id: string): Promise<Order | null>
  save(order: Order): Promise<void>
  findByUserId(userId: string): Promise<Order[]>
}

export interface PaymentGateway {
  charge(orderId: string, amount: number, currency: string): Promise<PaymentResult>
  refund(paymentId: string): Promise<void>
}
```

### Adapter (infrastructure) — implements the port

```typescript
// ✅ Adapter lives in infrastructure/ — depends on both the port and the DB driver
export class PostgresOrderRepository implements OrderRepository {
  constructor(private db: Pool) {}

  async findById(id: string): Promise<Order | null> {
    const result = await this.db.query(
      'SELECT * FROM orders WHERE id = $1',
      [id]
    )
    return result.rows[0] ? this.toDomain(result.rows[0]) : null
  }

  async save(order: Order): Promise<void> {
    await this.db.query(
      `INSERT INTO orders (id, status, total)
       VALUES ($1, $2, $3)
       ON CONFLICT (id) DO UPDATE SET status = $2, total = $3`,
      [order.id, order.status, order.total]
    )
  }

  async findByUserId(userId: string): Promise<Order[]> {
    const result = await this.db.query(
      'SELECT * FROM orders WHERE user_id = $1',
      [userId]
    )
    return result.rows.map(row => this.toDomain(row))
  }

  private toDomain(row: any): Order {
    return Order.reconstitute(row.id, row.items, row.status)
  }
}
```

### Use case (application layer)

```typescript
// ✅ Application layer — orchestrates domain, has no HTTP or DB imports
export class CreateOrderUseCase {
  constructor(
    private orderRepo: OrderRepository,
    private paymentGateway: PaymentGateway,
    private eventBus: EventBus
  ) {}

  async execute(dto: CreateOrderDto): Promise<string> {
    const order = Order.create(uuid(), dto.items)
    await this.orderRepo.save(order)
    await this.eventBus.publish(new OrderCreatedEvent(order.id, order.total))
    return order.id
  }
}
```

### Controller (interfaces layer)

```typescript
// ✅ Controller is thin — validates input, calls use case, maps result to HTTP
export class OrderController {
  constructor(private createOrder: CreateOrderUseCase) {}

  async handleCreate(req: Request, res: Response): Promise<void> {
    const dto = CreateOrderDto.parse(req.body)   // throws on invalid input
    const orderId = await this.createOrder.execute(dto)
    res.status(201).json({ id: orderId })
  }
}
```

---

## CQRS (Command Query Responsibility Segregation)

Separate the write model (commands) from the read model (queries). Apply when read and write patterns diverge significantly — not as a default.

### When to use CQRS

| Apply CQRS when… | Avoid CQRS when… |
|---|---|
| Read and write loads differ by an order of magnitude | Simple CRUD with symmetric read/write patterns |
| Read queries require complex joins that pollute the write model | Team is small and added complexity has no payoff |
| Different teams own reads vs. writes | The domain is still exploratory |

### Command side

```typescript
// Command side — optimized for writes and invariant enforcement
class PlaceOrderCommand {
  constructor(
    public readonly userId: string,
    public readonly items: { skuId: string; quantity: number }[]
  ) {}
}

class PlaceOrderHandler {
  constructor(
    private inventoryRepo: InventoryRepository,
    private orderRepo: OrderRepository,
    private eventBus: EventBus
  ) {}

  async handle(cmd: PlaceOrderCommand): Promise<string> {
    const inventory = await this.inventoryRepo.checkAvailability(cmd.items)
    if (!inventory.allAvailable) throw new Error('Items out of stock')

    const order = Order.create(uuid(), cmd.items)
    await this.orderRepo.save(order)
    await this.eventBus.publish(new OrderPlaced(order))
    return order.id
  }
}
```

### Query side

```typescript
// Query side — optimized for reads, can be a denormalized view
class GetOrderSummaryQuery {
  constructor(public readonly orderId: string) {}
}

// Read model can be a separate DB, Elasticsearch, or a materialized view
interface OrderSummaryReadModel {
  id: string
  customerName: string
  totalAmount: number
  itemCount: number
  status: string
  placedAt: Date
}

class GetOrderSummaryHandler {
  constructor(private readDb: ReadDatabase) {}

  async handle(query: GetOrderSummaryQuery): Promise<OrderSummaryReadModel | null> {
    return this.readDb.query(
      'SELECT * FROM order_summaries WHERE id = $1',
      [query.orderId]
    )
  }
}
```

---

## Event Sourcing

Store state as an immutable sequence of events rather than current state. The current state is always derived by replaying events.

### When to use event sourcing

| Use event sourcing when you need… | Avoid event sourcing when… |
|---|---|
| Complete audit trail (finance, healthcare, compliance) | Simple CRUD (overkill and complex infrastructure) |
| Time-travel queries ("what was the state on Dec 1?") | Team is unfamiliar — steep learning curve |
| Event-driven integration (projections, external subscribers) | You don't have a genuine audit or replay requirement |

### Events, reducer, and reconstruction

```typescript
// Events are facts, past tense, immutable
type OrderEvent =
  | { type: 'OrderPlaced';    orderId: string; items: OrderItem[]; at: Date }
  | { type: 'OrderConfirmed'; orderId: string; at: Date }
  | { type: 'OrderCancelled'; orderId: string; reason: string; at: Date }

interface OrderState {
  status: 'pending' | 'confirmed' | 'cancelled'
  items: OrderItem[]
  cancelReason?: string
}

const initialOrderState: OrderState = { status: 'pending', items: [] }

// Reducer rebuilds state from events — pure function, easy to test
function applyOrderEvent(state: OrderState, event: OrderEvent): OrderState {
  switch (event.type) {
    case 'OrderPlaced':
      return { ...state, items: event.items, status: 'pending' }
    case 'OrderConfirmed':
      return { ...state, status: 'confirmed' }
    case 'OrderCancelled':
      return { ...state, status: 'cancelled', cancelReason: event.reason }
  }
}

// Reconstruct current state by replaying all events
async function loadOrder(orderId: string): Promise<OrderState> {
  const events = await eventStore.load(orderId)
  return events.reduce(applyOrderEvent, initialOrderState)
}

// Append a new event — never mutate past events
async function cancelOrder(orderId: string, reason: string): Promise<void> {
  const state = await loadOrder(orderId)
  if (state.status === 'cancelled') throw new Error('Already cancelled')

  await eventStore.append(orderId, {
    type: 'OrderCancelled',
    orderId,
    reason,
    at: new Date(),
  })
}
```

---

## API Versioning Strategies

Choose one strategy per project and apply it consistently.

| Strategy | Example | When to use |
|---|---|---|
| URI path | `/v1/users` | Public APIs, clearest client experience |
| Header | `Accept: application/vnd.api+json; version=2` | Content negotiation, keeps URLs stable |
| Query param | `/users?version=2` | Quick prototypes only — avoid in production |

```
# ✅ URI versioning — most discoverable
GET /v1/orders/:id
GET /v2/orders/:id   # v2 adds payment breakdown field

# ❌ Never mix strategies within one API
GET /v1/orders/:id
GET /orders/:id?version=2
```

### Deprecation lifecycle

Never remove a version without a deprecation header and a sunset period:

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jan 2027 00:00:00 GMT
Link: </v2/orders>; rel="successor-version"
```

Minimum sunset period recommendations:
- Internal APIs: 3 months
- Partner APIs: 6 months
- Public APIs: 12 months

### Backward-compatible changes (safe to make without a version bump)

- Adding a new optional field to a response
- Adding a new optional request parameter
- Adding a new endpoint

### Breaking changes (require a new version)

- Removing or renaming a field
- Changing a field's type
- Changing error response structure
- Changing authentication requirements
