# Feature: Trim editor v2 — multi-segment cuts, frame-accurate option, thumbnail scrub

**Priority:** P2  ·  **Effort:** L  ·  **Risk:** med  ·  **Surface:** frontend

## Problem / motivation
Today the trim editor only does a **single** lossless `[inSec, outSec]` packet copy with the
in-point snapped *down* to a keyframe. That covers "drop the head and tail" but not the common
"remove a chunk in the middle" (a fumble, a pause, a doorbell mid-ASMR). Two further gaps:

- **Keyframe-snapped in-point.** `handleValueCommit` and `trimToBuffer` both call
  `getKeyPacket(a, { verifyKeyPackets: true })`, so the cut starts on the keyframe at/before the
  chosen time. With `KEYFRAME_INTERVAL_SECONDS = 1` (`recording-canvas.tsx:61`) the in-point can
  land up to ~1s *before* where the user dragged. The dev log calls this out as the defining
  limitation of the lossless copy ("In-point lands at ~0.01s… correct, not a bug" / "the start
  snaps to the nearest keyframe", `docs/logs/2026-06-13.md`, "Trim editor GUI verified live"). The
  `[[trim-editor-lossless-packet-copy]]` note records the same.
- **No scrub thumbnails.** The only visual is the live `<video>` preview seeked on drag
  (`handleValueChange`). Finding a cut point means scrubbing the whole clip in the player.

This is a UX/capability upgrade on an already-verified path, hence **P2** — it must not regress
the lossless single-cut flow that passed live verification (two record→trim cycles, both
`key_frame=1` at frame 0, `docs/logs/2026-06-13.md`).

## Current state (grounded)
All in `frontend/src/components/asmr-recorder/trim-editor.tsx`:

- **`trimToBuffer(input, inSec, outSec)`** — builds an `Output` (`Mp4OutputFormat({ fastStart:
  "in-memory" })`, `BufferTarget`), adds `EncodedVideoPacketSource(videoTrack.codec)` +
  (optional) `EncodedAudioPacketSource(audioCodec)`. Reads `videoSink.getKeyPacket(inSec,
  { verifyKeyPackets: true })`, then computes `origin = startKey.timestamp` and lowers it to
  `min(origin, audioStart.timestamp)` (where `audioStart = audioSink.getPacket(startKey.timestamp)`),
  then iterates `videoSink.packets(startKey)` / `audioSink.packets(audioStart)` re-emitting each
  `EncodedPacket` with `timestamp - origin`, breaking when `p.timestamp > outSec`. First packet of
  each track carries `decoderConfig` from `track.getDecoderConfig()`. Returns
  `output.target.buffer`.
- **State:** `inPoint`/`outPoint` (numbers), `duration`, `prevValRef`, `outTouchedRef`. Single
  radix `SliderPrimitive.Root` with `value={[inPoint, outPoint]}`, `onValueChange=
  handleValueChange`, `onValueCommit=handleValueCommit`, two thumbs.
- **`handleValueChange`** seeks the `<video>` to the moved thumb. **`handleValueCommit`** snaps
  the in-point via `videoSinkRef.current.getKeyPacket(a, { verifyKeyPackets: true })`.
- **`handleExport`** calls `trimToBuffer(input, inPoint, outPoint)` then
  `invoke<string>("save_media_recording", buffer)` (raw `ArrayBuffer` body — see
  `[[trim-editor-lossless-packet-copy]]`; not base64).
- **Inputs/refs:** `inputRef` (`new Input({ formats: ALL_FORMATS, source: new BlobSource(blob)
  })`), `videoSinkRef` (`EncodedPacketSink`). Blob arrives via the `recordingReadyForEdit`
  CustomEvent dispatched in `recording-canvas.tsx:1134`; the editor is mounted once in
  `frontend/src/components/asmr-recorder.tsx`.

Codec facts: recording is `avc1.42001f` = **H.264 Constrained Baseline, no B-frames** (top
comment on `trimToBuffer`; `webcodecs-implementation.md`). No B-frames ⇒ decode order ==
presentation order, so re-encoding a leading run of frames and concatenating with copied
packets is safe (no reorder gymnastics).

Mediabunny **1.46.0** is installed. Confirmed exports for the new work (verified against
`node_modules/mediabunny/dist/mediabunny.d.ts`): `VideoSampleSink` (`.samples(start, end)`,
`.getSample(t)`), `VideoSampleSource` (`constructor(VideoEncodingConfig)`, `.add(sample,
encodeOptions?: VideoEncoderEncodeOptions)`), `VideoSample` (`implements Disposable`), `CanvasSink`
(`.canvasesAtTimestamps(timestamps)` → yields `WrappedCanvas | null`), `canEncodeVideo(codec,
options?)`, and `VideoEncodingConfig { codec, bitrate, keyFrameInterval? }`. The track-level bitrate
is read from `InputVideoTrack.getAverageBitrate()` / `getBitrate()` (each
`Promise<number | null>`), **not** from `getDecoderConfig()` (a `VideoDecoderConfig` carries no
bitrate).

## Proposed design
Three independent layers; ship in order, each behind its own verify.

**(1) Multi-segment model.** Replace the two scalars with a list of *keep* segments. Internal
type:

```ts
type Segment = { in: number; out: number };   // seconds, in < out, sorted, non-overlapping
```

State `segments: Segment[]` (default `[{ in: 0, out: duration }]`). "Remove the middle" =
splitting one keep-segment into two at a remove range. Export concatenates each segment with a
**single shared timeline** and a **monotonically rebased timeline** so the output plays as one
continuous clip:

```ts
// before: trimToBuffer(input, inSec, outSec)
// after:
async function trimToBuffer(
  input: Input,
  segments: Segment[],
  frameAccurate: boolean,
): Promise<ArrayBuffer>
```

For each segment we keep the existing copy logic but track a running `timelineOffset` (sum of
prior segments' emitted durations) so segment *k*'s packets are written at
`(p.timestamp - segOrigin_k) + timelineOffset`, where `segOrigin_k` is the per-segment
`min(videoKeyframe, audioStart)` already used today. Audio is copied per segment the same way.
This generalizes the current single `origin` rebasing to N segments.

**(2) Optional frame-accurate in-points.** When `frameAccurate` is on, for each segment whose
chosen `in` is **after** its snapped keyframe, re-encode only the partial leading GOP:

- `kp = getKeyPacket(seg.in, { verifyKeyPackets: true })`; `firstKept` = the exact frame
  at/after `seg.in` (the requested in-point).
- Decode frames in `[kp.timestamp, firstKept)` with `VideoSampleSink.samples(kp.timestamp,
  firstKept)` — these are the frames we will *drop*. Decode `[firstKept, gopEnd)` (gopEnd = next
  keyframe timestamp) and **re-encode** them via `VideoSampleSource` configured to match the
  source (`codec: videoTrack.codec`, `bitrate` from `videoTrack.getAverageBitrate()` with a
  sane fallback, `keyFrameInterval` large so the run starts on a forced keyframe). Then
  **packet-copy** the remainder of the segment from `gopEnd` onward (the original tail keyframes
  are intact). Because there are **no B-frames**, the re-encoded run and the copied run
  concatenate cleanly.
- Guard with `await canEncodeVideo(videoTrack.codec, …)`; if false, fall back to the snapped
  lossless cut and surface a toast. Audio stays packet-copied (AAC frames are ~21ms, audible
  snap is sub-frame; do **not** re-encode audio in v1).

This is the only mode that touches `VideoDecoder`/`VideoEncoder`, so it reintroduces a WebCodecs
hardware dependency the lossless path deliberately avoids ([[trim-editor-lossless-packet-copy]]).
Hence it is **opt-in** (a checkbox), default off.

**(3) Thumbnail filmstrip.** A `CanvasSink(videoTrack, { width: 160, poolSize: 0 })`; request
~12 evenly spaced timestamps via `canvasesAtTimestamps(...)` once per loaded clip; render the
`WrappedCanvas` list as a strip under the slider, skipping any `null` the iterator yields.
Clicking a thumb seeks the `<video>`. This is read-only and fully decoupled from export.

## Implementation steps
1. **Segment state + UI scaffolding** (`trim-editor.tsx`). Add `segments` state and helpers
   `addRemoveRange(start,end)` / `removeSegmentAt(i)` that split/merge keep-segments. Keep the
   existing dual-handle slider as the active-segment editor; render *other* segments as inert
   colored bands on the track. Add a "Cut out selection" button that splits the segment under the
   current `[in,out]`. **verify:** in the GUI, cut a middle range → two keep bands shown; exported
   file (ffprobe in `test-results/`) has `format=duration` == sum of kept ranges (±1 frame).
2. **Generalize `trimToBuffer` to N segments** (`trim-editor.tsx`). Loop segments, carry
   `timelineOffset`, emit rebased packets per segment for video and audio. Single-segment input
   must produce byte-for-byte the same behavior as today. **verify:** with one full-range segment,
   the output still has `key_frame=1` at frame 0 and `nb_frames`/duration matches the current
   build for the same recording (regression check against the verified path).
3. **Frame-accurate re-encode helper** (new internal `reencodeLeadingGop`). Wire `frameAccurate`
   flag + `canEncodeVideo` guard + fallback. **verify:** `ffprobe -show_frames` on a frame-accurate
   cut: frame 0 is a keyframe, the re-encoded run has `firstKept - kp.timestamp` *fewer* frames
   than the lossless cut would contain, and the first presented PTS equals the requested in-point
   within one frame interval (~33ms at 30fps). Backend `println!` "Saved MP4" confirms the write.
4. **Thumbnail filmstrip** (`trim-editor.tsx`). Build the `CanvasSink` in the existing
   blob→`Input` effect (the same effect that sets `videoSinkRef`); fetch thumbnails once on load,
   store in state, render strip; click-to-seek. **verify:** GUI shows ~12 thumbnails matching the
   clip; clicking thumb N seeks the preview to ≈ N/12 of the duration (visual).
5. **Export wiring** (`handleExport`). Pass `segments` + `frameAccurate` to `trimToBuffer`;
   disable Save when total kept duration ≤ 0 (generalizes the current `trimmedDuration <= 0`
   guard). **verify:** full A/V decode of the output exits 0 under `ffmpeg -i out.mp4 -f null -`;
   container DTS clean under `ffmpeg -c copy -f null -` (per the DTS-measurement lesson in
   `docs/logs/2026-06-13.md`).

## Interface / API changes
- **TS (internal only):**
  - `type Segment = { in: number; out: number }`.
  - `trimToBuffer(input: Input, inSec: number, outSec: number)` → `trimToBuffer(input: Input,
    segments: Segment[], frameAccurate: boolean)`.
- **No Rust/IPC change.** Export still calls `invoke<string>("save_media_recording", buffer)`
  with a raw `ArrayBuffer`; `save_media_recording` in `src-tauri/src/lib.rs` (reads
  `tauri::ipc::InvokeBody::Raw`, sniffs EBML `1A 45 DF A3` / `ftyp` at offset 4) is unchanged.
- **No change** to the `recordingReadyForEdit` CustomEvent contract (`recording-canvas.tsx`).

## Edge cases & risks
- **Segment boundary across a keyframe (lossless mode).** Each segment's `in` still snaps down to
  its own keyframe in lossless mode — preserve the current snap-and-show behavior per segment so
  the preview matches the export.
- **Monotonic timestamps across joins.** Re-basing N segments must keep emitted video/audio PTS
  strictly increasing and never negative (the muxer rejects negative ts — existing `origin`
  comment). A boundary where audio of segment *k* overruns the next segment's video start must be
  handled via the per-segment `min(video, audio)` origin already used.
- **Frame-accurate = WebCodecs dependency.** Decode/encode can fail or be unavailable
  (`canEncodeVideo` false) on a webview without the H.264 encoder; must fall back to lossless +
  toast, never throw silently. Re-encoded GOP quality differs slightly from the copied tail —
  acceptable, but bitrate must be matched (`getAverageBitrate()`) to avoid a visible quality step
  at the join.
- **VideoSample/VideoFrame leaks.** `VideoSample implements Disposable`; every decoded sample from
  `VideoSampleSink.samples(...)` must be closed after `VideoSampleSource.add(...)` (or after being
  identified as a dropped leading frame) — WKWebView is unforgiving about un-closed `VideoFrame`s
  (the recorder already closes frames per-tick, `recording-canvas.tsx`).
- **Thumbnail decode cost / nulls.** `canvasesAtTimestamps` decodes ~12 frames on the main thread
  and may yield `null` for a timestamp with no available frame; cap count and width (160px), skip
  nulls, and only run once per loaded blob to avoid janking the dialog open.
- **Don't regress the verified path.** Step 2's single-segment equivalence check is the guardrail;
  the live-verified lossless cut (`key_frame=1` at frame 0) must still hold.

## Testing & verification
Hybrid, per house setup: launch with `npm run tauri dev` (default features — the ffmpeg cargo
feature fails to link on this machine and the trim path is pure frontend WebCodecs + the save
command; `docs/logs/2026-06-13.md`). User performs the GUI/TCC steps (native screen capture,
record, set cuts, toggle frame-accurate, Save); the agent captures backend `println!` ("Saved
MP4: …test-results/…") and runs `ffprobe`/`ffmpeg` on the output:
- `ffprobe -show_frames -select_streams v out.mp4` → first frame `key_frame=1`; PTS of first
  kept frame ≈ requested in-point (within one frame interval for frame-accurate mode).
- `ffprobe -show_format out.mp4` → `duration` == sum of kept ranges.
- `ffmpeg -i out.mp4 -f null -` exits 0 (full decode); `ffmpeg -c copy -f null -` → no real DTS
  warnings (the plain `-f null` CFR re-derivation warnings are a known artifact).
- A/V sync at a mid-clip join: spot-check audio doesn't drift after the removed segment.

**Cannot be auto-verified:** the dialog/slider/thumbnail UI and the frame-accurate visual quality
of the re-encoded join — macOS WKWebView can't be driven programmatically and the console isn't
piped to the terminal. These need the user's interactive pass.

## Out of scope / future
- Re-encoding audio for frame-accurate audio in-points (AAC ~21ms snap is inaudible; skip).
- Full transcode / resolution or bitrate change on export.
- Migrating the recorder itself off the deprecated `mp4-muxer` to Mediabunny
  ([[mp4-muxer-deprecated-mediabunny]]) — separate effort.
- Persisting an edit decision list / non-destructive project file.
- Web Worker offload for thumbnail/GOP decode (only if main-thread jank shows up).

## References
- `frontend/src/components/asmr-recorder/trim-editor.tsx` — `trimToBuffer`, `handleValueChange`,
  `handleValueCommit`, `handleExport`, the radix dual-handle slider, `adoptDuration`.
- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — `KEYFRAME_INTERVAL_SECONDS`
  (line 61), `recordingReadyForEdit` dispatch (line 1134).
- `frontend/src/components/asmr-recorder.tsx` — mounts `<TrimEditor />`.
- `src-tauri/src/lib.rs` — `save_media_recording(request: tauri::ipc::Request<'_>)`
  (raw `InvokeBody::Raw`, EBML/`ftyp` magic-byte sniff).
- Mediabunny **1.46.0** (`node_modules/mediabunny/dist/mediabunny.d.ts`): `EncodedPacketSink`,
  `EncodedVideoPacketSource`/`EncodedAudioPacketSource`, `VideoSampleSink`/`VideoSampleSource`,
  `VideoSample`, `CanvasSink`, `canEncodeVideo`, `VideoEncodingConfig`,
  `InputVideoTrack.getAverageBitrate()`. MPL-2.0; supersedes `mp4-muxer@5.2.2`.
- Dev log: `docs/logs/2026-06-13.md` — "Simple cut/trim editor", "replace base64 save IPC with
  raw bytes", "Trim editor GUI verified live", and the DTS-measurement correction.
- Memory: `[[trim-editor-lossless-packet-copy]]`, `[[mp4-muxer-deprecated-mediabunny]]`.
