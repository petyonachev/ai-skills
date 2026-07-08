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

## Structure

```
.claude-plugin/
  plugin.json        # plugin manifest (name, version)
  marketplace.json   # marketplace catalog listing the plugin
skills/<name>/SKILL.md
CLAUDE.md            # optional global operating constitution
```

## Releasing

1. Make changes to skills.
2. Bump `version` in `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`.
3. Commit and push to `main`.
4. Users pick it up with `/plugin marketplace update ai-skills` + `/plugin update`.
