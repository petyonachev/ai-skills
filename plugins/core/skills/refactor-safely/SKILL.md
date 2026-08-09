---
name: refactor-safely
description: >-
  Workflow for changing code structure without changing its behavior, protected
  by tests, in small verified steps. Use when cleaning up code, reducing
  complexity, extracting methods or classes, decoupling, renaming, removing
  duplication, or paying down technical debt. Establishes a test safety net
  first, transforms in small reversible steps, and proves behavior is unchanged.
  Triggers: "refactor", "clean this up", "reduce complexity", "extract", "split
  this class", "decouple", "remove duplication", "pay down tech debt", "improve
  this code".
---

# Refactor safely

**The prime directive: refactoring changes structure, never behavior.** Same
inputs produce the same outputs before and after — that is the entire safety
property, and every rule here exists to protect it. The moment you change what
the code *does*, you are no longer refactoring; you are writing a feature or
fixing a bug, and that must be a separate, separately-verified change.

Refactoring without tests is not refactoring — it is editing and hoping. The tests
are what let you move fast; without them, every change is a gamble on behavior you
cannot see.

## The workflow

### 1. Name the smell and the target

Refactor toward something, not away from boredom. State what is wrong (the code
smell — long method, duplication, feature envy, shotgun surgery) and what
structure you are moving toward. If you cannot name both, you are not refactoring,
you are fiddling. See the `refactoring-catalog` for the smell → technique
mapping.

### 2. Establish the safety net

You cannot safely refactor code you cannot test. Before touching anything:

- If the code has solid tests covering the behavior, run them and confirm green —
  that is your net.
- If coverage is thin, **write characterization tests first**: tests that pin the
  *current* behavior exactly as it is, even if that behavior looks wrong. You are
  not judging the behavior yet, only capturing it so you notice if a refactor
  changes it. Fixing "wrong" behavior is a separate change made *after* the
  refactor, against its own test.

No net, no refactor. This gate is not optional.

### 3. Understand the blast radius

Know everything the code you are changing touches: callers, subclasses, tests,
serialized data, public contracts. For a wide-reaching refactor (renaming a
widely-used symbol, changing a shared signature), fan out investigation agents
(`parallel-agents`) to map every usage before you start — a refactor that
compiles locally but breaks 30 call sites is not safe.

### 4. Transform in small, reversible steps

Run the `iterate` loop, one transformation at a time:

```
apply one small transformation → run the tests → green? → commit → next
```

Each step is a single named refactoring (extract method, introduce parameter
object, replace conditional with polymorphism). After each, the tests must be
green and the code committable. Never batch ten transformations and run the tests
once at the end — when it goes red you will not know which step broke it.

### 5. Verify behavior is unchanged

Hand off to `verify`: the same tests that passed before pass after, with no new
failures and none weakened to force green. For code without deterministic tests,
compare observed behavior before and after directly. Unchanged behavior is the
proof; anything else means you changed something you did not mean to.

### 6. Stop when the smell is gone

Refactoring is bottomless — there is always something more to polish. Stop when
the smell that motivated the work is resolved. Do not gold-plate, do not
rearchitect adjacent code that was not in scope, do not chase perfection. Minimal
blast radius applies to cleanup too.

## Never mix refactoring with behavior change

This is the one rule that most protects the safety net. A commit is *either* a
refactor (tests unchanged, still green) *or* a behavior change (tests change to
express new behavior) — never both. Mixing them destroys the property that green
tests prove the refactor was safe, because now you cannot tell whether a test
change reflects intended new behavior or an accidental regression.

If you must do both, sequence them: refactor to make the change easy (tests stay
green), commit, *then* make the easy change (tests change), commit.

## Refactor, rewrite, or leave it

Not everything should be refactored in place. Choose deliberately:

- **Refactor** when the structure is salvageable and each step keeps tests green.
- **Rewrite** when the design is fundamentally wrong and incremental steps only
  entrench it — but a rewrite is a feature-scale effort with its own plan and
  risk; do not smuggle it in under "refactoring." Escalate to `planning`.
- **Leave it** when the code is ugly but stable, untested, and not in your way. A
  risky refactor of working code you do not need to touch is negative value.

If a refactor balloons past its plan or starts requiring behavior changes to
proceed, stop and re-plan rather than pushing through (`iterate`, stop
conditions).

## Calibration — worked examples

| Task | Net needed | Agents | Steps | Notes |
|---|---|---|---|---|
| Rename a private variable in one method | Existing tests | No | 1 | Trivial and contained. |
| Extract a method from a long function | Existing tests | No | Few | Classic small refactor; loop per extraction. |
| Rename a public method used across the app | Characterize + full suite | Yes | Many | Map every call site first; risk is the blast radius. |
| Split a 500-line service into collaborators | Characterize first | Maybe | Many | Thin coverage likely — build the net before touching. |
| Untangle a class with no tests | **Write tests first** | Maybe | Many | The net *is* the first phase; do not skip to editing. |
| Replace a fundamentally wrong design | — | — | — | This is a rewrite: escalate to `planning`, not a refactor. |

The pattern: **the weaker the existing tests and the wider the usage, the more of
your effort goes into the safety net and blast-radius mapping before a single line
of structure changes.**

## Anti-patterns

- **Refactoring without tests** — editing and hoping; the cardinal sin.
- **Mixing behavior change into a refactor** — destroys the safety property.
- **Big-bang transformation** — many changes, one test run; unfindable breakage.
- **Aimless cleanup** — refactoring with no named smell or target.
- **Weakening tests to go green** — faking the proof that behavior is unchanged.
- **Scope creep** — rearchitecting code that was never in scope.
- **Rewrite disguised as refactor** — a feature-scale rewrite with no plan and no
  budget.

## Where this fits

`refactor-safely` is a Layer 1 workflow. It leans hard on `iterate` (the
step-verify-commit loop) and `verify` (behavior unchanged), uses `parallel-agents`
to map blast radius, and draws its techniques and smell definitions from the
`refactoring-catalog`, with `code-quality`, `solid`, and `design-patterns` as the
targets it refactors toward. When behavior must change, it hands back to
`feature-delivery` or `debug`.
