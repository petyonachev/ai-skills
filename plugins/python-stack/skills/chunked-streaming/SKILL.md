---
name: chunked-streaming
description: >-
  Processing real-time data in chunks — structuring a pipeline as bounded
  stages, handling multiple concurrent per-source streams, backpressure
  strategy, handling partial/incomplete chunks, windowing/buffering, and
  flush/finalization semantics. Use when building or debugging a pipeline
  that moves data (audio frames, byte streams, token streams) through
  multiple processing stages in real time. Builds on `async-python`'s
  primitives (Queue, TaskGroup, cancellation). Triggers: "chunk", "stream",
  "pipeline", "frame", "buffer", "backpressure", "windowing", "real-time
  processing", "audio frames", "per-user state", "concurrent streams".
---

# Chunked streaming

Real-time chunked processing is a pipeline problem: data arrives in pieces,
each piece needs one or more transformations, and the pieces must keep moving
without the pipeline as a whole falling behind or exhausting memory. Everything
here is about keeping that flow bounded and observable.

## Structuring a pipeline

- Model each stage (capture → decode → STT → LLM → TTS → playback, or
  whatever the actual stages are) as a coroutine reading from one
  `asyncio.Queue` and writing to the next, run under a single `TaskGroup` so a
  failure in one stage cancels the rest instead of leaving orphans consuming
  from a queue nobody drains anymore (`async-python`).
- **Bound every queue** (`maxsize=`). An unbounded queue between a fast
  producer and a slow consumer is a silent, growing memory leak, not
  backpressure — it just delays the symptom until the process is killed.
- Give each stage a single, nameable responsibility. A stage that both decodes
  and does business logic is harder to test and harder to reason about under
  backpressure than two stages chained together.

## Multiple concurrent streams

A pipeline rarely handles just one stream — a real-time system usually has
one logical stream per user/speaker/connection, all live at once:

- **Key per-stream state explicitly by source identity**: `dict[source_id,
  StreamState]`, where `StreamState` bundles everything that one stream's
  processing needs (buffers, windowing state, backpressure counters) — never
  a single shared buffer/state object serving multiple sources, which
  silently splices unrelated streams together the moment two are active at
  once.
- Create a source's state lazily on first data, and **tear it down
  explicitly** on disconnect/removal — flush whatever was buffered for that
  source (through the same flush/finalization path a normal end-of-stream
  uses) rather than silently dropping in-flight data when a source leaves.
- A cleanup path invoked at shutdown must handle *every* still-active source,
  not just the one that happens to be disconnecting — iterate all tracked
  sources and flush/finalize each, so a process shutdown doesn't silently
  drop whatever every other concurrently-active source had buffered.
- Keep the per-source state dictionary and any per-source resources (e.g. a
  codec decoder instance — `audio-processing`) owned by the same component
  that creates and removes stream state, so there's one place that knows the
  full set of currently-active sources at any time.

## Backpressure — a decision, not an accident

When a stage falls behind its input, something has to give. Decide explicitly
which:

- **Block the producer** — the queue fills, `put()` awaits until there's room.
  Correct default when every chunk matters and a delay is acceptable (e.g.
  transcript text).
- **Drop chunks** — when staleness is worse than gaps (e.g. video frames when
  falling behind means "skip ahead", not "buffer forever"). Drop the *oldest*
  queued chunk in favor of the newest when recency matters more than
  completeness.
- **Buffer up to a bound, then apply one of the above** — a small buffer
  absorbs jitter (brief slowdowns) without treating every hiccup as loss or
  as a stall.

Whichever you choose, make it visible: log or metric when a queue is
persistently near-full — that's the pipeline telling you where the real
bottleneck is, and it's the `performance` skill's profiling loop applied to a
streaming system.

## Partial and incomplete chunks

- A chunk boundary from the transport (a WebSocket frame, a socket read) is
  not guaranteed to align with a semantic boundary (a complete audio frame, a
  complete JSON message). Buffer bytes until you have a complete unit before
  handing it to the next stage — do not assume one `recv()` equals one frame.
- For length-prefixed or framed binary protocols, read exactly the declared
  length before treating the buffer as a complete chunk; for delimiter-based
  protocols, buffer until the delimiter appears and handle the case where it
  spans multiple reads.
- Guard against a peer that never completes a chunk (a stalled sender) with a
  timeout on assembly, not just on the individual read (`async-python`).

## Windowing and buffering

- **Fixed windows** (e.g. 20ms audio frames) are simplest when downstream
  processing genuinely operates on fixed-size units — pad or drop the final
  partial window explicitly, don't let it silently disappear.
- **Sliding windows** (overlapping) trade extra computation for smoother
  output (e.g. overlapping STT windows to avoid cutting a word at a boundary)
  — only pay for this where the quality difference is actually needed.
- Keep windowing logic isolated in one stage with clear inputs/outputs
  (raw chunks in, windows out) rather than smeared across the pipeline — it's
  the part most worth unit-testing in isolation from the async machinery
  around it.

## Flush and finalization semantics

- **Define what "end of stream" means and signal it explicitly** — a sentinel
  value on the queue, a `None`, or closing the queue — rather than relying on
  a timeout or a dropped connection to imply completion.
- Every stage must know how to flush its buffered-but-incomplete state on
  end-of-stream (e.g. a partial window that will never be completed by more
  data) — decide explicitly whether that gets processed short or discarded,
  don't leave it to fall out of scope silently.
- Propagate the end-of-stream signal downstream through every stage — a
  pipeline where only the first stage knows the stream ended leaves later
  stages waiting forever on a queue that will never receive more data and
  never gets told to stop.

## Testing

- Feed a stage a bounded fake queue with a known chunk sequence (including a
  deliberately partial/misaligned chunk and an end-of-stream signal); assert
  on what comes out the other end, not on internal buffering implementation
  details.
- Test the backpressure path deliberately: a producer faster than the
  consumer should behave the way you decided (block/drop/buffer-then-X), not
  however it happens to behave.
- Test flush/finalization as its own case — many pipelines are only ever
  tested on the happy "steady stream" path and never on "stream just ended
  mid-window."

## Pitfalls that recur

- **Unbounded queues** — the default failure mode; looks fine until it isn't.
- **Assuming transport chunks equal semantic chunks** — a `recv()` is not a
  frame.
- **No explicit end-of-stream signal** — later stages hang forever, or state
  is silently dropped.
- **Backpressure by accident** — whatever the queue happens to do under load,
  never decided on purpose.
- **Untested flush path** — the pipeline works until the stream actually ends.
- **Shared state across concurrent sources** — one buffer/decoder serving
  multiple streams splices unrelated data together.
- **Cleanup that only handles the disconnecting source** — a shutdown path
  that doesn't flush every other still-active source drops their buffered
  data silently.

## Where this fits

`chunked-streaming` composes `async-python`'s primitives (`Queue`,
`TaskGroup`, timeouts) into the pipeline shape this whole stack is built
around. It receives chunks from `python-websockets` or `process-ipc`
depending on the transport, feeds `audio-processing` for audio-specific DSP
and `llm-interaction`/`local-model-inference` for a processing stage, with
`decision-processes` governing what a stage does when a chunk requires a
multi-step decision rather than a straight transform.
