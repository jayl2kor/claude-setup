# Database Antipatterns — Full Reference

Complete before/after examples for common database misuse patterns.

---

## SELECT *

Fetching all columns wastes bandwidth, breaks when the schema changes, and prevents the database from using index-only scans.

```sql
-- BAD — fetches all columns; breaks when schema changes; prevents index-only scans
SELECT * FROM orders WHERE user_id = $1;
```

```sql
-- GOOD — explicit column list; documents intent; enables covering indexes
SELECT id, status, total_cents, created_at
FROM orders
WHERE user_id = $1;
```

**Rule**: Never use `SELECT *` in application code. Always name the columns the application actually needs.

---

## Implicit Joins (Comma Syntax)

The comma-separated table syntax for joins mixes join conditions with filter conditions in the WHERE clause, making cartesian products easy to produce accidentally.

```sql
-- BAD — implicit join; WHERE clause mixes join conditions with filter conditions;
--       easy to produce a cartesian product by accidentally omitting a predicate
SELECT u.name, o.total_cents
FROM users u, orders o, order_items oi
WHERE u.id = o.user_id
  AND o.id = oi.order_id
  AND u.status = 'active';
```

```sql
-- GOOD — explicit JOIN; join condition is co-located with the joined table;
--        WHERE clause only contains filters
SELECT u.name, o.total_cents
FROM users u
JOIN orders o ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
WHERE u.status = 'active';
```

**Rule**: Always use explicit `JOIN ... ON` syntax. The WHERE clause should contain only filters, never join predicates.

---

## Storing Delimited Lists in a Column

Packing multiple values into a single column (e.g., CSV tags) defeats relational design: values cannot be indexed, queried cleanly, or enforced with foreign keys.

```sql
-- BAD — tags stored as CSV; impossible to index; requires LIKE '%python%' scans;
--       inserting/removing a single tag requires string manipulation in app code
CREATE TABLE posts (
    id   SERIAL PRIMARY KEY,
    tags TEXT  -- "python,django,rest-api"
);
```

```sql
-- GOOD — proper junction table; indexable; queryable with standard SQL
CREATE TABLE posts (
    id SERIAL PRIMARY KEY
);

CREATE TABLE tags (
    id   SERIAL PRIMARY KEY,
    name TEXT UNIQUE NOT NULL
);

CREATE TABLE post_tags (
    post_id INT REFERENCES posts(id) ON DELETE CASCADE,
    tag_id  INT REFERENCES tags(id)  ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

-- Finding all posts tagged "python" is now an indexed lookup:
SELECT p.id FROM posts p
JOIN post_tags pt ON pt.post_id = p.id
JOIN tags t ON t.id = pt.tag_id
WHERE t.name = 'python';
```

**Also applies to**: JSON arrays stored where a junction table belongs, pipe-delimited permission strings, comma-joined user IDs.

---

## N+1 Query in ORM Code

Issuing one query to fetch a list and then one additional query per row to fetch a related object. With 100 rows this becomes 101 queries; with 10,000 rows it becomes the primary cause of page timeouts.

```python
# BAD — Django ORM: 1 query for orders + 1 query per order for user
orders = Order.objects.filter(status='pending')
for order in orders:
    print(order.user.email)  # triggers a new query every iteration
```

```python
# GOOD — select_related issues a single JOIN; prefetch_related batches reverse relations
orders = Order.objects.filter(status='pending').select_related('user')
for order in orders:
    print(order.user.email)  # no additional query — already fetched
```

**ORM-specific fixes:**

| ORM | Forward relation (ForeignKey) | Reverse relation (one-to-many) |
|---|---|---|
| Django | `.select_related('user')` | `.prefetch_related('order_set')` |
| SQLAlchemy | `joinedload(Order.user)` | `subqueryload(User.orders)` |
| ActiveRecord | `.includes(:user)` | `.includes(:orders)` |
| TypeORM | `relations: ['user']` | `relations: ['orders']` |
| Prisma | `include: { user: true }` | `include: { orders: true }` |

**Detection**: Enable query logging in development and look for repeated identical queries in a loop. Tools: Django Debug Toolbar, SQLAlchemy `echo=True`, Bullet gem (Rails).
