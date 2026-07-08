# Operating Constitution

This is the operating guide for a self-contained skill set. Every skill lives in
`skills/<name>/SKILL.md`. This file is the always-loaded index: it defines *how*
to work and routes each kind of task to its skill. Deploy it as the global
`CLAUDE.md` (e.g. `~/.claude/CLAUDE.md`); it references only the skills in this set.

## Core operating philosophy — the powerhouse loop

For any non-trivial task, default to the orchestration core, in this order:

1. **Plan first** (`planning`) — decompose, surface unknowns, decide the execution
   strategy. No production code for a multi-step task until a plan exists.
2. **Fan out** (`parallel-agents`) — investigate broadly and run independent work in
   parallel; use independent agents to verify risky claims.
3. **Loop** (`iterate`) — anything with a pass/fail signal runs as act → observe →
   diagnose → adjust, until green; never a single hopeful pass.
4. **Verify** (`verify`) — nothing is "done" without evidence: run it, observe real
   behavior, cite the output. Reading the code is not evidence.

Escalate (plan, loop, agents) when work is unfamiliar, cascading, broad, or
irreversible; stay simple when it is contained, mechanical, and reversible. Trivial
one-line changes skip the ceremony.

## Using skills

Skills are curated, battle-tested reference material — treat them as the source of
truth. **When a task matches a skill, use it; do not rely on general knowledge
where a dedicated skill exists.** Skills reference each other by name.

## Skill map

**Orchestration core (applies to almost everything)**
- Plan before acting → `planning`
- Delegate / parallelize / investigate broadly → `parallel-agents`
- Loop until a signal is green (tests, repro, lint) → `iterate`
- Before claiming done / ready / fixed → `verify`

**Flagship workflows (by work type)**
- Turn a vague idea into an agreed spec, before planning or code → `brainstorming`
- Deliver a feature or ticket end-to-end → `feature-delivery`
- Change structure without changing behavior → `refactor-safely`
- Find and fix a bug (root cause) → `debug`
- External APIs / third-party / webhooks / messaging → `integration-build`
- Review the current changes for issues (interactive, level-calibrated) → `code-review`
- Ship a finished branch (verify, PR/merge, cleanup) → `finish-branch`

**Standards & design**
- House conventions, layering, definition of done → `engineering-standards`
- System structure, boundaries, ADRs, tech choices → `architecture`
- SOLID review or violation fix → `solid`
- Apply or choose a design pattern → `design-patterns`
- Code smells and refactoring techniques → `refactoring-catalog`
- Code readability / craft review → `code-quality`
- Test strategy, TDD, PHPUnit → `testing`
- Schema, data modeling, indexes, migrations → `database-design`
- API design (REST/GraphQL), versioning, errors → `api-design`
- Security review, OWASP, auth, secrets → `security`
- Performance, N+1, caching, profiling → `performance`

**Symfony stack (Symfony 7.x / PHP 8.3+)**
- Symfony components and idioms → `symfony`
- Doctrine ORM/DBAL, mapping, queries → `doctrine`
- Symfony version upgrade / deprecations → `symfony-upgrade`
- PHP version upgrade / Rector → `php-upgrade`
- Composer, dependency updates, audit → `composer`

**Communication & documentation**
- PR description → `pr-writer`
- Ticket, story, bug, epic → `ticket-writer`
- Commit message → `commit-messages`
- Bug investigation HTML report → `investigator`

**Meta / setup**
- Author your global CLAUDE.md, wired to these skills → `global-config`

*(Python stack skills are planned as a future layer.)*

## Always-on conventions

Detailed in `engineering-standards`; the essentials:

- **Branch**: `PROJ-123-name-of-the-branch` (ticket prefix, kebab-case).
- **Commit**: `PROJ-123 name of the commit` — ticket prefix, **no colon**,
  imperative, under 72 chars; body for the *why* on non-trivial changes. **No agent
  co-author.**
- **PR**: title under 72 chars; `## Summary` + `## Test Plan`; linked ticket.
- **Dependencies**: never add, remove, or upgrade without explicit approval; never
  hand-edit lock files.
- **Definition of done**: `verify` passed at the behavior tier — tests first and
  green, real behavior exercised, diff self-reviewed. "It compiles" is a floor.

## Safety baseline

- Never commit, read, or log `.env` files, credentials, or secrets.
- Never force-push `main`/`master` or any shared branch.
- Never delete unmerged branches, modify CI/CD, or run destructive commands without
  explicit approval.
- Never access or modify production environments.

## Communication

Direct and minimal — no emojis, terse, just the facts. Avoid trailing summaries;
the reader can read the diff.
