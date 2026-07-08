---
name: iterate
description: >-
  The loop-until-green discipline. Use whenever the work has a pass/fail signal —
  a test suite, a reproduction, lint/type errors, a performance target, a
  deprecation count. Drive the change as a closed loop: act, observe the real
  output, diagnose from it, adjust, repeat until the signal is green — never a
  single hopeful pass. Covers red-green-refactor, reading output instead of
  guessing, and the critical judgment of when to break the loop and re-plan
  instead of thrashing. Triggers: "make the tests pass", "fix the failures",
  "get it green", "keep trying until", "loop until", red-green-refactor, TDD.
---

# Iterate

The gap between what you intended and what the code actually does is closed by
one thing: feeding real output back into the next attempt. A loop is how you do
that. The alternative — one hopeful pass, then declaring victory — is how bugs
ship.

**The rule: anything with a pass/fail signal runs as a loop — act, observe the
real output, diagnose from what you observed, adjust, repeat — until `verify`
passes. Not a single attempt. Not a guess.**

## The loop

```
1. Act        — make the smallest change aimed at the signal.
2. Observe    — run it. Read the actual output. Do not skip this.
3. Diagnose   — from the observed output, not from imagination: why did it fail?
4. Adjust     — change what the diagnosis points to.
5. Repeat     — until the signal is green, or the stop condition triggers.
```

Every iteration is driven by **observed evidence**. If you find yourself changing
code without having read the latest error, you have left the loop and started
guessing — the most common way loops go bad.

## Red-green-refactor — the canonical loop

For any behavior change, this is the default loop:

1. **Red** — write a failing test that expresses the desired behavior; run it,
   watch it fail (a test that has never failed proves nothing).
2. **Green** — write the minimum code to make it pass; run it, watch it pass.
3. **Refactor** — with the test as a safety net, clean up the code; run again,
   confirm still green.

Never skip red (you must see the test fail for the right reason). Never skip
refactor (green-but-ugly is technical debt created on purpose).

## When to keep looping vs. stop and re-plan

The dangerous failure mode is not stopping too early — it is **thrashing**: looping
forever, changing things at random, digging deeper into a wrong approach. A loop
needs a stop condition as much as a goal. Break out and step back when:

- **The same error survives ~3 attempts** — your diagnosis is wrong. Stop editing;
  re-read the code, or investigate the cause properly (`parallel-agents`).
- **Oscillation** — fixing X breaks Y, fixing Y breaks X. The approach is flawed;
  a bigger rethink is needed, not another patch.
- **You are changing code you do not understand** — stop guessing and investigate
  first. Random edits that happen to go green hide the real problem.
- **Diminishing returns** — each iteration gets marginally less wrong with no path
  to fully green. The design may be the problem.

When any of these fire, return to `planning`. Re-planning after three failed
attempts is discipline; a fourth blind attempt is thrashing.

## Loops that spawn agents

Loops and fan-out compose. A single loop iteration can dispatch a whole batch of
agents (`parallel-agents`):

- **Loop-until-dry** — for unknown-size discovery (find all bugs, all call sites),
  keep dispatching finder agents until K consecutive rounds surface nothing new.
  A fixed count misses the tail.
- **Loop-until-count** — accumulate toward a target (e.g. keep generating cases
  until N distinct ones exist).

## Calibration — worked examples

| Task / signal | Loop? | Loop until |
|---|---|---|
| A failing test suite | Yes | 0 failures. |
| Implementing to a new test (TDD) | Yes | The test is green, then refactor. |
| Fixing deprecations after an upgrade | Yes | Deprecation count is 0. |
| Reproducing then fixing a bug | Yes | The reproduction passes and a regression test is green. |
| Lint / type-check errors | Yes | Clean output. |
| Hitting a performance target | Yes | Measure → change → measure meets the target. |
| A flaky test | Yes | N consecutive green runs (stability, not one pass). |
| Finding all bugs (unknown size) | Yes | K consecutive rounds find nothing new (loop-until-dry). |
| A one-line config edit | No | Single pass + one `verify`. |
| Writing a design doc or ADR | No | No pass/fail signal — iterate on feedback, not in a loop. |

The pattern: **loop when there is an objective pass/fail signal to drive toward;
do not loop when there is nothing to measure against.**

## Anti-patterns

- **Guess-and-check** — editing without reading the latest output. If you did not
  observe the failure, you are not diagnosing, you are gambling.
- **Weakening the signal to force green** — deleting an assertion, loosening a
  test, or catching-and-ignoring an error to make the loop "pass." This fakes
  completion; the bug is still there.
- **No stop condition** — looping indefinitely on a wrong approach instead of
  re-planning.
- **Whack-a-mole** — treating oscillation as bad luck instead of a signal the
  approach is wrong.
- **One hopeful pass** — skipping the loop on work that clearly has a signal, then
  claiming done.
- **Looping the unmeasurable** — running a "loop" on something with no pass/fail
  criterion.

## Where this fits

`iterate` is the engine that runs between `planning` and `verify`: the plan names
the signal, the loop drives toward it, and `verify` is the gate that ends it. It
composes with `parallel-agents` (fan out each round) and is the beating heart of
every flagship workflow — `feature-delivery` (TDD loop), `refactor-safely`
(change-and-reverify loop), `debug` (hypothesize-and-test loop), and
`integration-build` (exercise-against-sandbox loop). This completes the
orchestration core: **plan it, loop it, verify it, and fan out when it pays.**
