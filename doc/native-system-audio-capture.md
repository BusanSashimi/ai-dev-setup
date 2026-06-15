# Feature: Native system-audio capture (SCK PCM → frontend mix)

**Priority:** P1  ·  **Effort:** L  ·  **Risk:** med-high  ·  **Surface:** both

## Problem / motivation

System (application) audio capture — recording what's *playing* on the machine, mixed
with the mic — **does not work in the packaged app today**, even though most of the
backend already exists.

The live recording path is the frontend WebCodecs pipeline (`RecordingCanvas`). Its
audio acquisition (`acquireAudioTrack`, `recording-canvas.tsx:437`) gets system audio
from `navigator.mediaDevices.getDisplayMedia({ video: { width: 1, height: 1 }, audio:
true })` (line 505-508). But **`getDisplayMedia` is absent in this WKWebView** — the
same gap that forced the native ScreenCaptureKit screen path. The code even guards for
it and logs a warning:

```ts
// recording-canvas.tsx:499-502
if (wantSystemAudio && !hasMediaApi("getDisplayMedia")) {
  console.warn("[Audio] System audio unavailable: navigator.mediaDevices.getDisplayMedia is missing in this webview.");
}
```

(See `[[webkit-getusermedia-works-getdisplaymedia-blocked]]`: getUserMedia/mic works in
the dev WKWebView, `getDisplayMedia` does not. So mic records fine; system audio is the
one missing source.)

Meanwhile a **complete SCK audio capturer already exists** —
`src-tauri/src/system_audio_macos.rs` — but it is only consumed by the **dead** native
recording pipeline (`manager.rs`; see `[[legacy-native-pipeline-removal]]`). It is not
exposed as a Tauri command and never reaches the live path.

This plan closes the gap the same way the screen path did: capture system audio
natively with SCK and **stream raw PCM to the frontend over a Tauri `Channel`**, where
it is mixed into the existing recording `AudioContext` alongside the mic. The transport
mirrors the just-shipped raw-byte screen transport
(`[[native-screen-transport-raw-bytes]]` / `native-screen-transport-raw-bytes.md`).

## Current state (grounded)

**Backend — SCK audio capture already implemented (`src-tauri/src/system_audio_macos.rs`):**
- `pub struct SystemAudioCapture { config, running, chunk_sender, chunk_receiver, stream, is_available }` (line 12-19).
- `start()` (line 71-124) builds an `SCContentFilter` over the first display and an
  `SCStreamConfiguration::new().with_captures_audio(true).with_sample_rate(self.config.sample_rate as i32).with_channel_count(self.config.channels as i32)`
  (line 93-96), attaches an `AudioHandler` via `stream.add_output_handler(handler, SCStreamOutputType::Audio)` (line 110), and `start_capture()`s.
- `AudioHandler::did_output_sample_buffer` (line 29-46) calls `extract_audio_samples`
  and pushes an `AudioChunk { samples: Vec<f32>, sample_rate: u32, channels: u16, timestamp: Duration }`
  through a **bounded(30)** crossbeam channel (`try_send`, line 45 — drops when full).
- `extract_audio_samples` (line 144-185) reads the `CMSampleBuffer` audio buffer list and
  returns **interleaved f32** samples (mono passthrough, else per-channel interleave).
- `take_receiver(&mut self) -> Option<Receiver<AudioChunk>>` (line 67-69), `stop()` (line 126-136).
- `AudioChunk` is defined in `src-tauri/src/audio.rs` (the otherwise-dead native mic
  module); `SystemAudioCaptureConfig` in `src-tauri/src/system_audio.rs`. **Both must
  survive** the legacy removal — see Edge cases.

**Backend — transport precedent (`src-tauri/src/screen_stream.rs`):**
- `start_screen_stream(section_index, …, on_frame: Channel<InvokeResponseBody>, state)`
  spawns a worker that sends `InvokeResponseBody::Raw(header ++ payload)` and blocks on a
  `parking_lot::Condvar` credit (`MAX_INFLIGHT = 2`); `ack_screen_frame` restores credit.
  `ScreenStreamState { streams: Mutex<HashMap<u32, StreamHandle>> }` (module-private).
- This is the exact shape to copy: a `SystemAudioStreamState`, a worker pulling
  `AudioChunk`s from `take_receiver()`, and a raw-bytes Channel.

**Frontend — audio acquisition + mix (`frontend/src/components/asmr-recorder/recording-canvas.tsx`):**
- `acquireAudioTrack(wantMic, wantSystemAudio)` (line 437-544): pushes mic and/or system
  `MediaStream`s into `audioStreams[]`. With **one** stream it returns that track
  directly (line 519-525). With **multiple**, it creates
  `new AudioContext({ sampleRate: AUDIO_SAMPLE_RATE })`, a `createMediaStreamDestination()`,
  connects each stream as a `createMediaStreamSource`, and returns the destination track
  (line 527-541). **This destination is the mix injection point.**
- `ensureAudioContext()` (line 555-562) returns the live recording `AudioContext` (WebKit
  may clamp the rate; everything downstream uses `audioCtx.sampleRate`).
- `startAudioProcessing(audioTrack, audioEncoder)` (line 571-657) wires the (possibly
  mixed) track through a **`ScriptProcessorNode`** (bufferSize 4096) — explicitly chosen
  "instead of MediaStreamTrackProcessor for WebKit/WKWebView compatibility" (line 568-569)
  — and feeds `AudioData` (`f32-planar`) into the `AudioEncoder`.
- Constants: `AUDIO_SAMPLE_RATE = 48000` (line 70), `AUDIO_NUM_CHANNELS = 2` (line 71).

**Frontend — native bridge precedent (`frontend/src/lib/native-screen.ts`):**
- `startNativeScreenStream` → `new Channel<ArrayBuffer>()`, `channel.onmessage` parses a
  fixed LE header via `DataView`, acks; `invoke("start_screen_stream", { …, onFrame: channel })`.
  Mirror this as `native-system-audio.ts`.

**Not wired:** `system_audio_macos.rs` is not in `lib.rs`'s `generate_handler!`
(line 179-204) and has no frontend caller. The frontend system-audio source is solely the
(blocked) `getDisplayMedia` branch above.

## Proposed design

Three layers, mirroring the screen path.

**(1) Backend: `system_audio_stream.rs` — stream SCK PCM over a raw Channel.**
A new module (sibling to `screen_stream.rs`) owning a `SystemAudioStreamState {
streams: Mutex<HashMap<u32, SysAudioHandle>> }` (single stream in practice, keyed like
screen for symmetry). Commands:

- `start_system_audio_stream(section_index: u32, sample_rate: u32, channels: u16, on_chunk: Channel<InvokeResponseBody>, state)`:
  builds `SystemAudioCapture::new(SystemAudioCaptureConfig { sample_rate, channels, .. })`,
  `take_receiver()`, `start()`, then spawns a worker that drains `AudioChunk`s and sends
  each as `InvokeResponseBody::Raw(header ++ pcm)`.
- `stop_system_audio_stream(section_index, state)`: stop + join, `capture.stop()`.
- `ack_system_audio_chunk(section_index, state)`: credit++ (same Condvar pattern as
  `ack_screen_frame`).

Wire format — **12-byte LE header** + interleaved **f32** PCM (the format SCK already
produces, no conversion):

```
bytes  0..4  : u32 sample_rate
bytes  4..6  : u16 channels
bytes  6..8  : u16 reserved (0)
bytes  8..16 : u64 timestamp_ms     // from AudioChunk.timestamp.as_millis()
bytes 16..   : f32 interleaved PCM   // little-endian, channels * frames samples
```

(16-byte header to keep the f32 payload 4-byte aligned for a zero-copy `Float32Array`
view on the JS side — see Edge cases.) Backpressure: audio is low-bandwidth (48 kHz × 2 ×
4 B ≈ 384 KB/s), and SCK already uses a `bounded(30)` channel that drops on overflow, so a
**larger credit (`MAX_INFLIGHT = 8`)** is plenty; the worker still blocks via Condvar so a
stalled frontend can't pile unbounded `Raw` messages into the webview.

**(2) Frontend: `native-system-audio.ts` — Channel → AudioContext source node.**
Mirror `native-screen.ts`. `startNativeSystemAudioStream(sectionIndex, audioCtx, destination, opts)`:
- `new Channel<ArrayBuffer>()`; `onmessage` parses the header, builds a
  `Float32Array(buf, 16)` view (interleaved), de-interleaves into an `AudioBuffer` of
  `channels` at `sample_rate`, and enqueues it into a **playback queue** feeding the mix.
- Feed the queue into `audioCtx` via an **`AudioWorkletNode`** (a tiny ring-buffer
  worklet) connected to the same `destination` node the mic mixes into. **Fallback:** if
  `audioCtx.audioWorklet` is unavailable in this WKWebView, use a `ScriptProcessorNode`
  pull-from-ring-buffer (the established WKWebView-compatible primitive, per
  `startAudioProcessing`). The node's `gain`/output goes to `destination`, **not** to
  `audioCtx.destination` (we must not play system audio back out the speakers — that would
  loop). Ack after each chunk is enqueued.

**(3) Integrate into `acquireAudioTrack`.** Replace the `getDisplayMedia` system-audio
branch with: if running under Tauri/WKWebView (no `getDisplayMedia`), force the
multi-stream mixing context (create the `AudioContext` + `destination` even for mic-only),
start the native system-audio stream feeding `destination`, and return the destination
track. The downstream `startAudioProcessing` path is **unchanged** — it already reads the
mixed track. Keep the `getDisplayMedia` branch as a non-WKWebView fallback.

Data flow:

```
SCK audio → AudioHandler → AudioChunk(f32 interleaved) → bounded(30)
   worker: drain → wait credit>0 → credit-- → Channel.send(Raw header++pcm)
JS onmessage(ArrayBuffer) → parse header → Float32Array view → de-interleave → AudioBuffer
   → ring buffer → AudioWorkletNode → destination (mix with mic)
   → invoke("ack_system_audio_chunk") → worker credit++
destination track → startAudioProcessing → AudioEncoder → muxer (unchanged)
```

## Implementation steps

1. **Preserve the SCK audio building blocks.** Confirm `AudioChunk` (`audio.rs`) and
   `SystemAudioCaptureConfig` (`system_audio.rs`) compile independently of the dead native
   pipeline. If `[[legacy-native-pipeline-removal]]` lands first, those types must already
   have been relocated into a slim `system_audio` module — **do the system-audio work
   before, or in coordination with, the removal** (see Edge cases). **verify:** `cargo build`
   (default features) clean with `system_audio_macos` reachable.

2. **`system_audio_stream.rs` — state + start/stop.** Add `SystemAudioStreamState`,
   `SysAudioHandle { running: Arc<AtomicBool>, worker: Option<JoinHandle<()>>, credit: Arc<(Mutex<u32>, Condvar)> }`,
   `const MAX_INFLIGHT: u32 = 8;`. `start_system_audio_stream` creates the capturer, takes
   the receiver, starts it, and spawns the worker loop (drain `recv_timeout` → build header
   → Condvar credit wait re-checking `running` → `Channel::send(Raw)`). Copy the
   stop/Drop/credit machinery verbatim from `screen_stream.rs`. **verify:** `cargo build`;
   with no frontend acking, `stop_system_audio_stream` returns within `RECV_TIMEOUT`.

3. **`ack_system_audio_chunk` + register.** Add the ack command (credit++/notify, capped at
   `MAX_INFLIGHT`) in `system_audio_stream.rs`; add `mod system_audio_stream;` and register
   `start_system_audio_stream` / `stop_system_audio_stream` / `ack_system_audio_chunk` in
   `lib.rs` `generate_handler!` (line 179-204) and `.manage(SystemAudioStreamState::default())`.
   **verify:** app launches; no "command not found" IPC error.

4. **`native-system-audio.ts`.** Implement `startNativeSystemAudioStream` /
   `stopNativeSystemAudioStream` mirroring `native-screen.ts`: Channel<ArrayBuffer>, header
   parse, de-interleave to `AudioBuffer`, ring-buffer → `AudioWorkletNode` (with
   `ScriptProcessorNode` fallback) → caller-supplied `destination`. Add the worklet
   processor as a small inline module string registered via `audioCtx.audioWorklet.addModule(URL.createObjectURL(blob))`.
   **verify:** `npx tsc -b` clean; the module registers without throwing in dev.

5. **Integrate in `acquireAudioTrack` (`recording-canvas.tsx`).** When `wantSystemAudio`
   and `getDisplayMedia` is absent, ensure the mixing `AudioContext` + `destination` exist,
   call `startNativeSystemAudioStream(0, audioCtx, destination, { sampleRate: audioCtx.sampleRate, channels: AUDIO_NUM_CHANNELS })`,
   and return the destination track. Store a cleanup ref so recording-stop tears the native
   stream down (mirror `audioProcessingCleanupRef`). Leave the mic and `startAudioProcessing`
   paths untouched. **verify:** `npx tsc -b`; `npx vite build`.

6. **End-to-end (hybrid).** `npm run tauri dev`, grant Screen Recording TCC (SCK audio
   shares the screen-recording permission), play audio (music/video), record 20-30s with
   mic + system audio, save. Confirm backend logs `System audio capture started: 48000Hz, 2
   channels` and the worker's sent count; `ffprobe -show_streams out.mp4` shows one AAC
   track; **listen**: both mic and system audio are present, in sync, no pitch-skew, no
   speaker feedback loop. **verify:** audible mix correct; A/V sync holds over 30s.

## Interface / API changes

- **Rust (new `system_audio_stream.rs`):**
  `start_system_audio_stream(section_index: u32, sample_rate: u32, channels: u16, on_chunk: Channel<tauri::ipc::InvokeResponseBody>, state)`,
  `stop_system_audio_stream(section_index: u32, state)`,
  `ack_system_audio_chunk(section_index: u32, state)`. New managed `SystemAudioStreamState`.
- **TS (new `native-system-audio.ts`):** `startNativeSystemAudioStream(sectionIndex, audioCtx, destination, opts): Promise<Channel<ArrayBuffer>>`,
  `stopNativeSystemAudioStream(sectionIndex)`.
- **`recording-canvas.tsx`:** `acquireAudioTrack` gains a native system-audio branch; no
  signature change. `startAudioProcessing` / muxer / encoder config **unchanged**.
- **No change** to `save_media_recording` or the trim/save flow.

## Edge cases & risks

- **Clock drift (highest risk).** SCK's audio clock and the WebKit `AudioContext` clock are
  independent; over minutes the producer and consumer rates diverge, so the ring buffer
  slowly fills or drains. Mitigate with a bounded ring buffer that **drops oldest on
  overflow** and **inserts a tiny silence pad on underflow**, plus a target fill level
  (~2-3 buffers ≈ 250 ms). A few dropped/padded samples per minute are inaudible; an
  unbounded queue would add growing latency and desync from video. Log drift counters.
- **Sample-rate mismatch.** SCK is configured at `sample_rate`, but the recording
  `AudioContext` may clamp to the hardware rate (the existing `ensureAudioContext` comment,
  line 549-553). Pass `audioCtx.sampleRate` into `startNativeSystemAudioStream` and resample
  in the worklet if `chunk.sample_rate != audioCtx.sampleRate` (linear is adequate for a
  mix that's re-encoded anyway), or request SCK at the same rate. **Do not** assume 48000.
- **No speaker feedback loop.** The system-audio source node must connect **only** to the
  mix `destination` (which feeds the encoder), never to `audioCtx.destination` (speakers).
  Connecting to speakers would replay captured audio and re-capture it → runaway feedback.
- **f32 alignment / `buf.slice`.** Tauri delivers `Raw` as an `ArrayBuffer`; a 16-byte
  header keeps the PCM region 4-byte aligned so `new Float32Array(buf, 16)` is a zero-copy
  view (a 12-byte header would force a copy). Verify the Channel's `ArrayBuffer` offset is 0.
- **AudioWorklet support in WKWebView.** Not guaranteed in this webview (the codebase
  deliberately uses `ScriptProcessorNode` elsewhere for WKWebView compat, line 568-569).
  **Verify `audioCtx.audioWorklet` exists at runtime**; if not, use the ScriptProcessor
  ring-buffer fallback. Don't assume worklets work.
- **TCC permission coupling.** SCK audio capture requires the **Screen Recording**
  permission (it's an `SCStream`). A user who only wants mic must still grant screen
  recording to get system audio — surface this in the UI when enabling system audio.
- **Dependency on the dead pipeline.** `AudioChunk`/`SystemAudioCaptureConfig` currently
  live in modules slated for removal. Sequence: land this (relocating those types into the
  surviving `system_audio` module) **before** `[[legacy-native-pipeline-removal]]`, or do
  the relocation as step 1 here.
- **`bounded(30)` drop-on-full.** SCK's handler `try_send`s and silently drops when the
  worker is blocked on credit. With `MAX_INFLIGHT = 8` and prompt acks this won't trigger in
  practice, but log dropped chunks so a regression is visible.

## Testing & verification

- **Build gates (auto):** `cargo build` (default features — `--features ffmpeg` does not
  link on this arm64 machine and is unrelated; `docs/logs/2026-06-13.md`), `npx tsc -b`,
  `npx vite build`.
- **Runtime (hybrid, per house setup):** `npm run tauri dev`; user grants Screen Recording
  TCC and drives capture (frontend console isn't piped to the terminal; WKWebView capture
  needs interactive TCC). Observable: backend `println!` `System audio capture started: …`,
  worker sent/dropped counts. Record mic+system 20-30s → save → `ffprobe -show_streams`
  (one AAC track, 48 kHz, stereo) and **listen** for: both sources present, in sync, no
  pitch-skew (validates the rate-coherence handling), no feedback howl.
- **Drift check (manual):** record ~3 min; confirm system audio doesn't progressively lead/
  lag the video, and the drift counters stay bounded.
- **Cannot auto-verify:** the audible mix quality, drift over long recordings, and
  AudioWorklet-vs-ScriptProcessor selection — all need the user's interactive pass.

## Out of scope / future

- Per-application audio capture (SCK can filter by app); v1 captures the whole system mix.
- Independent mic/system level faders and a pre-record monitor for system audio (the
  monitor today is mic-only, `use-audio-monitor.ts`).
- Capturing system audio **without** video/screen (SCK needs a content filter; we use the
  first display). Fine since the permission is already required.
- Windows/Linux system audio (the `system_audio_fallback.rs` loopback path) — macOS-first.

## References

- `src-tauri/src/system_audio_macos.rs` — `SystemAudioCapture` (12-19), `start()` (71-124,
  SCStreamConfiguration audio config 93-96), `AudioHandler` (21-47), `extract_audio_samples`
  (144-185), `take_receiver` (67-69), `stop()` (126-136).
- `src-tauri/src/audio.rs` — `AudioChunk { samples, sample_rate, channels, timestamp }`.
- `src-tauri/src/system_audio.rs` — `SystemAudioCaptureConfig`, macOS/fallback module routing.
- `src-tauri/src/screen_stream.rs` — transport precedent: `Channel<InvokeResponseBody>`,
  Condvar credit (`MAX_INFLIGHT`), `ack_screen_frame`, `StreamHandle`/Drop/stop, header build.
- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — `acquireAudioTrack` (437-544),
  `getDisplayMedia` system-audio branch + WKWebView guard (499-515), multi-stream mix
  (527-541), `ensureAudioContext` (555-562), `startAudioProcessing` ScriptProcessorNode
  (571-657), audio constants (70-73).
- `frontend/src/lib/native-screen.ts` — Channel/header/ack bridge to mirror.
- `src-tauri/src/lib.rs` — `generate_handler!` (179-204), `.manage(...)` registration.
- Memory: `[[webkit-getusermedia-works-getdisplaymedia-blocked]]`,
  `[[native-screen-transport-raw-bytes]]`, `[[legacy-native-pipeline-removal]]`.
- Dev log: `docs/logs/2026-06-13.md` (WKWebView capture facts), `docs/logs/2026-06-14.md`
  (raw-byte screen transport).
