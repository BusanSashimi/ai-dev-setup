# Chore: Test harness for pure logic + CI build gates

**Priority:** P3  ·  **Effort:** M  ·  **Risk:** low  ·  **Surface:** tooling + small refactors

## Problem / motivation

There is **no automated safety net**:
- **Frontend:** zero tests. `frontend/package.json` scripts are `dev`/`build`/`lint`/`preview`
  only — no test runner (no `vitest`/`jest`, no `@testing-library`).
- **Backend:** a handful of `#[test]`s exist (`screen.rs` ×2, `webcam.rs` ×1, `compositor.rs`
  ×2, `audio_mixer.rs` ×2) but **three of those four files are dead** and slated for deletion
  (`[[legacy-native-pipeline-removal]]`), so the real surviving coverage is `screen.rs`'s
  frame-conversion tests.
- **CI:** none (`.github/workflows/` does not exist). Every regression is caught only by the
  developer remembering to run `cargo build` / `npx tsc -b` / `npx vite build` by hand.

Most of this app is I/O-bound glue (WebCodecs, ScreenCaptureKit, Tauri IPC, WKWebView media)
that **cannot** be meaningfully unit-tested without the live runtime — and mocking WebCodecs
would be high-effort, low-value, and brittle. So the goal here is deliberately modest and
high-leverage:

1. **Build gates in CI** — `cargo build` + `cargo test` + `tsc -b` + `vite build` on every
   push. These alone catch the large majority of regressions (type errors, broken Rust,
   bundle failures) that currently rely on manual discipline.
2. **Unit tests for the few genuinely pure, correctness-critical, runtime-free functions** —
   the wire-format codecs and the trim segment math. These are exactly the spots where a
   silent off-by-one corrupts output, and they need no DOM/WebCodecs/SCK.

## Current state (grounded)

- `frontend/package.json` — scripts: `"dev"`, `"build": "tsc -b && vite build"`, `"lint"`,
  `"preview"`. No test script, no test deps.
- `frontend/vite.config.ts` — react plugin, `server.port 5178`, `@`→`./src` alias. No `test`
  block (vitest config).
- `src-tauri/src/*.rs` — `#[cfg(test)]` modules in `screen.rs`, `webcam.rs`, `compositor.rs`,
  `audio_mixer.rs`. Run via `cargo test` (default features — `--features ffmpeg` doesn't link
  on this arm64 machine; `docs/logs/2026-06-13.md`).
- **Pure, testable, runtime-free logic worth covering (live code):**
  - **Native wire-format header parse** — `frontend/src/lib/native-screen.ts`
    `channel.onmessage` decodes a 16-byte LE header (`DataView.getUint32`/`getBigUint64`,
    line 61-72). This is the contract with the Rust `Raw` sender; an off-by-one silently
    corrupts every frame. Currently inline in the closure — **extract a pure
    `parseFrameHeader(buf): { width, height, timestampMs, jpegOffset }`** to test. (The
    forthcoming `[[native-system-audio-capture]]` adds a second such header — same pattern.)
  - **Trim segment math** — `frontend/src/components/asmr-recorder/trim-editor.tsx`
    `handleCutSelection` splits the active keep-segment `[in,out]` into up to two segments
    with a `MIN_SEG` guard; `totalKeptDuration` sums them; `formatTime` (line 19-25) formats
    `mm:ss.t`. The split/sum logic is pure but currently embedded in a `useCallback` —
    **extract `splitSegment(seg, in, out, minSeg)` and `keptDuration(segments)`** to test
    the boundary cases (cut at head, at tail, spanning whole, sub-`MIN_SEG` slivers).
  - **Rust frame conversion** — `screen.rs` `ScreenFrame::to_rgba` stride handling (already
    tested; keep and don't regress).

## Proposed design

**(1) Vitest for the frontend.** Add `vitest` + (optionally) `@testing-library/react` as
devDeps, a `test` script, and a `test` block in `vite.config.ts` (`environment: "node"` —
the target functions are pure, no DOM needed; `jsdom` only if a component test is later
added). Co-locate tests as `*.test.ts` next to the source.

**(2) Small pure-function extractions** (the only source changes). Pull the header parse and
the segment math out of their closures into exported pure helpers, then unit-test them. This
is a refactor that *also* improves the code (the wire-format contract becomes a named,
documented function reused by the screen and system-audio bridges).

**(3) Keep `cargo test`.** No new Rust tests required beyond what survives removal; ensure
`screen.rs` conversion tests run in CI. (Optional: add a round-trip test for the Rust side of
the header — encode in Rust, assert the byte layout the TS `parseFrameHeader` expects.)

**(4) GitHub Actions CI** (`.github/workflows/ci.yml`) on `macos-latest` (SCK + the Swift
rpath are macOS-only): cache cargo + npm, then run the build gates and tests. **Build only —
no signing/notarization** (that's `[[packaging-and-distribution]]`, needs secrets + an Apple
account).

## Implementation steps

1. **Add vitest.** `frontend`: add `vitest` (+ `@testing-library/react` + `jsdom` only if a
   component test lands) to devDeps; add `"test": "vitest run"` and `"test:watch": "vitest"`;
   add a `test` block to `vite.config.ts`. **verify:** `npm run test` runs (0 tests) and exits 0.
2. **Extract + test the wire-format header.** Move the `native-screen.ts` header decode into
   an exported `parseFrameHeader(buf: ArrayBuffer)`; rewrite `onmessage` to use it. Add
   `native-screen.test.ts`: build a known 16-byte header + payload, assert width/height/
   timestamp/offset; include an endianness and a truncated-buffer case. **verify:**
   `npm run test` green; `npx tsc -b` clean; native screen still draws in dev.
3. **Extract + test the trim segment math.** Pull `splitSegment` / `keptDuration` out of
   `trim-editor.tsx` into a sibling pure module (or top-of-file exports); rewrite
   `handleCutSelection` / `totalKeptDuration` to call them. Add `trim-editor.segments.test.ts`
   covering: cut at head, at tail, mid (→2 segments), whole-segment cut (→0), sub-`MIN_SEG`
   slivers dropped, `keptDuration` sum, `formatTime` formatting. **verify:** `npm run test`
   green; multi-segment trim still works in dev (regression).
4. **CI workflow.** Add `.github/workflows/ci.yml` (`macos-latest`): checkout; setup-node +
   cache; `npm ci` (root + frontend); setup Rust + cache; `cd src-tauri && cargo build` +
   `cargo test` (default features); `cd frontend && npx tsc -b && npx vite build && npx vitest
   run`. **verify:** the workflow passes on a test branch; a deliberately-broken type fails it.
5. **Document.** Note in `ai-dev/README.md`: `npm run test` (frontend), `cargo test`
   (backend), and that CI runs build gates only (release builds are separate, see
   `[[packaging-and-distribution]]`). **verify:** instructions run cold.

## Interface / API changes

- **New devDeps:** `vitest` (+ optional `@testing-library/react`, `jsdom`). New `test`
  scripts; `vite.config.ts` gains a `test` block.
- **Refactors (behavior-preserving):** `parseFrameHeader` exported from `native-screen.ts`;
  `splitSegment`/`keptDuration` extracted from `trim-editor.tsx`. Callers updated in place.
- **New files:** `.github/workflows/ci.yml`, `*.test.ts` alongside sources.
- **No Rust API change** (keep existing `#[test]`s; optional header round-trip test).

## Edge cases & risks

- **Don't over-mock.** WebCodecs/SCK/Tauri-IPC/WKWebView paths are out of scope for unit
  tests — faking them is brittle and low-value. The build gates + a few pure tests are the
  right ROI; resist growing a mock framework.
- **Extraction must be behavior-preserving.** The refactors are the only source risk: verify
  the live native-screen draw and multi-segment trim still work in dev after extracting the
  helpers (steps 2-3 each have a dev regression check).
- **CI runner cost/availability.** `macos-latest` minutes are limited on free tiers; keep the
  workflow lean (build + test, cache aggressively, no matrix). A signed release build is a
  separate, secrets-bearing workflow (`[[packaging-and-distribution]]`).
- **Ordering vs. removal.** If `[[legacy-native-pipeline-removal]]` runs first, the
  `compositor.rs`/`webcam.rs`/`audio_mixer.rs` tests vanish with their modules — expected;
  `cargo test` then runs the surviving `screen.rs` tests. Don't port dead-module tests.
- **`tsc -b` already runs in `build`** — CI runs it explicitly too so a type error fails fast
  before the (slower) vite build.

## Testing & verification

- **Self-verifying by nature:** `npm run test` and `cargo test` are the deliverable. CI green
  on a clean branch and **red** on an injected type error / failing assertion is the
  acceptance check.
- **Build gates (auto):** `cargo build` + `cargo test` (default features), `npx tsc -b`,
  `npx vite build`, `npx vitest run`.
- **Cannot auto-verify:** that CI catches *every* class of regression — it catches
  compile/type/bundle/pure-logic breaks, not runtime WebCodecs/SCK behavior (those remain the
  hybrid manual passes documented in the other plans).

## Out of scope / future

- Component/integration tests that need a DOM or fake media streams.
- E2E driving of the WKWebView (not programmatically drivable on macOS, per the house notes).
- A signed+notarized **release** CI pipeline — `[[packaging-and-distribution]]`.
- Coverage thresholds / reporting (premature for this surface).

## References

- `frontend/package.json` — scripts (`dev`/`build`/`lint`/`preview`; no `test`).
- `frontend/vite.config.ts` — react plugin, port 5178, `@` alias (no `test` block).
- `frontend/src/lib/native-screen.ts` — header decode in `onmessage` (61-72) → extract
  `parseFrameHeader`.
- `frontend/src/components/asmr-recorder/trim-editor.tsx` — `formatTime` (19-25),
  `handleCutSelection` split + `MIN_SEG`, `totalKeptDuration` → extract `splitSegment`/`keptDuration`.
- `src-tauri/src/screen.rs` — surviving `#[cfg(test)]` frame-conversion tests.
- Related: `[[legacy-native-pipeline-removal]]` (removes dead-module tests),
  `[[packaging-and-distribution]]` (separate release CI), `[[native-system-audio-capture]]`
  (second wire-format header to cover).
- Dev log: `docs/logs/2026-06-13.md` — `cargo test`/default-features build facts.
