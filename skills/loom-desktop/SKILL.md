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
```

## Patterns

### Pattern 1: Synchronous (Quick Extraction)

The simplest pattern. The webview sends a `startTask` request, Bun spawns
Claude, collects the full output, and returns the result. No streaming — the
webview waits for the response.

Use this for fast, structured extraction: pull metadata from a file, classify
a document, extract TODOs from a codebase.

**Bun-side handler:**

```typescript
import { BrowserView } from "electrobun/bun";

const rpc = BrowserView.defineRPC<LoomRPC>({
  handlers: {
    startTask: async ({ prompt, model, tools, maxBudget }) => {
      const taskId = crypto.randomUUID();

      const schema = JSON.stringify({
        type: "object",
        properties: {
          result: { type: "string" },
          items: { type: "array", items: { type: "object" } },
        },
        required: ["result"],
      });

      try {
        const proc = Bun.spawnSync([
          "claude", "-p",
          "--output-format", "json",
          "--permission-mode", "dontAsk",
          "--tools", (tools || []).join(","),
          "--model", model || "sonnet",
          "--max-budget-usd", String(maxBudget || 1),
          "--json-schema", schema,
          "--no-session-persistence",
          prompt,
        ], { env: cleanEnv(), timeout: 60000 });

        if (proc.exitCode !== 0) {
          const stderr = proc.stderr.toString().trim();
          rpc.send.error({ taskId, message: stderr || `Exit code ${proc.exitCode}` });
          return { taskId };
        }

        const parsed = JSON.parse(proc.stdout.toString());
        if (parsed.is_error) {
          rpc.send.error({ taskId, message: parsed.result });
        } else {
          rpc.send.token({ taskId, text: parsed.result });
          rpc.send.done({
            taskId,
            cost: parsed.total_cost_usd ?? 0,
            duration: parsed.duration_ms ?? 0,
          });
        }
      } catch (err: any) {
        rpc.send.error({ taskId, message: err.message });
      }

      return { taskId };
    },
    abort: async () => ({ success: false }), // Can't abort sync
  },
});
```

The `--json-schema` flag gives you typed `structured_output` alongside the
narrative `result`. Design the schema to match the UI components you want to
render — the schema IS your API contract between Claude and the frontend.

**Don't do this:**
- Don't use string interpolation to build the command — use an args array.
  Shell strings break on special characters and are injection-vulnerable.
- Don't read `structured_output` without checking `is_error` first — when
  Claude hits a budget cap or tool failure, `structured_output` is `null`.

### Pattern 2: Streaming (Primary Pattern)

The Thin Bridge. This is the core pattern for most Loom desktop apps. Tokens
flow from Claude's stdout through the Bun process to the webview as typed RPC
messages, with status heartbeats every 2 seconds.

**Bun-side: Claude Manager**

The complete `spawnClaude()` function. This is the heart of the bridge:

```typescript
import { BrowserView } from "electrobun/bun";

// Active tasks for abort support
const activeTasks = new Map<string, {
  proc: ReturnType<typeof Bun.spawn>;
  heartbeat: ReturnType<typeof setInterval>;
}>();

function spawnClaude(
  taskId: string,
  prompt: string,
  opts: {
    model?: string;
    tools?: string[];
    maxBudget?: number;
    sessionId?: string;
  },
  rpc: any
) {
  // Heartbeat state
  let currentState: LoomRPC["webview"]["messages"]["status"]["state"] = "spawning";
  let lastToolName = "";
  let lastOutputTime = Date.now();
  const startTime = Date.now();

  // Build args
  const args = [
    "-p",
    "--output-format", "stream-json", "--verbose",
    "--permission-mode", "dontAsk",
    "--tools", (opts.tools || []).join(","),
    "--model", opts.model || "sonnet",
    "--max-budget-usd", String(opts.maxBudget || 1),
    "--no-session-persistence",
  ];
  if (opts.sessionId) {
    args.push("--session-id", opts.sessionId, "--continue");
  }
  args.push(prompt);

  // Spawn
  const proc = Bun.spawn(["claude", ...args], {
    stdout: "pipe",
    stderr: "pipe",
    env: cleanEnv(),
  });

  // Status heartbeat — every 2 seconds while the task runs
  const heartbeat = setInterval(() => {
    rpc.send.status({
      taskId,
      state: currentState,
      detail: currentState === "tool_use" ? lastToolName : undefined,
      elapsedMs: Date.now() - startTime,
      lastActivityMs: Date.now() - lastOutputTime,
    });
  }, 2000);

  // Track for abort
  activeTasks.set(taskId, { proc, heartbeat });

  // Parse stdout stream
  const parse = createStreamParser((event) => {
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
        cleanup();
        break;
    }
  });

  function cleanup() {
    clearInterval(heartbeat);
    activeTasks.delete(taskId);
    currentState = "idle";
  }

  // Pump stdout reader
  (async () => {
    const reader = proc.stdout.getReader();
    try {
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        parse(value);
      }
    } catch (err) {
      // Reader closed — process exited
    }

    // Process finished — check for errors
    const exitCode = await proc.exited;
    if (exitCode !== 0 && currentState !== "idle") {
      const stderr = await new Response(proc.stderr).text();
      rpc.send.error({
        taskId,
        message: stderr.trim() || `Claude exited with code ${exitCode}`,
      });
    }
    cleanup();
  })();

  return proc;
}
```

**Bun-side: Request handlers**

Wire the RPC to `spawnClaude`:

```typescript
const rpc = BrowserView.defineRPC<LoomRPC>({
  handlers: {
    startTask: async ({ prompt, model, tools, maxBudget, sessionId }) => {
      const taskId = crypto.randomUUID();
      spawnClaude(taskId, prompt, { model, tools, maxBudget, sessionId }, rpc);
      return { taskId };
    },
    abort: async ({ taskId }) => {
      const task = activeTasks.get(taskId);
      if (!task) return { success: false };

      task.proc.kill("SIGTERM");
      clearInterval(task.heartbeat);
      activeTasks.delete(taskId);
      rpc.send.error({ taskId, message: "Task aborted by user" });
      return { success: true };
    },
  },
});
```

**Webview-side: Receiving messages**

The frontend registers handlers for each message type. Here's a minimal React
component that renders streaming tokens with status:

```tsx
import { useState, useEffect, useRef } from "react";
import { Electroview } from "electrobun/view";

const electrobun = new Electroview<LoomRPC>();

function App() {
  const [output, setOutput] = useState("");
  const [status, setStatus] = useState<string>("idle");
  const [taskId, setTaskId] = useState<string | null>(null);
  const [cost, setCost] = useState<number | null>(null);
  const [tools, setTools] = useState<string[]>([]);
  const outputRef = useRef<HTMLPreElement>(null);

  useEffect(() => {
    electrobun.rpc.on.token(({ taskId: tid, text }) => {
      setOutput((prev) => prev + text);
    });

    electrobun.rpc.on.toolUse(({ tool }) => {
      setTools((prev) => [...prev, tool]);
    });

    electrobun.rpc.on.toolResult(({ tool, isError }) => {
      setTools((prev) => prev.filter((t) => t !== tool));
    });

    electrobun.rpc.on.status(({ state, detail, elapsedMs }) => {
      const secs = Math.round(elapsedMs / 1000);
      setStatus(detail ? `${state}: ${detail} (${secs}s)` : `${state} (${secs}s)`);
    });

    electrobun.rpc.on.done(({ cost: c, duration }) => {
      setCost(c);
      setStatus(`Done in ${Math.round(duration / 1000)}s`);
      setTaskId(null);
    });

    electrobun.rpc.on.error(({ message }) => {
      setStatus(`Error: ${message}`);
      setTaskId(null);
    });
  }, []);

  useEffect(() => {
    // Auto-scroll output
    if (outputRef.current) {
      outputRef.current.scrollTop = outputRef.current.scrollHeight;
    }
  }, [output]);

  async function runTask(prompt: string) {
    setOutput("");
    setCost(null);
    setStatus("spawning...");
    const { taskId: tid } = await electrobun.rpc.request.startTask({
      prompt,
      model: "sonnet",
      tools: ["Read", "Glob", "Grep"],
      maxBudget: 1,
    });
    setTaskId(tid);
  }

  async function abort() {
    if (taskId) {
      await electrobun.rpc.request.abort({ taskId });
    }
  }

  return (
    <div className="app">
      <header>
        <input
          type="text"
          placeholder="What should Claude do?"
          onKeyDown={(e) => {
            if (e.key === "Enter") runTask(e.currentTarget.value);
          }}
          disabled={!!taskId}
        />
        <button onClick={abort} disabled={!taskId}>Abort</button>
      </header>

      <div className="status-bar">
        {status}
        {tools.length > 0 && <span className="tools">Using: {tools.join(", ")}</span>}
        {cost !== null && <span className="cost">${cost.toFixed(4)}</span>}
      </div>

      <pre className="output" ref={outputRef}>{output}</pre>
    </div>
  );
}

export default App;
```

**Startup CLI check**

On app launch, verify Claude is accessible before showing the main UI:

```typescript
// In src/bun/index.ts, before creating the window:
function checkClaudeCLI(): boolean {
  try {
    const result = Bun.spawnSync(["claude", "--version"]);
    return result.exitCode === 0;
  } catch {
    return false;
  }
}

const claudeAvailable = checkClaudeCLI();

const win = new BrowserWindow({
  title: "My Claude App",
  frame: { width: 1200, height: 800 },
  url: claudeAvailable
    ? "views://mainview/index.html"
    : "views://mainview/setup.html",  // Show install instructions
  rpc,
});
```

If Claude isn't found, show a setup screen with install instructions. Don't
silently fail — the user needs to know why the app isn't working.
