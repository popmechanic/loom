---
name: loom-desktop
description: >
  Build native desktop applications where Claude Code CLI (`claude -p`) is the
  runtime, using ElectroBun (Bun + system webview) with typed RPC — no HTTP
  server, no auth.
when_to_use: >
  Use when building a native macOS/Windows/Linux tool with drag-and-drop, file
  dialogs, system tray, or desktop distribution. Triggers: "desktop app with
  Claude", "native Claude tool", "ElectroBun Claude app", "Claude on the dock".
  NOT for web apps (use loom / loom-local), direct Anthropic API, or chat
  replicas.
---

# Loom Desktop: Native Apps on the Claude Code Runtime

You're helping someone build a desktop application where Claude Code is the
runtime — not a chat wrapper, but a native tool where Claude reads files, runs
commands, and streams structured output to a local UI.

This is the desktop counterpart to web Loom. Where web Loom puts an Express
server between the browser and Claude, desktop Loom removes that layer entirely.
The Bun process spawns `claude -p` directly and pushes output to the webview via
typed RPC. No HTTP. No SSE formatting. No proxy timeouts. No auth.

Desktop means one user on their own machine. That simplifies everything: no
sessions to manage, no credentials to pass per-request, no rate limiting across
users. The architecture collapses to a thin bridge between subprocess and UI.

## Why Desktop

The question to help users explore is: **what would you build if Claude had full
access to your local filesystem, ran as a native app on your dock, and could
stream its reasoning to a custom interface — with zero server infrastructure?**

A desktop Loom app has things web apps don't:

- **Full local file access** — Claude reads and writes files on the user's machine directly. No upload step.
- **Native UX** — System tray, native menus, file dialogs, drag-and-drop. Feels like a real tool, not a browser tab.
- **Offline-first** — The app launches instantly. Claude needs network for inference, but the interface itself has no server dependency.
- **No infrastructure** — No VM to provision, no server to keep running, no auth to maintain. Ship a binary.
- **Privacy** — Files never leave the machine. Claude processes them locally via the CLI.

## Prerequisites

Before building, you need:

- **Claude CLI** installed and on PATH — `claude --version` should return a version string
- **Bun** v1.1+ — [bun.sh](https://bun.sh)
- **ElectroBun** — `bun add electrobun` — [github.com/blackboardsh/electrobun](https://github.com/blackboardsh/electrobun)
- **Platform**: macOS 14+, Windows 11+, or Ubuntu 22.04+ (other Linux with gtk3 & webkit2gtk-4.1)

ElectroBun is currently at v1.15.1 stable. It's actively maintained but
still evolving — check the issue tracker for platform-specific concerns.

## The Architecture: Thin Bridge

Every Loom desktop app has this shape:

```
┌─────────────────────────────────────────────────────┐
│                    ElectroBun App                     │
│                                                       │
│  ┌──────────────┐    Typed RPC     ┌──────────────┐  │
│  │   Webview     │◄───messages────►│  Bun Process  │  │
│  │  (React/HTML) │                 │               │  │
│  │               │  rpc.request    │  Claude       │  │
│  │  Renders UI   │──startTask()──►│  Manager      │  │
│  │  Shows tokens │  rpc.request    │               │  │
│  │  Tool status  │──abort()──────►│  Spawns       │  │
│  │               │                 │  claude -p    │  │
│  │               │◄─rpc.sendProxy │               │  │
│  │               │  .token()       │  Parses       │  │
│  │               │  .toolUse()     │  stdout       │  │
│  │               │  .done()        │  stream       │  │
│  │               │  .error()       │               │  │
│  │               │  .status()      │               │  │
│  └──────────────┘                 └──────┬───────┘  │
│                                          │           │
│                                    stdio │           │
│                                          ▼           │
│                                   ┌──────────┐       │
│                                   │ claude -p │       │
│                                   │ subprocess│       │
│                                   └──────────┘       │
└─────────────────────────────────────────────────────┘
```

**Webview**: The custom UI. React, vanilla HTML, whatever fits the app. Runs in
a system webview (WKWebView on macOS, WebView2 on Windows, WebKit2GTK on Linux).

**Bun Process**: The bridge. Receives requests from the webview via typed RPC,
spawns `claude -p` subprocesses, parses their NDJSON stdout, and pushes results
back as typed RPC messages. This is the Claude Manager — typically 80-100 lines
of TypeScript.

**Claude Subprocess**: The intelligence. `claude -p` with the right flags. This
is where the agentic work happens — reading files, running commands, generating
structured output.

### Why RPC Instead of HTTP

Web Loom uses Express + SSE because browsers need HTTP. Desktop Loom skips that:

| Concern | Web Loom (HTTP/SSE) | Desktop Loom (RPC) |
|---------|--------------------|--------------------|
| Transport | HTTP + SSE streams | WebSocket (encrypted) |
| Type safety | Manual JSON parsing | Compile-time typed schema |
| Proxy timeouts | Heartbeat hacks needed | Not applicable |
| Auth | OAuth per-request | Not needed (single user) |
| Complexity | ~200 lines of plumbing | ~20 lines of plumbing |

The typed RPC contract is defined once and shared between Bun and webview. See
`references/rpc-schema-reference.md` for the full schema.

## Project Setup

Scaffold an ElectroBun project:

```bash
bunx electrobun init hello-world
cd hello-world
bun install
bunx electrobun dev
```

This gives you a running app with a Bun process and webview. From here, you'll
add the Claude Manager to the Bun side and the task UI to the webview.

For the full setup guide — project structure, configuration, dev workflow — see
`references/electrobun-setup.md`.

For Claude CLI flags and output formats, see `references/cli-runtime-reference.md`.

## The Conversation

When someone comes to you with a desktop app idea, walk through these design
questions. Don't dump them all at once — have a natural conversation. But cover
this ground before you start building.

### 1. What does this app do?

Get concrete about the interface. What does someone see when they open it? What
do they click? What appears when Claude is working? What does the output look
like?

Desktop apps have a stronger "tool" identity than web apps. A web app might be a
dashboard someone visits. A desktop app is something someone installs and
reaches for. What's the tool? What problem does it solve?

Ask: "Walk me through opening this app. What do you see? What do you do first?"

### 2. What Claude capabilities does it need?

Map the app's features to what Claude does behind the scenes:

| Need | Claude Configuration |
|------|---------------------|
| Read and analyze files | `--tools "Read,Glob,Grep"` |
| Modify code or files | `--tools "Read,Edit,Write,Glob,Grep,Bash"` |
| Pure reasoning, no file access | Omit `--tools` and use `--permission-mode bypassPermissions` |
| Web research | `--tools "WebSearch,WebFetch"` |
| Structured data extraction | `--json-schema '{...}'` |
| Custom persona | `--system-prompt "You are..."` |

The tools list determines what Claude can do. Tighter is safer — only grant what
the app actually needs.

**Note:** `--tools ""` (empty string) is fragile and causes ambiguous CLI
behavior. For pure reasoning tasks, prefer omitting `--tools` entirely and
relying on `bypassPermissions` with no tool invocations, or pass file content
directly in the prompt so Claude doesn't need tools.

### 3. What's the interaction model?

How does the person's action translate to a Claude invocation?

| Interaction | Pattern |
|-------------|---------|
| Click a button, get a result | **Synchronous** — spawn, collect, return |
| Watch progress in real-time | **Streaming** — tokens flow as RPC messages |
| Multi-step conversation with context | **Conversational** — `--session-id` first turn, `--resume <id>` after |
| Long task, minimize to tray | **Background** — task registry + tray status |

Most desktop apps start with Streaming. Add Conversational if the user needs
follow-up questions. Add Background for tasks that take minutes.

### 4. What desktop features matter?

This is where desktop Loom diverges from web Loom. Ask which native features
the app needs:

- **File drag-and-drop** — Drop files onto the window to feed them to Claude
- **Native file dialogs** — "Open File..." to select input, save results to disk
- **System tray** — Minimize during long tasks, show progress in tooltip
- **Native menus** — App menu bar with New Task, Abort, Model Selection, Settings
- **Notifications** — Native OS notification when a background task completes

Not every app needs all of these. A simple analysis tool might only need
drag-and-drop. A code assistant might need the full set.

### 5. What are the cost and safety boundaries?

Desktop simplifies auth (no OAuth needed) but cost and permissions still matter:

- **Budget**: `--max-budget-usd` caps spending per task (print mode only). What's appropriate? $0.50 for quick analysis, $5 for deep codebase review. Pair with `--max-turns` to bound cost *and* loop count.
- **Model**: Haiku 4.5 for fast/cheap, Sonnet 4.6 for balanced, Opus 4.8 for best quality; `fable` (Fable 5) is the most capable alias — enable it for the hardest agentic work once it's available on your plan. `--fallback-model sonnet,haiku` (comma-separated, tried in order) for reliability.
- **Reasoning depth**: `--effort low|medium|high|xhigh|max` is a separate axis from the model. `xhigh` is Claude Code's own default for coding/agentic work; use `low`/`medium` for quick extraction.
- **Permissions**: Use `--permission-mode bypassPermissions` with explicit `--tools` list. `dontAsk` auto-denies anything not whitelisted, but also blocks reads outside the project directory — meaning dropped files from ~/Desktop won't be accessible. `bypassPermissions` is safer for desktop apps where you control the prompt.
- **Turn limit**: `--max-turns` prevents runaway agent loops.
- **Filesystem scope**: Consider constraining Claude to specific directories via `--allowedTools "Read(/path/**)"` patterns.

## Building It

Default to **TypeScript** for both the Bun process and the webview. Use
**React** for the UI unless the person prefers vanilla HTML.

### Shared Utilities

Four helpers used by every pattern: `resolveClaudePath()` (find claude binary),
`cleanEnv()` (remove nesting guards), `createStreamParser()` (buffer stdout
into JSON lines), and `deriveAndSendRPC()` (map stream events to RPC messages).

> See `references/desktop-patterns.md#shared-utilities` for the full
> implementations with explanatory prose.

### Event Type Mapping

`claude -p --output-format stream-json --verbose` emits newline-delimited JSON.
Each event maps to an RPC message:

| Stream Event | RPC Message | Notes |
|---|---|---|
| `system` (subtype: init) | `status` (state: "running") | Session started |
| `assistant` (text content block) | `token` | Complete text for thinking models |
| `stream_event` (content_block_delta) | `token` | Incremental for non-thinking models |
| `assistant` (tool_use content block) | `toolUse` | Tool invocation |
| `tool_result` | `toolResult` | Tool output |
| `result` (subtype: success) | `done` | Session complete with cost |
| parse error / process crash | `error` | Process failure |

### Patterns at a Glance

Most desktop apps start with Streaming — it's the primary pattern. Tokens flow
from Claude's stdout through Bun to the webview as typed RPC messages, with
status heartbeats every 2 seconds.

| Pattern | When to Use | Reference |
|---------|-------------|-----------|
| Synchronous | Quick extraction, classification | `references/desktop-patterns.md#pattern-1-synchronous` |
| Streaming | Primary pattern — tokens flow to UI | `references/desktop-patterns.md#pattern-2-streaming` |
| Conversational | Multi-turn with context retention | `references/desktop-patterns.md#pattern-3-conversational` |
| Background | Long tasks, tray minimization | `references/desktop-patterns.md#pattern-4-background` |

> Read `references/desktop-patterns.md` when implementing a specific
> interaction pattern — it has the complete Bun-side and webview-side code.

### Desktop Features

| Feature | Description | Reference |
|---------|-------------|-----------|
| File Drag-and-Drop | Drop files for Claude to analyze (use FileReader, not File.path) | `references/desktop-features.md#file-drag-and-drop` |
| Native File Dialogs | Open/save files via Utils.openFileDialog() | `references/desktop-features.md#native-file-dialogs` |
| System Tray | Minimize during long tasks, show progress | `references/desktop-features.md#system-tray` |
| Native Menus | App menu bar with keyboard shortcuts | `references/desktop-features.md#native-menus` |
| File Access Config | Permission modes and tools lists for desktop | `references/desktop-features.md#local-file-access-configuration` |

> Read `references/desktop-features.md` when adding native platform features
> to your app.

### Distribution

Build with `bunx electrobun build --env=stable`. Always distribute the DMG,
never a ZIP (zipping strips executable permissions). Claude CLI is an external
dependency — don't bundle it. For clean distribution without Gatekeeper warnings,
set up code signing with an Apple Developer account.

> Read `references/distribution.md` for the complete build, signing, DMG icon,
> and auto-update setup.

### Patterns at a Glance (extended)

Two additional patterns beyond the four core ones:

| Pattern | When to Use | Reference |
|---------|-------------|-----------|
| Interrupt, done right | Stop button that also kills tool grandchildren | `references/desktop-patterns.md#interrupt-done-right` |
| Surviving a webview reload | Dev reload or crash-recovery re-attaches to running process | `references/desktop-patterns.md#surviving-a-webview-reload` |

The duplex bridge pattern (one long-lived `claude -p --input-format stream-json` process fed turns on stdin) is covered in
`references/desktop-patterns.md#persistent-duplex-bridge-for-multi-turn` as a lower-latency alternative to per-turn `--resume`.

### Gotchas

Critical issues from real-world development. These 4 bite everyone:

- **#1 Env cleaning** — Remove `CLAUDECODE` and `CLAUDE_CODE_ENTRYPOINT`, but never `CLAUDE_CODE_OAUTH_TOKEN`. Use `cleanEnv()`.
- **#2 Stream buffering** — TCP chunks split JSON lines. Use `createStreamParser`, never raw `split("\n")`.
- **#13 macOS PATH** — GUI apps don't inherit shell PATH. Use `resolveClaudePath()` at startup.
- **#12 File.path** — Doesn't exist in system webviews. Use `FileReader.readAsText()` via RPC instead.
- **Bare kill orphans grandchildren** — `proc.kill()` only signals the top-level process. Spawn with `detached: true` and signal the process group to kill Bash subtrees too. See `references/desktop-patterns.md#interrupt-done-right`.

The remaining 12 gotchas (numbers match `references/gotchas.md`):

- #3 TextDecoder with `{ stream: true }` — prevents multi-byte UTF-8 corruption
- #4 `dontAsk` blocks reads outside project dir — use `bypassPermissions` for desktop
- #5 Extended thinking models — handle both `assistant` text blocks and `stream_event` deltas
- #6 RPC request timeout — return `taskId` immediately, before Claude finishes
- #7 ElectroBun version status — v1.15.1, check known platform issues
- #8 System webview differences — WKWebView/WebView2/WebKit2GTK behavior varies
- #9 Spawned Claude inherits user hooks — use `--setting-sources ""` to skip settings
- #10 Missing index.html — ElectroBun doesn't auto-generate it, create manually
- #11 stderr is a ReadableStream — read it once into a buffer, don't consume twice
- #14 ZIP strips executable permissions — always use DMG
- #15 No cross-compilation — build on the target platform
- #16 Process crash recovery — retry transient failures with backoff

> Read `references/gotchas.md` for full explanations, code samples, and fixes
> for all 16 issues.

## What to Generate

When you build the app, produce:

- [ ] `electrobun.config.ts` — App configuration (name, identifier, version, build options)
- [ ] `src/bun/index.ts` — Main process entry: startup CLI check, window creation, menu setup
- [ ] `src/bun/claude-manager.ts` — `resolveClaudePath()`, `cleanEnv()`, `createStreamParser()`, `spawnClaude()`, `abort()`, heartbeat
- [ ] `src/bun/rpc.ts` — RPC schema type definition (`LoomRPC`) and `BrowserView.defineRPC<LoomRPC>()` with handlers
- [ ] `src/mainview/index.html` — HTML shell (ElectroBun does NOT auto-generate this)
- [ ] `src/mainview/index.ts` — Webview entry point with `Electroview.defineRPC<LoomRPC>()` + `new Electroview({ rpc })` setup
- [ ] `src/mainview/App.tsx` — React UI (or vanilla HTML) with `handlers.messages` callbacks and task controls
- [ ] Startup CLI check — verify `claude --version` succeeds, show setup screen if missing
- [ ] Error handling for process crashes — `proc.exited` check, stderr capture, `error` RPC message

For simple apps, the Bun-side code fits in two files (`index.ts` + `claude-manager.ts`).
For complex apps, split RPC definitions, session management, and tray logic into
separate modules.

After generating, run `bunx electrobun dev` and verify the app launches. Then iterate
based on what the person sees.

**After the app is working**, ask the user if they'd like help building the
executable for distribution. If yes, run `bunx electrobun build --env=stable`
and explain:
- The build artifacts land in `artifacts/`
- Recipients need Claude CLI installed and authenticated on their machine
- Without an Apple Developer certificate, macOS Gatekeeper will block the app —
  the recipient can bypass with `xattr -cr /path/to/app.app` or right-click > Open
- For proper distribution without warnings, configure code signing and
  notarization in `electrobun.config.ts`

## The Possibility Space

The most interesting desktop Loom apps are not chat interfaces in a window.
They're tools that couldn't exist without an agentic runtime on the local
machine — applications where Claude has direct access to the filesystem, runs
as a native process, and streams its reasoning to a purpose-built interface.

A code review tool that you drag a project folder onto. A document processor
that watches a directory and summarizes new files. A research assistant that
reads your notes and finds connections. A local AI agent with a system tray
icon that's always available.

Help people think about what they actually want to make. The question isn't
"how do I put a chat box in a native window?" but "what would this tool look
like if there were an intelligence behind it — one that could read any file on
your machine and stream its thinking to a custom interface?"
