---
name: integration-build
description: >-
  Workflow for building or consuming external integrations — third-party APIs,
  webhooks, message queues, payment providers, object storage, email/SMS, any
  service across a network boundary. Treats the boundary as hostile: contract
  first, an anti-corruption layer, resilience (timeouts, retries, idempotency),
  and verification against a sandbox plus fault-injecting fakes. Use when
  integrating a third-party service, building an API client, handling webhooks,
  or writing message handlers. Triggers: "integrate", "third-party API",
  "consume an API", "build a client", "webhook", "message queue", "Messenger",
  "payment", "S3", "send emails", "external service".
---

# Integration build

Code inside your process is predictable; code across a network boundary is not.
The other side can be slow, fail halfway, return something the docs never
mentioned, deliver the same message twice, or change without telling you — and
some of its mistakes (a double charge, a duplicate email to a customer) you cannot
take back. This workflow exists to build integrations that survive that reality.

**The governing principle: treat the boundary as hostile. Design for failure and
duplication first; the happy path is the easy 10%.**

## Safety gate — before touching a real external resource

Before any call that hits a real external system, confirm — out loud — four
things:

- **Environment** — you are pointed at a sandbox/test endpoint, not production.
  Never develop or test destructive operations against a live external system.
- **Auth** — credentials come from environment/secrets management, never
  hardcoded or committed. Never log them.
- **Blast radius** — what real-world effect the call has (money moved, email sent,
  data written) and who it touches.
- **Reversibility** — whether you can undo it if it goes wrong, and what the
  recovery is if you cannot.

If any answer is unknown, stop and find out. This gate applies to databases,
queues, storage, and third-party APIs alike.

## The workflow

### 1. Understand the contract

Do not guess payloads. Read the API spec / message schema, and inspect *real*
requests and responses against the sandbox. Note the parts that bite later: auth
scheme, rate limits, pagination, error shapes, timeout behavior, API version, and
what the provider guarantees about ordering and delivery. The contract is the
integration; get it wrong and everything above it is wrong.

### 2. Design the boundary — an anti-corruption layer

Wrap the external service behind your own interface (a port/adapter). The
adapter maps the provider's model to yours; nothing outside it knows the
provider's field names, error codes, or quirks. This keeps the third party's
shapes out of your domain, makes the provider swappable, and gives you one place
to test and to fake. Never let raw external payloads flow into your business
logic.

### 3. Implement with resilience

Build the failure handling *with* the happy path, not after. Every outbound call
needs, at minimum:

- **Timeouts** — always, on every call. A missing timeout is an outage waiting to
  happen.
- **Retries with backoff + jitter** — but only for *idempotent* or safe
  operations. Never blindly retry something that mutates external state.
- **Idempotency** — use idempotency keys for writes; assume any message or webhook
  can arrive more than once and make handlers safe to run twice (dedupe on a
  stable id).
- **Circuit breaking / degradation** — stop hammering a failing dependency; define
  what your system does when it is down (fail closed, queue, serve stale).
- **Explicit error mapping** — turn transport and provider errors into your own
  meaningful failures; never swallow them.

### 4. Verify against sandbox *and* fakes

Two things must both be true, and each needs a different tool:

- **The contract is real** — exercise the actual sandbox so you know the happy path
  matches reality. Mocks lie about contracts; a mock cannot tell you the provider
  renamed a field.
- **The failure paths work** — timeouts, 500s, malformed responses, and duplicate
  deliveries are hard to trigger on demand against a real service, so drive them
  with a fault-injecting fake of your adapter. Test the 90% that is failure
  handling here.

Hand off to `verify`: for anything touching money or irreversible external state,
add adversarial verification — confirm idempotency actually holds under duplicate
delivery, and that a mid-operation failure leaves a recoverable state.

### 5. Make it observable

You will debug this in production, because integrations fail in production. Log
requests and responses at the boundary with correlation ids — **redacting secrets
and PII** — so that when the provider misbehaves you can prove it and trace it.

## Messaging specifics

For asynchronous work (message queues, event buses):

- Delivery is **at-least-once** — handlers must be idempotent; the same message
  will eventually be processed twice.
- Ordering is **not guaranteed** — do not depend on message order unless the
  transport promises it.
- Configure **retry and a dead-letter/failure transport** — failed messages must
  land somewhere you can inspect and replay, not vanish.
- **Version your messages** — producers and consumers deploy independently; a
  consumer must tolerate an old and a new shape during a rollout.

## Calibration — worked examples

| Task | Idempotency | Sandbox | Fakes | Care level |
|---|---|---|---|---|
| Read-only GET from a stable internal API | n/a | Optional | For failure paths | Low — timeout + error mapping. |
| Consume a third-party REST API (reads) | n/a | Yes | Yes | Adapter + timeout + retry + fault fakes. |
| Create/mutate a resource on an external API | Required | Yes | Yes | Idempotency key; retry only if safe. |
| **Payment / money movement** | Required | **Only** | Yes | Highest — no blind retry, adversarial idempotency check, reconciliation. |
| Handle inbound webhooks | Required | Yes | Yes | Verify signature, dedupe on id, ack fast, process async. |
| Async message handler | Required | — | Yes | At-least-once → idempotent; dead-letter; versioned. |

The pattern: **the moment an operation mutates external state — especially money
or anything irreversible — idempotency and failure verification stop being
optional.**

## Anti-patterns

- **Happy-path-only** — no timeout, no retry, no idempotency; works in the demo,
  pages you at 3am.
- **Leaky boundary** — provider payloads and error codes spread through the
  domain; swapping or faking the provider becomes impossible.
- **Blind retry on writes** — retrying a non-idempotent mutation and double-charging
  the customer.
- **Mock-only verification** — trusting mocks that cannot catch a changed contract.
- **Testing against production** — developing destructive calls on a live system.
- **Secrets in code or logs** — committed keys, or credentials/PII printed to logs.
- **Assuming exactly-once delivery** — non-idempotent handlers that break on the
  inevitable redelivery.

## Where this fits

`integration-build` is the Layer 1 workflow for external work. It runs on
`iterate` (exercise-against-sandbox loop) and `verify` (contract real, failures
handled, idempotency proven), uses `parallel-agents` for the adversarial
idempotency and failure checks, and defers component specifics (HTTP client,
message-queue config, serializer) to your stack plugin and design decisions to
`architecture` and `api-design`. Its anti-corruption boundary is the
same discipline `architecture` prescribes and `engineering-standards` enforces.
