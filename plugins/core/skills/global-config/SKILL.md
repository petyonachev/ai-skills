---
name: global-config
description: >-
  Interactively author your global Claude Code instructions file
  (~/.claude/CLAUDE.md), wired to this skill set so Claude actually uses the
  skills. Invoked as a command (/global-config). Discusses how you want Claude to
  behave — communication, autonomy, workflow defaults, conventions, safety — one
  topic at a time, then writes a lean, well-structured CLAUDE.md that encodes your
  preferences and routes tasks to the installed skills. Triggers: "/global-config",
  "write my CLAUDE.md", "set up my global instructions", "configure how Claude
  behaves", "personalize Claude", "global config".
---

# Global config

This authors your global `~/.claude/CLAUDE.md` — the always-on instructions loaded
into every session. Its two jobs: capture **how you want Claude to behave**, and
**wire in this skill set** so the skills are actually used instead of ignored. The
file is a *router and a constitution*, not a manual: it stays lean and points to
skills for depth.

Because a plugin cannot install global instructions for you, this file is how you
turn the skill set from "available" into "the default way Claude works."

## The interaction

Discuss preferences **one topic at a time** — propose a sensible default (drawn
from this skill set) for each, and let the user accept or adjust. Do not dump a
form. The goal is a file that reflects *their* preferences, not a generic template.

Elicit across these dimensions:

- **Operating philosophy** — should Claude default to the powerhouse loop (plan →
  fan out → loop → verify)? How aggressively (always for non-trivial work, only
  when complex, or conservatively)? This is the hook that makes the orchestration
  core (`planning`, `parallel-agents`, `iterate`, `verify`) the default.
- **Communication** — verbose or terse? Emojis or none? Explanations or just the
  diff?
- **Autonomy** — confirm before acting, or proceed on routine work? When must it
  stop and ask?
- **Testing discipline** — TDD by default? Definition of done (ties `verify`)?
- **Conventions** — branch/commit/PR format, dependency rules (default to the ones
  in `engineering-standards`, confirm or override).
- **Safety** — destructive commands, production, secrets, force-push rules.
- **Knowledge sources** — prefer skills first, official docs over blogs, etc.
- **Stack defaults** — primary language/framework, so routing to the stack skills
  (`symfony`, `doctrine`, …) is unambiguous.

## What belongs in the file — and what does not

The single most important authoring rule: **the CLAUDE.md routes; the skills carry
the depth.** It is loaded every session, so every line costs tokens forever — keep
it high-signal.

| Goes in CLAUDE.md | Belongs in a skill (just reference it) |
|---|---|
| The operating philosophy / default behavior | The step-by-step of a workflow |
| The skill-routing map (task → skill) | SOLID rules, patterns, component APIs |
| Communication, autonomy, safety rules | Testing patterns, security checklists |
| Conventions (branch/commit/PR) at a glance | Their full rationale and mechanics |

If you find yourself explaining *how* to do something in the CLAUDE.md, stop — that
is a skill's job. The file says *what to do and which skill to use*.

## Structure of the generated file

Produce these sections, in this order (the repo's own `CLAUDE.md` is a ready
template to adapt):

```markdown
# Operating Constitution

## Core operating philosophy
<the powerhouse loop, at the chosen aggressiveness; hooks planning/parallel-agents/iterate/verify>

## Using skills
<directive: when a task matches a skill, use it; skills are the source of truth>

## Skill map
<task → skill routing, grouped: orchestration core, workflows, standards & design, stack, communication>

## Behavior & workflow
<autonomy, testing discipline, and any personal working preferences>

## Conventions
<branch/commit/PR format; dependency discipline — a glance, detail in engineering-standards>

## Safety baseline
<destructive commands, production, secrets, force-push>

## Communication
<tone, verbosity, emojis>
```

Fill the Skill map from the installed set so routing is concrete (name each skill
by the task that triggers it). Keep each section to the essentials.

## Handling an existing CLAUDE.md

- **Never clobber it silently.** Read the current `~/.claude/CLAUDE.md` first.
- Back it up (e.g. `CLAUDE.md.bak`) before writing.
- Offer to **merge**: preserve the user's existing rules, and either integrate the
  skill wiring into their structure or wrap the managed additions in markers
  (`# >>> skills >>>` … `# <<< skills <<<`) so future updates touch only that block.
- Confirm the final content with the user before writing.

## Anti-patterns

- **Bloated CLAUDE.md** — long enough that it competes with the actual task for
  attention; loaded every session, so bloat is a permanent tax.
- **Duplicating skill content** — pasting SOLID/testing/patterns into the file
  instead of routing to the skill.
- **No skill wiring** — a behavior file that never references the skills, so they
  sit unused.
- **Vague preferences** — "write good code" instead of concrete, followable rules.
- **Clobbering existing config** — overwriting the user's file without backup or
  merge.
- **Contradicting the skills** — conventions in the file that conflict with
  `engineering-standards`; keep them aligned.

## Where this fits

`global-config` is the meta/setup skill that activates the whole set: it writes the
global instructions that make the powerhouse loop and the skill-routing map the
default. It mirrors the repo's `CLAUDE.md` (use it as the template), defers
convention detail to `engineering-standards`, and complements the plugin install —
the plugin delivers the skills, this makes them the way Claude works.
