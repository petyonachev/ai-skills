# Dependency injection and configuration

The container is the heart of Symfony. Used idiomatically it wires your objects
invisibly; misused, it becomes a global service locator that hides dependencies
and breaks testability. The rules below are the house-idiomatic way on 7.x /
PHP 8.3+.

---

## Autowiring is the default

Autowiring + autoconfiguration are on. Type-hint a dependency in the constructor
and the container provides it — no configuration needed for the common case.

```php
final class PlaceOrder
{
    public function __construct(
        private readonly OrderRepository $orders,
        private readonly EventDispatcherInterface $events,
    ) {}
}
```

Services are `private` and `final` by default. Keep them **stateless** — a service
is shared, so mutable per-request state is a bug.

---

## Injecting specific values with #[Autowire]

When the type alone is not enough — a scalar, an env var, a specific service, or an
expression — use `#[Autowire]`:

```php
public function __construct(
    #[Autowire('%env(int:MAX_RETRIES)%')] private readonly int $maxRetries,
    #[Autowire(service: 'monolog.logger.payments')] private readonly LoggerInterface $log,
    #[Autowire(expression: "service('router').generate('home')")] private readonly string $homeUrl,
) {}
```

---

## Tagged services (Strategy)

To inject a *set* of implementations selected at runtime — the idiomatic
Open/Closed fix (`design-patterns`, Strategy) — tag by interface and inject the
collection:

```php
// Autoconfigure: every ExportFormat is tagged 'app.export_format'
// config/services.yaml
//   _instanceof:
//     App\Export\ExportFormat:
//       tags: ['app.export_format']

final class ReportExporter
{
    /** @param iterable<ExportFormat> $formats */
    public function __construct(
        #[AutowireIterator('app.export_format')]
        private readonly iterable $formats,
    ) {}
}
```

Use `#[AutowireLocator(...)]` instead when you want lazy, keyed access rather than
iterating all of them.

---

## Factories

When construction needs logic, use a factory rather than smuggling it into the
service:

```php
// config/services.yaml
//   App\Payment\GatewayClient:
//     factory: ['@App\Payment\GatewayClientFactory', 'create']

final class GatewayClientFactory
{
    public function __construct(#[Autowire('%env(GATEWAY_DSN)%')] private readonly string $dsn) {}

    public function create(): GatewayClient
    {
        return new GatewayClient(Dsn::parse($this->dsn));
    }
}
```

---

## Service decoration

Wrap a service to add behavior (caching, logging) without touching it
(`design-patterns`, Decorator):

```php
#[AsDecorator(decorates: ExchangeRateProvider::class)]
final class CachingExchangeRateProvider implements ExchangeRateProvider
{
    public function __construct(
        #[AutowireDecorated] private readonly ExchangeRateProvider $inner,
        private readonly CacheInterface $cache,
    ) {}
    // ...
}
```

---

## Parameters and env vars

- **Env vars** carry deploy-varying config; use typed processors to cast them:
  `%env(int:PORT)%`, `%env(bool:FEATURE_X)%`, `%env(csv:HOSTS)%`,
  `%env(json:SETTINGS)%`, `%env(default:fallback:SOME_VAR)%`.
- `.env` holds **non-secret defaults** only. Real secrets go in the **secrets
  vault** (`secrets:set`) — never in `.env`, code, or logs (`security`).
- Reserve container *parameters* for genuinely static, app-wide constants; prefer
  injecting env vars directly with `#[Autowire]` over defining a parameter for
  everything.

---

## Compiler passes (rare)

Reach for a compiler pass only to manipulate the container at build time — e.g.
collecting tagged services when `#[AutowireIterator]` is not enough, or altering
definitions. Most applications never need one; if you are writing one for ordinary
wiring, step back.

```php
final class RegisterHandlersPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        // adjust definitions based on tags at compile time
    }
}
```

---

## Pitfalls

- **Injecting `ContainerInterface`** and pulling services out — the service-locator
  anti-pattern. Inject exactly what you need.
- **Making services `public`/`static`** to access them globally — breaks
  testability and hides dependencies.
- **Stateful services** — storing per-request data on a shared service.
- **Over-configuration** — YAML wiring for what autowiring already handles.
- **Secrets in parameters or `.env`** — use the vault.
