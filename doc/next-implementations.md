# Next Implementations — asmr-recorder

This is the index of planned work for the recorder. Each entry links to a detailed,
code-grounded implementation plan in this directory. Plans reference real files/symbols and
include step-by-step changes with verification checks.

_Last updated: 2026-06-14. The original five plans (P1 quick-wins → trim editor v2) and the
second wave (system audio, packaging, dead-code removal, tests, composition layouts) are all
shipped. This revision plans the **third wave** from the backlog — encoder tuning, AudioWorklet
migration, auto-update, system-audio polish, and per-section transport teardown — so the backlog
is now empty._

---

## Status snapshot (what's already landed)

Recently completed and on `main`:

- **2x2 WebCodecs recording pipeline** — composite → H.264 → `mp4-muxer` → raw-byte save IPC.
  **This is the live recording path** (the record button calls `startExternalRecording`,
  which runs `RecordingCanvas`; the native Rust recording pipeline is unreachable — see the
  dead-code plan below).
- **Audio quality** — stereo 48 kHz, CBR 256k AAC (WebKit VBR fallback), `esds` double-wrap fix.
- **Audio hardening (PR #2)** — sample-rate coherence; audio-encoder failures isolated to a
  video-only MP4.
- **Camera/mic hardening (PR #2)** — `NSCameraUsageDescription` + `NSMicrophoneUsageDescription`
  in `Info.plist`; mic-only startup prime.
- **Dev tooling (PR #2)** — fixed `dev.sh`; mediabunny code-split (main bundle ~460 kB).
- **Native screen capture** — ScreenCaptureKit → JPEG, with **region crop + multi-display
  picker** (PR #3) and **raw-byte transport + ack backpressure** (PR #4, replacing base64 +
  the 30fps cap).
- **Live audio monitor** — stereo meter + log-frequency spectrum, gated to recording (mic only).
- **Lossless trim editor + v2** — head/tail packet-copy, **multi-segment (mid-clip) cuts,
  opt-in frame-accurate re-encode, and a thumbnail filmstrip** (verified build; live pass pending).

### ✅ The original five plans (all shipped)

| Plan | Status |
|------|--------|
| [Camera & mic capture hardening](camera-mic-capture-hardening.md) | ✅ PR #2 |
| [Dev tooling & build hygiene](dev-tooling-and-build.md) | ✅ PR #2 |
| [Native screen: region + multi-display](native-screen-region-and-display.md) | ✅ PR #3 |
| [Native screen transport: raw bytes + backpressure](native-screen-transport-raw-bytes.md) | ✅ PR #4 |
| [Trim editor v2: multi-segment + frame-accurate + thumbnails](trim-editor-v2-multisegment.md) | ✅ merged |

Key environment facts (verified, easy to get wrong):

- `getUserMedia` (mic) **works** in the dev WKWebView; only `getDisplayMedia` (screen **and**
  system audio) is absent — which is why screen capture went native (SCK) and why **system
  audio is currently missing in the real app** (see the system-audio plan).
- `npm run tauri dev` is the working launcher (default features, no ffmpeg).
- `--features ffmpeg` fails to link on this arm64 machine and is a notarization footgun
  (Homebrew dylibs); the record/trim/save flow doesn't need it.
- The **live** recording path is the frontend WebCodecs pipeline; the native Rust pipeline
  (`manager.rs`/`encoder.rs`/`compositor.rs`/`external_recorder.rs` + the `start_recording`
  commands) is **unreachable from the mounted UI** — confirmed via the entry-point audit.

---

## Planned work (next wave)

| Plan | Priority | Effort | Risk | Surface |
|------|----------|--------|------|---------|
| [Native system-audio capture (SCK PCM → mix)](native-system-audio-capture.md) | **P1** ✅ | L | med-high | both |
| [Packaging & distribution (sign/notarize/version)](packaging-and-distribution.md) | **P1** ✅ | M | med | build |
| [Legacy native-pipeline removal (+ drop ffmpeg/cpal/nokhwa)](legacy-native-pipeline-removal.md) | P2 ✅ | L | med | backend |
| [Test harness for pure logic + CI build gates](test-harness-and-ci.md) | P3 ✅ | M | low | tooling |
| [Composition layouts (Solo · PiP · Side-by-Side)](composition-layouts.md) | P2 ✅ | M | med | frontend |

### P1 — Native system-audio capture · _L_
Close the last capture gap: system/application audio **doesn't work in the packaged app**
because it's acquired via `getDisplayMedia` (blocked in WKWebView). A complete SCK audio
capturer already exists (`system_audio_macos.rs`) but is only wired to the dead native
pipeline. Stream its raw PCM to the frontend over a Tauri `Channel` (mirroring the raw-byte
screen transport) and mix it into the recording `AudioContext` alongside the mic. →
[native-system-audio-capture.md](native-system-audio-capture.md)

### P1 — Packaging & distribution · _M_
Make the app shippable: code-sign (Developer ID + hardened runtime) and notarize so Gatekeeper
allows it and TCC grants persist, and fix `minimumSystemVersion` (`10.13` → `14.0`, matching
the SCK `macos_14_0` build feature). The camera/mic entitlements + usage strings are already
in place; this adds the signing/notarization plumbing and the version/category corrections. →
[packaging-and-distribution.md](packaging-and-distribution.md)

### P2 — Legacy native-pipeline removal · _L_
Delete the unreachable native recording pipeline (`manager`/`encoder`/`compositor`/
`external_recorder`/`webcam`/`audio` mic + the dead `start_recording`/`receive_video_frame`
commands and their unmounted frontend callers `Recorder.tsx`/`use-recording.ts`), and drop the
`cpal`, `nokhwa`, and `ffmpeg-next` deps (the arm64 link-failure feature). Audit-first; keep
the live screen/save paths and the SCK **audio** capturer (needed by the system-audio plan). →
[legacy-native-pipeline-removal.md](legacy-native-pipeline-removal.md)

### P3 — Test harness + CI · _M_
Add Vitest for the few pure, correctness-critical, runtime-free functions (the native
wire-format header parse; the trim segment-split math) and a `macos-latest` GitHub Actions
workflow running the build gates (`cargo build`/`cargo test`/`tsc -b`/`vite build`) on every
push. Deliberately modest — most of the app is I/O glue that can't be unit-tested without the
runtime. → [test-harness-and-ci.md](test-harness-and-ci.md)

### P2 — Composition layouts · _M_ ✅ (shipped, `d8083a7`)
Add **Solo / Fullscreen**, **Picture-in-Picture**, and **Side-by-Side** as selectable layouts
alongside the default 2×2 grid. The 2×2 geometry is currently hardcoded twice (CSS grid in
`preview.tsx`, Canvas-2D draws in `recording-canvas.tsx`); replace both with one data-driven
**layout registry** (`lib/layouts.ts`, normalized 0..1 slot rects, slot order = z-order) that
drives preview *and* recording. Adds a `fit:"cover"` mode to `drawSection` (required for the
portrait half-cells in side-by-side; avoids distorting non-16:9 sources) and z-ordering (PiP
overlay on top). Source model (fixed 4 sections) is unchanged — a layout just uses a subset. →
[composition-layouts.md](composition-layouts.md)

---

## Planned work (third wave — from the backlog)

These five were the backlog; each now has a detailed, code-grounded plan. The backlog is empty.

| Plan | Priority | Effort | Risk | Surface |
|------|----------|--------|------|---------|
| [System-audio polish (per-app capture · faders · monitor)](system-audio-polish.md) | P2 | M–L | med | both |
| [Encoder tuning (AAC CBR · adaptive backpressure · keyframe)](encoder-tuning.md) | P3 | S–M | low–med | frontend |
| [AudioWorklet migration (off `ScriptProcessorNode`)](audioworklet-migration.md) | P3 | M | med | frontend |
| [Auto-update (`tauri-plugin-updater` + GH manifest)](auto-update.md) | P3 | M | med | build+backend |
| [Per-section transport teardown (unregister Channel on stop)](per-section-transport-unregister.md) | P3 | S | low | both |

### P2 — System-audio polish · _M–L_
Three deferred items on top of the shipped SCK audio path: **per-app/per-process capture** (SCK
`SCContentFilter` + an app picker), **independent mic/system faders** (mostly shipped — only
mute/solo, dB labels, and record-time meters remain), and a **system-audio pre-record monitor**
(today's monitor is mic-only). Highest user value of the wave. Open risk: whether SCK actually
restricts the *audio* mix to the chosen app at the 14.0 floor needs a runtime test. →
[system-audio-polish.md](system-audio-polish.md)

### P3 — Encoder tuning · _S–M_
Finish the encoder pass started in `2d42ceb` (bitrate↔quality, 3s GOP). Remaining: **verify
whether WKWebView honors AAC CBR** (likely not → measure-and-document, don't over-promise), an
**adaptive `encodeQueueLimit(w,h,fps)`** backpressure helper (pure + unit-tested), and
**keyframe-spacing validation** against the trim editor on long recordings. →
[encoder-tuning.md](encoder-tuning.md)

### P3 — AudioWorklet migration · _M_
Move the recording mic tap off the deprecated `ScriptProcessorNode`. `native-system-audio.ts`
already ships the AudioWorklet-with-fallback pattern, so the WKWebView-support question is
half-answered; reuse that pattern (with a new input-side processor). Keep the fallback unless the
runtime branch proves AudioWorklet is always taken. → [audioworklet-migration.md](audioworklet-migration.md)

### P3 — Auto-update · _M_
`tauri-plugin-updater` + a GitHub-releases `latest.json`. Pairs with the shipped packaging work;
the spine is the **two-keypair** model (Apple Developer ID codesign/notarization vs. the separate
minisign updater key) plus a tag-triggered release workflow. → [auto-update.md](auto-update.md)

### P3 — Per-section transport teardown · _S_
Small hygiene fix deferred from the raw-byte transport plan: the per-section `Channel.onmessage`
is never unregistered on stop (verified — Tauri only frees it on a channel `end` the workers
never send), leaking the closure + stale `createImageBitmap`/`ack` work. Frontend-only neuter on
stop, both screen and system-audio paths. → [per-section-transport-unregister.md](per-section-transport-unregister.md)

---

## Recommended sequencing (third wave)

The first two waves are fully shipped. Suggested order for the third:

1. **Per-section transport teardown** — small, low-risk hygiene win; de-risks the transport
   before system-audio polish adds a second concurrent stream (the monitor).
2. **System-audio polish** — highest user value; builds directly on the shipped SCK audio path.
3. **AudioWorklet migration** — clean up the audio tap once polish has already been in that code;
   the two overlap on the AudioWorklet / `ScriptProcessorNode` question.
4. **Encoder tuning** — a measurement pass (CBR, backpressure, GOP); low coupling, can slot in
   anytime, but cheaper after the audio work settles.
5. **Auto-update** — distribution capstone; depends only on the already-shipped packaging/signing.

> Items 1-3 all touch the audio/transport surface — landing them in order keeps each change
> reviewing against a known-clean base instead of racing the others.

---

## Backlog (not yet planned in detail)

Empty — every previously-listed item now has a detailed plan above (third wave). New ideas land
here first, then graduate to a plan doc.

> Note: `ai-dev/` is its own git repository. These docs are written but **not committed** —
> commit them from inside `ai-dev/` if you want them tracked.
