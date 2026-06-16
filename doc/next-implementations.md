# Next Implementations — asmr-recorder

This is the index of planned work for the recorder. Each entry links to a detailed,
code-grounded implementation plan in this directory. Plans reference real files/symbols and
include step-by-step changes with verification checks.

_Last updated: 2026-06-15. The original five plans (P1 quick-wins → trim editor v2), the second
wave (system audio, packaging, dead-code removal, tests, composition layouts), and the third wave
(encoder tuning, AudioWorklet migration, auto-update, system-audio polish, per-section transport
teardown) are all shipped. The **fourth wave** below was discovered by a full code+docs audit once
the backlog ran dry — verified bug fixes, the mp4-muxer→mediabunny migration, a root-docs refresh,
and product-shell completeness._

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

## Planned work (fourth wave — from the 2026-06-15 codebase audit)

The third wave's backlog was empty, so a full 7-dimension audit (docs · recording core ·
transport/audio · Rust backend · trim editor · tech-debt/tests · release/CI), with every finding
adversarially re-verified against the code, surfaced this wave. Each item has a detailed plan.

| Plan | Priority | Effort | Risk | Surface |
|------|----------|--------|------|---------|
| [Bug fixes: trim A/V desync + native SCK stream leaks](bug-fixes-trim-desync-and-sck-leaks.md) | **P1** ✅ | M | med | frontend |
| [mp4-muxer → mediabunny migration](mp4-muxer-to-mediabunny-migration.md) | P2 ✅ | S | low–med | frontend |
| [Root-docs refresh (README + ARCHITECTURE + retire stale)](root-docs-refresh.md) | P2 ✅ | M | low | docs |
| [Product completeness (timeline · dead buttons · icon · reveal-in-Finder)](product-completeness-shell.md) | P2 ✅ | M | med | frontend+build |

> ✅ Shipped: **Bug fixes** (`a60f5fb` Bug A, `9072f9f` Bugs B & C); the **mp4-muxer → mediabunny
> migration** (recording now muxes with mediabunny; `mp4-muxer` dropped; main bundle −31 kB); and
> the **root-docs refresh** (README rewritten, new `/ARCHITECTURE.md`, the 4 stale recording docs
> moved to `docs/archive/` with a HISTORICAL banner). The docs-refresh adapted this plan to the
> post-migration reality: ARCHITECTURE names **mediabunny** (not mp4-muxer) as the live muxer.
>
> ✅ **Product completeness — done** (`ed74738` + `d6c5425`): timeline option C (honest read-only
> clip strip + "Open in editor", fake transport removed), reveal-in-Finder (`@tauri-apps/plugin-opener`
> in the save toast + Export), dead-transport-button removal, and a branded app icon (microphone mark
> on a gradient; source at `src-tauri/icons/app-icon.svg`, regenerated via `tauri icon`).
>
> **Fourth wave complete** + the preview-play-button follow-up (`560283c`): removed preview.tsx's
> dead play/pause overlay control and the now-orphaned `timelinePlayback` event (gone from the
> codebase). Open follow-ups (not yet planned): replace the placeholder icon with real designed art
> when available; and the interactive/manual verifications noted per item (record→trim ffprobe, SCK
> leak checks, reveal-in-Finder live, recording→mediabunny decode probe, auto-update round-trip).

### P1 — Bug fixes: trim frame-accurate A/V desync + native SCK stream leaks · _M_
Three verified defects on the audio/transport surface. **Bug A (HIGH):** the frame-accurate trim
export rebases video to `seg.in` but audio to the earlier keyframe `origin`, so audio leads video
by up to one ~3 s GOP — exactly in the feature's intended (non-keyframe in-point) case. **Bug B/C
(MED):** stopping a recording during the async audio-acquire window, or interrupting the monitor
(index 99) start while recording (index 0) starts, orphans native ScreenCaptureKit audio sessions
that live until app exit. Ship A first. → [bug-fixes-trim-desync-and-sck-leaks.md](bug-fixes-trim-desync-and-sck-leaks.md)

### P2 — mp4-muxer → mediabunny migration · _S_
Recording still muxes via the deprecated, unmaintained `mp4-muxer` while the trim editor already
uses `mediabunny` (bundled) — the app ships two MP4 muxers. Coupling is shallow (one import, one
target/ref, ~4 call sites). Swap recording onto mediabunny's `Output`/`Mp4OutputFormat`, drop the
`mp4-muxer` dep. Open question resolved in-plan: the WKWebView `extractAudioSpecificConfig` esds
handling is retained (mediabunny's `esds()` wraps the ASC the same way). →
[mp4-muxer-to-mediabunny-migration.md](mp4-muxer-to-mediabunny-migration.md)

### P2 — Root-docs refresh · _M_
The four root docs (README, RECORDING_IMPLEMENTATION, RECORDING_DEBUG_FINDINGS, DEBUGGING_GUIDE)
describe the **deleted** native FFmpeg / JSON-IPC-per-frame pipeline and cite a non-existent
`external_recorder.rs`; there is no accurate top-level ARCHITECTURE.md. Rewrite README, add
ARCHITECTURE.md (live path = frontend WebCodecs + muxer over raw-byte save IPC; capture = SCK over
Channel), and archive the three stale recording docs with a historical header. Sequence **after**
the muxer migration so ARCHITECTURE names the right muxer. → [root-docs-refresh.md](root-docs-refresh.md)

### P2 — Product completeness (shell polish) · _M_
The bottom-half Timeline (~350 lines, half the window) is **mock UI** — hardcoded tracks, empty
`clips`, a fake `setInterval` playhead that seeks no media — plus two dead `SkipBack`/`SkipForward`
toolbar buttons, the default Tauri placeholder app icon, and a save flow that only toasts a path
(the bundled `tauri-plugin-opener` is never called). Recommended: an honest read-only clip strip +
"Open in Editor" (option C), retire the dead transport, brand the icon, add a "Reveal in Finder"
toast action. → [product-completeness-shell.md](product-completeness-shell.md)

### Recommended sequencing (fourth wave)
1. **Bug fixes** — P1, user-facing desync + accumulating native leaks; ship Bug A first.
2. **mp4-muxer → mediabunny** — small, isolated; unifies the muxer and dissolves the untested
   `extractAudioSpecificConfig` risk.
3. **Root-docs refresh** — after the migration, so ARCHITECTURE.md names the surviving muxer.
4. **Product completeness** — gated on the Timeline wire-vs-remove decision (open question); the
   icon + reveal-in-Finder sub-items can land independently anytime.

---

## Backlog (not yet planned in detail)

Empty — the fourth wave (above) drained the 2026-06-15 audit findings into detailed plans. New
ideas land here first, then graduate to a plan doc.

> Note: `ai-dev/` is its own git repository. These docs are written but **not committed** —
> commit them from inside `ai-dev/` if you want them tracked.
