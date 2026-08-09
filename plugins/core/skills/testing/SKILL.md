---
name: testing
description: >-
  Language-agnostic testing strategy and patterns for building a suite that gives
  real confidence to change code fast. Use when writing tests, designing a test
  strategy, applying TDD, choosing a test level (unit/integration/functional),
  picking test doubles, fixing flaky tests, or judging coverage. Covers the
  testing pyramid, behavior-over-implementation, arrange-act-assert, the five
  doubles, and isolation/determinism. Runner and framework specifics (PHPUnit,
  pytest, xUnit, ...) live in your stack plugin. Triggers: "write tests", "test
  strategy", "TDD", "mock this", "flaky test", "test coverage", "how do I test X".
---

# Testing

A test suite has one job: give you the confidence to change code quickly without
breaking it. Everything here serves that. A test that never fails proves nothing;
a test coupled to implementation breaks on every refactor and teaches the team to
distrust the suite; a slow suite stops being run. Good tests are fast,
deterministic, and pinned to *behavior*, not to how the behavior is achieved.

**The central rule: test observable behavior through the public interface, not
implementation details.** If a refactor that preserves behavior breaks a test,
that test was testing the wrong thing.

## The pyramid

Most tests should be fast and low-level; few should be broad and slow. Inverting
this (the "ice-cream cone" — mostly slow end-to-end tests) gives a suite that is
slow, flaky, and hard to diagnose.

- **Unit** — a single class/function in isolation, no I/O. Fast, numerous. The
  base of the pyramid.
- **Integration** — a unit wired to a real collaborator it owns: the database, a
  repository, the container. Fewer, slower, higher confidence in wiring.
- **Functional / end-to-end** — a full slice through the stack (HTTP request →
  response). Fewest; reserve for critical paths.

## What to test, and at what level

- Test **behavior and contracts**, not getters, not framework internals, not
  private methods (test those through the public method that uses them).
- Cover the paths that matter: the happy path, the boundaries (empty, one, many,
  max), and the error/exception paths. Do not write tests for cases that cannot
  occur.
- Push logic down to the level where it can be tested fast. If something is hard
  to unit-test, that is usually a design smell (`solid`, `refactoring-catalog`) —
  fix the design rather than reaching for a heavier test.

## Test structure

- **Arrange–Act–Assert** — set up the world, perform the one action, assert the
  outcome. Keep the three visually distinct.
- **One behavior per test.** A test asserts one logical outcome; when it fails you
  should know exactly what broke from its name alone.
- **Descriptive names** — `it_rejects_an_order_below_the_minimum`, not `testOrder`.
  The name states the behavior and the condition.
- Keep tests DRY only where it aids clarity; a little duplication in tests is
  better than a shared helper that hides what is being tested.

## Test doubles — use the right one

- **Dummy** — passed but never used (fills a parameter).
- **Stub** — returns canned answers to calls made during the test.
- **Fake** — a working lightweight implementation (in-memory repository).
- **Spy** — records how it was called, asserted after the fact.
- **Mock** — pre-programmed with expectations, verified as part of the test.

The discipline that matters more than the taxonomy: **mock at the boundaries you
do not own** (third-party APIs, the clock, randomness, the network), and use
**real objects or fakes for collaborators you do own**. Mocking everything tests
your mocks, not your code — over-mocking is the most common way a suite becomes
green and worthless. If a test needs five mocks to stand up, the design is telling
you the unit does too much.

## Isolation and determinism

Every test must pass alone, in any order, every time:

- **No shared mutable state** between tests; no reliance on execution order.
- **No real time, randomness, or network.** Inject a clock and a seed; fake
  outbound calls. Nondeterminism is the root of every flaky test (`debug`).
- **Reset state** between tests — database in a transaction rolled back per test,
  containers rebuilt, in-memory state cleared.

## TDD discipline

Write the failing test first, watch it fail for the right reason, write the
minimum to pass, then refactor. This is the red-green-refactor loop run by
`iterate` — see it for the loop mechanics. The point of red-first is that a test
you never saw fail is a test you cannot trust.

## Coverage

Coverage is a guide, not a goal. High coverage of trivial code (getters,
generated code) while critical logic is untested is worse than an honest lower
number. Aim coverage at the paths where a bug would hurt. Chasing 100% breeds
assertion-free tests that execute code without checking anything.

## Calibration — worked examples

| What to test | Level | Doubles |
|---|---|---|
| Pure domain logic / a value object | Unit | None — use real objects. |
| A service orchestrating collaborators you own | Unit | Real collaborators or fakes; mock only true boundaries. |
| Repository / query correctness | Integration | Real test database, rolled back. |
| An HTTP endpoint, end to end | Functional | Real stack; fake external APIs. |
| A third-party API client | Integration | Faked HTTP / sandbox — see `integration-build`. |
| A console/CLI command | Functional | Real or faked dependencies. |
| Time- or randomness-dependent logic | Unit | Inject a clock / seed; never real time. |

The pattern: **test at the lowest level that gives real confidence, mock only what
you do not own, and use a real database when the query is the thing under test.**

## Anti-patterns

- **Testing implementation** — assertions on internals that break on safe
  refactors.
- **Over-mocking** — a suite of mocks that verifies nothing real.
- **Ice-cream cone** — mostly slow end-to-end tests; slow and flaky.
- **Assertion-free tests** — code executed, nothing checked (coverage theatre).
- **Interdependent tests** — pass only in a certain order or after another test
  ran.
- **Nondeterminism** — real time/random/network making tests flaky.
- **Testing the framework** — verifying that a library or the framework itself
  works, not your code.

## Where this fits

`testing` is the backbone of the TDD loops in `feature-delivery`, the safety net
in `refactor-safely`, the regression tests in `debug`, and the fault-injection
approach in `integration-build`. It runs inside `iterate` (red-green-refactor) and
underpins `verify` (tests are the floor, driven behavior is the proof). It defers
design fixes surfaced by hard-to-test code to `solid` and `refactoring-catalog`.
