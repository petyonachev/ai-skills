---
name: parallel-agents
description: >-
  Fan out work to subagents for parallelism, breadth, context isolation, and
  independent verification. Use when a task decomposes into independent
  subtasks, when investigation spans many files or subsystems, when a large
  mechanical change repeats across many sites, or when a risky claim needs
  adversarial checking. Covers when to delegate vs. stay single-threaded, how to
  brief a subagent so it returns something useful, write-conflict isolation, and
  synthesizing results. Triggers: "delegate", "parallelize", "fan out",
  "spawn agents", "investigate across the codebase", "verify this", "split this
  work", or any plan step marked independent.
---

# Parallel agents

Subagents are the force multiplier of the orchestration core. Used well, they
turn a serial investigation into a parallel sweep, keep the main context clean,
and provide independent perspectives that catch errors a single reasoning thread
misses. Used badly, they produce sprawl, context bleed, and confidently wrong
results.

A subagent has three properties you must plan around:

- **It does not share your context.** It sees only the prompt you give it. If you
  do not tell it something, it does not know it.
- **Its final message is the entire result.** You get back the text it returns —
  not its intermediate steps, not its file reads. Brief it to return exactly what
  you need.
- **It runs concurrently.** Launch independent agents in a single batch so they
  run at once, not one after another.

## When to fan out

Delegate when you gain **parallelism** (independent work runs at once),
**breadth** (search many places without reading them all into your context),
**isolation** (a big investigation stays out of your working memory), or
**independence** (a separate perspective verifies a claim). Stay single-threaded
when the work is one quick lookup, is inherently sequential, or would have agents
fighting over the same files.

### Calibration — worked examples

| Task | Fan out? | How | Why |
|---|---|---|---|
| Find where a known symbol is defined | No | grep / read inline | One lookup; delegating is slower. |
| Read one file to check a config value | No | read inline | Trivial, single source. |
| "How does auth work across this codebase?" | Yes | Explore agents, read-only | Broad and multi-file; you want the conclusion, not the file dumps. |
| Implement 3 independent CRUD endpoints | Yes | one agent per endpoint, worktree | Independent and substantial — parallelize. |
| A change where step 2 needs step 1's output | No | sequential inline | Dependency; parallelizing corrupts the result. |
| Two refactors touching the same service file | No | serialize | Write conflict — never parallel-edit one file. |
| Verify a risky bug-fix or security claim | Yes | 2–3 adversarial verifiers | Independent perspectives catch what one thread misses. |
| Migrate 40 call sites of a deprecated API | Yes | map / pipeline, one per site | Mechanical at scale; isolate each site. |
| "Why are these tests flaky?" (unknown cause) | Yes | multi-modal sweep | Location unknown; search several angles at once. |
| Fix a typo | No | inline | Trivial. |

The pattern: **fan out for breadth, scale, independence, and genuine parallelism;
stay inline for quick, sequential, or shared-file work.**

## Decomposition patterns

- **Fan-out / fan-in** — split into independent subtasks, run in parallel, then
  synthesize. The default for parallelizable work.
- **Map** — apply the same operation across many items (call sites, files,
  entities). One agent per item, or batched.
- **Scatter-gather / multi-modal sweep** — several agents each investigate the
  same question a *different* way (by caller, by data flow, by test, by log).
  Each is blind to the others; together they cover ground one search misses.
- **Adversarial verify** — spawn independent agents whose job is to *refute* a
  claim, not confirm it. Kill the claim if a majority refute. This is how you
  stop plausible-but-wrong findings from surviving. See `verify`.

## Briefing a subagent

The brief is the single biggest determinant of result quality. A vague brief
returns vague work. Every brief must contain:

1. **Objective** — the one specific question to answer or task to complete, stated
   sharply. Not "look into the order code" but "find every place that mutates
   `Order::status` and list the file, method, and the status it sets."
2. **Context** — the facts the agent needs but cannot see: relevant paths,
   conventions, prior decisions, what has already been tried. Assume it knows
   nothing about this conversation.
3. **Return format** — exactly what to hand back and how (a list, a table, a
   file:line map, a yes/no with evidence). Its final message is your only output;
   shape it.
4. **Boundaries** — what it must not touch or change, and how far to go. For
   read-only work, say "do not modify anything." For scoped edits, name the
   files it owns.

Brief template:

```
Objective: <the single specific goal>
Context: <paths, conventions, what's been tried, what it must know>
Return: <the exact shape of the answer you need back>
Boundaries: <what not to touch; how deep to go>
```

Pick the right agent for the job. This plugin ships three read-only specialists —
prefer them when the task matches:

- **`explorer`** — "find / understand / map" questions across many files. Returns
  a cited synthesis, not file dumps.
- **`code-reviewer`** — one review aspect over a diff (stability, security,
  cleanliness, tests, performance, architecture) at a given level.
- **`verifier`** — adversarially refute a specific claim; the independent checker
  behind `verify` and the adversarial-verify pattern above.

Otherwise use a general-purpose agent for multi-step tasks. For investigation and
review, prefer these read-only agents so they cannot cause side effects.

## Write safety and isolation

Never let two agents edit the same file concurrently — the writes collide. When
parallel agents must all modify files:

- Partition by file/module so no two agents touch the same file, **or**
- Give each agent an isolated git worktree so their changes cannot conflict, then
  integrate the results yourself.

Read-only investigation agents need no isolation — run as many as you like.

## Synthesizing results

Do not just concatenate what the agents return. Synthesis means:

- **Dedupe** overlapping findings.
- **Resolve conflicts** — if two agents disagree, that is a signal to investigate,
  not to pick one at random.
- **Filter** — an agent may return `null` or die on error; drop those and note
  the gap rather than pretending it was covered.
- **Verify before trusting** — an agent's confident answer can still be wrong.
  For anything that matters, confirm the claim (spot-check the file, or send an
  adversarial verifier) before acting on it.

## Anti-patterns

- **Agent sprawl** — spawning agents for trivial lookups a single grep would
  answer. Delegation has overhead; use it when it pays.
- **Context bleed** — assuming the agent knows what you know. It does not; brief
  it fully.
- **Parallel writes to one file** — guaranteed corruption. Partition or isolate.
- **Blind trust** — accepting agent output as fact without verification.
- **Concatenation as synthesis** — dumping raw results instead of reconciling
  them.
- **Delegating the undelegatable** — sending sequential, dependent steps to run
  in parallel.
- **Silent truncation** — capping coverage ("checked the first 10") without
  saying so, which reads as "checked everything."

## Where this fits

`parallel-agents` is called by `planning` (step 5, execution strategy), powers
the investigation and fan-out stages of every flagship workflow, and supplies the
independent verifiers that `verify` relies on for adversarial checking. Loops
(`iterate`) and fan-out compose: a loop can dispatch a fresh batch of agents each
round until the work converges.
