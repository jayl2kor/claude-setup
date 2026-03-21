# Async / Concurrency Antipatterns — Full Reference

Complete before/after examples for concurrency and async antipatterns.

---

## Callback Hell (Pyramid of Doom)

Deeply nested callbacks make error handling impossible to follow and control flow invisible.

```javascript
// BAD — deeply nested callbacks, error handling impossible to follow
getUser(userId, function(err, user) {
  if (err) handleError(err)
  else getOrders(user.id, function(err, orders) {
    if (err) handleError(err)
    else getInvoice(orders[0].id, function(err, invoice) {
      if (err) handleError(err)
      else sendEmail(invoice, function(err) {
        if (err) handleError(err)
      })
    })
  })
})
```

```javascript
// GOOD — async/await with centralized error handling
async function processUserInvoice(userId) {
  const user = await getUser(userId)
  const orders = await getOrders(user.id)
  const invoice = await getInvoice(orders[0].id)
  await sendEmail(invoice)
}

// BETTER — run independent operations concurrently, not serially
async function processUserInvoice(userId) {
  const user = await getUser(userId)
  const [orders, profile] = await Promise.all([getOrders(user.id), getProfile(user.id)])
  const invoice = await getInvoice(orders[0].id)
  await sendEmail(invoice)
}
```

**Detection signal**: More than two levels of callback nesting; error handling repeated at every level.

---

## Fire-and-Forget Without Error Handling

Launching an async operation without awaiting it or attaching an error handler causes unhandled promise rejections that crash the process silently in some runtimes.

```typescript
// BAD — unhandled promise rejection crashes the process silently in some runtimes
async function handleRequest(req, res) {
  sendAnalyticsEvent(req.path)  // unawaited — errors vanish
  res.json({ ok: true })
}
```

```typescript
// GOOD — explicit non-critical fire-and-forget with logging
async function handleRequest(req, res) {
  sendAnalyticsEvent(req.path).catch(err =>
    logger.warn('Analytics event failed', { path: req.path, err })
  )
  res.json({ ok: true })
}
```

**Detection signal**: A function call that returns a Promise but is neither awaited nor has `.catch()` attached. ESLint rule: `@typescript-eslint/no-floating-promises`.

---

## Race Condition via Shared Mutable State

Concurrent goroutines or threads reading and writing shared state without synchronization produce non-deterministic data corruption.

```go
// BAD — concurrent goroutines writing to a shared map without synchronization
var cache = map[string]string{}

func get(key string) string {
    return cache[key]   // data race: concurrent read and write
}

func set(key, value string) {
    cache[key] = value  // data race
}
```

```go
// GOOD — protect shared state with a mutex, or use sync.Map for read-heavy workloads
import "sync"

type SafeCache struct {
    mu    sync.RWMutex
    store map[string]string
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.store[key]
    return v, ok
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.store[key] = value
}
```

**Detection**: Run `go test -race ./...` — the Go race detector catches these at test time.

---

## Context Leak / Goroutine Leak

A goroutine that blocks forever because nobody reads from its channel, or because the caller returned early without cancelling the context.

```go
// BAD — goroutine blocks forever if nobody reads from the channel
func startWorker() {
    ch := make(chan int)    // unbuffered
    go func() {
        ch <- heavyComputation()  // goroutine leaks if caller never reads
    }()
    // caller may return early, abandoning the goroutine
}
```

```go
// GOOD — context cancellation ensures the goroutine can always exit
func startWorker(ctx context.Context) <-chan int {
    ch := make(chan int, 1)  // buffered so send does not block
    go func() {
        defer close(ch)
        result := heavyComputation()
        select {
        case ch <- result:
        case <-ctx.Done():  // caller cancelled — exit cleanly
        }
    }()
    return ch
}
```

**Detection**: Use `goleak` in tests (`defer goleak.VerifyNone(t)`), or profile goroutine count over time with `runtime.NumGoroutine()`.

**General rule**: Every goroutine must have a clear termination path. Pass `context.Context` from the caller; never create a goroutine with no way to stop it.
