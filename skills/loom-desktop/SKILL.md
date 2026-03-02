---
name: loom-desktop
description: >
  Build desktop applications where Claude Code CLI (`claude -p`) is the runtime,
  using ElectroBun. The Bun process spawns Claude processes that read files, run
  commands, and return structured output. A native desktop interface renders the
  results. Use when a user wants to build a desktop app powered by Claude — file
  analysis tools, code assistants, document processors, local AI agents with a
  native UI. Triggers: "build a desktop app that uses Claude", "native app with
  Claude", "ElectroBun + Claude", "desktop Claude tool", "local Claude app", or
  any desktop application needing Claude's agentic capabilities through a native
  interface. NOT for web apps (use loom), Anthropic API apps, or chat replicas.
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

ElectroBun is currently at v1.14.4 stable (v1.14.5-beta.0 in beta). It's
actively maintained but still evolving — check the issue tracker for
platform-specific concerns.

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
│  │               │◄─rpc.send      │               │  │
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
`@references/rpc-schema-reference.md` for the full schema.

## Project Setup

Scaffold an ElectroBun project:

```bash
npx electrobun init hello-world
cd myapp
bun install
electrobun dev
```

This gives you a running app with a Bun process and webview. From here, you'll
add the Claude Manager to the Bun side and the task UI to the webview.

For the full setup guide — project structure, configuration, dev workflow — see
`@references/electrobun-setup.md`.

For Claude CLI flags and output formats, see `@references/cli-runtime-reference.md`.

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
| Pure reasoning, no file access | `--tools ""` |
| Web research | `--tools "WebSearch,WebFetch"` |
| Structured data extraction | `--json-schema '{...}'` |
| Custom persona | `--system-prompt "You are..."` |

The tools list determines what Claude can do. Tighter is safer — only grant what
the app actually needs.

### 3. What's the interaction model?

How does the person's action translate to a Claude invocation?

| Interaction | Pattern |
|-------------|---------|
| Click a button, get a result | **Synchronous** — spawn, collect, return |
| Watch progress in real-time | **Streaming** — tokens flow as RPC messages |
| Multi-step conversation with context | **Conversational** — `--session-id` + `--continue` |
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

- **Budget**: `--max-budget-usd` caps spending per task. What's appropriate? $0.50 for quick analysis, $5 for deep codebase review.
- **Model**: Haiku for fast/cheap, Sonnet for balanced, Opus for best quality. `--fallback-model haiku` for reliability.
- **Permissions**: Use `--permission-mode dontAsk` with explicit `--tools` list. This auto-denies anything not whitelisted.
- **Turn limit**: `--max-turns` prevents runaway agent loops.
- **Filesystem scope**: Consider constraining Claude to specific directories via `--allowedTools "Read(/path/**)"` patterns.
