# Feature: Migrate the recording muxer off mp4-muxer onto mediabunny

**Priority:** P2  ·  **Effort:** S  ·  **Risk:** med  ·  **Surface:** frontend

## Problem / motivation

The app ships **two** MP4 muxers. The live recording path uses `mp4-muxer`
(`recording-canvas.tsx`), while the post-record trim editor already uses
`mediabunny`'s `Output` API (`trim-editor.tsx`). `mp4-muxer` is deprecated (see
`[[next-implementations]]` / the memory note "mp4-muxer deprecated → Mediabunny"),
and carrying a second muxer means a second AAC/esds code path, a second container
quirk surface, and dead weight in `frontend/package.json`. mediabunny is already
bundled (v1.46, `frontend/package.json:30`) and code-split (it is only reached via
the dynamic `await import("mediabunny")` in the trim editor), so adopting it for
recording adds **zero** new dependency — it removes one.

Goal: **one muxer in the app.** Swap the four `mp4-muxer` call sites in
`recording-canvas.tsx` to mediabunny's `Output` + `EncodedVideoPacketSource` /
`EncodedAudioPacketSource` + `BufferTarget`, then drop `mp4-muxer` from
`package.json`. The coupling is shallow (one import, one ref/type, ~4 call sites),
so this is a contained, mechanical migration — with one real landmine in the AAC
metadata handling, resolved below.

## Current state (grounded)

All `mp4-muxer` usage lives in **one file**, `frontend/src/components/asmr-recorder/recording-canvas.tsx`
(confirmed: `grep -rn mp4-muxer src/` hits only this file):

- **Import** — `import { Muxer, ArrayBufferTarget } from "mp4-muxer";` (line 9).
- **Type ref** — `const muxerRef = useRef<Muxer<ArrayBufferTarget> | null>(null);` (line 275).
- **Construction** — `new Muxer({ target, video: { codec: "avc", width, height }, audio?: { codec: "aac", numberOfChannels, sampleRate }, fastStart: "in-memory", firstTimestampBehavior: "offset" })` with `const target = new ArrayBufferTarget()` (lines 1036-1050).
- **Video chunk handoff** — inside the `VideoEncoder` output callback: `muxer.addVideoChunk(chunk, metadata)` (line 1055), incrementing `muxedChunkCountRef` (line 1056).
- **Audio chunk handoff** — inside the `AudioEncoder` output callback (lines 950-993): pulls `metadata.decoderConfig`, runs `extractAudioSpecificConfig(dc.description)` (line 956-958), and calls `muxer.addAudioChunk(chunk, { ...metadata, decoderConfig: { ...dc, description: asc } })` when an ASC was extracted, else `muxer.addAudioChunk(chunk, metadata)` (lines 959-966).
- **Finalize + read + save** — in the stop-cleanup async IIFE: `muxer.finalize()` (line 1426), `const mp4Buffer = muxer.target.buffer` (line 1427), an edit-copy `new Blob([mp4Buffer], …)` (line 1432), then `await saveRecording(mp4Buffer)` (line 1436) and the `recordingReadyForEdit` event (lines 1437-1441). `saveRecording` invokes the raw-byte `save_media_recording` IPC (lines 569-586, `invoke("save_media_recording", data)` at 574).

**The AAC quirk (the load-bearing risk).** WKWebView/Safari's `AudioEncoder` emits
`decoderConfig.description` as a full MPEG-4 **ES_Descriptor**, not a bare
AudioSpecificConfig (ASC). `extractAudioSpecificConfig` (lines 158-214) descends the
descriptor tree (tags 0x03 → 0x04 → 0x05) to the bare ASC, returning `null` for
Chrome (which already gives a bare ASC) — see the doc comment at lines 144-157. The
existing workaround exists because **mp4-muxer wraps whatever it receives in its own
DecoderSpecificInfo (esds)**; handing it the ES_Descriptor double-wraps it and the
audio track becomes undecodable (comment lines 149-154).

**mediabunny behaves IDENTICALLY here** — verified in the bundled source, this run:
- `esds()` (`node_modules/mediabunny/dist/modules/src/isobmff/isobmff-boxes.js:697-754`) takes `trackData.info.decoderConfig.description` and, at lines 728-736, wraps it once in `u8(0x05) /* DecoderSpecificInfo */ + length + description`. So a bare ASC in → correct esds; an ES_Descriptor in → double-wrapped esds (the same bug).
- The audio track-setup path (`isobmff-muxer.js:266-289`) only *rebuilds* a description when **none** is provided (the ADTS case, lines 268-288). When a description IS present it is used **verbatim**, and the error message at lines 274-276 states the contract explicitly: the description **"must be an AudioSpecificConfig as specified in ISO 14496-3"** — i.e. the bare ASC.

**Conclusion:** mediabunny does **not** auto-strip an ES_Descriptor. `extractAudioSpecificConfig`
**must be kept** and applied to the metadata before `audioSource.add(...)`, exactly as
today. This is the opposite of the optimistic "mediabunny builds the esds itself, so
delete the workaround" hypothesis — verifying it was the point of this plan. (It does
mean the separate "extractAudioSpecificConfig is untested" concern is **not** dissolved
by this migration; that helper survives unchanged and keeps its existing behavior.)

**The trim editor is the working migration template** — `frontend/src/components/asmr-recorder/trim-editor.tsx`:
- `const output = new mb.Output({ format: new mb.Mp4OutputFormat({ fastStart: "in-memory" }), target: new mb.BufferTarget() })` (lines 80-83).
- `const videoSource = new mb.EncodedVideoPacketSource(videoTrack.codec); output.addVideoTrack(videoSource)` (lines 84-85); audio analogously via `mb.EncodedAudioPacketSource(audioCodec)` + `output.addAudioTrack(audioSource)` (lines 86-89).
- `await output.start()` (line 91) **before** any `add`; per-packet `await videoSource.add(new mb.EncodedPacket(...), firstVideo ? videoMeta : undefined)` (lines 148-151, 160-163, 173-176) and `await audioSource.add(... , firstAudio ? audioMeta : undefined)` (lines 188-197) — note the decoder config (`videoMeta`/`audioMeta`, lines 93-98) is passed **only on the first packet**.
- `await output.finalize(); return output.target.buffer` (lines 203-204). Both recording and trim export then save via the same raw-byte `invoke("save_media_recording", buffer)` (trim-editor.tsx:469).

**mediabunny API facts (verified in `node_modules/mediabunny/dist/mediabunny.d.ts` this run):**
- `EncodedVideoPacketSource(codec: VideoCodec)` / `EncodedAudioPacketSource(codec: AudioCodec)` — constructors take a codec string (d.ts 1612-1614 / 1409-1411). Valid strings include `"avc"` and `"aac"` (`codec.js` `VIDEO_CODECS`/`AUDIO_CODECS`: `'avc'`, …; `'aac'`, …).
- `add(packet: EncodedPacket, meta?: EncodedVideoChunkMetadata | EncodedAudioChunkMetadata): Promise<void>` (d.ts 1625 / 1421). `meta` is the **standard WebCodecs metadata** type (d.ts 1421/1625 reference the DOM `EncodedVideoChunkMetadata`/`EncodedAudioChunkMetadata`) — i.e. exactly the `metadata` object the encoder `output` callback already receives. The docs say to pass `meta` "for the first call, including a valid decoder config" (d.ts 1415-1416 / 1619-1620).
- `EncodedPacket.fromEncodedChunk(chunk: EncodedVideoChunk | EncodedAudioChunk): EncodedPacket` (d.ts 1508-1513) converts a WebCodecs chunk directly — no manual byte copy needed.
- `Output.start()` / `finalize()` / `add()` are all **async** (d.ts 3287-3290, 3306-3310, 1421/1625). mp4-muxer's `addVideoChunk`/`finalize` were **synchronous** — this is the one structural difference to handle (see design §3).
- `Mp4OutputFormat({ fastStart })` accepts `false | 'in-memory' | 'reserve' | 'fragmented'` (d.ts `IsobmffOutputFormatOptions.fastStart`). **There is no `firstTimestampBehavior` option** — mediabunny does not auto-offset the first timestamp; the caller controls timestamps (trim-editor rebases manually). For recording this is a non-issue: video timestamps already start at ~0 (`updateFrame` computes `(performance.now() - recordingStartTimeRef.current) * 1000`, lines 1214-1215) and audio at 0 (`sampleOffset` starts at 0, line 791). The `firstTimestampBehavior: "offset"` on the current muxer is therefore a near-no-op we simply drop.

No test imports `recording-canvas.tsx` (`grep` of `src/**/*.test.*` finds none; the 6
test files are pure-logic units). No `manualChunks` config in `vite.config.ts` — mediabunny
already lands in its own chunk via the trim editor's dynamic import.

## Proposed design

Swap the muxer in place. Keep every surrounding behavior — the WebCodecs encoders, the
ASC workaround, `muxedChunkCountRef`, the WebM fallback, `saveRecording`, and the
`recordingReadyForEdit` event — untouched.

### 1. Import + ref type

Replace the static `mp4-muxer` import (line 9) with a static mediabunny import. mediabunny
is already in a code-split chunk; importing the `Output`/source/format/`BufferTarget`/`EncodedPacket`
classes statically into `recording-canvas.tsx` will pull (part of) mediabunny into this
component's chunk. To preserve the existing lazy-load characteristic and keep the
record-path bundle lean, prefer a **dynamic** `import("mediabunny")` inside
`initializeWebCodecs` (which is already `async`-adjacent — it's called from the async
`startRecording`). Recommendation: dynamic import, mirroring trim-editor's `const mb = await import("mediabunny")`.

```ts
// remove: import { Muxer, ArrayBufferTarget } from "mp4-muxer";
import type { Output, BufferTarget, EncodedVideoPacketSource, EncodedAudioPacketSource } from "mediabunny";
```

Ref type becomes a small holder so the stop-cleanup can finalize and read the buffer:

```ts
const muxerRef = useRef<{
  output: Output;
  videoSource: EncodedVideoPacketSource;
  audioSource: EncodedAudioPacketSource | null;
} | null>(null);
```

(Replaces `useRef<Muxer<ArrayBufferTarget> | null>` at line 275.)

### 2. Construction (in `initializeWebCodecs`)

`initializeWebCodecs` is currently **synchronous** and returns `boolean` (line 916-922). It
must become `async` to `await import("mediabunny")` and `await output.start()`. The single
caller, `startRecording`, already `await`s nothing between the `initializeWebCodecs(...)` call
(line 1302) and its `if (!webCodecsReady)` check — change that to `await`.

```ts
const mb = await import("mediabunny");
const output = new mb.Output({
  format: new mb.Mp4OutputFormat({ fastStart: "in-memory" }),
  target: new mb.BufferTarget(),
});
const videoSource = new mb.EncodedVideoPacketSource("avc");
output.addVideoTrack(videoSource);
const audioSource = hasAudio ? new mb.EncodedAudioPacketSource("aac") : null;
if (audioSource) output.addAudioTrack(audioSource);
await output.start();          // must precede any add()
muxerRef.current = { output, videoSource, audioSource };
```

The old `video: { width, height }` / `audio: { numberOfChannels, sampleRate }` config is not
needed at construction — mediabunny derives those from the first packet's `decoderConfig`
metadata (which the encoder supplies), so width/height/channels/rate flow through automatically.

### 3. Chunk handoff — convert chunk, pass metadata, handle async `add()`

The encoder `output` callbacks are **synchronous** but mediabunny `add()` is **async**. mp4-muxer's
`addVideoChunk` was sync. Two options:

- **(a) await inside the callback** by making the `output` callback `async` and `await`ing
  `add()`. Risk: WebCodecs may invoke `output` callbacks concurrently; awaiting could reorder.
- **(b) Fire-and-forget with a serialized tail-promise**, mirroring mediabunny's own contract
  ("await this Promise to respect backpressure", d.ts 1418-1420). RECOMMENDED: chain adds on a
  per-source promise so decode order is preserved and we can `await` the tail at finalize:

```ts
let videoTail: Promise<void> = Promise.resolve();
// video output callback:
output: (chunk, metadata) => {
  const m = muxerRef.current; if (!m) return;
  const pkt = mb.EncodedPacket.fromEncodedChunk(chunk);
  videoTail = videoTail.then(() =>
    m.videoSource.add(pkt, firstVideo ? metadata : undefined)
  ).then(() => { firstVideo = false; });
  muxedChunkCountRef.current++;
},
```

Audio callback keeps the ASC fix verbatim, then converts + adds:

```ts
const dc = metadata?.decoderConfig;
const asc = dc?.description ? extractAudioSpecificConfig(dc.description) : null;
const meta = dc && asc ? { ...metadata, decoderConfig: { ...dc, description: asc } } : metadata;
const pkt = mb.EncodedPacket.fromEncodedChunk(chunk);
audioTail = audioTail.then(() =>
  m.audioSource!.add(pkt, firstAudio ? meta : undefined)
).then(() => { firstAudio = false; });
```

`firstVideo`/`firstAudio` flags (and the `videoTail`/`audioTail` promises) live in the
`initializeWebCodecs` closure alongside the sources — passing `metadata` only on the first
packet matches mediabunny's expectation and the trim editor's pattern. The audio bitrate
instrumentation (lines 967-986) is unrelated and stays as-is.

### 4. Finalize + read buffer (in stop-cleanup)

```ts
const m = muxerRef.current;
if (videoEncoder && m) {
  await videoEncoder.flush(); videoEncoder.close();
  if (audioEncoder?.state === "configured") { await audioEncoder.flush(); audioEncoder.close(); }
  await Promise.all([videoTailRef.current, audioTailRef.current]); // drain queued adds
  if (muxedChunkCountRef.current === 0) { /* unchanged skip */ return; }
  await m.output.finalize();
  const mp4Buffer = m.output.target.buffer as ArrayBuffer;
  // ...unchanged: editBlob, saveRecording(mp4Buffer), recordingReadyForEdit event
}
```

`muxer.finalize()` (sync, line 1426) → `await m.output.finalize()`; `muxer.target.buffer`
(line 1427) → `m.output.target.buffer`. Because `add()` is async and the encoder flush emits
the last chunks just before finalize, **drain the tail promises after `flush()`/`close()` and
before `finalize()`** so no `add` is in flight when the file is sealed. The
`videoTail`/`audioTail` need a ref (`videoTailRef`/`audioTailRef`) so the cleanup closure can
await them, or be stored on the `muxerRef.current` holder object.

Everything downstream — the `editBlob` copy (line 1432), `saveRecording` (line 1436), the
`recordingReadyForEdit` dispatch (1437-1441), the WebM fallback branch, audio/context teardown
— is **unchanged**.

### 5. Drop the dependency

Remove `"mp4-muxer": "^5.2.2"` from `frontend/package.json:31` and run `npm install` to prune
`package-lock.json`.

## Implementation steps

1. **Swap import + ref type.** Replace the `mp4-muxer` import (line 9) with a mediabunny `import type {…}` and switch `muxerRef` (line 275) to the holder type from design §1. **verify:** `npx tsc -b` shows only the migration's expected in-progress errors (the call sites not yet updated) — no `mp4-muxer` module-resolution error remains.
2. **Make `initializeWebCodecs` async + build the mediabunny `Output`.** Change its signature to `Promise<boolean>`, `await import("mediabunny")`, construct `Output`/`Mp4OutputFormat`/`BufferTarget` + sources, `await output.start()`, store the holder in `muxerRef`. Update the caller in `startRecording` to `const webCodecsReady = await initializeWebCodecs(...)` (line 1302). Add `firstVideo`/`firstAudio` flags and `videoTail`/`audioTail` (held via refs or on the holder). **verify:** `npx tsc -b` clean for this function; `npx vite build` succeeds.
3. **Rewire the video chunk handoff.** In the `VideoEncoder.output` callback (line 1055) convert via `EncodedPacket.fromEncodedChunk(chunk)` and chain `videoSource.add(pkt, firstVideo ? metadata : undefined)` on `videoTail`; keep `muxedChunkCountRef.current++`. **verify:** `npx tsc -b` clean.
4. **Rewire the audio chunk handoff, KEEPING `extractAudioSpecificConfig`.** In the `AudioEncoder.output` callback (lines 950-993) keep the ASC extraction (lines 955-958) and the `{ ...dc, description: asc }` rewrite, then convert + chain `audioSource.add(pkt, firstAudio ? meta : undefined)`. Leave the bitrate instrumentation untouched. **verify:** `npx tsc -b` clean; `npm run test` still 48 passing (no unit touches this path, so count is unchanged — this gate guards against accidental breakage elsewhere).
5. **Rewire finalize/read.** In the stop-cleanup IIFE: after `videoEncoder.flush()/close()` and the audio flush, `await Promise.all([videoTail, audioTail])`, then `await m.output.finalize()` and read `m.output.target.buffer`. Keep the `muxedChunkCountRef === 0` skip, `editBlob`, `saveRecording`, and the edit event identical. Reset `muxerRef.current = null` where the old code nulled the three refs (lines 1451-1453 → null the holder; `videoEncoderRef`/`audioEncoderRef` nulling stays). **verify:** `npx vite build`; manual record produces a file (next gate).
6. **Drop the dependency.** Remove `mp4-muxer` from `frontend/package.json` and `npm install`. **verify:** `npx vite build` still succeeds (no stray `mp4-muxer` import); `grep -rn mp4-muxer src/` returns nothing.
7. **Real-recording ffprobe gate (the decisive check).** `npm run tauri dev`, record a clip **with system audio + mic on** (exercises the AAC/esds path), let the trim editor open, then run `ffprobe` on the saved MP4. **verify:** `ffprobe -v error -show_streams <file>.mp4` shows a video stream `codec_name=h264` and an audio stream `codec_name=aac`, `channels=2`, `sample_rate=48000`, and **`ffprobe -show_entries stream=codec_name -of csv <file>` plus a decode probe `ffprobe -v error -i <file> -f null -` emits no esds/ASC errors** (i.e. the AAC track decodes — proving the ASC workaround still produces a single, valid esds under mediabunny). Also open the file in the trim editor and scrub: thumbnails render and audio plays.

## Interface / API changes

- **No public component API change.** `RecordingCanvas` props (lines 26-74) and `RecordingCanvasRef` (76-81) are unchanged.
- **Internal:** `initializeWebCodecs` becomes `async` (`(...) => Promise<boolean>`); its single caller already runs inside an `async` function and just adds `await` (line 1302). `muxerRef` changes type from `Muxer<ArrayBufferTarget>` to a `{ output, videoSource, audioSource }` holder.
- **Dependency:** `mp4-muxer` removed from `frontend/package.json`; mediabunny (already present) is now used by both recording and trimming.
- **No backend / IPC change.** `save_media_recording` raw-byte command (`src-tauri/src/lib.rs`) and `saveRecording` are untouched; the output is still an MP4 `ArrayBuffer`.

## Edge cases & risks

- **AAC double-wrap regression (highest risk).** Verified mediabunny wraps `description` in an esds DecoderSpecificInfo just like mp4-muxer (`isobmff-boxes.js:728-736`) and requires a bare ASC (`isobmff-muxer.js:274-276`). Keeping `extractAudioSpecificConfig` is mandatory; step 7's ffprobe decode probe is the gate that catches a regression here.
- **Async `add()` reordering / backpressure.** WebCodecs `output` callbacks fire in decode order, but `add()` is async. Serializing on a per-source tail promise (design §3b) preserves order and lets finalize await completion. Not awaiting before `finalize()` could drop the trailing chunks — step 5 explicitly drains the tail before finalize.
- **`fromEncodedChunk` data ownership.** `EncodedChunk` is detached/neutered after the callback in some engines; `fromEncodedChunk` copies the bytes into an `EncodedPacket` synchronously (it reads the chunk in-callback), so the subsequent async `add` is safe. If WebKit proves otherwise, fall back to `chunk.copyTo(buf)` + `new EncodedPacket(buf, …)` — but the trim editor relies on `EncodedPacket` data being owned, so this is the expected contract.
- **`muxedChunkCountRef === 0` short-circuit.** Preserved (lines 1419-1424) — if no video chunk muxed, skip `finalize()` and the save, same as today.
- **WebM fallback path.** Untouched. mediabunny only powers the WebCodecs branch; `useWebCodecsRef=false` still routes to MediaRecorder (lines 1454-1470).
- **Bundle size.** Net **decrease**: `mp4-muxer` (~all of its ~tens of kB) leaves the bundle; mediabunny was already shipped. Whether the record path pulls mediabunny eagerly depends on static-vs-dynamic import (design §1 recommends dynamic to keep the initial chunk lean) — confirm with the `vite build` chunk report in step 6.
- **`firstTimestampBehavior` dropped.** mediabunny has no equivalent; recording timestamps already start at ~0, so there is nothing to offset. If a future audio-leading-video offset surfaces, rebase in the callback as the trim editor does (trim-editor.tsx:129-135) — out of scope here.

## Testing & verification

**Build gates (auto):**
- `npx tsc -b` — clean (only the pre-existing errors noted in the dev logs).
- `npx vite build` — succeeds; inspect the chunk report to confirm `mp4-muxer` is gone and the mediabunny chunk placement is as expected.
- `npm run test` — 48 tests still pass (no unit covers `recording-canvas.tsx`; this guards collateral damage).

**Manual (dev — cannot be auto-verified):**
- `npm run tauri dev`, record a clip **with mic + system audio enabled**, confirm the trim editor opens and the saved-path toast/event fires.
- **ffprobe the saved MP4** (step 7): `codec_name=h264` video + `codec_name=aac`, `channels=2`, `sample_rate=48000` audio; a `-f null -` decode probe emits **no esds/AAC errors** (the single-esds correctness check). This is the proof the migration preserved the WKWebView AAC fix.
- Scrub/play in the trim editor: thumbnails render, audio is audible and in sync, and a re-export still saves (the trim path was already mediabunny — this confirms the recorded file is well-formed enough for mediabunny's demuxer).
- Record a **video-only** clip (mic + system audio off) and confirm it saves and decodes (exercises the `audioSource === null` branch).

## Out of scope / future

- **Refactoring `extractAudioSpecificConfig` or unit-testing it.** It survives unchanged; the long-standing "untested helper" concern is *not* resolved by this migration (mediabunny needs the same bare ASC), and adding a parser unit test is a separate, optional task — do not bundle it here (`CLAUDE.md` §2/§3).
- **Sharing a single "build MP4 Output" helper between `recording-canvas.tsx` and `trim-editor.tsx`.** The two have different sources (live `EncodedPacket` from encoders vs. demuxed packet-copy) and different lifecycles; one shared abstraction across two call sites is premature.
- **Switching the recording path to mediabunny's `VideoSampleSource`/`AudioSampleSource` encode API** (let mediabunny own the encoder, not just the muxer). That would dissolve the manual `VideoEncoder`/`AudioEncoder` wiring AND potentially the ASC workaround, but it is a far larger rewrite of a verified WebCodecs path — explicitly deferred; this plan is muxer-swap only.
- **`firstTimestampBehavior`/offset handling**, fragmented MP4, or any container-feature change.

## References

- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — `mp4-muxer` import (9); `muxerRef` type (275); `saveRecording`/`save_media_recording` (569-586, invoke at 574); `extractAudioSpecificConfig` + WKWebView ASC doc comment (144-214); audio chunk handoff + ASC fix (950-993, `addAudioChunk` 960/965); `initializeWebCodecs` (916-1115), `new Muxer` (1036-1050), `addVideoChunk`+count (1055-1056), `encoder.configure` (1067-1075); caller `await`-point (1302), `webCodecsReady` check (1308); stop-cleanup finalize/read/save (1403-1453, `muxer.finalize()` 1426, `muxer.target.buffer` 1427, `editBlob` 1432, `saveRecording` 1436, `recordingReadyForEdit` 1437-1441, ref nulling 1451-1453); audio timestamp base (`sampleOffset` 791, 806), video timestamp base (1214-1215).
- `frontend/src/components/asmr-recorder/trim-editor.tsx` — `Output`/`Mp4OutputFormat`/`BufferTarget` (80-83), `EncodedVideoPacketSource`/`addVideoTrack` (84-85), `EncodedAudioPacketSource`/`addAudioTrack` (86-89), `output.start()` (91), decoder meta first-packet-only (93-98, 148-151, 188-197), `output.finalize()`+`target.buffer` (203-204), shared `save_media_recording` invoke (469).
- `frontend/package.json` — `mediabunny ^1.46.0` (30), `mp4-muxer ^5.2.2` (31, to remove).
- `frontend/node_modules/mediabunny/dist/mediabunny.d.ts` — `BufferTarget` (614); `EncodedAudioPacketSource` (1409-1422) / `EncodedVideoPacketSource` (1612-1626) constructors + async `add(packet, meta?)`; `EncodedPacket.fromEncodedChunk` (1508-1513); `Output` lifecycle `start`/`finalize`/`addVideoTrack`/`addAudioTrack` (3251-3310); `Mp4OutputFormat`/`IsobmffOutputFormatOptions.fastStart` (3110+).
- `frontend/node_modules/mediabunny/dist/modules/src/isobmff/isobmff-boxes.js` — `esds()` wraps `decoderConfig.description` in a single DecoderSpecificInfo (697-754, wrap at 728-736).
- `frontend/node_modules/mediabunny/dist/modules/src/isobmff/isobmff-muxer.js` — AAC track setup uses the provided description verbatim, requires a bare ASC (266-289; contract message 274-276).
- `frontend/node_modules/mediabunny/dist/modules/src/codec.js` — `VIDEO_CODECS` includes `'avc'`, `AUDIO_CODECS` includes `'aac'`.
- `frontend/vite.config.ts` — no `manualChunks`; mediabunny code-split via dynamic import.
- Related: `[[next-implementations]]`, `[[trim-editor-v2-multisegment]]`, `[[encoder-tuning]]`, `[[audioworklet-migration]]`.