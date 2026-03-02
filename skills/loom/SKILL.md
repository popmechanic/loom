---
name: loom
description: >
  Build applications where Claude Code CLI (`claude -p`) is the runtime.
  A server spawns Claude processes that read files, run commands, and return
  structured output. A custom interface renders the results in whatever form
  makes sense. Use when a user wants to build something with Claude as the
  engine: any app that calls `claude -p` or uses the Agent SDK instead of a
  traditional API backend. Triggers: "build an app that uses Claude",
  "make something powered by Claude", "wrap claude -p", "Claude as backend",
  "Claude as runtime", or any application needing Claude's agentic capabilities
  (file access, tool use, streaming) through a purpose-built interface. NOT for
  Anthropic API apps, chat replicas, or standard web apps without an AI runtime.
---

# Loom: Applications on the Claude Code Runtime

You're helping someone build an application where Claude Code is the runtime —
not a helper writing React components, but the actual engine that powers
the application's intelligence. The interface talks to a server that spawns
`claude -p` processes or uses the Agent SDK, streaming results back to the browser.

This is a different posture than normal web development. Normally, the backend
is a database and some business logic. Here, the backend is Claude — an agent
that can read files, run commands, search code, and reason about complex tasks.
The interface's job is to give that agent a form — to decide what the output
looks like and how someone interacts with it.

## Why This Matters

Most "AI-powered" web apps just wrap a chat API. They put a text box on screen,
send messages to an LLM, and show the response. That's a chat replica.

Claude Code is not a chat API. It's an agentic runtime with filesystem access,
tool use, multi-turn sessions, structured output, and streaming. Building an
interface on top of Claude Code means designing something that exposes these
capabilities in ways that make sense — not just conversations, but whatever
interaction paradigm fits what the person is trying to do.

The question to help users explore is: **what would you build if your backend
could read files, run code, search the web, coordinate multiple agents, and
stream its reasoning to the browser in real time?**

## The Architecture

Every Loom app has the same basic shape:

```
┌──────────────┐     HTTP/WS      ┌──────────────┐    stdio/SDK    ┌──────────────┐
│   Interface  │ ◄──────────────► │  Node Server  │ ◄────────────► │  claude -p   │
│  (React/HTML)│                  │  (Express/etc) │                │  (Agent SDK) │
└──────────────┘                  └──────────────┘                └──────────────┘
     UI layer                      Bridge layer                    Runtime layer
```

**Interface**: The custom UI. Whatever makes sense for what's being built.

**Node Server**: The bridge. Receives requests from the interface, spawns Claude
processes, parses output, streams results back. This is where you handle auth,
rate limiting, session management, and the mapping between web concepts and
Claude invocations.

**Claude Runtime**: The intelligence. `claude -p` with the right flags, or the
Agent SDK for programmatic control. This is where the agentic work happens —
reading files, running commands, generating structured output.

### Communication Patterns

| Pattern | When to Use | How |
|---------|-------------|-----|
| **REST + JSON** | One-shot requests, data extraction | POST → `claude -p --output-format json` → JSON response |
| **SSE (Server-Sent Events)** | Streaming text to browser | `claude -p --output-format stream-json --verbose` → SSE stream |
| **WebSocket** | Bidirectional, multi-turn sessions | WS connection → `claude -p --input-format stream-json` |
| **Background job** | Long-running tasks | Queue → `claude -p` process → poll for result or push via WS |

For most apps, start with **REST + SSE**: REST for triggering tasks, SSE for
streaming progress and results. Add WebSockets only if you need true
bidirectional communication (e.g., the user can interrupt or steer while
Claude is working).

## The Conversation

When someone comes to you with an idea, walk through these design questions.
Don't dump them all at once — have a natural conversation. But cover this ground
before you start building:

### 1. What does the person see and do?

Get concrete about the interface, not the AI. What's the layout? What does
someone click? What appears when Claude is working? What does the final output
look like?

Ask: "Walk me through the screen. What does someone see when they open this?
What do they do first? What happens next?"

### 2. What's the interaction model?

How does the person's action translate to a Claude invocation?

| Interaction | Claude Pattern |
|-------------|----------------|
| Click a button, get a result | One-shot `claude -p`, REST response |
| Watch progress in real-time | Stream-JSON → SSE to browser |
| Multi-step workflow with state | Session-based (`--session-id` + `--continue`) |
| Concurrent analysis of multiple items | Parallel `claude -p` processes |
| Person steers while Claude works | Bidirectional streaming via WebSocket |

### 3. What does Claude actually do?

Map each action to what Claude needs behind the scenes:

- **What tools?** Read-only analysis (`Read,Glob,Grep`) vs. modification (`Write,Edit,Bash`)
  vs. no tools at all (`--tools ""` for pure reasoning)
- **What data?** Files on the server? User-uploaded content? Piped from other services?
- **What output shape?** Free text for display? Structured JSON for rendering UI components?
  Both (use `--json-schema` for structured + `result` for narrative)?

### 4. How should the output render?

This is where custom interfaces shine over CLIs. You can render Claude's structured
output as rich UI:

- JSON schema with typed arrays → render as cards, lists, timelines, visualizations
- Schema with nodes and edges → render as an interactive graph
- Schema with sections and status → render as a progress view
- Stream-JSON events → animate a progress indicator or live log

Design the JSON schema to match the UI components you want to render. The schema
IS your API contract between Claude and the frontend.

### 5. What are the safety boundaries?

Web apps add security concerns that CLIs don't have:

- **Auth**: Who can access this? Do you need user accounts?
- **Sandboxing**: Claude has filesystem access — what directory should it be scoped to?
- **Cost**: Each request costs money — do you need rate limiting? Budget caps (`--max-budget-usd`)?
- **Permissions**: Use the tightest `--permission-mode` and `--allowedTools` that work.
  Prefer `dontAsk` (auto-denies unallowed tools) over `bypassPermissions` (skips all checks).
  Only use `bypassPermissions` when `--allowedTools` fully constrains Claude's capabilities
  and you need unattended execution in a trusted environment (CI/CD, local dev tools).
- **Multi-tenancy**: If multiple users, each needs isolated sessions and working directories

### 6. Model and performance

| Need | Model | Why |
|------|-------|-----|
| Fast responses (<3s) | `haiku` | Classification, extraction, routing |
| Good quality, reasonable speed | `sonnet` | Default for most apps |
| Best reasoning | `opus` | Complex analysis, code generation |
| Reliability | `--fallback-model haiku` | Auto-fallback on overload |

For web UIs, perceived speed matters. Use streaming to show partial results
immediately, even when using slower models.

## Building It

Default to **Node.js/TypeScript** with **Express** for the server and plain
**HTML/CSS/JS** or **React** for the frontend, unless the person prefers otherwise.

### The Server Layer

The server's job is simple: receive HTTP requests, spawn Claude, return results.

Read `references/cli-runtime-reference.md` for the full `claude -p` flag reference.
Here are the server patterns to reach for:

#### Safety Defaults

Every pattern below runs Claude in a server — no human sitting at a terminal
to approve tool use or notice runaway costs. Three flags are non-negotiable:

**`--permission-mode dontAsk`** — In a server context, there's nobody to click
"approve." Without this flag, Claude hangs forever waiting for interactive
input. `dontAsk` auto-denies any tool not in `--allowedTools`, which is exactly
what you want: predictable, unattended execution.

**`--max-budget-usd`** — Every HTTP request that spawns Claude is an open
checkbook. Set a hard cap: `0.50` for quick analysis with haiku, `1` for
typical sonnet tasks, `3–5` for complex multi-turn sessions. Adjust to your
use case, but never omit it.

**`--max-turns`** — Defense in depth alongside the budget cap. Prevents
conversational loops where Claude keeps trying approaches that won't work.
`5` for one-shot tasks, `10–15` for streaming, `20` for multi-turn sessions.

Every pattern also handles three failure modes:

- **stderr** — Claude writes warnings, errors, and diagnostics here. Always
  capture it; it's your best debugging signal when something goes wrong.
- **Non-zero exit codes** — Model overloaded, budget exhausted, permission
  denied, timeout hit. `execFileSync` throws; `spawn` emits a `close` event.
- **Malformed output** — A killed or timed-out process may emit partial JSON.
  Always wrap JSON.parse in try/catch and check `parsed.is_error` before
  using `structured_output`.

#### Shared Utilities

Every pattern below uses these two helpers. Define them once at the top of
your server file.

**`cleanEnv()`** — Remove nesting guards so `claude -p` can start.

When your server runs inside Claude Code (which it often does during
development), two environment variables — `CLAUDECODE` and
`CLAUDE_CODE_ENTRYPOINT` — tell Claude it's already running and block nested
processes. Remove exactly these two. Do NOT filter all `CLAUDE*` vars — that
kills auth tokens (`CLAUDE_CODE_OAUTH_TOKEN`) and feature flags.

```typescript
function cleanEnv(): NodeJS.ProcessEnv {
  const env = { ...process.env };
  delete env.CLAUDECODE;
  delete env.CLAUDE_CODE_ENTRYPOINT;
  return env;
}
```

**`createStreamParser()`** — Buffer stdout chunks into complete JSON lines.

TCP delivers data in arbitrary chunks. A JSON line can split across two
`data` events. Without buffering, the first half fails `JSON.parse` and
gets silently discarded. Both Julian and vibes-skill use this identical
buffer-and-split pattern.

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

Use `TextDecoder` with `{ stream: true }` — not `chunk.toString()` — to
handle multi-byte UTF-8 characters that split across chunk boundaries.

#### Pattern: REST Endpoint (One-Shot)

Someone triggers an action → server calls Claude → returns JSON.

```typescript
import express from "express";
import { execFileSync } from "child_process";

const app = express();
app.use(express.json());

app.post("/api/analyze", (req, res) => {
  const { content, task } = req.body;

  const schema = JSON.stringify({
    type: "object",
    properties: {
      findings: { type: "array", items: {
        type: "object",
        properties: {
          title: { type: "string" },
          severity: { enum: ["info", "warning", "error"] },
          description: { type: "string" }
        }
      }},
      summary: { type: "string" }
    },
    required: ["findings", "summary"]
  });

  try {
    const result = execFileSync("claude", [
      "-p", "--model", "sonnet", "--output-format", "json",
      "--permission-mode", "dontAsk", "--max-budget-usd", "1",
      "--json-schema", schema, "--tools", "", "--no-session-persistence"
    ], { input: `${task}\n\n${content}`, encoding: "utf-8", timeout: 60000 });

    const parsed = JSON.parse(result);
    if (parsed.is_error) {
      return res.status(502).json({ error: parsed.result });
    }
    res.json({ ...parsed.structured_output, cost: parsed.total_cost_usd });
  } catch (e: any) {
    // execFileSync throws on non-zero exit and timeout
    const stderr = e.stderr?.toString().trim();
    console.error(`[claude] exit ${e.status}, stderr: ${stderr}`);
    res.status(502).json({ error: stderr || "Claude process failed" });
  }
});
```

#### Pattern: SSE Streaming

Someone triggers a task → server streams Claude's output token-by-token.

Use `app.post` for CSRF safety — task data stays in the body instead of the URL.
(`app.get` works for quick prototyping with `EventSource`, but `EventSource` only
supports GET, so switch to `fetch()` + `ReadableStream` on the client for POST.)

```typescript
app.post("/api/stream", (req, res) => {
  const { task } = req.body;

  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  const cleanEnv = Object.fromEntries(
    Object.entries(process.env).filter(([k]) => !k.startsWith("CLAUDE"))
  );

  const proc = spawn("claude", [
    "-p", "--output-format", "stream-json", "--verbose",
    "--permission-mode", "dontAsk", "--max-budget-usd", "2", "--max-turns", "15",
    "--model", "sonnet", "--no-session-persistence",
    task
  ], { env: cleanEnv });

  proc.stdout.on("data", (chunk) => {
    for (const line of chunk.toString().split("\n").filter(Boolean)) {
      try {
        const event = JSON.parse(line);
        if (event.event?.delta?.text) {
          res.write(`data: ${JSON.stringify({ type: "token", text: event.event.delta.text })}\n\n`);
        } else if (event.type === "result") {
          res.write(`data: ${JSON.stringify({ type: "done", cost: event.total_cost_usd })}\n\n`);
        }
      } catch (e) {
        // Skip malformed JSON lines — expected for partial stream chunks
      }
    }
  });

  proc.stderr.on("data", (chunk) => {
    console.error(`[claude stderr] ${chunk.toString().trim()}`);
  });

  proc.on("close", (code) => {
    if (code !== 0) {
      res.write(`data: ${JSON.stringify({ type: "error", message: `Claude exited with code ${code}` })}\n\n`);
    }
    res.end();
  });

  proc.on("error", (err) => {
    res.write(`data: ${JSON.stringify({ type: "error", message: err.message })}\n\n`);
    res.end();
  });

  req.on("close", () => proc.kill());
});
```

#### Pattern: WebSocket Session

Persistent connection with multi-turn conversation and streaming.

```typescript
import { WebSocketServer } from "ws";
import { spawn, ChildProcess } from "child_process";
import { v4 as uuidv4 } from "uuid";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws) => {
  const sessionId = uuidv4();
  let activeProc: ChildProcess | null = null;

  ws.on("message", (raw) => {
    const { action, payload } = JSON.parse(raw.toString());

    const cleanEnv = Object.fromEntries(
      Object.entries(process.env).filter(([k]) => !k.startsWith("CLAUDE"))
    );

    const proc = spawn("claude", [
      "-p", "--output-format", "stream-json", "--verbose",
      "--permission-mode", "dontAsk", "--max-budget-usd", "3", "--max-turns", "20",
      "--session-id", sessionId, "--continue",
      "--model", "sonnet",
      payload.prompt
    ], { env: cleanEnv });

    activeProc = proc;

    proc.stdout.on("data", (chunk) => {
      for (const line of chunk.toString().split("\n").filter(Boolean)) {
        try {
          const event = JSON.parse(line);
          if (event.event?.delta?.text) {
            ws.send(JSON.stringify({ type: "token", text: event.event.delta.text }));
          } else if (event.type === "result") {
            ws.send(JSON.stringify({ type: "done", result: event }));
          }
        } catch (e) {
          // Skip malformed JSON lines — expected for partial stream chunks
        }
      }
    });

    proc.stderr.on("data", (chunk) => {
      console.error(`[claude stderr] ${chunk.toString().trim()}`);
    });

    proc.on("close", (code) => {
      activeProc = null;
      if (code !== 0) {
        ws.send(JSON.stringify({ type: "error", message: `Claude exited with code ${code}` }));
      }
    });

    proc.on("error", (err) => {
      activeProc = null;
      ws.send(JSON.stringify({ type: "error", message: err.message }));
    });
  });

  ws.on("close", () => {
    if (activeProc) activeProc.kill();
  });
});
```

#### Pattern: Background Job with Progress

Long-running task that reports progress.

```typescript
const jobs = new Map<string, { status: string; result?: any; error?: string }>();

app.post("/api/jobs", (req, res) => {
  const jobId = uuidv4();
  jobs.set(jobId, { status: "running" });
  res.json({ jobId });

  const cleanEnv = Object.fromEntries(
    Object.entries(process.env).filter(([k]) => !k.startsWith("CLAUDE"))
  );

  const proc = spawn("claude", [
    "-p", "--output-format", "stream-json", "--verbose",
    "--model", "sonnet", "--max-turns", "20", "--max-budget-usd", "5",
    "--permission-mode", "dontAsk",
    "--tools", "Read,Bash,Glob,Grep",
    "--no-session-persistence"
  ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv });

  proc.stdin.write(req.body.task);
  proc.stdin.end();

  let stderrBuf = "";

  proc.stdout.on("data", (chunk) => {
    for (const line of chunk.toString().split("\n").filter(Boolean)) {
      try {
        const event = JSON.parse(line);
        if (event.type === "result") {
          jobs.set(jobId, { status: "complete", result: event });
        }
      } catch (e) {
        // Skip malformed JSON lines — expected for partial stream chunks
      }
    }
  });

  proc.stderr.on("data", (chunk) => {
    stderrBuf += chunk.toString();
  });

  proc.on("close", (code) => {
    if (code !== 0 && jobs.get(jobId)?.status === "running") {
      jobs.set(jobId, { status: "failed", error: stderrBuf.trim() || `Exit code ${code}` });
    }
  });

  proc.on("error", (err) => {
    jobs.set(jobId, { status: "failed", error: err.message });
  });
});

app.get("/api/jobs/:id", (req, res) => {
  const job = jobs.get(req.params.id);
  res.json(job || { status: "not_found" });
});
```

#### Pattern: Parallel Analysis

Analyze multiple items concurrently, aggregate results. Uses `spawn` so each
Claude process runs in its own child process — truly parallel.

```typescript
app.post("/api/batch", async (req, res) => {
  const { items, task } = req.body;
  const TIMEOUT_MS = 30000;

  const cleanEnv = Object.fromEntries(
    Object.entries(process.env).filter(([k]) => !k.startsWith("CLAUDE"))
  );

  const outcomes = await Promise.allSettled(
    items.map((item: string) => new Promise<any>((resolve, reject) => {
      const proc = spawn("claude", [
        "-p", "--model", "haiku", "--output-format", "json",
        "--permission-mode", "dontAsk", "--max-budget-usd", "0.50",
        "--json-schema", schema, "--tools", "", "--no-session-persistence"
      ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv });

      const timeout = setTimeout(() => { proc.kill(); reject(new Error("Timeout")); }, TIMEOUT_MS);
      let stdout = "";
      let stderr = "";

      proc.stdin.write(`${task}\n\n${item}`);
      proc.stdin.end();
      proc.stdout.on("data", (chunk) => { stdout += chunk.toString(); });
      proc.stderr.on("data", (chunk) => { stderr += chunk.toString(); });

      proc.on("close", (code) => {
        clearTimeout(timeout);
        try {
          const parsed = JSON.parse(stdout);
          if (parsed.is_error) reject(new Error(parsed.result));
          else resolve(parsed.structured_output);
        } catch (e) {
          reject(new Error(stderr.trim() || `Exit code ${code}`));
        }
      });

      proc.on("error", (err) => { clearTimeout(timeout); reject(err); });
    }))
  );

  const results = outcomes.map((o, i) =>
    o.status === "fulfilled"
      ? { item: items[i], data: o.value }
      : { item: items[i], error: o.reason.message }
  );

  res.json({ results });
});
```

### The Frontend Layer

The frontend renders Claude's output as purpose-built UI, not chat bubbles.

#### Streaming Text Display

`fetch()` + `ReadableStream` supports POST (CSRF-safe, keeps task data out of URLs).
Use this as the default pattern for production apps.

```javascript
async function streamTask(task) {
  const output = document.getElementById("output");

  const response = await fetch("/api/stream", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ task })
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split("\n\n");
    buffer = lines.pop(); // keep incomplete chunk
    for (const line of lines) {
      if (!line.startsWith("data: ")) continue;
      const data = JSON.parse(line.slice(6));
      if (data.type === "token") output.textContent += data.text;
      else if (data.type === "done") document.getElementById("cost").textContent = `$${data.cost.toFixed(4)}`;
      else if (data.type === "error") output.textContent += `\n[Error: ${data.message}]`;
    }
  }
}
```

For quick prototypes, `EventSource` is simpler but limited to GET:

```javascript
const source = new EventSource(`/api/stream?task=${encodeURIComponent(task)}`);
source.onmessage = (e) => {
  const data = JSON.parse(e.data);
  if (data.type === "token") output.textContent += data.text;
  else if (data.type === "done") source.close();
};
```

#### Structured Result Rendering

```javascript
// Claude returns structured data via JSON schema
// Render it as whatever UI makes sense — not a chat message
function esc(s) {
  const d = document.createElement('div');
  d.textContent = s || '';
  return d.innerHTML;
}

function renderResults(data) {
  return data.items.map(item => `
    <div class="item item--${esc(item.type)}">
      <h3>${esc(item.title)}</h3>
      <p>${esc(item.description)}</p>
    </div>
  `).join("");
}
```

### Error Handling

Every pattern above handles errors inline — you won't find a separate error
handling block to copy-paste because it doesn't belong in one. Here's the
mental model behind the three failure modes:

**stderr** fires first. Claude writes diagnostics, warnings, and model errors
here before the process exits. Always pipe it somewhere — `console.error` at
minimum. In production, send it to your logging stack. This is your primary
debugging signal when a request fails silently.

**Non-zero exit codes** mean Claude didn't complete successfully. Common
causes: model overloaded (503 from upstream), budget exhausted (`--max-budget-usd`
hit), permission denied (tool not in `--allowedTools`), or process killed
by your timeout. For `execFileSync`, this throws — catch it. For `spawn`,
listen on the `close` event and check the code.

**Malformed output** happens when a process is killed mid-stream (timeout,
client disconnect, OOM). The stdout buffer contains partial JSON that won't
parse. Always wrap `JSON.parse` in try/catch, and always check `parsed.is_error`
before reaching for `structured_output` — Claude sets this flag when it
couldn't complete the task (budget exhausted mid-run, tool failures, etc.).

### Input Validation

When a Loom app accepts file paths or user-provided prompts, validate at the
boundary — before anything reaches Claude.

For paths, reject directory traversal. A simple helper:

```typescript
import path from "path";

function safePath(base: string, userPath: string): string {
  const resolved = path.resolve(base, userPath);
  if (!resolved.startsWith(path.resolve(base) + path.sep)) {
    throw new Error("Path traversal blocked");
  }
  return resolved;
}
```

For prompts, the best defense is structural: use `--json-schema` to constrain
Claude's output shape and `--allowedTools` to limit what it can do. This
reduces the blast radius of prompt injection — even if the prompt is
adversarial, Claude can only produce data matching your schema and can only
use the tools you allowed.

### Temp Directory Cleanup

If your app writes temporary files for Claude to analyze (uploaded content,
generated artifacts), clean up after each request:

```typescript
import { mkdtempSync, rmSync } from "fs";
import { tmpdir } from "os";
import path from "path";

async function withTempDir<T>(fn: (dir: string) => Promise<T>): Promise<T> {
  const dir = mkdtempSync(path.join(tmpdir(), "loom-"));
  try {
    return await fn(dir);
  } finally {
    rmSync(dir, { recursive: true, force: true });
  }
}
```

For long-running servers, consider a periodic sweep of your temp directory
to catch orphaned files from crashed requests.

## What to Generate

When you build the app, produce:

1. **`server.ts`** — Express server with the appropriate endpoint pattern(s)
2. **`public/index.html`** — The frontend (inline styles/scripts for simplicity,
   or a small React app for complex UIs)
3. **`package.json`** — Dependencies and start script
4. **A one-liner to run it** — so the person can verify it works immediately

For simple apps, a single `server.ts` serving a static `index.html` is ideal.
For complex UIs, scaffold a React frontend with a separate server.

After generating, offer to start the server and open it in the browser together.
Then iterate based on what the person sees.

## The Possibility Space

The most interesting Loom apps are not chat interfaces in a browser.
They're things that couldn't exist without an agentic runtime — applications
where the backend can read, reason, and act on context that traditional APIs
can't access.

Help people think about what they actually want to make. The question isn't
"how do I put a chat box in a browser?" but "what would this look like if
there were an intelligence behind it?"
