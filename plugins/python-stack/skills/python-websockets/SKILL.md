---
name: python-websockets
description: >-
  WebSocket client and server patterns in async Python — message contracts,
  reconnect/backoff, heartbeat/keepalive, backpressure on send, and graceful
  shutdown. Use when building or troubleshooting a WebSocket connection to or
  from a Python service (e.g. bridging to a companion process like a Discord
  bot over a raw socket). Companion to `async-python` (the underlying
  primitives) and the C#-side `csharp-stack:csharp` WebSocket section for the
  other end of the bridge. Triggers: "websocket", "ws://", "wss://",
  "websockets library", "reconnect", "heartbeat", "ping pong", "socket
  closed".
---

# Python WebSockets

A WebSocket is a long-lived connection, not a request/response call — most of
the discipline here is about what happens *between* messages: staying
connected, noticing when you're not, and not overwhelming either side.

## Client and server basics

- The `websockets` library is the straightforward choice for both ends when
  you don't need a full web framework; use `starlette`/`fastapi`'s WebSocket
  routes if you already have an HTTP server and want the endpoint alongside
  REST routes.
- Treat the connection object as an async context manager
  (`async with websockets.connect(uri) as ws:`) so the socket is always closed
  on the way out, including on an exception.
- **Never block inside the receive loop** — the same discipline as
  `async-python`'s core rule. A blocking call while holding the connection
  open stalls your ability to notice a close, a ping, or backpressure.

## Message contracts

- Define an explicit message shape (a small typed model — `dataclass` or
  `pydantic`) and serialize to JSON, rather than passing raw dicts across the
  boundary. A shape change on one side should fail loudly on the other, not
  silently misinterpret a field.
- **Version the contract** if the two ends can deploy independently (e.g. the
  Python service and a C# bot) — a `type`/`version` field lets the receiver
  branch on shape rather than guess.
- Distinguish text frames (JSON control/event messages) from binary frames
  (raw audio/bytes) deliberately — don't base64-encode binary data into JSON
  unless there's a real reason; it's 33% larger and slower to (de)serialize.

## Reconnect and backoff

- **A dropped connection is not exceptional — plan for it.** Wrap the
  connect-and-run loop in a retry with exponential backoff (+ jitter) rather
  than a bare `while True: connect()`, which hammers a struggling peer.
- Cap the backoff (e.g. 30s max) so recovery is fast once the peer is back,
  but don't retry-storm while it's down.
- On reconnect, decide explicitly what state needs to be resent (e.g.
  "re-announce this session id") — a fresh connection is a fresh handshake,
  not a resume, unless you've built resume semantics on top.

## Heartbeat and keepalive

- Use the protocol-level ping/pong (`websockets` sends pings automatically by
  default; confirm the interval/timeout suit a real-time use case) rather than
  inventing an application-level heartbeat unless you need liveness
  information the transport-level ping doesn't give you.
- **A missed pong means the connection is dead, even if the TCP socket
  hasn't noticed yet** — treat a pong timeout as a disconnect and reconnect,
  don't wait for a lower-level error.

## Backpressure on send

- `websocket.send()` can block if the peer isn't reading fast enough (its TCP
  receive buffer fills). In a real-time pipeline, decide explicitly what
  happens when send is slow: buffer up to a bound, drop the oldest/newest
  frame, or apply the same bounded-queue pattern from `chunked-streaming`
  in front of the socket rather than calling `send()` directly from the
  producer.
- Don't await `send()` from inside a tight producer loop with no bound — a
  slow consumer on the other end becomes unbounded memory growth on this end,
  the same failure mode as an unbounded `asyncio.Queue`.

## Graceful shutdown

- Send a proper close frame (`await ws.close(code=..., reason=...)`) instead
  of just dropping the connection — it tells the peer this was deliberate, not
  a crash, and lets it skip a reconnect-with-backoff cycle.
- On shutdown, drain or explicitly discard in-flight sends rather than letting
  the process exit mid-write.

## Testing

- Fake the WebSocket connection (a stub with `send`/`recv` you control) rather
  than opening a real socket in unit tests; reserve a real connection for a
  narrow integration test.
- Test the reconnect/backoff path explicitly — simulate a dropped connection
  and assert it retries with backoff, not just that the happy path connects.
- See the stack-agnostic `testing` skill and `async-python`'s testing notes
  for the general approach this specializes.

## Pitfalls that recur

- **Bare reconnect loop with no backoff** — retry-storms a struggling peer.
- **Blocking call in the receive loop** — misses pings/closes/backpressure.
- **Unbounded send** — a slow peer becomes unbounded memory growth here.
- **Untyped JSON blobs** — no contract, so a shape change breaks the other
  side silently at runtime.
- **Treating a TCP-level error as the only disconnect signal** — a missed pong
  means dead even if the socket hasn't errored yet.

## Where this fits

`python-websockets` builds on `async-python`'s never-block/timeout discipline
for the connection itself, and often feeds a `chunked-streaming` pipeline on
receipt of binary frames. It's the Python-side counterpart to the WebSocket
bridge section in `csharp-stack:csharp` when the two ends of your bridge are a
Python service and a C# bot.
