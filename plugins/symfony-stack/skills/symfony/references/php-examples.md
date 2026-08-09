# PHP worked examples for core design/quality skills

The stack-agnostic `solid`, `refactoring-catalog`, and `code-quality` skills
illustrate their points with language-neutral pseudocode. This file carries the
original PHP/Symfony-flavored versions of those same examples, for when a
concrete PHP illustration is more useful than the pseudocode. Not loaded until
referenced — progressive disclosure.

## `solid` — SOLID principles

### S — Single Responsibility

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

## `refactoring-catalog` — key transformations

### Extract Method

```php
// Before
public function printReport(Order $order): void
{
    echo "Order #{$order->id()}\n";
    $total = 0;
    foreach ($order->lines() as $line) {
        $total += $line->quantity() * $line->price();
    }
    echo "Total: {$total}\n";
}

// After — each step named
public function printReport(Order $order): void
{
    $this->printHeader($order);
    $this->printTotal($this->calculateTotal($order));
}
```

### Replace Nested Conditional with Guard Clauses

```php
// Before
public function payAmount(Employee $e): int
{
    if ($e->isSeparated()) {
        $result = 0;
    } else {
        if ($e->isRetired()) {
            $result = 0;
        } else {
            $result = $this->normalPay($e);
        }
    }
    return $result;
}

// After
public function payAmount(Employee $e): int
{
    if ($e->isSeparated()) {
        return 0;
    }
    if ($e->isRetired()) {
        return 0;
    }
    return $this->normalPay($e);
}
```

### Replace Conditional with Polymorphism

```php
// Before
public function speed(Bird $bird): float
{
    return match ($bird->type()) {
        'european'  => $this->baseSpeed(),
        'african'   => $this->baseSpeed() - $bird->load(),
        'norwegian' => $bird->isNailed() ? 0.0 : $this->baseSpeed(),
    };
}

// After
interface Bird { public function speed(): float; }
final class EuropeanSwallow implements Bird { public function speed(): float { /* ... */ } }
final class AfricanSwallow  implements Bird { public function speed(): float { /* ... */ } }
```

### Introduce Parameter Object

```php
// Before
public function findOrders(\DateTimeImmutable $from, \DateTimeImmutable $to): array {}

// After
public function findOrders(DateRange $range): array {}

final readonly class DateRange
{
    public function __construct(
        public \DateTimeImmutable $from,
        public \DateTimeImmutable $to,
    ) {}
}
```

### Replace Primitive with Value Object

```php
// Before
final class User
{
    public function __construct(private string $email) {}
}

// After
final readonly class Email
{
    public function __construct(public string $value)
    {
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid email: {$value}");
        }
    }
}

final class User
{
    public function __construct(private Email $email) {}
}
```

## `code-quality` — craft examples

### Naming

```php
// Before — names hide intent
$d = (new \DateTimeImmutable())->diff($sub->renewsAt)->days;
if ($d < 0) {
    $this->process($sub);
}

// After — names are the documentation
$daysUntilRenewal = (new \DateTimeImmutable())->diff($sub->renewsAt)->days;
if ($daysUntilRenewal < 0) {
    $this->chargeExpiredSubscription($sub);
}
```

### No flag arguments

```php
// Before — a boolean that switches behavior; callers read save(true) and guess
public function save(User $user, bool $andFlush): void { /* ... */ }

// After — two methods, each named for what it does
public function save(User $user): void { /* ... */ }
public function saveAndFlush(User $user): void { /* ... */ }
```

### Error handling

```php
// Before — failure vanishes silently
try {
    $this->gateway->charge($order);
} catch (\Throwable) {
    // nothing
}

// After — handle, or rethrow with context
try {
    $this->gateway->charge($order);
} catch (GatewayException $e) {
    throw new PaymentFailed("Charge failed for order {$order->id()}", previous: $e);
}
```

### Tell, don't ask

```php
// Before — pull state out, decide, push it back (Feature Envy)
if ($account->balance() >= $amount) {
    $account->setBalance($account->balance() - $amount);
}

// After — tell the object; it guards its own invariant
$account->withdraw($amount);
```
