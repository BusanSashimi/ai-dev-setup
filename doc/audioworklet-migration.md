# Feature: Migrate recording mic-processing off the deprecated `ScriptProcessorNode` to `AudioWorkletNode`

**Priority:** P3  ·  **Effort:** M  ·  **Risk:** med  ·  **Surface:** frontend (audio)

## Problem / motivation

The live recording audio path captures PCM with a **`ScriptProcessorNode`** — deprecated in
the Web Audio spec and removed-on-notice in every engine. It was chosen deliberately for
WKWebView compatibility: `recording-canvas.tsx:708-709` documents "Uses ScriptProcessorNode
instead of MediaStreamTrackProcessor for WebKit/WKWebView compatibility (Tauri on macOS)."

`ScriptProcessorNode` runs its `onaudioprocess` callback on the **main thread**, so every audio
block competes with React renders, the compositing `requestAnimationFrame` loop
(`compositeFrame`, line 434), and the WebCodecs video encode for the same thread. Under load
that produces audio glitches/gaps and adds latency — exactly the failure mode an ASMR recorder
must avoid. `AudioWorkletNode` moves the same work to a dedicated audio-render thread.

The migration is **low-risk to attempt** because the proven pattern already lives in this
repo: `native-system-audio.ts` ships an "AudioWorklet with `ScriptProcessorNode` fallback"
(`buildAudioSource`, line 79) that registers an inline processor via a Blob URL and probes
`audioCtx.audioWorklet` at runtime (line 85). That file is both the **reference implementation**
and the place that already had to answer "does this WKWebView support AudioWorklet?" — its
runtime `console.log` ("AudioWorklet source created" vs "ScriptProcessorNode source created",
lines 106/166) is the empirical answer this plan must read before committing to removal.

The asymmetry to respect: the system-audio worklet is a **source** (0 inputs, generates audio
from a ring buffer); the recording path is the **opposite** — an **input processor** that
consumes a `MediaStreamTrack` and reads `inputBuffer` to feed the encoder. We reuse the
*pattern* (Blob-URL registration, runtime probe, fallback), not the processor body.

## Current state (grounded)

**Recording mic/mix processing — `frontend/src/components/asmr-recorder/recording-canvas.tsx`:**
- `startAudioProcessing(audioTrack, audioEncoder)` (line 711-797): the function to migrate.
  - `const channelCount = AUDIO_NUM_CHANNELS;` (line 717) — forces stereo so
    `inputBuffer.numberOfChannels` matches the encoder/muxer (mono mics upmixed).
  - `ensureAudioContext()` (line 724) returns the live recording context whose **actual**
    `sampleRate` already drives encoder + muxer (WebKit may clamp 48k → hardware rate; see
    line 687-702). The migration must NOT create or re-clamp a context.
  - `audioCtx.createScriptProcessor(4096, channelCount, channelCount)` (line 731-735).
  - `scriptNode.onaudioprocess` (line 739-771): guards `audioEncoder.state !== "configured"`
    (740); builds **planar** `Float32Array(numFrames * numChannels)` by `getChannelData(ch)`
    (747-750); computes `timestampUs = (sampleOffset / inputBuffer.sampleRate) * 1e6` (752-753);
    constructs `new AudioData({ format: "f32-planar", sampleRate: inputBuffer.sampleRate,
    numberOfFrames, numberOfChannels, timestamp, data })` (756-763); `audioEncoder.encode` +
    `audioData.close()` (764-765); advances `sampleOffset` (770).
  - Graph: `source -> scriptNode -> silentGain(0) -> audioCtx.destination` (778-780); the silent
    sink is required for `onaudioprocess` to fire in WebKit.
  - `audioProcessingCleanupRef.current` set to null the handler + disconnect all three nodes
    (782-787). Teardown is called at stop (line 1242-1245).
- Audio constants: `AUDIO_SAMPLE_RATE = 48000` (line 90), `AUDIO_NUM_CHANNELS = 2` (line 91).
- Mix graph (`acquireAudioTrack`, 542-684): the multi-stream mixing `AudioContext`
  (`new AudioContext({ sampleRate: AUDIO_SAMPLE_RATE })`, 628) and `createMediaStreamDestination()`
  (630); mic chain `source -> hpf -> micGainNodeRef -> destination` (636-650); native system
  audio feeds `systemAudioGainNodeRef -> destination` (661-674). `startAudioProcessing` runs on
  the **mixed destination track** returned from here, not on raw mic — so it processes whatever
  the mix produced. Live gain control reads these refs (line 289-294).
- Invoked from `initializeWebCodecs` (line 947), after the `AudioEncoder` is configured.

**Reference pattern — `frontend/src/lib/native-system-audio.ts` (already shipped):**
- `WORKLET_PROCESSOR_NAME = "ring-buffer-source"` (line 24); inline `WORKLET_CODE` string with
  `class RingBufferSourceProcessor extends AudioWorkletProcessor` + `registerProcessor(...)`
  (line 30-64). This processor has **0 inputs** — a source, not an input processor.
- `buildAudioSource(audioCtx, numChannels, destination)` (line 79-186):
  - **Runtime probe + try:** `if (audioCtx.audioWorklet) { try { ... } catch { warn; fall through } }`
    (line 85-117).
  - **Blob-URL registration:** `new Blob([WORKLET_CODE], { type: "application/javascript" })`,
    `URL.createObjectURL`, `await audioCtx.audioWorklet.addModule(url)`, `revokeObjectURL`
    (line 87-90) — the workaround for bundler/CSP issues with a separate worklet file.
  - `new AudioWorkletNode(audioCtx, WORKLET_PROCESSOR_NAME, { numberOfInputs: 0,
    numberOfOutputs: 1, outputChannelCount: [numChannels] })` (line 92-96).
  - **Fallback:** `audioCtx.createScriptProcessor(4096, 0, numChannels)` ring-buffer (line 125).
  - Both branches return `{ pushChunk, disconnect }` and log which one was created (106/166).
- `audio-wire.ts` is the shared wire-format codec (pure, unit-tested in `audio-wire.test.ts`).

**Test harness — `frontend/vite.config.ts`:** vitest `environment: 'node'`,
`include: ['src/**/*.test.ts']`. There is **no DOM / AudioContext** in tests, so existing tests
cover only pure functions (`audio-wire.test.ts`, `layouts.test.ts`, `screen-wire.test.ts`,
`trim-segments.test.ts`). Any audio-node code is verified by build + manual recording, per
`[[test-harness-and-ci]]`.

## Proposed design

### 1. Decide the strategy by reading the runtime probe — do NOT assume

The system-audio path already runs `buildAudioSource` every recording with system audio. Before
touching the recording path, **observe** which branch WKWebView takes in `npm run tauri dev`:
the console prints `AudioWorklet source created` or `ScriptProcessorNode source created`
(native-system-audio.ts:106/166). That single observation gates everything:

- If WKWebView reliably supports AudioWorklet → migrate the recording path to a worklet, keep
  the `ScriptProcessorNode` only as a guarded fallback.
- If support is **absent or flaky** → do nothing to the live path; the deprecation is cosmetic
  and the proven primitive stays. (Record the finding in `docs/logs/`.)

This plan assumes support is *probably* present (the system-audio worklet would otherwise be
dead code) but **treats the fallback as permanent** — see Edge cases.

### 2. New inline worklet: a mic-input recorder processor (NOT the source processor)

The recording processor consumes input and posts planar PCM frames to the main thread, where
the existing `AudioData`/encoder code runs unchanged. Add a sibling inline module string in
`recording-canvas.tsx` (or a small `frontend/src/lib/audio-encode-worklet.ts` that exports the
string + name, mirroring how `WORKLET_CODE` is colocated):

```ts
const ENCODE_WORKLET_NAME = "encode-source";
const ENCODE_WORKLET_CODE = `
class EncodeSourceProcessor extends AudioWorkletProcessor {
  process(inputs) {
    const input = inputs[0];                 // one input, N channels
    if (input && input.length && input[0].length) {
      // Copy: the input arrays are reused across process() calls.
      const frames = input.map((ch) => ch.slice());
      this.port.postMessage(frames);          // structured clone of Float32Array[]
    }
    return true;                              // keep alive even with no consumers
  }
}
registerProcessor('encode-source', EncodeSourceProcessor);
`;
```

Notes that make this safe:
- **Separate thread, no closures.** The processor body can reference NOTHING from React/module
  scope — `audioEncoder`, `sampleOffset`, constants are all unavailable inside it. It only ships
  PCM out via `port.postMessage`; the encoder stays on the main thread.
- **Block size is 128 frames** (Web Audio render quantum), not 4096. That is ~128 messages/sec
  per the audio thread; structured-cloning two 128-sample `Float32Array`s per block is cheap and
  far below the encoder cadence. (Optionally batch ~32 quanta in the processor to post ~4096-frame
  chunks and match today's granularity — a minor optimization, not required.)
- **Copy on send** (`ch.slice()`): AudioWorklet reuses input buffers across calls, so we must
  copy before transferring ownership semantics via `postMessage` (structured clone copies anyway;
  the `.slice()` guards against aliasing if we later switch to transferables).

### 3. Migrated `startAudioProcessing` — worklet path with ScriptProcessor fallback

Mirror `buildAudioSource`'s shape exactly: probe → try worklet (Blob URL + `addModule`) →
`catch`/absent → ScriptProcessor. Keep the **encoder-feeding logic in one helper** so both
branches call it identically:

```ts
const audioCtx = ensureAudioContext();                       // unchanged: drives rate
const source = audioCtx.createMediaStreamSource(new MediaStream([audioTrack]));
let sampleOffset = 0;

// `emit()` is today's lines 743-770 unchanged EXCEPT it reads audioCtx.sampleRate
// (no inputBuffer on the worklet) and takes planar Float32Array[] as its arg:
//   guard state -> pack planarData -> new AudioData({f32-planar, audioCtx.sampleRate,
//   timestamp = sampleOffset/audioCtx.sampleRate*1e6}) -> encode/close -> sampleOffset += n
const emit = (planarChannels: Float32Array[]) => { /* see §4 contract */ };

if (audioCtx.audioWorklet) {
  try {
    const url = URL.createObjectURL(new Blob([ENCODE_WORKLET_CODE], { type: "application/javascript" }));
    await audioCtx.audioWorklet.addModule(url); URL.revokeObjectURL(url);
    const node = new AudioWorkletNode(audioCtx, ENCODE_WORKLET_NAME, {
      numberOfInputs: 1, numberOfOutputs: 1, outputChannelCount: [AUDIO_NUM_CHANNELS],
    });
    node.port.onmessage = ({ data }) => emit(data as Float32Array[]);
    source.connect(node);
    const silentGain = audioCtx.createGain(); silentGain.gain.value = 0;
    node.connect(silentGain); silentGain.connect(audioCtx.destination);  // keep graph rendering
    audioProcessingCleanupRef.current = () => {
      node.port.onmessage = null; source.disconnect(); node.disconnect(); silentGain.disconnect();
    };
    return;
  } catch (e) { console.warn("[Audio] AudioWorklet unavailable, using ScriptProcessorNode:", e); }
}
// Fallback: today's createScriptProcessor(4096, ch, ch) path, calling emit() from onaudioprocess.
```

Caveat: `startAudioProcessing` is currently **synchronous** and called from the synchronous
`initializeWebCodecs` (line 803, returns `boolean`). `audioWorklet.addModule` is **async**.
Resolve by (a) keeping the function synchronous and fire-and-forgetting the worklet setup
(`void setupWorkletPath().catch(fallbackToScriptProcessor)`), which preserves
`initializeWebCodecs`'s `boolean` return, OR (b) pre-registering the module once at context
creation. **(a) is the minimal change**; a few audio blocks may be dropped during the ~1-frame
registration window (inaudible at record start). Do not convert `initializeWebCodecs` to async —
that ripples into its `start()` caller.

### 4. The `f32-planar` contract is preserved end to end

The encoder/muxer config is untouched: `AUDIO_NUM_CHANNELS`, `audioCtx.sampleRate`, `f32-planar`,
the `extractAudioSpecificConfig` esds fix, and the silent-gain sink all stay. The ONLY change is
*where* the planar frames originate (worklet thread vs main-thread `onaudioprocess`).

## Implementation steps

1. **Observe the runtime probe (gate).** `npm run tauri dev`, record with system audio, read the
   console for `native-system-audio` "AudioWorklet source created" vs "ScriptProcessorNode source
   created". Log the result in `docs/logs/`. **verify:** the branch WKWebView takes is recorded;
   if it's ScriptProcessor, STOP (deprecation is cosmetic) and note it in `[[next-implementations]]`.
2. **Add the encode worklet module.** Add `ENCODE_WORKLET_NAME` + `ENCODE_WORKLET_CODE` (input
   processor that posts planar `Float32Array[]`), colocated with `startAudioProcessing` or in a
   small `audio-encode-worklet.ts`. **verify:** `npx tsc -b` clean (the string isn't type-checked,
   but exports are).
3. **Extract the `emit()` helper.** Factor today's planar-build + `AudioData` + `encode` +
   `sampleOffset` logic (lines 743-770) into one closure both node paths call. **verify:**
   ScriptProcessor path still records identically (no behavior change yet) in a dev recording.
4. **Add the worklet branch with fallback.** Implement the probe→worklet→catch→ScriptProcessor
   structure from §3; reuse the silent-gain sink and `audioProcessingCleanupRef` teardown shape.
   Keep `startAudioProcessing` synchronous (fire-and-forget worklet setup). **verify:** `tsc -b` +
   `npx vite build` clean.
5. **Dev recording on the worklet path.** Record 30-60s mic-only and mic+system; confirm the
   console logs the worklet branch, the saved MP4's AAC track plays in sync, no dropouts, gain
   sliders (`micGainNodeRef`/`systemAudioGainNodeRef`) still respond live. **verify:** audible
   audio correct + in sync; `ffprobe -show_streams` shows one AAC track at `audioCtx.sampleRate`.
6. **Decide fallback fate + docs.** Keep `ScriptProcessorNode` as the permanent guarded fallback
   (do NOT delete it — see Edge cases). Move the `[[next-implementations]]` backlog entry to
   "done"/landed; note the gate outcome. **verify:** links resolve; backlog updated.

## Interface / API changes

- **`recording-canvas.tsx`:** `startAudioProcessing(audioTrack, audioEncoder)` keeps its
  signature and remains synchronous; internally gains the worklet branch + `emit()` helper. New
  module-level `ENCODE_WORKLET_NAME` / `ENCODE_WORKLET_CODE` (or imported from a new
  `frontend/src/lib/audio-encode-worklet.ts`).
- **No change** to: `acquireAudioTrack`, the mix graph, gain-node refs, `ensureAudioContext`,
  `initializeWebCodecs`'s signature, the `AudioEncoder`/muxer config, `save_media_recording`, or
  any Rust. `native-system-audio.ts` is **unchanged** (it's the reference, not the target).

## Edge cases & risks

- **Fallback is permanent, not transitional.** WKWebView's AudioWorklet support is not
  contractually guaranteed and can regress across macOS/WebKit point releases. Even after a
  successful migration, `ScriptProcessorNode` must stay as the `catch`/absent branch — the system
  audio path keeps it for the same reason. **Do not remove `createScriptProcessor` from the
  recording path.**
- **Separate-thread isolation.** The processor body cannot close over `audioEncoder`,
  `sampleOffset`, or any module symbol. All state that must persist (encoder, offset) stays on the
  main thread; only PCM crosses the port. A reviewer should confirm the worklet string references
  nothing external.
- **Async registration vs sync caller.** `addModule` is async; `initializeWebCodecs` is sync and
  returns `boolean`. Fire-and-forget the worklet setup so the caller is untouched; tolerate a tiny
  start-of-record gap during registration. Failing to handle this would force an async ripple up
  to `start()`.
- **Render quantum (128) vs 4096.** Worklet `process()` fires per 128-frame quantum — ~375 Hz of
  small messages vs ~12 Hz today. Cheap, but if profiling shows overhead, batch quanta in the
  processor to ~4096 before posting. Timestamps use `sampleOffset` accumulation, so block size
  doesn't affect A/V sync.
- **Sample-rate coherence.** `emit()` must use `audioCtx.sampleRate` (the clamped, actual rate the
  encoder + muxer were configured with), NOT `AUDIO_SAMPLE_RATE`. Today's code reads
  `inputBuffer.sampleRate`; on the worklet there is no `inputBuffer`, so read `audioCtx.sampleRate`
  (equal to the worklet's global `sampleRate`). A mismatch yields pitch-skew (the same failure
  `[[native-system-audio-capture]]` warns about).
- **Buffer aliasing.** Worklet input arrays are reused; copy (`ch.slice()`) before posting.
- **CSP / bundler.** Blob-URL `addModule` sidesteps needing a separately-served worklet file;
  it's the proven approach (`native-system-audio.ts:87-90`). A future strict CSP could block
  `blob:` script URLs — if so the fallback covers it, and a bundled worklet asset is the escalation.
- **Encoder backpressure.** `emit()` keeps the `audioEncoder.state !== "configured"` guard so
  frames during teardown/closed states are dropped, identical to today.

## Testing & verification

- **Build gates (auto):** `npx tsc -b`, `npx vite build`, `npm run test` (existing pure tests
  must stay green; nothing new to unit-test in node — there is no AudioContext in the `node`
  vitest env, per `[[test-harness-and-ci]]`).
- **Unit-testable seam (optional):** if `emit()`'s planar-pack (interleave-free
  `Float32Array[] → Float32Array(numFrames*numChannels)`) is extracted as a pure function, add a
  tiny test asserting channel layout/offsets — the only non-DOM piece worth covering.
- **Runtime (hybrid, can't auto-verify):** `npm run tauri dev`; read which branch the console
  reports; record mic-only and mic+system 30-60s; **listen** for dropouts/sync; confirm live gain
  sliders still work; `ffprobe -show_streams out.mp4` → one AAC track at the context's actual rate.
- **Regression guard:** before/after the same recording on the ScriptProcessor fallback to prove
  the `emit()` extraction (step 3) changed nothing.

## Out of scope / future

- Replacing **WebCodecs `AudioEncoder`** or moving AAC encoding into the worklet (the worklet
  posts PCM out; the encoder stays main-thread — encoding in a worklet needs WASM AAC, out of
  scope).
- Migrating the **system-audio** node — already worklet-with-fallback; no work needed.
- `MediaStreamTrackProcessor` (the modern non-worklet API) — explicitly avoided for WKWebView
  compat (line 708-709); revisit only if WebKit ships it.
- Transferable `ArrayBuffer`s (zero-copy `postMessage`) instead of structured clone — a perf
  optimization if 128-quantum messaging ever shows up in profiles.
- Encoder bitrate/CBR work lives in `[[encoder-tuning]]`; per-source faders/monitor in
  `[[system-audio-polish]]`.

## References

- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — `startAudioProcessing`
  (711-797): WKWebView-compat comment (708-709), `createScriptProcessor(4096, …)` (731-735),
  `onaudioprocess` planar build + `AudioData`/`encode` (739-771), silent-gain sink (775-780),
  cleanup (782-787); `ensureAudioContext` rate contract (687-702); `acquireAudioTrack` mix graph
  (542-684), mic chain + gain refs (636-650), native-system gain (661-674); constants (90-91);
  invocation from `initializeWebCodecs` (947), encoder config (835-949); stop teardown (1242-1251).
- `frontend/src/lib/native-system-audio.ts` — **reference pattern:** `WORKLET_PROCESSOR_NAME`
  (24), inline `WORKLET_CODE` + `RingBufferSourceProcessor`/`registerProcessor` (30-64),
  `buildAudioSource` runtime probe (85), Blob-URL `addModule` (87-90), `AudioWorkletNode` opts
  (92-96), ScriptProcessor fallback (125), branch logs (106/166).
- `frontend/src/lib/audio-wire.ts` — shared wire codec (the colocation/pure-export precedent).
- `frontend/vite.config.ts` — vitest `environment: 'node'` (18-20): no DOM/AudioContext in tests.
- `ai-dev/doc/next-implementations.md` — backlog entry "AudioWorklet migration" (134-136).
- Related: `[[native-system-audio-capture]]` (worklet+fallback origin, AudioWorklet-support
  warning), `[[encoder-tuning]]`, `[[system-audio-polish]]`, `[[test-harness-and-ci]]`
  (pure-test-only philosophy + CI gates), `[[next-implementations]]` (roadmap index).
- Memory: `[[webkit-getusermedia-works-getdisplaymedia-blocked]]` (WKWebView mic works).
