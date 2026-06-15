# Feature: Multiple composition layouts (Solo · PiP · Side-by-Side) alongside the 2×2 grid

**Priority:** P2  ·  **Effort:** M  ·  **Risk:** med  ·  **Surface:** frontend (composite + preview + settings)

## Problem / motivation

The recorder composites sources into exactly **one** arrangement: a hardcoded 2×2 grid. The
geometry is duplicated in two unsynced places — the CSS grid in `preview.tsx` (the on-screen
preview) and the Canvas-2D draws in `recording-canvas.tsx` (the actual H.264 encode). For an
ASMR/streaming recorder, the 2×2 grid is the *least* common layout in the wild; the popular
ones (per OBS / vMix / mimoLive / broadcast guides) are:

| Layout | Sources | Overlap? | Why it matters here |
|--------|---------|----------|---------------------|
| **Solo / Fullscreen** | 1 | no | The dominant ASMR shot — one closeup filling the frame. mimoLive ships explicit "Solo A/B/C" toggles for this. |
| **Picture-in-Picture** | 2 | **yes** | The single most popular streaming layout (full-frame source + small corner overlay; gamers favor top-right/bottom-left). For ASMR: overhead/macro full-frame + face in the corner. **Requires z-ordering** the grid never does. |
| **Side-by-Side / 2-up** | 2 | no | Two-angle ASMR / interview framing. **Requires aspect-fit** the grid never needs. |
| 2×2 grid (current) | 4 | no | Multi-angle / multiview. Stays the default. |

Goal: add **Solo, PiP, and Side-by-Side** as selectable layouts, without regressing the 2×2
default, by replacing the two hardcoded geometries with **one data-driven layout registry**
that drives both render paths.

## Current state (grounded)

- **Composite (recording).** `frontend/src/components/asmr-recorder/recording-canvas.tsx`
  - `sectionWidth = outputWidth / 2`, `sectionHeight = outputHeight / 2` (lines 174-175).
  - `compositeFrame()` (328-414) hardcodes four `drawSection()` calls at the four quadrants
    (377-401), then draws the center cross grid lines (403-413).
  - `drawSection(ctx, source, destX, destY, destW, destH)` (245-323) **stretches** the source
    to the dest rect via the dest-rect `drawImage` (283, 300). It already supports a **source
    crop** via `source.region` using the 9-arg `drawImage` form (271-281) — this is the
    mechanism a `"cover"` fit will reuse.
  - Props (`RecordingCanvasProps`, 23-47): `outputWidth/Height`, `frameRate`, `isRecording`,
    `sectionSources` + `getSectionSources()` (a fixed **4-tuple** of `SectionSource`),
    audio flags. **No layout prop today.**
- **Preview (on-screen).** `frontend/src/components/asmr-recorder/preview.tsx`
  - 16:9 `Card` container (725-735), inside it a CSS grid `grid grid-cols-2 grid-rows-2 gap-1
    p-1` (738) mapping `[0,1,2,3] → renderSectionContent(index)` (739-743).
  - `renderSectionContent(index)` (594-720) returns the per-section content: live
    video/canvas element, or the "add source" / empty affordance, plus click-to-configure.
  - `getSectionSources()` (111-157) builds the recording 4-tuple from `sectionState`.
  - `<RecordingCanvas …>` instantiated at 863-873 with `outputWidth={externalConfig.outputWidth}`
    etc. — **the place to thread a `layout` prop.**
- **State / config.** `frontend/src/types/recording.ts`
  - `SectionState.sections` is a fixed `[SectionConfig, …×4]` tuple (124-127). **Keep as-is.**
  - `ExternalRecordingConfig` (66-83) + `defaultExternalRecordingConfig` (85-93) — the config
    object held in `recording-context.tsx` (`useState`, line 59) and mutated via
    `updateExternalConfig()` (142-153). **Not persisted to localStorage** (session-scoped,
    same as `videoQuality`/`outputResolution`). The existing unused `PipPosition` type (line 3)
    and `webcamPosition`/`webcamSize` fields (RecordingConfig, 20-21, defaults "top-right"/25)
    are leftovers from the dead native pipeline — **reuse `PipPosition` for the PiP corner.**
- **Settings UI.** `frontend/src/components/asmr-recorder/toolbar.tsx` — the Recording Settings
  `Dialog` (149-…) already has shadcn `Select`s for quality (242-256) and resolution (262+),
  each calling `updateExternalConfig({…})`. **The layout picker goes here.**
- **Tests.** Vitest now exists (`[[test-harness-and-ci]]`) and the house pattern is to unit-test
  *pure geometry/segment math*. Slot-rect computation is exactly that shape.

## Proposed design

### 1. One layout registry (single source of truth) — `frontend/src/lib/layouts.ts` (new)

Layouts are **data**: an ordered list of slots in **normalized 0..1 coordinates**, so the same
numbers serve canvas pixels (`×outputWidth/Height`) and preview CSS (`%`). Slot **order = draw
order = z-order** (the background slot first, the PiP overlay last).

```ts
import type { PipPosition } from "@/types/recording";

export type LayoutType = "grid-2x2" | "solo" | "side-by-side" | "pip";

export interface LayoutSlot {
  section: number;        // index into the 4 section sources
  x: number; y: number;   // normalized top-left (0..1)
  w: number; h: number;   // normalized size (0..1)
  fit: "fill" | "cover";  // "fill" = stretch (today); "cover" = crop-to-fill
}

export interface LayoutOpts { pipPosition?: PipPosition; pipSize?: number; } // PiP only

export const LAYOUT_LABELS: Record<LayoutType, string> = {
  "grid-2x2": "2×2 Grid", solo: "Solo / Fullscreen",
  "side-by-side": "Side-by-Side", pip: "Picture-in-Picture",
};

// Pure. Output is always 16:9, so a normalized SQUARE (w===h) stays 16:9 in pixels —
// that keeps the PiP corner at the output aspect for free.
export function computeSlots(layout: LayoutType, opts: LayoutOpts = {}): LayoutSlot[] {
  switch (layout) {
    case "grid-2x2": return [
      { section: 0, x: 0,   y: 0,   w: 0.5, h: 0.5, fit: "fill" },
      { section: 1, x: 0.5, y: 0,   w: 0.5, h: 0.5, fit: "fill" },
      { section: 2, x: 0,   y: 0.5, w: 0.5, h: 0.5, fit: "fill" },
      { section: 3, x: 0.5, y: 0.5, w: 0.5, h: 0.5, fit: "fill" },
    ];
    case "solo": return [{ section: 0, x: 0, y: 0, w: 1, h: 1, fit: "cover" }];
    case "side-by-side": return [
      { section: 0, x: 0,   y: 0, w: 0.5, h: 1, fit: "cover" }, // half-width => MUST cover
      { section: 1, x: 0.5, y: 0, w: 0.5, h: 1, fit: "cover" },
    ];
    case "pip": {
      const s = opts.pipSize ?? 0.25, m = 0.025; // size & margin as width-fractions
      const pos = opts.pipPosition ?? "top-right";
      const x = pos.endsWith("left")  ? m : 1 - s - m;
      const y = pos.startsWith("top") ? m : 1 - s - m;
      return [
        { section: 0, x: 0, y: 0, w: 1, h: 1, fit: "cover" }, // background (drawn first)
        { section: 1, x, y, w: s, h: s, fit: "cover" },        // overlay (drawn last = on top)
      ];
    }
  }
}
```

Why `"cover"` matters: every 2×2 cell is `(W/2)×(H/2)` = 16:9, so today's stretch is
aspect-correct for a 16:9 source — **`grid-2x2` keeps `fit:"fill"`, zero regression.** But a
side-by-side half is `(W/2)×H` ≈ 8:9 (portrait); a 16:9 source stretched into it is severely
squished, so side-by-side **requires** `"cover"`. Solo/PiP use `"cover"` so non-16:9 cameras
(e.g. 4:3 webcams) crop instead of distort.

### 2. `drawSection` gains a `fit` arg — the one new drawing capability

Add `fit: "fill" | "cover"` (default `"fill"`). `"cover"` computes a centered source crop rect
so the source fills the dest without distortion, composing with the existing `region` crop
(when both apply, cover crops *within* the region):

```
sourceRect = region ?? full source dimensions
if fit === "cover": shrink sourceRect to the dest aspect (centered), then drawImage(9-arg)
else: drawImage(dest-rect form)  // unchanged
```

`drawSection` already has the 9-arg path (271-281); this generalizes it from "region only" to
"region and/or cover".

### 3. `compositeFrame` iterates slots

Replace the hardcoded block (377-401) + grid lines (403-413) with:

```ts
const slots = computeSlots(layout, { pipPosition, pipSize });
for (const s of slots) {
  drawSection(ctx, currentSources[s.section],
    s.x * outputWidth, s.y * outputHeight, s.w * outputWidth, s.h * outputHeight, s.fit);
}
// grid lines only for grid-2x2 (or a thin 1px border per slot, layout-agnostic)
```

`layout`, `pipPosition`, `pipSize` arrive as new props on `RecordingCanvasProps`.

### 4. Preview unification — same rects drive the DOM

Replace the CSS `grid-cols-2 grid-rows-2` (preview.tsx 738-744) with **absolutely-positioned
divs** computed from `computeSlots(layout)`: for each slot, a `<div style={{position:absolute,
left:`${x*100}%`, top:`${y*100}%`, width:`${w*100}%`, height:`${h*100}%`, zIndex:i}}>` wrapping
`renderSectionContent(slot.section)`. This is the only mechanism that previews **overlap**
(PiP) and arbitrary rects faithfully, and it keeps a single geometry source for preview +
recording. The inner `renderSectionContent` is unchanged (still the per-section video/canvas +
configure affordance) — it just scales to its container. Use CSS `object-fit: cover` on the
preview elements to mirror the canvas `"cover"` fit visually.

### 5. Source-assignment model (UX) — no new assignment UI

A layout maps slots → section indices; **only the slots the active layout uses are shown**.
Solo shows section 0 full-frame; PiP shows section 0 (full) + section 1 (corner); Side-by-Side
shows 0 + 1; 2×2 shows all four. The existing click-a-cell-to-configure flow works unchanged on
the visible slots. Switching layouts never destroys section config — hidden sections (e.g. 2/3
in Solo) keep their sources and reappear on switching back to 2×2. (Their streams keep running;
pausing unused streams is a possible future optimization, not needed for correctness.)

### 6. Config + picker

- `recording.ts`: add `layout: LayoutType` (+ optional `pipPosition`, `pipSize`) to
  `ExternalRecordingConfig`; default `layout: "grid-2x2"`, `pipPosition: "top-right"`,
  `pipSize: 0.25` in `defaultExternalRecordingConfig` (preserves current behavior).
- `toolbar.tsx`: add a **Layout** `Select` in the settings dialog (reuse `LAYOUT_LABELS`),
  calling `updateExternalConfig({ layout })`. When `layout === "pip"`, reveal a small
  position `Select` (the four `PipPosition` values). (A thumbnail radio is a nice-to-have;
  a labeled `Select` matches the existing quality/resolution controls and is the minimal cut.)
- `preview.tsx`: pass `layout={externalConfig.layout}` (+ pip opts) to `<RecordingCanvas>`
  (863-873) and to the preview slot mapping.

## Implementation steps

1. **Registry + pure tests.** Add `frontend/src/lib/layouts.ts` (`LayoutType`, `LayoutSlot`,
   `computeSlots`, `LAYOUT_LABELS`). Add `layouts.test.ts`: assert slot counts (solo→1,
   pip/side-by-side→2, grid→4), that grid slots tile to the full frame with no overlap, that
   PiP corner stays inside [0,1] for all four positions and is drawn last, and that pixel
   mapping (`×W/H`) round-trips. **verify:** `npm run test` green; `npx tsc -b` clean.
2. **`drawSection` fit arg.** Add `fit` param + the centered-cover crop math (composing with
   `region`); default `"fill"`. **verify:** unit-test the crop-rect helper if extracted; 2×2
   recording is byte-for-byte unchanged in a dev recording (all slots `"fill"`).
3. **`compositeFrame` + props.** Add `layout`/`pipPosition`/`pipSize` to `RecordingCanvasProps`;
   replace the hardcoded quadrants/grid-lines with the `computeSlots` loop; gate grid lines to
   `grid-2x2`. **verify:** dev-record each layout; output MP4 shows the right arrangement, PiP
   overlay on top, side-by-side un-squished.
4. **Config + context.** Extend `ExternalRecordingConfig` + `defaultExternalRecordingConfig`;
   no change needed to `updateExternalConfig` (generic spread). **verify:** `tsc -b` clean;
   default config still records a 2×2.
5. **Picker UI.** Add the Layout `Select` (+ conditional PiP-position `Select`) to
   `toolbar.tsx`. **verify:** switching the picker updates the live preview immediately.
6. **Preview unification.** Replace the CSS grid with the absolute-positioned slot mapping from
   `computeSlots(layout)`; add `object-fit:cover` to preview media. **verify:** preview matches
   the recorded output for all four layouts (esp. PiP overlap + clickable corner); switching
   layouts preserves section config; recording overlay/controls (746-800) still render above.
7. **Docs / index.** Add a backlog→planned entry in `[[next-implementations]]`; note the new
   `layouts.ts` in `ai-dev/README.md` if it lists key modules. **verify:** links resolve.

## Interface / API changes

- **New:** `frontend/src/lib/layouts.ts` (`LayoutType`, `LayoutSlot`, `LayoutOpts`,
  `computeSlots`, `LAYOUT_LABELS`) + `layouts.test.ts`.
- **`recording.ts`:** `ExternalRecordingConfig` gains `layout: LayoutType` (+ optional
  `pipPosition: PipPosition`, `pipSize: number`); defaults added. (`PipPosition` reused.)
- **`recording-canvas.tsx`:** `RecordingCanvasProps` gains `layout` (+ pip opts); `drawSection`
  gains `fit`; `compositeFrame` rewritten to iterate slots. `sectionWidth/Height` (174-175) may
  be removed (now derived per-slot).
- **`preview.tsx`:** CSS grid → absolute slot mapping; `<RecordingCanvas>` gets new props.
- **`toolbar.tsx`:** new Layout (+ PiP position) `Select`s.
- **No Rust/back-end change** — composition is entirely frontend; the encoder receives the same
  finished canvas frames regardless of layout.

## Edge cases & risks

- **Two render paths drifting** is the core risk this design retires — both must derive from
  `computeSlots`. Resist re-introducing layout-specific CSS branches in the preview.
- **PiP z-order & hit-testing:** overlay slot is last (on top). In the preview the corner div
  needs a higher `z-index` *and* must stay click-through to configure section 1 while the
  background remains clickable elsewhere. Verify clicks hit the intended section.
- **`"cover"` × `region` interaction:** for a cropped screen source in side-by-side/PiP, cover
  must crop *within* the already-selected region, not the full frame. Cover this in step 2.
- **Non-16:9 sources:** `"cover"` crops (loses edges) by design; that's preferred over
  distortion, but document it (a future `"contain"`/letterbox option is possible).
- **Empty slots:** Solo/PiP/Side-by-Side leave sections unused; ensure the preview hides them
  and the composite simply doesn't reference them (no black-quadrant artifacts). Section 0 must
  have a sensible empty-state in Solo (full-frame "add source").
- **Switching layouts mid-config:** must be non-destructive (hidden sections retained). Do **not**
  tear down streams on switch.
- **Grid lines:** only `grid-2x2` draws the center cross; other layouts get none (or an optional
  per-slot hairline) — don't leave a stray cross on Solo.

## Testing & verification

- **Pure (vitest, auto):** `layouts.test.ts` for `computeSlots` (counts, tiling, bounds,
  z-order, pixel round-trip) and the cover-crop helper. This is the high-ROI safety net and
  mirrors `[[test-harness-and-ci]]`'s philosophy (test pure geometry, not the WebCodecs glue).
- **Build gates (auto):** `npx tsc -b`, `npx vite build`, `npm run test`.
- **Manual (dev, can't auto-verify):** record one short clip per layout; confirm the **output
  MP4** (not just preview) shows the correct arrangement, PiP overlay composited on top,
  side-by-side un-squished; confirm preview ≡ output; confirm layout switches preserve sources
  and the trim editor still opens the result.

## Out of scope / future

- **Custom layout editor** (drag/resize slots, save/reuse named templates, vMix/mimoLive-style).
  This plan ships a **fixed preset registry** — the popular layouts for a fraction of the code.
- **3×3 / N-up multiview**, dynamic source counts (>4 sections), and `"contain"`/letterbox fit.
- **Per-PiP styling** (border/shadow/rounded corner), animated transitions between layouts.
- **Pausing unused sections' streams** on layout switch (perf optimization).
- **Persisting `layout`** across app restarts (would require persisting `externalConfig`, which
  nothing does today — a separate, app-wide decision).

## References

- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — `drawSection` (245-323, stretch
  at 283/300, region crop 271-281), `compositeFrame` 2×2 hardcode (377-401) + grid lines
  (403-413), `sectionWidth/Height` (174-175), `RecordingCanvasProps` (23-47).
- `frontend/src/components/asmr-recorder/preview.tsx` — CSS grid (738-744), `renderSectionContent`
  (594-720), `getSectionSources` (111-157), `<RecordingCanvas>` (863-873).
- `frontend/src/types/recording.ts` — `ExternalRecordingConfig`/default (66-93), `SectionState`
  (124-141), `PipPosition` (3), `webcamPosition`/`webcamSize` (20-21, 53-54).
- `frontend/src/contexts/recording-context.tsx` — `externalConfig` state (59),
  `updateExternalConfig` (142-153).
- `frontend/src/components/asmr-recorder/toolbar.tsx` — settings `Dialog` + quality/resolution
  `Select`s (149-256+).
- Related: `[[test-harness-and-ci]]` (pure-test pattern + CI gates), `[[next-implementations]]`
  (roadmap index).
- Layout research (popular streaming layouts): OBS PiP (streamshark.io/obs-guide/picture-in-picture),
  facecam placement (resources.overlays.uno/post/webcam-overlay-in-obs), vMix Multiview templates
  (streamgeeks.us), mimoLive split-screen/Solo (mimolive.com), broadcast PiP/split-screen (Fiveable).
