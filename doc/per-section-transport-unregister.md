# Feature: Per-section transport teardown — unregister the screen/audio Channel callback on stop

**Priority:** P3  ·  **Effort:** S  ·  **Risk:** low  ·  **Surface:** both (frontend transport + Rust worker lifecycle)

## Problem / motivation

This is the deferred "Channel callback unregister on stop" item from
`[[native-screen-transport-raw-bytes]]` ("Out of scope / future", line 335). The
native-screen path (and the analogous native-system-audio path) creates one Tauri
`Channel<ArrayBuffer>` per section in `startNativeScreenStream`, sets
`channel.onmessage`, and hands it to the backend. When the stream is stopped we tell
the **backend** worker to exit, but we never tear down the **frontend** callback.

Why that leaks: the `@tauri-apps/api@2.9.1` `Channel` registers its message handler
via `transformCallback`, which stows the closure in `window.__TAURI_INTERNALS__`
keyed by `channel.id`. The *only* code that removes it is `cleanupCallback()` →
`window.__TAURI_INTERNALS__.unregisterCallback(this.id)` (core.js line 117-118), and
that fires **exclusively** when an `end` message arrives over the channel (core.js
line 84-91, 106-108). Our Rust workers only ever `Channel::send(Raw(...))` frame
payloads — they never close the channel — so `end` is never delivered and
`cleanupCallback` never runs. The closure (which captures `onFrame`, the section's
canvas/audio nodes, and `ack`) stays registered for the life of the webview.

Over repeated start/stop/reconfigure cycles on a section (clear → re-add, change
display, switch to camera, re-pick a region — each calls
`stopNativeStreamForSection` then `startNativeScreenStream` again, minting a fresh
`Channel`), this accumulates one orphaned closure per cycle. Worse than a pure memory
leak: an **in-flight frame already dispatched** to the old channel's `id` (or a stale
`createImageBitmap` resolving after teardown) can still run the old `onmessage`,
firing `createImageBitmap` work and an `ack` against a section whose canvas ref the
component has already nulled — wasted work, a misattributed credit bump, and console
noise. Small, low-risk hygiene fix; scope it tightly.

## Current state (grounded)

**Frontend screen — `frontend/src/lib/native-screen.ts`:**
- `startNativeScreenStream` (line 51-84) does `const channel = new Channel<ArrayBuffer>()`
  (56), sets `channel.onmessage` (64-72) — the handler parses the header, builds a
  `Blob`, `createImageBitmap(...).then(onFrame; ack)` `.catch(ack)` — then
  `invoke("start_screen_stream", { …, onFrame: channel })` (74-81) and returns the
  channel (83). The doc comment (47-49) explicitly notes "Tauri keeps the onmessage
  callback alive in an internal registry, so delivery does not depend on the reference
  being held" — i.e. the leak is known but unaddressed.
- `stopNativeScreenStream` (line 92-98) calls **only** `invoke("stop_screen_stream",
  { sectionIndex })`. **Nothing sets `channel.onmessage = null`; nothing unregisters
  the callback.** The function doesn't even have the channel reference.

**Frontend audio — `frontend/src/lib/native-system-audio.ts`:**
- `startNativeSystemAudioStream` (line 198-256) builds the audio nodes, stores a
  per-section `disconnect` in the module-level `cleanups` map (77, 211), creates
  `new Channel<ArrayBuffer>()` (213), sets `channel.onmessage` (218-246) — header
  parse → de-interleave → `pushChunk` → `ack` — then
  `invoke("start_system_audio_stream", { …, onChunk: channel })` (248-253) and returns
  the channel.
- `stopNativeSystemAudioStream` (line 259-266) runs the stored `disconnect()` (which
  detaches the AudioWorklet/ScriptProcessor nodes — 109, 180-184), deletes the
  `cleanups` entry, and `invoke("stop_system_audio_stream", { sectionIndex })`. So the
  **audio nodes ARE torn down**, but — same as screen — **`channel.onmessage` is never
  cleared and the callback is never unregistered.** The defect is identical on both
  paths; the audio side merely additionally cleans its Web Audio nodes.

**Consumers:**
- `frontend/src/components/asmr-recorder/preview.tsx` — `stopNativeStreamForSection`
  (line 240-247) calls `stopNativeScreenStream(index)` and nulls
  `nativeScreenChannels.current[index]`, `nativeCanvasRefs.current[index]`,
  `sectionRegions.current[index]`. It **drops the channel reference but does not
  unregister it.** Called from `handleClearSection` (562), `handleCameraSelect` (525),
  both `handleRegionConfirm` branches (395, 445), and the unmount cleanup effect
  (586-602, which iterates `nativeScreenChannels` and calls `stopNativeScreenStream`).
  The draw callback already guards a torn-down canvas (`if (!canvas) { bitmap.close();
  return; }`, 405) — so a late frame won't crash, but it still decodes + acks.
- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — system audio is a
  **single stream on section index 0**, started in `acquireAudioTrack`
  (`startNativeSystemAudioStream(0, …)`, line 666-674) with
  `nativeSystemAudioCleanupRef.current = () => stopNativeSystemAudioStream(0)` (672-673),
  invoked once in the recording-stop cleanup (1249-1252). One start/stop per recording
  session — so the audio leak grows once per recording, less aggressively than the
  per-reconfigure screen leak, but it is the same defect.

**Backend (already correct, for reference) — `src-tauri/src/screen_stream.rs` /
`system_audio_stream.rs`:** `stop_screen_stream` (screen 242-251) /
`stop_system_audio_stream` (audio 180-189) `remove` the `StreamHandle` from the
`HashMap` and call `handle.stop()` → sets `running=false`, joins the worker (screen
40-45, audio 37-42); the worker then calls `capture.stop()` and returns, dropping the
`credit` `Arc`. **The backend worker, SCStream, and credit state are fully released
today.** The leak is **purely frontend** — the registered `onmessage` closure. (The
backend `send` returning `Err` would `break` the worker, but since stop happens
backend-side first, the channel often isn't even dropped JS-side before the worker
exits.)

**Net finding:** the callback **IS leaked** on both paths — `cleanupCallback` never
runs because no `end` is ever sent and the JS `Channel` exposes no public unregister
(`cleanupCallback` is `private` in `core.d.ts` line 66; the public `unregister()` at
line 77 belongs to `PluginListener`, not `Channel`). The Rust side is clean. So the
fix is frontend-only on the JS object, with **one optional backend touch** (send a
terminal `end`-equivalent) considered and rejected below as over-engineering.

## Proposed design

Frontend-only, minimal. Stop holding a per-section list of channel references at the
transport-module level so `stop*` can neutralize the exact channel it created, and
neuter the handler so any in-flight/late delivery is a no-op.

Tauri's `Channel` has no public unregister in 2.9.1, but `onmessage` has a public
setter (core.js 120-122). Replacing it with a no-op means the registry entry still
exists (a tiny fixed object), but it **no longer retains** the heavy closure (`onFrame`,
canvas refs, audio nodes) and **no longer does work** (no `createImageBitmap`, no
`ack`, no `pushChunk`) when a stale frame lands. That removes the real costs (retained
graphics/audio objects, stale work against a torn-down canvas, misattributed acks).
The residual is one empty closure per cycle — negligible and unavoidable without a
backend `end` round-trip.

### 1. Track the created channel per section inside the transport module

`native-screen.ts`: add a module-level `const channels = new Map<number, Channel<ArrayBuffer>>()`.
In `startNativeScreenStream`, before `invoke`, `channels.set(sectionIndex, channel)`.
(Mirrors the existing `cleanups` map in `native-system-audio.ts`, line 77.)

### 2. Neuter + forget the channel on stop

```ts
// native-screen.ts
export async function stopNativeScreenStream(sectionIndex: number): Promise<void> {
  const channel = channels.get(sectionIndex);
  if (channel) {
    channel.onmessage = () => {};   // drop the closure; ignore in-flight/late frames
    channels.delete(sectionIndex);
  }
  try {
    await invoke("stop_screen_stream", { sectionIndex });
  } catch (e) {
    console.warn("[native-screen] stop failed:", e);
  }
}
```

`native-system-audio.ts` already has the `cleanups` map and runs `disconnect()` on
stop; add the same channel-tracking + neuter there:

```ts
// native-system-audio.ts — add alongside `cleanups`
const channels = new Map<number, Channel<ArrayBuffer>>();
// in start: channels.set(sectionIndex, channel) before invoke
// in stop:
const channel = channels.get(sectionIndex);
if (channel) { channel.onmessage = () => {}; channels.delete(sectionIndex); }
```

### 3. (Considered, rejected) backend `end` to truly unregister

We could make the worker send a terminal message so the JS `Channel` calls
`cleanupCallback`/`unregisterCallback` and frees the registry slot entirely. But the
Tauri `Channel` API exposes no "send end" from the Rust `Channel<InvokeResponseBody>`
side that we use, the registry entry is tiny once the closure is dropped, and this
would add a backend protocol change for no observable benefit. **Out of scope** — the
`onmessage` no-op already removes every real cost.

## Implementation steps

1. **`native-screen.ts` — track the channel.** Add `const channels = new Map<number,
   Channel<ArrayBuffer>>()` at module scope; `channels.set(sectionIndex, channel)` in
   `startNativeScreenStream` before the `invoke` (line ~74). **verify:** `npx tsc -b`
   clean (no new errors beyond the pre-existing ones noted in the dev logs).
2. **`native-screen.ts` — neuter on stop.** In `stopNativeScreenStream`, look up the
   channel, set `channel.onmessage = () => {}`, `channels.delete(sectionIndex)` before
   the existing `invoke("stop_screen_stream", …)`. **verify:** `npx tsc -b`; a manual
   start→stop→start cycle on a section leaves exactly one live `onmessage` (the new
   channel's) doing work.
3. **`native-system-audio.ts` — same treatment.** Add a `channels` map, set it in
   `startNativeSystemAudioStream`, and in `stopNativeSystemAudioStream` neuter +
   delete the channel (keep the existing `disconnect()` + `cleanups.delete` + backend
   stop). **verify:** `npx tsc -b`; record→stop→record twice and confirm no growth in
   active audio `onmessage` handlers.
4. **No consumer changes required.** `preview.tsx` `stopNativeStreamForSection` and the
   unmount effect already call `stopNativeScreenStream`; `recording-canvas.tsx`
   already calls `stopNativeSystemAudioStream(0)` — the new teardown rides those exact
   call sites. **verify:** `npx vite build` succeeds.

## Interface / API changes

- **No public signature changes.** `startNativeScreenStream` /
  `startNativeSystemAudioStream` still return `Promise<Channel<ArrayBuffer>>`;
  `stopNativeScreenStream` stays `async`, `stopNativeSystemAudioStream` stays sync.
- **Internal only:** a new module-level `channels: Map<number, Channel<ArrayBuffer>>`
  in each transport file (the audio one parallels the existing `cleanups` map).
- **No Rust changes.** Backend worker/SCStream/credit teardown is already correct.

## Edge cases & risks

- **In-flight frame after stop.** A `Raw` frame already dispatched to the old
  `channel.id` before stop runs, or a `createImageBitmap` that resolves after stop,
  now hits the no-op `onmessage` (or, for the bitmap, the existing `if (!canvas)`
  guard at preview.tsx 405) — it neither draws nor acks. Acceptable: the stream is
  stopping; the backend worker is exiting and no longer waits on that credit.
- **Double-stop.** `stopNativeStreamForSection` guards on
  `nativeScreenChannels.current[index]`, and the transport `stop*` now no-ops when the
  `channels` map has no entry. Calling stop twice is harmless (second call: map miss +
  a backend stop that's already a no-op, screen 246-249 / audio 184-187).
- **Channel reuse vs recreate.** We never reuse a `Channel`; every start mints a new
  one and `start_*` on the backend stops any existing handle for that section first
  (screen 118-121, audio 75-78). So a `channels.set` over an existing key during a
  rapid reconfigure can overwrite a not-yet-stopped entry — but the standard flow is
  stop-then-start (`stopNativeStreamForSection` precedes `startNativeScreenStream` in
  every `preview.tsx` path), so the old channel is neutered first. Even if overwritten,
  the orphaned old channel only loses its map entry (one residual empty registry slot,
  the pre-fix status quo) — never worse than today.
- **Ack after teardown.** A neutered `onmessage` never calls `ack`, so no late credit
  bump. Even if one slipped through, `ack_screen_frame`/`ack_system_audio_chunk` look
  up the *current* handle by `section_index` and cap at `MAX_INFLIGHT` (screen 258-270,
  audio 196-208), so a stray ack is harmless — same guarantee `[[native-screen-transport-raw-bytes]]`
  relies on for stale credit.
- **Registry slot not freed.** Setting `onmessage` to a no-op drops the closure but
  leaves a tiny entry in `__TAURI_INTERNALS__`. Truly freeing it needs a backend `end`
  (rejected above). The retained-object/stale-work costs — the ones that matter — are
  gone.

## Testing & verification

- **Build gates (auto):** `npx tsc -b` (expect only pre-existing errors), `npx vite build`.
  No new pure logic to unit-test — this is lifecycle wiring, not geometry/segment math
  (the shape `[[test-harness-and-ci]]` targets), so no vitest addition is warranted.
- **Manual (dev, can't auto-verify):**
  - Screen: assign a native-screen section, then clear/re-add (or re-pick display/region)
    several times. Before the fix, each cycle adds a live `onmessage`; after, only the
    current stream's handler runs. Sanity-check by logging in the handler (temporarily)
    or watching that `ack_screen_frame` is invoked by exactly one stream per section.
  - Audio: record→stop→record twice with system audio on; confirm no duplicate
    `pushChunk` into a stale node and no growth in active handlers, and that audio still
    records correctly (regression guard on the existing `disconnect()` path).
  - Confirm the backend still logs `[screen_stream] … stopped` / `[system_audio_stream]
    … stopped` on each stop (unchanged — backend teardown is untouched).

## Out of scope / future

- **Backend `end`-message protocol** to fully evict the registry slot (rejected above
  as over-engineering for a tiny residual).
- **Generalizing a shared "per-section Channel registry" helper** across the two
  transports — only two call sites; a shared abstraction is premature (`CLAUDE.md` §2).
- **Backpressure/transport redesign** — owned by `[[native-screen-transport-raw-bytes]]`.
- **System-audio node-graph hardening** (multi-section audio, AudioWorklet lifecycle) —
  see `[[native-system-audio-capture]]` and `[[system-audio-polish]]`.

## References

- `frontend/src/lib/native-screen.ts` — `startNativeScreenStream` (51-84), channel
  create (56), `channel.onmessage` (64-72), `ack` (60-62), `stopNativeScreenStream`
  (92-98, currently backend-only), registry-comment (47-49).
- `frontend/src/lib/native-system-audio.ts` — `startNativeSystemAudioStream` (198-256),
  `cleanups` map (77, 211), channel create (213), `channel.onmessage` (218-246),
  `stopNativeSystemAudioStream` (259-266), node `disconnect()` (109, 180-184).
- `frontend/src/components/asmr-recorder/preview.tsx` — `stopNativeStreamForSection`
  (240-247), `nativeScreenChannels` ref (88-90), `handleClearSection` (554-569),
  region/camera stop sites (395, 445, 525), unmount cleanup effect (586-602),
  draw-callback canvas guard (405).
- `frontend/src/components/asmr-recorder/recording-canvas.tsx` —
  `startNativeSystemAudioStream(0, …)` (666-674), `nativeSystemAudioCleanupRef`
  (257, 672-673), stop in recording cleanup (1249-1252).
- `src-tauri/src/screen_stream.rs` — `stop_screen_stream` (242-251), `StreamHandle::stop`
  (40-45) + `Drop` (48-57), `ack_screen_frame` (258-270), start-time stop-existing
  (118-121).
- `src-tauri/src/system_audio_stream.rs` — `stop_system_audio_stream` (180-189),
  `SysAudioHandle::stop` (37-42), `ack_system_audio_chunk` (196-208).
- `@tauri-apps/api@2.9.1` `core.js` — `Channel` (74-133): `transformCallback`
  registration (82, 69-73), `cleanupCallback` → `unregisterCallback` (117-118), the
  `end`-triggered cleanup (84-91, 106-108), public `onmessage` setter (120-122);
  `core.d.ts` — `Channel` surface (61-71, `cleanupCallback` `private` at 66) vs
  `PluginListener.unregister()` (77).
- Deferring plan: `[[native-screen-transport-raw-bytes]]` (this item, "Out of scope /
  future", line 335; ack/credit mechanism it introduced). Related:
  `[[native-system-audio-capture]]`, `[[system-audio-polish]]`,
  `[[next-implementations]]`.
