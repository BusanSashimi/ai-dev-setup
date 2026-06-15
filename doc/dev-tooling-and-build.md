# Feature: Dev tooling & build hygiene: fix dev.sh, ffmpeg link fragility, code-splitting

**Priority:** P1  ·  **Effort:** S  ·  **Risk:** low  ·  **Surface:** both

## Problem / motivation
Three independent papercuts that slow every dev/launch cycle and produce a noisy build, none of which touch the already-verified record→trim→save path:

1. **`dev.sh` is broken and never launches Tauri.** Confirmed in `docs/logs/2026-06-13.md` ("Live verification in the running Tauri app"): `cd frontend && npm run dev &` backgrounds the *whole compound* in a subshell, so the parent shell never leaves repo root; the next `cd ../src-tauri` resolves *above* the repo and fails with "No such file or directory". The script also requests `--features ffmpeg`, which doesn't link on this machine (see #2).
2. **`--features ffmpeg` fails to link on this arm64 machine.** Same dev-log entry: `ld` can't resolve the ffmpeg-next native symbols (`_avcodec_*`, `_avformat_*`, `_sws_*`) for arm64 — the known ffmpeg-next distribution fragility, made worse by `.cargo/config.toml` hard-coding `-L /opt/homebrew/lib` (a non-distributable dynamic link). The ffmpeg encoder (`src/encoder.rs`) is part of the **dormant native SCK recording path** (`manager.rs`/`external_recorder.rs`), not the live browser-WebCodecs recording. `npm run tauri dev` already works with **default features (no ffmpeg)** and links in ~7-9s (dev-log: "Trim editor GUI verified live").
3. **Vite emits a >500 kB chunk warning.** Verified by running `npx vite build`: a single `dist/assets/index-*.js` of **~819 kB** (gzip ~238 kB) trips Rollup's default 500 kB `chunkSizeWarningLimit`. Mediabunny (`node_modules/mediabunny` ≈ 9.9 MB on disk) is pulled in eagerly by `trim-editor.tsx`, even though the editor is only needed *after* a recording finishes.

> Note: the exact pre-split chunk size (~819 kB) was from a prior build snapshot — re-measure with `npx vite build` before/after step 3 rather than trusting the literal number.

## Current state (grounded)

**`dev.sh`** (repo root) — full contents:
```bash
#!/bin/bash
# Development script that properly sets DYLD_LIBRARY_PATH for Swift libraries
# This is needed because the screencapturekit crate uses Swift bindings

export DYLD_LIBRARY_PATH="/usr/lib/swift:$DYLD_LIBRARY_PATH"
export PATH="$HOME/.cargo/bin:$PATH"

# Start the frontend dev server in background
cd frontend && npm run dev &          # <-- backgrounds the whole compound
VITE_PID=$!

# Wait for Vite to be ready
sleep 2

# Run the Tauri app
cd ../src-tauri && cargo run --no-default-features --features ffmpeg   # cwd never moved -> fails

# Cleanup
kill $VITE_PID 2>/dev/null
```
The file ends at the `kill` line — there is **no** stray recursive `./dev.sh` invocation. (The original scope mentioned one; it is not present, so there is nothing to remove on that count.) The real bugs are the `cd`+backgrounding and the broken `--features ffmpeg`.

**`package.json`** (root) — the working launcher already injects the env the SCK Swift bindings need:
```json
"tauri": "export PATH=\"$HOME/.cargo/bin:$PATH\"; export DYLD_LIBRARY_PATH=\"/usr/lib/swift:$DYLD_LIBRARY_PATH\"; tauri",
"tauri:dev": "npm run tauri dev"
```
`tauri dev` runs `beforeDevCommand: "npm run dev --prefix frontend"` and loads `devUrl: http://localhost:5178` (from `tauri.conf.json`); vite is pinned to `port: 5178, strictPort: true` (`frontend/vite.config.ts`). This path uses **default cargo features** (no `--features ffmpeg`), so it links.

**`src-tauri/Cargo.toml`** features:
```toml
[features]
default = []
ffmpeg = ["ffmpeg-next"]
```
`ffmpeg-next = { version = "8.0", optional = true }`. The only consumer is `src/encoder.rs`, fully gated: **12** `#[cfg(feature = "ffmpeg")]` blocks plus **2** `#[cfg(not(feature = "ffmpeg"))]` fallbacks. The fallback path (line 132 dispatches to, and line 170 defines, `encode_loop_fallback`) does **not** error out — its doc comment says it "Saves raw frames as images instead of encoding to video," so a default build still compiles and runs (it just writes image frames rather than an encoded video). `encoder` is wired via `mod encoder;` (`lib.rs:11`) and consumed by `manager.rs` (`use crate::encoder::{Encoder, EncoderConfig}`) and `external_recorder.rs` (same import) — the native path. `src-tauri/.cargo/config.toml` forces the homebrew link:
```toml
[target.aarch64-apple-darwin]
rustflags = ["-L", "/opt/homebrew/lib"]

[env]
PKG_CONFIG_PATH = "/opt/homebrew/lib/pkgconfig"
DYLD_LIBRARY_PATH = "/usr/lib/swift"
```
`build.rs` adds only the Swift rpath (`cargo:rustc-link-arg=-Wl,-rpath,/usr/lib/swift`, macOS-only) plus `tauri_build::build()` — nothing ffmpeg-specific.

**Frontend lazy-load target.** `frontend/src/components/asmr-recorder.tsx` statically imports `TrimEditor` (`import { TrimEditor } from "./asmr-recorder/trim-editor"`, line 4) and mounts `<TrimEditor />` once (line 31). `trim-editor.tsx` statically imports mediabunny (lines 5-16):
```ts
import { Input, Output, BlobSource, BufferTarget, Mp4OutputFormat, ALL_FORMATS,
         EncodedPacketSink, EncodedVideoPacketSource, EncodedAudioPacketSource, EncodedPacket } from "mediabunny";
```
**Critical wiring detail:** the editor is event-driven, not prop-driven. `TrimEditor()` registers `window.addEventListener("recordingReadyForEdit", onReady)` (`trim-editor.tsx:175`) in a mount effect; `recording-canvas.tsx:1134` dispatches that `CustomEvent` (with `detail.blob`). The edit blob is captured from `muxer.target.buffer` at line 1129 *before* `saveRecording` reads it, and the event fires at line 1134 *after* `muxer.finalize()` (1123) + `await saveRecording(mp4Buffer)` (1133). So **the listener must already be live when a recording finishes** — naively `React.lazy`-ing the whole `TrimEditor` would mean no listener exists until first mount, and the handoff event would be dropped.

## Proposed design

Three surgical, independent changes. No IPC contracts or TS types change.

**A. Fix `dev.sh`** — keep it as a thin wrapper that defers to the *already-working* npm script, so there's one source of truth for the launch incantation. Replace the buggy manual vite+cargo orchestration with `exec npm run tauri dev` (which injects `DYLD_LIBRARY_PATH` itself via the `tauri` script). This eliminates the `cd`/backgrounding bug, the broken `--features ffmpeg`, and any need to manage `$VITE_PID` (Tauri's `beforeDevCommand` owns vite's lifecycle).

**B. ffmpeg link reconciliation** — make **default (no-ffmpeg) the documented and only-needed launch**, and document the ffmpeg path as opt-in for the dormant native encoder. No `Cargo.toml` change is required (`default = []` is already correct). Add a short "Local development" section to `ai-dev/README.md` stating: launch with `npm run tauri dev`; `--features ffmpeg` is optional, only powers the unused native `encoder.rs`, and currently fails to link on arm64 against `/opt/homebrew/lib` — to use it, install ffmpeg via homebrew and accept the non-distributable dynamic link. This converts an undocumented landmine into a known, opt-in path.

**C. Code-split mediabunny out of the initial bundle.** The simplest correct fix that respects the event-driven wiring: keep the tiny `TrimEditor` *shell* (which owns the `recordingReadyForEdit` listener) in the main bundle, and **lazy-import mediabunny only when it's actually used**. Mediabunny is the heavy dependency, not the React component. Convert the static `import ... from "mediabunny"` into a dynamic `await import("mediabunny")` inside the functions that use it (`trimToBuffer` and the `Input`-building effect). Rollup then splits mediabunny into its own async chunk, dropping the initial JS under 500 kB while keeping the listener live so no handoff is missed.

Before:
```ts
import { Input, Output, /* ... */ } from "mediabunny";
```
After (inside the editor's mediabunny-using code paths):
```ts
const mb = await import("mediabunny");   // resolved on first clip open / export
const input = new mb.Input({ formats: mb.ALL_FORMATS, source: new mb.BlobSource(blob) });
```

## Implementation steps

1. **`dev.sh`** — replace the body (keep the shebang) with:
   ```bash
   #!/bin/bash
   # Launch the dev app. `npm run tauri dev` runs vite (beforeDevCommand) and the
   # Tauri shell with DEFAULT cargo features; the npm `tauri` script injects
   # DYLD_LIBRARY_PATH=/usr/lib/swift for the screencapturekit Swift bindings.
   exec npm run tauri dev
   ```
   Remove the `cd frontend`, the `&` backgrounding, `VITE_PID`/`sleep`/`kill`, and the `--features ffmpeg` cargo invocation. **verify:** `bash dev.sh` from repo root reaches the Tauri compile/launch (vite serves :5178, `cargo` links with default features, the WKWebView window opens). Confirm `dev.sh` no longer errors with "No such file or directory" and no longer requests `--features ffmpeg` (`grep ffmpeg dev.sh` returns nothing).

2. **Docs (ffmpeg path)** — add a "Local development / build" section to `ai-dev/README.md` (it already exists) documenting: (a) `npm run tauri dev` is the launch command (default features, no ffmpeg); (b) `--features ffmpeg` only powers the dormant native `src/encoder.rs` path and is **not needed** for the live browser-WebCodecs recording/trim/save flow — without it the encoder compiles via its `#[cfg(not(feature = "ffmpeg"))]` fallback (`encode_loop_fallback`); (c) `--features ffmpeg` currently fails to link on arm64 (`ld: symbol(s) not found: _avcodec_*`/`_avformat_*`/`_sws_*`) because `.cargo/config.toml` points `-L` at `/opt/homebrew/lib`; to use it install `ffmpeg` via homebrew, and note the resulting dynamic link isn't distributable. **verify:** the section names the real symbols/paths and a fresh reader can launch without hitting either trap; cross-check with `cargo build --manifest-path src-tauri/Cargo.toml` (default) succeeding and `--features ffmpeg` reproducing the link error.

3. **`trim-editor.tsx` — drop the static mediabunny import.** Delete the top-level `import { ... } from "mediabunny"` (lines 5-16). The mediabunny *values* are used in exactly two places: the `Input`-building `useEffect` (`trim-editor.tsx:186`) and `trimToBuffer` (the export, lines 45-128). In each, add `const mb = await import("mediabunny");` at the top and prefix the symbols (`mb.Input`, `mb.BlobSource`, `mb.ALL_FORMATS`, `mb.Output`, `mb.BufferTarget`, `mb.Mp4OutputFormat`, `mb.EncodedPacketSink`, `mb.EncodedVideoPacketSource`, `mb.EncodedAudioPacketSource`, `mb.EncodedPacket`). For type-only references (`Input`, `EncodedPacketSink` in the `inputRef`/`videoSinkRef` `useRef<…>` generics at lines 146-147, and `trimToBuffer`'s `input: Input` param + the `EncodedPacket | null` local at line 80), use `import type { Input, EncodedPacketSink, EncodedPacket } from "mediabunny"` — type imports are erased at build time and don't pull the runtime module into the main chunk. The `Input`-building effect already returns a cleanup; make its async body resolve `mb` then build the input (guard against the effect being torn down before the dynamic import resolves — null-check `blob`/refs after the await). **verify:** `npx tsc -b` exits 0 on the file (type imports satisfy the generics); `npx vite build` succeeds and reports the main `index-*.js` chunk **< 500 kB** with a separate `mediabunny`-content async chunk listed, and **no** ">500 kB" warning. `npx eslint .` clean.

4. **Confirm the listener stays eager.** Do **not** wrap `TrimEditor` in `React.lazy`. Leave `asmr-recorder.tsx` mounting `<TrimEditor />` statically (line 31) so the `recordingReadyForEdit` listener registers at app start. (Only the mediabunny *runtime* is deferred, per step 3.) **verify:** read the diff — `asmr-recorder.tsx` is unchanged; `trim-editor.tsx` still calls `window.addEventListener("recordingReadyForEdit", …)` in its top-level mount effect (line 175, not behind any dynamic import).

5. **End-to-end smoke (hybrid).** Launch `npm run tauri dev`; user records a short clip and stops. **verify:** the trim dialog opens (proves the listener fired and the first `await import("mediabunny")` resolved on demand), drag handles + Save works, and the backend logs a `Saved MP4` line. `ffprobe` the trimmed file in `test-results/`: H.264+AAC, `key_frame=1` at frame 0 (lossless packet-copy intact — see dev-log "Trim editor GUI verified live").

## Interface / API changes
None. No Rust command signatures, no TS types, no IPC body shapes change. `save_media_recording` (the raw-bytes `tauri::ipc::Request` command), `start_screen_stream`/`stop_screen_stream`, and `trimToBuffer`'s behavior are all untouched. The `recordingReadyForEdit` CustomEvent contract (`{ detail: { blob: Blob } }`) is preserved.

## Edge cases & risks
- **Lazy-import race (primary risk).** The first `await import("mediabunny")` adds a one-time network/parse latency the *first* time the editor opens or exports. Mitigation: the listener and dialog are eager, so the dialog opens immediately; only the keyframe-lookup `Input` build (already in an async effect) and the export await mediabunny. Add null-guards after the `await` so a clip that's been replaced/closed mid-import doesn't crash. No functional regression — mediabunny was always async-used.
- **Type vs value imports.** If a mediabunny symbol used only as a *type* is left as a value import, Rollup keeps the runtime in the main chunk and the split silently no-ops. Verify the chunk actually moved (step 3) — don't trust the diff alone.
- **`dev.sh` env.** `exec npm run tauri dev` relies on the root `package.json` `tauri` script for `DYLD_LIBRARY_PATH`/`PATH`; if someone runs `tauri dev` directly without it, the SCK Swift bindings fail to load at runtime. The comment in `dev.sh` documents this. (`build.rs` already sets the rpath, so release bundles are unaffected.)
- **No regression to the verified path.** None of the three changes touch `recording-canvas.tsx`'s encode/mux/save logic, the audio esds/CBR fixes, or `lib.rs`. The trim algorithm bytes are identical (only *when* mediabunny loads changes).
- **WKWebView quirk (informational).** The editor's late-`durationchange` handling (`adoptDuration` on `loadedmetadata`+`durationchange`) and `isExporting` reset (dev-log) are unrelated to load timing and remain intact.

## Testing & verification
Launch via `npm run tauri dev` (default features). The user performs the GUI/TCC steps (native screen capture grant, record, drag handles, Save) — these can't be driven programmatically on macOS. Auto-observable: backend `println!`/`Saved MP4` stdout, and `ffprobe`/`ffmpeg -c copy -f null` on the `test-results/*.mp4` outputs (use `-c copy` to read true container DTS — plain `-f null` re-derives and lies for VFR, per the dev-log correction). Build-side checks are fully auto-verifiable: `npx tsc -b`, `npx eslint .`, `npx vite build` (assert main chunk < 500 kB and no size warning), and `bash dev.sh` reaching launch. **Cannot auto-verify:** the trim dialog UI and the first lazy-import latency in the live webview (frontend console isn't piped to the terminal) — covered by the hybrid smoke in step 5.

## Out of scope / future
- Fixing the ffmpeg link itself (statically linking ffmpeg or removing the homebrew `-L`) — the native encoder path is dormant; document, don't fix.
- Further code-splitting (vendor/React chunk via `manualChunks`) — only mediabunny is needed to clear the warning; don't add speculative chunking.
- The 4 pre-existing tsc errors in untouched files (`browser-recorder.tsx` dead plugin imports, `region-selector.tsx` `HANDLE_SIZE`, `recording-context.tsx` `SectionConfig`) — unrelated, flagged in prior logs.
- Raising `build.chunkSizeWarningLimit` instead of splitting — rejected; splitting is the real fix and keeps initial load lean.

## References
- `dev.sh`, `package.json` (root `tauri`/`tauri:dev` scripts), `src-tauri/Cargo.toml` (`[features] default=[]`, `ffmpeg=["ffmpeg-next"]`), `src-tauri/tauri.conf.json` (`beforeDevCommand`, `devUrl`), `src-tauri/.cargo/config.toml` (the `-L /opt/homebrew/lib` link + `PKG_CONFIG_PATH`), `src-tauri/build.rs` (Swift rpath).
- `src-tauri/src/encoder.rs` (sole ffmpeg consumer: 12 `#[cfg(feature = "ffmpeg")]` + 2 `#[cfg(not(...))]` fallback blocks; `encode_loop_fallback` at line 171), `src/lib.rs:11` (`mod encoder;`), `src/manager.rs` / `src/external_recorder.rs` (native path that uses `Encoder`/`EncoderConfig`).
- `frontend/vite.config.ts` (port 5178), `frontend/src/components/asmr-recorder.tsx:4,31` (`TrimEditor` import + mount), `frontend/src/components/asmr-recorder/trim-editor.tsx` (mediabunny import lines 5-16 + `recordingReadyForEdit` listener line 175 + `Input` effect line 186 + `trimToBuffer` lines 45-128), `frontend/src/components/asmr-recorder/recording-canvas.tsx:1124-1138` (editBlob capture + event dispatch).
- Libs: mediabunny (`^1.46.0`, MPL-2.0 demux/mux/trim, ≈9.9 MB on disk), ffmpeg-next 8.0, screencapturekit crate, mp4-muxer (deprecated, per `MEMORY.md`).
- Dev logs: `docs/logs/2026-06-13.md` — "Live verification in the running Tauri app" (dev.sh bug + ffmpeg arm64 link failure) and "Trim editor GUI verified live" (`npm run tauri dev` default-features launch, ~8.7s link). Prior bundle context: `ai-dev/webcodecs-implementation.md`, `ai-dev/mediarecorder-implementation.md`.
