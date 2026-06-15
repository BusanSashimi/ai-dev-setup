# Chore: Remove the dead native recording pipeline (+ drop ffmpeg/cpal/nokhwa)

**Priority:** P2  ·  **Effort:** L  ·  **Risk:** med  ·  **Surface:** backend + a little frontend

## Problem / motivation

The repo carries **two** recording implementations. Only one is reachable:

- **Live:** the frontend WebCodecs pipeline. The record button in `toolbar.tsx` calls
  `startExternalRecording` (line 81) / `stopExternalRecording` (75, 108), which just flips
  React state (`recording-context.tsx:199-228`, **no backend invoke**); `RecordingCanvas`
  does the 2×2 composite → H.264 → `mp4-muxer` and saves via
  `invoke("save_media_recording", buffer)`.
- **Dead:** a native Rust pipeline — `manager.rs` (RecordingManager) → `compositor.rs`,
  `encoder.rs` (FFmpeg), `webcam.rs` (nokhwa), `audio.rs` (cpal mic),
  `system_audio_fallback.rs` — driven by the `start_recording`/`stop_recording`/
  `start_external_recording`/`receive_video_frame*` commands.

The native path's only frontend callers are **never mounted**:
- `frontend/src/components/Recorder.tsx` (`invoke('start_audio_capture')` :10,
  `invoke('start_screen_capture')` :21) — not imported anywhere (`App.tsx` mounts only
  `ASMRRecorder`).
- `frontend/src/hooks/use-recording.ts` (`invoke("start_recording")` :64) — not imported
  anywhere.
- `recording-context.tsx`'s `startRecording`/`stopRecording` (149-177) — exposed on the
  context but called by **no mounted component** (verified: only `Recorder.tsx` /
  `use-recording.ts` reference them, both dead).

So `manager.rs`, `encoder.rs`, `compositor.rs`, `external_recorder.rs`, `webcam.rs`,
`audio.rs` (mic), and `system_audio_fallback.rs` are unreachable in the shipping app, yet
they:
- pull heavy native deps — **`cpal`**, **`nokhwa`**, and **`ffmpeg-next`** (the `ffmpeg`
  feature that **fails to link on this arm64 machine** and is a **notarization footgun** via
  Homebrew dylibs — see `[[packaging-and-distribution]]`);
- inflate `cargo build` time and the binary;
- and obscure the real architecture for anyone reading the backend.

Removing them de-risks the distributable build and clarifies the codebase. This is **pure
deletion of confirmed-dead code** — no behavior change — but it's wide, so it's gated behind
a careful per-command reachability audit (some commands are *not* dead; see below).

## Current state (grounded)

**`src-tauri/src/lib.rs` `generate_handler!` (line 179-204) — classify each command:**

| Command | Module | Reachable? | Evidence |
|---|---|---|---|
| `greet` | lib.rs | dead | scaffolding; no caller |
| `audio::start_audio_capture` | audio.rs | **dead** | only `Recorder.tsx:10` (unmounted) |
| `screen::start_screen_capture` | screen.rs | **dead** | only `Recorder.tsx:21` (unmounted) |
| `screen::check_screen_recording_permission` | screen.rs | **keep** | screen permission UX (verify caller) |
| `screen::list_displays` | screen.rs | **keep** | native screen picker (live) |
| `recording::get_available_devices` | recording.rs | **verify** | `recording-context.tsx:76` calls it on mount → `devices` state |
| `recording::get_recording_status` | recording.rs | **dead** | no caller |
| `recording::get_recording_status_live` | recording.rs | **dead** | `recording-context.tsx:127` only in the `!isExternalRecording` branch, which the live (external) path never hits |
| `recording::start_recording` | recording.rs→manager | **dead** | callers unmounted (above) |
| `recording::stop_recording` | recording.rs→manager | **dead** | callers unmounted |
| `start_external_recording` | lib.rs→external_recorder | **dead** | no live caller (frontend `startExternalRecording` is local state only) |
| `receive_video_frame` | lib.rs→external_recorder | **dead** | no caller |
| `receive_video_frame_base64` | lib.rs→external_recorder | **dead** | no caller |
| `stop_external_recording` | lib.rs→external_recorder | **dead** | no caller |
| `get_external_recording_status` | lib.rs→external_recorder | **dead** | no caller |
| `save_media_recording` | lib.rs | **keep** | the live save path (raw-body IPC) |
| `screen_stream::start_screen_stream` | screen_stream.rs | **keep** | live native screen |
| `screen_stream::stop_screen_stream` | screen_stream.rs | **keep** | live native screen |
| `screen_stream::ack_screen_frame` | screen_stream.rs | **keep** | live native screen |

**Modules and the deps they own (`Cargo.toml`):**
- `audio.rs` — cpal mic capture; **also defines `AudioChunk`** (consumed by
  `system_audio_macos.rs`, which we KEEP — see Risks). cpal (`Cargo.toml:27`).
- `webcam.rs` — nokhwa (`Cargo.toml:30`, `input-native`).
- `encoder.rs` — `ffmpeg-next` behind `feature = "ffmpeg"` (`Cargo.toml:33,55`), plus a
  non-ffmpeg PNG-dump fallback.
- `compositor.rs` — uses `image` (**which we KEEP** — `screen_stream.rs` JPEG-encodes with
  it). Removing compositor does **not** let us drop `image`.
- `manager.rs`, `external_recorder.rs`, `recording.rs` — orchestration/commands.
- `system_audio_fallback.rs` — cpal loopback (non-macOS); `system_audio_macos.rs` is the
  macOS impl we KEEP (needed by `[[native-system-audio-capture]]`).

**Live native code that must stay:** `screen.rs`, `screen_macos.rs`, `screen_stream.rs`
(native screen), `system_audio_macos.rs` + `AudioChunk` + `SystemAudioCaptureConfig` (for
`[[native-system-audio-capture]]`), `save_media_recording` (lib.rs).

**Rust unit tests live in some doomed modules:** `compositor.rs`, `webcam.rs`,
`audio_mixer.rs` carry `#[test]`s; deleting the modules deletes those tests (acceptable —
they cover dead code). `screen.rs` tests stay.

## Proposed design

A **two-phase, audit-first** removal so nothing live breaks:

**Phase 0 — confirm reachability (no deletion).** For each "verify"/"keep" row above, grep
the frontend for the exact `invoke("…")` string and confirm whether a **mounted** component
calls it. Specifically resolve:
- `recording::get_available_devices` — is `devices` (from `fetchDevices`, context line 74-82,
  called on mount line 110-113) actually consumed by any rendered UI (e.g.
  `camera-select-modal.tsx`)? If not, it and its caller are also dead.
- `screen::check_screen_recording_permission` — confirm its caller is live before keeping.

**Phase 1 — delete the confirmed-dead set.** Remove modules, commands, frontend files, and
deps that Phase 0 confirms unreachable. Keep the live set. Drop `cpal`, `nokhwa`,
`ffmpeg-next` + the `ffmpeg` feature once their last users are gone.

The cut is "all dead or nothing per item" — if Phase 0 shows `get_available_devices` is live,
keep `recording.rs`'s device-enumeration command (and only that) and excise the rest.

## Implementation steps

1. **Phase 0 audit.** Grep every `generate_handler!` command name across `frontend/src`;
   for each hit, trace whether the calling component/hook is reachable from `App.tsx →
   ASMRRecorder`. Produce the final keep/delete list (start from the table above).
   **verify:** a written list; no two entries contradict the live record/trim/save flow.

2. **Relocate the SCK-audio building blocks** (prereq for keeping system audio working).
   Move `AudioChunk` and `SystemAudioCaptureConfig` into a slim surviving module (e.g. keep
   a trimmed `system_audio.rs` that defines them + re-exports `system_audio_macos`), so
   removing `audio.rs` doesn't break `system_audio_macos.rs`. **Coordinate with
   `[[native-system-audio-capture]]`** — ideally that lands first and already owns these
   types. **verify:** `cargo build` clean with `system_audio_macos` still compiling.

3. **Delete dead frontend.** Remove `frontend/src/components/Recorder.tsx` and
   `frontend/src/hooks/use-recording.ts` (both unmounted). In `recording-context.tsx`, remove
   the now-orphaned `startRecording`/`stopRecording` (149-177) and their context-value
   entries — **only if** Phase 0 confirmed no live caller; keep `startExternalRecording`/
   `stopExternalRecording` (the live path) and `fetchDevices` iff `devices` is used.
   **verify:** `npx tsc -b` clean; record/trim/save still works in dev.

4. **Delete dead backend modules + commands.** Remove the confirmed-dead `mod`s
   (`manager`, `encoder`, `compositor`, `external_recorder`, `webcam`, `audio` [after step 2],
   `system_audio_fallback` on macOS-only scope) and their `generate_handler!` entries and
   `.manage(...)` state (`ExternalRecorderState`, `RecordingState`). Keep `save_media_recording`,
   `screen*`, `screen_stream*`, and the surviving system-audio module. **verify:** `cargo build`
   (default features) clean, **zero warnings** (no orphaned imports/`use`).

5. **Drop dead dependencies.** Remove `cpal`, `nokhwa`, `ffmpeg-next` from `Cargo.toml` and
   delete the `[features] ffmpeg` block once `encoder.rs` is gone. Re-check `tokio`'s
   `features = ["full", …]` — trim to what the surviving async commands need (optional).
   Leave `image`, `screencapturekit`, `parking_lot`, `crossbeam-channel`, `chrono`, `dirs`,
   `base64` (still used by `receive_video_frame_base64`? — that command is dead, so `base64`
   may also drop; verify no other user). **verify:** `cargo build` clean; `cargo tree` no
   longer lists `cpal`/`nokhwa`/`ffmpeg-sys-next`; build time drops.

6. **Prune `.cargo/config.toml` Homebrew paths (optional).** With `ffmpeg-next` gone, the
   `-L /opt/homebrew/lib` rustflag and `PKG_CONFIG_PATH` are no longer needed (the
   `/usr/lib/swift` `DYLD_LIBRARY_PATH` + build.rs rpath stay — SCK needs them).
   **verify:** clean `cargo build` from a shell without Homebrew env.

7. **Full regression (hybrid).** `npm run tauri dev`: assign sections (native screen +
   mic), record, trim (single + multi-segment), save; confirm `[Backend-MediaRecorder] Saved
   MP4` and a valid `ffprobe` output. **verify:** the live record→trim→save path is byte-for-
   byte unaffected.

## Interface / API changes

- **Removed Tauri commands:** the dead set from the table (`start_audio_capture`,
  `start_screen_capture`, `start_recording`, `stop_recording`, `get_recording_status[_live]`,
  `start_external_recording`, `receive_video_frame[_base64]`, `stop_external_recording`,
  `get_external_recording_status`, `greet`; plus `get_available_devices` iff Phase 0 says dead).
- **Removed files:** `manager.rs`, `encoder.rs`, `compositor.rs`, `external_recorder.rs`,
  `webcam.rs`, `audio.rs` (after relocating `AudioChunk`), `system_audio_fallback.rs`
  (macOS-only scope), `Recorder.tsx`, `use-recording.ts`.
- **Removed deps:** `cpal`, `nokhwa`, `ffmpeg-next` + `[features] ffmpeg` (and maybe `base64`).
- **No change** to the live record/trim/save IPC (`save_media_recording`, `screen_stream::*`).

## Edge cases & risks

- **`AudioChunk` / `SystemAudioCaptureConfig` are shared.** Deleting `audio.rs`/`system_audio.rs`
  naively breaks `system_audio_macos.rs` (kept). Step 2 relocates them first. **This is the
  single most important ordering constraint** — sequence after/with
  `[[native-system-audio-capture]]`.
- **`image` stays.** It's intuitive to drop it with `compositor.rs`, but `screen_stream.rs`
  JPEG-encodes frames with `image`. Do **not** remove it.
- **`get_available_devices` may be live.** It's invoked on mount (context line 76). Confirm
  whether the resulting `devices` is rendered before deleting `recording.rs` wholesale; it may
  need to be kept as a standalone command (or the call removed if `devices` is unused UI state).
- **`screen_fallback.rs` / `screen_windows.rs`** are platform-gated; this chore is macOS-first
  — leave non-macOS files unless Phase 0 explicitly covers them (don't expand scope).
- **House rule: don't delete pre-existing dead code unless asked.** This plan exists *because*
  removal was requested; still, every deletion must trace to the Phase 0 audit, and the diff
  should be reviewable as "remove unreachable module X" per commit (one module per commit eases
  bisect/revert).
- **Tests in deleted modules** disappear with them — fine (they tested dead code). Don't port
  them. `[[test-harness-and-ci]]` adds tests for the *live* logic instead.

## Testing & verification

- **Build gates:** `cargo build` (default features) clean + **zero warnings** after each
  phase; `cargo tree` confirms dropped deps; `npx tsc -b` + `npx vite build` clean.
- **Regression (hybrid):** full record → trim (single + multi-segment) → save in
  `npm run tauri dev`; `ffprobe` the output. The live path must be unchanged.
- **Cannot auto-verify:** that nothing *else* (a future/hidden entry point) used a removed
  command — mitigated by the Phase 0 grep audit and one-module-per-commit granularity.

## Out of scope / future

- Rewriting/keeping any native encode path (the WebCodecs path is the product direction;
  `[[mp4-muxer-deprecated-mediabunny]]` tracks the muxer migration separately).
- Trimming `tokio` features aggressively (optional micro-win).
- Non-macOS platform modules (`screen_windows.rs`, `screen_fallback.rs`,
  `system_audio_fallback.rs` beyond macOS scope).

## References

- `frontend/src/App.tsx` — mounts only `ASMRRecorder` (entry-point reachability root).
- `frontend/src/components/asmr-recorder/toolbar.tsx` — record button → `startExternalRecording`
  (51, 81) / `stopExternalRecording` (75, 108) (the live path).
- `frontend/src/contexts/recording-context.tsx` — `startRecording`/`stopRecording`
  (149-177, dead), `startExternalRecording`/`stopExternalRecording` (199-228, live, no
  backend invoke), `fetchDevices`→`get_available_devices` (74-82, called on mount 110-113),
  `get_recording_status_live` poll (127, dead branch).
- `frontend/src/components/Recorder.tsx` (10, 21) + `frontend/src/hooks/use-recording.ts`
  (64) — unmounted callers of the native commands.
- `src-tauri/src/lib.rs` — `generate_handler!` (179-204), `.manage(...)` (`RecordingState`,
  `ExternalRecorderState`, `ScreenStreamState`).
- `src-tauri/src/{manager,encoder,compositor,external_recorder,webcam,audio,recording,
  system_audio_fallback}.rs` — the dead/heavy modules.
- `src-tauri/src/{screen,screen_macos,screen_stream,system_audio_macos}.rs`,
  `save_media_recording` — the live native code to keep.
- `src-tauri/Cargo.toml` — `cpal` (27), `nokhwa` (30), `ffmpeg-next` (33), `image` (39, KEEP),
  `[features] ffmpeg` (55), `screencapturekit` (58).
- Related: `[[native-system-audio-capture]]` (must keep/relocate `AudioChunk` + SCK audio),
  `[[packaging-and-distribution]]` (ffmpeg feature is a notarization footgun),
  `[[test-harness-and-ci]]`.
