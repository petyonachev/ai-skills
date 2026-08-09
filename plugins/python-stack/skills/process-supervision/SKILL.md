---
name: process-supervision
description: >-
  Running and supervising a multi-process architecture from Python — spawning
  subprocesses, crash-restart with backoff, signal handling, and coordinated
  shutdown across a process tree. Use when a service is split across multiple
  OS processes (not just async tasks in one process), when writing a
  supervisor/orchestrator process, or when a worker process needs to restart
  itself on crash without taking the whole system down. Distinct from
  `async-python`'s task-level concurrency — this is process-level. Triggers:
  "subprocess", "supervisor", "multi-process", "crash restart", "SIGTERM",
  "SIGINT", "process tree", "worker process".
---

# Process supervision

Some concurrency problems don't fit in one process: a GPU-bound model server,
a component in a different language, or isolation so one component's crash or
memory leak can't take the others down with it. That's a different axis from
`async-python`'s task concurrency — here the unit of failure and restart is a
whole OS process.

## When to reach for a separate process

- **Isolation from a crash or memory leak** — a component that loads a large
  model or does unstable third-party work should not be able to bring down
  the process handling real-time I/O for everyone else.
- **A different language or runtime** — a component that's a natural fit for
  another language (or a library that isn't available/mature for asyncio)
  becomes its own process communicating over `process-ipc`.
- **Genuine CPU/GPU parallelism** — Python's GIL means CPU-bound work on
  threads doesn't parallelize; a separate process (or `ProcessPoolExecutor`,
  for tightly-coupled short-lived work) does.
- Don't split into multiple processes for something a `TaskGroup` in one
  process already solves — a process boundary costs IPC, serialization, and
  operational complexity that concurrent tasks don't (`engineering-standards`,
  no premature complexity).

## Spawning and restarting

- `asyncio.create_subprocess_exec(*args)` to launch a child process from
  async code without blocking the loop; `await proc.wait()` for its exit
  code.
- **Crash-restart with exponential backoff, capped, with a give-up
  threshold**: on a non-zero exit, wait `min(2 ** crash_count, max_delay)`
  before restarting, and stop restarting after a max crash count rather than
  looping forever against a process that will never come up. A clean exit
  (code 0) means "done", not "crashed" — don't restart it.
- Run each supervised process's restart loop as its own task under a
  `TaskGroup`, so a bug in the supervision logic for one process doesn't
  silently stop supervising the others.

## Signal handling and shutdown

- Register handlers for `SIGINT`/`SIGTERM` on the event loop
  (`loop.add_signal_handler`) that set an `asyncio.Event` rather than calling
  `sys.exit()` directly from a signal handler — let the main coroutine notice
  the event and shut down deliberately, in order.
- **Shutdown order matters.** Cancel the supervision tasks (so no new restarts
  are attempted), await their cancellation, *then* stop any servers this
  process itself hosts (e.g. an IPC server other processes depend on) —
  stopping shared infrastructure before its dependents have stopped can cause
  a flurry of connection-refused errors during shutdown.
- A subprocess spawned with `create_subprocess_exec` does **not** automatically
  receive the parent's signals — decide explicitly whether children should be
  terminated (send them a signal, or rely on them noticing their stdin/pipe
  closed) as part of the shutdown sequence, don't assume it happens for free.

## Health and liveness

- A process that's alive but stuck (deadlocked, wedged) is worse than one
  that's crashed — a crash gets restarted automatically; a hang does not.
  Where it matters, add a liveness check (a periodic heartbeat over
  `process-ipc`, or a `psutil`-based check) rather than relying on the exit
  code alone to signal trouble.
- Log every lifecycle transition (starting, crashed with exit code, giving up
  after N crashes, clean exit) — this is the audit trail for "why did this
  service stop responding," and it's cheap to add compared to reconstructing
  it after the fact.

## Testing

- Test the restart/backoff logic against a fake process object (something
  with a controllable exit code and a `wait()` you control) rather than a
  real subprocess — you want to assert the backoff timing and give-up
  threshold deterministically, not depend on real process startup time.
- Test the give-up path (max crashes exceeded) as its own case — it's easy to
  only ever exercise "crashes once, restarts, then runs fine."
- Test shutdown ordering with a fake dependency to assert supervision tasks
  are cancelled before shared infrastructure stops.

## Pitfalls that recur

- **Unbounded restart loop** — no backoff, or no give-up threshold, hammering
  a process that will never successfully start.
- **`sys.exit()` in a signal handler** — skips cleanup that the main coroutine
  would otherwise perform.
- **Wrong shutdown order** — stopping shared infrastructure while dependents
  are still running against it.
- **Assuming children die with the parent** — a subprocess left orphaned and
  running after the supervisor exits.
- **No liveness signal, only exit codes** — a hung-but-alive process looks
  the same as a healthy one from the outside.

## Where this fits

`process-supervision` is the process-level counterpart to `async-python`'s
task-level concurrency: it's what starts, restarts, and shuts down the worker
processes that `process-ipc` connects to and that may host a
`local-model-inference` model or a component in another language stack
(`csharp-stack`).
