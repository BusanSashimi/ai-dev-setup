# Feature: Native screen transport: raw-byte frames + true backpressure

**Priority:** P1  ·  **Effort:** M  ·  **Risk:** med  ·  **Surface:** both

## Problem / motivation

The WKWebView screen path (added 2026-06-13) streams the captured display to the
frontend as **base64-encoded JPEG strings** over a Tauri `Channel`, with no
worker→JS flow control. The dev log explicitly defers the fix:

> Deferred (noted): true worker->JS backpressure (ack / raw-bytes
> `InvokeResponseBody::Raw` instead of base64) — partially mitigated by the 30fps
> cap + in-flight gate
> — `docs/logs/2026-06-13.md` (Part B "Deferred", line ~160)

Two real costs today:

1. **Base64 tax.** `ScreenStreamFrame.data` is `STANDARD.encode(&jpeg)` in
   `screen_stream.rs` (line 185) — a `String` that is **~33% larger** than the JPEG
   bytes and must be UTF-8-validated and JSON-string-escaped through the Channel
   (the Channel serializes `Raw` vs `Json` payloads differently; a `String` goes the
   `Json` route). On the JS side `native-screen.ts` re-parses it via
   `fetch("data:image/jpeg;base64,...")`, which **decodes the base64 on the main
   thread** before `createImageBitmap`.
2. **No backpressure → lossy mitigation.** `Channel.send` is fire-and-forget, so
   `native-screen.ts` added a *single-in-flight, latest-wins* gate (`decoding` /
   `pending`, line 49-75) and `preview.tsx` caps native screen to
   `Math.min(externalConfig.frameRate || 30, 30)` (line 261). When decode can't keep
   up, frames are **silently dropped** rather than throttling the producer. The dev
   log records the rationale: "60fps per-frame main-thread JPEG decode would jank the
   composite — capped native screen to 30fps" (line ~153).

The precedent for the fix already exists in-repo: `save_media_recording` in
`lib.rs` was migrated from a base64 `String` arg to Tauri v2 raw-body IPC
(`tauri::ipc::InvokeBody::Raw(video_bytes)`, line 85) on the *upload* direction
(`docs/logs/2026-06-13.md` line ~423). Note that is the request-**body** enum
(`InvokeBody`); the worker→JS Channel uses the response-body enum
(`InvokeResponseBody`) — two distinct types that share a `Raw(Vec<u8>)` variant.
This plan applies the same raw-bytes idea to the *download* (worker→JS) direction
and adds the ack loop the Channel lacks.

## Current state (grounded)

**Backend — `src-tauri/src/screen_stream.rs`:**
- `ScreenStreamFrame { data: String, width, height, timestamp_ms }` — `data` is
  base64 JPEG (line 32-43; `#[derive(Clone, Serialize)]`, `serde(rename_all = "camelCase")`).
- `encode_frame(rgba: RgbaImage, src_w, src_h, max_dimension) -> Result<(Vec<u8>, u32, u32), String>`
  downscales to fit `max_dimension` (`FilterType::Triangle`, never upscales), drops
  alpha, and JPEG-encodes at `JPEG_QUALITY = 80` (line 77-110).
- `start_screen_stream(section_index, display_index, fps, max_dimension, on_frame: Channel<ScreenStreamFrame>, state)`
  (async command, line 116-231) spawns a worker thread that pulls `ScreenFrame`s from
  `ScreenCapture` (`receiver.recv_timeout(RECV_TIMEOUT)`, `RECV_TIMEOUT = 100ms`),
  **drains the channel to the latest frame** (`while let Ok(newer) = receiver.try_recv()`,
  line 167-169), converts via `to_rgba()`, encodes, base64s into a `ScreenStreamFrame`,
  and `on_frame.send(frame)`. On send error it `break`s (frontend dropped channel).
  Worker exit calls `capture.stop()` and
  `println!("[screen_stream] section {} stopped: {} frames sent, {} errors", …)` (line 206-209).
- `ScreenStreamState { streams: Mutex<HashMap<u32, StreamHandle>> }` (parking_lot
  `Mutex`; `streams` is **module-private**, so any command touching it must live in
  this file). `StreamHandle { running: Arc<AtomicBool>, worker: Option<JoinHandle<()>> }`
  with `stop()` (signal + join) + `Drop` (signal only, no join). `stop_screen_stream(section_index, state)`
  removes + stops.
- `ScreenFrame` (in `screen.rs`) exposes `to_rgba()` (stride-correct BGRA→RGBA),
  `to_packed_bgra()`, and public fields `width: u32`, `height: u32`, `stride: usize`,
  `timestamp: Duration`.

**Frontend — `frontend/src/lib/native-screen.ts`:**
- `interface ScreenStreamFrame { data: string; width; height; timestampMs }`.
- `startNativeScreenStream(sectionIndex, onFrame, opts)` creates
  `new Channel<ScreenStreamFrame>()`, sets `channel.onmessage`, and
  `invoke("start_screen_stream", { …, onFrame: channel })`. The `decode()` path is
  `fetch("data:image/jpeg;base64," + frame.data).then(r=>r.blob()).then(createImageBitmap)`.
  `decoding`/`pending` enforce single-in-flight latest-wins (line 49-75).
- `stopNativeScreenStream(sectionIndex)` → `invoke("stop_screen_stream", …)`.

**Consumer — `frontend/src/components/asmr-recorder/preview.tsx`:**
- `startNativeScreen(index)` (line 230-283) calls `startNativeScreenStream(index, (bitmap)=>{…draw into nativeCanvasRefs…; bitmap.close()}, { fps: Math.min(externalConfig.frameRate || 30, 30), maxDimension: Math.ceil(externalConfig.outputWidth / 2) })`.
  The `fps` cap is line 261. Teardown via `stopNativeStreamForSection` (line 220-226)
  called from clear/camera/region/unmount.

**Tauri ground-truth (`tauri-2.9.5/src/ipc/channel.rs`, `ipc/mod.rs`):**
- `Channel<TSend = InvokeResponseBody>`; `send(data: TSend) where TSend: IpcResponse`
  (channel.rs line 292-294). `InvokeResponseBody` is `enum { Json(String), Raw(Vec<u8>) }`
  (mod.rs line 99-104) and `impl IpcResponse for InvokeResponseBody` (mod.rs line 127),
  so `Channel<InvokeResponseBody>::send(InvokeResponseBody::Raw(bytes))` is valid.
- Delivery of a `Raw` body to JS depends on size: payloads **< 1024 bytes**
  (`MAX_RAW_DIRECT_EXECUTE_THRESHOLD`) are eval'd inline as
  `new Uint8Array([...]).buffer` (channel.rs line 163-166); **larger** payloads go via
  the `plugin:__TAURI_CHANNEL__|fetch` path (line 168-181). A downscaled-quadrant JPEG
  is tens of KB, so it takes the **fetch path** — which also resolves to an
  `ArrayBuffer`. Either way `channel.onmessage` receives an `ArrayBuffer` directly —
  **no base64, no data: URL**.
- The Channel is **one-directional** (worker→JS only). It has no return/ack path,
  which is why backpressure needs a separate `invoke`.

## Proposed design

Two independent changes, both shippable on their own:

**(A) Raw-byte frame transport.** Send the JPEG bytes plus a tiny header as
`tauri::ipc::InvokeResponseBody::Raw` instead of a base64 `String` inside a JSON
struct. The JS side gets an `ArrayBuffer`, decodes via `new Blob([bytes])` →
`createImageBitmap(blob)` (no base64, off the data: URL path).

Frame wire format (self-describing, avoids a parallel metadata channel): a fixed
**16-byte little-endian header** prepended to the JPEG payload:

```
bytes  0..4   : u32 width
bytes  4..8   : u32 height
bytes  8..16  : u64 timestamp_ms
bytes 16..    : JPEG payload
```

Channel signature before/after:

```rust
// before
on_frame: Channel<ScreenStreamFrame>                  // ScreenStreamFrame.data: String (base64)
// after
on_frame: Channel<tauri::ipc::InvokeResponseBody>     // send Raw(header ++ jpeg)
```

`ScreenStreamFrame` (the Rust struct) is **deleted**; the TS interface is replaced
by the parsed-from-ArrayBuffer shape.

**(B) Ack-based backpressure.** Keep the JPEG encode (it is the right size/CPU
trade for a downscaled quadrant), but make the producer **wait for the consumer**.
Add a per-stream bounded credit (a small int, e.g. `MAX_INFLIGHT = 2`). The worker
decrements before `send` and **blocks** when credit hits 0; the frontend calls a
new `invoke("ack_screen_frame", { sectionIndex })` after each `createImageBitmap`
resolves, which increments the credit (via a `parking_lot::Condvar` stored alongside
the credit in `ScreenStreamState`). This replaces the lossy latest-wins gate with
real flow control: the worker still *drains to latest* (so we skip stale frames), but
it never outruns the decoder and never silently piles up.

Data flow:

```
SCK frame → worker: drain-to-latest → encode JPEG
          → wait for credit>0 (Condvar) → credit-- → Channel.send(Raw header++jpeg)
JS onmessage(ArrayBuffer) → parse header → createImageBitmap(Blob)
          → draw → invoke("ack_screen_frame") → worker credit++ (notify)
```

With (B) in place, the frontend's `decoding`/`pending` gate and the
`Math.min(…, 30)` cap in `preview.tsx` are **removed** — the producer self-limits to
the consumer's real rate, so higher fps is safe (it degrades to the decode rate
instead of janking).

## Implementation steps

1. **`screen_stream.rs` — add per-stream credit to state.** Extend `StreamHandle`
   with `credit: Arc<(parking_lot::Mutex<u32>, parking_lot::Condvar)>` and add
   `const MAX_INFLIGHT: u32 = 2;`. Build the `Arc` in `start_screen_stream`, clone it
   into the worker closure (the same way `worker_running` is cloned from `running`),
   and store the other clone in the inserted `StreamHandle` so `ack_screen_frame` can
   find it via `ScreenStreamState`.
   **verify:** `cargo check` clean (default features — `--features ffmpeg` does not
   link on this machine; see `docs/logs/2026-06-13.md` "Live verification").

2. **`screen_stream.rs` — replace base64 with raw header+JPEG.** Delete
   `ScreenStreamFrame` and the `base64::{… STANDARD …}` import (the only uses are the
   import at line 17 and `STANDARD.encode(&jpeg)` at line 185 — both removed; the
   `base64` crate stays in the workspace, still used by `receive_video_frame_base64`).
   In the worker, build the 16-byte LE header and concatenate:
   ```rust
   let mut payload = Vec::with_capacity(16 + jpeg.len());
   payload.extend_from_slice(&width.to_le_bytes());
   payload.extend_from_slice(&height.to_le_bytes());
   payload.extend_from_slice(&timestamp_ms.to_le_bytes());
   payload.extend_from_slice(&jpeg);
   if on_frame.send(tauri::ipc::InvokeResponseBody::Raw(payload)).is_err() { break; }
   ```
   Change the command param to `on_frame: Channel<tauri::ipc::InvokeResponseBody>`.
   (`width`/`height` are the `u32`s returned by `encode_frame`; `timestamp_ms` is the
   existing `u64`.)
   **verify:** `cargo check`; the existing `println!("… frames sent …")` still fires.

3. **`screen_stream.rs` — block the worker on credit.** Before `send`, lock the credit
   mutex and wait while credit is 0, re-checking the existing `running` `AtomicBool` so
   a dropped frontend doesn't deadlock the worker:
   ```rust
   {
       let (m, cv) = &*credit;          // `credit` is the cloned Arc captured by the worker
       let mut c = m.lock();
       while *c == 0 && worker_running.load(Ordering::Relaxed) {
           cv.wait_for(&mut c, RECV_TIMEOUT);   // parking_lot: &mut MutexGuard, returns WaitTimeoutResult (ignored)
       }
       if !worker_running.load(Ordering::Relaxed) { break; }
       *c -= 1;
   }
   ```
   The `wait_for(RECV_TIMEOUT)` ensures a frontend that stops acking still lets the
   worker observe `running == false` and exit within `RECV_TIMEOUT` — no deadlock on
   teardown. Keep the drain-to-latest loop (skipping stale frames while blocked is
   fine; we re-drain on the next iteration).
   **verify:** with the frontend never acking, `stop_screen_stream` still logs the
   stopped line within ~100ms (no hang).

4. **`screen_stream.rs` — add `ack_screen_frame` command** (must live here, since
   `ScreenStreamState.streams` is module-private):
   ```rust
   #[tauri::command]
   pub fn ack_screen_frame(section_index: u32, state: tauri::State<'_, Arc<ScreenStreamState>>) {
       if let Some(h) = state.streams.lock().get(&section_index) {
           let (m, cv) = &*h.credit;
           let mut c = m.lock();
           if *c < MAX_INFLIGHT { *c += 1; cv.notify_one(); }
       }
   }
   ```
   Initialize each new stream's credit to `MAX_INFLIGHT` so the first frames flow
   without waiting.
   **verify:** `cargo check`; command registered in step 5.

5. **`lib.rs` — register `ack_screen_frame`** in `generate_handler!` next to
   `screen_stream::start_screen_stream` / `screen_stream::stop_screen_stream`
   (line 200-201).
   **verify:** `cargo check`; app launches without a "command not found" IPC error.

6. **`native-screen.ts` — parse ArrayBuffer, drop base64.** Replace the
   `ScreenStreamFrame` interface with a parsed metadata shape
   `{ width: number; height: number; timestampMs: number }`. Type the channel as
   `new Channel<ArrayBuffer>()`; `onmessage` now receives an `ArrayBuffer`:
   ```ts
   channel.onmessage = (buf: ArrayBuffer) => {
     const dv = new DataView(buf);
     const width = dv.getUint32(0, true);
     const height = dv.getUint32(4, true);
     const timestampMs = Number(dv.getBigUint64(8, true));
     const jpeg = new Blob([buf.slice(16)], { type: "image/jpeg" });
     createImageBitmap(jpeg)
       .then((bitmap) => { onFrame(bitmap, { width, height, timestampMs }); ack(sectionIndex); })
       .catch((e) => { console.warn("[native-screen] decode failed:", e); ack(sectionIndex); });
   };
   ```
   Add `function ack(i: number) { invoke("ack_screen_frame", { sectionIndex: i }).catch(() => {}); }`.
   **Always ack** (even on decode failure) so a single bad frame can't starve credit.
   Remove the `decoding`/`pending` gate and the `decode()` helper. The return type
   becomes `Promise<Channel<ArrayBuffer>>`.
   **verify:** `npx tsc -b` no new errors (only the pre-existing ones noted in the log).

7. **`preview.tsx` — drop the 30fps cap.** Change line 261
   `fps: Math.min(externalConfig.frameRate || 30, 30)` to
   `fps: externalConfig.frameRate || 30` (the producer now self-limits). Leave
   `maxDimension` and the draw/`bitmap.close()` callback unchanged.
   **verify:** `npx tsc -b`; `npx vite build` succeeds.

8. **End-to-end record check (hybrid).** `npm run tauri dev` (or launch vite + `cargo
   run --manifest-path src-tauri/Cargo.toml` with default features — `dev.sh` has a
   pre-existing `cd` bug and `--features ffmpeg` fails to link locally, both noted in
   the log). User assigns a native screen section, records 20-30s, trims, saves.
   Confirm backend logs `[screen_stream] … started` then `… frames sent, N errors`
   and `[Backend-MediaRecorder] Saved MP4: … (bytes)`; `ffprobe` the output in
   `test-results/` for a valid H.264 track with sane frame count.
   **verify:** ffprobe shows a decodable video track; `errors` count near 0.

## Interface / API changes

**Rust (`screen_stream.rs`):**
- `ScreenStreamFrame` struct: **removed**.
- `start_screen_stream(… on_frame: Channel<ScreenStreamFrame> …)` →
  `on_frame: Channel<tauri::ipc::InvokeResponseBody>` (sends `Raw`).
- **New:** `ack_screen_frame(section_index: u32, state)` (sync command).
- `StreamHandle` gains `credit: Arc<(parking_lot::Mutex<u32>, parking_lot::Condvar)>`.

**TS (`native-screen.ts`):**
- `interface ScreenStreamFrame { data: string; … }` → metadata-only
  `{ width: number; height: number; timestampMs: number }` (no `data`; bitmap
  delivered separately).
- `onFrame(bitmap, frame)` signature unchanged for `preview.tsx` (still
  `(bitmap: ImageBitmap, frame) => void`; `preview.tsx` ignores the second arg).
- `startNativeScreenStream` return type → `Promise<Channel<ArrayBuffer>>`.

**IPC body shapes:**
- Channel payload: JSON `{ data, width, height, timestampMs }` → raw `ArrayBuffer`
  (16-byte LE header + JPEG).
- New ack: `invoke("ack_screen_frame", { sectionIndex })`.

## Edge cases & risks

- **Teardown / no-ack deadlock.** The worker's credit wait MUST be
  `cv.wait_for(&mut guard, RECV_TIMEOUT)` and re-check the `running` `AtomicBool`, or
  a frontend that stops acking (tab hidden, channel dropped) would hang the worker and
  leak the SCStream. The `Drop`/`stop()` contract in the existing code (signal-only
  Drop, join in `stop()`) is preserved — the timeout makes the join bounded.
- **Stale credit after stop/restart.** `ack_screen_frame` looks up the *current*
  handle by `section_index`; a late ack for a replaced stream harmlessly bumps the
  new stream's credit (capped at `MAX_INFLIGHT`). Initializing credit to
  `MAX_INFLIGHT` on each new handle avoids cross-stream credit leakage.
- **WKWebView ArrayBuffer transfer.** Tauri delivers `Raw` via the
  `plugin:__TAURI_CHANNEL__|fetch` path for payloads ≥ 1024 bytes (which a quadrant
  JPEG always is) and inline as `new Uint8Array(...).buffer` for tiny ones; both
  resolve to an `ArrayBuffer` in WKWebView. `buf.slice(16)` copies the JPEG region —
  fine for a downscaled quadrant (tens of KB). `createImageBitmap(Blob)` is the same
  decode path as today, minus the base64 step, so the already-verified color pipeline
  is unchanged.
- **Ordering.** The `@tauri-apps/api` Channel preserves message order via an
  index-based reorder buffer in `core.js` (`_Channel_nextMessageIndex` /
  `_Channel_pendingMessages`). Acks are unordered but only adjust a counter, so order
  doesn't matter.
- **Regression surface.** The browser `getDisplayMedia` path and the
  record/trim/save pipeline (`save_media_recording`) are untouched — this only swaps
  the native-screen transport feeding `nativeCanvasRefs`.
- **`MAX_INFLIGHT` too low** could under-utilize a fast decoder (latency-bound at 1);
  2 keeps one frame decoding while the next is in flight. Tunable constant.

## Testing & verification

- **Build gates (auto):** `cargo check` (default features), `npx tsc -b` (expect only
  the pre-existing errors noted in `docs/logs/2026-06-13.md`), `npx vite build`.
- **Runtime (hybrid):** `npm run tauri dev`. The user grants Screen Recording TCC and
  drives the native picker (the frontend console is not piped to the terminal, and
  WKWebView capture needs interactive TCC). Observable signals:
  - backend `println!` lines: `[screen_stream] … started`, `… frames sent, N errors`.
  - record → trim → save, then `ffprobe -show_streams` / `-count_packets` on the
    `test-results/recording_*.mp4` for a valid H.264 track and frame count consistent
    with the recording length × target fps.
- **Backpressure sanity (manual):** raise `externalConfig.frameRate` to 60 and
  confirm the composite stays smooth (no jank) and `errors` stays ~0 — the producer
  throttles to decode rate instead of dropping/janking.
- **Cannot auto-verify:** main-thread CPU/jank and the perceived frame rate (needs a
  human watching the live composite); the exact base64-vs-raw byte savings (inferred
  from the 33% base64 expansion, not instrumented).

## Out of scope / future

- Moving JPEG decode to a Web Worker / `OffscreenCanvas` (would remove main-thread
  decode entirely; orthogonal to transport).
- Replacing JPEG with raw BGRA over the same Channel (the original "bandwidth wall"
  the team rejected — `docs/logs/2026-06-13.md` line ~131).
- HiDPI / multi-display correctness and region selection on the native path (also
  deferred in the same log entry).
- Channel callback unregister on stop (separate deferred item).

## References

- `src-tauri/src/screen_stream.rs` — `start_screen_stream` (line 116-231),
  `ScreenStreamFrame` (line 32-43), `encode_frame` (line 77-110), `RECV_TIMEOUT`
  (line 29), `JPEG_QUALITY` (line 26), `StreamHandle`/`ScreenStreamState` (line 45-74),
  drain-to-latest (line 167-169), base64 encode (line 185).
- `frontend/src/lib/native-screen.ts` — `startNativeScreenStream`,
  `stopNativeScreenStream`, the base64 `decode()` + latest-wins gate (line 49-75).
- `frontend/src/components/asmr-recorder/preview.tsx` — `startNativeScreen`
  (line 230-283), `Math.min(…, 30)` cap (line 261), `stopNativeStreamForSection`
  (line 220-226).
- `src-tauri/src/lib.rs` — `save_media_recording` raw-body precedent
  (`tauri::ipc::InvokeBody::Raw`, line 80-138); `generate_handler!` registration
  (line 179-202).
- `src-tauri/src/screen.rs` — `ScreenFrame::to_rgba`, `to_packed_bgra`, and fields
  `width`/`height`/`stride`/`timestamp`.
- Tauri `2.9.5/src/ipc/channel.rs` — `Channel<TSend>::send` (line 292),
  `InvokeResponseBody::Raw` delivery (inline `Uint8Array` < 1024B at line 163-166,
  else the fetch path at line 168-181); `ipc/mod.rs` — `InvokeResponseBody` enum
  (line 99-104) + `impl IpcResponse` (line 127). `@tauri-apps/api@2.9.1` `core.js`
  `Channel` (ordered ArrayBuffer delivery, line 74-128).
- Dev log: `docs/logs/2026-06-13.md` Part B + "Deferred" (line ~150-161) and the
  raw-bytes save migration ("Follow-up: replace base64 save IPC", line ~411-428).
  Libs: `image` (JPEG), mediabunny (trim), ScreenCaptureKit.
