# Creational patterns

Reference for the creational GoF patterns — how objects are made, decoupling
construction from use. Each entry: intent, when to reach for it, an original PHP
example, and the idiomatic Symfony construct where one exists. In a Symfony app
the DI container handles most creation concerns; reach for an explicit pattern
only when the container does not already solve it.

---

## Factory Method

**Intent** — define an interface for creating an object, letting the implementation
decide which concrete class to instantiate.
**Use when** — creation logic varies or is non-trivial and should not be hardwired
at the call site.

```php
interface PaymentGatewayFactory
{
    public function create(): PaymentGateway;
}

final class StripeGatewayFactory implements PaymentGatewayFactory
{
    public function __construct(private readonly string $apiKey) {}

    public function create(): PaymentGateway
    {
        return new StripeGateway($this->apiKey);
    }
}
```

**Symfony** — use a DI service factory (`factory:` in config or `#[Autowire]`) when
construction needs logic; the container is your factory.
**Watch for** — do not wrap a plain `new` in a factory; only when creation truly
varies or is complex.

---

## Abstract Factory

**Intent** — create families of related objects without naming their concrete
classes.
**Use when** — you must produce matching *sets* of objects that have to be used
together (a whole storage backend, a themed UI kit).

```php
interface StorageFactory
{
    public function fileStore(): FileStore;
    public function metadataStore(): MetadataStore;
}

final class S3StorageFactory implements StorageFactory
{
    public function fileStore(): FileStore { return new S3FileStore(); }
    public function metadataStore(): MetadataStore { return new DynamoMetadataStore(); }
}
```

**Watch for** — heavy machinery, justified only when the product family must stay
internally consistent. DI configuration (binding a set of interfaces per
environment) often achieves the same thing more simply.

---

## Builder

**Intent** — construct a complex object step by step.
**Use when** — an object has many optional parts and a telescoping constructor would
be unreadable.

```php
final class ReportBuilder
{
    /** @var list<string> */
    private array $columns = [];
    private ?DateRange $range = null;

    public function withColumn(string $name): self
    {
        $this->columns[] = $name;
        return $this;
    }

    public function forRange(DateRange $range): self
    {
        $this->range = $range;
        return $this;
    }

    public function build(): Report
    {
        return new Report($this->columns, $this->range);
    }
}
```

**Symfony** — Doctrine's `QueryBuilder` and the Form builder are Builders.
**Watch for** — for an immutable value object with a few fields, named constructors
or `readonly` + named arguments are simpler than a builder.

---

## Prototype

**Intent** — create new objects by cloning a configured existing instance.
**Use when** — building fresh is expensive and copying a template is cheaper.

```php
final class Template
{
    /** @param array<string, mixed> $variables */
    public function __construct(public string $body, public array $variables) {}

    public function __clone(): void
    {
        // deep-copy any mutable members here
    }
}

$draft = clone $template;
```

**Watch for** — PHP's `clone` is *shallow*: implement `__clone()` to deep-copy
mutable objects/collections, or clones will share references and corrupt each other.

---

## Singleton

**Intent** — guarantee a class has exactly one instance with a global access point.
**Use when** — almost never in a Symfony application.

```php
// Avoid this. Shown for recognition, not endorsement.
final class Config
{
    private static ?self $instance = null;

    public static function instance(): self
    {
        return self::$instance ??= new self();
    }

    private function __construct() {}
}
```

**Symfony** — the container already provides a single shared instance of every
service. Inject it. True Singleton is global mutable state: hidden coupling,
hostile to tests.
**Watch for** — the most-abused pattern. If you are reaching for it, you almost
certainly want a service.

---

## Object Pool

**Intent** — reuse a set of initialized objects instead of repeatedly creating and
destroying them.
**Use when** — creation is genuinely expensive and instances are interchangeable
(connections, worker handles).

```php
final class ConnectionPool
{
    /** @var list<Connection> */
    private array $idle = [];

    public function acquire(): Connection
    {
        return array_pop($this->idle) ?? $this->create();
    }

    public function release(Connection $connection): void
    {
        $this->idle[] = $connection;
    }

    private function create(): Connection { /* expensive setup */ }
}
```

**Watch for** — rarely needed at the application layer: DB drivers, HTTP clients,
and the runtime already pool for you. Misused, it adds lifecycle bugs (leaked or
double-released objects) for no measured gain.
