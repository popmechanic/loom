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

## Building It

Default to **TypeScript** for both the Bun process and the webview. Use
**React** for the UI unless the person prefers vanilla HTML.

### Shared Utilities

Every pattern below uses these helpers. Define them once in a shared module
(e.g., `src/bun/claude-manager.ts`).

**`cleanEnv()`** — Remove nesting guards so `claude -p` can start.

When you develop inside Claude Code (which is common), two environment
variables — `CLAUDECODE` and `CLAUDE_CODE_ENTRYPOINT` — tell Claude it's
already running and block nested processes. Remove exactly these two. Do NOT
filter all `CLAUDE*` vars — that kills auth tokens
(`CLAUDE_CODE_OAUTH_TOKEN`) and feature flags.

If running inside **cmux** (the Claude terminal app), `CMUX_*` surface
identifiers trigger nesting detection. These are terminal-state vars, not
auth — safe to remove.

```typescript
function cleanEnv(): NodeJS.ProcessEnv {
  const env = { ...process.env };
  delete env.CLAUDECODE;
  delete env.CLAUDE_CODE_ENTRYPOINT;
  // cmux terminal sets CMUX_* vars that trigger nesting detection.
  // These are terminal-state identifiers, not auth tokens — safe to remove.
  if (env.CMUX_SURFACE_ID) {
    delete env.CMUX_SURFACE_ID;
    delete env.CMUX_PANEL_ID;
    delete env.CMUX_TAB_ID;
    delete env.CMUX_WORKSPACE_ID;
    delete env.CMUX_SOCKET_PATH;
  }
  return env;
}
```

**`createStreamParser()`** — Buffer stdout chunks into complete JSON lines.

TCP delivers data in arbitrary chunks. A JSON line can split across two read
events. Without buffering, the first half fails `JSON.parse` and gets silently
discarded.

```typescript
function createStreamParser(onEvent: (event: any) => void) {
  const decoder = new TextDecoder();
  let buffer = "";

  return (chunk: Buffer | Uint8Array) => {
    buffer += decoder.decode(chunk, { stream: true });
    const lines = buffer.split("\n");
    buffer = lines.pop() || "";
    for (const line of lines) {
      if (!line.trim()) continue;
      try {
        onEvent(JSON.parse(line));
      } catch (err) {
        console.warn("[claude stdout] JSON parse error:", (err as Error).message, line.slice(0, 200));
      }
    }
  };
}
```

Use `TextDecoder` with `{ stream: true }` — not `chunk.toString()` — to handle
multi-byte UTF-8 characters that split across chunk boundaries.

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

### `deriveAndSendRPC()`

This function reads a parsed stream event and fires the appropriate RPC message.
It also tracks state for the heartbeat system.

```typescript
// State for heartbeat
let currentState: "spawning" | "running" | "thinking" | "tool_use" | "idle" = "spawning";
let lastToolName = "";
let lastOutputTime = Date.now();

function deriveAndSendRPC(
  taskId: string,
  event: any,
  rpc: { send: LoomRPC["webview"]["messages"] }
) {
  lastOutputTime = Date.now();

  switch (event.type) {
    case "system":
      currentState = "running";
      break;

    case "assistant": {
      const msg = event.message;
      if (!msg?.content) break;
      for (const block of msg.content) {
        if (block.type === "text") {
          rpc.send.token({ taskId, text: block.text });
          currentState = "running";
        } else if (block.type === "tool_use") {
          rpc.send.toolUse({ taskId, tool: block.name, input: block.input });
          currentState = "tool_use";
          lastToolName = block.name;
        }
      }
      break;
    }

    case "stream_event": {
      const delta = event.event?.delta;
      if (delta?.text) {
        rpc.send.token({ taskId, text: delta.text });
        currentState = "running";
      }
      if (delta?.type === "input_json_delta") {
        // Tool input streaming — ignore, wait for full assistant message
      }
      break;
    }

    case "tool_result": {
      const content = event.content ?? event.message?.content;
      const text = typeof content === "string"
        ? content
        : Array.isArray(content)
          ? content.map((b: any) => b.text ?? "").join("")
          : JSON.stringify(content);
      rpc.send.toolResult({
        taskId,
        tool: lastToolName,
        output: text.slice(0, 10000),
        isError: !!event.is_error,
      });
      currentState = "running";
      break;
    }

    case "result":
      rpc.send.done({
        taskId,
        cost: event.total_cost_usd ?? 0,
        duration: event.duration_ms ?? 0,
      });
      currentState = "idle";
      break;
  }
}
