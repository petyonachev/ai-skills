---
name: architecture
description: >-
  Making the architectural decisions that are expensive to change — component
  boundaries, dependency direction, technology choices, and how to evolve a
  mixed/evolving codebase deliberately rather than by accident. Use when designing
  system structure, deciding where a boundary goes, choosing a datastore or
  library, splitting or merging modules, weighing monolith vs. services, or
  writing an Architecture Decision Record. Emphasizes reversibility and pragmatic
  evolution over purity. Triggers: "architecture", "system design", "component
  boundaries", "should we use X", "ADR", "decouple this", "split this module",
  "monolith vs microservices", "technology choice".
---

# Architecture

Architecture is the set of decisions that are expensive to reverse. The job is
not to make every decision "correct" by some ideal — it is to spend deliberation
where reversal is costly, and to keep the cheap decisions cheap. In a
**mixed/evolving codebase**, the goal is explicitly not purity; it is deliberate
evolution: knowing which direction is better and moving that way in safe steps,
without pretending you can rewrite the world.

## The reversibility lens

Before deliberating, ask how hard the decision is to undo:

- **Two-way door** (reversible) — an implementation choice hidden behind an
  interface, a folder name, a library you can swap in a day. Decide quickly, pick
  a sane default, move on. Deliberating these is waste.
- **One-way door** (costly to reverse) — a datastore, a public API contract, a
  module split, a message broker, a cross-cutting pattern adopted repo-wide. These
  earn real deliberation and usually an ADR.

Match the weight of the decision to the cost of being wrong. Most decisions are
two-way doors dressed up as one-way doors.

## Boundaries and dependencies

The core of architecture is *where the lines go* and *which way the arrows point*.

- **High cohesion, low coupling** — put things that change together on the same
  side of a boundary; keep things that change for different reasons apart. Draw
  boundaries along the seams where change rates or responsibilities differ.
- **Dependency direction** — stable, abstract things should not depend on
  volatile, concrete ones. Domain logic must not depend on the framework, the
  database, or a third party; those depend inward, through interfaces (`solid`
  DIP). This is what keeps business rules testable and infrastructure swappable.
- **Depend on abstractions at boundaries** — the anti-corruption layer around a
  third party (`integration-build`) and the repository interface in front of the
  ORM are the same idea: the outside world does not leak in.

## Deciding in a mixed/evolving codebase

This is the situation you are actually in, and it has its own rules:

- **Match the module you are in.** Follow the local conventions and structure of
  the code you are changing, even if another module does it differently. Consistency
  within a boundary beats global uniformity imposed one file at a time.
- **Do not import a foreign style.** Dropping a hexagonal domain layer into a
  module built as classic layered MVC creates a confusing hybrid that is worse
  than either. Pick the module's existing idiom.
- **Evolve deliberately, never half-way.** If a module's structure is genuinely
  wrong, improving it is a *decision* with a plan (strangler fig: build the new
  path alongside the old, migrate callers incrementally, delete the old), not a
  drive-by half-migration that leaves two competing structures.
- **Surface standardization as a choice.** When you think the codebase should
  converge on one approach, raise it and record it (ADR) — do not silently push
  your preference into whatever you happen to touch (`engineering-standards`).

## Technology choices

- Evaluate on **fit, reversibility, operational cost, and team familiarity** — not
  novelty. A boring, proven tool the team knows beats an exciting one nobody
  operates.
- **YAGNI on infrastructure.** Do not add a queue, cache, search cluster, or new
  service until a real requirement demands it. Each is operational weight forever.
- Prefer the **monolith until it hurts.** Microservices trade in-process calls for
  network calls, transactions for eventual consistency, and one deploy for many —
  buy that complexity only when team scale or independent-scaling needs justify it,
  never for greenfield tidiness. A well-modularized monolith gets most of the
  benefit at a fraction of the cost.

## Architecture Decision Records

Write an ADR when a decision is significant and hard to reverse — future you will
want to know *why*. Keep it lightweight:

```
# ADR-NNN: <title>

## Context
<the forces: what problem, what constraints, what we know>

## Decision
<what we chose>

## Alternatives considered
<the other options and why they lost>

## Consequences
<what this makes easy, what it makes hard, what we accept>
```

An ADR nobody reads is waste — keep them short, in-repo, and only for decisions
that genuinely constrain the future.

## Calibration — worked examples

| Decision | Deliberate? | ADR? | Notes |
|---|---|---|---|
| Which folder/namespace a new class goes in | No | No | Follow the local convention. |
| Reversible detail behind an interface | No | No | Two-way door — pick and move. |
| A small schema tweak | Light | No | See `database-design`. |
| A naming/structure convention for a new module | Light | Maybe | Cheap now, sticky later. |
| Introduce a new layer/boundary in a module | Yes | Yes | Changes the dependency rules. |
| Choose a datastore / broker / major library | Yes | Yes | One-way door, operational cost. |
| Split a module or extract a service | Yes | Yes | High blast radius; plan the strangler. |
| Adopt a pattern repo-wide (e.g. CQRS everywhere) | Yes | Yes | Affects everyone; needs buy-in. |

The pattern: **deliberate and record when a decision is hard to reverse or crosses
module boundaries; just follow the local convention when it is contained and
cheap.**

## Anti-patterns

- **Architecture astronautics** — layers, abstractions, and patterns for problems
  you do not have. Over-engineering is a failure mode, not caution.
- **Big-bang rewrite** — replacing a working system all at once instead of
  strangling it incrementally; the classic project-killer.
- **Half-migration** — leaving two competing structures because a "cleanup" was
  never finished.
- **Premature microservices** — a distributed monolith with all the network pain
  and none of the independence.
- **Resume-driven tech choices** — picking tools for novelty over fit and
  operability.
- **Ivory-tower ADRs** — decisions recorded and then ignored, or written for
  trivia.

## Where this fits

`architecture` is the Layer 2 skill for the "how it is structured" decisions that
the flagship workflows assume. It sets the boundaries and dependency rules that
`engineering-standards` enforces at the code level, supplies the design forces that
`design-patterns` and `solid` resolve locally, and feeds `database-design` and
`api-design` for the data and contract decisions. `integration-build` applies its
anti-corruption boundary; `feature-delivery` and `refactor-safely` operate within
the structure it defines.
