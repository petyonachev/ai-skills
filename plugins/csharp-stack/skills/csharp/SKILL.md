---
name: csharp
description: >-
  The idiomatic way to build a NetCord-based Discord bot in C#/.NET — general
  conventions, async/await with Task, NuGet, NetCord gateway/voice client
  idioms, and Unix domain socket IPC patterns for bridging to a companion
  process (e.g. a Python audio pipeline) with a length-prefixed binary
  protocol. Use when implementing or troubleshooting the bot: gateway events,
  voice/audio, slash commands, IPC messages, or NuGet dependency changes.
  Triggers: "C#", ".NET", "NetCord", "Discord bot", "Unix socket", "IPC",
  "gateway", "voice client".
---

# C#

This is the language/framework specialization of the stack-agnostic core
skills, scoped to the shape of project you actually build: a Discord bot on
**NetCord** that bridges Discord voice/events to a companion process (e.g. the
Python audio pipeline) over a **Unix domain socket** with a length-prefixed
binary protocol — not a WebSocket; both ends live on the same host, so a
same-host IPC transport is the right choice (see the Python-side
`python-stack:process-ipc` skill for the same protocol from the other end).
Target modern C#/.NET (8+) with nullable reference types enabled.

## Conventions

- **Nullable reference types on** (`<Nullable>enable</Nullable>`) — a `string?`
  vs `string` distinction catches null-handling bugs at compile time; do not
  suppress warnings with `!` without a reason.
- **`async`/`await` all the way down** — an `async void` method should only
  ever be a top-level event handler; everywhere else, `async Task` (or
  `async Task<T>`), so exceptions propagate and callers can await completion.
- Constructor injection for dependencies (the gateway client, the WebSocket
  bridge, configuration) — no static/service-locator access.
- One project/solution layout: keep the bot's command/event handlers thin,
  push logic that isn't Discord-specific into plain classes that don't
  reference `NetCord` types, so it stays testable independent of the gateway.

## NetCord idioms

- Register gateway event handlers (voice state updates, message/interaction
  events) through NetCord's client event subscriptions, not polling.
- Slash commands: one command = one handler method; keep the handler thin and
  delegate to a service for anything beyond parsing the interaction and
  replying.
- Voice: NetCord's voice client hands you the raw audio stream — treat it the
  same way the Python side treats real-time audio: no blocking calls in the
  audio-handling path, since a stall there is an audible glitch, not just a
  slow response.

## Unix domain socket bridge (to the companion process)

- Use `System.Net.Sockets.Socket` with a `UnixDomainSocketEndPoint`
  (cross-platform since .NET 5) wrapped in a `NetworkStream`, with an
  explicit reconnect/backoff strategy — a dropped connection to the companion
  process should not silently stop forwarding audio/events.
- **Timeouts on send/receive** — a hung write to a dead socket should not
  block the caller indefinitely; use a `CancellationToken` with a deadline on
  every read/write, not just on connect.
- **Match the wire format byte-for-byte with the other end.** A length-prefixed
  binary header (e.g. 1-byte message type + 8-byte id + 2-byte payload
  length, little-endian) is cheaper than JSON/text framing on a hot audio
  path — build it with `System.Buffers.Binary.BinaryPrimitives`
  (`WriteUInt64LittleEndian`, etc.) rather than `BinaryWriter`, so the
  endianness is explicit in code rather than relying on the platform
  default. Keep the format's definition next to a comment pointing at the
  Python-side `struct.pack` format string it must match
  (`python-stack:process-ipc`) — this contract exists nowhere the compiler
  can check it.
- Read a message in two steps: read exactly the header size, parse the
  payload length from it, then read exactly that many more bytes
  (`Stream.ReadExactlyAsync` — do not assume one `Read`/`ReceiveAsync` call
  returns a whole message; Unix-socket streams have no message boundaries of
  their own).
- Use JSON (a small DTO, versioned if the two sides deploy independently) for
  lower-frequency control messages where framing overhead doesn't matter, and
  reserve the binary format for the hot data path — see the stack-agnostic
  `api-design` skill for contract discipline generally.
- Treat the IPC boundary as a true external boundary (`integration-build`):
  validate what comes back (including the length field on a binary message —
  a corrupted length must not drive an oversized read), don't trust it
  blindly.

## Dependency management (NuGet)

- `dotnet add package <name>` / `dotnet remove package <name>` — never
  hand-edit `.csproj` package references or `packages.lock.json` directly.
- Never edit lock files by hand — let the tool manage them
  (`engineering-standards`, dependency discipline).

## Pitfalls that recur

- **`async void` outside event handlers** — swallows exceptions; use
  `async Task` so callers can observe failures.
- **Blocking calls in the voice/audio path** — `.Result`/`.Wait()` on a `Task`,
  or synchronous I/O, stalling real-time audio the same way a blocking call
  stalls the Python side.
- **No reconnect strategy on the IPC bridge** — a transient disconnect
  silently stops the bot forwarding to the companion process.
- **Binary format drift between the two languages** — an endianness or field-
  order mismatch with the Python side is a silent corruption, not a clean
  error.
- **Assuming one `Read`/`ReceiveAsync` returns a whole message** — a stream
  has no message boundaries of its own; only the length-prefix framing
  creates them.

## Where this fits

`csharp` is the Layer 3 stack anchor that the flagship workflows lean on for
the "how" of implementation, the same role `symfony-stack:symfony` plays for
Symfony repos and `python-stack:python` plays for the companion service. It
realizes the layering of `engineering-standards` and the integration-boundary
discipline of `integration-build` for the Unix-socket bridge specifically,
matching wire formats with `python-stack:process-ipc` on the other end.
