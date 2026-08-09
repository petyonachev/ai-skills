---
name: local-model-inference
description: >-
  Running a local ML model (STT, TTS, or a self-hosted LLM) from async code
  — loading once at startup, serializing GPU access through a dedicated
  executor, keeping the event loop free during inference, and the
  memory/warm-up lifecycle. Use when integrating a local model (e.g.
  faster-whisper, a local TTS model, a self-hosted LLM) into an async
  service, or diagnosing a stall/OOM around model inference. Distinct from
  `llm-interaction`, which assumes a hosted API — this is for a model running
  in-process or in a worker process you control. Triggers: "local model",
  "GPU inference", "faster-whisper", "load the model", "CUDA", "VRAM",
  "ThreadPoolExecutor", "self-hosted".
---

# Local model inference

A local model — running on your own GPU/CPU, not called over an API — has a
different shape of concern than a hosted LLM call: there's no rate limit, but
there is a real, finite piece of hardware underneath it that inference blocks
on for real work, and a model that takes seconds to load in the first place.

## Load once, reuse

- **Load the model once, at service startup, not per-request.** Model loading
  is slow (seconds to tens of seconds) and the whole point of a dedicated
  process/service is to pay that cost once (`process-supervision` — this is
  often why the model lives in its own supervised worker process).
- Keep the loaded model as an instance attribute of a class the service holds
  for its lifetime, not a local variable re-created per call.
- Log when loading starts and when the model is ready — a service that's
  "up" (process running) but not yet "ready" (model still loading) needs to
  be distinguishable, especially if `process-supervision` or a health check
  is watching it.

## Keep inference off the event loop

- Inference is blocking, synchronous, CPU/GPU-bound work — running it inline
  in an `async def` blocks the event loop for the full duration, the same
  failure mode `async-python`'s core rule warns about, just with a much
  longer blocking window (whisper/TTS inference can take hundreds of
  milliseconds to seconds).
- Run it via `loop.run_in_executor(executor, blocking_fn, *args)` and `await`
  the result — this hands the blocking call to a thread while the event loop
  keeps servicing everything else.

## Serializing GPU access

- **A single GPU context generally cannot run multiple inferences
  concurrently without contention** (memory pressure, context-switching
  overhead, or outright errors depending on the framework). Use a
  `ThreadPoolExecutor(max_workers=1)` dedicated to that model so all
  inference calls are serialized onto one thread, one at a time — this is
  not a performance compromise, it's what correct GPU usage requires.
- If you have multiple models or multiple GPUs, give each its own dedicated
  single-worker executor rather than sharing one pool — sharing risks one
  model's inference blocking another's for no reason, and makes it unclear
  which executor is backing which GPU.
- A thread pool (not a process pool) is correct here specifically because the
  work is GPU-bound, not CPU-bound on the Python side — the GIL isn't the
  constraint; a stuck/slow driver call is. (Contrast `async-python`'s general
  advice to use a process pool for CPU-bound work — that's for
  Python-interpreter-bound work, not GPU calls that release the GIL while
  waiting on the device.)

## Memory and batching

- **Know the model's VRAM footprint and plan for it explicitly** — running
  multiple models on one GPU (e.g. STT and TTS side by side) means their
  memory has to fit together, not just each fit alone.
- Pick a compute precision (`float16`/`int8`/etc.) deliberately as a
  memory/quality/speed trade-off, not by default — the same model can differ
  by multiples in memory footprint depending on precision.
- Batch inference calls only when there's a real throughput benefit and
  the added latency (waiting to fill a batch) is acceptable for the use
  case — a real-time voice pipeline usually cannot afford to wait for a
  batch to fill, and single-request inference is the right default there.

## Inference call discipline

- Turn model output into the exact form the rest of the pipeline needs at the
  call boundary (e.g. join transcript segments into one string, not leaking
  the model's internal segment/token representation upward) — keep the
  model's native API contained to this one integration point.
- Pass deterministic inference parameters where reproducibility matters for
  debugging (fixed temperature/seed) rather than leaving sampling
  nondeterminism in a path you'll need to debug later.
- Suppress the model's own redundant preprocessing when a prior pipeline
  stage already did it (e.g. skip a model's built-in voice-activity
  filtering if `chunked-streaming`/VAD already segmented the input) — running
  it twice wastes time and can produce conflicting decisions about where a
  segment starts and ends.

## Testing

- **Never load the real model in unit tests** — multi-second startup cost per
  test run, and needs the actual hardware/weights available. Fake the model
  object at the boundary (a stub `.transcribe()`/`.generate()` returning a
  fixed result).
- If you need a real-inference test, make it an explicit, separately-run
  integration test, not part of the default suite (`testing`, pyramid).
- Test the executor-serialization behavior itself (that concurrent calls
  don't overlap) with a fake blocking function and a short sleep, rather than
  only testing against the real model.

## Pitfalls that recur

- **Inference inline in an `async def`** — blocks the loop for the full
  inference duration.
- **A new executor (or model reload) per request** — pays the load/setup cost
  every time instead of once.
- **Concurrent inference on one GPU context with no serialization** —
  contention, corrupted results, or crashes depending on the framework.
- **Process pool for GPU-bound work** — adds IPC/serialization overhead for
  no benefit; a thread pool is correct here.
- **Testing against the real model** — slow, hardware-dependent, and
  non-deterministic test suite.

## Where this fits

`local-model-inference` is typically hosted in its own `process-supervision`-
managed worker process, reached over `process-ipc` from the rest of the
system, and is the local-model counterpart to `llm-interaction` (which covers
the hosted-API case — streaming, rate limits, retries — that doesn't apply
here). Output from a local model commonly feeds `decision-processes` logic
downstream.
