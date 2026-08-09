---
name: commit-messages
description: >-
  Write commit messages that explain a change to future readers — house
  convention is a ticket-prefixed imperative subject (no colon), under 72
  characters, with a body for the why on non-trivial changes, and no agent
  co-author. Use when writing a commit message or structuring commits. Covers
  atomic commits, subject/body craft, and referencing the ticket. Triggers:
  "commit message", "write a commit", "how should I commit this", "commit
  convention".
---

# Commit messages

A commit message is a note to whoever runs `git blame` on this line in two years —
usually you. The diff shows *what* changed; the message must supply *why*. The
house convention is fixed (`engineering-standards`); the craft is in using it well.

## The convention

- **Subject**: `PROJ-123 imperative subject` — ticket prefix, **no colon**,
  imperative mood, under 72 characters. `PROJ-123 add order confirmation webhook`,
  not `PROJ-123: added webhook stuff`.
- **Body** (for non-trivial changes): blank line after the subject, then the *why*
  — the problem, the reason for this approach, trade-offs, anything a reviewer
  would ask. Wrap at ~72 columns.
- **No agent co-author.** Do not add co-author trailers.

## Subject craft

- **Imperative mood** — "add", "fix", "remove", as if completing "this commit
  will…". Matches Git's own generated messages (merge, revert).
- **Specific** — `PROJ-123 guard against negative order totals`, not
  `PROJ-123 fix bug`.
- **Under 72 chars** so it is not truncated in logs and tools.

## Body — when and what

Skip it for trivial, self-evident changes. Write it when the change is non-trivial:

```
PROJ-451 debounce the search endpoint to cut duplicate queries

Rapid keystrokes were firing one query per character, spiking the DB.
Debouncing at 250ms in the controller collapses these into one request
without a perceptible delay. Considered a client-side fix but the API is
consumed by three frontends, so the server is the right place.
```

Explain the *why* and the *why-this-way*, not the *what* (the diff has the what).

## Atomic commits

One logical change per commit. A commit should be independently understandable and,
ideally, leave the code working. Do not bundle an unrelated refactor into a feature
commit — separate them (`refactor-safely`: refactoring and behavior change are
different commits). Clean up `wip`/`fix`/`asdf` commits before the branch ships
(`finish-branch`).

## Anti-patterns

- **`wip` / `fix` / `asdf`** — meaningless in permanent history.
- **Colon after the ticket** — `PROJ-123:` breaks the house convention.
- **Past tense / vague** — "added stuff", "fixed it".
- **What-not-why body** — narrating the diff instead of explaining the reason.
- **Kitchen-sink commit** — many unrelated changes in one.
- **Co-author trailer** — not used here.

## Where this fits

`commit-messages` is the Layer 5 skill enforcing the commit convention from
`engineering-standards`, applied during `finish-branch` and every `feature-delivery`
/ `refactor-safely` / `debug` commit. It pairs with `pr-writer` (the PR is the
change's summary; commits are its history).
