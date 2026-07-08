---
name: debug
description: >-
  Systematic debugging workflow: reproduce, investigate from evidence,
  hypothesize and test one variable at a time, find the root cause, fix it, and
  prevent recurrence. Use for any bug, error, crash, failing test, flaky
  behavior, regression, or production incident — anything where code does not do
  what it should and you need to find out why. Fixes root causes, never symptoms.
  Triggers: "there's a bug", "this is broken", "why is X happening", "it's
  failing", "crash", "error", "flaky test", "regression", "doesn't work",
  "investigate this incident".
---

# Debug

Debugging is applied science, not guesswork. You observe a symptom, form a
falsifiable hypothesis about the cause, test it, and narrow until you reach the
true root — then fix that. The failure mode this skill prevents is **fixing what
looks wrong instead of what is wrong**: patching the symptom, leaving the cause,
and shipping a bug that comes back wearing a different mask.

**Two rules govern everything: reproduce before you diagnose, and fix the root
cause, not the symptom.**

## The workflow

### 1. Reproduce

A bug you cannot reproduce is a bug you cannot confirm fixed. Get a reliable,
ideally minimal, reproduction first: the exact inputs, state, and steps that
trigger it, reduced to the smallest case that still fails. If it only reproduces
intermittently, invest in making it deterministic — add logging, pin timing,
narrow the conditions — before trying to fix it. The reproduction becomes your
regression test later, so capture it as one.

### 2. Investigate from evidence

Read what the system actually tells you — the full error, the stack trace, the
logs, the failing assertion — before forming any theory. Do not theorize ahead of
the data. Narrow the problem space by bisection: which layer, which commit, which
input boundary. `git bisect` for regressions; binary-search the code path for
logic errors.

If the location is unknown or spans unfamiliar code, fan out a multi-modal sweep
(`parallel-agents`): several agents each searching a different way — by stack
frame, by data flow, by recent change, by log correlation. Each is blind to the
others; together they find what one angle misses.

### 3. Hypothesize and test — one variable at a time

Run the `iterate` loop as a scientific loop:

```
state a falsifiable hypothesis → predict what you'd observe if it's true →
test it (change ONE thing) → confirm or refute → narrow
```

Change one variable per test. Shotgun debugging — altering several things at once
— means that even if the symptom disappears you will not know why, and you will
not know what else you broke. A hypothesis you cannot state as "if X is the cause,
then I should see Y" is not yet sharp enough to test.

### 4. Reach the root cause

The first plausible line is rarely the root. Ask *why* down the causal chain: the
null pointer (proximate) because the repository returned null (proximate) because
the query filtered on the wrong column (root). Stop when fixing the cause would
prevent the whole chain — not at the first place you *could* patch. If you cannot
explain *why* the bug happened, you have not found the root cause and any fix is a
guess.

### 5. Fix and add a regression test

Write a test that fails on the bug (red — this is your reproduction formalized),
apply the smallest fix that addresses the *root* cause, and watch it pass (green).
The regression test is not optional: it is what stops the bug returning and proves
you fixed the real thing. Fix the root cause only — resist expanding into unrelated
cleanup.

### 6. Verify and prevent

Hand off to `verify`: the reproduction now passes, the full suite is green, no new
breakage, and — for anything you changed based on a claim about the cause — confirm
the cause was real (an adversarial verifier that tries to reproduce the bug
*after* the fix). Then prevent recurrence: could this class of bug be eliminated
(a type, a guard, a lint rule)? For a significant or production bug, capture a
short post-mortem — what happened, root cause, fix, prevention.

## Bug category strategies

- **Logic error** — trace the data flow; check boundaries and off-by-ones; add
  assertions to pin where reality diverges from expectation.
- **Race / concurrency** — hunt shared mutable state and ordering assumptions;
  reproduce under stress or forced timing. Never "fix" with a `sleep` — that hides
  the race, it does not remove it.
- **Integration failure** — inspect the actual boundary: real payloads,
  serialization, timeouts, error handling. The bug is usually in the contract, not
  the logic.
- **Performance regression** — measure, never guess. Profile to find the real hot
  path; `git bisect` to find the commit. The slow thing is rarely where you assume.
- **Flaky test** — find the source of nondeterminism (time, ordering, shared
  state, external calls) and remove *it*. Retrying until green buries the bug.
- **Environment-specific** — diff the environments: config, versions, data,
  feature flags. "Works on my machine" is a difference waiting to be found.

## Calibration — worked examples

| Situation | Reproduce effort | Agents | Method |
|---|---|---|---|
| Clear stack trace, reliably reproducible | Low | No | Trace to root, fix, regression test. |
| "Works locally, fails in production" | Medium | Maybe | Diff environments; gather prod evidence first. |
| Intermittent / flaky test | High | Maybe | Find the nondeterminism; loop run-to-reproduce until deterministic. |
| Race condition | High | Maybe | Stress-reproduce; investigate shared state and ordering. |
| Performance regression | Medium | Maybe | Profile + `git bisect`; measure before and after. |
| "Something's broken, cause unknown" | Medium | Yes | Multi-modal sweep to locate before hypothesizing. |

The pattern: **the harder the bug is to reproduce, the more of your effort goes
into reproduction and evidence-gathering before you touch a line of code.**

## Anti-patterns

- **Symptom fixing** — patching where the error surfaces, not where it originates.
- **No reproduction** — "fixing" a bug you never saw fail, so you cannot prove it
  is gone.
- **Shotgun debugging** — changing many things at once; even success teaches you
  nothing.
- **Theorizing ahead of the data** — deciding the cause before reading the error.
- **Sleep-to-fix** — masking a race or timing bug instead of removing it.
- **Retry-to-green** — hiding a flaky test's nondeterminism behind retries.
- **No regression test** — leaving the door open for the exact same bug to return.

## Where this fits

`debug` is the Layer 1 workflow for defect work. It runs on `iterate` (the
hypothesize-test loop) and `verify` (reproduction gone, cause confirmed), and uses
`parallel-agents` for locating unknown causes and for adversarially confirming the
root cause. When a fix turns out to need structural change, hand off to
`refactor-safely`; when it grows into new behavior, to `feature-delivery`.
