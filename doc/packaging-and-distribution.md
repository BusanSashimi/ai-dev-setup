# Feature: Packaging & distribution — signing, notarization, macOS version floor

**Priority:** P1  ·  **Effort:** M  ·  **Risk:** med  ·  **Surface:** build/config

## Problem / motivation

The app builds a `.app`/`.dmg` (`bundle.active: true`, `targets: "all"`) but is **not
shippable**: an unsigned, un-notarized bundle is blocked by Gatekeeper on any other Mac
("app is damaged / cannot be opened"), and one config value (`minimumSystemVersion`) is
inconsistent with what the code actually requires.

This is the natural follow-on to the camera/mic Info.plist work (PR #2) — that added the
TCC **usage strings**; this adds the **signing + notarization + version** plumbing needed
for the strings (and the whole app) to function off the developer's machine.

Two correctness bugs and the signing gap:

1. **`minimumSystemVersion` is wrong.** `tauri.conf.json` declares `"minimumSystemVersion":
   "10.13"` (High Sierra, 2017), but the macOS dependency is
   `screencapturekit = { version = "1", features = ["async", "macos_14_0"] }`
   (`Cargo.toml:58`) and the native screen/audio paths call ScreenCaptureKit. On 10.13–13.x
   the app would launch and then **fail at capture** (or fail to load symbols). The floor
   must match the SCK requirement.
2. **No code signing / notarization.** `tauri.conf.json` `bundle.macOS` has `entitlements`,
   `frameworks`, `minimumSystemVersion` — but **no `signingIdentity`** and **no notarization**.
   Without a Developer ID signature + notarization, Gatekeeper blocks the app, and —
   subtly — **TCC grants don't stick**: an ad-hoc/unsigned binary changes identity on each
   rebuild, so macOS re-prompts for Screen Recording / camera / mic every launch (or
   silently denies). Stable signing is what makes the screen-recording grant persist.
3. **No app category** (`bundle.category`) — minor, but required for a polished
   Finder/Launchpad presentation and App Store-style metadata.

What's **already correct** (don't redo): `entitlements.plist` exists and **is wired**
(`bundle.macOS.entitlements: "entitlements.plist"`, `tauri.conf.json:38`) with
`com.apple.security.device.camera` + `com.apple.security.device.audio-input`; `Info.plist`
has `NSCameraUsageDescription` + `NSMicrophoneUsageDescription`.

> **Accuracy note (do not "fix" these):** macOS has **no** screen-recording *entitlement*
> and **no** `NSScreenRecordingUsageDescription` Info.plist key that grants capture — screen
> recording is a pure **TCC** permission with a system-fixed prompt, triggered on first
> `SCStream` start and managed in System Settings → Privacy & Security → Screen Recording.
> The only thing the developer controls is whether the app is **stably signed** so the grant
> persists. Do not add a fake screen-capture entitlement or usage string.

## Current state (grounded)

**`src-tauri/tauri.conf.json`:**
```json
"identifier": "com.asmr-recorder.app",
"bundle": {
  "active": true,
  "targets": "all",
  "icon": ["icons/32x32.png", "icons/128x128.png", "icons/128x128@2x.png", "icons/icon.icns", "icons/icon.ico"],
  "macOS": {
    "entitlements": "entitlements.plist",
    "frameworks": [],
    "minimumSystemVersion": "10.13"     // ← inconsistent with SCK macos_14_0
  }
}
```
- `app.security.csp: null` (line 24) — fine for a local app, but worth setting a real CSP
  before distribution (defense-in-depth; the webview loads only bundled assets).
- No `bundle.macOS.signingIdentity`, no notarization keys, no `bundle.category`.

**`src-tauri/entitlements.plist`:** `com.apple.security.device.camera` +
`com.apple.security.device.audio-input` (both `<true/>`). No App Sandbox
(`com.apple.security.app-sandbox`) — correct for a Developer ID (non-MAS) distribution.

**`src-tauri/Info.plist`:** `NSCameraUsageDescription`, `NSMicrophoneUsageDescription`.

**`Cargo.toml`:** `default = []`; `ffmpeg = ["ffmpeg-next"]` (optional, **off by default**).
`screencapturekit` features `["async", "macos_14_0"]` (line 58).

**`.cargo/config.toml`:** `rustflags = ["-L", "/opt/homebrew/lib"]`,
`PKG_CONFIG_PATH = /opt/homebrew/lib/pkgconfig`, `DYLD_LIBRARY_PATH = /usr/lib/swift`.
**`src-tauri/build.rs`:** `cargo:rustc-link-arg=-Wl,-rpath,/usr/lib/swift` (Swift runtime
for the SCK bindings).

**Dev log:** `--features ffmpeg` fails to link on this arm64 machine; the default build
(no ffmpeg) is the one that works and is the one to ship.

## Proposed design

Configure a **Developer ID** (non-App-Store) signed + notarized build, fix the version
floor, and document the prerequisites. No code changes — config + a build script + docs.

**(1) Version floor.** Set `bundle.macOS.minimumSystemVersion` to `"14.0"` to match the
`macos_14_0` SCK feature. (If a lower floor is ever wanted, SCK is available from 12.3 and
the crate feature would need to drop to `macos_12_3` — out of scope; 14.0 matches today's
build.)

**(2) Signing.** Sign with a **Developer ID Application** certificate. Tauri reads the
identity from `bundle.macOS.signingIdentity` or the `APPLE_SIGNING_IDENTITY` env var and
**automatically enables the hardened runtime** for Developer ID builds (required for
notarization). The existing camera/mic entitlements are applied via the already-wired
`entitlements.plist`.

**(3) Notarization.** Tauri runs notarization during `tauri build` when the Apple
credentials are present in the environment — either:
- API key: `APPLE_API_KEY`, `APPLE_API_ISSUER`, `APPLE_API_KEY_PATH`, **or**
- Apple ID: `APPLE_ID`, `APPLE_PASSWORD` (app-specific password), `APPLE_TEAM_ID`.

Tauri staples the ticket to the `.app`/`.dmg`. Keep all secrets in the environment / a
local `.env` that is gitignored — **never** in `tauri.conf.json`.

**(4) Hardened-runtime sanity for the Swift/SCK linkage.** The default build links only
**system** Swift libs via the `/usr/lib/swift` rpath (no Homebrew dylibs — `ffmpeg` is off),
so the hardened runtime should not need `com.apple.security.cs.disable-library-validation`.
**Verify** this against the first notarized build; only add the disable-library-validation
entitlement if `codesign --verify`/notarization flags the Swift load. (The `ffmpeg` feature
*would* pull Homebrew dylibs and break a clean notarized build — it must stay **off** for
distribution; this ties into `[[legacy-native-pipeline-removal]]`, which drops the feature
entirely.)

**(5) Category + CSP polish.** Add `bundle.category: "public.app-category.utilities"` (or
`...productivity`). Optionally set a restrictive `app.security.csp` (the app loads only
bundled assets + the Tauri IPC) before public release.

## Implementation steps

1. **Fix the version floor.** `tauri.conf.json`: `minimumSystemVersion` `"10.13"` → `"14.0"`.
   **verify:** `npm run tauri build` (default features) produces a bundle; `plutil -p
   "$(...)/Contents/Info.plist" | grep LSMinimumSystemVersion` shows `14.0`.

2. **Add category (+ optional CSP).** Add `"category": "public.app-category.utilities"` to
   `bundle`. (Optional) replace `app.security.csp: null` with a real policy and confirm the
   app still loads. **verify:** build succeeds; app launches; `plutil -p` shows
   `LSApplicationCategoryType`.

3. **Signing config.** Add `bundle.macOS.signingIdentity` (or document setting
   `APPLE_SIGNING_IDENTITY`). Document the prerequisite: a Developer ID Application cert in
   the login keychain (`security find-identity -v -p codesigning` lists it). **verify (user,
   needs an Apple Developer account):** `tauri build` emits "signing with identity …";
   `codesign --verify --deep --strict --verbose=2 "<App>.app"` exits 0;
   `codesign -d --entitlements - "<App>.app"` shows the camera/mic entitlements + hardened
   runtime flags.

4. **Notarization config + docs.** Document the required env vars (API-key or Apple-ID set)
   and add a gitignored `.env.example`. **verify (user):** `tauri build` runs notarization
   and staples; `xcrun stapler validate "<App>.app"` and `spctl -a -vvv -t install
   "<App>.dmg"` report "accepted / Notarized Developer ID".

5. **Clean-machine smoke test (user).** Copy the `.dmg` to a second Mac (or a fresh user
   account) running macOS 14+, install, launch: app opens without the "damaged" dialog,
   the camera/mic/screen-recording TCC prompts appear **once** and persist across relaunch
   (validates stable signing). **verify:** record a short clip end-to-end on the clean
   machine; `save_media_recording` writes to `~/Movies`/Videos (release path).

6. **Document in `ai-dev/README.md`.** Add a "Release build (signed + notarized)" section:
   prerequisites (Apple Developer account, Developer ID cert, env vars), the exact
   `tauri build` invocation (default features only — **never** `--features ffmpeg`), and the
   verification commands above. **verify:** another developer can follow it cold.

## Interface / API changes

- **`tauri.conf.json`:** `bundle.macOS.minimumSystemVersion` → `"14.0"`; add
  `bundle.macOS.signingIdentity`; add `bundle.category`; (optional) `app.security.csp`.
- **Environment (not committed):** Apple notarization credentials (API key or Apple-ID set)
  + `APPLE_SIGNING_IDENTITY`.
- **No source/Rust/TS changes.** Entitlements + Info.plist already correct.

## Edge cases & risks

- **Requires an Apple Developer account** ($99/yr) + a Developer ID Application cert. Steps
  3-5 cannot be completed (or auto-verified) without it — they are user/CI steps. Steps 1-2
  (version floor, category) are doable and verifiable today with no account.
- **`ffmpeg` feature breaks notarization.** It links Homebrew dylibs from `/opt/homebrew/lib`
  that are neither signed nor present on a clean machine → notarization/Gatekeeper failure.
  The distributable build **must** be default-features. Guard the release docs against
  `--features ffmpeg`. (Removing the feature entirely — `[[legacy-native-pipeline-removal]]`
  — eliminates the footgun.)
- **Hardened runtime + Swift.** If the first notarized build flags the `/usr/lib/swift` load,
  add `com.apple.security.cs.disable-library-validation` to `entitlements.plist` — but
  **only if needed**; system Swift libs normally pass. Don't add it speculatively.
- **`minimumSystemVersion` bump drops <14 users.** Intentional and correct given SCK — the
  app cannot function on <14 anyway. Note it in the README/release notes.
- **Secrets hygiene.** Notarization credentials must never land in `tauri.conf.json` or git.
  Use env vars + gitignored `.env`; in CI use encrypted secrets (see
  `[[test-harness-and-ci]]` for the macOS runner).
- **`targets: "all"`** also emits non-macOS bundle types that are meaningless on this
  macOS-first app; consider narrowing to `["app", "dmg"]` (optional, cosmetic).

## Testing & verification

- **Auto (no account):** `npm run tauri build` (default features) succeeds; `plutil -p` on
  the bundled `Info.plist` confirms `LSMinimumSystemVersion = 14.0` and the category.
- **User (needs Apple Developer account):** `codesign --verify --deep --strict`,
  `codesign -d --entitlements -`, `xcrun stapler validate`, `spctl -a -t install` on the
  signed+notarized output; clean-machine install + record smoke test.
- **Cannot auto-verify:** anything requiring an Apple account, a second Mac, or interactive
  Gatekeeper/TCC behavior — these are inherently manual/CI.

## Out of scope / future

- **CI release pipeline** (GitHub Actions macOS runner doing signed+notarized builds with
  encrypted secrets) — `[[test-harness-and-ci]]`.
- **Auto-update** (`tauri-plugin-updater` + an update server / GitHub releases manifest).
- **Mac App Store** distribution (needs App Sandbox + different provisioning) — Developer ID
  is the right target for a screen recorder.
- **DMG styling** (background image, icon layout) and a custom installer.
- **Windows/Linux** packaging (this app is macOS-first via SCK).

## References

- `src-tauri/tauri.conf.json` — `identifier` (5), `app.security.csp` (24), `bundle` (27-42),
  `bundle.macOS.entitlements` (38, already wired), `minimumSystemVersion` (40).
- `src-tauri/entitlements.plist` — camera + audio-input entitlements (wired via tauri.conf).
- `src-tauri/Info.plist` — `NSCameraUsageDescription`, `NSMicrophoneUsageDescription`.
- `src-tauri/Cargo.toml` — `default = []`, `ffmpeg = ["ffmpeg-next"]` (33, 55), `screencapturekit`
  `macos_14_0` (58).
- `src-tauri/.cargo/config.toml`, `src-tauri/build.rs` — Swift `/usr/lib/swift` rpath; Homebrew
  paths used only when `ffmpeg` is enabled.
- Tauri v2 macOS signing/notarization: `bundle.macOS.signingIdentity`, hardened runtime
  auto-enabled for Developer ID, notarization via `APPLE_API_KEY`/`APPLE_ID` env vars,
  ticket stapling.
- Dev log: `docs/logs/2026-06-13.md` / `2026-06-14.md` — camera/mic Info.plist work (PR #2),
  `--features ffmpeg` arm64 link failure.
- Related: `[[legacy-native-pipeline-removal]]` (drops the `ffmpeg` footgun),
  `[[test-harness-and-ci]]` (release CI).
