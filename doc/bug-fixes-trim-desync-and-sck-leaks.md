# Feature: Fix trim frame-accurate audio desync + native SCK system-audio stream leaks

**Priority:** P1  ·  **Effort:** M  ·  **Risk:** med  ·  **Surface:** frontend (trim editor + recording transport/audio lifecycle)

## Problem / motivation

Three verified defects share the audio/transport surface and are fixed together:

- **Bug A (HIGH, headline):** the frame-accurate trim export plays audio **ahead of**
  video by up to one ~3 s GOP. It fires exactly in the feature's intended case — a
  frame-accurate cut whose in-point is *not* on a keyframe. The audio is rebased
  against a different time origin than the video, so output time 0 carries audio that
  belongs ~(seg.in − keyframe) seconds earlier than the first video frame.
- **Bug B (MED):** stopping a recording *during* the async audio-acquire window orphans
  the native ScreenCaptureKit system-audio session (and, in one sub-case, the
  `AudioContext`). The "cancelled" early-return only stops the mixed track; it never
  runs the native cleanup. The leaked SCK session lives until app exit and accumulates
  across quick record/stop cycles.
- **Bug C (MED, overlaps B):** the system-audio monitor (SCK stream on reserved index
  99) can leak an un-stoppable backend stream when its start is interrupted mid-flight
  while recording starts on index 0 — a *cross-index* race the backend's same-index
  eviction can't self-heal.

The recurring "only one SCK stream at a time" claim (`use-system-audio-monitor.ts`
26-27, `recording-context.tsx` 84-85) is an **unverified code comment** — there is no
evidence in the Rust SCK layer that concurrent streams are rejected. So B/C's impact is
bounded *if* SCK tolerates concurrent streams (then it's a pure resource leak), but
worst-case (if SCK serializes/contends) it could starve the live recording's audio. We
fix the leaks regardless and note the assumption explicitly.

## Current state (grounded)

### Bug A — `frontend/src/components/asmr-recorder/trim-editor.tsx`, `trimToBuffer` (69-205)

Per keep-segment, the code computes a single `origin` and `segOffset` (129-135):

```ts
const startKey = await videoSink.getKeyPacket(seg.in, { verifyKeyPackets: true }); // 117
let audioStart: EncodedPacket | null = null;
if (audioSink) audioStart = await audioSink.getPacket(startKey.timestamp);          // 123
const origin = Math.min(startKey.timestamp, audioStart?.timestamp ?? startKey.timestamp); // 129-132
const segOffset = timelineOffset;                                                  // 135
```

- **Lossless / else branch (168-180):** video is rebased `p.timestamp - origin + segOffset`
  (172). Audio (185-200) is rebased with the **same** base `p.timestamp - origin + segOffset`
  (192). Both share `origin` → **correctly aligned**.
- **Frame-accurate branch (138-167):** entered only when `canFrameAccurate &&
  startKey.timestamp < seg.in - 0.001` (138) — i.e. the in-point is strictly *after* the
  preceding keyframe. Here video is rebased against **`seg.in`**, not `origin`:
  re-encoded GOP `rp.timestamp - seg.in + segOffset` (147) and copied tail
  `p.timestamp - seg.in + segOffset` (159). So output time 0 == clip time `seg.in`
  for **video**.
- **Audio is still rebased against `origin`** (192) regardless of branch, and
  `audioStart` was fetched from `getPacket(startKey.timestamp)` (123) where
  `startKey.timestamp ≤ seg.in`. The 138 guard guarantees `startKey.timestamp < seg.in`,
  and `origin = min(startKey.timestamp, audioStart.timestamp) ≤ startKey.timestamp <
  seg.in`. Net: **audio output starts at clip time `origin`, video at `seg.in`** →
  audio **leads** video by `(seg.in − origin)`.

**Max skew:** the recorder writes a keyframe every `KEYFRAME_INTERVAL_SECONDS = 3`
(`recording-canvas.tsx` 90; enforced in the encode loop at 1221-1226 via
`Math.round(frameRateRef.current * KEYFRAME_INTERVAL_SECONDS)`). The preceding keyframe
is therefore up to ~3 s before `seg.in`, so the desync is **up to ~3 s** of audio
lead (plus the audio packet may start slightly before that keyframe, so technically up
to ~3 s + one audio-packet duration).

**Why the else branch is fine but this isn't:** the else branch keeps video anchored to
`origin` (the same anchor as audio), so the relative A/V offset is preserved. The
frame-accurate branch deliberately shifts video to `seg.in` to drop the partial GOP's
pre-roll frames, but forgot to shift audio by the same amount.

**AAC granularity constraint:** AAC frames are 1024 samples; at 48 kHz that's
1024/48000 ≈ **21.3 ms** per packet, and a packet can't be sub-divided without
re-encoding. So audio can be aligned to `seg.in` only to within one packet boundary.

### Bug B — `frontend/src/components/asmr-recorder/recording-canvas.tsx`

The recording effect (1259-1497) declares `let cancelled = false` (1264) and runs
`acquireAudioTrack(...)` (1285) which `await`s. The cancelled early-return (1296-1299):

```ts
if (cancelled) {
  if (audioTrack) audioTrack.stop();   // 1297
  return;                              // 1298
}
```

`acquireAudioTrack` (592-751) on the native-system-audio path:
- creates `audioContextRef.current = audioContext` (679),
- `await`s mic `getUserMedia` (611) when mic is wanted,
- `await`s `startNativeSystemAudioStream(0, …)` (732-738),
- assigns `nativeSystemAudioCleanupRef.current = () => stopNativeSystemAudioStream(0)`
  **only after** that await (739-740),
- builds `audioStreamRef.current` (743-746).

The effect's normal cleanup (1370-1489) is what releases everything:
`nativeSystemAudioCleanupRef.current()` (1390-1393), `audioStreamRef` track stops
(1473-1476), `audioContextRef.current.close()` (1477-1480). **The cancelled branch
(1296-1299) calls none of these.** Two sub-cases when cleanup sets `cancelled = true`
(1371) before `acquireAudioTrack` resolves:

1. **Stop races the mic await (611):** `audioContextRef.current` is set (679) but the
   native stream hasn't started and the cleanup ref is null → if execution later
   reaches 732 it still starts the SCK stream after `cancelled` is true (no guard
   between 679 and 732). On the early return at 1296 only `audioTrack` (null here)
   would be stopped → **AudioContext + SCK both leak.**
2. **Stop races the SCK await (732):** `startNativeSystemAudioStream(0)` resolves and the
   backend stream is live, but `nativeSystemAudioCleanupRef.current` is assigned at 739
   *after* the await — if cancellation arrives between resolve and 739, or before 1302
   wiring, the cancelled branch returns having stopped only the track. The cleanup ref
   is null → **the SCK session leaks** (the AudioContext may also leak unless it was the
   one closed by a later normal cleanup — but a cancelled start never reaches the normal
   cleanup's resource release for *this* run).

The durable leak is the **backend SCK session**: `system_audio_stream.rs` keeps the
worker + SCStream alive in `state.streams` (167-177) until `stop_system_audio_stream`
removes+stops it (188-198). Nothing else evicts index 0.

### Bug C — `frontend/src/lib/native-system-audio.ts` + `use-system-audio-monitor.ts`

`startNativeSystemAudioStream` (202-263) has **no internal cancellation guard** — it
runs `buildAudioSource` (210), registers `cleanups`/`streamChannels` (215, 252), then
`await invoke("start_system_audio_stream", …)` (254-260) and returns. *(Correction to
the original finding: the "cancelled flag at ~72/103" lives in the **monitor hook**, not
in this file — this file checks nothing.)*

`stopNativeSystemAudioStream` (266-278) is **synchronous, returns `void`**, neuters the
channel, runs the node `disconnect()`, and fires `invoke("stop_system_audio_stream", …)`
**un-awaited** (277).

The monitor hook `use-system-audio-monitor.ts` runs the SCK stream on `MONITOR_SECTION =
99` (9). Its effect has `let cancelled = false` (41) and `teardown()` →
`stopNativeSystemAudioStream(MONITOR_SECTION)` (44-48). The effect deps are
`[enabled, appBundleId]` (116), and `enabled = captureSystemAudio && !isExternalRecording`
(`recording-context.tsx` 88-89). When recording starts, `isExternalRecording` flips
true → the monitor effect re-runs → cleanup (111-115) sets `cancelled = true` and calls
`teardown()` → `stopNativeSystemAudioStream(99)`.

**The cross-index orphan:** the monitor's `startNativeSystemAudioStream(99)` may still be
*in its `await invoke("start_system_audio_stream")`* (254) when teardown fires
`stopNativeSystemAudioStream(99)`. On the backend, `start_system_audio_stream` only
inserts the handle into `state.streams` at **167-177** — *after* `capture.start()` (97)
and the worker spawn (105). If the teardown's `stop_system_audio_stream` (188-198) runs
its `remove(&99)` **before** that insert lands, it removes nothing, and the start then
inserts a live handle for 99 that **no one will ever stop** → orphaned SCK stream 99,
concurrent with the recording's freshly-started stream 0.

Same-index rapid toggles self-heal because `start_system_audio_stream` evicts an
existing same-index handle on entry (81-84) and again on insert (167-177). The
cross-index case (99 vs 0) gets no such eviction. The un-awaited monitor stop (277) also
means stream 99's backend worker may still be in its `RECV_TIMEOUT`-paced shutdown
(`RECV_TIMEOUT = 100ms`, 24) when stream 0 starts.

### Already-correct context (for reference)

- `native-system-audio.ts` channel neutering (`streamChannels` 81, neuter 269) is
  **already shipped** (the `[[per-section-transport-unregister]]` plan landed); we do
  not touch it.
- Backend same-index eviction: `system_audio_stream.rs` 81-84 + 167-177;
  `screen_stream.rs` mirrors it (118, 220, 246). The leak is frontend-orchestration, not
  a backend stop bug.

## Proposed design

Surgical, frontend-only. Three independent fixes; A is the urgent one.

### 1. Bug A — anchor frame-accurate audio to `seg.in`

Extract the per-stream rebase into a pure function so it is testable and the two
branches can't drift apart again, then use it for audio in the frame-accurate branch
with a `seg.in` anchor and a boundary drop.

Add to `trim-segments.ts` (it already holds the pure trim math + has tests):

```ts
/** Rebase a source packet timestamp onto the output timeline. */
export function rebaseTimestamp(srcTs: number, anchor: number, segOffset: number): number {
  return srcTs - anchor + segOffset;
}

/**
 * Should this audio packet be kept given a frame-accurate in-point?
 * AAC packets are ~21ms and can't be sub-trimmed, so keep the first packet whose
 * END crosses seg.in (accept up to ~one-packet pre-roll) and drop fully-earlier ones.
 */
export function keepAudioPacket(pktStart: number, pktDuration: number, segIn: number): boolean {
  return pktStart + pktDuration > segIn;
}
```

In `trimToBuffer`, the **frame-accurate** branch chooses the audio anchor = `seg.in`
(video's anchor), and drops audio packets that end before `seg.in`:

```ts
const audioAnchor = (canFrameAccurate && startKey.timestamp < seg.in - 0.001)
  ? seg.in
  : origin;
...
for await (const p of audioSink.packets(audioStart)) {
  if (p.timestamp > seg.out) break;
  if (audioAnchor === seg.in && !keepAudioPacket(p.timestamp, p.duration ?? 0, seg.in)) {
    continue;                                   // drop pre-in-point packets
  }
  await audioSource.add(
    new mb.EncodedPacket(p.data, p.type,
      rebaseTimestamp(p.timestamp, audioAnchor, segOffset),  // was: p.timestamp - origin + segOffset
      p.duration, p.sequenceNumber),
    firstAudio ? audioMeta : undefined,
  );
  firstAudio = false;
}
```

Result: in the frame-accurate branch both audio and video use `seg.in` as anchor →
output time 0 == `seg.in` for both. The first kept audio packet may start up to ~21 ms
before `seg.in` (it's the straddling packet), so audio leads by **at most one AAC
packet (~21 ms)** instead of up to ~3 s — and never trails. The else branch is
untouched (`audioAnchor === origin`, identical to today). **Boundary policy (chosen):**
keep the straddling packet, accept ≤~21 ms residual pre-roll; do **not** re-encode the
boundary (rejected in Out of scope).

### 2. Bug B — make the cancelled branch run native cleanup

Mirror the normal cleanup's resource release in the cancelled early-return so a stop that
races the acquire window can't orphan the SCK session or AudioContext:

```ts
if (cancelled) {
  if (audioTrack) audioTrack.stop();
  if (nativeSystemAudioCleanupRef.current) {           // close SCK if it started
    nativeSystemAudioCleanupRef.current();
    nativeSystemAudioCleanupRef.current = null;
  }
  if (audioStreamRef.current) {
    audioStreamRef.current.getTracks().forEach((t) => t.stop());
    audioStreamRef.current = null;
  }
  if (audioContextRef.current) {
    audioContextRef.current.close().catch(() => {});
    audioContextRef.current = null;
  }
  return;
}
```

This covers sub-case 2 (SCK started, cleanup ref assigned at 739-740 → now stopped) and
sub-case 1's AudioContext (set at 679 → now closed). It does **not** cover the narrow
window where `cancelled` is set between `audioContextRef.current = audioContext` (679)
and `startNativeSystemAudioStream` (732), because that path still runs the start *after*
cancellation. To close that, also assign the cleanup ref defensively **before** the
await isn't possible (the ref needs the section index, which is constant `0`), so add a
guard inside `acquireAudioTrack` instead: this is fix #2b below. Given CLAUDE.md
simplicity-first, the cancelled-branch cleanup above is the primary fix; #2b is a small
add that closes the residual window.

### 2b. Bug B residual — guard the native start against a stop already in flight

`acquireAudioTrack` can't see the effect's `cancelled` (different scope). The cheapest
correct closure is to **assign the cleanup ref immediately after the awaited start
resolves** (already done at 739-740) and have the cancelled branch run it (#2). The only
remaining gap is the start *itself* firing after cancellation. Bug C's fix (#3) — a
post-await guard inside `startNativeSystemAudioStream` keyed on a per-call cancel token —
also benefits index 0; but for the recording path the simpler, sufficient fix is #2's
cleanup branch (the SCK session is stopped a few ms after it starts). **Recommend: ship
#2 alone for Bug B; do not thread a cancel token into the recording path** (over-engineering
for a ~ms window — CLAUDE.md §2). Document the residual in Edge cases.

### 3. Bug C — re-check cancellation after the await + await the stop

Two small changes:

**3a. Make `stopNativeSystemAudioStream` await the backend stop:**

```ts
export async function stopNativeSystemAudioStream(sectionIndex: number): Promise<void> {
  const channel = streamChannels.get(sectionIndex);
  if (channel) { channel.onmessage = () => {}; streamChannels.delete(sectionIndex); }
  const disconnect = cleanups.get(sectionIndex);
  if (disconnect) { disconnect(); cleanups.delete(sectionIndex); }
  await invoke("stop_system_audio_stream", { sectionIndex }).catch(() => {});
}
```

Callers: `use-system-audio-monitor.ts` `teardown` (44-48) and the recording cleanup ref
(`recording-canvas.tsx` 739-740) are fire-and-forget; making the function async is
backward-compatible (returning a promise they ignore). No required caller edits, but the
monitor `teardown` may optionally `void`-mark it.

**3b. Have `startNativeSystemAudioStream` accept an abort signal and tear itself down if
cancelled after the await** — so a cross-index stop that lost the race to the backend
insert still gets cleaned up:

```ts
export async function startNativeSystemAudioStream(
  sectionIndex, audioCtx, destination, opts,
  signal?: AbortSignal,                         // new optional param
): Promise<Channel<ArrayBuffer>> {
  ...
  await invoke("start_system_audio_stream", { ... });
  if (signal?.aborted) {                        // stop raced the start → self-heal
    await stopNativeSystemAudioStream(sectionIndex);
  }
  return channel;
}
```

`use-system-audio-monitor.ts` passes a controller it aborts in cleanup:

```ts
const ac = new AbortController();
...
await startNativeSystemAudioStream(MONITOR_SECTION, ctx, analyserInput, {...}, ac.signal);
...
return () => { cancelled = true; ac.abort(); setState(IDLE); teardown(); };
```

Now if teardown's `remove(&99)` ran before the backend insert (the orphan window), the
post-await `signal.aborted` check fires a *second* `stop_system_audio_stream(99)` after
the insert has landed → removes+stops the orphan. The recording path (#2) does **not**
need the signal — its #2 cleanup already stops index 0.

## Implementation steps

1. **Bug A pure helpers + tests first.** Add `rebaseTimestamp` and `keepAudioPacket` to
   `trim-segments.ts`. Add a `describe("rebaseTimestamp" / "keepAudioPacket")` block to
   `trim-segments.test.ts` covering: video/audio share an anchor in the lossless case
   (equal output ts); frame-accurate audio anchored to `seg.in` removes the lead;
   straddling packet kept, fully-earlier packet dropped; ~21 ms residual bound.
   **verify:** `npm run test` — new cases pass, existing 48 still green.
2. **Bug A wire-up.** In `trimToBuffer` (185-200) compute `audioAnchor` (= `seg.in` only
   when the frame-accurate branch was taken, else `origin`), replace the inline
   `p.timestamp - origin + segOffset` (192) with `rebaseTimestamp(p.timestamp,
   audioAnchor, segOffset)`, and `continue` on `!keepAudioPacket(...)` when anchored to
   `seg.in`. Leave video (147, 159, 172) and the else branch untouched.
   **verify:** `npx tsc -b` clean (only pre-existing errors); `npx vite build` succeeds.
3. **Bug A manual A/V sync.** Record ≥10 s of clappable audio + visible motion, set a
   frame-accurate in-point a few hundred ms after a keyframe (so 138 is true), export,
   and confirm the clap lines up with the visual within one frame (no ~3 s lead).
   **verify:** play exported MP4; audio no longer precedes video.
4. **Bug B cancelled branch.** In `recording-canvas.tsx` extend the cancelled
   early-return (1296-1299) to also stop `nativeSystemAudioCleanupRef`, `audioStreamRef`
   tracks, and `audioContextRef`, nulling each (per §2). **verify:** `npx tsc -b`;
   manual — start a recording with system audio on and click Stop within ~1 s
   (during permission/SCK-start), repeat ~5×, then confirm backend logs show one
   `[system_audio_stream] section 0 stopped` per start with no growing count of live
   sessions.
5. **Bug C — async stop.** Make `stopNativeSystemAudioStream` `async` and `await` the
   `invoke` (§3a). **verify:** `npx tsc -b` — callers compile (return value already
   ignored). `npm run test` green.
6. **Bug C — abort guard.** Add the optional `signal` param + post-await
   `signal?.aborted` self-stop to `startNativeSystemAudioStream` (§3b); wire an
   `AbortController` through `use-system-audio-monitor.ts` (abort in cleanup, pass
   `ac.signal`). **verify:** `npx tsc -b`; `npx vite build`; manual — toggle system
   audio on (monitor starts on 99), immediately start recording (recording starts 0 +
   monitor tears down 99), repeat ~5×; confirm backend logs show every `section 99
   started` paired with a `section 99 stopped` and no orphaned 99 stream alongside 0.
7. **Log the session** per CLAUDE.md §5 in `docs/logs/2026-06-15.md` (append).
   **verify:** file updated with what/why + the SCK-concurrency assumption note.

## Interface / API changes

- `trim-segments.ts`: **new exports** `rebaseTimestamp(srcTs, anchor, segOffset)` and
  `keepAudioPacket(pktStart, pktDuration, segIn)` (pure, used by `trimToBuffer` + tests).
- `native-system-audio.ts`:
  - `stopNativeSystemAudioStream` return type `void → Promise<void>` (callers ignore it
    today; backward-compatible).
  - `startNativeSystemAudioStream` gains an optional trailing `signal?: AbortSignal`.
- `recording-canvas.tsx`: no signature change — only the cancelled-branch body grows.
- `use-system-audio-monitor.ts`: internal — adds an `AbortController`; no exported API
  change.
- **No Rust / backend changes.** The backend stop/evict logic is already correct.

## Edge cases & risks

- **Bug A — segment whose in-point IS on a keyframe.** Then 138 is false → else branch →
  `audioAnchor === origin`, identical to today. No behavior change for keyframe-aligned
  or non-frame-accurate cuts. The fix only alters the exact case that was broken.
- **Bug A — multi-segment exports.** Each segment recomputes `origin`/`segOffset`; the
  per-segment `audioAnchor` keeps audio and video joined at the same `segOffset`, so
  segment boundaries stay gapless (the else branch already relied on this; the
  frame-accurate branch now matches).
- **Bug A — ≤~21 ms residual audio pre-roll.** The kept straddling AAC packet may begin
  up to ~21 ms before `seg.in`. This is sub-frame at typical fps and inaudible as
  "lead"; sample-exact alignment would require re-encoding the boundary packet (rejected,
  Out of scope). Documented as the accepted boundary policy.
- **Bug A — audio shorter than video at the cut.** If the first kept packet starts after
  `seg.in` (gap), output simply has a ≤21 ms silent head — acceptable and far better
  than a 3 s lead.
- **Bug B — residual start-after-cancel window.** §2b: `cancelled` set between
  `audioContextRef.current = audioContext` (679) and the SCK start (732) still starts the
  stream. The §2 cleanup runs the cleanup ref (assigned at 739-740) on the *next* effect
  run if the component stays mounted; if it doesn't, the stream is stopped by the §2
  branch only when the ref was set. This window is sub-millisecond in practice; we accept
  it rather than thread a cancel token into the hot recording path (CLAUDE.md §2).
- **Bug C — double stop.** §3b can fire `stop_system_audio_stream(99)` twice (teardown +
  post-await guard). The backend `remove`+`stop` is idempotent (188-198: second call's
  `remove` returns `None`). Harmless.
- **SCK concurrency assumption (UNVERIFIED).** The "one SCK stream at a time" comments
  (`use-system-audio-monitor.ts` 26-27, `recording-context.tsx` 84-85) are unverified.
  If SCK *allows* concurrent streams, B/C are pure resource leaks (fixed here). If SCK
  *serializes/contends*, the orphaned 99 stream could degrade recording-0 audio until app
  exit — making these fixes more than hygiene. Either way the fix is correct; we do not
  rely on the assumption.
- **`stopNativeSystemAudioStream` now async.** A caller that previously relied on its
  synchronous side effects (node `disconnect()`) still gets them synchronously — only the
  backend `invoke` is now awaitable. No caller awaited it before, so no ordering change.

## Testing & verification

**Build gates (auto):**
- `npx tsc -b` — expect only the pre-existing errors noted in the dev logs.
- `npx vite build` — succeeds.
- `npm run test` (vitest) — currently 48 tests pass; the new `rebaseTimestamp` /
  `keepAudioPacket` cases (Bug A math) raise the count and pass.
- `cargo build` (from `src-tauri/`, via `npm run tauri` which injects
  `DYLD_LIBRARY_PATH=/usr/lib/swift`) — unchanged (no Rust edits), still compiles.

**Manual (dev — can't auto-verify):**
- **Bug A:** frame-accurate trim with a non-keyframe in-point; clap/visual sync within
  one frame, no ~3 s audio lead. Compare against a lossless (frame-accurate off) export
  to confirm the else branch is unchanged.
- **Bug B:** record with system audio on, Stop within ~1 s of pressing Record, ×5;
  watch the Tauri console for one `[system_audio_stream] section 0 stopped` per record
  and no accumulating live sessions; confirm no AudioContext warnings pile up.
- **Bug C:** toggle system audio on (monitor 99 starts) then immediately Record (monitor
  99 tears down, recording 0 starts), ×5; confirm every `section 99 started` is paired
  with `section 99 stopped` and only stream 0 is live during recording.

**Vitest plan for Bug A (the pure offset calc):** in `trim-segments.test.ts`,
mirroring its existing `describe`/`it` style —
- `rebaseTimestamp`: `rebaseTimestamp(5, 5, 0) === 0`; `rebaseTimestamp(5, 2, 10) === 13`;
  lossless parity — video and audio fed the same `(anchor, segOffset)` yield equal output
  ts for equal input ts.
- frame-accurate alignment: with `anchor = seg.in`, an audio packet at `seg.in` maps to
  `segOffset` (no lead); a packet at the preceding keyframe (`< seg.in`) would map
  negative → asserts it's dropped by `keepAudioPacket`.
- `keepAudioPacket`: packet fully before `seg.in` (`start+dur ≤ segIn`) → false; straddling
  (`start < segIn < start+dur`) → true; at/after → true; assert the kept straddling
  packet's lead ≤ its duration (~21 ms at 48 kHz / 1024 samples).

## Out of scope / future

- **Re-encoding the boundary AAC packet** for sample-exact frame-accurate audio. The
  ≤~21 ms residual is inaudible for ASMR trims and re-encoding the boundary adds an
  encoder round-trip + drift risk for no perceptible gain (CLAUDE.md §2). Rejected.
- **Threading an AbortSignal through the recording-canvas audio path (index 0).** The
  §2 cancelled-branch cleanup already stops the index-0 SCK session; a cancel token would
  be speculative complexity in the hot path. Rejected; the sub-ms residual window is
  documented instead.
- **Verifying / enforcing the "one SCK stream at a time" invariant in Rust** (e.g. a
  global SCK lock or a single shared capture multiplexed to sections). Larger design
  question, not required to fix these leaks. Noted as an open assumption.
- **Generalizing a shared start/stop lifecycle helper** across screen + system-audio
  transports — only the audio path needs the abort guard today; a shared abstraction is
  premature.

## References

- `frontend/src/components/asmr-recorder/trim-editor.tsx` — `trimToBuffer` (69-205),
  `getKeyPacket` (117), `audioStart = getPacket(startKey.timestamp)` (123),
  `origin = min(...)` (129-132), `segOffset` (135), frame-accurate guard (138), video
  rebase `- seg.in` (147, 159), else-branch video rebase `- origin` (172), audio rebase
  `- origin` (185-200, the bug at 192), `KEYFRAME` re-encode helper `reencodeLeadingGop`
  (26-58).
- `frontend/src/components/asmr-recorder/trim-segments.ts` — `Segment` (2), `MIN_SEG` (5),
  `splitSegment` (20-30), `keptDuration` (33-35); target for new `rebaseTimestamp` /
  `keepAudioPacket`.
- `frontend/src/components/asmr-recorder/trim-segments.test.ts` — existing vitest pattern
  (1-40+) to extend.
- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — `KEYFRAME_INTERVAL_SECONDS = 3`
  (90), keyframe enforcement (1216-1226), refs `audioStreamRef`/`audioContextRef` (285-286),
  `nativeSystemAudioCleanupRef` (288), `acquireAudioTrack` (592-751): mic await (611),
  `audioContextRef.current = audioContext` (679), native start (732-738), cleanup-ref
  assign (739-740), `audioStreamRef` build (743-746); recording effect `cancelled` (1264),
  cancelled branch (1296-1299, the Bug B gap), normal cleanup (1370-1489) incl. native
  cleanup (1390-1393), track stop (1473-1476), context close (1477-1480).
- `frontend/src/lib/native-system-audio.ts` — `streamChannels` (81, already-shipped
  neuter), `startNativeSystemAudioStream` (202-263, no cancel guard), `await invoke`
  (254-260), `stopNativeSystemAudioStream` (266-278, sync, un-awaited invoke at 277).
- `frontend/src/hooks/use-system-audio-monitor.ts` — `MONITOR_SECTION = 99` (9),
  `cancelled` (41), `teardown` (44-48), start (61-70), post-await `cancelled` checks
  (72, 103), deps `[enabled, appBundleId]` (116).
- `frontend/src/contexts/recording-context.tsx` — `useSystemAudioMonitor(captureSystemAudio
  && !isExternalRecording, ...)` (88-89), "one SCK stream at a time" comment (84-85),
  `isExternalRecording` (76).
- `src-tauri/src/system_audio_stream.rs` — `RECV_TIMEOUT = 100ms` (24), same-index evict
  on entry (81-84), worker spawn (105-165), insert handle (167-177),
  `stop_system_audio_stream` remove+stop (188-198), `ack` (205-217).
- `src-tauri/src/screen_stream.rs` — parallel evict/insert/stop (118, 220, 246).
- Related plans: `[[per-section-transport-unregister]]` (channel-neuter, already
  shipped), `[[native-system-audio-capture]]`, `[[system-audio-polish]]`,
  `[[next-implementations]]`.
