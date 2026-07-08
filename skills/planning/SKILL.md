---
name: planning
description: >-
  Plan before acting on any non-trivial task. Use BEFORE writing code, running
  migrations, or making changes whenever the work is 3+ steps, touches multiple
  files, is ambiguous, or is hard to reverse. Decompose the goal, surface
  unknowns, decide the agent and verification strategy, and produce a written
  plan. Triggers: "implement", "build", "add feature", "refactor", "migrate",
  "fix this bug", "how should I approach", or any request where the path is not
  a single obvious edit.
---

# Planning

A plan is the cheapest place to be wrong. Ten minutes of planning routinely
saves hours of misdirected implementation, wrong abstractions, and rework. This
skill defines *when* to plan, *how* to plan, and what a plan must contain before
implementation is allowed to start.

The rule: **do not write production code for a non-trivial task until a plan
exists.** For a genuinely trivial change (single file, obvious fix, no ambiguity)
skip planning and act — over-planning a one-liner is its own failure.

## When to plan

Plan when **any** of these is true:

- The task takes 3 or more distinct steps.
- It touches more than one file, layer, or service.
- The requirements are ambiguous or under-specified.
- The change is hard to reverse (migrations, deletions, public API changes, data
  backfills, anything touching money, auth, or PII).
- You do not yet understand the existing code well enough to name the exact
  files and symbols you will change.

Skip planning only when the change is a single obvious edit with no unknowns.
When in doubt, plan — a short plan is cheap; a wrong implementation is not.

## Calibration — worked examples

The decisions that matter are at the boundary, not the obvious cases. Use these
contrasting examples to calibrate *plan / loop / agents*. The dangerous mistakes
are the **deceptive** ones: tasks that look trivial but cascade, and tasks that
look huge but are mechanical.

| Task | Plan | Loop | Agents | Why |
|---|---|---|---|---|
| Fix a typo in a validation message | No | No | No | Single obvious edit, no unknowns. |
| Change one value in `services.yaml` | No | No | No | Contained, reversible, nothing to verify beyond a reload. |
| Rename a private method + its callers in one class | No | No | No | Mechanical and contained; the tests already cover it. |
| **Add a nullable `phone` field to the `User` entity** | Light | Yes | No | *Looks* trivial but cascades: entity, migration, form, fixtures, serializer, tests. Loop until the migration and suite are green. |
| Add a REST endpoint to create an `Order` | Yes | Yes | Maybe | Multi-layer: controller, validation, service, persistence, tests. Agents only if the surrounding code is unfamiliar. |
| **Refactor the 800-line `OrderService`** | Yes | Yes | Yes | Fan out agents to map callers and existing tests first, add characterization tests, then refactor in small verified steps. |
| "Why is checkout slow in production?" | Yes | Yes | Yes | Location unknown — fan out to investigate broadly; then measure → change → measure as a loop. |
| Integrate a Stripe payment webhook | Yes | Yes | Maybe | Hard to reverse and touches money: plan risks explicitly, loop against a sandbox, verify idempotency. |
| Upgrade Symfony 6.4 → 7.0 | Yes | Yes | Yes | Broad but mechanical: agents audit deprecations across the codebase; loop fixing them until green. |
| Get 12 tests green after a bad merge | Light | Yes | Maybe | The loop *is* the task; agents only if failures span unrelated areas. |

Read the pattern, not the rows: **escalate when the work is unfamiliar,
cascading, broad, or irreversible; stay simple when it is contained, mechanical,
and reversible.**

## The method

Work these phases in order. Do not skip ahead to solutions before the goal and
constraints are pinned down.

### 1. Understand the goal

State, in one or two sentences, what "done" looks like from the user's
perspective. If you cannot state it crisply, you do not understand it yet — ask
a clarifying question. A clarifying question is always cheaper than a wrong
implementation.

Separate the explicit request from assumed requirements. Implement what was
asked; do not invent scope.

### 2. Map the terrain

Before designing anything, know what already exists. Identify the concrete files,
classes, functions, tables, and endpoints the task will touch — by name. Find the
existing patterns you should follow so the change reads like the surrounding code.

If mapping the terrain requires reading across many files or several subsystems,
**this is the point to fan out investigation agents** — see the `parallel-agents`
skill. Send them to answer specific questions ("where is X handled", "what calls
Y", "how is Z currently tested") and plan from their findings, not from
guesses. Never plan against imagined code.

### 3. Surface unknowns and risks

List what you do not know yet and what could go wrong:

- Unknowns that must be resolved before or during implementation (investigate now
  if they change the approach; defer if they only affect details).
- Risks: data loss, downtime, security exposure, performance cliffs, breaking
  changes for consumers.
- Assumptions you are making — state them so they can be challenged.

A plan that pretends there are no unknowns is a fantasy, not a plan.

### 4. Decompose and sequence

Break the work into small, independently verifiable steps. Each step should:

- Have a clear "done" condition (ideally a test that passes).
- Be as small as possible while still being meaningful.
- Be ordered so that each step leaves the code in a working, committable state
  where practical.

Mark which steps are independent (can be parallelized) and which are sequential
(depend on earlier steps).

### 5. Decide the execution strategy

Explicitly choose, and record, how the work will run:

- **Parallelize?** If steps are independent and substantial, dispatch them to
  parallel agents (`parallel-agents`). If they share files or must be sequenced,
  keep them single-threaded.
- **Loop?** Anything with a pass/fail signal (tests, lint, type checks, a
  reproduction) should run as a `iterate` loop — implement, run, read the
  failure, fix, repeat — not as a single hopeful pass.
- **Verify how?** Name the exact commands that will prove each step and the whole
  task are done (`verify`). If you cannot name them, the plan is incomplete.

### 6. Write the plan down

A plan that lives only in your head cannot be reviewed. Produce it as text using
the format below. For substantial work, present it and get agreement before
implementing. When the harness offers plan mode, use it: draft in plan mode and
exit for approval before touching code.

## Plan output format

```
## Goal
<one or two sentences: what "done" looks like>

## Context
<the concrete files/classes/tables involved, and the existing patterns to follow>

## Unknowns & risks
- <unknown or risk> — <how it will be resolved or mitigated>

## Steps
1. <small step> — verified by: <test/command>
2. <small step> — verified by: <test/command>
   ...

## Execution
- Parallelizable: <which steps, or "none">
- Loops: <which steps run as verify loops>
- Final verification: <commands that prove the whole task is done>
```

Keep it proportional: a three-step task gets a five-line plan; a migration gets a
thorough one. The format scales down, not just up.

## Anti-patterns

- **Coding before planning** on a multi-step task — the most expensive habit
  there is.
- **Planning against imagined code** — always map real files first; investigate
  when unsure.
- **Vague steps** ("improve the service") that have no verifiable done condition.
- **Ignoring reversibility** — treating a migration or deletion like an ordinary
  edit.
- **A plan with no verification** — if nothing proves it works, it is not done,
  it is hoped.
- **Over-planning the trivial** — a one-line fix does not need a six-section
  document.

## Where this fits

`planning` is the entry point of the orchestration core. It hands off to
`parallel-agents` (for investigation and independent work), `iterate` (for the
implement-run-fix loop), and `verify` (for evidence-based completion). Every
flagship workflow — `feature-delivery`, `refactor-safely`, `integration-build`,
`debug` — begins here.
