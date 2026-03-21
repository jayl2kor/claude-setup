# SOLID Violations — Full Reference

All five SOLID principles with complete before/after code examples.

---

## Single Responsibility Principle (SRP)

A function or class should have one reason to change.

```python
# BAD — this class changes when persistence changes AND when report format changes
class ReportService:
    def generate(self, data) -> str:  # formats report
        ...
    def save_to_db(self, report: str) -> None:  # persists report
        ...
    def email_to_stakeholders(self, report: str) -> None:  # delivers report
        ...
```

```python
# GOOD — each class has one reason to change
class ReportFormatter:
    def format(self, data: ReportData) -> str: ...

class ReportRepository:
    def save(self, report: str) -> None: ...

class ReportNotifier:
    def notify(self, report: str, recipients: list[str]) -> None: ...

class ReportOrchestrator:
    def __init__(self, formatter, repo, notifier):
        self._formatter = formatter
        self._repo = repo
        self._notifier = notifier

    def run(self, data: ReportData, recipients: list[str]) -> None:
        report = self._formatter.format(data)
        self._repo.save(report)
        self._notifier.notify(report, recipients)
```

---

## Open/Closed Principle (OCP)

Software entities should be open for extension but closed for modification. Adding a new variant should not require editing existing code.

```python
# BAD — every new payment method requires editing this function
def process_payment(order, method):
    if method == 'stripe':
        stripe.charge(order.total)
    elif method == 'paypal':
        paypal.pay(order.total)
    elif method == 'apple_pay':  # new requirement = edit existing function
        apple.pay(order.total)
```

```python
# GOOD — new methods extend without modifying existing code
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount_cents: int) -> Receipt: ...

class StripeGateway(PaymentGateway):
    def charge(self, amount_cents: int) -> Receipt: ...

class PayPalGateway(PaymentGateway):
    def charge(self, amount_cents: int) -> Receipt: ...

# Adding ApplePayGateway never touches existing classes
def process_payment(order: Order, gateway: PaymentGateway) -> Receipt:
    return gateway.charge(order.total_cents)
```

---

## Liskov Substitution Principle (LSP)

A subclass must be usable everywhere the base class is expected, without breaking callers.

```python
# BAD — ReadOnlyFile violates the contract of File
class File:
    def write(self, data: bytes) -> None: ...

class ReadOnlyFile(File):
    def write(self, data: bytes) -> None:
        raise NotImplementedError("This file is read-only")  # LSP violation
```

```python
# GOOD — separate the readable and writable contracts
from typing import Protocol

class Readable(Protocol):
    def read(self) -> bytes: ...

class Writable(Protocol):
    def write(self, data: bytes) -> None: ...

class ReadWriteFile(Readable, Writable):
    def read(self) -> bytes: ...
    def write(self, data: bytes) -> None: ...

class ReadOnlyFile(Readable):
    def read(self) -> bytes: ...
    # write() does not exist — callers that need it get a type error at compile time
```

**Detection signal**: Subclass throws `NotImplementedError`, `UnsupportedOperationException`, or equivalent for an inherited method.

---

## Interface Segregation Principle (ISP)

Clients should not be forced to depend on methods they do not use. Prefer many small interfaces over one large one.

```typescript
// BAD — DocumentProcessor forces all implementors to support all operations
interface DocumentProcessor {
  parse(raw: string): Document
  render(doc: Document): string
  validate(doc: Document): ValidationResult
  encrypt(doc: Document): EncryptedDocument
  compress(doc: Document): Buffer
}

class SimpleHTMLRenderer implements DocumentProcessor {
  render(doc: Document): string { ... }
  parse(): Document { throw new Error('not supported') }      // forced stub
  validate(): ValidationResult { throw new Error('not supported') }
  encrypt(): EncryptedDocument { throw new Error('not supported') }
  compress(): Buffer { throw new Error('not supported') }
}
```

```typescript
// GOOD — small, focused interfaces; compose as needed
interface Parser    { parse(raw: string): Document }
interface Renderer  { render(doc: Document): string }
interface Validator { validate(doc: Document): ValidationResult }
interface Encryptor { encrypt(doc: Document): EncryptedDocument }

class SimpleHTMLRenderer implements Renderer {
  render(doc: Document): string { ... }
  // Only implements what it actually does
}

class SecurePDFProcessor implements Parser, Renderer, Validator, Encryptor { ... }
```

**Detection signal**: Implementors stub out methods with `throw new Error('not supported')` or equivalent.

---

## Dependency Inversion Principle (DIP)

High-level modules should not import concrete low-level modules. Both should depend on abstractions.

```typescript
// BAD — OrderService is permanently coupled to PostgresOrderRepository
import { PostgresOrderRepository } from './PostgresOrderRepository'

class OrderService {
  private repo = new PostgresOrderRepository()  // concrete dependency, untestable
  async getOrder(id: string) { return this.repo.findById(id) }
}
```

```typescript
// GOOD — depend on an abstraction; inject the concrete at the composition root
interface OrderRepository {
  findById(id: string): Promise<Order | null>
  save(order: Order): Promise<void>
}

class OrderService {
  constructor(private readonly repo: OrderRepository) {}
  async getOrder(id: string) { return this.repo.findById(id) }
}

// In tests: inject a mock. In production: inject PostgresOrderRepository.
const service = new OrderService(new PostgresOrderRepository(pool))
```

**Detection signal**: `new ConcreteClass()` inside a class constructor or method body; imports of concrete infrastructure types inside domain or service layers.
