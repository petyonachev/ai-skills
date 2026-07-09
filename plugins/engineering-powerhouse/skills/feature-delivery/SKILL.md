---
name: feature-delivery
description: >-
  End-to-end workflow for delivering a feature or ticket: understand the goal,
  investigate the codebase, plan, implement with TDD, self-review, verify, and
  ship. Use when starting work on a ticket, building a feature into an existing
  application, adding an endpoint or use case, or implementing a user story from
  request to pull request. Orchestrates the core skills (planning,
  parallel-agents, iterate, verify) into one disciplined flow. Triggers:
  "implement this ticket", "build this feature", "add an endpoint", "work on
  PROJ-123", "deliver this story", "start on this feature".
---

# Feature delivery

This is the conductor for feature work. It does not replace the orchestration
core — it sequences it. Each phase hands off to a core skill; this skill's job is
to keep the phases in the right order and stop you skipping the ones that feel
optional but are not (understanding, verification).

The spine: **understand → investigate → plan → implement (TDD) → self-review →
verify → ship.** Never jump straight from a ticket to code.

## The workflow

### 1. Understand the goal

Restate what the ticket actually asks in one or two sentences, and name the
acceptance criteria. Separate the explicit request from assumed scope — implement
what was asked, not what you imagine around it. If the goal or a criterion is
ambiguous, ask now; a clarifying question is cheaper than building the wrong
thing. Capture non-goals explicitly so scope does not creep.

### 2. Investigate the codebase

Before designing, know where the feature lands. Identify by name the layers,
files, and patterns it will touch, and find the existing convention to follow so
the change reads like the surrounding code (this codebase is mixed/evolving —
match the local style of the module you are in, do not import a different one).

If the feature spans unfamiliar or multiple areas, fan out read-only
investigation agents (`parallel-agents`): "where is X handled", "what already
does something like this", "how is this layer tested". Plan from their findings,
never from guesses.

### 3. Plan

Hand off to `planning`: decompose into small, independently verifiable steps,
surface unknowns and risks, and decide the execution strategy (what to
parallelize, what loops, how to verify). For anything beyond a small change,
write the plan down and get agreement before implementing.

### 4. Implement with TDD

Run the `iterate` loop, strictly red-green-refactor, one small step at a time:

1. Write a failing test that expresses the next slice of behavior; watch it fail.
2. Write the minimum code to make it pass; watch it pass.
3. Refactor with the test as a safety net; confirm still green.

Follow `engineering-standards` and, for framework specifics, the `symfony` skill.
Keep each step in a working, committable state. Independent slices can be
dispatched to parallel agents with partitioned files; anything sharing a file
stays sequential.

### 5. Self-review before claiming anything

Read your own diff as a reviewer would, against a checklist:

- Does it satisfy every acceptance criterion, and nothing beyond scope?
- Does it match the module's existing conventions and layering?
- Security: inputs validated at the boundary, no injection/authz gaps, no secrets
  committed.
- Any obvious performance traps (N+1 queries, unbounded loops)?
- Dead code, leftover debug, commented-out blocks removed?

Fix what the review surfaces before moving on. For a large or high-risk diff,
dispatch an independent reviewer agent — a fresh perspective catches what you are
blind to.

### 6. Verify

Hand off to `verify`. Tests green is the floor; **drive the real feature** — hit
the endpoint, run the command, walk the flow — and observe correct behavior,
including the obvious edge cases and error paths. Cite the evidence.

### 7. Ship

Once verified, hand off to `finish-branch` (pre-push checks, integration path,
branch hygiene) and `pr-writer` (the PR description). Reference the ticket in the
commit and PR per `engineering-standards`.

## Scaling to feature size

Match the ceremony to the work. The phases are the same; their weight is not.

| Feature | Investigate | Plan | Agents | Notes |
|---|---|---|---|---|
| Tweak a label / copy / config default | Skim | Skip | No | One step; go straight to change + verify. |
| Add a field that cascades (entity→form→API→tests) | Light | Light | No | Deceptively multi-layer — do not treat as trivial. |
| Add a new endpoint / use case | Yes | Yes | Maybe | Full loop; agents if the area is unfamiliar. |
| Cross-cutting feature (touches many modules) | Yes | Yes | Yes | Fan out investigation and independent slices. |
| Feature on money / auth / PII / public API | Yes | Yes | Yes | Plan risks explicitly; adversarial verification before ship. |

The pattern: **more investigation and planning as the feature gets broader,
less-understood, or more irreversible; near-zero process for a contained tweak.**

## What a feature usually touches (Symfony)

A feature rarely lives in one file. Trace it through the layers it affects and
test each meaningful one:

- Entry point (controller / message handler / console command) — thin; delegates.
- Application/service layer — where the business logic lives, not in the
  controller or the entity.
- Domain / entity + repository — persistence and invariants.
- Input handling — DTO / Form + validation at the boundary.
- Output — serialization / response shape / API contract.
- Schema — a migration if the model changed (see `verify` for migration proof).
- Tests — unit for logic, integration for wiring, at the level that gives real
  confidence.

Defer detailed component patterns to the `symfony` skill; this list is the
checklist for *what not to forget*, not how to build each part.

## Anti-patterns

- **Ticket-to-code** — skipping understanding and investigation, coding against an
  imagined codebase.
- **Scope creep** — building beyond the acceptance criteria "while you're in
  there."
- **Fat controller / anemic service** — business logic in the wrong layer because
  it was faster to type.
- **Skipping red** — writing code first, tests after (or never), so nothing proves
  the behavior.
- **Self-certified done** — claiming complete on green units without driving the
  actual feature.
- **Big-bang diff** — one enormous untested change instead of small verified
  steps.

## Where this fits

`feature-delivery` is the primary Layer 1 workflow. It orchestrates the entire
Layer 0 core (`planning`, `parallel-agents`, `iterate`, `verify`), leans on
`engineering-standards` and `symfony` for the *how*, and ends in `finish-branch`
and `pr-writer`. For pure defect work use `debug`; for structural change without
behavior change use `refactor-safely`; for third-party/external work use
`integration-build`.
