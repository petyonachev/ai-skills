---
name: engineering-standards
description: >-
  The house engineering constitution — the always-on conventions every piece of
  work assumes: core principles, code-change hygiene, the quality bar, PHP and
  Symfony conventions, definition of done, naming/branch/commit/PR conventions,
  dependency discipline, and the security baseline. Consult whenever writing or
  changing code, reviewing a diff, or deciding how something should be built.
  This is the standard the flagship workflows enforce. Triggers: "coding
  standards", "house style", "how we build", "conventions", "definition of
  done", "is this the right way", or any implementation work.
---

# Engineering standards

This is the constitution: the cross-cutting rules that hold for *every* task, so
the workflows do not have to restate them. Deep topics have their own skills
(`code-quality`, `solid`, `design-patterns`, `security`, `testing`) — this skill
is the always-on baseline and points to them for depth.

## Core principles

- **Simplicity first** — make every change as simple as it can be. The best
  solution is usually the one with the fewest moving parts, not the cleverest.
- **Root causes only** — no temporary patches or symptom fixes; find and fix the
  real problem (`debug`).
- **Minimal blast radius** — touch only what the task requires. No drive-by
  refactors, no unrelated cleanups riding along.
- **Prove it works** — nothing is done without evidence (`verify`). Assertion is
  not proof.
- **Elegance check** — for any non-trivial change, pause and ask: is there a more
  elegant way? Then take it if there is.

## Code-change hygiene

- **Read before editing.** Understand the existing code and its patterns before
  changing it. Never edit code you have not read.
- **Minimal diffs.** The smallest change that solves the problem. Resist the urge
  to tidy things that were not in scope.
- **Follow existing patterns.** Match the codebase's style, naming, and structure.
  New code should read like the code around it.
- **One task at a time.** Complete the requested task; do not fix unrelated issues
  you notice along the way (note them instead).
- **Edit over create.** Prefer changing an existing file to adding a new one.
  Search before creating. Never create documentation files unless asked.
- **No premature abstraction.** Do not introduce interfaces, base classes, or
  design patterns until the task actually demands them. Two call sites is not a
  pattern.
- **No compatibility shims.** When replacing code, remove the old version
  entirely. No dead branches kept "just in case."
- **Remove dead code.** No commented-out blocks, unused imports, or orphaned
  functions left behind.

## The quality bar

- Write clean, well-structured code with practical trade-offs — production-ready,
  not gold-plated.
- Handle the edge cases likely to occur in production; do not invent defensive
  code for cases that cannot happen.
- **Fail fast in development, degrade gracefully in production** — surface errors
  loudly while building, recover safely once live.
- Add inline comments **only** where the logic is not self-evident. Do not add
  docblocks, type annotations, or comments to code you did not change.
- Restructure toward industry-standard conventions when the existing structure
  deviates significantly — but suggest the improvement, do not silently impose it.

## PHP and Symfony conventions

- `declare(strict_types=1)` in every PHP file. Follow PSR-12.
- Use modern PHP (8.3+): `readonly` properties, enums, named arguments,
  first-class callable syntax where they improve clarity.
- **Constructor injection** for dependencies — no service location, no static
  access to services.
- **Layering**: controllers/handlers stay thin and delegate; **business logic
  lives in the application/service layer**, not in controllers and not in
  entities. Validate input at the boundary (DTO/Form + Validator).
- The codebase is **mixed/evolving**: match the local conventions of the module
  you are in. Do not import a different architectural style into a module that
  does not use it; if a module's style is genuinely wrong, raise it rather than
  half-migrate it. See `architecture` for boundary decisions and `symfony` for
  component specifics.

## Definition of done

A change is done when — and only when — `verify` passes at the behavior tier:
tests written first and green, the real behavior exercised, edges and error paths
covered, lint/type-check/build clean, and the diff self-reviewed for scope,
secrets, and dead code. "It compiles" and "the units pass" are floors, not the
finish line.

## Naming and delivery conventions

- **Branch**: `PROJ-123-name-of-the-branch` — ticket prefix, kebab-case.
- **Commit subject**: `PROJ-123 name of the commit` — ticket prefix (no colon),
  imperative, under 72 characters; body for the *why* on non-trivial changes.
- **Do not add the agent as a commit co-author.**
- **PR**: title under 72 characters; `## Summary` and `## Test Plan` sections;
  linked ticket. Full mechanics in `finish-branch` and `pr-writer`.

## Dependency discipline

- **Never add, remove, or upgrade a dependency without explicit approval.** When
  proposing one, explain why it is needed and list the alternatives considered.
- Never edit lock files (`composer.lock`, `package-lock.json`) by hand — let the
  tool manage them. See `composer` for the update workflow.

## Security baseline

- Never read, commit, or log `.env` files, credentials, API keys, or secrets.
- Sanitize and validate all user input at system boundaries.
- Build with OWASP Top 10 awareness; flag injection, XSS, CSRF, and authz gaps in
  review. Depth lives in `security`.

## Don't guess

- Do not fabricate file paths, API endpoints, config keys, or names — search or
  ask.
- Do not assume unstated requirements or architecture — verify, then decide.
- A clarifying question is cheaper than a wrong implementation. Ask when stuck.

## Where this fits

`engineering-standards` is the Layer 2 anchor. Every flagship workflow
(`feature-delivery`, `refactor-safely`, `debug`, `integration-build`,
`finish-branch`) enforces it, and it defers depth to the design and quality
skills: `code-quality`, `solid`, `design-patterns`, `refactoring-catalog`,
`testing`, `security`, `database-design`, `api-design`, and the stack skills
`symfony`, `doctrine`, `composer`.
