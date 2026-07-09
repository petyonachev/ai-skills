---
name: verify
description: >-
  Evidence-based completion discipline. Use BEFORE claiming any work is "done",
  "fixed", "complete", "ready", "should work", or "working now" — for code,
  config, migrations, refactors, or bugfixes. Requires running the actual thing,
  observing real output, and citing it — never asserting success from reading
  the code. Covers tiered definitions of done, evidence patterns by task type,
  driving behavior end-to-end (not just tests), adversarial self-checking, and
  the fake-completion failure modes. Triggers: "done", "finished", "complete",
  "ready", "should work", "it works now", before every commit and PR.
---

# Verify

The most damaging habit an AI engineer has is declaring victory without proof —
"this should fix it", "the feature is done", "that works now" — based on having
*written* code rather than having *watched* it work. This skill exists to make
that impossible.

**The rule: a claim of "done" requires evidence — a command you ran, output you
observed, and can cite. Reading the code is not evidence. Hope is not evidence.**

If you cannot show what proves the work, the work is not done; it is hoped.

## The tiers of done

Done is not one thing. Know which tier you have actually reached, and state it
honestly:

1. **It builds** — compiles, lints, type-checks clean. The floor, not the goal.
2. **Tests pass** — the relevant suite runs green, and you ran it, not a subset
   you assume covers it.
3. **Behavior is correct** — you drove the actual flow (the endpoint, the command,
   the UI path) and observed the right result. This is the tier that matters, and
   the one most often skipped.
4. **Edges hold** — the original reproduction is gone, error paths behave, and
   likely edge cases were exercised.

Tests passing while the feature is broken is common — wrong wiring, missing
config, a mocked boundary that hides the real failure. Tier 2 is necessary but
never sufficient. **Drive the real behavior.**

## What counts as evidence

| Claim | Evidence that proves it | NOT sufficient |
|---|---|---|
| "The bug is fixed" | Reproduction fails before the change, passes after; a regression test added | "I changed the code that looked wrong." |
| "The feature works" | Suite green **and** you exercised the real flow and saw correct output | "The unit tests pass." |
| "The refactor is safe" | Full suite green, identical behavior, no new failures | "It still compiles." |
| "The migration is safe" | Ran up **and** down on a database copy; schema and data verified | "The migration file looks right." |
| "The config change works" | Reloaded/restarted and observed the effect | "I edited the YAML." |
| "No regressions" | Full suite + lint + type-check green | "My change is small." |
| "It's done" (whole task) | Every plan step's verification passed + a final end-to-end check | "I finished writing it." |
| "Tests pass" | Ran the full relevant suite; saw N passed, 0 failed | "The tests I wrote pass." (never ran the rest) |

The left column is what you are tempted to say; the right column is why saying it
is a lie until you have the middle column.

## Verify by task type

- **New feature** — run the suite; then hit the endpoint / run the command / walk
  the UI path and confirm the actual output; check the obvious edge cases.
- **Bug fix** — reproduce the bug *first* (see it fail), apply the fix, confirm
  the reproduction now passes, and add a regression test so it stays fixed. No
  reproduction means no proof you fixed anything.
- **Refactor** — behavior must be unchanged: the same tests that passed before
  pass after, with no new failures. If coverage is thin, add characterization
  tests *before* refactoring.
- **Migration / schema change** — apply on a copy, verify the resulting schema,
  confirm data integrity, and confirm it rolls back. Never verify a migration by
  reading it.
- **Config / infra change** — apply it and observe the effect in a running
  system, not by inspecting the file.
- **Integration** — exercise against the real sandbox or a faithful fake; verify
  timeouts, errors, and retries, not just the happy path.

## Adversarial self-check

Before you say done, turn on yourself. Ask: *how could this be wrong? what did I
not run? which path did I assume instead of exercise?* Address what surfaces.

For high-risk or hard-to-reverse work (money, auth, data, public APIs), do not
rely on your own pass — dispatch independent verifiers whose job is to **refute**
the claim (`parallel-agents`, adversarial-verify pattern). If a majority cannot
break it, confidence is earned. If they can, it was not done.

## Reporting completion

State what you verified, not just that you did:

```
Verified:
- <command run> → <observed result, e.g. "142 passed, 0 failed">
- <flow exercised> → <observed behavior>
Remaining / not covered:
- <anything you did not verify, stated plainly>
```

If a step was skipped, a test failed, or something is unverified, **say so**.
Honest partial completion beats a false "done" every time.

## Anti-patterns — the faces of fake completion

- **"Should work"** — the phrase itself is a confession that you did not check.
- **Verifying by reading** — inspecting code or a diff instead of running it.
- **Green-unit, broken-feature** — tests pass, but you never drove the real flow.
- **Subset-as-whole** — ran the tests you wrote, claimed the whole suite passes.
- **Silent failures** — failing or skipped tests left unmentioned.
- **Wrong-path verification** — exercising a path other than the one you changed.
- **Skip-and-omit** — skipping a verification step and not disclosing it.

## Where this fits

`verify` is the gate at the end of every `iterate` loop (the loop runs until
verification passes) and the final step of every flagship workflow —
`feature-delivery`, `refactor-safely`, `integration-build`, `debug`, and
`finish-branch` all end here. It draws its adversarial verifiers from
`parallel-agents`. Nothing is reported complete until it clears this gate.
