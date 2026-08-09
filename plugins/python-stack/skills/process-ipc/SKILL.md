---
name: process-ipc
description: >-
  Inter-process communication over Unix domain sockets between local
  processes (a Python service talking to a sibling Python process, or to a
  process written in another language) — connect-with-retry, message
  framing (newline-delimited JSON for control messages, length-prefixed
  binary for a hot data path), graceful teardown, and keeping a binary wire
  format in sync across two languages. Use when building or debugging a Unix
  socket client/server, designing an IPC protocol between local processes, or
  diagnosing a stuck/corrupted IPC connection. Distinct from
  `python-websockets` (different transport — same-host IPC, not a network
  protocol). Triggers: "Unix socket", "IPC", "inter-process", "local socket",
  "start_unix_server", "open_unix_connection", "struct.pack", "wire format".
---

# Process IPC (Unix domain sockets)

When two processes live on the same host, a Unix domain socket is usually the
right choice over a network protocol (WebSocket, HTTP) — lower overhead, no
port management, and the OS enforces that only local processes can connect.
This is the transport for talking to a sibling worker process, a supervised
service, or a companion process written in a different language.

## Connecting and serving

- **Server**: `asyncio.start_unix_server(handler, path=socket_path)`. Remove a
  stale socket file at the same path before binding (`os.unlink` if it
  exists) — a crashed prior instance leaves the file behind and binding fails
  otherwise.
- **Client**: `asyncio.open_unix_connection(socket_path)`, wrapped in a
  **connect-with-retry** loop — the server process may not have created its
  socket yet (startup ordering between supervised processes is not
  guaranteed). Bound the retries and raise a clear error on exhaustion rather
  than retrying forever.
- Clean up on disconnect: cancel the read task, close the writer, `await
  writer.wait_closed()` — an unclosed writer or an uncancelled read task
  outlives the logical connection and leaks.

## Choosing a message framing

Two framings cover most needs; do not reach for a third without a specific
reason:

- **Newline-delimited JSON** — for control/protocol messages (acquire/release,
  status, small structured commands). Simple, human-debuggable
  (`nc`/`socat` can talk to it), and `reader.readline()` does the framing for
  free. Fine for anything not on a tight latency/throughput budget.
- **Length-prefixed binary** — for a hot data path (audio frames, any binary
  payload sent at high frequency). A fixed-size header (`struct.pack`/`unpack`
  with an explicit format string, e.g. `"<BQH"` for a 1-byte type + 8-byte id +
  2-byte length, little-endian) followed by exactly that many payload bytes.
  Avoids JSON's per-message parsing and encoding overhead on the path that
  actually needs it.
- Read a length-prefixed message in two steps: read exactly `HEADER_SIZE`
  bytes (`await reader.readexactly(HEADER_SIZE)`), unpack the length, then
  read exactly that many more bytes. Never assume a single `read()` returns
  a whole message — TCP/Unix-socket streams have no message boundaries of
  their own; the framing is what creates them.

## Cross-language wire contracts

When the two ends are different languages (e.g. a Python service and a C#
service), the binary framing is a contract that exists nowhere the type
checker can see it — treat it with extra care:

- **Write the format down once, in both places, with an explicit comment
  cross-referencing the other side** (e.g. Python's `struct` format string and
  the C# `BinaryReader`/struct layout that must match it byte-for-byte).
  Endianness in particular is a silent, hard-to-debug mismatch if the two
  sides disagree.
- Prefer fixed-width integer types over anything platform-dependent, and
  state the endianness explicitly in the format (`<` for little-endian in
  `struct`) rather than relying on the platform default.
- **Version the protocol if the two sides can deploy independently** — a type
  byte or version field lets a receiver detect and reject a shape it doesn't
  understand instead of misparsing it as something else.
- Changing the binary layout is a breaking change to both codebases at once —
  treat it with the same care as a database migration touching two services:
  plan the rollout order, don't assume both sides update atomically.

## Backpressure and errors

- `writer.write()` followed by `await writer.drain()` — `drain()` is what
  applies backpressure; skipping it lets you queue unbounded data into the
  OS send buffer if the peer reads slowly, the same failure mode as an
  unbounded `asyncio.Queue` (`chunked-streaming`).
- Wrap the read loop in a broad exception handler at the connection level —
  one malformed message or a peer disconnecting mid-read should close that
  connection cleanly, not crash the server or leave other connections
  affected.
- Treat every field read off the wire as untrusted input (`security`) —
  validate lengths and types before acting on them, especially on a
  length-prefixed binary path where a corrupted length could otherwise drive
  a huge or negative read.

## Testing

- Test the framing/parsing logic (encode/decode functions) as pure functions,
  independent of an actual socket — feed bytes in, assert the parsed message
  out, including a deliberately truncated/malformed input.
- For the connection-handling logic, use a real Unix socket pair
  (`socket.socketpair()` wrapped for asyncio, or a real `start_unix_server` on
  a temp path) rather than mocking the reader/writer — the framing bugs that
  matter here are almost always about byte boundaries, which a mock tends to
  paper over.
- Test the retry-on-connect path (server not up yet) and the
  disconnect-mid-message path explicitly — both are easy to get right on the
  happy path and wrong under real conditions.

## Pitfalls that recur

- **Assuming one `read()`/`recv()` equals one message** — streams have no
  inherent message boundaries; only the framing creates them.
- **Stale socket file blocking a restart** — the old file left behind by a
  crashed process prevents the new server from binding.
- **No connect retry** — a client that starts before its server's socket
  exists fails immediately instead of waiting briefly.
- **Endianness/format drift between two languages** — a silent, hard-to-debug
  corruption rather than a clean error.
- **Skipping `drain()`** — unbounded backpressure-free writes into the OS
  buffer.
- **Trusting a length field from the wire without validation** — a corrupted
  or malicious length driving an oversized read.

## Where this fits

`process-ipc` is the transport this stack uses for same-host communication
between processes — a sibling worker (`process-supervision` starts and
supervises it), a service hosting a `local-model-inference` model, or a
companion process in another language such as `csharp-stack`. Use
`python-websockets` instead when the peer is over the network, not on the
same host.
