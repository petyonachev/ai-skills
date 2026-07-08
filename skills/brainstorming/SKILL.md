---
name: brainstorming
description: >-
  Enter an extended, collaborative design conversation that turns a vague idea
  into an agreed, written spec BEFORE any planning or implementation. Invoked as
  a command (/brainstorming). Explores goals, constraints, solutions, and risks
  one focused question at a time, resolves open issues, and grows a living
  project-plan Markdown file bit by bit until the requirements and chosen approach
  are settled. Hard-gates implementation until the spec is approved. Triggers:
  "/brainstorming", "let's brainstorm", "I want to build", "help me figure out",
  "let's discuss a project", "spec this out", "think through this idea".
---

# Brainstorming

This is a *mode*, not a task. It turns a half-formed idea into a spec both sides
agree on — the problem, the requirements, the chosen approach, the risks — written
down before a single line is planned or coded. Most bad implementations trace back
to skipping this: building the wrong thing well, or discovering a fatal constraint
halfway through. Brainstorming pays a little conversation now to avoid a lot of
rework later.

**Hard gate: no planning and no implementation while in this mode.** The output is
an agreed written spec, nothing else. Only after it is approved does work hand off
to `planning` (which decomposes it into steps) and then a delivery workflow.

## Rules of engagement

- **One focused question at a time.** Never fire a questionnaire. Ask the single
  most useful question, listen, and let the answer shape the next one. A back-and-
  forth beats a form.
- **Explore before solutioning.** Understand the problem, users, and constraints
  before proposing how to build anything. Jumping to a solution in the first
  exchange is the classic mistake.
- **Surface and resolve issues as they arise.** When an ambiguity, conflict, or
  unknown appears, name it and work it out (or park it explicitly) before moving
  on. Do not paper over a hole to keep momentum.
- **Challenge assumptions.** Play the constructive skeptic — "what happens if…",
  "why not…", "have we considered…". Your value here is pressure-testing, not
  agreeing.
- **Offer options with a recommendation.** When there are choices (approach, scope,
  trade-off), lay out the real alternatives, give your recommendation and why, and
  let the user decide. Do not silently pick for them.
- **The user drives; you facilitate.** It is their project. Guide the structure,
  supply expertise and options, but the decisions are theirs.

## What to explore

Cover these dimensions over the conversation — in whatever order the discussion
naturally takes, not as a checklist recited at once:

- **Problem & goal** — what are we really solving, and why? What does success look
  like?
- **Users & stakeholders** — who is this for, who is affected, who decides?
- **Non-goals** — what are we explicitly *not* doing? Scope boundaries prevent
  creep later.
- **Constraints** — technical, time, team, budget, existing systems, compliance.
- **Requirements** — functional (what it does) and non-functional (performance,
  scale, security, availability).
- **Solution approaches** — the viable options, with trade-offs; converge on one.
- **Risks & unknowns** — what could go wrong, what we do not yet know, what to
  prototype or research.
- **Scope / MVP** — the smallest version that delivers value; what is phase two.

## The living project document

Capture outcomes in a Markdown file as you go — the spec is built **incrementally**,
not written from memory at the end. After each point is resolved, update the
document; it grows section by section as the conversation converges.

- Default to a single living file, e.g. `docs/brainstorm/<project>.md` (confirm the
  location with the user). For a large effort, split into `requirements.md`,
  `decisions.md`, and `open-questions.md`.
- Keep an **open questions** section live throughout — add to it the moment an
  unknown surfaces, remove entries as they are resolved. The spec is not done while
  it has unresolved blockers.
- Record **decisions with their rationale** (a mini-ADR — see `architecture`) so
  the "why" is not lost.

Living-document skeleton:

```markdown
# <Project / feature> — Brainstorm & Spec
Status: In discussion | Ready for planning
Last updated: <date>

## Problem & goal
<what we're solving and why; what success looks like>

## Users & stakeholders
<who it's for and who decides>

## Non-goals
<explicitly out of scope>

## Constraints
<technical, time, team, compliance, existing systems>

## Requirements
### Functional
- ...
### Non-functional
- <performance, scale, security, availability>

## Proposed approach
<the chosen solution and, briefly, the alternatives rejected and why>

## Decisions
- <decision> — <rationale> (<date>)

## Open questions
- [ ] <unresolved item — owner / how it'll be resolved>

## Risks
- <risk> — <likelihood/impact, mitigation>

## Scope / phasing
- MVP: ...
- Later: ...

## Next steps
<what happens once approved — hands to `planning`>
```

## Exit criteria — when brainstorming is done

Leave the mode only when all of these hold, and confirm with the user:

- The problem, goal, and success criteria are stated and agreed.
- Functional and non-functional requirements are captured.
- A solution approach is chosen, with the alternatives and rationale recorded.
- Scope and non-goals are explicit; an MVP is defined.
- Open questions are resolved, or consciously parked as known risks.
- The user has **approved** the spec.

Then set the document status to *Ready for planning* and hand off to `planning`.
Do not slide from brainstorming straight into code.

## Anti-patterns

- **Jumping to implementation** — writing code (or even a plan) mid-brainstorm; the
  gate exists to prevent exactly this.
- **Question barrage** — dumping ten questions at once instead of a real dialogue.
- **Solutioning first** — proposing how before understanding what and why.
- **Chat-only** — decisions living only in the conversation, lost when it ends;
  persist to the document.
- **Deciding for the user** — silently choosing instead of presenting options.
- **Analysis paralysis** — exploring forever; the exit criteria define "enough".
- **Ignoring open questions** — declaring done with unresolved blockers.

## Where this fits

`brainstorming` is the Layer 1 workflow *upstream* of everything else: it produces
the agreed spec that `planning` decomposes and `feature-delivery` (or another
workflow) implements. It borrows the mini-ADR habit from `architecture` and the
one-question-at-a-time discipline that keeps requirements honest. It is the answer
to "how should I build X?" — start by agreeing what X is.
