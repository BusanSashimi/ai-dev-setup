# Feature: System-audio polish — per-app capture, mic/system faders, pre-record monitor

**Priority:** P2  ·  **Effort:** M–L  ·  **Risk:** med  ·  **Surface:** both (Rust SCK + frontend audio/UI)

## Problem / motivation

`[[native-system-audio-capture]]` shipped the core: SCK captures system audio, streams raw f32
PCM over a Tauri `Channel`, and the frontend mixes it into the recording `AudioContext` alongside
the mic. Three rough edges remain — they were explicitly deferred to "Out of scope / future" in
that plan (`native-system-audio-capture.md:262-264`):

1. **System audio is all-or-nothing.** SCK captures the *whole* system mix (every app's output).
   For an ASMR/streaming recorder you often want only one app — the game, the browser tab, the
   instrument plugin — without Slack pings, Mail chimes, or notification sounds bleeding in. SCK
   *can* filter by application; we just don't expose it.
2. **The faders are 80% there but under-labeled and lack mute/solo.** Independent gain already
   ships (see Current state) — this item is mostly *don't re-plan shipped work*, plus the small
   missing pieces (mute/solo, dB labels, per-source meters during record).
3. **You can't hear-check system audio before recording.** The pre-record monitor
   (`use-audio-monitor.ts`) is **mic-only** and gated to `captureMic`. There's no way to verify
   the system-audio level is sane until you've recorded and played back — which for SCK is
   especially error-prone (wrong app, app muted, app outputting silence).

Goal: add per-app capture, finish the faders (mute/solo + labels + record-time meters), and add a
system-audio pre-record monitor — all building on the shipped native path, no transport changes.

## Current state (grounded)

**Backend — SCK capturer captures the whole first display (`src-tauri/src/system_audio_macos.rs`):**
- `start()` (line 67-120) gets `SCShareableContent::get()` (79), takes `displays().first()` (81-82),
  and builds `SCContentFilter::create().with_display(display).with_excluding_windows(&[]).build()`
  (84-87). **No app filter** — every audio-producing app on that display is captured.
- `SystemAudioCaptureConfig { sample_rate, channels }` (`system_audio.rs:11-16`); `AudioChunk
  { samples: Vec<f32>, sample_rate, channels, timestamp }` (`system_audio.rs:2-8`). **No app field.**
- `SCStreamConfiguration::new().with_captures_audio(true).with_sample_rate(...).with_channel_count(...)`
  (89-92).

**Backend — the SCK crate already supports per-app filtering (`screencapturekit` 1.5.0):**
- `SCShareableContent::applications() -> Vec<SCRunningApplication>` (`shareable_content/mod.rs:265-281`);
  `SCRunningApplication` exposes `application_name()`, `bundle_identifier()`, `process_id()`
  (`running_application.rs:61-81`).
- `SCContentFilterBuilder::with_including_applications(&[&SCRunningApplication], &[&SCWindow])`
  (`stream/content_filter.rs:494-509`) — **not feature-gated**, builds
  `sc_content_filter_create_with_display_including_applications_excepting_windows`. This wraps
  Apple's `SCContentFilter(display:including:exceptingWindows:)`, available since macOS 12.3, so it
  is **available at our macOS 14.0 floor** (`tauri.conf.json:41 minimumSystemVersion "14.0"`;
  `Cargo.toml:44 screencapturekit = { version = "1", features = ["async","macos_14_0"] }`).
- ⚠️ This is a **video/window** content filter; whether SCK routes only the included apps'
  *audio* through it is the one thing that needs a runtime check (see Edge cases).

**Backend — transport + registration (unchanged, from `[[native-system-audio-capture]]`):**
- `start_system_audio_stream(section_index, sample_rate, channels, on_chunk, state)` /
  `stop_system_audio_stream` / `ack_system_audio_chunk` (`system_audio_stream.rs:67-208`), 16-byte
  LE header + interleaved f32 (`system_audio_stream.rs:59-66`, parsed by `audio-wire.ts:26-35`).
- Registered in `lib.rs:89-91`; `SystemAudioStreamState` managed at `lib.rs:81`.
- `recording.rs:get_available_devices` sets `has_system_audio = is_system_audio_available()`
  (`recording.rs:45`; `system_audio_macos.rs:136-138` returns `true`). **No app-listing command.**

**Frontend — FADERS ARE ALREADY SHIPPED. Do not re-implement:**
- `ExternalRecordingConfig` has `micGain` and `systemAudioGain` (0–3, default 1.0)
  (`types/recording.ts:80,86,108,110`).
- `recording-canvas.tsx` inserts a per-source `GainNode`: `micGainNodeRef` (mic, after the
  high-pass `BiquadFilterNode`) at lines 644-649; `systemAudioGainNodeRef` for both the
  `getDisplayMedia` (652-659) and native-SCK (661-665) paths. The native path connects the SCK
  worklet output **into** that gain node (666-668).
- The prop-sync effect live-ramps both gains via `setTargetAtTime(value, t, 0.02)` (288-291) — so
  dragging a slider mid-record is already smooth and click-free. The high-pass toggles by
  switching `type` to `"allpass"` (292-295).
- `toolbar.tsx` already renders both sliders: **Mic Gain** (287-303) and **System Audio Gain**
  (345-361), each `min=0 max=3 step=0.05` with a `Math.round(gain*100)%` label, calling
  `updateExternalConfig({...})`. `externalConfig` persists to `localStorage` (`recording-context.tsx:60-91`).
- The System Audio toggle is gated on `devices.hasSystemAudio` (`toolbar.tsx:329,341`).

  **So independent mic/system faders with live, persisted, click-free gain already exist.**
  **Missing only:** per-source **mute/solo**, **dB labels** (today is %), and **per-source level
  meters while recording** (the meters today are pre-record monitor only — see below).

**Frontend — pre-record monitor is MIC-ONLY (`use-audio-monitor.ts`):**
- `useAudioMonitor(enabled, deviceId)` opens its **own** `getUserMedia` mic stream (67-76),
  decoupled from the recording pipeline (the docstring, 4-14), and returns `{ active, error,
  analyserL, analyserR, analyserMix }` (18-28). It early-returns if `!enabled` (42-45) or
  `!hasMediaApi("getUserMedia")` (46-49).
- Wired once in `recording-context.tsx:77-80` as `useAudioMonitor(externalConfig.captureMic,
  externalConfig.micDeviceId)` → exposed as `audioMonitor`. `toolbar.tsx` draws a `StereoMeter`
  from `audioMonitor.analyserL/R` **only when `externalConfig.captureMic`** (246-252).
- There is **no system-audio monitor**: the SCK stream starts only inside `acquireAudioTrack`
  (`recording-canvas.tsx:661-674`), i.e. **at record time**. Nothing taps SCK before recording.

## Proposed design

### 1. Per-app system-audio capture

**Backend — enumerate audio-capable apps.** New command in a small module (or appended to
`recording.rs`):

```rust
#[derive(serde::Serialize)]
pub struct AudioApp { pub bundle_id: String, pub name: String, pub pid: i32 }

#[tauri::command]
pub fn list_audio_apps() -> Result<Vec<AudioApp>, String> {
    let content = SCShareableContent::get().map_err(|e| e.to_string())?;
    let mut apps: Vec<AudioApp> = content.applications().iter()
        .filter(|a| !a.application_name().is_empty())
        .map(|a| AudioApp { bundle_id: a.bundle_identifier(), name: a.application_name(), pid: a.process_id() })
        .collect();
    apps.sort_by(|a, b| a.name.to_lowercase().cmp(&b.name.to_lowercase()));
    apps.dedup_by(|a, b| a.bundle_id == b.bundle_id);   // one row per app, not per process
    Ok(apps)
}
```

SCK can't pre-filter to "apps that *produce* audio" — `applications()` is every running app — so we
list all and let the user pick (this is what Audio Hijack / Loopback do too). Matching is by
**bundle id** (stable across launches; pid is not).

**Backend — thread a chosen app through to the filter.** Add an optional field to the capture
config and the start command:

```rust
// system_audio.rs
pub struct SystemAudioCaptureConfig { pub sample_rate: u32, pub channels: u16, pub app_bundle_id: Option<String> }

// system_audio_macos.rs start(): after resolving `display`
let filter = match &self.config.app_bundle_id {
    Some(bid) => {
        let app = content.applications().into_iter()
            .find(|a| a.bundle_identifier() == *bid)
            .ok_or_else(|| format!("App {bid} not running / not shareable"))?;
        SCContentFilter::create().with_display(display)
            .with_including_applications(&[&app], &[]).build()   // include only this app
    }
    None => SCContentFilter::create().with_display(display).with_excluding_windows(&[]).build(),
};
```

`start_system_audio_stream` gains an optional `app_bundle_id: Option<String>` param, forwarded into
`SystemAudioCaptureConfig`. `None` ⇒ today's whole-system behavior (zero regression).

**Frontend — config + picker.** Add `systemAudioApp?: string` (bundle id; `undefined` = whole
system) to `ExternalRecordingConfig`. `native-system-audio.ts`'s `SystemAudioOptions` gains
`appBundleId?: string`, passed to `invoke("start_system_audio_stream", { ..., appBundleId })`.
`recording-canvas.tsx` reads `externalConfig.systemAudioApp` via a ref (mirror
`systemAudioGainRef`) and passes it into `startNativeSystemAudioStream` at line 666. In `toolbar.tsx`,
under the System Audio toggle, add a `Select` populated from `invoke<AudioApp[]>("list_audio_apps")`
(fetched lazily when the toggle is on), with a **"Entire system (all apps)"** default option →
`updateExternalConfig({ systemAudioApp: undefined })`.

### 2. Finish the faders (mute/solo + dB labels + record-time meters)

The gain plumbing is done (Current state). Three additive pieces:

- **Mute/solo.** Add `micMuted`/`systemAudioMuted` booleans to `ExternalRecordingConfig`. Mute is
  *not* a fourth gain value — it's an independent override so un-muting restores the slider value.
  In the prop-sync effect (`recording-canvas.tsx:288-291`) compute the effective target as
  `muted ? 0 : gain` and `setTargetAtTime` that; solo = mute the *other* source. Two small mute
  `Button`s (toggle) next to each slider in `toolbar.tsx`.
- **dB labels.** Replace the `%` label with dB next to each slider:
  `gain<=0 ? "−∞" : (20*Math.log10(gain)).toFixed(1)+" dB"` (a tiny pure helper, unit-tested).
  Keep the 0–3 linear slider; just relabel (engineers read gain in dB, % is ambiguous).
- **Per-source meters during record.** Today meters come from the *pre-record* monitor and vanish
  once recording starts. Tap `AnalyserNode`s **inside the recording graph** so the meters keep
  moving while recording: in `acquireAudioTrack`, after each `GainNode`, also `connect()` a leaf
  `AnalyserNode` (mic → `micRecAnalyserRef`, system → `systemRecAnalyserRef`), store them on refs,
  and surface them (via a callback prop or context) so `toolbar.tsx`/timeline can draw a
  `StereoMeter` per source during `isExternalRecording`. Analysers are passive leaves — no effect
  on the encoded mix.

### 3. System-audio pre-record monitor

The architectural wrinkle: the SCK stream currently only starts **inside** `acquireAudioTrack` at
record time (`recording-canvas.tsx:661-674`), and a `start_system_audio_stream` already running for
`section_index 0` would be **replaced** by the recording one (`system_audio_stream.rs:75-78` removes
and stops any existing stream for that index). So a monitor and a recording can't both own index 0.

**Design — a dedicated monitor stream on a reserved section index, torn down before record.**

- Add `useSystemAudioMonitor(enabled, appBundleId)` (new hook, sibling to `use-audio-monitor.ts`).
  It is **Tauri-only** (guard `isTauri()`); in a plain browser it stays idle (mic monitor already
  covers the browser case via `getUserMedia`). It opens its own `AudioContext`, calls
  `startNativeSystemAudioStream(MONITOR_SECTION = 99, ctx, analyserChain, { sampleRate, channels,
  appBundleId })`, and routes the worklet output through `AnalyserNode`s (reuse the L/R + mix
  analyser shape from `use-audio-monitor.ts:92-123`) and a **zero-gain** sink to `ctx.destination`
  to keep the graph rendering **without speaker playback** (the no-feedback rule from
  `[[native-system-audio-capture]]` Edge cases). Returns the same `AudioMonitorState` shape.
- Use a reserved monitor index (`99`) distinct from the recording index (`0`) so the two SCK streams
  never collide in `SystemAudioStreamState.streams` (`system_audio_stream.rs:53-55`).
- **Contention guard:** running two SCK `SCStream`s at once (monitor + record) is wasteful and may
  contend for SCK. Stop the monitor the instant recording starts: gate the monitor hook on
  `enabled = externalConfig.captureSystemAudio && isTauri() && !isExternalRecording`, so it tears
  down (its `useEffect` cleanup calls `stopNativeSystemAudioStream(99)`) right as the record path
  spins up its own stream on index 0.
- Wire it in `recording-context.tsx` next to `audioMonitor` as `systemAudioMonitor`; in
  `toolbar.tsx` draw a `StereoMeter` from it under the System Audio toggle (mirror the mic meter at
  246-252), shown when `externalConfig.captureSystemAudio`.

## Implementation steps

1. **`list_audio_apps` command.** Add `AudioApp` + `list_audio_apps` (enumerate
   `SCShareableContent::applications()`, dedup by bundle id, sort by name); register in
   `lib.rs:82-92`. **verify:** `cargo build` clean; `invoke("list_audio_apps")` from devtools
   returns a sorted, deduped list including a known running app.
2. **App field through capture.** Add `app_bundle_id: Option<String>` to `SystemAudioCaptureConfig`
   (`system_audio.rs`), update `Default` and `system_audio_fallback.rs`; branch the filter in
   `system_audio_macos.rs:start()` (include-app vs whole-display); add `app_bundle_id` param to
   `start_system_audio_stream`. **verify:** `cargo build`; `None` path records whole system as
   before; an app path logs the chosen bundle id at start.
3. **Frontend per-app config + picker.** Add `systemAudioApp?: string` to `ExternalRecordingConfig`
   + default `undefined`; `appBundleId` in `SystemAudioOptions` and the `invoke` call
   (`native-system-audio.ts`); a `systemAudioAppRef` in `recording-canvas.tsx` passed at line 666;
   the app `Select` (with "Entire system" default) in `toolbar.tsx`. **verify:** `npx tsc -b`;
   picker lists apps; selection persists in localStorage.
4. **dB label helper + mute/solo.** Add a pure `gainToDb(g)` helper + vitest (`gainToDb(1)=0`,
   `gainToDb(0)="−∞"`, `gainToDb(2)≈6.0`). Relabel both sliders. Add `micMuted`/`systemAudioMuted`
   to config + default `false`; effective-gain (`muted?0:gain`) in the prop-sync effect
   (`recording-canvas.tsx:288-291`); mute buttons in `toolbar.tsx`. **verify:** `npm run test`
   green; muting drops that source to silence and un-muting restores the prior slider value (no
   click).
5. **Record-time per-source meters.** In `acquireAudioTrack`, add leaf `AnalyserNode`s after the
   mic and system gain nodes, store on refs, surface via context; draw per-source `StereoMeter`s in
   the toolbar/timeline during `isExternalRecording`. **verify:** meters move for each present
   source while recording; muting a source flattens its meter; the saved MP4 is unchanged
   (analysers are passive).
6. **System-audio pre-record monitor.** Add `useSystemAudioMonitor` (Tauri-only, reserved index 99,
   analyser chain, zero-gain sink, gated off during `isExternalRecording`); wire `systemAudioMonitor`
   in `recording-context.tsx`; draw its meter in `toolbar.tsx`. **verify (hybrid):** with system
   audio enabled and music playing, the meter moves **before** recording; starting a recording stops
   the monitor stream (one SCK stream at a time — confirm backend logs one start on index 99 then a
   stop, then a start on index 0); no speaker feedback.
7. **Docs / index.** Update `[[next-implementations]]` (move these three from backlog → done);
   cross-link from `[[native-system-audio-capture]]` "Out of scope". **verify:** wiki-links resolve.

## Interface / API changes

- **Rust:** new `list_audio_apps() -> Result<Vec<AudioApp>, String>` (+ `AudioApp { bundle_id,
  name, pid }`), registered in `lib.rs`. `SystemAudioCaptureConfig` gains `app_bundle_id:
  Option<String>`. `start_system_audio_stream` gains `app_bundle_id: Option<String>`. `stop_*` /
  `ack_*` / wire format / `AudioChunk` **unchanged**.
- **TS:** `ExternalRecordingConfig` gains `systemAudioApp?: string`, `micMuted: boolean`,
  `systemAudioMuted: boolean` (+ defaults). `SystemAudioOptions` gains `appBundleId?: string`.
  New hook `useSystemAudioMonitor`. New pure `gainToDb`. `RecordingContextValue` gains
  `systemAudioMonitor` (+ optional record-time analyser refs).
- **`recording-canvas.tsx`:** `acquireAudioTrack` passes `appBundleId` and adds leaf analysers;
  prop-sync effect honors mute. **No change** to the encoder/muxer/save path.
- **No change** to the 16-byte wire header, `audio-wire.ts`, or `save_media_recording`.

## Edge cases & risks

- **Per-app *audio* via a video content filter (highest-uncertainty claim).**
  `with_including_applications` is documented/built as a *content* (window/video) filter
  (`content_filter.rs:248-271,494-509`). Whether SCK actually restricts the **audio** mix to the
  included apps is **not provable from the crate** — it depends on Apple's SCK audio routing.
  Apple's docs say the audio capture honors the filter's apps, but this **must** be verified at
  runtime (step 6 of the monitor and a dedicated per-app record test). If audio is *not* filtered,
  fall back to whole-system capture and surface the picker as "best effort," or move to per-process
  tap APIs (CoreAudio process taps, macOS 14.4+) as a future item — out of scope here.
- **macOS-version availability.** `with_including_applications` itself is not feature-gated and the
  underlying `SCContentFilter(display:including:exceptingWindows:)` predates our 14.0 floor, so the
  **API** is safe at the floor. The risk is purely the *audio-routing* behavior above, not symbol
  availability. (`set_content_rect`, `current_process`, `included_applications` ARE feature-gated to
  14.2/14.4/15.2 — we use none of them.)
- **App-list churn.** `applications()` is a point-in-time snapshot; an app the user picks may quit
  before/while recording. `start()` errors with "App … not running" — handle gracefully (toast +
  fall back to whole system, or block record). Offer a refresh; don't cache stale lists across a
  recording session.
- **Monitor-vs-record contention for the SCK stream.** Two `SCStream`s at once is the core wrinkle.
  Mitigated by (a) distinct section indices (99 monitor, 0 record) so they don't evict each other in
  `SystemAudioStreamState`, and (b) gating the monitor off during `isExternalRecording` so only one
  runs at a time. Still verify SCK tolerates the stop→start handoff without a glitch on the recorded
  track.
- **Feedback loops.** Both the monitor and the record path must connect the SCK source **only** to
  an analyser/mix node and a **zero-gain** sink — never audibly to `ctx.destination` — exactly as
  `[[native-system-audio-capture]]` requires. The monitor is the new risk: if its sink gain isn't
  zero, captured system audio replays out the speakers and SCK re-captures it → runaway howl.
- **TCC coupling.** `list_audio_apps` and any SCK stream need Screen Recording permission
  (`SCShareableContent::get()` errors without it, `mod.rs:157`). The monitor will surface the TCC
  prompt *before* recording — arguably a feature (fail fast), but message it clearly.
- **dB at the slider extremes.** `gain=0` ⇒ −∞ dB (guard the `log10`); `gain=3` ⇒ +9.5 dB (loud) —
  fine, but the meter (item 2/3) should show clipping so users don't blow out the mix.
- **Mute persistence.** `externalConfig` persists to localStorage (`recording-context.tsx:84-91`);
  a persisted `micMuted:true` could silently record silence next launch. Consider resetting mutes on
  load, or showing an obvious muted indicator.

## Testing & verification

- **Pure (vitest, auto):** `gainToDb` (0 → −∞, 1 → 0, 2 → +6.02, 3 → +9.54). Mirrors the
  pure-helper test pattern from `[[test-harness-and-ci]]`.
- **Build gates (auto):** `cargo build` (default features; `--features ffmpeg` is unrelated and
  doesn't link on this arm64 box per `[[native-system-audio-capture]]`), `npx tsc -b`,
  `npx vite build`.
- **Runtime (hybrid — needs interactive TCC + audible check, can't auto-verify):**
  - **App enumeration:** `list_audio_apps` from devtools returns a sane, sorted, deduped list.
  - **Per-app capture (the load-bearing test):** play audio in App A (e.g. Music) AND App B (e.g. a
    browser video); pick App A; record 20-30s; `ffprobe` shows one AAC track; **listen** — only App
    A's audio is present, App B is absent. This is the only proof the audio filter works.
  - **Faders:** drag mic & system gain mid-record (no clicks); mute/un-mute restores level;
    per-source meters move while recording and flatten on mute.
  - **Monitor:** before recording, with system audio on and music playing, the system meter moves;
    starting record stops the monitor stream (logs: start idx 99 → stop idx 99 → start idx 0); no
    speaker feedback at any point.
- **Cannot auto-verify:** the per-app *audio* routing behavior, monitor/record handoff glitch-free,
  feedback-loop absence, and audible mute/solo — all need the user's interactive pass.

## Out of scope / future

- **Per-*process* (not per-app) audio taps** via CoreAudio process taps (macOS 14.4+) — finer than
  SCK's per-app, and the fallback if SCK's app filter doesn't restrict audio (see Edge cases).
- **Multi-app capture** (include N apps, or exclude-list "everything except Slack/Mail"): the builder
  already supports `&[&app]` slices and `with_excluding_applications`, but the UI/config here is
  single-app for the minimal cut.
- **Persisted-mute UX hardening**, solo for >2 sources, and a full mixer panel.
- **Resampling the monitor** if `AudioContext` clamps the rate (the monitor only meters, so exact
  pitch doesn't matter; the *recording* path's rate coherence is already handled per
  `[[native-system-audio-capture]]`).
- **Moving the SCK stream to an `AudioWorklet`-only path** — tracked separately in
  `[[audioworklet-migration]]`; this plan reuses the existing worklet/ScriptProcessor source.
- **Encoder-side gain/limiting** (vs Web Audio `GainNode`) — see `[[encoder-tuning]]`.
- **Per-section transport teardown** when a section is unregistered — `[[per-section-transport-unregister]]`.

## References

- `src-tauri/src/system_audio_macos.rs` — `start()` filter build (67-120, `with_display` +
  `with_excluding_windows` at 84-87, no app filter), `SCShareableContent::get()` (79),
  `extract_audio_samples` (140-181), `stop()` (122-132), `is_system_audio_available` (136-138).
- `src-tauri/src/system_audio.rs` — `AudioChunk` (2-8), `SystemAudioCaptureConfig` (11-25).
- `src-tauri/src/system_audio_stream.rs` — `start_system_audio_stream` (67-176, evict-existing at
  75-78), wire header (59-66), `stop_*`/`ack_*` (179-208), `SystemAudioStreamState` (53-55).
- `src-tauri/src/recording.rs` — `get_available_devices` / `has_system_audio` (21,26,45).
- `src-tauri/src/lib.rs` — `generate_handler!` (82-92), `.manage(...)` (80-81).
- `~/.cargo/.../screencapturekit-1.5.0/src/stream/content_filter.rs` —
  `with_including_applications` (494-509), `with_excluding_applications` (517-532),
  `DisplayIncludingApplications` build (664-688); feature-gated bits we avoid (set_content_rect 122,
  included_applications 251).
- `~/.cargo/.../screencapturekit-1.5.0/src/shareable_content/mod.rs` — `applications()` (265-281),
  `get()` perm note (157); `running_application.rs` — `application_name`/`bundle_identifier`/`process_id`
  (61-81).
- `frontend/src/components/asmr-recorder/recording-canvas.tsx` — gain nodes
  (`micGainNodeRef` 644-649, `systemAudioGainNodeRef` 652-659/661-665), native stream start
  (661-674), prop-sync `setTargetAtTime` (288-291), `acquireAudioTrack` (542-684), audio constants
  (90-93).
- `frontend/src/lib/native-system-audio.ts` — `startNativeSystemAudioStream`/`stop*` (198-266),
  `SystemAudioOptions` (66-69), the inline worklet source (30-186).
- `frontend/src/lib/audio-wire.ts` — 16-byte header parse (12-35).
- `frontend/src/hooks/use-audio-monitor.ts` — mic-only monitor, `enabled`/`getUserMedia` gating
  (38-49), analyser chain + zero-gain sink (92-131), returned shape (18-28).
- `frontend/src/contexts/recording-context.tsx` — `audioMonitor` wiring (77-80), `externalConfig`
  localStorage persistence (60-91), `updateExternalConfig` (163-174).
- `frontend/src/components/asmr-recorder/toolbar.tsx` — mic meter (246-252), Mic Gain slider
  (287-303), System Audio toggle gated on `hasSystemAudio` (319-343), System Audio Gain slider
  (345-361).
- `frontend/src/types/recording.ts` — `ExternalRecordingConfig` (`micGain`/`systemAudioGain`/
  `captureSystemAudio` 80,84,86) + defaults (107-121), `DeviceList.hasSystemAudio` (55).
- `src-tauri/Cargo.toml:44` (screencapturekit feature `macos_14_0`), `src-tauri/tauri.conf.json:41`
  (`minimumSystemVersion "14.0"`).
- Extends: `[[native-system-audio-capture]]` (shipped base + deferred items, "Out of scope" 262-264).
  Related: `[[audioworklet-migration]]`, `[[encoder-tuning]]`,
  `[[per-section-transport-unregister]]`, `[[next-implementations]]`,
  `[[webkit-getusermedia-works-getdisplaymedia-blocked]]`, `[[test-harness-and-ci]]`.
