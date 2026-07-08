---
name: ticket-writer
description: >-
  Write well-formed tickets of the right type — story, bug, task/subtask, epic,
  or initiative — each with its proper structure and testable acceptance
  criteria. Use when writing a ticket, drafting a user story, filing a bug
  report, scoping an epic, or turning a rough idea into a backlog item. Produces
  clean Markdown ready to paste into Jira/Linear. Triggers: "write a ticket",
  "user story", "bug report", "scope an epic", "acceptance criteria", "break this
  into subtasks", "file a bug".
---

# Ticket writer

A ticket exists so someone who is not you can pick it up and deliver the right
thing without a conversation. That means: the right *type* for the work, a clear
statement of the outcome, and **testable acceptance criteria** — the part most
tickets get wrong. Describe the problem and the desired outcome; do not prescribe
the implementation unless it is a genuine constraint.

## Pick the type

- **Story** — a user-facing increment of value. Has acceptance criteria.
- **Bug** — something broken; needs reproduction and expected-vs-actual.
- **Task / Subtask** — a unit of technical work, often under a story.
- **Epic** — a large body of work spanning many stories; has a goal and scope.
- **Initiative** — a strategic objective spanning epics.

## Templates

**Story**
```
## <concise outcome-focused title>
As a <role>, I want <capability> so that <benefit>.

### Acceptance criteria
- Given <context>, when <action>, then <observable outcome>.
- ...

### Notes
<constraints, out-of-scope, dependencies, links>
```

**Bug**
```
## <what is broken, briefly>
### Steps to reproduce
1. ...
### Expected
<what should happen>
### Actual
<what happens instead>
### Environment
<version, env, user/role, data conditions>
### Severity
<Critical / High / Medium / Low — impact and frequency>
```

**Epic**
```
## <epic title>
### Goal
<the outcome and why it matters>
### Scope
<in scope / out of scope>
### Success metrics
<how we know it worked>
### Breakdown
<the stories/tasks it decomposes into>
```

## Quality bar

- **Acceptance criteria are testable** — phrased so QA can pass/fail them
  (Given/When/Then). "Works well" is not a criterion.
- **Right-sized** — a story fits in a sprint; if it cannot, it is an epic. Split
  vague giants (INVEST: Independent, Negotiable, Valuable, Estimable, Small,
  Testable).
- **Outcome over solution** — say what and why; leave how to delivery unless the
  approach is a real constraint.
- **Self-contained** — links, environment, and context included so no side-channel
  is needed to start.

## Anti-patterns

- **Vague acceptance criteria** — untestable, so "done" is a matter of opinion.
- **Solution smuggled in** — the ticket dictates implementation instead of outcome.
- **Giant story** — a sprint-plus of work that should be an epic.
- **Bug with no reproduction** — unactionable; see `debug` and `investigator`.
- **Missing context** — reader must ask questions before starting.

## Where this fits

`ticket-writer` is the Layer 5 skill that feeds `feature-delivery` (a well-formed
ticket is its input). Bug tickets connect to `debug` (fixing) and `investigator`
(documenting); acceptance criteria become the `verify` targets at delivery.
