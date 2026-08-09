---
name: code-quality
description: >-
  The craft of writing and reviewing readable, changeable, honest code — naming,
  function design, control flow, comments, error handling, immutability, and
  cohesion/coupling. Use when writing new code, reviewing a diff for quality
  (not correctness — that's code-review/verify), improving clarity, or judging
  whether code is maintainable. This is the craft layer beneath
  engineering-standards and the lens for reviewing new code. Triggers: "code
  quality", "is this readable", "clean this up for clarity", "review this code",
  "naming", "clean code", "improve maintainability".
---

# Code quality

Code is read far more often than it is written, and changed far more often than
it is read from scratch. Quality means optimizing for the next person (usually
you, in six months): code that is **readable, changeable, and honest** — it does
what it appears to do, with no hidden surprises. This is the craft layer beneath
`engineering-standards`; that skill says *follow the standard*, this one says
*here is what good looks like*.

## Naming

Names are the primary documentation. Get them right and comments become mostly
unnecessary.

- **Reveal intent** — `daysSinceLastOrder`, not `d`. The name says what and why.
- **Searchable and pronounceable** — no cryptic abbreviations; single letters only
  in the tiniest scopes (a loop index).
- **Consistent vocabulary** — one word per concept across the codebase (`fetch`
  vs `get` vs `retrieve` — pick one).
- **Avoid disinformation** — do not call it a `List` if it is a `Map`; do not
  imply a type or behavior it does not have.
- Name booleans and methods as questions/actions: `isActive`, `hasExpired`,
  `calculateTotal`.

```
// Before — names hide intent
d = daysBetween(now(), sub.renewsAt)
if (d < 0) {
    process(sub)
}

// After — names are the documentation
daysUntilRenewal = daysBetween(now(), sub.renewsAt)
if (daysUntilRenewal < 0) {
    chargeExpiredSubscription(sub)
}
```

## Functions and methods

- **Small, one thing** — a method does one thing at one level of abstraction. If
  you can extract a meaningfully-named method from inside it, it was doing two
  things.
- **Few parameters** — 0–3 ideal; more usually means a missing parameter object
  (`refactoring-catalog`, Data Clumps).
- **No flag arguments** — a boolean parameter that switches behavior is two methods
  wearing a trench coat; split them.
- **Command-query separation** — a method either does something or answers
  something, not both. Getters do not mutate.
- **No surprising side effects** — the name must account for everything the method
  does.

```
// Before — a boolean that switches behavior; callers read save(user, true) and guess
function save(user: User, andFlush: boolean): void { /* ... */ }

// After — two methods, each named for what it does
function save(user: User): void { /* ... */ }
function saveAndFlush(user: User): void { /* ... */ }
```

## Control flow

- **Guard clauses over nesting** — return early on the exceptional cases; keep the
  happy path at the left margin. Deep nesting is a readability tax.
- **Fail fast** — validate preconditions up front and reject bad input immediately.
- Prefer a flat sequence of small steps to a pyramid of conditionals.

## Comments

- Explain **why**, not **what** — the code says what; a comment justifies a
  non-obvious decision, a workaround, a constraint.
- A comment that explains *what* unclear code does is a smell — rename and extract
  until the code explains itself, then delete the comment
  (`refactoring-catalog`, Comments smell).
- **Delete commented-out code.** Version control remembers it.
- Per `engineering-standards`: do not add docblocks or comments to code you did
  not change.

## Error handling

- Use exceptions, not error codes or nulls, for exceptional conditions.
- **Never swallow errors** — no empty `catch`. If you catch, handle or rethrow with
  context.
- Do not use exceptions for ordinary control flow.
- Messages must help the reader diagnose: what failed, with what input.
- Fail loudly in development, degrade gracefully in production
  (`engineering-standards`).

```
// Before — failure vanishes silently
try {
    gateway.charge(order)
} catch (e) {
    // nothing
}

// After — handle, or rethrow with context
try {
    gateway.charge(order)
} catch (GatewayException e) {
    throw new PaymentFailed("Charge failed for order " + order.id(), cause: e)
}
```

## State and immutability

- Prefer immutability: `readonly` properties, value objects, no setters where a
  new instance will do. Immutable objects are simpler to reason about and safe to
  share.
- Minimize mutable state and its scope; the less that can change, the less that can
  go wrong.
- Model domain concepts as value objects rather than bare primitives (`solid`,
  Primitive Obsession).

## Cohesion and coupling

- **High cohesion** — things that change together live together; a class's members
  should relate to one purpose.
- **Low coupling** — depend on abstractions and on as little as possible
  (`solid`, DIP).
- **Tell, don't ask** — tell an object to do something rather than pulling its
  data out to act on it (avoids Feature Envy).
- **Law of Demeter** — talk to immediate collaborators, not their internals; long
  `a.getB().getC()` chains are a coupling smell.

```
// Before — pull state out, decide, push it back (Feature Envy)
if (account.balance() >= amount) {
    account.setBalance(account.balance() - amount)
}

// After — tell the object; it guards its own invariant
account.withdraw(amount)
```

## Duplication — with a caveat

DRY: extract genuinely-shared logic. But **duplication is far cheaper than the
wrong abstraction** — two things that merely *look* alike today may diverge
tomorrow. Wait until the duplication is real and stable before unifying it, in
line with `engineering-standards` ("no premature abstraction"). Deduplicating
coincidental similarity creates coupling that hurts more than the duplication did.

## Review checklist

When reviewing new code for quality (correctness and bugs are `verify`/`debug`):

- Names reveal intent; no disinformation.
- Methods small, single-purpose, few parameters, no flag args.
- Happy path flat; guard clauses handle the rest.
- No swallowed errors; messages are diagnostic.
- No commented-out or dead code; comments explain *why*.
- State minimized; domain concepts modelled, not primitive-obsessed.
- Cohesive units, loose coupling, no message chains.
- Duplication is real, not coincidental, before it is abstracted away.

## Anti-patterns

- **Clever over clear** — terse code that impresses and confuses.
- **What-comments** — narrating unclear code instead of clarifying it.
- **Boolean-flag methods** and functions that do two things.
- **Arrow code** — deeply nested conditionals.
- **Swallowed exceptions** — silent failure.
- **Speculative abstraction** — DRYing up things that only look alike.

## Where this fits

`code-quality` is the Layer 2 craft reference beneath `engineering-standards`. It
is the lens `feature-delivery`'s self-review and `finish-branch`'s diff review
apply, a criteria skill the `code-review` workflow dispatches, and the target
`refactor-safely` moves code toward; it leans on `solid`, `design-patterns`, and
`refactoring-catalog` for the structural moves. For correctness and bug-finding,
use `verify` and `debug`.
