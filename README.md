# ai-skills

A self-contained set of [Claude Code](https://claude.com/claude-code) skills for
disciplined software engineering — an orchestration core, flagship workflows,
standards & design references, and the Symfony stack. Distributed as a Claude Code
**plugin**, installable and updateable via the `/plugin` system.

## Install

This repository is both a plugin and its own marketplace. Add the marketplace,
then install the plugin:

```
/plugin marketplace add petyonachev/ai-skills
/plugin install engineering-powerhouse@ai-skills
```

That makes all skills available. Skills self-activate based on the task (e.g.
starting a bug fix pulls in `debug`); some are also invoked directly as commands,
such as `/code-review`.

## Update

```
/plugin marketplace update ai-skills
/plugin update engineering-powerhouse@ai-skills
```

Updates are versioned via `.claude-plugin/plugin.json` and
`.claude-plugin/marketplace.json`. Bump the `version` in both on each release; the
commands above pull the latest from `main`.

## Optional: the operating constitution

`CLAUDE.md` in this repo is the "operating constitution" — the powerhouse loop
(plan → fan out → loop → verify), the skill-routing map, and the house
conventions. A plugin cannot install global instructions for you, so to make it
always-on, copy its contents into your global `~/.claude/CLAUDE.md` (or import it).
Without it, the skills still self-activate; you just lose the always-on default
behavior and conventions.

## What's inside

32 composable skills in `skills/`, layered:

- **Orchestration core** — `planning`, `parallel-agents`, `iterate`, `verify`
- **Flagship workflows** — `brainstorming`, `feature-delivery`, `refactor-safely`,
  `debug`, `integration-build`, `code-review`, `finish-branch`
- **Standards & design** — `engineering-standards`, `architecture`, `solid`,
  `design-patterns`, `refactoring-catalog`, `code-quality`, `testing`,
  `database-design`, `api-design`, `security`, `performance`
- **Symfony stack** — `symfony`, `doctrine`, `symfony-upgrade`, `php-upgrade`,
  `composer`
- **Communication** — `pr-writer`, `ticket-writer`, `commit-messages`,
  `investigator`
- **Meta / setup** — `global-config`

Several skills carry `references/` files for progressive-disclosure depth
(design-pattern families, Symfony DI/forms, the investigator HTML report template).

## Agents

Three read-only specialist agents in `agents/` give the orchestration core
purpose-built workers to fan out to (via the `parallel-agents` skill). Each is a
thin definition wired to the skills — the skill stays the source of truth, the
agent adds an isolated context, an enforced read-only tool boundary, and a fixed
stance:

- **`explorer`** — breadth investigation ("how does X work", "where is Y
  handled"); returns a cited synthesis, not file dumps.
- **`code-reviewer`** — reviews a diff through one aspect at a chosen level; the
  dispatch target of the `code-review` workflow's aspect fan-out.
- **`verifier`** — adversarially refutes a specific claim; the independent checker
  behind `verify`.

They are picked automatically when a task matches their description, the same way
skills self-activate.

## Structure

```
.claude-plugin/
  plugin.json        # plugin manifest (name, version)
  marketplace.json   # marketplace catalog listing the plugin
skills/<name>/SKILL.md
agents/<name>.md     # read-only specialist agents to fan out to
CLAUDE.md            # optional global operating constitution
```

## Releasing

1. Make changes to skills.
2. Bump `version` in `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`.
3. Commit and push to `main`.
4. Users pick it up with `/plugin marketplace update ai-skills` + `/plugin update`.
