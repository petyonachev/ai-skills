---
name: decision-processes
description: >-
  Structuring multi-step decision/orchestration logic — explicit state
  representation, choosing a state machine vs a graph orchestrator, per-step
  retry and fallback, and making each decision observable. Use when a
  pipeline stage must decide what happens next based on prior steps (e.g.
  "did the user ask a question or give a command", "did the tool call
  succeed, and what now"), when branching logic is getting hard to follow, or
  when debugging why a multi-step flow did what it did. Builds on
  `async-python` and often governs `llm-interaction` tool-calling flows.
  Also covers decoupling decision logic with an in-process event bus and
  discarding a stale in-flight decision when new input supersedes it.
  Triggers: "decision process", "orchestration", "state machine", "workflow",
  "multi-step", "what happens next", "branching logic", "agent loop", "event
  bus", "pub/sub", "stale", "supersede", "race condition".
---

# Decision processes

A multi-step decision process is any flow where the next action depends on
the outcome of a prior one — an intent classification that picks a handler, a
tool-call result that determines whether to call another tool or respond, a
conversation that moves through stages. The failure mode to avoid is the same
one control-flow craft always avoids (`code-quality`): logic so implicit it
can only be understood by tracing execution, not by reading it.

## Make the state explicit

- **Represent "where we are" as data, not as which function happens to be on
  the call stack.** An explicit state (an enum, a small dataclass) that you
  can log, inspect, and test in isolation beats state implied by nested
  `if`/`elif` chains or which callback fired last.
- Name states for what they mean, not their mechanics: `AwaitingToolResult`,
  not `Step3`. The name should make the decision log readable without
  cross-referencing code.
- Keep the state minimal — exactly what's needed to decide the next action.
  A state object that accumulates every piece of data ever seen becomes its
  own god-object problem (`solid`, SRP).

## State machine vs graph orchestrator

- **A simple state machine** (a `dict[State, Callable]` dispatch, or a
  `match` on the current state) is the right default: one current state, one
  set of valid transitions, easy to test each transition in isolation. Use it
  when the flow is a bounded sequence with a handful of states.
- **A graph orchestrator** (nodes with edges, possibly cyclic, possibly
  running steps concurrently) earns its complexity only when you genuinely
  have branching *and* merging paths, need to resume from arbitrary points,
  or want introspectable execution graphs for a debugging/replay tool. Don't
  reach for one because "agent frameworks use graphs" — reach for one because
  a plain state machine actually can't express what you need
  (`engineering-standards`, no premature abstraction).
- Either way, the transition logic — given this state and this input, what's
  the next state — should be a pure function you can unit test without
  standing up the whole pipeline.

## Decoupling with an event bus

A lightweight in-process pub/sub bus is a common way to wire multiple
independent decision-making components to the same stream of inputs without
them depending on each other directly:

- A minimal event bus is a `dict[event_type, list[handler]]` with a
  `subscribe(event_type, handler)` and an `async emit(event)` that awaits
  each registered handler for that type in order — this is often all you
  need; reach for a message-queue library only once you need durability,
  cross-process delivery, or multiple consumers competing for one event
  (at which point you likely want `process-ipc` between separate processes
  instead).
- **Isolate handler failures.** One subscriber raising should not stop the
  others from receiving the event or crash the emitter — wrap each handler
  call in the bus's `emit()`, log the exception with the handler and event
  type, and continue to the next subscriber.
- Keep events as plain, immutable data (a `dataclass`) describing *what
  happened*, not commands describing what to do — the bus decouples emitters
  from subscribers precisely because the emitter doesn't know or care who's
  listening or what they'll decide to do about it.
- An event bus is a structural tool for decoupling, not a replacement for the
  explicit-state discipline above — a subscriber reacting to an event still
  needs its own clear state and transition logic; scattering decision logic
  across many uncoordinated subscribers just relocates the "hidden control
  flow" problem rather than solving it.

## Per-step retry and fallback

- **Decide the fallback per step, not globally.** "Retry the LLM call once,
  then fall back to a canned response" is a decision about *that* step; a
  blanket "retry everything three times" hides which failures are actually
  recoverable and produces surprising latency when a non-idempotent step
  retries.
- Distinguish failures that warrant a retry (transient: rate limit, timeout)
  from ones that warrant a different next state (a tool call that succeeded
  but returned "not found" is not a failure — it's an outcome the decision
  logic should branch on explicitly).
- Cap total steps/time for the whole process, not just per-step timeouts
  (`async-python`) — a decision process that can retry-and-branch
  indefinitely is a liveness bug, not a feature.

## Discarding superseded decisions

A decision process often takes time (an LLM call, waiting for a shared
resource) during which new input can arrive that makes the in-flight decision
irrelevant — a new message arrives while still generating a reply to the
previous one, for example:

- **Cancel in-flight work when new input supersedes it**, rather than letting
  two decisions race to completion — cancel the running task
  (`async-python`) as soon as the newer input is known, rather than waiting
  for it to finish and then discarding the result (which wastes the work and
  can still produce a visible side effect before you get the chance to
  discard it).
- When the in-flight work can't be cancelled cleanly before it produces a
  result (e.g. it already acquired an external resource you must release
  properly), **timestamp the input that triggered each pending decision** and
  compare that timestamp against the latest known input when the decision is
  about to be acted on — if newer input has since arrived, discard the
  now-stale result instead of acting on it.
- This is a different failure mode than a retry: a stale result isn't
  *wrong* in isolation, it's just answering a question that's no longer the
  current one. Treat "supersede and discard" as its own explicit branch in
  the transition logic, not as an error case.

## Observability

- **Log every transition**: the prior state, the input/outcome that triggered
  it, and the new state. When a multi-step flow does something surprising in
  production, this is the only way to reconstruct *why* without reproducing
  it live.
- Include enough context in each log entry to be useful alone — "transitioned
  to Fallback" without the triggering error is not debuggable at 2am.
- If the process drives user-visible behavior (what the bot says or does
  next), the transition log should be able to explain any single response
  after the fact.

## Avoiding hidden control flow

- A decision "process" that's actually a five-level-deep conditional inside
  one function is the smell this skill exists to prevent — extract it into
  named states and an explicit transition function even if you don't adopt a
  formal state-machine library.
- Side effects (calling a tool, sending a message) belong in the transition's
  *action*, separate from the *decision* of which state to go to next —
  mixing them makes the transition function impossible to test without
  mocking every side effect.
- Avoid deciding the same thing in two places (e.g. both a top-level router
  and a nested handler independently deciding "is this a question or a
  command") — one decision, one place, one state.

## Testing

- Test each transition as a pure function: given state X and input Y, assert
  the next state (and intended action) is Z — no pipeline, no LLM call, no
  I/O needed.
- Test the fallback/retry paths as their own cases, not just the "everything
  succeeds" path through the whole process.
- Test that the process actually terminates — a state machine with a cycle
  that never reaches a terminal state under some input is a real bug class
  here, worth a dedicated test.

## Pitfalls that recur

- **Implicit state** — "where we are" is inferable only by tracing which
  function called which.
- **Graph orchestrator for a five-state flow** — complexity with no payoff.
- **Global retry policy** — hides which failures are actually transient.
- **Untraceable transitions** — no log, so a production surprise can't be
  reconstructed.
- **Decision and action tangled together** — untestable without mocking
  everything.
- **No bound on total steps** — a process that can loop indefinitely.
- **Letting two decisions race** — acting on a stale result instead of
  cancelling or discarding it once newer input has arrived.
- **Decision logic scattered across event-bus subscribers with no shared
  state model** — decoupled delivery, but the "hidden control flow" problem
  relocated rather than solved.

## Where this fits

`decision-processes` governs the "what happens next" logic that sits on top
of `chunked-streaming` pipeline stages and `llm-interaction` tool-calling
results, built on `async-python`'s cancellation/timeout discipline so a
step's failure doesn't leave the process hanging. It's the stack-specific
elaboration of the same discipline `code-quality` (control flow) and
`architecture` (structure) prescribe generally.
