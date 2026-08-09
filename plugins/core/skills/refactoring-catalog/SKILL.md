---
name: refactoring-catalog
description: >-
  The reference catalog of code smells and the refactoring techniques that
  resolve them — the "what to fix and how" that the refactor-safely workflow
  draws on. Use when identifying a code smell, choosing a refactoring technique,
  naming what is wrong with a piece of code, or looking up the mechanics of a
  specific transformation. Covers the smell families (bloaters, OO abusers,
  change preventers, dispensables, couplers) and the technique groups, with a
  smell→technique map. Triggers: "code smell", "what's wrong with this code",
  "refactor technique", "extract method", "long method", "duplication", "feature
  envy", "how do I refactor this".
---

# Refactoring catalog

This is the catalog: the smells to recognize and the techniques that fix them. The
*workflow* — establish tests, transform in small steps, keep behavior unchanged —
lives in `refactor-safely`; do not apply anything here without that safety net.
Recognizing the smell by name is half the work; it tells you which technique to
reach for.

## Code smells by family

### Bloaters — things that have grown too large

- **Long Method** — a method doing too much; hard to name, hard to follow.
- **Large Class** — a class with too many fields/methods; multiple
  responsibilities (`solid` SRP).
- **Long Parameter List** — many parameters signalling a missing object.
- **Primitive Obsession** — strings/ints standing in for domain concepts (money,
  email) that should be value objects.
- **Data Clumps** — the same group of fields travelling together everywhere.

### OO abusers — object orientation used wrongly

- **Switch Statements** — type-based conditionals that should be polymorphism
  (OCP).
- **Refused Bequest** — a subclass that does not want what it inherits (LSP).
- **Temporary Field** — a field set only in certain circumstances.
- **Alternative Classes with Different Interfaces** — classes doing the same thing
  with mismatched signatures.

### Change preventers — one change forces many

- **Divergent Change** — one class changed for many different reasons (SRP).
- **Shotgun Surgery** — one change scattered across many classes.
- **Parallel Inheritance Hierarchies** — every new subclass here forces one there.

### Dispensables — things that add no value

- **Comments** compensating for unclear code — fix the code instead.
- **Duplicate Code** — the same logic in more than one place.
- **Dead Code** — unreachable or unused code.
- **Lazy Class** — a class that no longer earns its existence.
- **Speculative Generality** — abstraction built for a future that never came
  (`engineering-standards`).
- **Data Class** — fields and accessors with no behavior.

### Couplers — excessive coupling between classes

- **Feature Envy** — a method more interested in another class's data than its own.
- **Inappropriate Intimacy** — classes reaching into each other's internals.
- **Message Chains** — `a.getB().getC().getD()`; violates Law of Demeter.
- **Middle Man** — a class that only delegates.

## Smell to technique

| Smell | Technique(s) |
|---|---|
| Long Method | Extract Method; Replace Temp with Query; Decompose Conditional. |
| Large Class | Extract Class; Extract Interface; Move Method. |
| Long Parameter List | Introduce Parameter Object; Preserve Whole Object. |
| Primitive Obsession | Replace Primitive with Value Object; Replace Type Code with Class. |
| Switch Statements | Replace Conditional with Polymorphism; Strategy. |
| Duplicate Code | Extract Method; Pull Up Method; Form Template Method. |
| Feature Envy | Move Method; Extract Method then move. |
| Message Chains | Hide Delegate; Extract Method. |
| Divergent Change / Shotgun Surgery | Move Method/Field; Extract Class; Inline Class. |
| Nested conditionals | Replace Nested Conditional with Guard Clauses. |

## Technique groups

- **Composing methods** — Extract/Inline Method, Extract Variable, Replace Temp
  with Query, Split Temporary Variable.
- **Moving features between objects** — Move Method, Move Field, Extract Class,
  Inline Class, Hide Delegate, Remove Middle Man.
- **Organizing data** — Replace Primitive with Object, Encapsulate Field/Collection,
  Replace Magic Number with Constant.
- **Simplifying conditionals** — Decompose Conditional, Consolidate Conditional,
  Replace Nested Conditional with Guard Clauses, Replace Conditional with
  Polymorphism, Introduce Null Object.
- **Simplifying method calls** — Rename Method, Add/Remove Parameter, Introduce
  Parameter Object, Replace Constructor with Factory Method.
- **Dealing with generalization** — Pull Up/Push Down Method/Field, Extract
  Superclass, Extract Interface, Replace Inheritance with Delegation.

## Key transformations

The five worth seeing rather than naming:

**Extract Method** — pull each intention out of a method that does several things.
```
// Before
function printReport(order: Order): void {
    print("Order #" + order.id())
    total = 0
    for line in order.lines():
        total += line.quantity() * line.price()
    print("Total: " + total)
}

// After — each step named
function printReport(order: Order): void {
    printHeader(order)
    printTotal(calculateTotal(order))
}
```

**Replace Nested Conditional with Guard Clauses** — flatten arrow code; keep the
happy path at the left margin.
```
// Before
function payAmount(e: Employee): number {
    if (!e.isSeparated()) {
        if (!e.isRetired()) {
            return normalPay(e)
        } else {
            return 0
        }
    } else {
        return 0
    }
}

// After
function payAmount(e: Employee): number {
    if (e.isSeparated()) { return 0 }
    if (e.isRetired())   { return 0 }
    return normalPay(e)
}
```

**Replace Conditional with Polymorphism** — a type switch that grows with each new
type becomes one class per type (Open/Closed).
```
// Before
function speed(bird: Bird): number {
    switch (bird.type()) {
        case "european":  return baseSpeed()
        case "african":   return baseSpeed() - bird.load()
        case "norwegian": return bird.isNailed() ? 0.0 : baseSpeed()
    }
}

// After
interface Bird { speed(): number; }
class EuropeanSwallow implements Bird { speed(): number { /* ... */ } }
class AfricanSwallow  implements Bird { speed(): number { /* ... */ } }
```

**Introduce Parameter Object** — a cluster of arguments that travel together
becomes a type (fixes Data Clumps / Long Parameter List).
```
// Before
function findOrders(from: Date, to: Date): Order[] {}

// After
function findOrders(range: DateRange): Order[] {}

class DateRange {
    constructor(public from: Date, public to: Date) {}
}
```

**Replace Primitive with Value Object** — a primitive that can hold anything becomes
a type that cannot exist in an invalid state (fixes Primitive Obsession).
```
// Before
class User {
    constructor(private email: string) {}
}

// After
class Email {
    constructor(public value: string) {
        if (!isValidEmail(value)) {
            throw new Error("Invalid email: " + value)
        }
    }
}

class User {
    constructor(private email: Email) {}
}
```

## Mechanics reminder

Every technique is applied the same disciplined way: under a passing test suite,
one transformation at a time, tests green after each, committed in working steps.
That mechanic is `refactor-safely` running the `iterate` loop — this catalog only
tells you *which* transformation, not *how* to sequence it safely.

## Anti-patterns

- **Applying a technique without tests** — see `refactor-safely`; the cardinal sin.
- **Smell-hunting out of scope** — refactoring code you were not tasked to touch.
- **Chasing purity** — removing every smell instead of the one causing pain.
- **Over-abstracting** — "fixing" speculative generality by adding more of it.

## Where this fits

`refactoring-catalog` is a Layer 2 reference consumed by `refactor-safely` (the
workflow) and pointed to by `code-quality`, `solid`, and `design-patterns` (the
targets you refactor toward). Smells here frequently map to SOLID violations and
resolve into GoF patterns.
