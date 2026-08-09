---
name: async-python
description: >-
  asyncio fundamentals and patterns for real-time, latency-sensitive Python
  services — the event loop, coroutines and tasks, structured concurrency with
  TaskGroup, cancellation, timeouts, and never blocking the loop. Use when
  writing or debugging any `async def` code, deciding how to run blocking or
  CPU-bound work, structuring concurrent tasks, or diagnosing a stall/stutter
  in an async pipeline. Companion to `python` (conventions/uv/testing) and
  `chunked-streaming` (which builds pipelines on top of these primitives).
  Triggers: "asyncio", "async def", "await", "event loop", "TaskGroup",
  "coroutine", "task", "cancellation", "asyncio.timeout", "run_in_executor",
  "blocking call", "it stutters", "it stalls".
---

# Async Python

asyncio gives you concurrency on a single thread via cooperative scheduling:
exactly one coroutine runs at a time, and it only yields control at an
`await`. Every pattern here follows from that one fact.

## The one rule that matters most

**Never block the event loop.** A single synchronous, blocking call anywhere in
an async call graph stalls *every* other coroutine scheduled on that loop —
in a real-time pipeline this shows up as an audible stutter or a backed-up
queue, not just a slow response for the one caller.

- No blocking I/O (`requests`, sync file reads, sync DB drivers, `time.sleep`)
  inside an `async def`. Use the async equivalent (`httpx.AsyncClient`,
  `aiofiles`, an async driver) or `await asyncio.to_thread(...)` to push
  unavoidable blocking work off the loop.
- CPU-bound work (resampling audio, heavy numeric processing, cryptography)
  belongs in a process pool (`loop.run_in_executor(ProcessPoolExecutor(), fn)`),
  not inline in a coroutine — it starves the loop the same way blocking I/O
  does, because Python's GIL means CPU-bound work on a thread pool still
  blocks other coroutines from running.
- `asyncio.to_thread` (thread pool) is for **blocking I/O**; a process pool is
  for **CPU-bound** work. Using the wrong one either doesn't help (thread pool
  + CPU-bound, GIL-bound) or adds needless IPC overhead (process pool + I/O).

## Structured concurrency

- Prefer `asyncio.TaskGroup` (3.11+) over bare `asyncio.create_task()`. A
  `TaskGroup` gives every task an owner: if one fails, its siblings are
  cancelled and the exception propagates to the caller — nothing is left
  running unsupervised.
- **Fire-and-forget is a bug magnet.** A bare `asyncio.create_task()` with no
  reference held has no owner; if it raises, the exception is logged (if
  you're lucky) and otherwise vanishes. If you truly need fire-and-forget,
  hold the reference and attach a done-callback that surfaces failures.
- `asyncio.gather()` still has a place for a fixed, known set of independent
  awaitables where you want all results — but `TaskGroup` is the better
  default when tasks can spawn tasks or when you want cancellation to
  propagate on first failure.

## Cancellation

- **Cancellation must propagate cleanly.** A task that catches
  `asyncio.CancelledError` and swallows it breaks the cancellation contract —
  the caller thinks the task stopped, but it's still running. Catch it only to
  clean up (close a connection, release a lock), then re-raise.
- Cancellation can land at *any* `await` point — write cleanup with `finally`
  or an async context manager (`async with`), not a linear sequence of steps
  that assumes it always reaches the end.
- A `TaskGroup`'s cancellation of siblings on failure relies on each sibling
  actually honoring cancellation — a coroutine stuck in a blocking call (see
  above) cannot be cancelled until it next hits an `await`.

## Timeouts

- **Every external call gets a timeout.** `async with asyncio.timeout(5):`
  around anything network-bound (an API call, a socket read) — a hung call
  otherwise stalls that task, and anything awaiting its result, indefinitely.
- Prefer `asyncio.timeout()` (3.11+) over `asyncio.wait_for()` — it composes
  better with `TaskGroup` and nested timeouts, and doesn't wrap the awaited
  coroutine in an extra task.
- A timeout firing is cancellation under a different name — the same cleanup
  discipline applies.

## Synchronization primitives

Needed less often than in threaded code (only one coroutine runs at a time),
but still real when multiple coroutines share mutable state across `await`
points:

- **`asyncio.Lock`** — guard a critical section that spans an `await` (e.g.
  read-modify-write on shared state with an awaited step in between).
- **`asyncio.Semaphore`** — cap concurrency (e.g. at most N in-flight LLM
  calls at once, even if 50 requests arrive together).
- **`asyncio.Event`** — one coroutine signals, others wait; good for "wait
  until warmed up" / "wait until shutdown requested" patterns.
- Do **not** reach for `threading.Lock` in async code — it blocks the thread
  (and therefore the loop) if contended, defeating the purpose.

## Testing (pytest-asyncio)

- Mark async tests `@pytest.mark.asyncio` (or `asyncio_mode = "auto"` in
  config). See the `python` skill for the general test setup.
- Test cancellation and timeout paths deliberately — a task that leaks a
  connection or lock on cancellation is a production incident waiting to
  happen, and the happy path alone won't catch it.
- Fake time with `asyncio`-aware fixtures rather than real `sleep()`s when
  testing timeout/retry logic — real sleeps make the suite slow and flaky.

## Pitfalls that recur

- **Sync call in an async function** — the single most common way a real-time
  pipeline silently stalls.
- **Fire-and-forget tasks** — no reference held, no error handling; exceptions
  disappear.
- **Swallowing `CancelledError`** — breaks the cancellation contract; the
  caller believes the task stopped.
- **No timeout on external calls** — one hung request stalls the whole call
  graph waiting on it.
- **CPU-bound work on a thread pool** — doesn't help; still GIL-bound. Needs a
  process pool.
- **`threading.Lock` in async code** — blocks the loop instead of yielding.

## Where this fits

`async-python` is the language-level foundation the rest of the Python stack
builds on: `chunked-streaming` composes these primitives (`Queue`, `TaskGroup`,
timeouts) into pipeline stages, `python-websockets` and `llm-interaction` both
depend on the never-block and always-timeout discipline here, and
`decision-processes` orchestrates on top of it. See the stack-agnostic
`performance` skill for the profiling loop when something feels slow, not just
stalled.
