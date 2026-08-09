---
name: symfony-patterns
description: >-
  The Gang of Four design patterns as idiomatically applied in Symfony — which
  pattern maps to which native Symfony construct (tagged services, service
  decoration, EventDispatcher, Messenger, DI factories), with worked PHP
  examples. Companion to the stack-agnostic `design-patterns` skill: use that
  one to pick the pattern, this one for the Symfony-idiomatic way to build it.
  Triggers: "design pattern" + "Symfony", "tagged service", "AsDecorator",
  "EventDispatcher", "which pattern in Symfony".
---

# Symfony patterns

Symfony already embodies many GoF patterns; recognizing them tells you the
idiomatic place to plug in rather than reinventing:

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

## Reinventing framework patterns

Hand-rolling an event system or a DI container is an anti-pattern here —
Symfony already provides one. Reach for the native construct above before
reaching for a hand-built version of it.

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

## Where this fits

`symfony-patterns` is the Symfony-idiomatic companion to the stack-agnostic
`design-patterns` skill: that skill supplies the problem→pattern selection
guide and the general principles, this one supplies the native Symfony
construct and a worked PHP example once a pattern is chosen.
