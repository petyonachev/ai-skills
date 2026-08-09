---
name: pr-writer
description: >-
  Write pull request descriptions that let a reviewer understand and verify the
  change fast, and serve as the record later. Use when opening a PR, writing a PR
  description, or preparing a change for review. Produces a Summary (what and
  why, not a diff restatement) and a Test Plan (how it was verified), with the
  linked ticket, matching house conventions. Triggers: "write a PR", "pull
  request description", "PR message", "prepare this for review", "open a PR".
---

# PR writer

A PR description is written for the reviewer, not the author. Its job is to let
someone who did not write the code understand *what changed and why*, and *verify
it works*, in the least time — and to stand as the record when someone reads it in
six months. A good description gets a faster, better review; a diff-restatement
wastes everyone's time.

## Conventions

- **Title** — short, under 72 characters, states the change. Ticket-prefixed to
  match the branch/commit convention (`engineering-standards`).
- **Body** — two required sections, `## Summary` and `## Test Plan`, plus a linked
  ticket. Keep it proportional: a one-line fix gets three lines; a feature gets a
  real write-up.

## Structure

```
## Summary
<What changed and WHY — the problem it solves and the approach taken.
 Not a restatement of the diff; the reviewer can read the diff.>

## Test Plan
<How you verified it: commands run and their result, flows exercised,
 edge cases checked. What a reviewer should run to confirm.>

## Notes (optional)
<Risks, out-of-scope items, follow-ups, screenshots, migration/rollout steps.>

Closes PROJ-123
```

## What makes it good

- **Lead with why.** The reviewer can see *what* from the diff; what they cannot
  see is the intent, the constraint, the alternative you rejected.
- **The Test Plan is evidence, not intention** — mirror `verify`: state what you
  actually ran and observed, so the reviewer can reproduce it.
- **Call out risk and scope** — what could break, what you deliberately left out,
  what follows in a later PR. Surprises in review erode trust.
- **Link the ticket** so the PR connects to its context.
- **Keep the PR itself focused** — one logical change. A tight PR needs a short
  description because it does one thing; if the description sprawls, the PR
  probably should have been split (`finish-branch`).

## Anti-patterns

- **Diff restatement** — "changed X, updated Y, added Z"; the diff already says
  that.
- **No why** — what changed with no reason it changed.
- **No test plan** — or a hand-wave ("tested locally") with nothing reproducible.
- **Kitchen-sink PR** — many unrelated changes, impossible to summarize.
- **Hidden risk** — burying a breaking change or migration in the diff with no
  mention.

## Where this fits

`pr-writer` is the Layer 5 skill that `finish-branch` hands off to when opening a
PR. It draws its Test Plan discipline from `verify`, its title/ticket conventions
from `engineering-standards` and `commit-messages`, and its scope guidance from
`finish-branch`.
