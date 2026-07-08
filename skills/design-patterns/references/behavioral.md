# Behavioral patterns

Reference for the behavioral GoF patterns — how objects interact and distribute
responsibility. Each entry: intent, when to reach for it, an original PHP example,
and the idiomatic Symfony construct where one exists. Apply with the restraint the
parent skill demands — recognize the force first, then name the pattern.

---

## Chain of Responsibility

**Intent** — pass a request along a chain of handlers until one handles it.
**Use when** — several objects may handle a request and the handler is decided at
runtime; you want to decouple sender from receiver.

```php
abstract class Handler
{
    private ?Handler $next = null;

    public function setNext(Handler $next): Handler
    {
        $this->next = $next;
        return $next;
    }

    public function handle(Request $request): ?Response
    {
        return $this->next?->handle($request);
    }
}

final class AuthHandler extends Handler
{
    public function handle(Request $request): ?Response
    {
        if (!$request->hasToken()) {
            return Response::unauthorized();
        }
        return parent::handle($request);
    }
}
```

**Symfony** — the HttpKernel event flow, Security firewall listeners, and Messenger
middleware are all chains. Add a middleware/listener rather than hand-rolling one.

---

## Command

**Intent** — encapsulate a request as an object, so it can be queued, logged, or
undone.
**Use when** — you need to decouple the thing issuing an action from the thing
performing it, or to defer/queue/record actions.

```php
final class SendWelcomeEmail
{
    public function __construct(public readonly int $userId) {}
}

final class SendWelcomeEmailHandler
{
    public function __invoke(SendWelcomeEmail $command): void
    {
        // perform the action
    }
}
```

**Symfony** — Messenger (message + handler) and Console commands *are* this pattern.
Use them; do not build a bespoke command bus.

---

## Iterator

**Intent** — access elements of a collection sequentially without exposing its
internals.
**Use when** — you want uniform traversal over a custom aggregate, or to stream
large sets lazily.

```php
final class Page implements \IteratorAggregate
{
    /** @param list<Item> $items */
    public function __construct(private readonly array $items) {}

    public function getIterator(): \Iterator
    {
        return new \ArrayIterator($this->items);
    }
}
```

**Symfony/PHP** — implement `\IteratorAggregate`/`\Iterator`; generators (`yield`)
give lazy iteration for free, and Doctrine's `toIterable()` streams query results.

---

## Mediator

**Intent** — centralize complex communication between objects so they do not refer
to each other directly.
**Use when** — a tangle of objects each know about many others; a hub reduces the
coupling.

```php
interface DialogMediator
{
    public function notify(object $sender, string $event): void;
}
// Colleagues talk to the mediator, never to each other.
```

**Symfony** — the EventDispatcher is a mediator: components emit and listen without
mutual references. Prefer it over a custom mediator for cross-cutting coordination.

---

## Memento

**Intent** — capture and restore an object's state without exposing its internals.
**Use when** — you need undo/redo, snapshots, or rollback of an object's state.

```php
final class EditorState
{
    public function __construct(public readonly string $content) {}
}

final class Editor
{
    private string $content = '';

    public function save(): EditorState
    {
        return new EditorState($this->content);
    }

    public function restore(EditorState $state): void
    {
        $this->content = $state->content;
    }
}
```

---

## Observer

**Intent** — notify dependents automatically when an object changes state.
**Use when** — one change should trigger reactions in decoupled parts of the system.

```php
final class OrderPlaced
{
    public function __construct(public readonly int $orderId) {}
}

#[AsEventListener]
final class SendConfirmationOnOrderPlaced
{
    public function __invoke(OrderPlaced $event): void
    {
        // react to the event
    }
}
```

**Symfony** — the EventDispatcher with `#[AsEventListener]` is the idiomatic Observer.
Never hand-roll subject/observer wiring.

---

## State

**Intent** — let an object alter its behavior when its internal state changes; it
appears to change class.
**Use when** — behavior depends on state and you have sprawling conditionals on a
status field.

```php
interface OrderState
{
    public function pay(Order $order): void;
    public function cancel(Order $order): void;
}

final class PendingState implements OrderState { /* legal moves from pending */ }
final class PaidState implements OrderState    { /* legal moves from paid */ }
```

**Symfony** — the Workflow component models states and transitions declaratively;
prefer it for real domain state machines.
**Watch for** — State vs. Strategy share this structure but differ in intent:
Strategy's caller picks an interchangeable algorithm; State's object switches its
*own* behavior as its state evolves.

---

## Strategy

**Intent** — define a family of interchangeable algorithms and select one at runtime.
**Use when** — a behavior varies along one axis and callers (or config) choose which.

```php
interface PricingStrategy
{
    public function price(Cart $cart): Money;
}

final class Checkout
{
    public function __construct(private readonly PricingStrategy $strategy) {}

    public function total(Cart $cart): Money
    {
        return $this->strategy->price($cart);
    }
}
```

**Symfony** — inject implementations via tagged services and select by key (see the
parent `SKILL.md` hero example). The idiomatic Open/Closed fix.

---

## Template Method

**Intent** — define the skeleton of an algorithm, deferring specific steps to
subclasses.
**Use when** — several variants share an overall sequence but differ in individual
steps.

```php
abstract class ImportJob
{
    final public function run(): void
    {
        $rows  = $this->read();
        $valid = $this->validate($rows);
        $this->persist($valid);
    }

    abstract protected function read(): array;
    abstract protected function validate(array $rows): array;
    abstract protected function persist(array $rows): void;
}
```

**Watch for** — if the steps vary independently, prefer composition (Strategy);
Template Method locks the skeleton behind inheritance, which is a stronger claim.

---

## Visitor

**Intent** — add new operations to an object structure without modifying its classes.
**Use when** — a stable class hierarchy needs many distinct, unrelated operations.

```php
interface NodeVisitor
{
    public function visitText(TextNode $node): string;
    public function visitImage(ImageNode $node): string;
}

interface Node
{
    public function accept(NodeVisitor $visitor): string;
}
```

**Watch for** — costly when the hierarchy changes often: every new node type forces
every visitor to change. Great for stable trees (ASTs), poor for evolving ones.

---

## Null Object

**Intent** — provide a do-nothing implementation of an interface to eliminate null
checks.
**Use when** — callers would otherwise litter the code with `if ($x !== null)`.

```php
interface Logger
{
    public function log(string $message): void;
}

final class NullLogger implements Logger
{
    public function log(string $message): void {} // intentionally empty
}
```

**Watch for** — do not hide real errors behind a silent no-op; use it only where
"nothing happens" is genuinely correct behavior.
