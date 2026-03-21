# Database Patterns — Full Reference

Full examples for N+1 prevention, connection pooling, migration rules, and the outbox pattern. See [../SKILL.md](../SKILL.md) for the summary.

---

## N+1 Problem and Batch Loading

The N+1 problem occurs when you run 1 query to fetch a list, then N additional queries — one per row — to fetch related data. Always batch related data in a second query or use a JOIN.

### Three-tier fix

```typescript
// ❌ BAD — N+1: 1 query for orders, then 1 query per order for user (51 queries for 50 rows)
const orders = await db.query('SELECT * FROM orders LIMIT 50')
for (const order of orders) {
  order.user = await db.query('SELECT * FROM users WHERE id = $1', [order.userId])
}

// ✅ GOOD — 2 queries total, regardless of result set size
const orders = await db.query('SELECT * FROM orders LIMIT 50')
const userIds = [...new Set(orders.map(o => o.userId))]
const users = await db.query('SELECT * FROM users WHERE id = ANY($1)', [userIds])
const userMap = new Map(users.map(u => [u.id, u]))
orders.forEach(o => { o.user = userMap.get(o.userId) })

// ✅ BEST — JOIN when the relationship is always needed
const orders = await db.query(`
  SELECT
    o.*,
    u.name  AS user_name,
    u.email AS user_email
  FROM orders o
  JOIN users u ON u.id = o.user_id
  LIMIT 50
`)
```

### When to use batch vs JOIN

| Situation | Approach |
|---|---|
| Relationship is always needed in the response | JOIN |
| Relationship is conditionally needed | Batch (2-query) |
| Multiple independent relationships on the same root | Multiple batches |
| Nested relationships (user → orders → items) | Batch each level separately |

### DataLoader pattern (for GraphQL or repeated N+1 across resolvers)

```typescript
import DataLoader from 'dataloader'

// Batch function: receives an array of keys, returns an array of values in the same order
const userLoader = new DataLoader<string, User>(async (userIds) => {
  const users = await db.query(
    'SELECT * FROM users WHERE id = ANY($1)',
    [userIds]
  )
  const userMap = new Map(users.map(u => [u.id, u]))
  // Return in the same order as the input keys — DataLoader requires this
  return userIds.map(id => userMap.get(id) ?? new Error(`User ${id} not found`))
})

// Usage — each call is batched automatically within the same tick
const user = await userLoader.load(order.userId)
```

---

## Connection Pooling

Never create a new database connection per request. Creating a connection is expensive (TLS handshake, authentication, memory allocation on the DB server). Use a pool sized to your database's `max_connections` minus headroom for admin tools and migrations.

```typescript
// ✅ Single pool, created once at application startup, shared for its lifetime
import { Pool } from 'pg'

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                   // Do not exceed pg max_connections minus headroom
  idleTimeoutMillis: 30_000, // Release idle connections after 30 s
  connectionTimeoutMillis: 5_000,  // Fail fast if pool is exhausted
})

// ✅ Wrapper that guarantees release even on error
export async function query<T>(sql: string, params?: unknown[]): Promise<T[]> {
  const client = await pool.connect()
  try {
    const result = await client.query(sql, params)
    return result.rows
  } finally {
    client.release()   // Always release — put this in finally, never in try
  }
}

// ✅ Transaction helper — acquire once, run multiple queries atomically
export async function withTransaction<T>(
  fn: (client: PoolClient) => Promise<T>
): Promise<T> {
  const client = await pool.connect()
  try {
    await client.query('BEGIN')
    const result = await fn(client)
    await client.query('COMMIT')
    return result
  } catch (err) {
    await client.query('ROLLBACK')
    throw err
  } finally {
    client.release()
  }
}
```

### Pool sizing guidance

```
pool_size = (core_count * 2) + effective_spindle_count

For a 4-core server with SSDs: pool_size ≈ 9
For pg max_connections = 100:  leave ~20 for admin/migrations → app pool max = 80
```

If you have multiple application instances, divide the budget across them:
- 4 instances, 80 connection budget → max 20 per instance

---

## Database Migrations

Apply migrations in CI before deploying application code. Never run migrations inside application startup — that causes race conditions in multi-instance deploys.

```
migrations/
  0001_create_users.sql
  0002_add_users_email_index.sql
  0003_create_orders.sql
  0004_add_orders_status_index.sql

# Run in CI pipeline before deploy — not inside the application
npx db-migrate up         # node-db-migrate
flyway migrate            # Flyway
alembic upgrade head      # Python/SQLAlchemy
```

### Migration rules

**Always backward-compatible.** The old application version must still work while migrations are applied. This means:

1. **Adding a column**: Add as `NULL` or with a default. Populate in a subsequent migration. Add `NOT NULL` only after all rows are populated.

```sql
-- ✅ Step 1: Add nullable
ALTER TABLE orders ADD COLUMN shipped_at TIMESTAMPTZ;

-- (Deploy application that writes shipped_at)

-- ✅ Step 2: Backfill and then add constraint (separate migration, separate deploy)
UPDATE orders SET shipped_at = updated_at WHERE status = 'shipped';
ALTER TABLE orders ALTER COLUMN shipped_at SET NOT NULL;
```

2. **Renaming a column**: Never rename in one step. Add the new column, backfill, update application, then remove the old column in a later migration.

3. **Dropping a column**: Remove application references first, deploy, then drop the column in a subsequent migration.

4. **Every `up` must have a `down`**: The rollback migration must restore the schema to its prior state.

5. **Lock the migration table**: Prevent concurrent migration runs in multi-instance deploys. Most tools do this automatically; verify it for your tool.

### Indexes on large tables

Create indexes concurrently to avoid locking the table:

```sql
-- ✅ Non-blocking index creation on Postgres
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
CREATE INDEX CONCURRENTLY idx_orders_status_created ON orders(status, created_at DESC);

-- ❌ This locks the table and blocks writes while indexing
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

---

## Outbox Pattern

The dual-write problem: if you write to the database and then publish a message to a queue/event bus, the system can crash between the two operations, leaving them inconsistent.

**Wrong approach:**
```typescript
// ❌ BAD — dual write: DB commit and event publish are not atomic
await orderRepo.save(order)
await eventBus.publish(new OrderPlaced(order))  // crash here = event never published
```

**Outbox pattern:** Write the event to an `outbox` table in the same database transaction as the business write. A separate relay process reads unpublished events and publishes them. The database transaction guarantees atomicity.

```typescript
// ✅ Write order and outbox event in one transaction
await withTransaction(async (client) => {
  await client.query(
    'INSERT INTO orders (id, status, total) VALUES ($1, $2, $3)',
    [order.id, order.status, order.total]
  )
  await client.query(
    `INSERT INTO outbox (id, aggregate_id, event_type, payload, created_at)
     VALUES ($1, $2, $3, $4, now())`,
    [
      uuid(),
      order.id,
      'OrderPlaced',
      JSON.stringify({ orderId: order.id, total: order.total }),
    ]
  )
})

// Separate relay process (runs on a schedule or triggered by LISTEN/NOTIFY)
async function publishPendingEvents(): Promise<void> {
  const events = await db.query(
    `SELECT * FROM outbox WHERE published_at IS NULL
     ORDER BY created_at ASC
     LIMIT 100
     FOR UPDATE SKIP LOCKED`   // prevents duplicate processing in multi-instance
  )

  for (const event of events) {
    await eventBus.publish({ type: event.event_type, payload: event.payload })
    await db.query(
      'UPDATE outbox SET published_at = now() WHERE id = $1',
      [event.id]
    )
  }
}
```

### Outbox table schema

```sql
CREATE TABLE outbox (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_id  UUID NOT NULL,
  event_type    TEXT NOT NULL,
  payload       JSONB NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at  TIMESTAMPTZ,                -- NULL = pending
  failed_at     TIMESTAMPTZ,
  retry_count   INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_outbox_pending ON outbox(created_at)
  WHERE published_at IS NULL;
```

### Alternatives to a custom outbox

- **Debezium** (CDC): reads the Postgres WAL directly and publishes events; no outbox table needed in the application
- **Transactional outbox with Kafka Connect**: same principle, automated relay
- Use a custom outbox only when you don't have infrastructure for CDC
