---
name: python
description: >-
  The idiomatic way to build Python services here — general conventions, uv
  for dependency management, and pytest for testing. This is the anchor skill
  for the Python stack; it routes to companion skills for async programming
  (`async-python`), same-host IPC (`process-ipc`), multi-process supervision
  (`process-supervision`), WebSocket clients/servers (`python-websockets`),
  chunked real-time data processing (`chunked-streaming`), audio DSP
  (`audio-processing`), local ML model inference (`local-model-inference`),
  hosted LLM API interaction (`llm-interaction`), and multi-step
  decision/orchestration logic (`decision-processes`). Use for general Python
  conventions, uv commands, or test setup; use the companion skills for their
  specific concern. Triggers: "Python", "uv", "pytest", "pyproject.toml",
  "Python conventions".
---

# Python

This is the language/framework specialization of the stack-agnostic core
skills, and the anchor for the rest of the Python stack. Target modern Python
(3.12+) with full type hints. The project shape this stack is built for:
fully-async, latency-sensitive services that move data through a pipeline in
real time (e.g. audio frames through STT → LLM → TTS) — see the companion
skills below for the depth on each concern.

## Companion skills

- **`async-python`** — asyncio fundamentals: the event loop, tasks, structured
  concurrency, cancellation, timeouts, never blocking the loop.
- **`process-ipc`** — Unix domain socket IPC between local processes: framing,
  connect-with-retry, cross-language wire contracts.
- **`process-supervision`** — running a multi-process architecture: spawning,
  crash-restart, signal handling, coordinated shutdown.
- **`chunked-streaming`** — processing real-time data in chunks: pipeline
  stages, concurrent per-source streams, backpressure, windowing, flush
  semantics.
- **`audio-processing`** — real-time audio DSP: PCM formats, resampling, VAD,
  codec decode/encode.
- **`python-websockets`** — WebSocket client/server patterns (network, not
  same-host): message contracts, reconnect/backoff, keepalive.
- **`local-model-inference`** — running a local ML model (STT/TTS/self-hosted
  LLM): GPU-access serialization, load-once lifecycle.
- **`llm-interaction`** — calling *hosted* LLM APIs from async Python:
  streaming completions, retries, structured output, prompt management.
- **`decision-processes`** — multi-step orchestration logic: explicit state,
  event-bus decoupling, superseded-decision handling, per-step fallback,
  observability.

Reach for the anchor skill (`python`) for conventions/uv/testing; reach for a
companion skill once you're in its specific territory.

## Conventions

- Full type hints; run a type checker (`mypy`/`pyright`) in CI.
- `async def` functions do only async work — if a function has no `await`
  in it, it should not be `async def` (a common accidental-sync-in-async smell;
  see `async-python`).
- Dependency injection via plain constructor parameters — no framework DI
  container needed for this project shape; pass collaborators in explicitly.

## Dependency management (`uv`)

- `uv add <package>` / `uv remove <package>` — never hand-edit `pyproject.toml`
  dependencies or `uv.lock`.
- `uv run <command>` to run inside the project's environment without manually
  activating a venv.
- `uv lock` after manual `pyproject.toml` edits (e.g. version constraints) to
  resync the lockfile.
- Never edit `uv.lock` by hand — let the tool manage it
  (`engineering-standards`, dependency discipline).

## Testing (pytest + pytest-asyncio)

- Mark async tests `@pytest.mark.asyncio` (or set `asyncio_mode = "auto"` in
  config so plain `async def test_...` works without the decorator).
- **Fake the clock and the network** — never let a test depend on real time or
  a real external API call; use fixtures/fakes for the STT/LLM/TTS clients
  (`llm-interaction` has the LLM-specific testing notes).
- See the stack-agnostic `testing` skill for the strategy (pyramid, doubles,
  isolation) this specializes.

## Where this fits

`python` is the Layer 3 stack anchor that the flagship workflows lean on for
the "how" of implementation, the same role `symfony-stack:symfony` plays for
Symfony repos. It realizes the layering of `engineering-standards`, and hands
off the async/streaming/LLM/orchestration depth to its companion skills.
