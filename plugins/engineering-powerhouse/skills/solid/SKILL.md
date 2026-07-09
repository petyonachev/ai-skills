---
name: solid
description: >-
  The five SOLID principles for object-oriented design, applied to reduce
  coupling and make change safe — with the discipline to apply them in response
  to real pain, not preemptively. Use when reviewing class design, fixing a
  SOLID violation, reducing coupling, deciding where a responsibility belongs, or
  judging whether an abstraction earns its keep. Covers detection, the fix, and
  the trade-off for each principle. Triggers: "SOLID", "single responsibility",
  "open/closed", "Liskov", "interface segregation", "dependency inversion",
  "tight coupling", "this class does too much", "class design review".
---

# SOLID

SOLID is five principles that make object-oriented code easier to change by
lowering coupling and localizing the reasons a class must change. They are
**guidelines serving a goal (changeability), not laws**. Misapplied — reached for
before any real pain exists — they produce the opposite: a maze of indirection and
one-method classes that is harder to follow than the code they replaced. Apply
them when the design actually hurts, in line with `engineering-standards` ("no
premature abstraction").

## The five principles

### S — Single Responsibility

A class should have one reason to change. **Smell:** a class that mixes business
rules, persistence, and formatting; a "god" service that grows without bound.
**Fix:** extract collaborators along the axes of change (a `Formatter`, a
`Repository`, a policy object). **Trade-off:** do not atomize every method into
its own class — a responsibility is a reason to change, not a single line.

```php
// Before — one class changes for three unrelated reasons
final class Invoice
{
    public function total(): Money { /* business rule */ }
    public function save(): void { /* persistence */ }
    public function toPdf(): string { /* formatting */ }
}

// After — each responsibility isolated; each has one reason to change
final class Invoice { public function total(): Money { /* ... */ } }
final class InvoiceRepository { public function save(Invoice $invoice): void {} }
final class InvoicePdfRenderer { public function render(Invoice $invoice): string {} }
```

### O — Open/Closed

Software should be open for extension, closed for modification. **Smell:** a
`switch`/`if-else` on a type flag that you must edit every time a new type
appears. **Fix:** polymorphism or Strategy — add a new class instead of editing
the conditional (`design-patterns`). **Trade-off:** only abstract the axis that
genuinely varies; a conditional that never grows does not need a hierarchy.

```php
// Before — every new shipping method edits this class
public function cost(Order $order, string $method): int
{
    return match ($method) {
        'standard' => $order->weight() * 2,
        'express'  => $order->weight() * 5,
    };
}

// After — a new method is a new class; this code never changes again
interface ShippingMethod
{
    public function cost(Order $order): int;
}

final class ExpressShipping implements ShippingMethod
{
    public function cost(Order $order): int
    {
        return $order->weight() * 5;
    }
}
```

### L — Liskov Substitution

A subtype must be usable anywhere its base type is, without surprising the caller.
**Smell:** a subclass that throws on an inherited method, strengthens
preconditions, or weakens guarantees (the classic `Square extends Rectangle`).
**Fix:** rethink the hierarchy; prefer composition over inheritance when the
"is-a" does not truly hold. **Trade-off:** inheritance is a strong claim — most
"is-a" relationships are better modelled as "has-a".

```php
// Before — subtype can't honor the inherited contract; callers break
final class ReadOnlyOrderRepository extends DoctrineOrderRepository
{
    public function save(Order $order): void
    {
        throw new \LogicException('read-only'); // violates substitutability
    }
}

// After — don't force a false is-a; expose only the honored capability
interface OrderReader
{
    public function find(int $id): ?Order;
}

final class ReadOnlyOrderRepository implements OrderReader
{
    public function find(int $id): ?Order { /* ... */ }
}
```

### I — Interface Segregation

No client should depend on methods it does not use. **Smell:** a fat interface
whose implementers throw `NotImplemented` for half its methods. **Fix:** split
into focused role interfaces so each client depends only on what it needs.
**Trade-off:** do not shatter a genuinely cohesive interface into fragments.

```php
// Before — a fat interface forces implementers to stub what they don't use
interface Report
{
    public function toPdf(): string;
    public function toCsv(): string;
    public function toXlsx(): string;
}

// After — role interfaces; a client depends only on the format it needs
interface PdfRenderable { public function toPdf(): string; }
interface CsvRenderable { public function toCsv(): string; }
```

### D — Dependency Inversion

High-level policy should depend on abstractions, not on low-level details.
**Smell:** a service that `new`s a concrete database or HTTP client inside itself,
welding policy to mechanism. **Fix:** depend on an interface and inject the
concrete implementation (constructor injection via the Symfony container — see
`engineering-standards` and `symfony`). **Trade-off:** an interface with exactly
one implementation and no prospect of another is often just ceremony.

```php
// Before — policy welded to mechanism, impossible to test in isolation
final class OrderService
{
    public function place(Order $order): void
    {
        (new MySqlOrderRepository())->save($order);
    }
}

// After — depend on an abstraction, inject the concretion
final class OrderService
{
    public function __construct(
        private readonly OrderRepository $repository,
    ) {}

    public function place(Order $order): void
    {
        $this->repository->save($order);
    }
}
```

## Detection — smell to principle

| Smell | Principle | Fix |
|---|---|---|
| Class changes for many unrelated reasons | SRP | Extract collaborators by responsibility. |
| `switch`/`if` on type that grows with each new type | OCP | Polymorphism / Strategy. |
| Subclass throws on or breaks an inherited method | LSP | Rework hierarchy; prefer composition. |
| Implementers stub out unused interface methods | ISP | Split into role interfaces. |
| Business logic `new`s its own infrastructure | DIP | Inject an abstraction. |

## Applying without over-engineering

SOLID and "no premature abstraction" are not in tension — both say *abstract in
response to real, present forces*. The trigger to apply a principle is pain you
can name: a class you keep editing for unrelated reasons, a conditional that grows
every sprint, a test you cannot write because a dependency is hard-wired. Absent
that pain, the simpler concrete code is the better code. Refactor *toward* SOLID
when the smell appears (`refactor-safely`), not speculatively.

## Anti-patterns

- **Speculative SOLID** — interfaces and layers added "in case" the code varies,
  when it never does.
- **Interface-per-class reflex** — a one-implementation interface for everything.
- **SRP atomization** — classes so small the logic is scattered and unreadable.
- **Inheritance for reuse** — using `extends` to share code, breaking LSP.

## Where this fits

`solid` is a Layer 2 design reference. `refactor-safely` refactors code toward
these principles; `design-patterns` supplies the concrete mechanisms (Strategy for
OCP, injection for DIP); `code-quality` and `engineering-standards` hold the
broader craft and the anti-premature-abstraction discipline. Hard-to-test code
(`testing`) is often a DIP or SRP violation pointing here.
