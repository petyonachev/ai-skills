# Structural patterns

Reference for the structural GoF patterns — how classes and objects are composed
into larger structures. Each entry: intent, when to reach for it, an original PHP
example, and the idiomatic Symfony construct where one exists.

---

## Adapter

**Intent** — convert one interface into another that the client expects.
**Use when** — integrating a class or library whose interface does not match yours.

```php
// Your domain expects this
interface Notifier
{
    public function notify(User $user, string $message): void;
}

// Adapt a third-party SDK to it
final class TwilioNotifier implements Notifier
{
    public function __construct(private readonly TwilioClient $client) {}

    public function notify(User $user, string $message): void
    {
        $this->client->messages->create($user->phone(), ['body' => $message]);
    }
}
```

**Symfony** — the anti-corruption layer around a third party (`integration-build`)
is Adapter. It keeps the provider's field names and quirks out of your domain and
makes the provider swappable and fakeable.

---

## Bridge

**Intent** — decouple an abstraction from its implementation so the two vary
independently.
**Use when** — variation runs along two axes (report types × output formats) and a
class-per-combination would explode.

```php
interface Renderer // the implementation axis
{
    public function render(array $data): string;
}

abstract class Report // the abstraction axis
{
    public function __construct(protected readonly Renderer $renderer) {}

    abstract public function export(): string;
}
```

**Watch for** — not the same as Adapter: Adapter reconciles a mismatch *after the
fact*; Bridge is a deliberate up-front separation of two dimensions of change.

---

## Composite

**Intent** — compose objects into trees and let clients treat individual objects and
compositions uniformly.
**Use when** — you have part-whole hierarchies and want callers to ignore the
leaf-vs-branch distinction.

```php
interface Component
{
    public function price(): Money;
}

final class Product implements Component
{
    public function price(): Money { /* ... */ }
}

final class Bundle implements Component
{
    /** @param list<Component> $items */
    public function __construct(private readonly array $items) {}

    public function price(): Money
    {
        return array_reduce(
            $this->items,
            static fn (Money $sum, Component $c) => $sum->add($c->price()),
            Money::zero(),
        );
    }
}
```

**Watch for** — worth it only when clients genuinely treat leaves and composites the
same; otherwise it adds indirection for no gain.

---

## Decorator

**Intent** — attach responsibilities to an object dynamically by wrapping it in
another with the same interface.
**Use when** — you want to add behavior (caching, logging, retry, metrics) without
subclassing or editing the original.

```php
final class CachingExchangeRateProvider implements ExchangeRateProvider
{
    public function __construct(
        private readonly ExchangeRateProvider $inner,
        private readonly CacheInterface $cache,
    ) {}

    public function rateFor(Currency $currency): float
    {
        return $this->cache->get((string) $currency, fn () => $this->inner->rateFor($currency));
    }
}
```

**Symfony** — `#[AsDecorator]` + `#[AutowireDecorated]` (see the parent `SKILL.md`).
The idiomatic way to layer behavior onto an existing service.
**Watch for** — Decorator *adds* behavior (same interface); Proxy *controls access*
(same behavior). Different intent, same shape.

---

## Facade

**Intent** — provide a simple, task-focused interface over a complex subsystem.
**Use when** — callers need a small API over a tangle of collaborators.

```php
final class Checkout // hides pricing, inventory, payment, notification
{
    public function __construct(
        private readonly PricingStrategy $pricing,
        private readonly Inventory $inventory,
        private readonly PaymentGateway $payment,
    ) {}

    public function purchase(Cart $cart): Receipt
    {
        // orchestrate the subsystem behind one method
    }
}
```

**Symfony** — application/service-layer classes often act as facades over domain
collaborators.
**Watch for** — a facade that keeps growing becomes a god object (SRP violation);
keep it a thin coordinator, not a home for logic.

---

## Flyweight

**Intent** — share common intrinsic state across many objects to save memory.
**Use when** — you hold huge numbers of objects with heavily duplicated state.

```php
final class CurrencyRegistry
{
    /** @var array<string, Currency> */
    private array $flyweights = [];

    public function get(string $code): Currency
    {
        return $this->flyweights[$code] ??= new Currency($code);
    }
}
```

**Watch for** — a memory optimization only. Reach for it under measured memory
pressure, not speculatively; otherwise it is premature optimization.

---

## Proxy

**Intent** — provide a placeholder that controls access to another object (lazy
creation, access control, remote stand-in).
**Use when** — you need to defer an expensive creation, guard access, or represent a
remote object locally.

```php
final class LazyReportProxy implements Report
{
    private ?Report $real = null;

    /** @param \Closure(): Report $factory */
    public function __construct(private readonly \Closure $factory) {}

    public function rows(): array
    {
        return ($this->real ??= ($this->factory)())->rows();
    }
}
```

**Symfony/Doctrine** — Doctrine uses lazy-loading proxies for entity associations,
and Symfony supports lazy services; you rarely hand-write one.
**Watch for** — distinguish from Decorator: Proxy keeps the same behavior and
controls *access*; Decorator changes behavior by *adding* to it.
