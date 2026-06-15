# Feature: In-app auto-update via `tauri-plugin-updater` + a GitHub-releases manifest

**Priority:** P3  ·  **Effort:** M  ·  **Risk:** med  ·  **Surface:** build + backend + a small frontend affordance

## Problem / motivation

Once the app ships as a signed + notarized `.dmg` ([[packaging-and-distribution]]), there is **no
update path**: a user who installed `0.1.0` stays on `0.1.0` forever unless they manually
re-download. For a portfolio desktop app distributed outside the Mac App Store (Developer ID,
not MAS), the standard answer is Tauri v2's **updater plugin** — the app checks a hosted
**update manifest** on launch (or on demand), downloads the new bundle, verifies it against a
**minisign public key baked into the build**, installs, and relaunches.

The subtlety that makes this a *pairing* with packaging, not a standalone feature: the updater's
signature is a **second, independent signature** from macOS codesigning. There are **two
keypairs**:

1. **Apple Developer ID** (codesign + notarization) — makes Gatekeeper accept the `.app`/`.dmg`
   and makes TCC grants persist (already specified in [[packaging-and-distribution]]).
2. **minisign / Tauri updater key** (`tauri signer generate`) — makes the *updater client* trust
   that a downloaded artifact came from us. The public half goes in `tauri.conf.json`; the
   private half + its password are CI secrets.

A release build must therefore produce **both**: a signed/notarized `.app` + `.dmg` for first
install, **and** a minisign-signed updater artifact (`.app.tar.gz` + `.app.tar.gz.sig`) plus a
`latest.json` manifest for subsequent updates.

## Current state (grounded)

- **`src-tauri/tauri.conf.json`** (44 lines total): `version: "0.1.0"` (L4),
  `identifier: "com.asmr-recorder.app"` (L5), `productName: "asmr-recorder"` (L3). The
  packaging work already set `bundle.targets: ["app", "dmg"]` (L29),
  `bundle.category: "public.app-category.utilities"` (L30), and
  `bundle.macOS.minimumSystemVersion: "14.0"` (L41). **There is no top-level `plugins` key
  and no `bundle.createUpdaterArtifacts` key today** — both must be added.
- **`src-tauri/Cargo.toml`**: `tauri = { version = "2", … }` (L21), `tauri-plugin-opener = "2"`
  (L22). No `tauri-plugin-updater` and **no `tauri-plugin-process`** (needed for relaunch). The
  matching plugin versions are the `2` major line.
- **`src-tauri/src/lib.rs`**: plugins register in `run()` via the builder chain —
  `tauri::Builder::default().plugin(tauri_plugin_opener::init())` (L78-79), then `.manage(…)`
  (L80-81), then `.invoke_handler(tauri::generate_handler![…])` (L82-92), then
  `.run(tauri::generate_context!())` (L93). **This is where the updater + process plugins get
  added.** Note: there is currently **no `.setup(|app| …)` closure** — the desktop-only
  registration form (below) introduces one.
- **`src-tauri/capabilities/default.json`**: `permissions: ["core:default", "opener:default"]`
  for `windows: ["main"]`. The updater/process plugin permissions must be added here.
- **`.github/workflows/ci.yml`**: build-gate workflow on `macos-latest` — `actions/checkout@v4`,
  `actions/setup-node@v4` (node 20, npm cache over `package-lock.json` + `frontend/package-lock.json`),
  `dtolnay/rust-toolchain@stable`, `actions/cache@v4` over `~/.cargo/registry|git` +
  `src-tauri/target`, then `npm ci` (root + `--prefix frontend`), `cargo build`/`cargo test`
  (both with `DYLD_LIBRARY_PATH: /usr/lib/swift`), `tsc -b`, `vite build`, `npm run test`. The
  release workflow mirrors this setup (node/rust/caches/DYLD) but swaps the build gates for
  `tauri-apps/tauri-action`. CI is **build-only, no signing** by design ([[test-harness-and-ci]]).
- **`frontend/package.json`**: `@tauri-apps/api: "^2.9.1"` (L24). No `@tauri-apps/plugin-updater`
  and no `@tauri-apps/plugin-process`. Frontend tests run on `vitest` (`"test": "vitest run"`).
- **Update-check UI location**: `frontend/src/components/asmr-recorder/toolbar.tsx` already
  imports shadcn `Button` (L2), `Dialog`/`DialogContent`/… (L13-20), `toast` from
  `@/hooks/use-toast` (L37), and lucide icons including `RotateCcw`/`Download` (L23,35) — i.e.
  the exact primitives an update affordance needs. A small `useUpdater` module + one toolbar
  button/menu item is the minimal cut.

## Proposed design

Add the updater plugin (Rust + JS) + the process plugin (relaunch), generate an updater
keypair, configure the `updater` block + `createUpdaterArtifacts`, host `latest.json` + the
signed artifact on GitHub releases, add a tag-triggered release workflow that builds + Apple-signs
+ notarizes + minisign-signs + uploads, and wire a small in-app check/prompt/install/relaunch UX.

### 1. Generate the updater keypair (one-time, local)

```bash
npm run tauri signer generate -- -w ~/.tauri/asmr-recorder-updater.key
```

This emits a **password-protected private key** (`~/.tauri/asmr-recorder-updater.key`) and a
**public key** (`.key.pub`, base64 minisign). The public key string goes in
`tauri.conf.json`; the private key **file contents** + the password become CI secrets
(`TAURI_SIGNING_PRIVATE_KEY`, `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`). **Never commit the private
key.** This keypair is wholly separate from the Apple Developer ID cert in
[[packaging-and-distribution]].

### 2. `tauri.conf.json` — `updater` plugin block + `createUpdaterArtifacts`

Add a top-level `plugins.updater` block and the bundle flag (the bundle flag is what makes
`tauri build` emit the `.app.tar.gz` + `.sig` updater artifacts):

```jsonc
"bundle": {
  "active": true,
  "targets": ["app", "dmg"],
  "createUpdaterArtifacts": true,        // ← NEW: emit .app.tar.gz + .app.tar.gz.sig
  "category": "public.app-category.utilities",
  …
},
"plugins": {
  "updater": {
    "pubkey": "dW50cnVzdGVkIGNvbW1lbnQ6 …(minisign public key from step 1).key.pub…",
    "endpoints": [
      "https://github.com/<owner>/asmr-recorder/releases/latest/download/latest.json"
    ]
    // no "windows" install-mode block — macOS-only app
  }
}
```

- `endpoints` supports the template vars `{{target}}`, `{{arch}}`, `{{current_version}}`; the
  GitHub `releases/latest/download/latest.json` form is the simplest (one static URL, always
  the newest release). The plugin GETs the manifest, compares `version` against the running
  `version`, and only offers an update if newer.
- `createUpdaterArtifacts: true` is the modern value. (An older `"v1Compatible"` value exists
  only for apps migrating a v1 updater install base — **not us**, this is a fresh updater.)

### 3. `latest.json` manifest shape (hosted on the GitHub release)

```json
{
  "version": "0.2.0",
  "notes": "Bug fixes and the new composition layouts.",
  "pub_date": "2026-07-01T00:00:00Z",
  "platforms": {
    "darwin-aarch64": {
      "signature": "<contents of asmr-recorder.app.tar.gz.sig>",
      "url": "https://github.com/<owner>/asmr-recorder/releases/download/v0.2.0/asmr-recorder_0.2.0_aarch64.app.tar.gz"
    }
  }
}
```

- Platform keys are `<os>-<arch>`: `darwin-aarch64` (Apple Silicon), `darwin-x86_64` (Intel).
  This app's only target floor is macOS 14 / SCK; **ship `darwin-aarch64`** (the dev machine is
  arm64) and optionally add `darwin-x86_64` if a universal/Intel build is wanted (see Out of
  scope). The `signature` is the **literal text** of the `.sig` file, not a path.
- `version` must be a clean semver and **strictly greater** than the installed version for the
  update to fire (see Edge cases — monotonicity).

### 4. Rust registration — `src-tauri/src/lib.rs`

Add the updater + process plugins. The updater plugin is **desktop-only** and the docs use a
`.setup()` closure (lib.rs has none today, so this introduces one). Register process via the
normal `.plugin()` chain:

```rust
tauri::Builder::default()
    .plugin(tauri_plugin_opener::init())
    .plugin(tauri_plugin_process::init())          // ← relaunch()
    .setup(|app| {                                  // ← NEW closure
        #[cfg(desktop)]
        app.handle().plugin(tauri_plugin_updater::Builder::new().build())?;
        Ok(())
    })
    .manage(screen_stream_state)
    …
```

`Cargo.toml`: add `tauri-plugin-updater = "2"` and `tauri-plugin-process = "2"` under
`[dependencies]` (alongside `tauri-plugin-opener = "2"`, L22).

### 5. Capabilities — `src-tauri/capabilities/default.json`

The plugin commands are denied unless allowed. Add:

```jsonc
"permissions": [
  "core:default",
  "opener:default",
  "updater:default",          // check + download + install
  "process:allow-restart"     // relaunch()
]
```

(`updater:default` bundles `allow-check`/`allow-download`/`allow-install`/`allow-download-and-install`;
if a tighter set is preferred, list those individually.)

### 6. Frontend JS plugin + update-check UX

`frontend/package.json`: add `@tauri-apps/plugin-updater` and `@tauri-apps/plugin-process`
(both `^2`), matching `@tauri-apps/api ^2.9.1` (L24).

New tiny module `frontend/src/lib/updater.ts`:

```ts
import { check } from "@tauri-apps/plugin-updater";
import { relaunch } from "@tauri-apps/plugin-process";

export async function checkForUpdate() {
  const update = await check();        // null if up-to-date or no endpoint reachable
  return update;                        // { version, body, downloadAndInstall, … } | null
}

export async function applyUpdate(update, onProgress?: (pct: number) => void) {
  let total = 0, got = 0;
  await update.downloadAndInstall((e) => {
    if (e.event === "Started") total = e.data.contentLength ?? 0;
    else if (e.event === "Progress") { got += e.data.chunkLength; onProgress?.(total ? got / total : 0); }
  });
  await relaunch();                     // restart into the new version
}
```

Wire it in `toolbar.tsx` (reuse the already-imported `Button`, `Dialog`, `toast`, `Download`
icon):

- **On launch**: call `checkForUpdate()` once from a mount effect (e.g. in `asmr-recorder.tsx`
  or the toolbar). If non-null, surface a non-blocking `toast` / small dialog: "Update to
  v{update.version} available" with an **Update** button.
- **Manual check**: a "Check for updates" menu item (or a button in the settings `Dialog`,
  L149+ region) calling `checkForUpdate()`; toast "You're up to date" when null.
- **Apply**: the Update button calls `applyUpdate(update, setPct)`, showing a progress bar
  (reuse the `Progress`/`Slider` patterns), then `relaunch()`.
- **Never auto-install** without consent — always prompt. Don't block recording: if a recording
  is in progress, defer the prompt (the app shouldn't relaunch mid-record).

### 7. Release workflow — `.github/workflows/release.yml` (tag-triggered)

Mirror `ci.yml`'s setup (node 20 + npm cache, `dtolnay/rust-toolchain@stable`, cargo cache,
`DYLD_LIBRARY_PATH`), but build + sign + upload via `tauri-apps/tauri-action`. Trigger on `v*`
tags.

```yaml
name: Release
on:
  push:
    tags: ["v*"]
jobs:
  release:
    runs-on: macos-latest
    permissions:
      contents: write          # create the GitHub release + upload assets
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm",
                cache-dependency-path: "package-lock.json\nfrontend/package-lock.json" }
      - uses: dtolnay/rust-toolchain@stable
        with: { targets: aarch64-apple-darwin }
      - uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            src-tauri/target
          key: ${{ runner.os }}-cargo-${{ hashFiles('src-tauri/Cargo.lock') }}
          restore-keys: ${{ runner.os }}-cargo-
      - run: npm ci
      - run: npm ci --prefix frontend
      - uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DYLD_LIBRARY_PATH: /usr/lib/swift
          # ── Updater (minisign) signing — the SECOND keypair ──
          TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
          TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY_PASSWORD }}
          # ── Apple Developer ID codesign + notarization (from packaging plan) ──
          APPLE_CERTIFICATE: ${{ secrets.APPLE_CERTIFICATE }}              # base64 .p12
          APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
          APPLE_SIGNING_IDENTITY: ${{ secrets.APPLE_SIGNING_IDENTITY }}
          APPLE_ID: ${{ secrets.APPLE_ID }}
          APPLE_PASSWORD: ${{ secrets.APPLE_PASSWORD }}                    # app-specific pw
          APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
        with:
          tagName: ${{ github.ref_name }}        # e.g. v0.2.0
          releaseName: "ASMR Recorder ${{ github.ref_name }}"
          args: --target aarch64-apple-darwin
          # uploadUpdaterJson defaults to true → action builds & uploads latest.json
```

`tauri-action` runs `tauri build` (which, with `createUpdaterArtifacts: true`, Apple-signs the
`.app`/`.dmg`, notarizes via the Apple env vars, and minisign-signs the `.app.tar.gz` via the
`TAURI_SIGNING_*` env vars), creates the GitHub release for the tag, uploads the `.dmg` +
updater artifacts, and — because `uploadUpdaterJson` defaults to `true` — generates and uploads
`latest.json` pointing at the uploaded artifact + its signature. The `endpoints` URL in step 2
(`releases/latest/download/latest.json`) then resolves to it.

## Implementation steps

1. **Generate the updater keypair.** Run `tauri signer generate` (step 1). Save the public key;
   store the private key + password as repo secrets `TAURI_SIGNING_PRIVATE_KEY` (file contents)
   and `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`. **verify:** `~/.tauri/asmr-recorder-updater.key.pub`
   exists and is a `untrusted comment: minisign public key …` block; the secrets are set in
   `gh secret list`.
2. **Add deps.** `Cargo.toml`: `tauri-plugin-updater = "2"`, `tauri-plugin-process = "2"`.
   `frontend/package.json`: `@tauri-apps/plugin-updater@^2`, `@tauri-apps/plugin-process@^2`
   (then `npm install --prefix frontend`). **verify:** `cargo build` (with
   `DYLD_LIBRARY_PATH=/usr/lib/swift`) and `npx tsc -b` both succeed.
3. **Config block.** `tauri.conf.json`: add `bundle.createUpdaterArtifacts: true` and the
   `plugins.updater` block (pubkey + endpoints). **verify:** `tauri.conf.json` validates against
   its `$schema` (no editor/CLI schema error); `npm run tauri build` now emits
   `asmr-recorder.app.tar.gz` + `.sig` in `src-tauri/target/release/bundle/macos/`.
4. **Register plugins (Rust).** `lib.rs`: add `.plugin(tauri_plugin_process::init())` and the
   `.setup(|app| { #[cfg(desktop)] app.handle().plugin(tauri_plugin_updater::Builder::new().build())?; Ok(()) })`
   closure. **verify:** `cargo build` succeeds; app launches in `npm run tauri dev`.
5. **Capabilities.** `capabilities/default.json`: add `"updater:default"` + `"process:allow-restart"`.
   **verify:** calling `check()` from the frontend doesn't throw a "not allowed / permission"
   IPC error.
6. **Frontend module + UX.** Add `frontend/src/lib/updater.ts` (`checkForUpdate`, `applyUpdate`);
   add a launch-time check + a "Check for updates" affordance + an update prompt in
   `toolbar.tsx` (reusing `Dialog`/`toast`/`Download`). Defer the prompt while
   `isExternalRecording`. **verify:** `npx tsc -b` + `npm run test` clean; in dev, "Check for
   updates" shows "up to date" (no newer release) without crashing.
7. **Release workflow.** Add `.github/workflows/release.yml` (step 7 YAML). **verify:** push a
   throwaway tag (e.g. `v0.1.1-test`) on a branch; the workflow runs `tauri-action`, produces a
   notarized `.dmg` + `.app.tar.gz` + `.sig` + `latest.json`, and creates the GitHub release.
   (Requires the Apple + updater secrets; otherwise it fails at the signing step — expected.)
8. **End-to-end update test.** Install `v0.1.1-test` locally; bump `version` to a higher value,
   tag + release `v0.1.2-test`; launch the installed `0.1.1` build and confirm it detects,
   downloads, verifies, installs, and relaunches into `0.1.2`. **verify:** the relaunched app
   reports the new version (e.g. a version label / `getVersion()`); macOS doesn't re-prompt
   Gatekeeper (artifact is notarized + minisign-verified).
9. **Docs + index.** Note the release/update flow in `ai-dev/README.md`; move **Auto-update**
   from the [[next-implementations]] backlog into the planned table. **verify:** links resolve.

## Interface / API changes

- **`src-tauri/Cargo.toml`:** + `tauri-plugin-updater = "2"`, `tauri-plugin-process = "2"`.
- **`src-tauri/tauri.conf.json`:** + `bundle.createUpdaterArtifacts: true`; + `plugins.updater`
  (`pubkey`, `endpoints`). `version` (L4) becomes the **update gate** — bump it per release.
- **`src-tauri/src/lib.rs`:** + `.plugin(tauri_plugin_process::init())` and a `.setup()` closure
  registering the updater plugin (desktop-only). No command-handler change.
- **`src-tauri/capabilities/default.json`:** + `"updater:default"`, `"process:allow-restart"`.
- **`frontend/package.json`:** + `@tauri-apps/plugin-updater`, `@tauri-apps/plugin-process`.
- **New:** `frontend/src/lib/updater.ts`; an update affordance in `toolbar.tsx`;
  `.github/workflows/release.yml`; the hosted `latest.json` (generated by `tauri-action`).
- **New secrets (not committed):** `TAURI_SIGNING_PRIVATE_KEY`, `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`,
  plus the Apple set already required by [[packaging-and-distribution]].

## Edge cases & risks

- **Two keys, two purposes — don't conflate.** The minisign `pubkey` in `tauri.conf.json` and
  the Apple Developer ID identity are **different keypairs solving different problems** (updater
  trust vs. Gatekeeper/TCC). A release needs **both** signatures; missing the minisign key →
  updater refuses every download ("signature verification failed"); missing the Apple signature →
  Gatekeeper blocks first install and breaks notarization. The release workflow must inject both
  env sets.
- **Unsigned/un-notarized downloaded artifact.** Even though the updater verifies *its own*
  minisign signature, the **downloaded `.app` must still be Apple-notarized** or macOS Gatekeeper
  will quarantine/block the replaced binary after relaunch. The updater path inherits all the
  notarization requirements of [[packaging-and-distribution]] — there is no shortcut.
- **Version monotonicity.** The plugin only offers an update when the manifest `version` is
  **strictly greater** (semver) than the installed `version`. Tag, `Cargo.toml`/`tauri.conf.json`
  `version`, and `latest.json` `version` must agree. A stale `latest.json` (lower version) →
  silent "no update". Keep `version` in `tauri.conf.json` (L4) as the single source.
- **Rollback.** There is **no built-in downgrade.** To pull a bad release, the manifest must be
  pointed back at an older artifact — but clients already on the bad version won't downgrade
  (monotonicity). Mitigation: ship a *higher* fixed version, not a lower one. Consider keeping
  releases as drafts until smoke-tested (`releaseDraft: true`) so a bad `latest.json` never goes
  public.
- **Chicken-and-egg: updating an already-installed *unsigned* build.** The very first
  distributed build must be the **signed + notarized + minisign-signed** one. A pre-updater build
  in the wild (e.g. a hand-shared unsigned `0.1.0`) has **no** updater plugin and **no** baked
  pubkey, so it can never auto-update — those users must reinstall manually once. Auto-update only
  helps from the first updater-enabled release forward.
- **macOS-only scope.** This app is macOS-first (SCK, `minimumSystemVersion 14.0`). The plan ships
  `darwin-aarch64` only. Adding `darwin-x86_64` means a second target build (and a universal or
  per-arch artifact in `latest.json`); Windows/Linux updater entries are irrelevant (no such
  builds). Don't add the `plugins.updater.windows` install-mode block.
- **Endpoint reachability / offline.** `check()` rejects/returns null if GitHub is unreachable;
  the launch check must swallow errors (no crash, no scary toast on a flaky network) — treat
  "couldn't reach update server" as "up to date for now".
- **Relaunch mid-recording.** `relaunch()` kills the process; never apply an update while
  `isExternalRecording` is true. Gate the Update button / defer the prompt.
- **`createUpdaterArtifacts` vs. `uploadUpdaterJson`.** These are two different switches:
  `createUpdaterArtifacts` (in `tauri.conf.json`) makes `tauri build` *produce* the `.tar.gz` +
  `.sig`; `tauri-action`'s `uploadUpdaterJson` (default `true`) makes the action *generate +
  upload* `latest.json`. Both are required for the GitHub-release flow.

## Testing & verification

- **Build gates (auto, no secrets):** `cargo build` + `cargo test` (default features,
  `DYLD_LIBRARY_PATH=/usr/lib/swift`), `npx tsc -b`, `npx vite build`, `npm run test` all green
  after the dep + config + Rust + frontend changes. `npm run tauri build` emits the
  `.app.tar.gz` + `.sig` (proves `createUpdaterArtifacts`).
- **Config sanity (auto):** `tauri.conf.json` validates against `$schema`; the `pubkey` is a
  non-empty minisign block; `endpoints` is a valid URL.
- **Frontend (auto):** a unit test for the progress-math in `applyUpdate` (the `got/total`
  fraction) if extracted as a pure helper — mirrors the [[test-harness-and-ci]] "test pure logic"
  pattern. The `check()`/`relaunch()` calls themselves need the live Tauri runtime and can't be
  unit-tested.
- **Release pipeline (CI, needs secrets):** a throwaway tag triggers `release.yml`; assert it
  produces a notarized `.dmg`, an `.app.tar.gz` + `.sig`, and a `latest.json`, and creates the
  GitHub release.
- **End-to-end (manual, needs two tagged releases):** install vN, release vN+1, confirm the
  installed app detects → downloads → minisign-verifies → installs → relaunches into vN+1 with no
  Gatekeeper prompt (step 8). This is the only true acceptance test and is inherently manual.
- **Cannot auto-verify:** Apple notarization, minisign round-trip on a real artifact, Gatekeeper
  behavior on the updated binary, and the relaunch — all require the live macOS runtime + an
  Apple account + the secrets ([[packaging-and-distribution]]).

## Out of scope / future

- **Intel / universal builds** (`darwin-x86_64` or a universal `.app`) — add a matrix arm in
  `release.yml` and a second `latest.json` platform key when an Intel audience appears.
- **Auto-download / silent background install** — this plan always prompts; a "download in
  background, install on quit" mode is a later refinement.
- **Release notes surfacing** — show `update.body` (manifest `notes`) in the prompt; trivial
  follow-up.
- **A dedicated update server** (vs. GitHub releases) with channels (stable/beta), staged
  rollout, or delta updates — overkill for a portfolio app.
- **Windows/Linux** updater entries — out of scope while the app is macOS-only.
- **CSP hardening for the update fetch** — the app currently has `app.security.csp: null`
  ([[packaging-and-distribution]] step); a real CSP must allow the updater endpoint host.

## References

- `src-tauri/tauri.conf.json` — `version` (4), `identifier` (5), `bundle` block: `targets` (29),
  `category` (30), `macOS.minimumSystemVersion` (41); no `plugins` / `createUpdaterArtifacts` yet.
- `src-tauri/Cargo.toml` — `tauri = "2"` (21), `tauri-plugin-opener = "2"` (22); add updater +
  process.
- `src-tauri/src/lib.rs` — builder chain: `.plugin(tauri_plugin_opener::init())` (79),
  `.manage` (80-81), `.invoke_handler` (82-92), `.run` (93); no `.setup()` today.
- `src-tauri/capabilities/default.json` — `permissions: ["core:default", "opener:default"]`.
- `.github/workflows/ci.yml` — setup steps to mirror (node 20 + cache 16-23, rust 25-26, cargo
  cache 28-36, `DYLD_LIBRARY_PATH: /usr/lib/swift` 48/53).
- `frontend/package.json` — `@tauri-apps/api ^2.9.1` (24); add the updater + process JS plugins.
- `frontend/src/components/asmr-recorder/toolbar.tsx` — `Button` (2), `Dialog` (13-20), `toast`
  (37), `Download`/`RotateCcw` icons (23-35), settings `Dialog` region (149+),
  `isExternalRecording` from context (62) — the update affordance + recording gate live here.
- Tauri v2 updater plugin (crate `tauri-plugin-updater`, JS `@tauri-apps/plugin-updater`):
  `plugins.updater` (`pubkey`/`endpoints`), `bundle.createUpdaterArtifacts`,
  `tauri signer generate`, `TAURI_SIGNING_PRIVATE_KEY[_PASSWORD]`, `check()` +
  `downloadAndInstall(cb)` + `relaunch()` (`@tauri-apps/plugin-process`),
  `Builder::new().build()` registration, capabilities `updater:default` / `process:allow-restart`
  — https://v2.tauri.app/plugin/updater/
- `tauri-apps/tauri-action` — tag-triggered build/sign/upload, `tagName`/`releaseName`/`args`
  inputs, `uploadUpdaterJson` (default `true`) — https://github.com/tauri-apps/tauri-action
- Related: [[packaging-and-distribution]] (Apple Developer ID signing + notarization this
  depends on), [[dev-tooling-and-build]] (default-features-only build; no `--features ffmpeg`),
  [[test-harness-and-ci]] (the CI workflow this release flow mirrors; pure-logic test pattern),
  [[next-implementations]] (roadmap index — move Auto-update from backlog to planned).

> **API-accuracy flags (verify against the installed plugin version):** (1) the desktop
> `.setup()` registration form (`app.handle().plugin(tauri_plugin_updater::Builder::new().build())`)
> and the plain `.plugin(tauri_plugin_process::init())` form are per the current v2 docs, but
> confirm against the exact `tauri-plugin-updater`/`tauri-plugin-process` versions `cargo add`
> resolves. (2) `tauri-action` is pinned `@v0` here; check for the latest major before use.
> (3) The progress callback event shape (`e.event` ∈ `Started`/`Progress`/`Finished`,
> `e.data.contentLength`/`chunkLength`) is from the docs — confirm field names in the installed
> `@tauri-apps/plugin-updater` types.
