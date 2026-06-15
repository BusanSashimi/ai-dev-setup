# Feature: Native screen capture — region selection + multi-display picker

**Priority:** P1  ·  **Effort:** M  ·  **Risk:** med  ·  **Surface:** both

## Problem / motivation
On macOS the desktop app runs in WKWebView, which has no `navigator.mediaDevices.getDisplayMedia`. The native path (`screen_stream::start_screen_stream` → `ScreenCapture` over SCK → JPEG → Tauri `Channel`) was added as the WKWebView substitute, but it only captures the **full primary display**. The browser path has a real `RegionSelector` and a per-section crop; native does not.

The 2026-06-13 dev log calls this out explicitly as the known follow-up:
- `docs/logs/2026-06-13.md` (lines 144-146): *"v1 scope: full-display capture, NO region selection for native (follow-up — would need adapting RegionSelector to native frames)… Display defaults to primary (index 0); multi-display picker is a follow-up."*

So on macOS today a user cannot crop a screen quadrant or pick a second monitor — the single biggest native/browser parity gap.

## Current state (grounded)

**Backend**
- `src-tauri/src/screen.rs`: `ScreenCaptureConfig { fps, display_index }` (no `region` today); `ScreenFrame { data, width, height, stride, timestamp }` with `to_rgba()`. `check_screen_recording_permission()` calls `SCShareableContent::get()` and returns `Ok(!content.displays().is_empty())` — it returns only a `bool`, no per-display dimensions. The legacy `start_screen_capture()` is the place that already reads `display.width()/height()` for the primary display on each platform.
- `src-tauri/src/screen_macos.rs`: `ScreenCapture::new(config)` looks up `content.displays().get(config.display_index)`, stores `width/height` from `display.width()/height()`. `start()` builds `SCContentFilter::create().with_display(display).with_excluding_windows(&[]).build()` and an `SCStreamConfiguration::new().with_width(self.width).with_height(self.height).with_pixel_format(PixelFormat::BGRA)…`. **No source_rect / no crop.**
- `src-tauri/src/screen_stream.rs`: `start_screen_stream(section_index, display_index, fps, max_dimension, on_frame, state)` builds `ScreenCaptureConfig { fps, display_index }`, spawns the worker (latest-wins `recv_timeout`/`try_recv`), `encode_frame()` downscales to `max_dimension` + JPEG into `ScreenStreamFrame { data, width, height, timestamp_ms }`. `StreamHandle` (worker + `Arc<AtomicBool>`) has `stop()` (signal + join) and `Drop` (signal-only, no join). `stop_screen_stream(section_index, state)`. Registered in `src-tauri/src/lib.rs` (`generate_handler![… screen_stream::start_screen_stream, screen_stream::stop_screen_stream]`).

**Frontend**
- `frontend/src/lib/native-screen.ts`: `startNativeScreenStream(sectionIndex, onFrame, opts)` with `NativeScreenOptions { displayIndex?, fps?, maxDimension? }`; `invoke("start_screen_stream", { sectionIndex, displayIndex, fps, maxDimension, onFrame: channel })`; single-in-flight latest-wins decode gate. `stopNativeScreenStream(sectionIndex)`. Also exports the `ScreenStreamFrame` TS type (`data, width, height, timestampMs`).
- `frontend/src/components/asmr-recorder/preview.tsx`: `handleRecordOption("screen")` branches on `hasMediaApi("getDisplayMedia")` → browser `startScreenCapture()` else `startNativeScreen(selectedSection)`. `startNativeScreen` calls `startNativeScreenStream(index, …, { fps: Math.min(externalConfig.frameRate||30, 30), maxDimension: Math.ceil(externalConfig.outputWidth/2) })`, stores the channel in `nativeScreenChannels`, draws each bitmap into `nativeCanvasRefs[index]`, and sets the section label to `"Screen (native)"`. `getSectionSources()` returns `{ type:"canvas", element: nativeCanvasRefs[index] }` for native (checked **before** the `hasRegion`/`sectionRegions` branches, so it takes priority). `stopNativeStreamForSection` tears down channel + native canvas refs. `sectionRegions` + `RegionSelector` are **browser-only**: `handleRegionConfirm` → `startCanvasRendering` and the `{ type:"video", element: sourceVideo, region }` 9-arg `drawImage` crop performed by `drawSection` in `recording-canvas.tsx`.
- `frontend/src/components/asmr-recorder/region-selector.tsx`: `RegionSelectorProps { open, onClose, onConfirm, screenWidth?, screenHeight?, stream? }`; renders a `<video srcObject=stream>` background with a draggable/resizable box; `onConfirm({x,y,width,height})` in **source pixels** (rounded). Already supports the no-stream fallback (grid background, region-selector.tsx:226) — needed for native, which has no `MediaStream`. (Only `MIN_SIZE` is defined here; there is no `HANDLE_SIZE` constant.)

**Crate API available (verified, screencapturekit 1.5.0 — `Cargo.lock` resolves `"1"` to 1.5.0)**
- `SCStreamConfiguration`: `with_source_rect(CGRect)`, `with_destination_rect(CGRect)`, `with_scales_to_fit(bool)`, `with_preserves_aspect_ratio(bool)` (`src/stream/configuration/dimensions.rs`).
- `SCDisplay`: `display_id() -> u32`, `width() -> u32`, `height() -> u32`, `frame() -> CGRect` (`src/shareable_content/display.rs`).
- `screencapturekit::cg::CGRect { x, y, width, height: f64 }`; `CGRect::new(x,y,width,height)` takes four `f64`s. `CGRect` is also re-exported from `screencapturekit::prelude`, which `screen_macos.rs` already glob-imports (`use screencapturekit::prelude::*`), so **no new import is needed** there.

## Proposed design
Two independent additions, both backend-driven (native cropping must happen at the SCK config level — we cannot crop in the browser since native frames arrive pre-downscaled JPEGs with no full-display context):

1. **Display picker** — a new `list_displays` command enumerates `SCShareableContent::get().displays()` → `[{ index, displayId, width, height, isPrimary }]`. The frontend shows them and passes the chosen index into the existing `displayIndex` arg (already plumbed end-to-end).

2. **Region crop** — add an optional `region` to the stream config. `ScreenCaptureConfig` gains `region: Option<CaptureRegion>` (`x,y,width,height` in display pixels). `screen_macos.rs::start()` applies it via `with_source_rect(CGRect::new(x,y,w,h))` and sets `with_width/height` to the **region** size (so downstream `ScreenFrame`/JPEG/composite all see the cropped dimensions; no browser-side cropping needed).

IPC contract change (`start_screen_stream`), before → after:
```rust
// before
async fn start_screen_stream(section_index: u32, display_index: usize, fps: u32,
    max_dimension: u32, on_frame: Channel<ScreenStreamFrame>, state: …) -> Result<(), String>
// after  (region optional → omitting it = full display, unchanged behavior)
async fn start_screen_stream(section_index: u32, display_index: usize, fps: u32,
    max_dimension: u32, region: Option<CaptureRegion>,
    on_frame: Channel<ScreenStreamFrame>, state: …) -> Result<(), String>
```

UI flow on native: the **display picker runs first** (so the region selector knows the chosen display's true pixel size), then `RegionSelector` opens with `stream={null}` (grid fallback) and `screenWidth/Height` = the picked display's `width/height`. `onConfirm` stores into `sectionRegions[index]` and starts the native stream with `{ displayIndex, region }`.

## Implementation steps

1. **`screen.rs` — config type.** Add `#[derive(Clone, Debug)] pub struct CaptureRegion { pub x: u32, pub y: u32, pub width: u32, pub height: u32 }` and `pub region: Option<CaptureRegion>` to `ScreenCaptureConfig` (default `None`). Update the existing `Default` impl. (`Clone + Debug` are required: the region is threaded through a moved config and logged with `{:?}` in step 3.)
   **verify:** `cargo build` (in `src-tauri/`) compiles; `cargo test screen` still passes (no behavior change with `None`).

2. **`screen.rs` — display enumeration command.** Add `#[command] pub fn list_displays() -> Result<Vec<DisplayInfo>, String>` (macOS: map `SCShareableContent::get()?.displays()` to `DisplayInfo { index, display_id, width, height, is_primary }`, where `is_primary = frame().x == 0.0 && frame().y == 0.0`). For non-macOS, return a single synthetic entry built from the primary-display `width()/height()` lookups (the same per-platform primary-display calls already used by the legacy `start_screen_capture` — `check_screen_recording_permission` itself only returns a `bool`, so reuse the dimension lookups, not that fn). `DisplayInfo` derives `Serialize` with `#[serde(rename_all = "camelCase")]`.
   **verify:** add a `#[test]`-guarded `println!` or run a tiny `cargo run`-style probe; on launch the registered command returns ≥1 display. Backend `println!` is observable in the `npm run tauri dev` terminal.

3. **`screen_macos.rs` — apply region.** In `new()`, when `config.region` is `Some(r)` set `self.width = r.width; self.height = r.height` (clamp to display bounds). In `start()`, after `SCStreamConfiguration::new()`, when region present add `.with_source_rect(CGRect::new(r.x as f64, r.y as f64, r.width as f64, r.height as f64))` and keep `.with_width(self.width).with_height(self.height)` at the region size; add `.with_scales_to_fit(false)` so SCK crops rather than scales. Leave the no-region path byte-identical. Extend the existing `start()` log to include the region — do it here (not in `start_screen_stream`), because `self.config.region` is still live in `start()`: `println!("Screen capture started: {}x{} @ {}fps region={:?}", self.width, self.height, self.config.fps, self.config.region)`.
   **verify:** with a region set, the JPEG `ScreenStreamFrame.width/height` (logged in `start_screen_stream`) match the region aspect; full-display path logs unchanged dims.

4. **`screen_stream.rs` — thread region through.** Add `region: Option<CaptureRegion>` param to `start_screen_stream` (between `max_dimension` and `on_frame`); pass into `ScreenCaptureConfig { fps, display_index, region }`. Note: by the time the final start `println!` runs, the config (and therefore `region`) has already been moved into `ScreenCapture::new` and onward into the worker closure, so it cannot be referenced there — the region log belongs in `screen_macos.rs::start()` (step 3). If you want it in the `screen_stream` log line too, `clone()` the region before moving it into the config.
   **verify:** the `screen_macos` start log prints the region; with no region the existing full-display logs are unchanged.

5. **`lib.rs` — register `list_displays`.** Add `screen::list_displays` to `generate_handler!` next to `screen::check_screen_recording_permission`.
   **verify:** `cargo build`; frontend `invoke("list_displays")` resolves (no `unknown command` error in the dev terminal).

6. **`native-screen.ts` — TS surface.** Add `export interface DisplayInfo { index: number; displayId: number; width: number; height: number; isPrimary: boolean }` and `export async function listDisplays(): Promise<DisplayInfo[]>` (`invoke("list_displays")`). Add `region?: { x: number; y: number; width: number; height: number }` to `NativeScreenOptions`; pass it in the `invoke("start_screen_stream", { … region: opts.region ?? null … })` body.
   **verify:** `npm run build` (frontend) type-checks; `tsc` clean.

7. **`preview.tsx` — display picker + native region flow.** In `startNativeScreen(index)`: first `const displays = await listDisplays()`. If `>1`, show a small picker (the codebase already has `Dialog` + `Select` primitives under `components/ui/`, used by `CameraSelectModal`) to choose `displayIndex`; default 0 otherwise. Then call `setScreenDimensions({ width: displays[displayIndex].width, height: displays[displayIndex].height })`, `setPendingStream(null)`, and `setShowRegionSelector(true)` — but route confirm to a **native** handler (track via a `nativeRegionPending` ref/flag) instead of the browser `handleRegionConfirm` (which early-returns when `pendingStream` is null). The native confirm: store `sectionRegions.current[index] = region`, then `startNativeScreenStream(index, …, { displayIndex, region, fps, maxDimension })`. Note: `getSectionSources` still returns `{ type:"canvas", element: nativeCanvasRefs[index] }` for native (its native check runs before the `hasRegion` branch, and the canvas is already region-sized by the backend), so the region badge at preview.tsx:626 — gated on `hasRegion && sectionRegions.current[index]` — will now light up for native too.
   **verify (GUI/TCC, user-driven):** pick a region in WKWebView → the section canvas shows only that region; `ffprobe test-results/<latest>.mp4` shows the composite quadrant filled by the cropped region (correct aspect).

8. **`preview.tsx` — teardown.** `stopNativeStreamForSection(index)` already nulls `nativeScreenChannels`/`nativeCanvasRefs`; additionally clear `sectionRegions.current[index]` there for native (mirrors the browser clear at preview.tsx:485 in `handleClearSection`) so a re-record doesn't reuse a stale crop.
   **verify:** switch a native-region section to camera then back; backend `[screen_stream] section N stopped: X frames sent, 0 errors` prints once per stop (no leaked worker), and the new stream logs the fresh region.

## Interface / API changes
- **Rust new command:** `screen::list_displays() -> Result<Vec<DisplayInfo>, String>`; `DisplayInfo { index: usize, display_id: u32, width: u32, height: u32, is_primary: bool }` (`camelCase` over IPC).
- **Rust changed command:** `start_screen_stream(…, max_dimension: u32, region: Option<CaptureRegion>, on_frame, state)`. `CaptureRegion { x, y, width, height: u32 }` (derives `Clone, Debug`).
- **Rust changed type:** `ScreenCaptureConfig` gains `region: Option<CaptureRegion>`.
- **TS new:** `DisplayInfo`, `listDisplays()`. **TS changed:** `NativeScreenOptions.region?`; IPC body gains `region`.
- Omitting `region` (`None`/`null`) is fully backward-compatible — identical to today's full-display behavior.

## Edge cases & risks
- **Region out of display bounds.** Clamp in `screen_macos.rs::new()` to `[0, display.width()] × [0, display.height()]`; reject zero-area. SCK silently green-frames an invalid `source_rect`, so clamp rather than trust the UI.
- **HiDPI / points vs pixels.** `SCDisplay::width()/height()` are pixels but `frame()` is in points; `source_rect` is interpreted in the filter's coordinate space. `RegionSelector` already works in source pixels for the browser path, and we feed it the display's pixel `width/height`, so keep the region in pixels and confirm against `ScreenStreamFrame.width/height` in step 3. On a Retina display the captured frame may be 2× the point size — verify the first run on the actual hardware and adjust scaling if the crop is half-size.
- **Multi-display TCC.** Screen Recording permission is per-app, not per-display; `list_displays` itself may return empty until granted — surface the same permission message `start_screen_stream` already produces.
- **Teardown/leaks (must not regress).** The `StreamHandle` `stop()`/`Drop` + insert-replace race fixes from 2026-06-13 are untouched; region/display only change config, not lifecycle. Step 8 adds a `sectionRegions` clear to avoid a stale-crop reuse.
- **WKWebView.** `RegionSelector` with `stream={null}` already renders the grid fallback (region-selector.tsx:226) — no `getUserMedia` needed, safe in WKWebView. The drag math uses `window.innerWidth/Height` scaling, independent of any media API.
- **`max_dimension` interaction.** Cropped frames are smaller, so `encode_frame`'s `scale = min(max_dimension/longest, 1.0)` may now be 1.0 (no downscale) — fine, but a tall/narrow region could still exceed budget; the existing cap holds since region ≤ display.

## Testing & verification
Hybrid setup — `npm run tauri dev`; user performs the TCC grant + GUI region drag/display pick (the webview console is **not** piped to the terminal):
1. **Backend log (observable):** confirm `list_displays` count, the `Screen capture started: WxH @ Nfps region=Some(...)` line (from `screen_macos.rs::start()`), and `[screen_stream] section N started: display D, …` + the matching `[screen_stream] section N stopped: X frames sent, 0 errors`.
2. **Output (observable):** `ffprobe -v error -show_entries stream=width,height,codec_name -of default=nw=1 test-results/<file>.mp4` — verify the file muxes and the quadrant containing the native-region section matches the region's aspect (no black bars / no double-crop).
3. **Single + dual monitor:** with one display, picker is skipped (default 0). With two, pick index 1 and confirm the log shows `display 1` and capture matches that monitor's resolution.
4. **Can't auto-verify:** the visual correctness of the crop and the TCC dialog — user confirms in the running app. There's no headless SCK path.

## Out of scope / future
- Window-level capture (`SCContentFilter::with_window`) and app-exclusion.
- Live region re-adjustment while streaming (today region is fixed at start; changing it = stop + restart).
- Windows/Linux region+display parity (`screen_windows.rs`/`screen_fallback.rs` get full-display only here).
- Moving the region preview to show *live native frames* instead of the static grid (would need a transient full-display native stream behind `RegionSelector`).

## References
- Backend: `src-tauri/src/screen_stream.rs`, `src-tauri/src/screen.rs`, `src-tauri/src/screen_macos.rs`, `src-tauri/src/lib.rs`
- Frontend: `frontend/src/lib/native-screen.ts`, `frontend/src/components/asmr-recorder/preview.tsx`, `frontend/src/components/asmr-recorder/region-selector.tsx`, `frontend/src/components/asmr-recorder/recording-canvas.tsx` (`drawSection` 9-arg `drawImage` crop)
- Crate: `screencapturekit` 1.5.0 — `src/stream/configuration/dimensions.rs` (`with_source_rect`/`with_destination_rect`/`with_scales_to_fit`/`with_preserves_aspect_ratio`), `src/shareable_content/display.rs` (`display_id`/`width`/`height`/`frame`), `screencapturekit::cg::CGRect` (also re-exported via `screencapturekit::prelude`). Recorded composite still goes WebCodecs H.264 → mp4-muxer; trim via mediabunny (unchanged by this feature).
- Dev logs: `docs/logs/2026-06-13.md` (native v1 scope + named follow-ups at lines 144-146; teardown/leak fixes), `ai-dev/dev-logs/2026-01-28.md` (browser `RegionSelector` origin).
