# Feature: Refresh stale, code-contradicting root documentation

**Priority:** P2  ·  **Effort:** M  ·  **Risk:** low  ·  **Surface:** docs

## Problem / motivation

The four root-level docs (`README.md`, `RECORDING_IMPLEMENTATION.md`,
`RECORDING_DEBUG_FINDINGS.md`, `DEBUGGING_GUIDE.md`) plus the unlabeled
`initial-development-instructions` describe a recording engine that **no longer
exists**. They present `cpal` + `scrap` + `ffmpeg-next` and a JSON-IPC per-frame
transport (480x270 / 960x540 downscaling) as the live pipeline. In reality the
native Rust recording pipeline was deleted (see `[[legacy-native-pipeline-removal]]`),
the live path is the **frontend WebCodecs pipeline** muxing with `mp4-muxer` and
saving via a **raw-byte IPC**, and `ffmpeg-next` is not a dependency at all.

This is a solo portfolio repo (git user `BusanSashimi`), so "onboarding a new
hire" is not the driver. The real harm is **trust**: a reader — or, more sharply,
an AI coding agent pointed at the repo root — that reads these docs will reach for
deleted symbols (`external_recorder.rs`, `RECORDING_SCALE` scaling, FFmpeg encode)
and write code against an architecture that was removed. The docs don't merely
omit; they assert false things and cite a file that isn't there. The fix is
doc-only: rewrite `README.md`, add a single accurate `ARCHITECTURE.md`, and retire
the obsolete recording docs behind an unmistakable "historical" label.

## Current state (grounded)

**README.md is wrong about the engine:**
- `README.md:18` — "High-Fidelity Audio Recording ... powered by `cpal`".
- `README.md:19` — "Screen Capture ... with `scrap`".
- `README.md:135-139` — Key Libraries table lists `cpal` (audio I/O), `scrap`
  (screen capture), `ffmpeg-next` (media processing, "planned") as the stack.
- The actual dependency reality in `src-tauri/Cargo.toml`:
  - `screencapturekit = { version = "1", features = ["async", "macos_14_0"] }`
    is the macOS capture dep (`Cargo.toml:45-46`).
  - `cpal = "0.15"` is gated to **non-macOS only**
    (`[target.'cfg(not(target_os = "macos"))'.dependencies]`, `Cargo.toml:48-50`)
    and used only by `system_audio_fallback.rs`.
  - `scrap = "0.5"` is gated to **non-macOS-non-Windows only**
    (`[target.'cfg(not(any(target_os = "macos", target_os = "windows")))']`,
    `Cargo.toml:55-56`), used only by `screen_fallback.rs`.
  - **`ffmpeg-next` is absent** — `grep -in ffmpeg src-tauri/Cargo.toml` returns
    nothing. (812 `ffmpeg` hits exist under `src-tauri/target/` but those are
    **stale build-cache fingerprints** from the deleted pipeline, not source or
    manifest — verified: every match path is `src-tauri/target/{debug,release}/...`.)
- README never mentions WebCodecs, `mp4-muxer`/`mediabunny`, ScreenCaptureKit,
  React 19, the raw-byte save IPC, or the **DYLD/Swift prerequisite**. The macOS
  "System Dependencies" section is only `xcode-select --install`
  (`README.md:41-43`); there is no mention that `DYLD_LIBRARY_PATH=/usr/lib/swift`
  is required for the `screencapturekit` Swift interop — which the `tauri` npm
  script injects (`package.json` root: `"tauri": "... export
  DYLD_LIBRARY_PATH=\"/usr/lib/swift:$DYLD_LIBRARY_PATH\"; tauri"`) and `dev.sh`
  relies on (`dev.sh` execs `npm run tauri dev`, comment notes the injection).

**The live stack (verified, for the rewrite):**
- React 19.2 + Vite 7 + Vitest 4: `frontend/package.json:32` (`react ^19.2.0`),
  `:52` (`vite ^7.2.4`), `:53` (`vitest ^4.1.8`).
- Live recording path = WebCodecs in
  `frontend/src/components/asmr-recorder/recording-canvas.tsx`, muxed with
  `mp4-muxer`: `recording-canvas.tsx:9` (`import { Muxer, ArrayBufferTarget } from
  "mp4-muxer"`). `RECORDING_SCALE = 1 / 1` at `recording-canvas.tsx:84` (full
  resolution; the old 1/4 downscale is gone).
- Trim editor uses `mediabunny` (dynamic import, lossless packet-copy):
  `trim-editor.tsx:5,74,297` (`import("mediabunny")`). Both deps bundled:
  `frontend/package.json:30` (`mediabunny ^1.46.0`), `:31` (`mp4-muxer ^5.2.2`).
- Save IPC is raw-byte, not base64: `src-tauri/src/lib.rs:14-20` —
  `save_media_recording` requires `tauri::ipc::InvokeBody::Raw(video_bytes)`.
- Backend is SCK capture over Tauri Channel + the save command. Modules in
  `lib.rs:3-7` (`screen`, `system_audio`, `recording`, `screen_stream`,
  `system_audio_stream`); handler list `lib.rs:89-100`. `main.rs` is a 4-line shim
  (`main.rs:4-6` → `asmr_recorder_lib::run()`); all wiring is in `lib.rs`.
- macOS capture is native because `getDisplayMedia` is blocked in WKWebView while
  `getUserMedia` (mic) works — see memory `webkit-getusermedia-works-getdisplaymedia-blocked`
  and `[[native-system-audio-capture]]`.

**RECORDING_IMPLEMENTATION.md describes the deleted pipeline:**
- `:1-21` — two modes "Native Recording" / "Composite Recording" where "Tauri
  encodes frames with FFmpeg" (`:36`).
- `:100-106` — data-flow diagram ending `[FFmpeg Encoder] → [MP4]`.
- `:119` — "Code Locations ... Backend frame receiver:
  `src-tauri/src/external_recorder.rs`". **That file does not exist** —
  `grep -rn external_recorder src-tauri/src/` returns nothing; the name survives
  only in `docs/logs/2026-06-14.md` and `ai-dev/dev-logs/*` (which record its
  deletion) and `ai-dev/doc/legacy-native-pipeline-removal.md`.

**RECORDING_DEBUG_FINDINGS.md + DEBUGGING_GUIDE.md are built on the dead JSON
transport:**
- `RECORDING_DEBUG_FINDINGS.md:53-209` — JSON-IPC throughput bottleneck,
  `Array.from(imageData.data)`, 480x270 / 960x540 scaling tables, backend log line
  `"Received first frame: 960x540, 2073600 bytes"`.
- `DEBUGGING_GUIDE.md:10-16` — "Increased recording scale from 1/6 to 1/2 ...
  960x540"; `:185-200` — "Backend Logging ... `Received first frame: 960x540 ...`"
  and "check `RECORDING_SCALE` is actually 1/2".
- None of this matches source: `grep -rn '960x540\|480x270\|Received first frame'
  frontend/src/ src-tauri/src/` returns **0 matches**; `RECORDING_SCALE` is `1/1`
  (`recording-canvas.tsx:84`).

**initial-development-instructions** (`/initial-development-instructions`, root,
unlabeled bootstrap prompt): instructs creating `src-tauri/src/audio.rs` (`:39`),
`frontend/src/components/Recorder.tsx` (`:54-62`), adding `cpal`/`scrap`/`ffmpeg-next`
(`:34-38`), and wiring commands in `main.rs` (`:47-48`). None of `audio.rs` /
`Recorder.tsx` exist; wiring lives in `lib.rs:89-100`, not `main.rs`.

**No accurate top-level architecture doc exists.** Root has no `ARCHITECTURE.md`
or `CONTRIBUTING.md`. The only correct architecture is spread across 16
`ai-dev/doc/*.md` topic files (`legacy-native-pipeline-removal`,
`native-screen-transport-raw-bytes`, `native-system-audio-capture`,
`trim-editor-v2-multisegment`, `next-implementations`, …). `README.md:113-131`
links nothing into them.

**Note:** the cross-reference `[[mp4-muxer-to-mediabunny-migration]]` does **not**
yet exist as an `ai-dev/doc/*.md` file (`ls ai-dev/doc/mp4-muxer-to-mediabunny-migration.md`
→ not found); it lives only as the memory note `mp4-muxer-deprecated-mediabunny`.
ARCHITECTURE.md should therefore name `mp4-muxer` as the **current** muxer and
forward-reference the migration as planned, not landed.

## Proposed design

Doc-only, surgical. Three concrete deliverables, one retirement action. No code
touched, no symbols introduced. Every edited doc must, after editing, contain
**zero** references to `external_recorder`, `ffmpeg`, `RECORDING_SCALE`, `480x270`,
or `960x540`.

### 1. Rewrite `README.md` (accurate stack, features, dev/build incl. DYLD/Swift)

Keep the existing badge/quick-start skeleton; replace the two wrong sections and
add the missing macOS prerequisite + an ARCHITECTURE.md link. Concrete replacement
content:

**Replace Features (`README.md:16-24`) with:**

```markdown
## ✨ Features

- 🎙️ **System + Mic Audio** — Microphone via `getUserMedia`; system / per-app
  audio captured natively through ScreenCaptureKit (macOS 14+)
- 🖥️ **Native Screen & Region Capture** — Full-display or cropped-region capture
  via ScreenCaptureKit, streamed to the UI over a Tauri Channel
- 🧩 **Multi-Section Composition** — Composite multiple screen/camera sources into
  one canvas, encoded in-browser with WebCodecs (H.264)
- ✂️ **Lossless Trim Editor** — Post-record trimming via `mediabunny`
  packet-copy (no re-encode)
- 🍎 **macOS 14+** — ScreenCaptureKit-based; non-macOS builds fall back to
  `cpal`/`scrap` stubs (not the supported path)
```

**Replace Key Libraries (`README.md:133-139`) with:**

```markdown
### 📚 Key Libraries

| Library              | Where      | Purpose                                                       |
| -------------------- | ---------- | ------------------------------------------------------------- |
| `screencapturekit`   | Rust       | Native screen + system-audio capture (macOS 14+)             |
| WebCodecs (`VideoEncoder`) | Frontend | In-browser H.264 encode of the composite canvas         |
| `mp4-muxer`          | Frontend   | Muxes encoded frames to MP4 (live recording path)            |
| `mediabunny`         | Frontend   | Demux/encode for the lossless trim editor                    |
| `tauri-plugin-updater` | Rust     | In-app auto-update via GitHub releases                        |
| `cpal` / `scrap`     | Rust       | Non-macOS fallback stubs only (cfg-gated; not the live path) |
```

**Add to the macOS "System Dependencies" block (after `README.md:43`):**

```markdown
ScreenCaptureKit uses Swift interop, so the dynamic linker must see the Swift
runtime. The `npm run tauri` script (and `dev.sh`) inject this automatically:

    DYLD_LIBRARY_PATH=/usr/lib/swift

Always launch via `npm run tauri dev` / `npm run tauri build` (or `./dev.sh`),
not a bare `tauri` — otherwise the screencapturekit bindings fail to load.
Requires macOS 14+ and a Swift toolchain (ships with Xcode / Command Line Tools).
```

**Add an architecture pointer** (near the Tech Stack heading, ~`README.md:106`):
`> See [ARCHITECTURE.md](./ARCHITECTURE.md) for the live recording/capture/save
data flow.`

### 2. Add top-level `ARCHITECTURE.md`

New file, ~80-120 lines, the single source of truth at root. Outline:

```markdown
# Architecture

## Recording data flow (live path)
- Sources (native screen, region, camera) → composited on a canvas in
  `frontend/src/components/asmr-recorder/recording-canvas.tsx`
- Canvas frames → WebCodecs `VideoEncoder` (H.264) → muxed with `mp4-muxer`
  (recording-canvas.tsx:9). [Forward note: a planned migration would move muxing
  to `mediabunny` to unify with the trim editor — see
  [[mp4-muxer-to-mediabunny-migration]] (not yet written).]
- Encoded MP4 bytes → `save_media_recording` over a RAW-BYTE Tauri IPC
  (`src-tauri/src/lib.rs:14-20`, `InvokeBody::Raw`) → written to disk. No base64,
  no ~23-min string-length cap.

## Capture (Rust / ScreenCaptureKit)
- `screen_stream.rs` — SCK frames → JPEG → frontend over a Tauri `Channel`.
- `system_audio_stream.rs` — SCK system/per-app audio → PCM over a `Channel`.
- `screen.rs` / `system_audio.rs` — config + cfg-gated platform module selection
  (macos → screencapturekit; non-macos → fallback stubs).
- `recording.rs` — device/app enumeration only (`list_audio_apps`).
- Wiring + the save command live in `lib.rs`; `main.rs` is a 4-line shim.

## Why native capture (WKWebView constraint)
- `getUserMedia` (mic) works in WKWebView; `getDisplayMedia` (screen + system
  audio) is BLOCKED → hence the native SCK pipeline. (memory:
  webkit-getusermedia-works-getdisplaymedia-blocked)

## Trim editor
- `trim-editor.tsx` uses `mediabunny` (dynamic import) for lossless packet-copy
  trimming; export reuses the same raw-byte `save_media_recording`.

## What was removed
- The legacy native Rust recording pipeline (manager/encoder/compositor/
  external_recorder/webcam/cpal/nokhwa/ffmpeg) was DELETED — see
  [[legacy-native-pipeline-removal]]. There is exactly ONE recording path today.

## Deeper topic docs
- Link table into ai-dev/doc/: [[native-screen-transport-raw-bytes]],
  [[native-system-audio-capture]], [[trim-editor-v2-multisegment]],
  [[auto-update]], [[next-implementations]], [[legacy-native-pipeline-removal]].
```

### 3. Retire the three recording docs + the bootstrap prompt

**Recommended: MOVE, don't delete.** Create `docs/archive/` and move the four
historical files there, each prepended with a one-line banner:

```markdown
> **HISTORICAL / OBSOLETE.** Describes the deleted native FFmpeg/JSON-IPC
> recording pipeline. The live path is frontend WebCodecs + mp4-muxer + raw-byte
> save IPC — see /ARCHITECTURE.md. Kept for history only.
```

Files to move:
- `RECORDING_IMPLEMENTATION.md` → `docs/archive/RECORDING_IMPLEMENTATION.md`
- `RECORDING_DEBUG_FINDINGS.md` → `docs/archive/RECORDING_DEBUG_FINDINGS.md`
- `DEBUGGING_GUIDE.md` → `docs/archive/DEBUGGING_GUIDE.md`
- `initial-development-instructions` → `docs/archive/initial-development-instructions.md`
  (add `.md` extension + the banner; it is the project's origin prompt).

Rationale (CLAUDE.md §2/§3): `git mv` preserves history and the debugging context
without an irreversible delete, and the banner makes mis-trust impossible. `docs/`
already exists (`docs/logs/`), so `docs/archive/` is the natural home; using it
(rather than `ai-dev/`) keeps human-history docs separate from the AI-planning
corpus in `ai-dev/doc/`. Outright delete is the alternative if you want a clean
root — but the banner+move costs nothing and the only downside of keeping them is
disk space.

## Implementation steps

1. **Rewrite README Features + Key Libraries.** Replace `README.md:16-24` and
   `:133-139` with the §1 content. **verify:** `grep -nE
   'cpal.*High-Fidelity|scrap.*Screen Capture|ffmpeg-next' README.md` returns
   nothing except the intentional "fallback stubs only" row.
2. **Add the DYLD/Swift prerequisite + ARCHITECTURE link to README.** Insert the
   §1 macOS-dependency note after `:43` and the architecture pointer near `:106`.
   **verify:** `grep -n 'DYLD_LIBRARY_PATH' README.md` and `grep -n
   'ARCHITECTURE.md' README.md` each return ≥1.
3. **Write `ARCHITECTURE.md`** at repo root per the §2 outline, citing real
   files/lines (recording-canvas.tsx:9, lib.rs:14-20). **verify:** `test -f
   ARCHITECTURE.md`; `grep -nE 'external_recorder|ffmpeg|RECORDING_SCALE|480x270|
   960x540' ARCHITECTURE.md` returns **0**.
4. **Create `docs/archive/` and move the four docs with banners.** `git mv` each
   file, prepend the §3 banner. **verify:** `ls docs/archive/` lists all four;
   `ls RECORDING_IMPLEMENTATION.md RECORDING_DEBUG_FINDINGS.md DEBUGGING_GUIDE.md
   initial-development-instructions 2>&1` shows "No such file" for all (they're
   gone from root).
5. **Repo-wide stale-symbol sweep (excluding archive + build cache + logs).**
   `grep -rnE 'external_recorder|ffmpeg-next|RECORDING_SCALE|480x270|960x540'
   README.md ARCHITECTURE.md` returns **0**. (The moved docs legitimately still
   contain these terms under their HISTORICAL banner; `src-tauri/target/` and
   `docs/logs/`, `ai-dev/dev-logs/` are out of scope.) **verify:** zero hits in
   the two live files.

## Interface / API changes

None. Docs only. No code, no build config, no dependency, no public symbol changes.

## Edge cases & risks

- **Stale build cache still contains `ffmpeg`.** `src-tauri/target/` holds 812
  `ffmpeg` fingerprint hits from the deleted pipeline. These are not source and
  are `.gitignore`d (`target/`); leave them. A `cargo clean` would clear them but
  is out of scope for a docs task — do not couple it in.
- **Moving `DEBUGGING_GUIDE.md` could break an inbound link.** `grep -rn
  'DEBUGGING_GUIDE\|RECORDING_IMPLEMENTATION\|RECORDING_DEBUG_FINDINGS' --include
  '*.md' .` (excl. node_modules/.git) before moving; fix any live references
  (none expected outside the docs themselves and logs).
- **`[[mp4-muxer-to-mediabunny-migration]]` is a dangling wikilink** until that
  plan is written. Acceptable — it signals intended future work and matches the
  memory note. Name `mp4-muxer` as the current muxer so ARCHITECTURE.md is true
  today regardless of when the migration lands.
- **Over-claiming cross-platform.** The old README said "Works on macOS, Windows,
  and Linux". The honest claim is macOS 14+ first-class; non-macOS compiles via
  cfg-gated fallback stubs but is not the supported path. The §1 Features wording
  reflects this — don't restore the cross-platform claim.

## Testing & verification

- **Build gates (auto):** None apply meaningfully — this is a docs-only change and
  touches no file under `frontend/` or `src-tauri/src/`. (`npx tsc -b`, `npx vite
  build`, `npm run test`, `cargo build` are unaffected and need not be run for
  this task; if run anyway, they must show no new errors vs baseline.)
- **Doc-review gates (the real verification):**
  - `grep -rnE 'external_recorder|ffmpeg-next|RECORDING_SCALE|480x270|960x540'
    README.md ARCHITECTURE.md` → **0 hits**.
  - `test -f ARCHITECTURE.md && test -d docs/archive` → present.
  - Root no longer contains the four obsolete files (`ls` shows them only under
    `docs/archive/`).
  - Each archived file's first line is the HISTORICAL banner.
- **Manual read-through:** confirm README Key Libraries lists screencapturekit /
  WebCodecs / mp4-muxer / mediabunny and marks cpal/scrap as fallback-only, and
  that the DYLD/Swift note is present. Confirm ARCHITECTURE.md's file:line cites
  resolve (recording-canvas.tsx:9, lib.rs:14-20).

## Out of scope / future

- **`CONTRIBUTING.md`** — solo portfolio repo; no contributors. Don't add one
  (CLAUDE.md §2). Add only if/when the repo opens to contributors.
- **Consolidating the 16 `ai-dev/doc/*.md` topic files into ARCHITECTURE.md** —
  no; ARCHITECTURE.md should *link* to them, not absorb them. They are
  per-feature plans, not architecture reference.
- **Writing the `[[mp4-muxer-to-mediabunny-migration]]` plan** — separate task;
  this doc only forward-references it.
- **`cargo clean` to purge stale ffmpeg target artifacts** — unrelated to docs;
  do not couple.
- **Auto-generated docs / a docs site / badge churn** — speculative; the existing
  badge block stays as-is.

## References

- `README.md` — Features (16-24), macOS deps (38-45), Tech Stack (106-131), Key
  Libraries table (133-139), commands (143-148).
- `src-tauri/Cargo.toml` — `screencapturekit` macOS dep (45-46), `cpal` non-macOS
  cfg-gate (48-50), `scrap` non-macos-non-windows cfg-gate (55-56);
  `ffmpeg-next` absent (verified by grep).
- `src-tauri/src/lib.rs` — `save_media_recording` raw-byte IPC (14-20), module
  list (3-7), handler wiring (89-100).
- `src-tauri/src/main.rs` — 4-line shim (4-6).
- `src-tauri/src/screen.rs` — platform module cfg-selection (98-114), `list_displays`
  (120-174). `src-tauri/src/system_audio.rs` — cfg-selection (31-41).
- `frontend/package.json` — `react ^19.2.0` (32), `mediabunny ^1.46.0` (30),
  `mp4-muxer ^5.2.2` (31), `vite ^7.2.4` (52), `vitest ^4.1.8` (53).
- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — `mp4-muxer` import
  (9), `RECORDING_SCALE = 1 / 1` (84, 251-252).
- `frontend/src/components/asmr-recorder/trim-editor.tsx` — `mediabunny` dynamic
  import (5, 74, 297).
- Root `package.json` — `tauri` script DYLD injection; `dev.sh` — execs
  `npm run tauri dev`.
- `RECORDING_IMPLEMENTATION.md` — modes/FFmpeg (1-21, 36), data-flow (100-106),
  dead `external_recorder.rs` cite (119).
- `RECORDING_DEBUG_FINDINGS.md` — JSON-IPC + 480x270/960x540 (53-209).
- `DEBUGGING_GUIDE.md` — 960x540 scale (10-16), backend log + `RECORDING_SCALE`
  (185-200).
- `initial-development-instructions` — bootstrap prompt (audio.rs 39, Recorder.tsx
  54-62, cpal/scrap/ffmpeg-next 34-38, main.rs wiring 47-48).
- Related plans/notes: [[legacy-native-pipeline-removal]],
  [[native-screen-transport-raw-bytes]], [[native-system-audio-capture]],
  [[trim-editor-v2-multisegment]], [[auto-update]], [[next-implementations]];
  forward-ref [[mp4-muxer-to-mediabunny-migration]] (not yet written; see memory
  note `mp4-muxer-deprecated-mediabunny`).
