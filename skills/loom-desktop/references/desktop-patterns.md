# Desktop Patterns Reference

Complete Bun-side and webview-side code patterns for Loom desktop apps. Read this when implementing a specific interaction pattern.

## Table of Contents

- [Shared Utilities](#shared-utilities)
  - [resolveClaudePath()](#resolveclaudepath)
  - [cleanEnv()](#cleanenv)
  - [createStreamParser()](#createstreamparser)
- [deriveAndSendRPC()](#deriveandsendrpc)
- [Pattern 1: Synchronous](#pattern-1-synchronous)
- [Pattern 2: Streaming](#pattern-2-streaming)
  - [Bun-side: Claude Manager (spawnClaude)](#bun-side-claude-manager-spawnclaude)
  - [Bun-side: Request Handlers](#bun-side-request-handlers)
  - [Webview-side: Receiving Messages](#webview-side-receiving-messages)
  - [Startup CLI Check](#startup-cli-check)
- [Pattern 3: Conversational](#pattern-3-conversational)
  - [Session Management](#session-management)
  - [Conversational Request Handler](#conversational-request-handler)
  - [Chat UI Component](#chat-ui-component)
- [Pattern 4: Background](#pattern-4-background)
  - [Task Registry](#task-registry)
  - [Task List Handler](#task-list-handler)
  - [System Tray Integration](#system-tray-integration)
  - [Completion Notification](#completion-notification)
  - [Task List Component](#task-list-component)

---

## Shared Utilities

Every pattern below uses these helpers. Define them once in a shared module
(e.g., `src/bun/claude-manager.ts`).

### resolveClaudePath()

Find the absolute path to `claude` at startup.

macOS GUI apps don't inherit the shell's PATH. Without this, the app works
in dev (launched from terminal) but fails when distributed and opened from
Finder. Always include this and use `CLAUDE_BIN` in all `Bun.spawn()` calls.

```typescript
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
      const file = Bun.file(p);
      if (file.size > 0) return p;
    } catch {}
  }

  return "claude"; // fallback (works if launched from terminal)
}

const CLAUDE_BIN = resolveClaudePath();
```

**Use `CLAUDE_BIN` everywhere.** Every `Bun.spawn()` and `Bun.spawnSync()` call
in your app uses `CLAUDE_BIN` instead of the string `"claude"`. The patterns
below all reference `CLAUDE_BIN`.

### cleanEnv()

Remove nesting guards so `claude -p` can start.

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

### createStreamParser()

Buffer stdout chunks into complete JSON lines.

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

---

## deriveAndSendRPC()

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
  rpc: { sendProxy: LoomRPC["webview"]["messages"] }
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
          rpc.sendProxy.token({ taskId, text: block.text });
          currentState = "running";
        } else if (block.type === "tool_use") {
          rpc.sendProxy.toolUse({ taskId, tool: block.name, input: block.input });
          currentState = "tool_use";
          lastToolName = block.name;
        }
      }
      break;
    }

    case "stream_event": {
      const delta = event.event?.delta;
      if (delta?.text) {
        rpc.sendProxy.token({ taskId, text: delta.text });
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
      rpc.sendProxy.toolResult({
        taskId,
        tool: lastToolName,
        output: text.slice(0, 10000),
        isError: !!event.is_error,
      });
      currentState = "running";
      break;
    }

    case "result":
      rpc.sendProxy.done({
        taskId,
        cost: event.total_cost_usd ?? 0,
        duration: event.duration_ms ?? 0,
      });
      currentState = "idle";
      break;
  }
}
```

---

## Pattern 1: Synchronous

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
    requests: {
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
            CLAUDE_BIN, "-p",
            "--output-format", "json",
            "--permission-mode", "bypassPermissions",
            "--setting-sources", "",
            "--model", model || "sonnet",
            "--max-budget-usd", String(maxBudget || 1),
            "--json-schema", schema,
            "--no-session-persistence",
            prompt,
          ], { env: cleanEnv(), timeout: 60000 });

          if (proc.exitCode !== 0) {
            const stderr = proc.stderr.toString().trim();
            rpc.sendProxy.error({ taskId, message: stderr || `Exit code ${proc.exitCode}` });
            return { taskId };
          }

          const parsed = JSON.parse(proc.stdout.toString());
          if (parsed.is_error) {
            rpc.sendProxy.error({ taskId, message: parsed.result });
          } else {
            rpc.sendProxy.token({ taskId, text: parsed.result });
            rpc.sendProxy.done({
              taskId,
              cost: parsed.total_cost_usd ?? 0,
              duration: parsed.duration_ms ?? 0,
            });
          }
        } catch (err: any) {
          rpc.sendProxy.error({ taskId, message: err.message });
        }

        return { taskId };
      },
      abort: async () => ({ success: false }), // Can't abort sync
    },
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

---

## Pattern 2: Streaming

The Thin Bridge. This is the core pattern for most Loom desktop apps. Tokens
flow from Claude's stdout through the Bun process to the webview as typed RPC
messages, with status heartbeats every 2 seconds.

### Bun-side: Claude Manager (spawnClaude)

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
    isFirstTurn?: boolean;
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
    "--permission-mode", "bypassPermissions",
    "--setting-sources", "",
    "--model", opts.model || "sonnet",
    "--max-budget-usd", String(opts.maxBudget || 1),
    "--no-session-persistence",
  ];
  // Add tools if specified (omitting --tools entirely = all tools available)
  if (opts.tools && opts.tools.length > 0) {
    args.push("--tools", opts.tools.join(","));
  }
  if (opts.sessionId) {
    if (opts.isFirstTurn) {
      args.push("--session-id", opts.sessionId);
    } else {
      args.push("--resume", opts.sessionId);
    }
  }
  args.push(prompt);

  // Spawn
  const proc = Bun.spawn([CLAUDE_BIN, ...args], {
    stdout: "pipe",
    stderr: "pipe",
    env: cleanEnv(),
  });

  // Status heartbeat — every 2 seconds while the task runs
  const heartbeat = setInterval(() => {
    rpc.sendProxy.status({
      taskId,
      state: currentState,
      detail: currentState === "tool_use" ? lastToolName : undefined,
      elapsedMs: Date.now() - startTime,
      lastActivityMs: Date.now() - lastOutputTime,
    });
  }, 2000);

  // Track for abort
  activeTasks.set(taskId, { proc, heartbeat });

  // Collect stderr proactively — ReadableStream can only be consumed once.
  // If you read proc.stderr in an error handler after already consuming it
  // elsewhere, you'll get "ReadableStream already used".
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
            rpc.sendProxy.token({ taskId, text: block.text });
            currentState = "running";
          } else if (block.type === "tool_use") {
            rpc.sendProxy.toolUse({ taskId, tool: block.name, input: block.input });
            currentState = "tool_use";
            lastToolName = block.name;
          }
        }
        break;
      }

      case "stream_event": {
        const delta = event.event?.delta;
        if (delta?.text) {
          rpc.sendProxy.token({ taskId, text: delta.text });
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
        rpc.sendProxy.toolResult({
          taskId,
          tool: lastToolName,
          output: text.slice(0, 10000),
          isError: !!event.is_error,
        });
        currentState = "running";
        break;
      }

      case "result":
        rpc.sendProxy.done({
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
      const stderr = stderrChunks.join("");
      rpc.sendProxy.error({
        taskId,
        message: stderr.trim() || `Claude exited with code ${exitCode}`,
      });
    }
    cleanup();
  })();

  return proc;
}
```

### Bun-side: Request Handlers

Wire the RPC to `spawnClaude`:

```typescript
const rpc = BrowserView.defineRPC<LoomRPC>({
  handlers: {
    requests: {
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
        rpc.sendProxy.error({ taskId, message: "Task aborted by user" });
        return { success: true };
      },
    },
  },
});
```

### Webview-side: Receiving Messages

The frontend defines message handlers via `Electroview.defineRPC()` — there
is no `electrobun.rpc.on.*` event API. Message handlers are declared upfront
and connected to React state via module-level callback refs:

```tsx
import { useState, useEffect, useRef } from "react";
import { Electroview } from "electrobun/view";

// Module-level callback refs — set by the React component on mount.
// defineRPC handlers fire these to bridge into React state.
const callbacks = {
  onToken: null as ((data: { taskId: string; text: string }) => void) | null,
  onToolUse: null as ((data: { taskId: string; tool: string; input: any }) => void) | null,
  onToolResult: null as ((data: { taskId: string; tool: string; output: string; isError: boolean }) => void) | null,
  onStatus: null as ((data: { taskId: string; state: string; detail?: string; elapsedMs: number }) => void) | null,
  onDone: null as ((data: { taskId: string; cost: number; duration: number }) => void) | null,
  onError: null as ((data: { taskId: string; message: string }) => void) | null,
};

// Define RPC with message handlers — this is how the webview receives
// messages from the Bun process. NOT electrobun.rpc.on.* (that doesn't exist).
const rpc = Electroview.defineRPC<LoomRPC>({
  handlers: {
    messages: {
      token: (data) => callbacks.onToken?.(data),
      toolUse: (data) => callbacks.onToolUse?.(data),
      toolResult: (data) => callbacks.onToolResult?.(data),
      status: (data) => callbacks.onStatus?.(data),
      done: (data) => callbacks.onDone?.(data),
      error: (data) => callbacks.onError?.(data),
    },
  },
});

const electrobun = new Electroview({ rpc });

function App() {
  const [output, setOutput] = useState("");
  const [status, setStatus] = useState<string>("idle");
  const [taskId, setTaskId] = useState<string | null>(null);
  const [cost, setCost] = useState<number | null>(null);
  const [tools, setTools] = useState<string[]>([]);
  const outputRef = useRef<HTMLPreElement>(null);

  useEffect(() => {
    callbacks.onToken = ({ text }) => setOutput((prev) => prev + text);
    callbacks.onToolUse = ({ tool }) => setTools((prev) => [...prev, tool]);
    callbacks.onToolResult = ({ tool }) => setTools((prev) => prev.filter((t) => t !== tool));
    callbacks.onStatus = ({ state, detail, elapsedMs }) => {
      const secs = Math.round(elapsedMs / 1000);
      setStatus(detail ? `${state}: ${detail} (${secs}s)` : `${state} (${secs}s)`);
    };
    callbacks.onDone = ({ cost: c, duration }) => {
      setCost(c);
      setStatus(`Done in ${Math.round(duration / 1000)}s`);
      setTaskId(null);
    };
    callbacks.onError = ({ message }) => {
      setStatus(`Error: ${message}`);
      setTaskId(null);
    };
    return () => {
      callbacks.onToken = null;
      callbacks.onToolUse = null;
      callbacks.onToolResult = null;
      callbacks.onStatus = null;
      callbacks.onDone = null;
      callbacks.onError = null;
    };
  }, []);

  useEffect(() => {
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

### Startup CLI Check

On app launch, verify Claude is accessible before showing the main UI:

```typescript
// In src/bun/index.ts, before creating the window:
function checkClaudeCLI(): boolean {
  try {
    const result = Bun.spawnSync([CLAUDE_BIN, "--version"]);
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

---

## Pattern 3: Conversational

Multi-turn sessions where Claude retains context across messages. Uses
`--session-id` (first turn) and `--resume` (subsequent turns) flags. The Bun
process maintains a session map.

Use this for: code assistants where you ask follow-up questions, interactive
debugging sessions, multi-step analysis that builds on previous answers.

### Session Management

```typescript
// Session registry
const sessions = new Map<string, {
  sessionId: string;
  turns: number;
  lastActivity: number;
}>();

// Clean up stale sessions periodically (optional)
setInterval(() => {
  const staleMs = 30 * 60 * 1000; // 30 minutes
  const now = Date.now();
  for (const [key, session] of sessions) {
    if (now - session.lastActivity > staleMs) {
      sessions.delete(key);
    }
  }
}, 60000);
```

### Conversational Request Handler

Replace the `startTask` handler with session awareness:

```typescript
const rpc = BrowserView.defineRPC<LoomRPC>({
  handlers: {
    requests: {
      startTask: async ({ prompt, model, tools, maxBudget, sessionId }) => {
        const taskId = crypto.randomUUID();

        let actualSessionId: string;
        if (sessionId && sessions.has(sessionId)) {
          // Follow-up turn — reuse session
          actualSessionId = sessionId;
          const session = sessions.get(sessionId)!;
          session.turns++;
          session.lastActivity = Date.now();
        } else {
          // First turn — create new session
          actualSessionId = crypto.randomUUID();
          sessions.set(actualSessionId, {
            sessionId: actualSessionId,
            turns: 1,
            lastActivity: Date.now(),
          });
        }

        const isFirstTurn = !sessionId || !sessions.has(sessionId);
        spawnClaude(taskId, prompt, {
          model, tools, maxBudget,
          sessionId: actualSessionId,
          isFirstTurn,
        }, rpc);

        return { taskId };
      },
      abort: async ({ taskId }) => {
        const task = activeTasks.get(taskId);
        if (!task) return { success: false };
        task.proc.kill("SIGTERM");
        clearInterval(task.heartbeat);
        activeTasks.delete(taskId);
        rpc.sendProxy.error({ taskId, message: "Task aborted by user" });
        return { success: true };
      },
    },
  },
});
```

Note: `spawnClaude` from Pattern 2 already handles session flags: it uses
`--session-id <id>` on the first turn and `--resume <id>` on follow-up
turns. The `isFirstTurn` flag controls which flag is used.

### Chat UI Component

A minimal chat component that maintains conversation context:

```tsx
import { useState, useEffect, useRef } from "react";
import { Electroview } from "electrobun/view";

type Message = {
  role: "user" | "assistant";
  content: string;
  taskId?: string;
};

// Module-level callback refs for message handlers
const chatCallbacks = {
  onToken: null as ((data: { taskId: string; text: string }) => void) | null,
  onDone: null as (() => void) | null,
  onError: null as ((data: { message: string }) => void) | null,
};

const rpc = Electroview.defineRPC<LoomRPC>({
  handlers: {
    messages: {
      token: (data) => chatCallbacks.onToken?.(data),
      toolUse: () => {},
      toolResult: () => {},
      status: () => {},
      done: () => chatCallbacks.onDone?.(),
      error: (data) => chatCallbacks.onError?.(data),
    },
  },
});

const electrobun = new Electroview({ rpc });

function ChatApp() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState("");
  const [sessionId, setSessionId] = useState<string | null>(null);
  const [streaming, setStreaming] = useState(false);
  const messagesEnd = useRef<HTMLDivElement>(null);

  useEffect(() => {
    chatCallbacks.onToken = ({ taskId, text }) => {
      setMessages((prev) => {
        const last = prev[prev.length - 1];
        if (last?.taskId === taskId && last.role === "assistant") {
          return [...prev.slice(0, -1), { ...last, content: last.content + text }];
        }
        return [...prev, { role: "assistant", content: text, taskId }];
      });
    };
    chatCallbacks.onDone = () => setStreaming(false);
    chatCallbacks.onError = ({ message }) => {
      setStreaming(false);
      setMessages((prev) => [...prev, { role: "assistant", content: `Error: ${message}` }]);
    };
    return () => {
      chatCallbacks.onToken = null;
      chatCallbacks.onDone = null;
      chatCallbacks.onError = null;
    };
  }, []);

  useEffect(() => {
    messagesEnd.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  async function send() {
    if (!input.trim() || streaming) return;
    const prompt = input.trim();
    setInput("");
    setStreaming(true);

    setMessages((prev) => [...prev, { role: "user", content: prompt }]);

    const { taskId } = await electrobun.rpc.request.startTask({
      prompt,
      model: "sonnet",
      tools: ["Read", "Glob", "Grep", "Bash"],
      maxBudget: 2,
      sessionId: sessionId ?? undefined,
    });

    // After first turn, store session ID for follow-ups
    if (!sessionId) setSessionId(taskId); // Use taskId as proxy, or track via RPC
  }

  return (
    <div className="chat">
      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i} className={`message ${msg.role}`}>
            <pre>{msg.content}</pre>
          </div>
        ))}
        <div ref={messagesEnd} />
      </div>
      <div className="input-row">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && send()}
          placeholder="Ask a follow-up question..."
          disabled={streaming}
        />
        <button onClick={send} disabled={streaming}>Send</button>
      </div>
    </div>
  );
}
```

**Don't do this:**
- Don't spawn concurrent Claude processes on the same session — one at a time.
  `--resume` resumes a specific session; parallel spawns create race conditions.
- Don't forget `--resume` on follow-up turns. Without it, Claude starts a
  fresh conversation. Use `--session-id` only on the first turn.

---

## Pattern 4: Background

For tasks that take minutes — analyzing an entire codebase, batch-processing
documents, comprehensive code review. The user can navigate away or minimize
to the system tray. Status messages continue flowing.

### Task Registry

```typescript
type TaskRecord = {
  proc: ReturnType<typeof Bun.spawn>;
  heartbeat: ReturnType<typeof setInterval>;
  status: "running" | "completed" | "error";
  result?: { cost: number; duration: number };
  error?: string;
  startTime: number;
};

const taskRegistry = new Map<string, TaskRecord>();
```

Spawning is identical to Pattern 2 — the difference is lifecycle management.
Background tasks survive webview navigation and can be listed, polled, or
cancelled from any view.

### Task List Handler

Add a request to list active tasks (extend the RPC schema):

```typescript
// Add to your RPC schema's bun.requests:
listTasks: {
  params: {};
  response: {
    tasks: Array<{
      taskId: string;
      status: "running" | "completed" | "error";
      elapsedMs: number;
    }>;
  };
};
```

```typescript
// In the handler:
listTasks: async () => {
  const tasks = Array.from(taskRegistry.entries()).map(([taskId, record]) => ({
    taskId,
    status: record.status,
    elapsedMs: Date.now() - record.startTime,
  }));
  return { tasks };
},
```

### System Tray Integration

Minimize to the system tray during long tasks. Show progress in the tooltip:

```typescript
import { Tray } from "electrobun/bun";

const tray = new Tray({
  title: "Claude",
  image: "views://assets/tray-icon.png",
  width: 22,
  height: 22,
});

tray.setMenu([
  { label: "Show Window", action: "show" },
  { type: "separator" },
  { label: "No active tasks", action: "status", enabled: false },
  { type: "separator" },
  { label: "Quit", role: "quit" },
]);

// Listen for tray clicks
tray.on("tray-clicked", () => {
  win.focus();
});

tray.on("tray-item-clicked", (e) => {
  if (e.data.action === "show") win.focus();
});

// Update tray when tasks change
function updateTrayStatus() {
  const running = Array.from(taskRegistry.values())
    .filter((t) => t.status === "running");

  if (running.length === 0) {
    tray.setMenu([
      { label: "Show Window", action: "show" },
      { type: "separator" },
      { label: "No active tasks", action: "status", enabled: false },
      { type: "separator" },
      { label: "Quit", role: "quit" },
    ]);
  } else {
    const items = running.map((t, i) => ({
      label: `Task ${i + 1}: ${Math.round((Date.now() - t.startTime) / 1000)}s`,
      action: `task-${i}`,
      enabled: false,
    }));
    tray.setMenu([
      { label: "Show Window", action: "show" },
      { type: "separator" },
      ...items,
      { type: "separator" },
      { label: "Quit", role: "quit" },
    ]);
  }
}
```

### Completion Notification

Send a native notification when a background task finishes:

```typescript
import { Utils } from "electrobun/bun";

// In the "result" handler of spawnClaude, after cleanup:
Utils.showNotification({
  title: "Task Complete",
  body: `Finished in ${Math.round(duration / 1000)}s — $${cost.toFixed(4)}`,
});
updateTrayStatus();
```

### Task List Component

```tsx
function TaskList() {
  const [tasks, setTasks] = useState<Array<{
    taskId: string;
    status: string;
    elapsedMs: number;
  }>>([]);

  async function refresh() {
    const { tasks } = await electrobun.rpc.request.listTasks({});
    setTasks(tasks);
  }

  useEffect(() => {
    refresh();
    const interval = setInterval(refresh, 3000);
    return () => clearInterval(interval);
  }, []);

  return (
    <div className="task-list">
      <h3>Background Tasks</h3>
      {tasks.length === 0 && <p>No active tasks</p>}
      {tasks.map((task) => (
        <div key={task.taskId} className={`task ${task.status}`}>
          <span className="task-id">{task.taskId.slice(0, 8)}</span>
          <span className="task-status">{task.status}</span>
          <span className="task-time">{Math.round(task.elapsedMs / 1000)}s</span>
          {task.status === "running" && (
            <button onClick={() => electrobun.rpc.request.abort({ taskId: task.taskId })}>
              Cancel
            </button>
          )}
        </div>
      ))}
    </div>
  );
}
```
