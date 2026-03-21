---
name: antipattern-detection
description: This skill should be used when the user asks to "review", "audit", "refactor", or "find problems" in code, or when code exhibits symptoms like excessive coupling, growing complexity, hard-to-test classes, unexplained performance issues, or security concerns. Trigger phrases include "review this code", "audit the codebase", "find antipatterns", "what's wrong with this", "help me refactor", "why is this hard to test", "find code smells".
version: 1.0.0
---

# Antipattern Detection

Expert identification and remediation of code antipatterns across GoF structural problems, SOLID violations, concurrency bugs, database misuse, security flaws, and architecture smells. Every antipattern entry shows the bad pattern and the correct refactored form.

## Categories

| Category | Key Antipatterns | Reference |
|---|---|---|
| GoF Structural | God Object, Spaghetti Code, Golden Hammer, Copy-Paste Programming | (below) |
| SOLID Violations | SRP, OCP, LSP, ISP, DIP violations | [solid-violations.md](references/solid-violations.md) |
| Async / Concurrency | Callback Hell, Fire-and-Forget, Race Conditions, Goroutine Leaks | [async-patterns.md](references/async-patterns.md) |
| Database | SELECT *, Implicit Joins, Delimited Lists, N+1 Queries | [db-antipatterns.md](references/db-antipatterns.md) |
| Frontend | Prop Drilling, Over-Fetching, Premature useMemo | [language-specific.md](references/language-specific.md) |
| Security | Hardcoded Credentials, SQL Injection, eval() on Input | (below) |
| Architecture | Circular Deps, Anemic Domain Model, Big Ball of Mud | [language-specific.md](references/language-specific.md) |

---

## GoF Structural Antipatterns

### God Object

A single class that knows too much and does too much. Recognizable by 1000+ line classes, dozens of unrelated methods, and direct access to many subsystems.

```python
# BAD — UserManager does authentication, emailing, billing, and reporting
class UserManager:
    def login(self, email, password): ...
    def send_welcome_email(self, user): ...
    def charge_subscription(self, user, plan): ...
    def generate_monthly_report(self): ...
```

```python
# GOOD — Single responsibility per class; wire them together at the app layer
class AuthService:
    def login(self, email: str, password: str) -> User: ...

class BillingService:
    def charge_subscription(self, user: User, plan: Plan) -> Receipt: ...

class NotificationService:
    def send_welcome_email(self, user: User) -> None: ...

class AnalyticsService:
    def generate_monthly_report(self) -> Report: ...
```

**Detection signal**: Class has more than 7–10 public methods spanning unrelated concerns, or its constructor takes more than 5 dependencies.

### Spaghetti Code

Logic with no clear structure: deeply nested conditionals, global mutable state, and side effects buried inside loops or calculations.

```javascript
// BAD — side effects buried in price calculation loop
function processOrder(order) {
  if (order) {
    if (order.items.length > 0) {
      let total = 0
      for (let i = 0; i < order.items.length; i++) {
        if (order.items[i].inStock) {
          total += order.items[i].price
          db.save(order.items[i])  // side effect buried in loop
        }
      }
      sendEmail(order.user)  // side effect buried in price calc
      return total
    }
  }
  return 0
}
```

```javascript
// GOOD — pure calculation separate from side effects
function calculateOrderTotal(order) {
  const subtotal = order.items
    .filter(i => i.inStock)
    .reduce((sum, i) => sum + i.price, 0)
  return applyDiscount(subtotal, order.discount)
}

async function processOrder(order) {
  if (!order || order.items.length === 0) return { total: 0 }
  const total = calculateOrderTotal(order)
  await Promise.all([db.saveOrder({ ...order, total }), emailService.sendConfirmation(order.user)])
  return { total }
}
```

### Golden Hammer

Applying a familiar technology to every problem regardless of fit (e.g., message queues for synchronous calls, microservices for a two-person team).

**Remediation**: Write a one-paragraph justification naming the specific requirement satisfied and at least one alternative considered. If you cannot write it, reconsider the choice.

### Copy-Paste Programming

Duplicated logic scattered across the codebase; fixing a bug requires changing the same logic in 3+ places.

**Rule of Three**: Duplicate once (acceptable), duplicate twice (extract it).

---

## SOLID Violations (Summary)

Full before/after code examples for all five violations: [references/solid-violations.md](references/solid-violations.md)

| Principle | Violation Signal | Quick Fix |
|---|---|---|
| Single Responsibility | Class changes for two unrelated reasons | Split into focused classes |
| Open/Closed | Adding a variant requires editing existing `if/elif` | Use polymorphism / strategy pattern |
| Liskov Substitution | Subclass throws `NotImplementedError` for inherited method | Separate contracts into distinct interfaces |
| Interface Segregation | Implementors forced to stub methods they don't use | Break the interface into smaller focused ones |
| Dependency Inversion | High-level module `new`s a concrete dependency | Inject via constructor; depend on abstract interface |

---

## Security Antipatterns

### Hardcoded Credentials

```python
# BAD — credentials in source code
DB_PASSWORD = "super-secret-prod-password-123"
API_KEY = "sk_live_abcdefghijklmnop"
```

```python
# GOOD — credentials from environment or secrets manager
import os

def get_db_password() -> str:
    password = os.environ.get("DB_PASSWORD")
    if not password:
        raise RuntimeError("DB_PASSWORD environment variable is not set")
    return password
```

Enforce with `detect-secrets` in pre-commit hooks and `git-secrets` on the repo.

### SQL Injection

```python
# BAD — user input concatenated into SQL
query = f"SELECT * FROM users WHERE username = '{username}'"  # exploitable
```

```python
# GOOD — parameterized queries
cursor.execute("SELECT id, email FROM users WHERE username = %s", (username,))
```

Never build SQL by string concatenation or f-strings. This applies equally to NoSQL query operators.

### eval() / exec() on User Input

```javascript
// BAD — arbitrary code execution
const result = eval(req.body.formula)  // RCE vulnerability
```

```javascript
// GOOD — sandboxed math library
import { evaluate } from 'mathjs'
const result = evaluate(req.body.formula)
```

---

## Detection Checklist

Scan for these signals systematically when reviewing a codebase:

**Structural:**
- [ ] Classes/modules with more than one distinct concern (God Object / SRP violation)
- [ ] `switch`/`if-else` chains that grow when new variants are added (OCP violation)
- [ ] Constructor parameter count > 5 (likely God Object or missing abstraction)
- [ ] Duplicated logic in 3+ places (Copy-Paste Programming)

**Concurrency:**
- [ ] Shared mutable state without synchronization primitives
- [ ] Unawaited promises without `.catch()` (fire-and-forget leaks)
- [ ] Goroutines or async tasks with no cancellation path

**Database:**
- [ ] `SELECT *` in application queries
- [ ] ORM loops that trigger per-row queries (N+1)
- [ ] Comma-delimited values stored in a single column

**Security:**
- [ ] String interpolation used to build SQL queries
- [ ] Credentials, tokens, or keys in source files or committed config
- [ ] `eval()`, `exec()`, or `Function()` called with any external input

**Architecture:**
- [ ] Circular imports detected by static analysis
- [ ] HTTP framework types (`Request`, `Response`) imported inside domain or service layers
- [ ] No clear direction to the import graph (every module imports every other)

---

## References

- [references/solid-violations.md](references/solid-violations.md) — All 5 SOLID violation patterns with full before/after code
- [references/async-patterns.md](references/async-patterns.md) — Callback Hell, Fire-and-Forget, Race Conditions, Goroutine Leaks
- [references/db-antipatterns.md](references/db-antipatterns.md) — SELECT *, Implicit Joins, Delimited Lists, N+1 Queries
- [references/language-specific.md](references/language-specific.md) — Frontend patterns (Prop Drilling, Over-Fetching) and Architecture smells (Circular Deps, Anemic Domain, Big Ball of Mud)
