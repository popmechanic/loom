# Gotchas Reference

Critical issues from real-world Loom desktop development. Read these before building — they'll save you hours of debugging.

## Table of Contents

- [1. Environment variable cleaning](#1-environment-variable-cleaning)
- [2. Stream buffering](#2-stream-buffering)
- [3. TextDecoder with stream true](#3-textdecoder-with-stream-true)
- [4. dontAsk blocks reads outside project dir](#4-dontask-blocks-reads-outside-project-dir)
- [5. Extended thinking models](#5-extended-thinking-models)
- [6. RPC request timeout](#6-rpc-request-timeout)
- [7. ElectroBun version status](#7-electrobun-version-status)
- [8. System webview differences](#8-system-webview-differences)
- [9. Spawned Claude inherits user hooks](#9-spawned-claude-inherits-user-hooks)
- [10. Missing index.html](#10-missing-indexhtml)
- [11. stderr is a ReadableStream — read it once](#11-stderr-is-a-readablestream--read-it-once)
- [12. File.path doesn't exist in system webviews](#12-filepath-doesnt-exist-in-system-webviews)
- [13. macOS GUI apps don't inherit shell PATH](#13-macos-gui-apps-dont-inherit-shell-path)
- [14. ZIP strips executable permissions](#14-zip-strips-executable-permissions)
- [15. No cross-compilation](#15-no-cross-compilation)
- [16. Process crash recovery](#16-process-crash-recovery)

---

### 1. Environment variable cleaning

**Remove:** `CLAUDECODE` and `CLAUDE_CODE_ENTRYPOINT` — these nesting
guards block `claude -p` from spawning when your dev environment runs inside
Claude Code.

**Do not remove:** `CLAUDE_CODE_OAUTH_TOKEN` and other `CLAUDE_*` auth
vars. Stripping all `CLAUDE_*` variables kills authentication and causes
silent failures.

**Also remove in cmux:** `CMUX_SURFACE_ID`, `CMUX_PANEL_ID`, `CMUX_TAB_ID`,
`CMUX_WORKSPACE_ID`, `CMUX_SOCKET_PATH` — these trigger nesting detection
in the Claude terminal app.

Always use `cleanEnv()`. Never hand-filter environment variables.

---

### 2. Stream buffering

TCP chunks split JSON lines arbitrarily. Never do this:

```typescript
// BROKEN: chunk may contain half a JSON line
chunk.toString().split("\n").forEach(line => JSON.parse(line));
```

Always use `createStreamParser` to buffer across chunk boundaries.

---

### 3. TextDecoder with `{ stream: true }`

Multi-byte UTF-8 characters (emoji, CJK, accented characters) can split
across chunk boundaries. `chunk.toString()` corrupts them. Always use:

```typescript
const decoder = new TextDecoder();
buffer += decoder.decode(chunk, { stream: true });
```

The `{ stream: true }` option tells the decoder to hold incomplete byte
sequences for the next chunk.

---

### 4. `dontAsk` blocks reads outside project dir

`--permission-mode dontAsk` auto-denies everything not explicitly allowed,
including reads outside the project directory. If a user drops a file from
`~/Desktop` onto your app and Claude tries to `Read` it, `dontAsk` will
deny the request.

**Solution:** Use `--permission-mode bypassPermissions` instead for desktop
apps. You control the prompt and tools list, so `bypassPermissions` is safe
and avoids the filesystem scope restriction.

With `bypassPermissions` you must pass an explicit, minimal `--tools`
allowlist; never omit it — omitting `--tools` grants access to all tools on
the whole filesystem. Treat dropped-file content as untrusted input: it must
not be able to expand the effective tool scope.

**Alternative:** Read files in the Bun process (via `FileReader` in the
webview → RPC → Bun), and include the content directly in the prompt. This
way Claude doesn't need `Read` access at all for dropped files.

`--tools ""` (empty string) is also fragile — it can cause ambiguous CLI
behavior. For pure reasoning tasks, omit `--tools` entirely.

---

### 5. Extended thinking models

Models with extended thinking (haiku-4.5, sonnet-4.6, and others) deliver
text as complete `assistant` content blocks — not incremental `stream_event`
deltas. The `assistant` event arrives with an array like
`[{type:"thinking"}, {type:"text"}]`. The thinking block has no visible
output, so the UI looks "stuck" until the text block arrives.

Your stream handler processes both delivery patterns:

- `assistant` events with `message.content[].type === "text"` blocks (thinking models)
- `stream_event` events with `event.delta.text` deltas (non-thinking models)

Filter out `type: "thinking"` blocks — they contain no user-visible content.
The `deriveAndSendRPC()` function in `desktop-patterns.md#deriveandsendrpc` handles
both patterns by checking `block.type === "text"` and ignoring thinking blocks.

If the UI appears frozen during the thinking phase, show a "Thinking..."
indicator via the `status` heartbeat messages.

---

### 6. RPC request timeout

ElectroBun's default RPC request timeout may be too short for `startTask`
calls with long prompts or slow model startup. If `startTask` times out
before Claude spawns, increase the timeout in your RPC configuration or
return the `taskId` immediately (before Claude finishes) as shown in the
streaming pattern.

---

### 7. ElectroBun version status

ElectroBun is at v1.15.1 stable (March 2026). Known platform issues:

- **Linux**: Self-extracting binary issues on some distributions
- **Windows**: WebView2 preload script truncation in some configurations
- **No `saveFileDialog()`** — use `Bun.write()` with a known path as workaround
- **Menu support**: Full on macOS, basic on Windows, not supported on Linux

New in v1.15.x: `--watch` race condition fixed, WebGPU/WGPU native rendering,
`GpuWindow`, `GlobalShortcut`, `Screen`, `Session`, `Socket`, `BuildConfig` APIs.

Check [github.com/blackboardsh/electrobun/issues](https://github.com/blackboardsh/electrobun/issues)
for current status.

---

### 8. System webview differences

The webview is not a bundled browser — it's the system's native webview:

| Platform | Engine | Notes |
|----------|--------|-------|
| macOS | WKWebView (WebKit) | Most consistent, best support |
| Windows | WebView2 (Chromium) | Requires Edge WebView2 Runtime |
| Linux | WebKit2GTK | Version varies by distro, test carefully |

CSS and JavaScript behavior can differ across engines. Test on all target
platforms. For consistent rendering, consider enabling CEF (Chromium Embedded
Framework) via the `bundleCEF` build option — but this increases bundle size
significantly.

---

### 9. Spawned Claude inherits user hooks

`claude -p` loads `~/.claude/` hooks and settings (superpowers, vibes, etc.)
by default. This adds seconds of startup time and massive prompt bloat from
skill SessionStart hooks.

**Fix:** Add `--setting-sources ""` to skip user and project settings entirely.
This flag tells the CLI to not load any settings files, which prevents hooks
from running on the spawned process.

```bash
claude -p --setting-sources "" --permission-mode bypassPermissions "task"
```

**Do NOT use** `--no-user-config` — that flag doesn't exist and will cause
an error. `--setting-sources ""` is the correct way to skip settings.

---

### 10. Missing index.html

ElectroBun bundles JavaScript and CSS from your entrypoints but does NOT
auto-generate an HTML file. The webview loads `views://mainview/index.html`,
so you must create it manually.

Create `src/mainview/index.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>My Claude App</title>
</head>
<body>
  <div id="root"></div>
  <script src="index.js"></script>
</body>
</html>
```

Then add a `copy` directive to `electrobun.config.ts` so the HTML file is
included in the build:

```typescript
build: {
  views: {
    mainview: {
      entrypoint: "src/mainview/index.ts",
    },
  },
  copy: {
    "src/mainview/index.html": "views/mainview/index.html",
  },
},
```

Without this, the webview loads a blank page with no error message.

---

### 11. stderr is a ReadableStream — read it once

`proc.stderr` is a `ReadableStream`. If you consume it in one place (e.g.,
a diagnostic reader), any subsequent attempt to read it (e.g., in an error
handler) will throw `ReadableStream already used`.

**Fix:** Collect stderr proactively into a buffer variable, then reference
the buffer in error handlers:

```typescript
const stderrChunks: string[] = [];
const stderrDecoder = new TextDecoder();
(async () => {
  const reader = proc.stderr.getReader();
  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      stderrChunks.push(stderrDecoder.decode(value, { stream: true }));
    }
  } catch {}
})();

// Later, in error handler:
const stderr = stderrChunks.join("");
```

---

### 12. `File.path` doesn't exist in system webviews

`File.path` is an Electron-specific extension. In WKWebView (macOS),
WebView2 (Windows), and WebKit2GTK (Linux), dropped files do NOT have a
`.path` property. Accessing it returns `undefined`.

**Fix:** Use `FileReader.readAsText()` in the webview to read file content,
then send the content to the Bun process over RPC. See
`desktop-features.md#file-drag-and-drop` for the complete pattern.

---

### 13. macOS GUI apps don't inherit shell PATH

When launched from Finder (not terminal), the Bun process gets a minimal
PATH (`/usr/bin:/bin:/usr/sbin:/sbin`). `claude` installed via npm is
typically at `/usr/local/bin/claude` or `~/.local/bin/claude` — neither
of which is in the GUI PATH. The app launches fine but every `claude -p`
spawn fails with "Executable not found in $PATH".

**Fix:** Resolve the absolute path to `claude` once at startup using a
login shell, then use it for all subsequent spawns:

```typescript
import fs from "fs";

function resolveClaudePath(): string {
  // Try interactive login shell (sources both .zprofile AND .zshrc)
  for (const flags of ["-lic", "-lc", "-ic"]) {
    try {
      const result = Bun.spawnSync(["zsh", flags, "which claude"], {
        timeout: 5000,
      });
      const resolved = result.stdout.toString().trim();
      if (resolved && result.exitCode === 0 && !resolved.includes("not found")) {
        return resolved;
      }
    } catch {}
  }

  // Direct path check — common install locations
  const home = process.env.HOME || "";
  const candidates = [
    `${home}/.claude/local/claude`,
    `/usr/local/bin/claude`,
    `/opt/homebrew/bin/claude`,
    `${home}/.local/bin/claude`,
    `${home}/.npm-global/bin/claude`,
  ];

  for (const p of candidates) {
    try {
      fs.accessSync(p, fs.constants.X_OK);
      return p;
    } catch {}
  }

  return "claude"; // fallback (works if launched from terminal)
}

const CLAUDE_BIN = resolveClaudePath();
```

Then use `CLAUDE_BIN` instead of `"claude"` in all `Bun.spawn()` calls.
The login shell (`zsh -l`) sources `~/.zshrc` and picks up the full PATH.

**Always include `resolveClaudePath()`** in every Loom desktop app. This
is required for distribution — the app will work in dev (launched from
terminal) but fail for end users who double-click it in Finder.

---

### 14. ZIP strips executable permissions

Never distribute a `.app` bundle via ZIP. Zipping strips executable
permissions from binaries in `Contents/MacOS/`. The recipient opens the
app and nothing happens — the `MacOS/` folder appears empty. Always use
the DMG from `artifacts/`. DMGs are disk images that preserve permissions,
structure, and code signatures exactly.

---

### 15. No cross-compilation

`bunx electrobun build` targets the current host platform and architecture
only. An Apple Silicon build won't run on an Intel Mac (and vice versa).
If the recipient has a different architecture, they must build from source
on their own machine.

---

### 16. Process crash recovery

Claude processes can fail for transient reasons — network timeouts, rate
limits, or API errors. Desktop apps should handle this gracefully:

```typescript
async function spawnWithRetry(
  taskId: string,
  prompt: string,
  opts: any,
  rpc: any,
  maxRetries = 2
) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      const proc = spawnClaude(taskId, prompt, opts, rpc);
      const exitCode = await proc.exited;
      if (exitCode === 0) return;
      if (attempt < maxRetries) {
        console.warn(`Claude exited ${exitCode}, retrying (${attempt + 1}/${maxRetries})...`);
        await new Promise(r => setTimeout(r, 1000 * (attempt + 1)));
      }
    } catch (err) {
      if (attempt === maxRetries) throw err;
    }
  }
}
```

Don't retry on user-initiated aborts (`SIGTERM`). Only retry on unexpected
exits (non-zero exit code without an abort request).
