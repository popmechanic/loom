# Distribution Reference

Building, signing, and distributing Loom desktop apps. Read this when preparing your app for distribution or when setting up code signing.

## Table of Contents

- [Build](#build)
- [Sharing the Build](#sharing-the-build)
- [Claude CLI as External Dependency](#claude-cli-as-external-dependency)
- [App Icon](#app-icon)
- [DMG Icon](#dmg-icon)
- [Code Signing and Notarization](#code-signing-and-notarization)
- [Auto-Updates](#auto-updates)

---

## Build

```bash
bunx electrobun build --env=stable
```

This produces a self-extracting archive (~12MB — the Bun runtime is the main
component; the system webview adds nothing to bundle size). Build artifacts
land in an `artifacts/` directory, named by channel, platform, and arch.

Build environments:
- **`dev`** — Fast iteration with terminal output, no code signing
- **`canary`** — Pre-release builds with optional signing and update manifests
- **`stable`** — Production-ready with full signing and distribution artifacts

Builds always target the current host platform and architecture. Cross-compilation
is not supported — build on macOS for macOS, Windows for Windows. If the user
wants to share with someone on a different architecture (e.g., Apple Silicon
app to an Intel Mac user), the recipient must build from source on their machine.

---

## Sharing the Build

**Always use the DMG, never a ZIP.** Zipping a `.app` bundle strips executable
permissions from binaries in `Contents/MacOS/`. The recipient opens the app and
nothing happens — the `MacOS/` folder appears empty. DMGs are disk images that
preserve file permissions, structure, and signatures exactly.

The build produces a DMG at:
```
artifacts/stable-macos-arm64-symmetry-terminal.dmg
```

Without code signing, the recipient must run `xattr -cr /path/to/app.app`
after dragging the app out of the DMG.

---

## Claude CLI as External Dependency

Don't bundle Claude CLI with the app. It updates independently, has its own
licensing, and the user needs it configured with their own credentials. The
startup check pattern (shown in `desktop-patterns.md#startup-cli-check`) handles
the missing case gracefully by showing a setup screen with install instructions.

---

## App Icon

To add a custom app icon:

1. Start with a 1024x1024 PNG source image.

2. Generate all required sizes:

```bash
mkdir icon.iconset
sips -z 1024 1024 source.png --out icon.iconset/icon_512x512@2x.png
sips -z 512 512   source.png --out icon.iconset/icon_512x512.png
sips -z 512 512   source.png --out icon.iconset/icon_256x256@2x.png
sips -z 256 256   source.png --out icon.iconset/icon_256x256.png
sips -z 256 256   source.png --out icon.iconset/icon_128x128@2x.png
sips -z 128 128   source.png --out icon.iconset/icon_128x128.png
sips -z 64 64     source.png --out icon.iconset/icon_32x32@2x.png
sips -z 32 32     source.png --out icon.iconset/icon_32x32.png
sips -z 32 32     source.png --out icon.iconset/icon_16x16@2x.png
sips -z 16 16     source.png --out icon.iconset/icon_16x16.png
```

3. Add to `electrobun.config.ts`:

```typescript
mac: {
  icons: "icon.iconset",
},
```

---

## DMG Icon

ElectroBun does not automatically set a volume icon on the DMG. To stamp
your app icon onto the DMG file after building:

```bash
iconutil -c icns icon.iconset -o icon.icns
sips -i icon.icns
DeRez -only icns icon.icns > /tmp/icon.rsrc
Rez -append /tmp/icon.rsrc -o artifacts/stable-macos-arm64-*.dmg
SetFile -a C artifacts/stable-macos-arm64-*.dmg
```

**Xcode Dependency:** `Rez`, `DeRez`, and `SetFile` ship only with full Xcode
and are deprecated. On a Command Line Tools-only machine or in CI, these tools
silently no-op and the volume icon won't be set.

For CI pipelines or CLT-only machines, use the `fileicon` tool instead (install
via `brew install fileicon`):

```bash
iconutil -c icns icon.iconset -o icon.icns
fileicon set artifacts/stable-macos-arm64-*.dmg icon.icns
```

Or set the volume icon during DMG creation using a tool like `create-dmg`.

---

## Code Signing and Notarization

Without signing, macOS Gatekeeper blocks the app. The recipient can bypass
with `xattr -cr`, but for clean distribution you need an Apple Developer
account ($99/year).

**Setup steps:**

1. **Apple Developer Account** — enroll at [developer.apple.com](https://developer.apple.com)

2. **Create signing certificate** — open Xcode > Settings > Accounts, add your
   developer account, click Manage Certificates, create a **Developer ID
   Application** certificate

3. **Create App Identifier** — in the Apple Developer portal under Certificates,
   Identifiers & Profiles > Identifiers, create one matching your app's
   `identifier` (e.g., `com.loom.symmetry-terminal`). Check **App Attest**.

4. **Generate app-specific password** — at [account.apple.com](https://account.apple.com)
   under Sign-In and Security > App-Specific Passwords

5. **Set environment variables** — add to `~/.zshrc`:

```bash
export ELECTROBUN_DEVELOPER_ID="Developer ID Application: Your Name (TEAMID)"
export ELECTROBUN_TEAMID="TEAMID"
export ELECTROBUN_APPLEID="your@email.com"
export ELECTROBUN_APPLEIDPASS="xxxx-xxxx-xxxx-xxxx"
```

The `DEVELOPER_ID` is the full signing identity: certificate type + team
name + team ID in parentheses. Find it in Xcode's certificate manager.

6. **Enable in config:**

```typescript
mac: {
  codesign: true,
  notarize: true,
},
```

7. **Build:** `bunx electrobun build --env=stable` — ElectroBun signs all
   binaries, submits to Apple for notarization (takes 2-5 minutes), and
   produces a DMG that opens cleanly on any Mac.

**Windows** — Authenticode signing via the build configuration. Unsigned apps
trigger SmartScreen warnings. See ElectroBun docs for certificate setup.

---

## Auto-Updates

ElectroBun uses BSDIFF binary patching for incremental updates — subsequent
patches can be as small as 14KB. Configure the update URL in your config:

```typescript
// In electrobun.config.ts:
export default {
  // ...app and build config...
  release: {
    baseUrl: "https://your-cdn.com/releases",
  },
} satisfies ElectrobunConfig;
```

Upload the `artifacts/` directory to any static file host (S3, Cloudflare R2,
GitHub Releases). No server infrastructure needed.

The Updater API in the Bun process:

```typescript
import { Updater } from "electrobun/bun";

// Check for updates on launch
const localInfo = Updater.getLocalInfo();
const update = await Updater.checkForUpdate();

if (update) {
  // Download the patch (incremental if available, full fallback)
  await Updater.downloadUpdate();
  // Apply: closes app, swaps binary, relaunches
  Updater.applyUpdate();
}
```
