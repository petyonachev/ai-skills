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

```
// Before — one class changes for three unrelated reasons
class Invoice {
    total(): Money { /* business rule */ }
    save(): void { /* persistence */ }
    toPdf(): string { /* formatting */ }
}

// After — each responsibility isolated; each has one reason to change
class Invoice { total(): Money { /* ... */ } }
class InvoiceRepository { save(invoice: Invoice): void {} }
class InvoicePdfRenderer { render(invoice: Invoice): string {} }
```
(a worked example in your language lives in your stack plugin)

### O — Open/Closed

Software should be open for extension, closed for modification. **Smell:** a
`switch`/`if-else` on a type flag that you must edit every time a new type
appears. **Fix:** polymorphism or Strategy — add a new class instead of editing
the conditional (`design-patterns`). **Trade-off:** only abstract the axis that
genuinely varies; a conditional that never grows does not need a hierarchy.

```
// Before — every new shipping method edits this class
function cost(order: Order, method: string): number {
    switch (method) {
        case "standard": return order.weight() * 2;
        case "express":  return order.weight() * 5;
    }
}

// After — a new method is a new class; this code never changes again
interface ShippingMethod {
    cost(order: Order): number;
}

class ExpressShipping implements ShippingMethod {
    cost(order: Order): number {
        return order.weight() * 5;
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

```
// Before — subtype can't honor the inherited contract; callers break
class ReadOnlyOrderRepository extends SqlOrderRepository {
    save(order: Order): void {
        throw new Error("read-only"); // violates substitutability
    }
}

// After — don't force a false is-a; expose only the honored capability
interface OrderReader {
    find(id: number): Order | null;
}

class ReadOnlyOrderRepository implements OrderReader {
    find(id: number): Order | null { /* ... */ }
}
```

### I — Interface Segregation

No client should depend on methods it does not use. **Smell:** a fat interface
whose implementers throw `NotImplemented` for half its methods. **Fix:** split
into focused role interfaces so each client depends only on what it needs.
**Trade-off:** do not shatter a genuinely cohesive interface into fragments.

```
// Before — a fat interface forces implementers to stub what they don't use
interface Report {
    toPdf(): string;
    toCsv(): string;
    toXlsx(): string;
}

// After — role interfaces; a client depends only on the format it needs
interface PdfRenderable { toPdf(): string; }
interface CsvRenderable { toCsv(): string; }
```

### D — Dependency Inversion

High-level policy should depend on abstractions, not on low-level details.
**Smell:** a service that `new`s a concrete database or HTTP client inside itself,
welding policy to mechanism. **Fix:** depend on an interface and inject the
concrete implementation (constructor injection via your framework's DI
container — see `engineering-standards` and your stack plugin). **Trade-off:**
an interface with exactly one implementation and no prospect of another is
often just ceremony.

```
// Before — policy welded to mechanism, impossible to test in isolation
class OrderService {
    place(order: Order): void {
        new SqlOrderRepository().save(order);
    }
}

// After — depend on an abstraction, inject the concretion
class OrderService {
    constructor(private repository: OrderRepository) {}

    place(order: Order): void {
        this.repository.save(order);
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
