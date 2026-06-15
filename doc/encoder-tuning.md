# Feature: Encoder tuning — AAC CBR enforcement, adaptive backpressure, keyframe-spacing validation

**Priority:** P3  ·  **Effort:** S–M  ·  **Risk:** low–med  ·  **Surface:** frontend (encoder)

## Problem / motivation

The live recording path (WebCodecs → H.264 → `mp4-muxer`) now derives its video bitrate from the
quality preset (commit `2d42ceb`), but three encoder knobs remain unverified or unrefined, and
all three are most exposed on the *long, quiet, low-motion* recordings ASMR produces:

1. **Audio is under-encoded.** WebKit's AAC encoder treats the requested 256 kbps as a loose VBR
   ceiling and encodes quiet content at ~80–130 kbps (observed). There is a `bitrateMode:
   "constant"` attempt with a fallback, but it is unverified whether CBR actually takes effect in
   WKWebView — if WebKit silently ignores it, we are shipping a placebo and should pivot to
   *measuring* and *documenting* the limit rather than promising a fix.
2. **Backpressure is a single fixed threshold** (`ENCODE_QUEUE_LIMIT = 4`) chosen without data,
   and the same number governs a 720p/30 and a 4K/30 encode despite wildly different per-frame
   cost. We should validate it against long recordings and consider scaling by resolution/fps.
3. **Keyframe spacing went 1s → 3s** (also `2d42ceb`) to cut bitrate on slow content, but a wider
   GOP directly worsens trim-editor cut granularity — lossless cuts can only start on a keyframe.
   The 3s GOP needs validation against the trim editor's keyframe-snap on long recordings.

This is a measurement-and-tuning plan, not a feature plan. Be honest where the platform wins.

## Current state (grounded)

- **Video bitrate wired to quality (done).** `frontend/src/types/recording.ts`
  - `VideoQuality` (line 7) and `VIDEO_QUALITY_BITRATES` (lines 9-13): `low 4M / medium 8M /
    high 16M`. `defaultExternalRecordingConfig.videoQuality = "medium"` (line 113).
  - `recording-canvas.tsx`: `videoBitrate = VIDEO_QUALITY_BITRATES[videoQuality]` (line 223),
    fed to `encoder.configure({ … bitrate: videoBitrate … })` (line 940) and to the
    MediaRecorder fallback `videoBitsPerSecond` (line 1013). The old hardcoded
    `VIDEO_BITRATE = 12_000_000` is gone. (Verified: a `medium` clip produced ~6.1 Mbps video —
    expected VBR undershoot on low-motion content; 256k-nominal AAC; 30fps.)
- **Keyframe spacing (done, needs validation).** `recording-canvas.tsx`
  - `KEYFRAME_INTERVAL_SECONDS = 3` (line 81; comment at 79-80 explicitly cites trim cuts).
  - Per-frame keyframe decision: `keyframeIntervalFrames = max(1, round(fps × 3))` (lines
    1089-1092); `isKeyFrame = frameCount % keyframeIntervalFrames === 0` (1093-1094);
    `encoder.encode(videoFrame, { keyFrame: isKeyFrame })` (1095). At 30fps that forces a
    keyframe every 90 frames; WebKit may also insert its own.
- **Audio constants + CBR attempt.** `recording-canvas.tsx`
  - Constants: `AUDIO_SAMPLE_RATE = 48000` (90), `AUDIO_NUM_CHANNELS = 2` (91),
    `AUDIO_BITRATE = 256_000` (92), `AAC_CODEC = "mp4a.40.2"` (93).
  - `AudioEncoderConfig baseAudioConfig` (lines 870-875): codec/channels/sampleRate/bitrate.
  - **CBR attempt + fallback** (lines 876-887): `configure({ …baseAudioConfig, bitrateMode:
    "constant" })` inside a `try`; on throw, warns and re-`configure`s with `baseAudioConfig`
    (default = VBR). The comment at 866-869 documents the VBR ceiling problem.
  - `startAudioProcessing` (lines 711-797): a `ScriptProcessorNode` (`bufferSize = 4096`, lines
    730-735) builds `f32-planar` `AudioData` (756-763) and calls `audioEncoder.encode` (764).
    No effective-mode read-back exists anywhere.
  - **Note (discrepancy from the brief):** the comment at lines 88-89 still says "256 kbps is
    negligible next to the **12 Mbps** video budget" — a stale reference to the removed
    `VIDEO_BITRATE = 12_000_000`. Minor, flag-only.
- **Backpressure.** `recording-canvas.tsx`
  - `ENCODE_QUEUE_LIMIT = 4` (line 85; rationale comment 82-84).
  - Drop check in `updateFrame` (lines 1066-1074): if `useWebCodecsRef.current && encoder &&
    encoder.encodeQueueSize > ENCODE_QUEUE_LIMIT`, increment `droppedFrameCountRef` and `return`
    *before* compositing. `lastFrameTimeRef` is still marked every tick (1059) so the watchdog
    (1208-1219) tracks loop liveness, not encode success — intentional drops don't trip it.
  - Drop count is logged every 30 frames (1102-1116). The fixed `4` does not scale with
    `recordingWidth/Height` (221-222) or `frameRate`.
- **No vitest coverage for any of the above** — these constants/branches live inside the
  WebCodecs component, none extracted as pure functions yet (cf. `[[test-harness-and-ci]]`).

## Proposed design

### 1. AAC CBR — verify, then measure-and-document (do not assume a fix)

**Measured result (2026-06-15):** `bitrateMode:"constant"` **appears to be honored** in
WKWebView. Packet-level ffprobe of a 35s recording (`recording_20260614_150600.mp4`, camera +
ambient audio):

```
packets: 1627   duration: 34.7s
size bytes:  min 441 / max 877 / mean 683 / stdev 25
bitrate kbps: min 165 / max 329 / mean 256 / stdev 9
coefficient of variation: 3.6%
```

3.6% CoV is consistent with CBR — a VBR encoder on varied content would show 50–100%+ CoV and
a mean well below 256 kbps (~80–130 kbps as originally predicted). Mean matches the
`AUDIO_BITRATE = 256_000` target exactly. **Caveat:** the test recording had audible content; a
near-silent recording would be the definitive test (VBR might still allocate 256 kbps for loud
content). For now, treat CBR as working.

Original baseline assessment (preserved for context): *WebKit's `AudioEncoder` very likely does
not honor `bitrateMode:"constant"` for AAC. The current code's own fallback masks this — if
WebKit silently accepts the config but ignores the mode (the common WebKit pattern), the `try`
never throws, we believe CBR is on, and the encoder keeps doing VBR.*

```ts
// After configure(), there is no standard "effective bitrateMode" read-back in WebCodecs.
// The only reliable signal is the OUTPUT: measure mean encoded audio bitrate over a window
// and compare to AUDIO_BITRATE. Do this from the existing output() callback (lines 837-860),
// where chunk.byteLength and chunk.timestamp are already in hand.
```

Plan, in priority order:
- **(a) Instrument.** In the `AudioEncoder.output` callback, accumulate `chunk.byteLength` and
  span `(lastTs − firstTs)`; every N seconds log effective kbps. This is the ground truth that
  tells us whether CBR took effect at all. Cheap, removable, no behavior change.
- **(b) If CBR is ignored (expected):** keep the `bitrateMode:"constant"` request (harmless, and
  correct on Chromium for the browser-fallback case), and **document the WKWebView limitation** in
  the code comment + this plan. Do **not** add speculative encoder fiddling.
- **(c) If we still want a fuller-bitrate track:** the only *controllable* lever is the signal,
  not the codec — VBR spends bits where there's energy, so **pre-amplify** quiet input via the
  existing mic `GainNode` (`micGainRef`, used at 644-649) so the encoder sees more energy and
  allocates more bits. This is a UX/loudness change, not free, and risks clipping — treat as an
  opt-in future (see Out of scope), not part of the core fix. Raising `AUDIO_BITRATE` does nothing
  under VBR for quiet content (the ceiling is never the binding constraint).

Bottom line: CBR appears honored (3.6% CoV, 256 kbps mean). Instrumentation is in place for
further monitoring. Pre-amplification is the only real knob if we ever want louder ASMR content.

### 2. Adaptive backpressure threshold

Replace the bare `ENCODE_QUEUE_LIMIT = 4` with a small pure helper so it (i) scales with encode
cost and (ii) becomes unit-testable:

```ts
// frontend/src/lib/encoder-tuning.ts (new) — pure, no WebCodecs refs
export function encodeQueueLimit(width: number, height: number, fps: number): number {
  // Higher pixel-throughput encodes finish slower per frame, so a deeper queue is
  // needed before dropping is the right call; very cheap encodes want a tight queue
  // so we drop early instead of buffering latency. Clamp to a sane band.
  const pixelsPerSec = width * height * fps;
  const hd1080_30 = 1920 * 1080 * 30;          // reference point
  const scaled = Math.round(4 * (pixelsPerSec / hd1080_30));
  return Math.min(8, Math.max(3, scaled));     // band [3, 8]
}
```

`updateFrame` calls it once at start-of-recording (memoized into a ref), not per frame. The band
keeps the change conservative: today's `4` is recovered at ~1080p/30, 720p/30 tightens toward 3,
4K/30 widens toward 8. **This is a tuning hypothesis to validate (step 3 below), not a proven
curve** — if measurement shows the fixed `4` is already fine across resolutions, ship the helper
returning a constant or skip it. Do not over-engineer.

### 3. Keyframe-spacing validation (measure first, change only if needed)

No code change is assumed. The question is empirical: **does a 3s GOP hurt the trim editor enough
to matter?** The trim editor does lossless packet-copy and can only start a segment on a keyframe
(see `[[trim-editor-v2-multisegment]]`), so the worst-case "slop" between the user's drag point
and the actual cut is one GOP (~3s at 30fps). Validate by:
- ffprobe the GOP structure of a long recording (keyframe interval, count, max GOP) and confirm
  it matches the 90-frame cadence (and note any extra WebKit-inserted IDRs).
- In the trim editor, confirm cut-snap lands within ≤3s and the bitrate win vs a 1s GOP is real.
- **If 3s slop is unacceptable**, the lever is a config: expose `KEYFRAME_INTERVAL_SECONDS` as a
  quality-linked value (e.g. high → 2s, medium → 3s) rather than a global retune. Out of scope to
  build now; capture the decision in `[[next-implementations]]`.

## Implementation steps

1. ✅ **Audio-bitrate instrumentation.** In the `AudioEncoder.output` callback (recording-canvas.tsx
   837-860) accumulate `byteLength`/timestamp span and log effective kbps every ~5s.
   **Shipped:** commit `4653cd7`. Console output (WebKit devtools only): `[WebCodecs] Audio
   effective bitrate: N kbps (target 256kbps, CBR attempted)`.
2. ✅ **Determine if CBR is honored.** ffprobe packet analysis of `recording_20260614_150600.mp4`:
   mean 256 kbps, CoV 3.6% → **CBR appears honored**. See measurement block above.
   Caveat: near-silent recording not yet tested; treat as likely-CBR until contradicted.
3. **Document the finding.** Update the comment at `recording-canvas.tsx` ~880 (the CBR attempt
   block) to note CBR appears honored at 3.6% CoV. The stale "12 Mbps" comment was already fixed
   in a prior session. **verify:** comment matches measured reality; `tsc -b` clean.
4. ✅ **Backpressure helper + tests.** `frontend/src/lib/encoder-tuning.ts` + `encoder-tuning.test.ts`
   shipped in commit `4653cd7`. 7 tests pass. `encodeQueueLimit` wired to `encodeQueueLimitRef`
   in `startRecording`; backpressure check uses the ref.
5. **Backpressure validation (long run).** Record ≥10 min at 1080p/30 and at 4K/30; inspect the
   periodic FPS/dropped log (1102-1116). **verify:** sustained fps ≈ target with bounded drops;
   no watchdog stalls (1213-1217); queue never grows unbounded.
6. **Keyframe validation (long run).** ffprobe GOP structure of a ≥10 min clip; open it in the
   trim editor and confirm keyframe-snap ≤3s and lossless cuts. **verify:** keyframe cadence ≈
   every 90 frames; trim cut-snap within one GOP; file remains seekable.
7. **Index.** Note findings + any deferred config (`KEYFRAME_INTERVAL_SECONDS` per-quality,
   pre-amplify) in `[[next-implementations]]`. **verify:** wiki-links resolve.

## Interface / API changes

- **New:** `frontend/src/lib/encoder-tuning.ts` (`encodeQueueLimit`) + `encoder-tuning.test.ts`.
- **`recording-canvas.tsx`:** `ENCODE_QUEUE_LIMIT` constant (85) replaced by a per-recording ref
  computed from `encodeQueueLimit(...)`; backpressure check (1066-1074) reads the ref. Audio
  `output` callback (837-860) gains removable instrumentation. Two comment edits (88-89, 866-869).
- **No type/config additions** in `recording.ts` for the core plan (keyframe-per-quality and
  pre-amplify are explicitly deferred). **No Rust/back-end change.**

## Edge cases & risks

- **CBR appears controlled (measured 2026-06-15).** 3.6% CoV on a 35s recording with audible
  content. Near-silent recording is the remaining edge case. If a quiet recording shows VBR
  behavior, revisit pre-amplification.
- **Pre-amplification clips.** Raising mic gain to feed the VBR encoder can distort quiet→loud
  transients; out of scope, and would need a limiter, not a raw gain bump.
- **`encodeQueueSize` semantics.** It counts frames *submitted but not yet emitted*; on a fast HW
  encoder it may sit near 0, so the adaptive band mostly matters under sustained overload (4K).
  Validate the helper doesn't make 720p drop *more* than the old `4` did.
- **Dropping early = lower effective fps**, not corruption — but too-aggressive dropping looks
  like stutter. Keep the lower band ≥3.
- **Wider GOP = coarser seeking** in any external player too, not just our trim editor; 3s is a
  defensible default but document the tradeoff.
- **WebKit-inserted keyframes.** WebKit may emit IDRs beyond our forced cadence (scene cuts);
  ffprobe is the only truth. Our forced 90-frame cadence is a *ceiling* on GOP length, not exact.
- **Browser fallback (MediaRecorder/WebM, 988-1044)** has no `encodeQueueSize` and no AAC config;
  all of this is WebCodecs-path only. Don't regress the fallback.

## Testing & verification

- **Pure (vitest, auto):** `encoder-tuning.test.ts` for `encodeQueueLimit` (reference point,
  band clamps, monotonicity). This is the only cleanly-pure piece; mirrors the
  `[[test-harness-and-ci]]` "test the math, not the WebCodecs glue" philosophy.
- **Build gates (auto):** `npx tsc -b`, `npx vite build`, `npm run test`.
- **Manual (dev, cannot auto-verify) — `npm run tauri dev` + ffprobe:**
  - Audio: `ffprobe -select_streams a -show_entries stream=bit_rate,codec_name,sample_rate,channels`
    on the saved MP4; cross-check against the in-app effective-kbps log; with and without the CBR
    request to settle whether it changes anything in WKWebView.
  - Backpressure: long recordings at 720p/1080p/4K; read the periodic `fps=/dropped=` log;
    confirm no watchdog stalls and a bounded queue.
  - Keyframes: ffprobe GOP structure (`-select_streams v -show_entries
    frame=pict_type,pts_time`); confirm ~90-frame cadence and trim-editor snap ≤3s.

## Out of scope / future

- **Guaranteeing 256 kbps AAC.** If WebKit won't do CBR, a fuller track requires either
  pre-amplification + limiter or a native Rust AAC encode — both far beyond this plan.
- **Pre-amplify / loudness normalization** of quiet ASMR input (needs a limiter; UX decision).
- **Per-quality `KEYFRAME_INTERVAL_SECONDS`** (e.g. high=2s) — a config addition, deferred until
  measurement proves 3s slop matters.
- **A real VBV/rate-control curve** for `encodeQueueLimit` (this plan ships a clamped linear
  hypothesis, validated empirically, not a tuned model).
- **AudioWorklet capture migration.** The `ScriptProcessorNode` in `startAudioProcessing`
  (711-797) is deprecated; `native-system-audio.ts` already proves AudioWorklet works in WKWebView
  with a ScriptProcessor fallback. Migrating the *encode* tap is its own change — see
  `[[audioworklet-migration]]`.
- **System-audio bitrate** specifics — gated on `[[native-system-audio-capture]]` landing fully.

## References

- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — `KEYFRAME_INTERVAL_SECONDS`
  (81) + keyframe decision (1089-1095); `ENCODE_QUEUE_LIMIT` (85) + backpressure check
  (1066-1074); audio constants (90-93) + stale "12 Mbps" comment (88-89); `baseAudioConfig` +
  CBR attempt/fallback (866-887); `AudioEncoder.output` callback (837-860); `startAudioProcessing`
  / `ScriptProcessorNode` (711-797); `videoBitrate` + `encoder.configure` (223, 935-943);
  FPS/dropped log (1102-1116); watchdog (1208-1219).
- `frontend/src/types/recording.ts` — `VideoQuality` (7), `VIDEO_QUALITY_BITRATES` (9-13),
  `defaultExternalRecordingConfig` (107-121).
- `frontend/src/lib/native-system-audio.ts` — AudioWorklet-with-ScriptProcessor-fallback proof
  (84-166) informing `[[audioworklet-migration]]`.
- Related: `[[trim-editor-v2-multisegment]]` (keyframe-snap / lossless cut granularity),
  `[[test-harness-and-ci]]` (pure-test pattern + CI gates), `[[native-system-audio-capture]]`,
  `[[audioworklet-migration]]` (forward ref), `[[next-implementations]]` (roadmap index).
