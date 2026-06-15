# AI Development Guidelines - Cursor Rules and Documentation

This directory contains Cursor rules (`.mdc` files) and documentation for AI-assisted development of the busansashimi portfolio project.

---

## Project Structure Overview

```
busansashimi/
├── project/          # Main Next.js project (deployed service)
│   ├── amplify/      # AWS Amplify Gen 2 backend
│   ├── src/          # Source code
│   │   ├── app/      # Next.js App Router pages
│   │   ├── components/  # UI components (shadcn/ui)
│   │   ├── content/  # Article content
│   │   ├── hooks/    # Custom React hooks
│   │   ├── lib/      # Utilities and config
│   │   ├── types/    # TypeScript types
│   │   └── utils/    # API utilities
│   ├── system/       # Custom system components
│   └── public/       # Static assets
├── reference/        # v0-maintained design reference project
│   ├── app/          # Reference pages and components
│   ├── components/   # Reference UI components
│   └── ...
└── ai-dev/           # Cursor rules and documentation
    ├── *.mdc         # Cursor rule files
    ├── doc/          # Additional documentation
    └── README.md     # This file
```

---

## Cursor Rules (.mdc Files)

The following rule files are applied to guide AI-assisted development:

| File                             | Purpose                                                      |
| -------------------------------- | ------------------------------------------------------------ |
| `nextjs-typescript-tailwind.mdc` | Next.js, TypeScript, and Tailwind CSS development guidelines |
| `shadcn-nextjs.mdc`              | shadcn/ui and Next.js best practices                         |
| `tailwind.mdc`                   | Tailwind CSS conventions and patterns                        |
| `tailwind-shadcn.mdc`            | Tailwind + shadcn integration guidelines                     |
| `clean-code.mdc`                 | Clean code principles and readability                        |
| `code-quality.mdc`               | Code quality standards and error handling                    |
| `dev-log.mdc`                    | Automatic logging of operations to daily log files           |

---

## Directory Explanations

### project/

The main development project that is deployed as a production service.

```
project/
└── src/
    └── app/
        └── ...           # Project that is actually developed to deploy as a service
```

### reference/

A reference project maintained by v0 (Vercel's AI design tool) as a design reference. Use this directory to compare designs and component implementations.

```
reference/
└── app/
    └── ...               # Reference project that is maintained by v0 as design reference
```

---

## Tech Stack

### Frontend

- Next.js 15 (App Router)
- React 19
- TypeScript
- Tailwind CSS v4
- SCSS Modules
- shadcn/ui
- Radix UI

### Backend

- AWS Amplify Gen 2
- GraphQL (AWS AppSync)
- DynamoDB
- AWS Cognito (Authentication)

---

## Usage

1. **For AI-assisted development**: The `.mdc` files are automatically applied by Cursor to guide code generation and suggestions.

2. **For design reference**: Compare implementations between `project/` and `reference/` directories when working on UI components.

3. **For documentation**: Add additional documentation files to the `doc/` subdirectory as needed.

---

## Local Development / Build (asmr-recorder)

**Launch command:** `npm run tauri dev` (from repo root). The root `package.json` `tauri` script injects `DYLD_LIBRARY_PATH=/usr/lib/swift` for the screencapturekit Swift bindings, so always use it rather than calling `cargo run` or `tauri dev` directly. Alternatively run `./dev.sh`, which delegates to the same npm script.

**Default cargo features (no `--features ffmpeg`):** the live recording/trim/save flow — WebCodecs composite → H.264 → mediabunny → raw-byte IPC — does not need ffmpeg. `cargo build` with default features links in ~7-9s.

**`--features ffmpeg` is opt-in and currently broken on arm64:** it only powers the dormant native `src/encoder.rs` path (unused by the browser pipeline). On this machine `ld` cannot resolve `_avcodec_*`/`_avformat_*`/`_sws_*` because `.cargo/config.toml` points `-L` at `/opt/homebrew/lib` and the required dylibs are absent. To use it: install `ffmpeg` via Homebrew (`brew install ffmpeg`) and accept that the resulting dynamic link is not distributable. Do **not** try to build with `--features ffmpeg` for the live recording pipeline — it is not needed.

**Pre-existing tsc errors (do not regress):** `browser-recorder.tsx` dead plugin imports, `region-selector.tsx` `HANDLE_SIZE`, `recording-context.tsx` `SectionConfig` — 4 errors total; unrelated to the recording/trim flow.

---

## Release Build — signed + notarized (macOS Developer ID)

**Prerequisites:**
1. An [Apple Developer Program](https://developer.apple.com/programs/) membership ($99/yr).
2. A **Developer ID Application** certificate installed in your login keychain.
   Verify: `security find-identity -v -p codesigning` — look for `Developer ID Application: …`.
3. macOS 14+ on the build machine (matches the `minimumSystemVersion: "14.0"` floor).

**Set credentials (never commit these):**
```sh
cp .env.example .env    # fill in APPLE_SIGNING_IDENTITY + notarization vars
source .env             # or export each var in your shell
```

**Build (default features only — never `--features ffmpeg`):**
```sh
npm run tauri build
```
Tauri reads `APPLE_SIGNING_IDENTITY` and the notarization vars automatically, signs the
`.app` with the hardened runtime, submits to Apple's notarization service, and staples the
ticket to the `.dmg`.

**Verify the signed output:**
```sh
# Adjust the app name/version path as needed
APP="src-tauri/target/release/bundle/macos/ASMR Recorder.app"
DMG="src-tauri/target/release/bundle/dmg/ASMR Recorder_*.dmg"

# Code signature
codesign --verify --deep --strict --verbose=2 "$APP"

# Entitlements (should show camera + audio-input + hardened runtime)
codesign -d --entitlements - "$APP"

# Info.plist floor
plutil -p "$APP/Contents/Info.plist" | grep -E 'LSMinimumSystemVersion|LSApplicationCategoryType'

# Notarization / Gatekeeper acceptance
xcrun stapler validate "$APP"
spctl -a -vvv -t install "$DMG"
```

**Clean-machine smoke test:**
Copy the `.dmg` to a second Mac (or fresh user account) running macOS 14+. Install and
launch: the "damaged" dialog must **not** appear; camera/mic/screen-recording TCC prompts
appear once and persist across relaunch. Record a short clip end-to-end.

**Notes:**
- The `ffmpeg` cargo feature links Homebrew dylibs that are neither signed nor present on a
  clean machine → notarization will fail. **Always build for release with default features.**
- Screen-recording permission has no entitlement or Info.plist key. It is a TCC grant
  triggered on first `SCStream` start. Stable Developer ID signing is what makes the grant
  persist across rebuilds.
- If the first notarized build flags the `/usr/lib/swift` load (unusual — these are system
  libs), add `com.apple.security.cs.disable-library-validation` to `entitlements.plist` and
  rebuild. Do not add it speculatively.

---

## Key Development Guidelines

- Use TypeScript for all code; prefer interfaces over types
- Use functional and declarative programming patterns
- Follow mobile-first responsive design with Tailwind CSS
- Minimize `use client`; favor React Server Components (RSC)
- Use descriptive variable names with auxiliary verbs (e.g., `isLoading`, `hasError`)
- Implement accessibility features on all interactive elements
- Handle errors and edge cases early with guard clauses
