# RPC Schema Reference

The typed contract between the Bun process and webview in Loom desktop apps.

## Overview

ElectroBun's RPC system connects the Bun main process and the webview via a WebSocket with AES-256-GCM encryption. Communication is typed via TypeScript generics — you define a schema, and both sides get compile-time type safety.

Two kinds of communication:
- **Requests** — Call a function, get a response. Like an async function call across process boundaries.
- **Messages** — Fire-and-forget. Like an event emitter across process boundaries.

Two directions:
- **`bun`** — Handlers that run in the main process, callable from the webview.
- **`webview`** — Handlers that run in the webview, callable from the main process.

## The LoomRPC Schema

This is the typed contract for Loom desktop apps. The webview sends requests to the Bun process (start a task, abort). The Bun process sends messages to the webview (tokens, tool events, status updates).

```typescript
type LoomRPC = {
  bun: {
    requests: {
      startTask: {
        params: {
          prompt: string;
          model?: string;
          tools?: string[];
          maxBudget?: number;
          sessionId?: string;  // For conversational pattern
        };
        response: { taskId: string };
      };
      abort: {
        params: { taskId: string };
        response: { success: boolean };
      };
    };
    messages: {};
  };
  webview: {
    requests: {};
    messages: {
      // Streaming output
      token:      { taskId: string; text: string };
      toolUse:    { taskId: string; tool: string; input: any };
      toolResult: { taskId: string; tool: string; output: string; isError: boolean };
      done:       { taskId: string; cost: number; duration: number };
      error:      { taskId: string; message: string };

      // Liveness & progress
      status: {
        taskId: string;
        state: "spawning" | "running" | "thinking" | "tool_use" | "idle";
        detail?: string;          // e.g., "Reading src/index.ts"
        elapsedMs: number;
        lastActivityMs: number;
      };
    };
  };
};
```

### What each message carries

| Message | Purpose | Frequency |
|---------|---------|-----------|
| `token` | Text output from Claude | Many per task (one per text block or delta) |
| `toolUse` | Claude invoked a tool | One per tool invocation |
| `toolResult` | Tool returned output | One per tool completion |
| `done` | Task finished | Once per task |
| `error` | Process crashed or parse failed | Once per failure |
| `status` | Heartbeat with current state | Every 2 seconds while running |

## Bun-Side Usage

### Defining the RPC

In the main process, use `BrowserView.defineRPC<LoomRPC>()` to create the RPC instance and register request handlers:

```typescript
import { BrowserView, BrowserWindow } from "electrobun/bun";

const rpc = BrowserView.defineRPC<LoomRPC>({
  handlers: {
    startTask: async ({ prompt, model, tools, maxBudget, sessionId }) => {
      const taskId = crypto.randomUUID();
      // Spawn claude -p (see SKILL.md patterns)
      spawnClaude(taskId, prompt, { model, tools, maxBudget, sessionId });
      return { taskId };
    },
    abort: async ({ taskId }) => {
      const success = abortTask(taskId);
      return { success };
    },
  },
});
```

### Sending messages to the webview

From any Bun-side code that has the `rpc` reference:

```typescript
// Send a token
rpc.send.token({ taskId, text: "Hello from Claude" });

// Send tool use notification
rpc.send.toolUse({ taskId, tool: "Read", input: { file_path: "/src/index.ts" } });

// Send tool result
rpc.send.toolResult({
  taskId,
  tool: "Read",
  output: "file contents...",
  isError: false,
});

// Send status heartbeat
rpc.send.status({
  taskId,
  state: "running",
  detail: "Reading src/index.ts",
  elapsedMs: 5200,
  lastActivityMs: 200,
});

// Signal completion
rpc.send.done({ taskId, cost: 0.03, duration: 5200 });

// Signal error
rpc.send.error({ taskId, message: "Process exited with code 1" });
```

### Wiring RPC to a window

Pass the `rpc` to the BrowserWindow constructor:

```typescript
const win = new BrowserWindow({
  title: "My Claude App",
  frame: { width: 1200, height: 800 },
  url: "views://mainview/index.html",
  rpc,
});
```

## Webview-Side Usage

### Setting up the Electroview

In the webview entry point, create an `Electroview` instance with the same schema:

```typescript
import { Electroview } from "electrobun/view";

const electrobun = new Electroview<LoomRPC>();
```

### Sending requests to Bun

Requests return promises:

```typescript
// Start a task
const { taskId } = await electrobun.rpc.request.startTask({
  prompt: "Analyze all TypeScript files in src/",
  model: "sonnet",
  tools: ["Read", "Glob", "Grep"],
  maxBudget: 1,
});

// Abort a task
const { success } = await electrobun.rpc.request.abort({ taskId });
```

### Receiving messages from Bun

Register handlers for each message type:

```typescript
// Listen for streaming tokens
electrobun.rpc.on.token(({ taskId, text }) => {
  appendToOutput(taskId, text);
});

// Listen for tool usage
electrobun.rpc.on.toolUse(({ taskId, tool, input }) => {
  showToolIndicator(taskId, tool, input);
});

// Listen for tool results
electrobun.rpc.on.toolResult(({ taskId, tool, output, isError }) => {
  updateToolResult(taskId, tool, output, isError);
});

// Listen for completion
electrobun.rpc.on.done(({ taskId, cost, duration }) => {
  markComplete(taskId, cost, duration);
});

// Listen for errors
electrobun.rpc.on.error(({ taskId, message }) => {
  showError(taskId, message);
});

// Listen for status heartbeats
electrobun.rpc.on.status(({ taskId, state, detail, elapsedMs }) => {
  updateStatusBar(taskId, state, detail, elapsedMs);
});
```

### Minimal React component

```tsx
import { useState, useEffect } from "react";
import { Electroview } from "electrobun/view";

const electrobun = new Electroview<LoomRPC>();

function App() {
  const [output, setOutput] = useState("");
  const [status, setStatus] = useState<string>("idle");
  const [taskId, setTaskId] = useState<string | null>(null);

  useEffect(() => {
    electrobun.rpc.on.token(({ text }) => {
      setOutput((prev) => prev + text);
    });
    electrobun.rpc.on.status(({ state, detail }) => {
      setStatus(detail ? `${state}: ${detail}` : state);
    });
    electrobun.rpc.on.done(() => {
      setStatus("done");
    });
    electrobun.rpc.on.error(({ message }) => {
      setStatus(`error: ${message}`);
    });
  }, []);

  async function runTask() {
    setOutput("");
    setStatus("spawning");
    const { taskId } = await electrobun.rpc.request.startTask({
      prompt: "Analyze the codebase",
      tools: ["Read", "Glob", "Grep"],
    });
    setTaskId(taskId);
  }

  async function abort() {
    if (taskId) await electrobun.rpc.request.abort({ taskId });
  }

  return (
    <div>
      <div className="status-bar">{status}</div>
      <pre className="output">{output}</pre>
      <button onClick={runTask}>Run</button>
      <button onClick={abort} disabled={!taskId}>Abort</button>
    </div>
  );
}
```

## Message Flow

### Streaming pattern

```
Webview                        Bun                          claude -p
  │                             │                              │
  │─── startTask(prompt) ─────►│                              │
  │◄── { taskId } ─────────────│                              │
  │                             │── Bun.spawn("claude"...) ──►│
  │                             │                              │
  │◄── status(spawning) ───────│                              │
  │◄── status(running) ────────│◄── system init ──────────────│
  │◄── token(text) ────────────│◄── assistant text ───────────│
  │◄── token(text) ────────────│◄── stream_event delta ───────│
  │◄── status(tool_use) ───────│◄── tool_use block ───────────│
  │◄── toolUse(Read,...) ──────│                              │
  │◄── toolResult(Read,...) ───│◄── tool_result ──────────────│
  │◄── status(running) ────────│                              │
  │◄── token(text) ────────────│◄── assistant text ───────────│
  │◄── done(cost,duration) ────│◄── result ───────────────────│
  │                             │                              │
```

### Conversational pattern (multi-turn)

```
Webview                        Bun
  │                             │
  │─── startTask(prompt) ─────►│   // First turn, no sessionId
  │◄── { taskId } ─────────────│   // Bun generates sessionId internally
  │◄── ...tokens... ───────────│
  │◄── done ────────────────────│
  │                             │
  │─── startTask(prompt, ──────►│   // Follow-up, with sessionId
  │     sessionId) ─────────────│   // Bun spawns with --session-id <id> --continue
  │◄── { taskId } ─────────────│
  │◄── ...tokens... ───────────│
  │◄── done ────────────────────│
```

### Abort pattern

```
Webview                        Bun                          claude -p
  │                             │                              │
  │─── startTask(prompt) ─────►│── spawn ────────────────────►│
  │◄── { taskId } ─────────────│                              │
  │◄── ...tokens... ───────────│◄── ...stream events... ─────│
  │                             │                              │
  │─── abort(taskId) ──────────►│── SIGTERM ──────────────────►│
  │◄── { success: true } ──────│                              │✕
  │◄── error("aborted") ───────│   // Clean up heartbeat
```

## Extending the Schema

Add custom request/message types for app-specific needs:

```typescript
type MyAppRPC = {
  bun: {
    requests: {
      // Standard Loom requests
      startTask: { /* ... */ };
      abort: { /* ... */ };

      // App-specific requests
      openFile: {
        params: {};
        response: { path: string | null };
      };
      saveResult: {
        params: { content: string; filename: string };
        response: { saved: boolean; path: string };
      };
      getSettings: {
        params: {};
        response: { model: string; maxBudget: number; tools: string[] };
      };
    };
    messages: {};
  };
  webview: {
    requests: {};
    messages: {
      // Standard Loom messages
      token: { /* ... */ };
      toolUse: { /* ... */ };
      toolResult: { /* ... */ };
      done: { /* ... */ };
      error: { /* ... */ };
      status: { /* ... */ };

      // App-specific messages
      settingsChanged: { model: string; maxBudget: number };
      fileDropped: { path: string; size: number };
    };
  };
};
```

The schema is the API contract. Both sides get type errors if they send or receive the wrong shape.
