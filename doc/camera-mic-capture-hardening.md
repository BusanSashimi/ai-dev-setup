# Feature: Camera & mic capture hardening (dev + packaged builds)

**Priority:** P1  ·  **Effort:** S  ·  **Risk:** med  ·  **Surface:** both

## Problem / motivation
Webcam + mic capture goes through the browser `getUserMedia` path, but the macOS
bundle is **missing the TCC usage strings** (`NSCameraUsageDescription`,
`NSMicrophoneUsageDescription`). In a packaged `.app`, macOS *requires* those
`Info.plist` keys before it will even show the camera/mic permission prompt; without
them the system can deny access (or terminate the process) the first time
`getUserMedia` touches a device. `src-tauri/tauri.conf.json` only declares
`bundle.macOS.entitlements: "entitlements.plist"`, and `src-tauri/entitlements.plist`
only carries the *sandbox* entitlements `com.apple.security.device.camera` /
`com.apple.security.device.audio-input` — those gate sandbox access, not the TCC
consent dialog, and they are not a substitute for the usage strings.

The 2026-06-13 dev log issued a **correction** worth grounding everything in: in this
WKWebView `getUserMedia` (mic, and by extension camera) **works**, while
`getDisplayMedia` (screen) does not — see
`docs/logs/2026-06-13.md` → "Trim editor GUI verified live + CORRECTION: getUserMedia
(mic) works in this WKWebView" → "CORRECTION to the earlier 'browser-capture is fully
blocked' finding" (a verified recording contained a real mic track,
`aac (LC), 48000 Hz, stereo, 256 kb/s`). So this is not a "make capture work from
scratch" task; it is hardening the path that already works in dev so it also works in a
packaged build, plus tightening device enumeration/selection. The same log's earlier
"Runtime verify attempt + screen-capture diagnosis" entry first flagged the missing
usage strings.

## Current state (grounded)
- **Bundle config** — `src-tauri/tauri.conf.json` `bundle.macOS` block:
  `{ "entitlements": "entitlements.plist", "frameworks": [], "minimumSystemVersion": "10.13" }`.
  No `Info.plist`-equivalent and no usage strings anywhere (a repo-wide grep for
  `NSCameraUsageDescription`/`NSMicrophoneUsageDescription` returns nothing in source or
  config). No `Info.plist` exists under `src-tauri/` (only `entitlements.plist`);
  `src-tauri/gen/schemas/` holds only ACL/capability schemas (`acl-manifests.json`,
  `capabilities.json`, `desktop-schema.json`, `macOS-schema.json`), confirming Tauri
  generates the bundle `Info.plist` at build time.
- **Entitlements** — `src-tauri/entitlements.plist` declares exactly
  `com.apple.security.device.camera` and `com.apple.security.device.audio-input`
  (sandbox device access), but NOT the TCC usage-description strings.
- **Feature-detection guard** — `frontend/src/lib/utils.ts` `hasMediaApi(method)`
  returns whether `navigator.mediaDevices[method]` is a function, for
  `'getUserMedia' | 'getDisplayMedia' | 'enumerateDevices'`. Its doc comment still
  asserts `navigator.mediaDevices is frequently undefined ... especially in dev over
  http://localhost`, which the 2026-06-13 correction shows is **inaccurate for
  getUserMedia/enumerateDevices** in this WKWebView.
- **Device enumeration** — `recording-context.tsx` `fetchBrowserDevices()`: guards on
  `hasMediaApi("enumerateDevices")`, primes permission via
  `navigator.mediaDevices.getUserMedia({ video: true, audio: true })` (stops the tracks
  immediately in the `.then`), then `enumerateDevices()` into `browserDevices`. Called on
  mount (`useEffect` alongside `fetchDevices`) and again when the camera modal opens.
- **Camera selection** — `camera-select-modal.tsx` filters `browserDevices` to
  `kind === "videoinput"`, and on select calls
  `navigator.mediaDevices.getUserMedia({ video: { deviceId: { exact: selectedDeviceId } }, audio: false })`
  (guarded by `hasMediaApi("getUserMedia")`), previews it in a `<video>`, and on confirm
  hands the live stream to `preview.tsx` `handleCameraSelect` via the `onSelectCamera`
  prop (without stopping it).
- **Camera wiring** — `preview.tsx` `handleRecordOption("camera")` opens
  `CameraSelectModal` (`setShowCameraModal(true)`); `handleCameraSelect(deviceId, deviceName, stream)`
  tears down any native screen stream for the section (`stopNativeStreamForSection`),
  wires `track.onended → clearSection`, then `setSectionSource(... "camera" ...)` +
  `setSectionStream(...)`.
- **Mic capture** — `recording-canvas.tsx` `acquireAudioTrack(wantMic, wantSystemAudio)`:
  guards `hasMediaApi("getUserMedia")`, then
  `getUserMedia({ audio: { echoCancellation:false, noiseSuppression:false, autoGainControl:false, channelCount:2, sampleRate:48000 } })`
  (channels/rate via the module constants `AUDIO_NUM_CHANNELS`/`AUDIO_SAMPLE_RATE`),
  reads back `getSettings()` and warns about DSP/mono fallbacks. A second mic consumer is
  `hooks/use-audio-monitor.ts`, gated to `status.isRecording && externalConfig.captureMic`.

## Proposed design
Two thrusts, no IPC-contract changes:

1. **Add the macOS TCC usage strings to the packaged bundle.** Tauri v2's bundler
   auto-discovers a raw `Info.plist` placed next to `tauri.conf.json` (i.e.
   `src-tauri/Info.plist`) **by filename** and **merges** its keys into the generated
   bundle `Info.plist` (a sibling `Info.dev.plist` is merged for dev builds). Add a
   minimal `src-tauri/Info.plist` containing `NSCameraUsageDescription` and
   `NSMicrophoneUsageDescription`. Keep `entitlements.plist` as-is (the sandbox device
   entitlements are still wanted; they are orthogonal to TCC). No `tauri.conf.json` key
   references the plist — placement/filename is the contract — but we will verify the
   merge in the built `.app` (see verify steps).

   ```xml
   <!-- src-tauri/Info.plist (merged into the generated bundle Info.plist) -->
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
     <key>NSCameraUsageDescription</key>
     <string>ASMR Recorder uses the camera to record webcam video into your composite.</string>
     <key>NSMicrophoneUsageDescription</key>
     <string>ASMR Recorder uses the microphone to record audio with your video.</string>
   </dict>
   </plist>
   ```

2. **Solidify enumeration/selection and correct the stale guard doc.** The mount-time
   `getUserMedia({video:true,audio:true})` prime in `fetchBrowserDevices` triggers BOTH
   camera and mic TCC prompts at app launch even when the user only wants the mic — and
   in a packaged build that double-prompt at startup is jarring. Narrow the prime to mic
   only (`{ audio: true }`), since camera labels are only needed once the camera modal is
   open (where the modal already re-calls `fetchBrowserDevices` and the per-device
   `getUserMedia` will prompt for camera at the moment of intent). Also fix the
   `hasMediaApi` doc comment in `utils.ts` to reflect the verified reality
   (getUserMedia/enumerateDevices work in this WKWebView; only getDisplayMedia is absent).

No data-flow or type changes; this is config + a constraint narrowing + a comment fix.

## Implementation steps
1. **Create `src-tauri/Info.plist`** with the two usage strings (block above).
   **verify:** `plutil -lint src-tauri/Info.plist` exits 0 ("OK"). File is tracked by git
   and sits next to `tauri.conf.json`.

2. **Confirm the bundler merges it** — build the `.app` once:
   `npm run tauri build` (the npm `tauri` script injects
   `DYLD_LIBRARY_PATH=/usr/lib/swift` for the SCK Swift bindings; build with default
   features to avoid the known `--features ffmpeg` arm64 link failure documented in the
   2026-06-13 log). **verify (observable):**
   `plutil -p "src-tauri/target/release/bundle/macos/asmr-recorder.app/Contents/Info.plist" | grep -E "NSCameraUsageDescription|NSMicrophoneUsageDescription"`
   prints both strings. This is the single load-bearing check — if the keys are absent
   here, the merge convention didn't take and we fall back (see Edge cases).

3. **Narrow the enumeration prime** in `recording-context.tsx` `fetchBrowserDevices`:
   change the prime `getUserMedia({ video: true, audio: true })` to
   `getUserMedia({ audio: true })`. Leave the `hasMediaApi("getUserMedia")` guard and the
   immediate `track.stop()` cleanup (inside the `.then`) intact. **verify:** with the app
   running (`npm run tauri dev`), only the **mic** TCC prompt appears at startup, not
   camera; `browserDevices` still populates (camera entries appear without labels until
   the camera modal grants camera — acceptable, the modal lists them and labels fill in
   after the first per-device `getUserMedia`). Confirm via the camera modal still showing
   cameras.

4. **Fix the stale `hasMediaApi` doc comment** in `frontend/src/lib/utils.ts` to state
   that `getUserMedia`/`enumerateDevices` are available in this WKWebView and only
   `getDisplayMedia` is absent (ground it in the 2026-06-13 correction). Code unchanged.
   **verify:** `npx tsc -b` exits 0 (no behavior change; comment-only — note there are 4
   pre-existing tsc errors per the log; this change must not add any).

5. **Regression-guard the verified mic + camera record path.** With usage strings present,
   do one record→stop in the running app with a camera section + mic on. **verify
   (observable):** in a `npm run tauri dev` (debug) build the backend stdout logs
   `[Backend-MediaRecorder] Saved MP4: <path> (<N> bytes)` (format string
   `lib.rs:130-135`; debug writes the file under `../test-results/`, release writes to the
   user's Videos dir), and `ffprobe` on that file shows a video track with a non-black
   camera quadrant and an `aac (LC), 48000 Hz, stereo` audio track (matching the prior
   verified result). The trim path is untouched.

## Interface / API changes
- **No Rust command signatures change.** `save_media_recording`,
  `screen_stream::start_screen_stream`, `recording::get_available_devices`, etc. are
  untouched.
- **No TS types change.** `RecordingContextValue`, `ExternalRecordingConfig`,
  `MediaDeviceInfo[]` usage all unchanged.
- **New file:** `src-tauri/Info.plist` (build-time bundle merge; not an API).
- **Behavior change (frontend):** `fetchBrowserDevices` prime constraint
  `{ video:true, audio:true }` → `{ audio:true }` (one prompt at startup instead of two).

## Edge cases & risks
- **Merge convention didn't take / keys absent in built app (step 2 fails).** Tauri v2's
  bundle-`Info.plist` merge is filename-based (`src-tauri/Info.plist`, plus
  `Info.dev.plist` for dev) and is not referenced from `tauri.conf.json` — there is no
  config key in the bundle schema to point at it (the `bundle.macOS` block only exposes
  `entitlements`/`frameworks`/`minimumSystemVersion` and friends; `src-tauri/gen/schemas/`
  here are ACL manifests, not the bundle-config schema, so do not look there for a plist
  key). If `plutil -p` on the bundled `Info.plist` doesn't show the keys, the likely cause
  is a `@tauri-apps/cli` too old to auto-discover the file: upgrade the CLI (it is `^2`)
  and rebuild, and add a matching `src-tauri/Info.dev.plist` if the dev binary also needs
  the strings. Do NOT ship without step 2 passing — that is the whole point of the feature.
- **Dev vs packaged divergence.** `npm run tauri dev` runs the *dev binary*, which uses a
  generated dev `Info.plist`; the usage strings primarily matter for the packaged `.app`
  and for a fresh TCC state. Dev already works (per the log) because the dev binary
  inherits the terminal/Tauri signing context. So the usage strings are verified in the
  **build** (step 2), and the runtime record check (step 5) is the end-to-end proof.
- **TCC caching.** macOS caches the consent decision per bundle identifier
  (`com.asmr-recorder.app`, from `tauri.conf.json` `identifier`). If a prior build was
  denied, the new strings won't re-prompt until reset:
  `tccutil reset Camera com.asmr-recorder.app` and
  `tccutil reset Microphone com.asmr-recorder.app`. Document this in testing.
- **Narrowed prime → missing camera labels.** Dropping `video:true` from the startup prime
  means camera `MediaDeviceInfo.label` is empty until the camera modal grants camera. The
  modal already handles empty labels (`device.label || \`Camera ${index + 1}\``), so this
  degrades gracefully and matches the per-intent prompt UX. Acceptable.
- **No teardown/leak regressions.** The startup prime still calls `track.stop()` on every
  primed track; the modal's `getUserMedia` streams are stopped on cancel/close/unmount
  (`camera-select-modal.tsx` cleanup effects); `acquireAudioTrack` mic stream and
  `use-audio-monitor` are unchanged. No new stream is held open by this change.
- **WKWebView quirk (unchanged).** `getDisplayMedia` remains absent; the native SCK path
  (`startNativeScreenStream` → `screen_stream::start_screen_stream`) is the screen source
  and is not touched.
- **Entitlements interaction.** Leaving `entitlements.plist` in place is correct — the
  usage strings (TCC) and the sandbox device entitlements are complementary, not
  redundant. Removing either could break a sandboxed/notarized build.

## Testing & verification
Hybrid setup (per `dev-log.mdc` and the 2026-06-13 method): launch via
`npm run tauri dev`; the user performs GUI/TCC interactions (grant camera/mic, pick a
camera in the modal, record, stop); the agent captures backend stdout and inspects
output with `ffprobe`/`afinfo`. The frontend console is NOT piped to the terminal, so
all proof is from (a) backend `println!`/`[Backend-MediaRecorder] Saved ...` lines,
(b) `ffprobe` on files in `test-results/` (debug build), and (c) `plutil`/`plutil -p`
on the plists.

Auto-verifiable: `plutil -lint` (step 1); `plutil -p` on the bundled `Info.plist`
(step 2); `npx tsc -b` (step 4); `ffprobe` on the recorded MP4 (step 5).

NOT auto-verifiable: the interactive TCC dialog itself and the in-modal camera preview
(macOS Tauri can't be driven programmatically) — these are the user's GUI steps. The
"single mic prompt, not two" outcome (step 3) is user-observed at first launch on a
TCC-reset bundle.

## Out of scope / future
- A mic/camera **device picker** for recording audio (mic capture currently uses the
  default device in `acquireAudioTrack`; no UI to choose a specific mic).
- Hot-plug handling (`navigator.mediaDevices.ondevicechange`) to refresh `browserDevices`.
- Native screen region selection (deferred per the native-screen v1 notes).
- Windows/Linux capture permission equivalents (this plan is macOS-only).
- Notarization / hardened-runtime signing config beyond the existing entitlements.

## References
- Code: `frontend/src/components/asmr-recorder/preview.tsx`
  (`handleRecordOption`, `handleCameraSelect`),
  `frontend/src/components/asmr-recorder/camera-select-modal.tsx`,
  `frontend/src/contexts/recording-context.tsx` (`fetchBrowserDevices`),
  `frontend/src/components/asmr-recorder/recording-canvas.tsx` (`acquireAudioTrack`),
  `frontend/src/lib/utils.ts` (`hasMediaApi`),
  `frontend/src/hooks/use-audio-monitor.ts`.
- Config: `src-tauri/tauri.conf.json` (`bundle.macOS`), `src-tauri/entitlements.plist`,
  new `src-tauri/Info.plist`. Tauri v2 (`tauri = "2"` in `src-tauri/Cargo.toml`); npm
  `tauri` script injects `DYLD_LIBRARY_PATH=/usr/lib/swift`.
- Backend save: `src-tauri/src/lib.rs` `save_media_recording` (raw-body IPC; logs
  `[Backend-MediaRecorder] Saved {EXT}: {path} ({n} bytes)`; debug → `../test-results/`,
  release → Videos dir).
- Native screen path (not touched): `src-tauri/src/screen_stream.rs`
  (`start_screen_stream`/`stop_screen_stream`), `frontend/src/lib/native-screen.ts`
  (ScreenCaptureKit via the `screencapturekit` crate); muxing via `mp4-muxer`
  (deprecated → Mediabunny for trim).
- Dev logs: `docs/logs/2026-06-13.md` — "Runtime verify attempt + screen-capture
  diagnosis" (first flagged the missing usage strings) and "Trim editor GUI verified live
  + CORRECTION: getUserMedia (mic) works in this WKWebView" → "CORRECTION to the earlier
  'browser-capture is fully blocked' finding" (the grounding correction). Prior
  native-screen rollout in `ai-dev/dev-logs/fixing-screen-recording-issue.md`.
