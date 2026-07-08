---
name: design-patterns
description: >-
  The Gang of Four design patterns as a shared vocabulary and named responses to
  recurring design forces — with the discipline to apply a pattern only when a
  real force demands it, never as a goal. Use when choosing a pattern to solve a
  design problem, refactoring toward a pattern, evaluating existing pattern usage,
  or naming a structure you already have. Covers all three families with a
  problem→pattern selection guide and deeper notes on the patterns that earn
  their keep in PHP/Symfony. Triggers: "design pattern", "apply a pattern",
  "strategy", "factory", "decorator", "observer", "refactor to a pattern",
  "which pattern".
---

# Design patterns

A design pattern is a named, proven response to a recurring design force. Its
value is twofold: it solves a real problem well, and it gives the team a shared
word for a structure ("this is a Strategy") that would otherwise take a paragraph
to explain.

**A pattern is the answer to a specific problem — never a goal.** Reaching for
patterns to make code "proper" produces the classic over-engineered result:
indirection nobody needs. The rule is: recognize the *force* first (things vary
along an axis, a dependency must be swappable, an algorithm has interchangeable
steps), then reach for the pattern that resolves it. If you cannot name the force,
you do not need the pattern.

## Selection guide — problem to pattern

| The force you feel | Pattern | Family |
|---|---|---|
| A behavior varies; callers pick which at runtime | **Strategy** | Behavioral |
| Something must react to state changes without tight coupling | **Observer** | Behavioral |
| A request should be a first-class object (queue, log, undo) | **Command** | Behavioral |
| An algorithm's skeleton is fixed but steps vary | **Template Method** | Behavioral |
| A request should pass along a chain until handled | **Chain of Responsibility** | Behavioral |
| Object behavior changes with its internal state | **State** | Behavioral |
| Complex object creation should be hidden from callers | **Factory Method** / **Abstract Factory** | Creational |
| An object needs step-by-step construction | **Builder** | Creational |
| An incompatible interface must fit your code | **Adapter** | Structural |
| Add responsibilities to an object without subclassing | **Decorator** | Structural |
| A complex subsystem needs a simple entry point | **Facade** | Structural |
| Tree of part-whole objects treated uniformly | **Composite** | Structural |
| Control access to / defer creation of an object | **Proxy** | Structural |
| Avoid null checks with a do-nothing default | **Null Object** | Behavioral |

The remaining GoF patterns — Prototype, Object Pool, Singleton, Bridge, Flyweight,
Mediator, Iterator, Visitor, Memento, Interpreter — solve narrower forces; reach
for them when that specific force appears, not before.

## The patterns that earn their keep in Symfony

Symfony already embodies many patterns; recognizing them tells you the idiomatic
place to plug in rather than reinventing:

- **Strategy** → tagged services + a service locator: inject a set of
  interchangeable handlers, select by key. The idiomatic OCP fix.
- **Decorator** → service decoration (`#[AsDecorator]`): wrap a service to add
  caching, logging, or metrics without touching it.
- **Observer** → the EventDispatcher: emit domain events, subscribe listeners; the
  native way to decouple side effects.
- **Command** → Messenger messages/handlers and Console commands: a request as an
  object, dispatched to its handler.
- **Factory** → DI factories for services whose construction is non-trivial.
- **Adapter** → the anti-corruption layer around a third party (`integration-build`).
- **Null Object** → a no-op implementation of an interface to delete scattered
  null checks.

The three you reach for most, in idiomatic Symfony:

```php
// Strategy via tagged services — the idiomatic OCP fix
final class ReportExporter
{
    /** @param iterable<ExportFormat> $formats */
    public function __construct(
        #[AutowireIterator('app.export_format')]
        private readonly iterable $formats,
    ) {}

    public function export(Report $report, string $key): string
    {
        foreach ($this->formats as $format) {
            if ($format->key() === $key) {
                return $format->export($report);
            }
        }
        throw new \InvalidArgumentException("Unknown format: {$key}");
    }
}
// A new format = a new tagged class. ReportExporter never changes.
```

```php
// Decorator via #[AsDecorator] — add caching without touching the original
#[AsDecorator(decorates: ExchangeRateProvider::class)]
final class CachingExchangeRateProvider implements ExchangeRateProvider
{
    public function __construct(
        #[AutowireDecorated] private readonly ExchangeRateProvider $inner,
        private readonly CacheInterface $cache,
    ) {}

    public function rateFor(Currency $currency): float
    {
        return $this->cache->get((string) $currency, fn () => $this->inner->rateFor($currency));
    }
}
```

```php
// Null Object — delete scattered null checks with a do-nothing implementation
final class NullLogger implements Logger
{
    public function log(string $message): void {} // intentionally does nothing
}
```

## On Singleton

Avoid the classic Singleton — it is global mutable state in disguise, hostile to
testing and hidden coupling. In Symfony the **container already manages a single
shared instance** of each service; inject it. Reserve true Singleton only for the
rare genuinely-global concern, and even then prefer injection.

## Deeper reference

For the full catalog — every pattern with intent, when-to-use, and an original
PHP/Symfony example — load the family file for the pattern you are applying
(progressive disclosure; these are not loaded until you need them):

- `references/creational.md` — Factory Method, Abstract Factory, Builder,
  Prototype, Singleton, Object Pool.
- `references/structural.md` — Adapter, Bridge, Composite, Decorator, Facade,
  Flyweight, Proxy.
- `references/behavioral.md` — Chain of Responsibility, Command, Iterator,
  Mediator, Memento, Observer, State, Strategy, Template Method, Visitor, Null
  Object.

## Anti-patterns

- **Pattern-itis** — applying patterns to demonstrate knowledge; every problem
  gets a factory and three interfaces.
- **Forcing the fit** — bending a problem to match a pattern instead of choosing
  the pattern that fits the problem.
- **Cargo-culting** — copying a pattern's structure without the force that
  justifies it.
- **Singleton as global state** — the most-abused pattern; usually a design smell.
- **Reinventing framework patterns** — hand-rolling an event system or DI when
  Symfony provides one.

## Where this fits

`design-patterns` is a Layer 2 design reference. It supplies the concrete
mechanisms behind `solid` (Strategy for OCP, injection for DIP), is the target
`refactor-safely` refactors toward, and maps onto the native constructs documented
in `symfony`. Applied with the restraint `engineering-standards` demands ("no
premature abstraction"), it resolves real forces without adding ceremony.
