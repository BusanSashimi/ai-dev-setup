# Feature: Product completeness / shell polish — honest timeline, dead-button cleanup, branded icon, reveal-in-Finder

**Priority:** P2  ·  **Effort:** M  ·  **Risk:** low  ·  **Surface:** frontend + bundle assets (one new JS dep; no Rust code change)

## Problem / motivation

The core record→save→trim loop works and saves real files (`save_media_recording`, `src-tauri/src/lib.rs:14-71`). But four shell-level gaps make a finished app look unfinished — exactly the things a portfolio reviewer notices first:

1. **The entire bottom-half Timeline is mock UI.** Tracks are hardcoded, clips never populate from a real recording, and Play drives a fake CSS-only playhead that scrubs no media. It looks like a working editor and isn't — the worst kind of gap.
2. **Two dead transport buttons** (SkipBack / SkipForward) render with no `onClick` at all.
3. **The app ships Tauri's stock placeholder icon** (verified: the default yellow/blue mark) — reads as "never shipped."
4. **Post-save UX is a path-string toast** the user can't act on; the registered `tauri-plugin-opener` is never called from JS, so there's no "Reveal in Finder."

This plan does all four, ordered by value-per-effort, with ITEM 2 gated on the ITEM 1 decision.

## Current state (grounded)

### ITEM 1 — Timeline is mock UI (`frontend/src/components/asmr-recorder/timeline.tsx`)
- `Track` shape (21-32) carries `clips: Array<{id,name,start,duration}>`; the two initial tracks are hardcoded `"Audio Track"`/`"Screen Recording"` with **`clips: []`** (38-53). Nothing ever pushes a clip — `setTracks` is only called by `addTrack` (132-141), `deleteTrack` (143-145), `toggleTrackMute` (147-149). **No recording data ever enters this component.**
- Fake transport: `isPlaying` drives `setInterval(() => setPlayheadPosition(prev => prev + 1), 100)` (63-80); `playheadPosition` only feeds a CSS `left:` on the playhead div (336-341, via `getScaledPosition` 104). It **seeks no media** — there is no `<video>`, no media ref, nothing.
- `handlePlay` (116-124) flips local `isPlaying` and dispatches `timelinePlayback`. `handleTimelineClick` (126-130) sets `playheadPosition` from the click X — again CSS only.
- The only "real" thing the timeline renders is live audio-meter graphics from the recording context (`StereoMeter`/`Spectrum`, 242-247, 309-314) — those are live-input monitors, unrelated to any recorded clip.
- The `clips.map` render block (316-331) is reachable but always empty (no clip is ever added).

### Preview's timeline listener (`frontend/src/components/asmr-recorder/preview.tsx:185-203`)
- `handleTimelinePlayback` only does `setIsPlaying(event.detail.isPlaying)` — **plays nothing**. The Preview "play" just toggles a `<video>` mute/UI state; there is no recorded-clip playback anywhere.

### How a finished recording flows today (the real data the timeline lacks)
- On stop, the WebCodecs path finalizes the MP4, `await saveRecording(mp4Buffer)` then dispatches **`recordingReadyForEdit`** with the in-memory `Blob` (`recording-canvas.tsx:1432-1441`). `saveRecording` (569-586) `invoke("save_media_recording", data)` and on success dispatches **`recordingSaved` `{ path }`** (577-579).
- `recording-context.tsx:308-321` listens for `recordingSaved` and stores `status.outputPath`.
- `recording-context.tsx`'s `recordingReadyForEdit` consumer is the **TrimEditor** (`trim-editor.tsx:265-289`): it `URL.createObjectURL(blob)`, sets `videoUrl`, and **plays via a plain `<video ref={videoRef}>` element** (252, 499-500) — `video.currentTime`, `video.play()`, `video.pause()` (373-405). **Mediabunny is used only for thumbnails + packet-copy export** (`new mb.Input` / `EncodedPacketSink` / `CanvasSink`, 300-328) — NOT for playback. This is the key reuse insight: real clip playback here is just an HTML `<video>` + object URL.
- The **Open** + **Re-edit** toolbar feature (commit `4f5d792`) is in `toolbar.tsx`: `handleOpenFile`/`handleFileChange` (129-138) open a file picker and dispatch `recordingReadyForEdit` with the chosen file; `handleReEdit` (140-145) re-dispatches the cached `lastBlob` (captured 116-122). Both routes already feed the TrimEditor.

### ITEM 2 — Dead transport buttons (`frontend/src/components/asmr-recorder/toolbar.tsx`)
- SkipBack button (695-697): `<Button variant="outline" size="sm" className="bg-transparent"><SkipBack/></Button>` — **no `onClick`**.
- SkipForward button (719-721): same, **no `onClick`**.
- The Play (699-707) and Stop (709-717) between them are wired (`handlePlay` 213-220, `handleStop` 222-244) but only dispatch the same `timelinePlayback` event that plays nothing.

### ITEM 3 — Stock icon (`src-tauri/icons/`, `src-tauri/tauri.conf.json`)
- `tauri.conf.json:32-38` `bundle.icon` references `icons/32x32.png`, `128x128.png`, `128x128@2x.png`, `icon.icns`, `icon.ico`. All present (`ls src-tauri/icons/`): `icon.png` 512×512, full `Square*Logo.png` set, `.icns`, `.ico`. **Read of `128x128.png` confirms the stock Tauri default placeholder mark** (unbranded). `@tauri-apps/cli ^2` is in root `package.json:9`; root `package.json:5` has the `tauri` script (sets `DYLD_LIBRARY_PATH`), so `npm run tauri icon <src>` is available.

### ITEM 4 — Post-save UX is a dead-end toast
- `toolbar.tsx` save toast (105-113): on `status.outputPath` change, `toast({ title:"Recording saved", description:\`Saved to: ${path}\` })` — **string only, no action**.
- `toolbar.tsx` `handleExport` (246-258): if `status.outputPath`, toasts the path; else "No recording." **No file operation.**
- The toast system **supports an action element**: `use-toast.ts:13` `action?: ToastActionElement`; `toast.tsx:56-69` exports `ToastAction`. So a clickable toast button is already wired.
- `tauri-plugin-opener` is **registered** (`lib.rs:79`) and **granted** (`capabilities/default.json:8` `"opener:default"`), and the Rust crate is in `src-tauri/Cargo.toml:22`. **But the JS package `@tauri-apps/plugin-opener` is NOT in `frontend/package.json`** (only `@tauri-apps/api ^2.9.1`, `plugin-process`, `plugin-updater` at lines 24-26). So the frontend currently **cannot** call it — the dep must be added.

### App shell (`frontend/src/components/asmr-recorder.tsx`)
- Composition: `<Toolbar/>`, `<Preview/>`, then a fixed `h-64` `<Timeline/>` (24-27), with `<TrimEditor/>` overlaid (31). The timeline always occupies the bottom 16rem.

## Proposed design

### ITEM 4 (do first — cheapest, isolated) — Reveal in Finder / Open file

Add the JS opener dep and use `revealItemInDir(path)` (Finder reveal) where we have a path, `openPath(path)` to open in the default player.

#### 1. Add the dependency
`cd frontend && npm i @tauri-apps/plugin-opener` (matches the registered Rust plugin + existing `opener:default` grant — no Rust/capability change).

#### 2. Save toast gets a "Reveal in Finder" action
In `toolbar.tsx`, the `recordingSaved`-driven toast (105-113) gains an action:
```tsx
import { revealItemInDir } from "@tauri-apps/plugin-opener";
import { ToastAction } from "@/components/ui/toast";
// ...
toast({
  title: "Recording saved",
  description: status.outputPath,
  action: (
    <ToastAction altText="Reveal in Finder"
      onClick={() => revealItemInDir(status.outputPath!).catch(() => {})}>
      Reveal in Finder
    </ToastAction>
  ),
});
```

#### 3. Export button reveals instead of re-toasting the path
`handleExport` (246-258) becomes: if `status.outputPath`, `revealItemInDir(path)`; else keep the "No recording" toast. (Rename intent stays "Export" = "show me the file" — honest for an app that auto-saves on stop and has no separate export step outside the TrimEditor.)

### ITEM 1 — Timeline (the big decision)

Three options; **RECOMMEND C**, with A specced enough to choose deliberately.

#### Option A — WIRE it (full real editor): effort L, value high-but-redundant
Populate tracks/clips from the just-recorded/opened blob and implement real playback+seek. Mechanically feasible by reusing the TrimEditor's approach: a hidden `<video>` + `URL.createObjectURL(blob)`, listen for `recordingReadyForEdit`, set `clips` to one clip spanning `video.duration`, drive `playheadPosition` from `video.currentTime` via `requestAnimationFrame` (replace the fake `setInterval`), and make `handleTimelineClick` set `video.currentTime`. **Why we reject it as the default:** it rebuilds, in a second surface, exactly what the TrimEditor already does (playback, scrub, thumbnails) — duplicating playback state, object-URL lifecycle, and a transport. The mock multi-track UI (add/delete/mute arbitrary empty tracks) has no backing model (recordings are a single composited MP4, not separable tracks), so "wiring" it honestly still means deleting the multi-track fiction. High effort to reach a feature the app already has elsewhere — violates CLAUDE.md §2.

#### Option B — REMOVE the timeline entirely: effort S, value medium
Delete `<Timeline/>` + its `h-64` container from `asmr-recorder.tsx:24-27`, delete `timeline.tsx`. Honest and minimal, but **loses the live audio-meter/spectrum surface** (the one real thing the timeline shows during setup/recording) and leaves a visually empty bottom region unless the layout is reflowed. Throws away usable UI.

#### Option C — SCOPE-DOWN to an honest read-only state (RECOMMENDED): effort M, value high
Keep the timeline as a believable, honest surface; **remove the deception** (fake transport + mock multi-track editing) and show the **real recorded clip as a read-only strip** that routes to the editor for actual editing.

Concretely:

##### C.1 Drive a single real "clip" from `recordingReadyForEdit`
Add a listener (mirror `trim-editor.tsx:265-289`) that, on a finished/opened recording, reads its duration (cheap: a throwaway `<video>` with `loadedmetadata`, or reuse the blob the TrimEditor already has) and stores one read-only clip `{ name, durationSec }`. Replace the hardcoded `clips:[]` mock tracks with a single rendered strip representing that clip; show its duration. Until a recording exists, show an honest empty state ("Record or open a clip to see it here") instead of two fake tracks.

##### C.2 Remove the fake transport
Delete the `setInterval` playhead (63-80), `isPlaying`/Play button (177-179), `handlePlay` (116-124), and the `timelinePlayback` dispatch/listen wiring **on the timeline side**. The playhead div either goes away or becomes a static "0:00" marker. (The Preview's `timelinePlayback` listener at 185-203 plays nothing and can be left or removed in ITEM 2 cleanup.)

##### C.3 Add one real affordance: "Open in editor"
The clip strip (or a button on it) re-dispatches `recordingReadyForEdit` with the cached blob (exactly like `handleReEdit`, `toolbar.tsx:140-145`) so the TrimEditor — the surface that *does* real playback/scrub/trim — opens. This makes the timeline a true entry point, not a dead toy.

##### C.4 Keep the live meters
Retain `StereoMeter`/`Spectrum` (242-247, 309-314) — they're real and useful. The zoom controls (91-101) can stay (they harmlessly scale the strip) or be dropped; recommend keeping them minimal since they still scale the visual.

**Net for C:** timeline stops lying, shows the real clip + duration, and hands off editing to the existing editor — no duplicated playback engine, no mock multi-track model. This is the simplicity-first choice.

### ITEM 2 — Dead SkipBack / SkipForward (gated on ITEM 1)
- **If C or B:** there is no timeline transport to skip within → **remove both buttons** (`toolbar.tsx:695-697`, 719-721) and drop the now-unused `SkipBack`/`SkipForward` imports (27-28). Also remove the timeline-only Play/Stop transport in the toolbar if it no longer drives anything (Play 699-707 / Stop 709-717 dispatch `timelinePlayback`, which under C nothing consumes — confirm and remove the dead dispatch, keeping the recording Stop semantics intact via the Record button).
- **If A:** wire SkipBack → `video.currentTime -= step`, SkipForward → `+= step` (e.g. 5s), clamped to `[0, duration]`.

### ITEM 3 — Branded icon
1. Obtain a 1024×1024 (or larger) source PNG/SVG (see openQuestions — this plan does not invent art).
2. From repo root: `npm run tauri icon path/to/source.png`. This regenerates the full set in `src-tauri/icons/` (the exact files `tauri.conf.json:32-38` already references — no config edit needed).
3. Rebuild and confirm the new icon in the Dock and the built `.app`/`.dmg`.

## Implementation steps

1. **ITEM 4 dep + reveal.** `cd frontend && npm i @tauri-apps/plugin-opener`; wire `revealItemInDir` into the save toast (toolbar.tsx 105-113) and `handleExport` (246-258). **verify:** `npx tsc -b` clean (only pre-existing errors); `npm run tauri dev`, record a clip, click "Reveal in Finder" on the toast and the Export button — Finder opens with the file selected.
2. **DECISION GATE.** Confirm A/B/C (recommend C) before touching `timeline.tsx`. **verify:** decision recorded in this doc's openQuestions / a commit message.
3. **ITEM 1 (Option C) — honest clip strip.** In `timeline.tsx`: add the `recordingReadyForEdit` listener + duration read; replace mock `tracks`/`clips` (38-53) with the single read-only clip model + empty state; remove the fake-transport effect (63-80), Play (177-179), `handlePlay` (116-124); add the "Open in editor" re-dispatch; keep meters. **verify:** `npx tsc -b`; record a clip → strip shows real duration; click "Open in editor" → TrimEditor opens with that clip; no moving fake playhead.
4. **ITEM 2 — remove dead/now-dead transport.** Delete SkipBack/SkipForward buttons (toolbar.tsx 695-697, 719-721) and unused imports; remove any toolbar transport whose `timelinePlayback` dispatch no longer has a consumer (Play/Stop 699-717) — keep recording controls. **verify:** `npx tsc -b` (no unused-import errors); no toolbar button is a no-op; recording Record/Stop still works.
5. **ITEM 3 — icon.** Run `npm run tauri icon <source>` from root; do not edit `tauri.conf.json`. **verify:** `git status` shows the `src-tauri/icons/*` regenerated; `npm run tauri dev` shows the new Dock icon; `cargo build` (or a `tauri build`) succeeds with the new `.icns`.
6. **Full build gate.** **verify:** from `frontend/`: `npx tsc -b`, `npx vite build`, `npm run test` (48 tests still pass — none of these changes touch tested geometry/segment logic). From root: `npm run tauri dev` launches.

## Interface / API changes
- **New dep:** `@tauri-apps/plugin-opener` in `frontend/package.json` (pairs with the already-registered Rust plugin + existing `opener:default` grant — no Rust/capability change).
- **`timeline.tsx` (Option C):** mock `Track`/`clips` model replaced by a single read-only clip derived from `recordingReadyForEdit`; fake-transport state/handlers removed. No exported-component signature change (`<Timeline/>` still takes no props).
- **`toolbar.tsx`:** save toast + Export gain `revealItemInDir`/`openPath` calls; SkipBack/SkipForward (and possibly Play/Stop transport) removed. No prop changes.
- **No new custom events** — reuse `recordingReadyForEdit` (timeline→editor) and `recordingSaved`/`status.outputPath` (path for reveal).
- **No Rust changes.** Icons are regenerated assets, not code.

## Edge cases & risks
- **Reveal before any save:** `status.outputPath` is empty until `recordingSaved` fires — Export keeps the "No recording" branch; the save-toast action only renders when a path exists. Guard with `path && revealItemInDir(path)`.
- **Path that no longer exists** (user moved/deleted the file): `revealItemInDir` rejects → swallow with `.catch(() => {})` (or a soft toast); never crash.
- **Dev vs release save dir:** in dev the file lands in `../test-results` (`lib.rs:40-49`), in release in the user's Videos dir (52-57). `revealItemInDir(absolutePath)` works for both since the command returns an absolute path.
- **Opener dep/version drift:** pin to the `^2` line matching the Rust crate (`Cargo.toml:22`) to avoid a JS↔Rust protocol mismatch.
- **Timeline duration read (Option C):** if metadata can't load (corrupt/odd blob), fall back to showing the clip strip without a duration label rather than blocking — the TrimEditor already tolerates this (`trim-editor.tsx:312-315` null-coalesces duration).
- **WebM fallback path:** only the WebCodecs/MP4 path dispatches `recordingReadyForEdit` (recording-canvas.tsx 1429-1441 comment). A MediaRecorder/WebM fallback recording won't populate the timeline clip — acceptable and consistent with today's TrimEditor behavior; note it, don't special-case it.
- **Icon regen overwrites the whole set:** `tauri icon` rewrites every file in `icons/` — review the diff so no hand-tuned asset is silently lost (none exist today; all are stock).
- **Removing toolbar Play/Stop:** ensure the *recording* stop path (`handleRecord`/`stopExternalRecording`) is untouched — only the timeline-playback transport is dead. Verify by recording + stopping after the change.

## Testing & verification

### Build gates (auto)
- `frontend/`: `npx tsc -b` (expect only pre-existing errors), `npx vite build`, `npm run test` (48 vitest tests; unaffected — no geometry/segment logic changes).
- root: `cargo build` (icons don't affect compile, but the `.icns` must be valid); `npm run tauri dev` launches.

### Manual (dev — can't auto-verify)
- **ITEM 4:** record → save toast shows "Reveal in Finder" → click opens Finder with file selected; Export button reveals the same; with no recording, Export shows "No recording."
- **ITEM 1 (C):** record/open a clip → timeline shows one read-only strip with the real duration and an honest empty state beforehand; "Open in editor" opens the TrimEditor on that clip; confirm no fake playhead motion.
- **ITEM 2:** visually confirm no no-op buttons remain in the toolbar; Record/Stop unaffected.
- **ITEM 3:** new icon appears in Dock (dev) and in a `tauri build` `.app`/`.dmg`.

## Out of scope / future
- **Multi-clip / multi-track non-linear editing in the timeline** — there is no multi-track recording model (recordings are one composited MP4); building a real NLE is a separate large feature, not "completeness." Explicitly rejected per CLAUDE.md §2.
- **Option A's duplicated playback engine** — rejected above; real editing already lives in the TrimEditor.
- **Frame-accurate timeline scrubbing / waveform rendering of the recorded clip** — the TrimEditor already provides thumbnails + frame-accurate trim; don't duplicate.
- **"Open in default player" vs "Reveal in Finder" toggle / preferences** — ship the single reveal action; add `openPath` only if asked.
- **Icon as code-generated SVG pipeline** — out of scope; a static source artifact + `tauri icon` is sufficient.

## References
- `frontend/src/components/asmr-recorder/timeline.tsx` — `Track`/`clips` shape (21-32), hardcoded mock tracks (38-53), fake-transport effect (63-80), `timelinePlayback` listen (82-89), `handlePlay` (116-124), `handleTimelineClick` (126-130), `addTrack`/`deleteTrack`/`toggleTrackMute` (132-149), meters (242-247, 309-314), empty `clips.map` (316-331), playhead CSS (336-341).
- `frontend/src/components/asmr-recorder/preview.tsx` — `timelinePlayback` listener that plays nothing (185-203).
- `frontend/src/components/asmr-recorder/toolbar.tsx` — save toast (105-113), `handleReEdit` (140-145), `handlePlay` (213-220), `handleStop` (222-244), `handleExport` (246-258), SkipBack (695-697), Play/Stop (699-717), SkipForward (719-721), Open/Re-edit (726-749), `SkipBack`/`SkipForward` imports (27-28).
- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — `saveRecording` + `recordingSaved` dispatch (569-586), MP4 finalize + `recordingReadyForEdit` dispatch (1426-1441).
- `frontend/src/components/asmr-recorder/trim-editor.tsx` — `recordingReadyForEdit` listener + object URL (265-289), `<video ref>` playback/scrub (252, 359-405, 499-500), mediabunny used only for thumbnails/export (300-328).
- `frontend/src/contexts/recording-context.tsx` — `recordingSaved` → `status.outputPath` (308-321).
- `frontend/src/hooks/use-toast.ts` — `action?: ToastActionElement` (13). `frontend/src/components/ui/toast.tsx` — `ToastAction` (56-69).
- `frontend/src/components/asmr-recorder.tsx` — shell composition, `h-64` Timeline container (24-27), TrimEditor overlay (31).
- `frontend/package.json` — deps: `@tauri-apps/api ^2.9.1`, `plugin-process`, `plugin-updater` (24-26); **no `plugin-opener`**.
- `src-tauri/src/lib.rs` — `save_media_recording` returns path (14-71), `tauri_plugin_opener::init()` registered (79).
- `src-tauri/capabilities/default.json` — `"opener:default"` granted (8).
- `src-tauri/Cargo.toml` — `tauri-plugin-opener = "2"` (22), `dirs = "5.0"` (43).
- `src-tauri/tauri.conf.json` — `bundle.icon` list (32-38). `src-tauri/icons/` — stock default mark (verified read of `128x128.png`), full set present.
- root `package.json` — `@tauri-apps/cli ^2` (9), `tauri` script with `DYLD_LIBRARY_PATH` (5).
- Commit `4f5d792` — "open + re-edit recordings from the toolbar" (the existing editor entry points reused by ITEM 1-C).
- Related plans: [[per-section-transport-unregister]], [[next-implementations]], [[trim-editor-lossless-packet-copy]].