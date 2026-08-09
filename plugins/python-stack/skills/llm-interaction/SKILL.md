---
name: llm-interaction
description: >-
  Calling LLM APIs from async Python — streaming completions, retry/backoff
  on rate limits, per-call timeouts, structured output and tool/function
  calling, prompt and context management, and the sync-client-in-async-code
  trap. Use when integrating an LLM call into a service, debugging a stalled
  or dropped LLM request, designing tool-calling logic, or managing
  conversation/context state. Builds on `async-python`; an application of
  `integration-build` to this specific external dependency. Triggers: "LLM",
  "completion", "streaming response", "tool calling", "function calling",
  "prompt", "context window", "rate limit", "token budget".
---

# LLM interaction

An LLM API is an external, unreliable, rate-limited, latency-variable
dependency — treat it with the same discipline as any other third-party call
(`integration-build`), with a few things specific to LLM APIs layered on top.

## Streaming completions

- **Stream when the caller can act on partial output** (e.g. feeding TTS
  incrementally, showing tokens as they arrive) — it cuts perceived latency
  even though total time is roughly the same. Don't stream if the next stage
  needs the complete response anyway (e.g. parsing structured output); buffer
  and use the non-streaming call instead.
- Iterate the stream with `async for` — confirm the SDK's streaming call is
  actually async-native, not a sync generator wrapped to look iterable (see
  the sync-client trap below).
- Handle a stream that ends early (truncation, a dropped connection) as a
  distinct case from a stream that completes normally — a partial response
  fed downstream as if complete is a silent correctness bug, not a crash.

## Retries and rate limits

- **Retry on 429 (rate limit) and 5xx with exponential backoff + jitter** —
  respect a `Retry-After` header when the API sends one instead of guessing.
- **Do not retry blindly on every failure.** A 400 (bad request — your prompt
  or params are wrong) will fail identically every time; retrying it just
  delays surfacing a bug you need to fix.
- Cap total retry time/attempts and surface a clear failure once exhausted —
  an unbounded retry loop against a persistently down API is its own
  incident.
- Use `asyncio.Semaphore` to cap concurrent in-flight LLM calls
  (`async-python`) — sending 50 requests at once against a rate limit
  guarantees 429s; a bounded concurrency limit is often cheaper than tuning
  retry logic to absorb them.

## Timeouts

- **Every LLM call gets a timeout**, streaming or not (`async-python`). A
  hung request otherwise stalls the calling stage, and everything upstream
  backed up behind it, indefinitely.
- For streaming, also bound the **inter-token** gap, not just total call
  time — a stream that goes silent mid-response without erroring is a
  distinct failure mode from a slow-but-steady one, and a total-time-only
  timeout won't catch it if it eventually resumes.

## Structured output and tool calling

- Prefer the API's native structured-output/JSON-schema mode over asking for
  JSON in the prompt and hoping — it's validated server-side and far more
  reliable than parsing free text.
- **Validate tool-call arguments before executing them** — the model can emit
  a malformed or semantically-invalid call; treat its output as untrusted
  input at the same boundary discipline as user input (`security`).
- Keep the set of tools/functions the model can call to what the current step
  actually needs, rather than exposing everything always — a smaller, precise
  tool surface produces more reliable calls and reduces blast radius if the
  model picks the wrong one (`decision-processes` for the multi-step logic
  around which tools are available when).

## Prompt and context management

- **Track token budget explicitly** rather than discovering the context limit
  via a runtime error — know the model's limit, the fixed overhead
  (system prompt, tool definitions), and the variable part (conversation
  history) as separate numbers.
- Trim/summarize conversation history deliberately (oldest-first truncation,
  summarization, or a sliding window) rather than letting a growing history
  eventually blow the context window — decide the strategy before it's a
  production incident.
- Keep prompts as versioned, reviewable artifacts (not string-interpolated
  inline all over the codebase) — a prompt change is a behavior change and
  deserves the same review discipline as code (`engineering-standards`).

## The sync-client-in-async-code trap

Many LLM SDKs ship both a sync and an async client (e.g. `OpenAI` vs
`AsyncOpenAI`). Using the sync client inside an `async def` blocks the event
loop for the full duration of the call — in a real-time pipeline this is the
single most common way an LLM integration silently stalls everything else
(`async-python`'s core rule, applied here specifically because it's an easy
mistake with these SDKs). Always import and use the async client variant in
async code; there is no safe way to call the sync client without
`asyncio.to_thread`.

## Testing

- **Never call the real API in tests** — slow, flaky, costs money, and
  non-deterministic output makes assertions unreliable. Fake the client at
  the boundary.
- Fixture a few representative responses (including a truncated stream, a
  tool call, and a rate-limit error) rather than only the single happy-path
  response — the retry/timeout/validation logic is exactly the part worth
  testing and the part a single happy fixture never exercises.
- Test tool-call argument validation with a deliberately malformed call, not
  just a well-formed one.
- See `async-python`'s testing notes and the stack-agnostic `testing` skill
  for the general approach this specializes.

## Pitfalls that recur

- **Sync client in async code** — the loop stalls for the full call duration.
- **No timeout, or total-time-only timeout on a stream** — a silently-stalled
  stream isn't caught.
- **Blind retry on 4xx** — retrying a bug instead of fixing it.
- **Unbounded concurrent calls** — guaranteed rate-limit errors under load.
- **Trusting tool-call arguments** — executing model output without
  validating it first.
- **Testing against the real API** — flaky, slow, and non-deterministic
  suites.

## Where this fits

`llm-interaction` is `integration-build` applied to this one specific,
high-value external dependency, built on `async-python`'s never-block/timeout
discipline. It's typically one stage in a `chunked-streaming` pipeline, and
its tool-calling output frequently drives `decision-processes` logic for
what happens next.
