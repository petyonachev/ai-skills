# ai-skills

A self-contained set of [Claude Code](https://claude.com/claude-code) skills for
disciplined software engineering — a stack-agnostic core (orchestration,
flagship workflows, standards & design references) plus per-language stack
tracks for Symfony, Python, and C#. Distributed as a Claude Code
**marketplace** of four **plugins**, installable and updateable via the
`/plugin` system.

## Install

This repository is both a marketplace and the source for its plugins. Add the
marketplace once, then install `core` plus whichever stack plugin matches each
repo:

```
/plugin marketplace add petyonachev/ai-skills

# once, applies everywhere
/plugin install core@ai-skills

# once per repo, in that repo — keeps other stacks' skills out of this repo's sessions
/plugin install symfony-stack@ai-skills --scope local   # in a Symfony repo
/plugin install python-stack@ai-skills --scope local    # in a Python repo
/plugin install csharp-stack@ai-skills --scope local    # in a C# repo
```

`core` at **user scope** makes the orchestration core, flagship workflows, and
standards & design skills available in every session. Each stack plugin at
**local scope** in the matching repo makes its language-specific skills
available only there — a Python repo never loads `symfony`, and vice versa.
See [Configuration scopes](https://code.claude.com/docs/en/settings#configuration-scopes)
for the general mechanism.

Skills self-activate based on the task (e.g. starting a bug fix pulls in
`debug`); some are also invoked directly as commands, such as `/core:code-review`
(namespaced, because Claude Code ships its own built-in `/code-review`).

## Update

```
/plugin marketplace update ai-skills
/plugin update core@ai-skills
/plugin update symfony-stack@ai-skills   # etc., for whichever stacks you installed
```

Updates are versioned via each plugin's own `plugins/<name>/.claude-plugin/plugin.json`
and the shared `.claude-plugin/marketplace.json`. Bump the relevant plugin's
`version` (and the marketplace's) on each release; each plugin versions
independently, so a `core` release doesn't force a stack-plugin bump or vice
versa.

## Optional: the operating constitution

`CLAUDE.md` in this repo is the "operating constitution" — the powerhouse loop
(plan → fan out → loop → verify), the skill-routing map, and the house
conventions. It is entirely stack-agnostic; stack-specific conventions live in
each stack plugin's skills instead. A plugin cannot install global
instructions for you, so to make it always-on, copy its contents into your
global `~/.claude/CLAUDE.md` (or import it). Without it, the skills still
self-activate; you just lose the always-on default behavior and conventions.

## What's inside

Four plugins in `plugins/`:

- **`core`** — stack-agnostic, install once at user scope:
  - *Orchestration core* — `planning`, `parallel-agents`, `iterate`, `verify`
  - *Flagship workflows* — `brainstorming`, `feature-delivery`, `refactor-safely`,
    `debug`, `integration-build`, `code-review`, `finish-branch`
  - *Standards & design* — `engineering-standards`, `architecture`, `solid`,
    `design-patterns`, `refactoring-catalog`, `code-quality`, `testing`,
    `database-design`, `api-design`, `security`, `performance`
  - *Communication* — `pr-writer`, `ticket-writer`, `commit-messages`,
    `investigator`
  - *Meta / setup* — `global-config`
  - Read-only specialist agents in `agents/` (see below)
- **`symfony-stack`** — Symfony 7.x/PHP 8.3+, install per Symfony repo:
  `symfony`, `symfony-patterns`, `doctrine`, `symfony-upgrade`, `php-upgrade`,
  `composer`
- **`python-stack`** — async/real-time/multi-process Python, install per
  Python repo: `python` (anchor: conventions, uv, pytest), `async-python`
  (asyncio fundamentals), `process-ipc` (Unix-socket IPC framing),
  `process-supervision` (multi-process spawn/restart/shutdown),
  `chunked-streaming` (pipeline stages, concurrent streams, backpressure),
  `audio-processing` (PCM/resampling/VAD/codec DSP), `python-websockets`
  (network WebSocket patterns), `local-model-inference` (local
  STT/TTS/LLM model serving), `llm-interaction` (hosted LLM APIs),
  `decision-processes` (state machines, event bus, superseded decisions)
- **`csharp-stack`** — NetCord/.NET, install per C# repo: `csharp` (incl.
  Unix-socket IPC matching `python-stack:process-ipc`)

Several skills carry `references/` files for progressive-disclosure depth
(design-pattern families, Symfony DI/forms, the investigator HTML report
template, a PHP-worked-examples appendix for the core design/quality skills).

## Agents

Read-only specialist agents in `plugins/core/agents/` give the orchestration
core purpose-built workers to fan out to (via the `parallel-agents` skill).
Each is a thin definition wired to the skills — the skill stays the source of
truth, the agent adds an isolated context, an enforced read-only tool boundary,
and a fixed stance:

- **`explorer`** — breadth investigation ("how does X work", "where is Y
  handled"); returns a cited synthesis, not file dumps.
- **`verifier`** — adversarially refutes a specific claim; the independent checker
  behind `verify`, and the adjudicator of contested `code-review` findings.
- **Topic reviewers** — `review-architecture`, `review-reusability`,
  `review-safety`, `review-scalability`, `review-simplicity`,
  `review-security`, `review-tests`. The `code-review` workflow runs two per
  topic that cross-examine each other's findings under the shared
  `code-review-protocol` skill (preloaded into each), so only claims verified
  against the code reach the report.

They are picked automatically when a task matches their description, the same way
skills self-activate.

## Structure

```
.claude-plugin/
  marketplace.json         # marketplace catalog listing all 4 plugins
CLAUDE.md                  # optional global operating constitution (stack-agnostic)
plugins/
  core/
    .claude-plugin/plugin.json
    skills/<name>/SKILL.md
    agents/<name>.md       # read-only specialist agents to fan out to
  symfony-stack/
    .claude-plugin/plugin.json
    skills/<name>/SKILL.md
  python-stack/
    .claude-plugin/plugin.json
    skills/<name>/SKILL.md
  csharp-stack/
    .claude-plugin/plugin.json
    skills/<name>/SKILL.md
```

## Releasing

1. Make changes to skills in the relevant plugin(s).
2. Bump `version` in that plugin's `plugins/<name>/.claude-plugin/plugin.json`
   and its entry in `.claude-plugin/marketplace.json`.
3. Commit and push to `main`.
4. Users pick it up with `/plugin marketplace update ai-skills` +
   `/plugin update <name>@ai-skills` for whichever plugin(s) changed.
