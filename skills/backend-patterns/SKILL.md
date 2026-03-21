---
name: backend-patterns
description: Activate when designing REST APIs, selecting architecture patterns (clean/hexagonal/microservices/monolith), applying CQRS or event sourcing, optimizing database queries, managing migrations, implementing 12-factor app principles, or structuring service/repository layers for server-side applications.
version: 1.0.0
---

# Backend Development Patterns

Expert-level patterns for scalable, maintainable server-side applications. These patterns apply across Node.js, Python, Go, Java, and other backend stacks.

## When to Activate

- Designing or reviewing REST API endpoints and versioning strategies
- Choosing between monolith, modular monolith, or microservices
- Implementing repository, service, or controller layers (clean/hexagonal architecture)
- Applying CQRS or event sourcing to a domain
- Fixing N+1 queries, adding indexes, or structuring database migrations
- Implementing connection pooling, caching layers, or background queues
- Structuring error handling, validation, and logging for production APIs

---

## REST API Design: Richardson Maturity Levels

The Richardson Maturity Model defines four levels of REST maturity. Aim for Level 2 minimum; Level 3 only when clients must not hardcode URLs.

| Level | Description | Example |
|---|---|---|
| **0** | Single URI, single verb (RPC-over-HTTP) | `POST /api { "method": "getUser" }` |
| **1** | Resources exist but wrong HTTP verbs | `POST /users/deleteById` |
| **2** | Correct verbs + status codes | `DELETE /users/:id → 204` |
| **3** | Hypermedia (HATEOAS) | Response includes `_links` to related resources |

Never ship Level 0. Level 3 (HATEOAS) is rarely worth the overhead for internal APIs.

**Consistent Error Responses** — adopt RFC 7807 Problem Details:

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "The request body contains invalid fields.",
  "errors": [
    { "field": "quantity", "message": "Must be a positive integer" }
  ]
}
```

Never return raw exception messages or stack traces in production responses.

### API Versioning — Quick Reference

| Strategy | Example | When to use |
|---|---|---|
| URI path | `/v1/users` | Public APIs, clearest client experience |
| Header | `Accept: application/vnd.api+json; version=2` | Content negotiation, keeps URLs stable |
| Query param | `/users?version=2` | Quick prototypes only — avoid in production |

Choose one strategy and apply it consistently. Never mix strategies within one API. Signal deprecation with `Deprecation: true` and `Sunset:` headers before removing a version.

See [references/architecture-patterns.md](references/architecture-patterns.md) for backward-compatible vs breaking change rules and minimum sunset period recommendations.

---

## Architecture: Monolith vs Microservices

| Criterion | Choose Monolith | Choose Microservice |
|---|---|---|
| Team size | < 8 engineers | Independent teams with clear ownership |
| Domain clarity | Boundaries still evolving | Domain is proven and stable |
| Scaling needs | Uniform across the system | One module has a radically different profile |
| Data model | Cross-cutting joins are common | Module data is never joined with others |
| Latency | Tight budget — avoid network hops | Independent deployment unblocks a team |

**Start with a modular monolith.** Strong module boundaries enforced by code, single deployable unit. Extract a service only when a specific module needs independent scaling and the domain boundary is proven.

```
src/
  modules/
    orders/       # owns its DB schema, service, and routes
      domain/
      application/
      infrastructure/
      routes.ts
    inventory/    # cannot import from orders/ directly — use events
    payments/
  shared/         # only truly cross-cutting utilities (logger, config)
```

**Decision test**: If you need a distributed transaction to complete a user-visible operation, your service boundaries are wrong. Redesign first.

---

## Clean / Hexagonal Architecture — Directory Layout

Structure code so the domain has zero framework dependencies. Frameworks (Express, Fastify, Next.js) are infrastructure, not the core.

```
src/
  domain/            # Pure business logic — no imports from outside this folder
    Order.ts         # Entity with invariants
    OrderRepository.ts  # Interface (port)
    OrderService.ts  # Use cases
  application/       # Orchestrates domain, no framework code
    CreateOrderUseCase.ts
    CreateOrderDto.ts
  infrastructure/    # Implements ports (adapters)
    PostgresOrderRepository.ts
    StripePaymentGateway.ts
  interfaces/        # HTTP, CLI, event handlers — thin layer, no business logic
    http/
      OrderController.ts
      OrderRouter.ts
```

The dependency rule: domain ← application ← infrastructure/interfaces. Nothing in `domain/` imports from any other layer.

See [references/architecture-patterns.md](references/architecture-patterns.md) for full TypeScript examples of the domain entity, port interface, and adapter, plus CQRS, event sourcing, and API versioning strategies.

---

## Database Patterns — Summary

**N+1 prevention** — never query inside a loop. Prefer a JOIN when the relationship is always needed; use a two-query batch (`WHERE id = ANY($1)`) when it is conditional.

```typescript
// ❌ BAD — 51 queries for 50 rows
for (const order of orders) {
  order.user = await db.query('SELECT * FROM users WHERE id = $1', [order.userId])
}

// ✅ GOOD — 2 queries regardless of result size
const userIds = [...new Set(orders.map(o => o.userId))]
const users = await db.query('SELECT * FROM users WHERE id = ANY($1)', [userIds])
```

**Connection pooling** — create one pool per process at startup, shared for its lifetime. Never open a connection per request. Size the pool to your database's `max_connections` minus headroom; divide the budget across application instances.

**Migrations** — run in CI before deploying; never inside `app.listen()` (race condition in multi-instance deploys). Always backward-compatible: add nullable columns before populating, then add `NOT NULL` in a later migration. Never rename a column in one step.

**Outbox pattern** — never publish events before the write transaction commits (dual write problem). Write the event to an `outbox` table in the same transaction; a separate relay process publishes it and marks it done.

See [references/database-patterns.md](references/database-patterns.md) for full N+1 fix examples, connection pool setup, migration rules, and the outbox pattern.

---

## The 12-Factor App — Key Factors

| Factor | Common Violation | Correct Pattern |
|---|---|---|
| **Config** | Hardcoded URLs in source | All config via environment variables |
| **Backing services** | DB URL baked into code | URL from env, swappable without code changes |
| **Build/release/run** | Code modified at runtime | Immutable release artifact; config injected at run |
| **Processes** | State stored in memory (sessions, caches) | Stateless processes; state in Redis/DB |
| **Logs** | Writing log files, rotating them | Write to stdout; infrastructure handles routing |
| **Dev/prod parity** | SQLite in dev, Postgres in prod | Identical backing services at all stages |

```typescript
// ✅ Factor III: Config from environment — fail fast if missing
const config = {
  databaseUrl: process.env.DATABASE_URL!,
  redisUrl:    process.env.REDIS_URL!,
  port:        Number(process.env.PORT ?? 3000),
  jwtSecret:   process.env.JWT_SECRET!,
}
for (const [key, val] of Object.entries(config)) {
  if (val === undefined || val === '') throw new Error(`Missing required env: ${key}`)
}

// ✅ Factor XI: Logs to stdout only
console.log(JSON.stringify({ level: 'info', message: 'Server started', port: config.port }))
```

---

## Common Mistakes to Avoid

**API design:**
- Returning 200 for errors (`{ success: false }` with status 200)
- Exposing auto-increment integer IDs in URLs — use UUIDs or opaque slugs
- Missing idempotency keys on mutation endpoints — dangerous for payment operations
- No pagination on list endpoints — add `limit`/`offset` or cursor pagination from day one

**Architecture:**
- Letting HTTP framework types leak into the domain layer
- Importing across module boundaries in a modular monolith
- Starting with microservices before domain boundaries are proven

**Database:**
- Running migrations inside `app.listen()` — race condition in multi-instance deploys
- Missing indexes on foreign keys and frequently filtered columns
- Using `SELECT *` in application queries — breaks when columns are added or removed
- Storing JSON blobs for data you will query inside — use proper columns and indexes

**CQRS/ES:**
- Applying event sourcing to entities that are trivially CRUD
- Publishing events before the write transaction commits — use the outbox pattern

---

## References

- [references/architecture-patterns.md](references/architecture-patterns.md) — Full clean/hexagonal arch TypeScript examples, CQRS, event sourcing, API versioning strategies
- [references/database-patterns.md](references/database-patterns.md) — N+1 prevention (3-tier fix), connection pooling, migration rules, outbox pattern
