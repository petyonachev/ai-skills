---
name: audio-processing
description: >-
  Real-time audio DSP building blocks — PCM formats, resampling with
  anti-aliasing, voice activity detection (VAD) framing, codec decode/encode
  (Opus), and speech-segment buffering/trimming. Use when processing raw
  audio frames, converting sample rates, running VAD, decoding/encoding
  Opus, or debugging garbled/aliased/misaligned audio. Sits below
  `chunked-streaming` (which handles the pipeline/backpressure shape) and
  above the raw bytes. Triggers: "PCM", "sample rate", "resample",
  "downsample", "VAD", "voice activity", "Opus", "audio frame", "aliasing",
  "s16le".
---

# Audio processing

Real-time audio work is mostly bookkeeping — knowing exactly what format your
bytes are in at every stage — with a few DSP facts that bite hard if
ignored. Get the format (rate, channels, sample width) explicit and correct
at every boundary, and treat frame math as exact, not approximate.

## PCM formats — be explicit, always

- State the format everywhere it matters: sample rate (e.g. 48kHz), channel
  count (mono/stereo), sample width and encoding (`s16le` — signed 16-bit
  little-endian is the common case). A buffer of raw bytes carries none of
  this information itself — a comment or a type/dataclass wrapping it should.
- Different sources disagree on format (e.g. a voice platform delivering
  48kHz stereo, a VAD model requiring 16kHz mono) — conversion between them is
  a first-class step in the pipeline, not an afterthought.
- `numpy.frombuffer(data, dtype=np.int16)` to interpret raw bytes as samples;
  remember this is a *view*, not a copy — mutating it mutates the original
  buffer, and it requires the byte length to be an exact multiple of the
  sample width (trim any odd trailing byte before reshaping).

## Resampling — anti-aliasing is not optional

- Changing sample rate by a simple integer ratio (e.g. 48kHz → 16kHz is ÷3)
  is decimation: keeping every Nth sample. **Decimation without a low-pass
  filter first aliases** — frequency content above the new Nyquist limit
  folds back into the audible range as noise/distortion, corrupting
  downstream processing (a VAD or STT model sees corrupted input, not just
  "lower quality" input).
- A cheap, adequate anti-aliasing filter for speech is often good enough — a
  boxcar (moving-average) filter over the decimation window (e.g. averaging
  each group of 3 samples before dropping to 1) acts as a simple low-pass
  filter at roughly the new Nyquist frequency. Reach for a proper resampling
  library (`scipy.signal.resample`, `librosa`) when quality requirements
  exceed what a boxcar filter delivers, but know why you're upgrading, not
  by default.
- Downmixing stereo to mono: average the two channels
  (`(left.astype(int32) + right.astype(int32)) // 2`), not just discard one
  channel — dropping a channel loses anything panned there.

## Frame-size math — exact, not approximate

- A model or codec that requires fixed-size frames (e.g. "10ms frames" for a
  VAD model) has an exact byte count implied by the sample rate and width —
  compute it explicitly (`hop_size * bytes_per_sample`) and treat any
  mismatch as a bug, not something to round.
- **Buffer incoming audio and only process complete frames** — a
  network/transport chunk boundary rarely aligns with the frame size the
  model needs. Accumulate into a buffer, slice off complete
  frames as they become available, and leave the remainder buffered for the
  next chunk (the same discipline `chunked-streaming` describes generally,
  applied to fixed-size audio frames specifically).
- Trim buffers explicitly to a multiple of the required unit before reshaping
  with numpy (e.g. `numpy.reshape`) — an odd leftover sample/byte will raise
  or silently misalign the reshape otherwise.

## Voice activity detection (VAD)

- A VAD model gives a per-frame speech/non-speech signal, not a
  segment boundary — segment detection is built on top: track a
  "speaking" state per source, accumulate frames into a buffer while
  speaking, and decide when enough consecutive silent frames means the
  segment ended (a threshold in frames, i.e. milliseconds, not a single
  silent frame — one silent frame is normal mid-utterance).
- **The transport can stop delivering frames without telling you the speaker
  stopped** (e.g. a voice platform pausing frame delivery on silence
  instead of sending silent frames). VAD's silence-frame counter alone
  cannot detect this — pair it with an inactivity timeout that fires
  end-of-segment if no frames arrive at all for some duration, independent of
  what the VAD itself has seen.
- When emitting a completed segment, **trim the trailing silence** that was
  buffered while confirming the speech had ended — otherwise every segment
  carries a fixed tail of silence that a downstream STT model has to eat
  through for no benefit.
- Keep VAD state **per source** (per speaker/user/channel) when multiple
  concurrent audio sources exist — a shared VAD state across sources produces
  segments that splice together audio from different speakers
  (`chunked-streaming`'s per-entity keyed state pattern applies directly
  here).

## Codec decode/encode (Opus and similar)

- Decode as early as possible in the pipeline (at ingestion) so every later
  stage works with a known, fixed PCM format rather than each needing its own
  codec awareness.
- A codec decoder/encoder instance is typically **stateful per source** (it
  tracks things like the encoder's internal prediction state) — do not share
  one decoder across multiple concurrent audio sources; keep one per source,
  the same way VAD state is kept per source.
- Handle a decode failure (corrupted/truncated packet) as a recoverable,
  per-packet event — log and drop the packet, don't let one bad packet crash
  the source's whole pipeline.

## Testing

- Generate synthetic PCM (a sine wave, or fixed known sample values) for
  deterministic assertions rather than relying on a real recorded audio file
  for most tests — reserve real audio fixtures for a narrow
  integration/quality test.
- Test resampling by asserting the output length and spot-checking known
  frequency content is preserved (or that a known-aliasing case is
  correctly filtered), not just that it "runs without error."
- Test VAD segment boundaries with a crafted frame sequence (speech, then N
  silent frames, then more speech) to assert the segment-end/trim logic
  fires at the right point — this is exactly the kind of edge-case logic that
  looks right by inspection and is wrong in practice.

## Pitfalls that recur

- **Decimating without anti-aliasing** — corrupts everything downstream with
  aliased noise, not just "lower quality" audio.
- **Assuming a transport chunk equals one processing frame** — buffer and
  slice explicitly.
- **Relying only on VAD silence to detect end-of-speech** — misses the case
  where the transport just stops delivering frames.
- **Shared VAD/codec state across concurrent sources** — splices unrelated
  audio together.
- **Not trimming trailing silence** — every segment carries dead weight into
  the next stage.

## Where this fits

`audio-processing` supplies the DSP building blocks that a `chunked-streaming`
pipeline stage uses on the audio-specific parts of its job, and commonly feeds
`local-model-inference` (an STT model) once a segment is complete. It sits
below the pipeline-shape concerns (queues, backpressure, staging) and above
the raw bytes coming off `process-ipc` or `python-websockets`.
