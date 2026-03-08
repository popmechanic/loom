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
processes, parses output, streams results back. This is where you handle
Anthropic OAuth authentication, rate limiting, session management, and the
mapping between web concepts and Claude invocations. The server verifies
credentials exist before spawning Claude and shows a setup screen if they're
missing (see `references/oauth-reference.md`).

**Claude Runtime**: The intelligence. `claude -p` with the right flags, or the
Agent SDK for programmatic control. This is where the agentic work happens —
reading files, running commands, generating structured output. Before the
runtime can start, the user needs valid Anthropic credentials. See the OAuth
setup in the Building It section below.

### Communication Patterns

| Pattern | When to Use | How |
|---------|-------------|-----|
| **REST + JSON** | One-shot requests, data extraction | POST → `claude -p --output-format json` → JSON response |
| **SSE (Server-Sent Events)** | Streaming text to browser | `claude -p --output-format stream-json --verbose` → SSE stream |
| **WebSocket** | Bidirectional, multi-turn sessions | WS connection → `claude -p --input-format stream-json` |
| **Background job** | Long-running tasks | Queue → `claude -p` process → poll for result or push via WS |
| **HTTP Hooks** | Tool visibility, permission approval, lifecycle events | Configure in `.claude/settings.local.json`, server receives POSTs at lifecycle points |

For most apps, start with **REST + SSE**: REST for triggering tasks, SSE for
streaming progress and results. Add WebSockets only if you need true
bidirectional communication (e.g., the user can interrupt or steer while
Claude is working). Add **HTTP Hooks** when you need structured tool-lifecycle
events or UI-driven permission approval (see "HTTP Hooks" section below).

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
| Multi-step workflow with state | Session-based (`--session-id` first turn, `--resume <id>` after) |
| Concurrent analysis of multiple items | Parallel `claude -p` processes |
| Person steers while Claude works | Bidirectional streaming via WebSocket |

### 3. What does Claude actually do?

Map each action to what Claude needs behind the scenes:

- **What tools?** Read-only analysis (`Read,Glob,Grep`) vs. modification (`Write,Edit,Bash`)
  vs. no tools at all (`--tools ""` for pure reasoning)
- **What persona?** `--system-prompt "You are..."` replaces the default system prompt
  entirely — use this for character personas, branded assistants, or any app where
  Claude should NOT inherit the user's CLAUDE.md settings. `--append-system-prompt`
  adds to the default prompt (preserving user settings, skills, etc.) — use this
  when Claude should still act as a general assistant with extra instructions.
- **What data?** Files on the server? User-uploaded content? Piped from other services?
  Claude's Read tool natively handles images and PDFs — save uploaded binary
  files to a temp directory and pass the path (see "Handling File Uploads" below).
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

- **Auth (required)**: Every Loom app needs valid Anthropic credentials before Claude
  can start. The standard pattern is Anthropic OAuth — users authenticate with their
  own Anthropic account via a one-time setup screen (see `references/oauth-reference.md`).
  For multi-user apps, each user gets their own server-side session with tokens stored
  in memory. The server injects `CLAUDE_CODE_OAUTH_TOKEN` into each spawned Claude
  process — no shared credential file. This ensures each user's Claude processes
  use only their own Anthropic subscription.
- **Sandboxing**: Claude has filesystem access — what directory should it be scoped to?
- **Rate limiting**: Do you need to throttle requests per user or globally?
- **Permissions**: Use the tightest `--permission-mode` and `--allowedTools` that work.
  Prefer `dontAsk` (auto-denies unallowed tools) over `bypassPermissions` (skips all checks).
  Only use `bypassPermissions` when `--allowedTools` fully constrains Claude's capabilities
  and you need unattended execution in a trusted environment (CI/CD, local dev tools).
- **Multi-tenancy**: If multiple users, each needs isolated sessions (see the multi-user
  pattern in `references/oauth-reference.md`). Each user authenticates independently,
  gets a session cookie, and their Claude processes receive their own token via env var.
  Consider separate working directories per user if they have persistent file operations.

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

### Authentication Setup

Before any server pattern below works, each user needs valid Anthropic
credentials. For multi-user apps, tokens are stored in an in-memory session
store — NOT in a shared file. Your server should:

1. Use `requireAuth` middleware on protected endpoints
2. Show the OAuth setup screen if no valid session cookie exists
3. After token exchange, fetch the user's profile from `https://api.anthropic.com/v1/me`
   and store it in the session — the frontend needs this to show who's logged in
4. Call `refreshSessionIfNeeded()` before spawning Claude processes
5. Inject the user's token via `CLAUDE_CODE_OAUTH_TOKEN` env var on each spawn

**Always log OAuth errors server-side.** The exchange endpoint calls
Anthropic's token endpoint over HTTPS — this is the most failure-prone
path (DNS issues on fresh VMs, transient network errors, expired codes).
Both the `!resp.ok` branch and the `catch` block must `console.error`
the actual error, not silently return a generic message. Without this,
`journalctl` shows nothing when the exchange fails and you're debugging
blind.

See `references/oauth-reference.md` for the complete implementation —
PKCE utilities, server endpoints, session store, and a ready-to-use
React setup screen component.

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

**Critical:** `dontAsk` without `--allowedTools` means **no tools at all**.
If your task needs file access, bash, or any other tool, you MUST pair
`--permission-mode dontAsk` with `--allowedTools "Read,Bash,..."` or
`--tools "Read,Bash,..."`. Omitting both gives you a Claude that can reason
but can't act — and the failure is silent (no error, just missing results).

**`--max-turns`** — Prevents
conversational loops where Claude keeps trying approaches that won't work.
`5` for one-shot tasks, `10–15` for streaming, `20` for multi-turn sessions.

Every pattern also handles three failure modes:

- **stderr** — Claude writes warnings, errors, and diagnostics here. Always
  capture it; it's your best debugging signal when something goes wrong.
- **Non-zero exit codes** — Model overloaded, permission denied, timeout hit.
  `execFileSync` throws; `spawn` emits a `close` event.
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

If the server runs inside **cmux** (the Claude terminal app), additional
`CMUX_*` surface identifiers trigger nesting detection and cause the spawned
process to receive SIGTERM immediately. These are terminal-state vars, not
auth — safe to remove. Detect cmux via `CMUX_SURFACE_ID` and strip only the
identifiers that trigger detection, leaving port/integration vars intact.

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

  const env = cleanEnv();

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
      "--permission-mode", "dontAsk",
      "--json-schema", schema, "--tools", "", "--no-session-persistence"
    ], { input: `${task}\n\n${content}`, encoding: "utf-8", timeout: 60000, env: cleanEnv() });

    const parsed = JSON.parse(result);
    if (parsed.is_error) {
      return res.status(502).json({ error: parsed.result });
    }
    res.json(parsed.structured_output);
  } catch (e: any) {
    // execFileSync throws on non-zero exit and timeout
    const stderr = e.stderr?.toString().trim();
    console.error(`[claude] exit ${e.status}, stderr: ${stderr}`);
    res.status(502).json({ error: stderr || "Claude process failed" });
  }
});
```

**Don't do this:**
- Don't use `execSync("claude -p ...")` with string interpolation — use
  `execFileSync` with an args array. Shell strings break on special characters,
  are injection-vulnerable, and fail silently on quoting errors.
- Don't read `structured_output` without checking `is_error` first — when
  Claude hits a tool failure, `structured_output` is `null`.
- Don't skip env cleanup — always use `cleanEnv()` before spawning.
  Do NOT strip all `CLAUDE_*` vars; some are required for auth.

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
  res.flushHeaders();

  // Heartbeat keeps connections alive through proxies (nginx, Cloudflare)
  const heartbeat = setInterval(() => {
    try { res.write(`:heartbeat\n\n`); } catch { clearInterval(heartbeat); }
  }, 5000);

  const proc = spawn("claude", [
    "-p", "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--permission-mode", "dontAsk", "--allowedTools", "Read,Glob,Grep,Bash",
    "--max-turns", "15",
    "--model", "sonnet", "--no-session-persistence",
    task
  ], { env: cleanEnv() });

  let gotResult = false;

  const parse = createStreamParser((event) => {
    // With --include-partial-messages, tokens arrive as stream_event deltas.
    // The assistant event re-delivers the complete text — DO NOT forward it
    // or the frontend will display every token twice.
    if (event.type === "stream_event" && event.event?.delta?.text) {
      res.write(`data: ${JSON.stringify({ type: "token", text: event.event.delta.text })}\n\n`);
    } else if (event.type === "assistant" && event.message?.content) {
      // Only forward tool_use — text was already streamed via stream_event.
      for (const block of event.message.content) {
        if (block.type === "tool_use") {
          res.write(`data: ${JSON.stringify({ type: "tool", name: block.name })}\n\n`);
        }
      }
    } else if (event.type === "result") {
      gotResult = true;
      if (event.is_error) {
        res.write(`data: ${JSON.stringify({ type: "error", message: event.result })}\n\n`);
      } else if (event.subtype === "error_max_turns") {
        res.write(`data: ${JSON.stringify({ type: "warning", message: "Task incomplete — reached turn limit" })}\n\n`);
      } else {
        res.write(`data: ${JSON.stringify({ type: "done" })}\n\n`);
      }
    }
  });

  proc.stdout.on("data", parse);

  proc.stderr.on("data", (chunk) => {
    const msg = chunk.toString().trim();
    if (msg) console.error(`[claude stderr] ${msg}`);
  });

  proc.on("close", (code) => {
    clearInterval(heartbeat);
    if (!gotResult) {
      res.write(`data: ${JSON.stringify({ type: "error", message: code !== 0
        ? `Claude exited with code ${code}`
        : "Process finished without producing a result" })}\n\n`);
    }
    res.end();
  });

  proc.on("error", (err) => {
    clearInterval(heartbeat);
    res.write(`data: ${JSON.stringify({ type: "error", message: err.message })}\n\n`);
    res.end();
  });

  // IMPORTANT: Use res.on("close"), NOT req.on("close").
  // req "close" fires prematurely on POST-based SSE endpoints, killing the
  // spawned process before it can respond. res "close" fires only when the
  // client actually disconnects.
  res.on("close", () => { clearInterval(heartbeat); if (!proc.killed) proc.kill(); });
});
```

**Don't do this:**
- Don't use `req.on("close")` for SSE cleanup — use `res.on("close")`.
  `req` "close" fires prematurely on POST-based SSE endpoints, killing the
  spawned Claude process before it can respond. This causes silent SIGTERM
  on every request. `res` "close" fires only when the client disconnects.
- Don't split on `\n` without buffering the last incomplete line — stream
  chunks can split a JSON line across two `data` events, causing silent
  data loss and intermittent `JSON.parse` failures.
- Don't skip the `gotResult` guard on `close` — if Claude exits without
  emitting a `result` event (timeout, turn limit hit), the frontend
  gets no signal that something went wrong.
- Don't ignore `is_error` on the result — tool failures set
  `is_error: true` with `structured_output: null`.
- Don't omit `--include-partial-messages` — without it, text arrives as a
  single complete block in `assistant` events instead of token-by-token
  `stream_event` deltas. This makes streaming apps feel broken.
- Don't forward text from `assistant` events when using `--include-partial-messages`
  — the same text was already delivered token-by-token via `stream_event`. Only
  forward `tool_use` blocks from `assistant` events, or text will appear twice.

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
  let isFirstTurn = true;

  ws.on("message", (raw) => {
    const { action, payload } = JSON.parse(raw.toString());

    // Prevent concurrent subprocess spawns — one at a time per connection
    if (activeProc) {
      ws.send(JSON.stringify({ type: "error", message: "Processing in progress" }));
      return;
    }

    // First turn: --session-id creates the session
    // Subsequent turns: --resume continues that specific session
    const sessionArgs = isFirstTurn
      ? ["--session-id", sessionId]
      : ["--resume", sessionId];

    const proc = spawn("claude", [
      "-p", "--output-format", "stream-json", "--verbose",
      "--include-partial-messages",
      "--permission-mode", "dontAsk", "--allowedTools", "Read,Write,Edit,Bash,Glob,Grep",
      "--max-turns", "20",
      ...sessionArgs,
      "--model", "sonnet",
      payload.prompt
    ], { env: cleanEnv() });

    isFirstTurn = false;

    proc.stdin.end();
    activeProc = proc;
    let lineBuf = "";
    let gotResult = false;

    const parse = createStreamParser((event) => {
      // With --include-partial-messages, tokens arrive as stream_event deltas.
      // The assistant event re-delivers the complete text — DO NOT forward it
      // or the frontend will display every token twice.
      if (event.type === "stream_event" && event.event?.delta?.text) {
        ws.send(JSON.stringify({ type: "token", text: event.event.delta.text }));
      } else if (event.type === "assistant" && event.message?.content) {
        // Only forward tool_use — text was already streamed via stream_event.
        for (const block of event.message.content) {
          if (block.type === "tool_use") {
            ws.send(JSON.stringify({ type: "tool", name: block.name, input: block.input }));
          }
        }
      } else if (event.type === "result") {
        gotResult = true;
        if (event.is_error) {
          ws.send(JSON.stringify({ type: "error", message: event.result }));
        } else if (event.subtype === "error_max_turns") {
          ws.send(JSON.stringify({ type: "warning", message: "Task incomplete — reached turn limit" }));
        } else {
          ws.send(JSON.stringify({ type: "done" }));
        }
      }
    });

    proc.stdout.on("data", parse);

    proc.stderr.on("data", (chunk) => {
      const msg = chunk.toString().trim();
      if (msg) console.error(`[claude stderr] ${msg}`);
    });

    proc.on("close", (code) => {
      activeProc = null;
      if (!gotResult) {
        ws.send(JSON.stringify({ type: "error", message: code !== 0
          ? `Claude exited with code ${code}`
          : "Process finished without producing a result" }));
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

**Don't do this:**
- Same buffering and `is_error` rules as SSE above — stream chunks split
  across `data` events, and results can carry `is_error: true`.
- Don't forget to null `activeProc` in both `close` and `error` handlers —
  stale references prevent cleanup on WebSocket disconnect.

#### Pattern: Background Job with Progress

Long-running task that reports progress.

```typescript
const jobs = new Map<string, { status: string; result?: any; error?: string }>();

app.post("/api/jobs", (req, res) => {
  const jobId = uuidv4();
  jobs.set(jobId, { status: "running" });
  res.json({ jobId });

  const proc = spawn("claude", [
    "-p", "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--model", "sonnet", "--max-turns", "20",
    "--permission-mode", "dontAsk",
    "--tools", "Read,Bash,Glob,Grep",
    "--no-session-persistence"
  ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv() });

  proc.stdin.write(req.body.task);
  proc.stdin.end();

  let lineBuf = "";
  let stderrBuf = "";

  const parse = createStreamParser((event) => {
    if (event.type === "result") {
      if (event.is_error) {
        jobs.set(jobId, { status: "failed", error: event.result });
      } else if (event.subtype === "error_max_turns") {
        jobs.set(jobId, { status: "incomplete", result: event });
      } else {
        jobs.set(jobId, { status: "complete", result: event });
      }
    }
  });

  proc.stdout.on("data", parse);

  proc.stderr.on("data", (chunk) => {
    stderrBuf += chunk.toString();
  });

  proc.on("close", (code) => {
    if (jobs.get(jobId)?.status === "running") {
      jobs.set(jobId, { status: "failed", error: stderrBuf.trim() || `Exit code ${code || "unknown"}` });
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

**Don't do this:**
- Don't check `code !== 0` in the `close` handler — also check that no result
  was received. A zero exit code without a `result` event means Claude
  finished without producing output (e.g., exceeded turn limits).

#### Pattern: Parallel Analysis

Analyze multiple items concurrently, aggregate results. Uses `spawn` so each
Claude process runs in its own child process — truly parallel.

```typescript
app.post("/api/batch", async (req, res) => {
  const { items, task } = req.body;
  const TIMEOUT_MS = 30000;

  const outcomes = await Promise.allSettled(
    items.map((item: string) => new Promise<any>((resolve, reject) => {
      const proc = spawn("claude", [
        "-p", "--model", "haiku", "--output-format", "json",
        "--permission-mode", "dontAsk",
        "--json-schema", schema, "--tools", "", "--no-session-persistence"
      ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv() });

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

#### Stream-JSON Event Types

When using `--output-format stream-json --verbose`, Claude emits these event
types as newline-delimited JSON. The patterns above use `createStreamParser`
to handle buffering. Here's what each event type means and what to forward
to the frontend:

| Event Type | Shape | What It Means | Forward? |
|------------|-------|---------------|----------|
| `system` | `{type:"system", subtype:"init", session_id, model}` | Session started, model identified | Optional (show model) |
| `stream_event` | `{type:"stream_event", event:{delta:{text}}}` | Incremental token | Yes (live text) |
| `assistant` | `{type:"assistant", message:{content:[...]}}` | Complete message block with text and tool_use | Yes (text + tool progress) |
| `tool_result` | `{type:"tool_result", tool_name, content, is_error}` | Tool completed | Optional (show result) |
| `result` | `{type:"result", total_cost_usd, usage, is_error}` | Session complete | Yes (done + cost) |
| `compact` | `{type:"compact"}` | Context window compacted | No (internal) |

**Important: extended thinking changes the streaming pattern.** Models with
extended thinking (haiku-4.5) do NOT emit `stream_event` tokens. Instead,
text arrives as complete blocks in `assistant` events — first a `thinking`
block, then a `text` block. Always handle both `assistant` text blocks and
`stream_event` deltas so your app works with any model.

**Extracting text and tool use from `assistant` events:**

```typescript
if (event.type === "assistant" && event.message?.content) {
  for (const block of event.message.content) {
    if (block.type === "text" && block.text) {
      // Complete text block — forward as token(s) to the frontend.
      // With extended-thinking models, this IS the primary text delivery.
    } else if (block.type === "tool_use") {
      // Claude is calling a tool: block.name, block.input
      // Forward to frontend for progress indication
    }
  }
}
```

Track tool use for progress estimation:

```typescript
let toolsUsed = 0;
let hasEdited = false;
// ... inside event handler:
if (block.type === "tool_use") {
  toolsUsed++;
  if (block.name === "Edit" || block.name === "Write") hasEdited = true;
}
// Progress: hasEdited means nearly done, toolsUsed >= 3 means well underway
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

  // Handle expired sessions (server restart wipes in-memory sessions)
  if (response.status === 401) {
    showSetupScreen(); // redirect to OAuth setup
    return;
  }

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
causes: model overloaded (503 from upstream), permission denied (tool not
in `--allowedTools`), or process killed
by your timeout. For `execFileSync`, this throws — catch it. For `spawn`,
listen on the `close` event and check the code.

**Malformed output** happens when a process is killed mid-stream (timeout,
client disconnect, OOM). The stdout buffer contains partial JSON that won't
parse. Always wrap `JSON.parse` in try/catch, and always check `parsed.is_error`
before reaching for `structured_output` — Claude sets this flag when it
couldn't complete the task (tool failures, turn limit exceeded, etc.).

#### Error Surfacing Checklist

Every streaming pattern (SSE, WebSocket, Background Job) should surface
errors through three channels. If you're generating a new Loom app, verify
all three are present:

1. **`proc.stderr`** → forward to the client as a diagnostic event so the
   frontend can show it (or at minimum, log it server-side):
   ```typescript
   proc.stderr.on("data", (chunk) => {
     const msg = chunk.toString().trim();
     if (msg) {
       res.write(`data: ${JSON.stringify({ type: "stderr", message: msg })}\n\n`);
     }
   });
   ```

2. **`proc.on("close")` + `gotResult` guard** → if the process exits
   without emitting a `result` event, tell the client something went wrong:
   ```typescript
   proc.on("close", (code) => {
     if (!gotResult) {
       res.write(`data: ${JSON.stringify({ type: "error", message: code !== 0
         ? `Claude exited with code ${code}`
         : "Process finished without producing a result" })}\n\n`);
     }
     res.end();
   });
   ```

3. **`is_error` check on result** → before accessing `structured_output`,
   check whether Claude flagged the run as failed:
   ```typescript
   if (event.type === "result") {
     gotResult = true;
     if (event.is_error) {
       res.write(`data: ${JSON.stringify({ type: "error", message: event.result })}\n\n`);
     } else {
       res.write(`data: ${JSON.stringify({ type: "done", data: event.structured_output })}\n\n`);
     }
   }
   ```

If any of these three is missing, the app will fail silently in at least
one failure mode.

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

### Handling File Uploads and Drops

Claude's Read tool natively handles images (PNG, JPG, JPEG, GIF, WebP, BMP)
and PDFs — it can see images visually and parse PDF pages. Apps that accept
file uploads or drag-and-drop should categorize files into three tiers:

| Category | Examples | Handling |
|----------|----------|----------|
| **Text** | `.ts`, `.md`, `.json`, `.csv` | Inline content in prompt via `<file>` tags |
| **Claude-readable binary** | `.png`, `.jpg`, `.pdf`, `.gif`, `.webp`, `.bmp` | Save to temp dir, pass file path in prompt |
| **Other binary** | `.zip`, `.exe`, `.mp4` | Text description only (name, type, size) |

For Claude-readable binary files, save them to a temp directory and include
the path in the prompt. Claude will use its Read tool to analyze them:

```typescript
import { mkdtempSync, writeFileSync, rmSync } from "fs";
import { tmpdir } from "os";
import path from "path";

const NATIVE_READ_EXTS = new Set([
  "png", "jpg", "jpeg", "gif", "webp", "bmp", "pdf",
]);

function buildPromptWithFiles(
  userMessage: string,
  files: Array<{ name: string; content: string; passAsFile?: boolean }>,
): { prompt: string; tempDir?: string } {
  const textParts: string[] = [];
  const filePaths: string[] = [];
  let tempDir: string | undefined;

  for (const f of files) {
    if (f.passAsFile) {
      if (!tempDir) tempDir = mkdtempSync(path.join(tmpdir(), "loom-"));
      const filePath = path.join(tempDir, f.name);
      writeFileSync(filePath, Buffer.from(f.content, "base64"));
      filePaths.push(filePath);
    } else {
      textParts.push(`<file name="${f.name}">\n${f.content}\n</file>`);
    }
  }

  const parts: string[] = [];
  if (filePaths.length > 0) {
    parts.push("Use your Read tool to examine these files:");
    for (const p of filePaths) parts.push(p);
  }
  if (textParts.length > 0) parts.push("", ...textParts);
  parts.push("", userMessage);

  return { prompt: parts.join("\n"), tempDir };
}
```

**Important:** Clean up the temp directory after the Claude process exits.
Pass `tempDir` to your spawn function and call `rmSync(tempDir, { recursive: true })`
in the cleanup handler.

On the frontend (browser/webview), detect file type and base64-encode
Claude-readable binary files before sending them to the server:

```typescript
const nativeReadExts = new Set([
  "png", "jpg", "jpeg", "gif", "webp", "bmp", "pdf",
]);

async function processDroppedFile(file: File) {
  const ext = file.name.split(".").pop()?.toLowerCase() || "";
  const isNativeReadable = nativeReadExts.has(ext)
    || file.type.startsWith("image/")
    || file.type === "application/pdf";

  if (isNativeReadable) {
    const bytes = new Uint8Array(await file.arrayBuffer());
    let binary = "";
    for (let i = 0; i < bytes.byteLength; i++) {
      binary += String.fromCharCode(bytes[i]);
    }
    return { name: file.name, content: btoa(binary), passAsFile: true };
  }

  // Fall through to text reading or binary description
}
```

### Advanced Patterns

These patterns appear in production but aren't needed for every app. Reach
for them when the basic patterns aren't enough.

#### Structured Extraction (Async Haiku)

For lightweight data extraction tasks — form field suggestions, entity
parsing, classification — spawn a one-shot Haiku process with no tools
and a JSON schema. Async with a timeout kill, unlike the synchronous
`execFileSync` REST pattern.

```typescript
async function extract<T>(prompt: string, schema: object, timeoutMs = 30000): Promise<T> {
  const proc = spawn("claude", [
    "-p", "--model", "haiku", "--output-format", "json",
    "--json-schema", JSON.stringify(schema),
    "--tools", "", "--no-session-persistence",
    "--permission-mode", "dontAsk"
  ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv() });

  proc.stdin.write(prompt);
  proc.stdin.end();

  const timer = setTimeout(() => proc.kill(), timeoutMs);

  try {
    let stdout = "";
    for await (const chunk of proc.stdout) stdout += chunk.toString();
    const exitCode = await new Promise<number>(r => proc.on("close", r));
    if (exitCode !== 0) {
      let stderr = "";
      for await (const chunk of proc.stderr) stderr += chunk.toString();
      throw new Error(`Extraction failed (exit ${exitCode}): ${stderr.slice(0, 200)}`);
    }
    const wrapper = JSON.parse(stdout);
    if (wrapper.structured_output) return wrapper.structured_output as T;
    if (typeof wrapper.result === "string") return JSON.parse(wrapper.result) as T;
    return wrapper as T;
  } finally {
    clearTimeout(timer);
  }
}
```

#### Persistent Session (Long-Lived Process)

Instead of spawning a new process per request, keep one Claude process alive
and send messages via JSONL on stdin. Lower latency, continuous context, no
session serialization overhead. The tradeoff is lifecycle management.

```typescript
import { spawn, ChildProcess } from "child_process";

let proc: ChildProcess | null = null;
let sessionActive = false;
let lastActivity = Date.now();

function startSession(systemPrompt?: string) {
  const args = [
    "-p",
    "--input-format", "stream-json",
    "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--permission-mode", "dontAsk",
    "--allowedTools", "Read,Write,Edit,Bash,Glob,Grep",
  ];
  if (systemPrompt) args.push("--append-system-prompt", systemPrompt);

  proc = spawn("claude", args, {
    stdio: ["pipe", "pipe", "pipe"],
    env: cleanEnv(),
  });
  sessionActive = true;

  const parse = createStreamParser((event) => {
    if (event.type === "stream_event" && event.event?.delta?.text) {
      broadcast({ type: "token", text: event.event.delta.text });
    } else if (event.type === "assistant" && event.message?.content) {
      for (const block of event.message.content) {
        if (block.type === "tool_use") {
          broadcast({ type: "tool", name: block.name, input: block.input });
        }
      }
    } else if (event.type === "result") {
      if (event.is_error) {
        broadcast({ type: "error", message: event.result });
      } else if (event.subtype === "error_max_turns") {
        broadcast({ type: "warning", message: "Task incomplete — reached turn limit" });
      } else {
        broadcast({ type: "done" });
      }
    }
  });
  proc.stdout!.on("data", parse);
  proc.stderr!.on("data", (chunk) => console.error(`[claude] ${chunk}`));
  proc.on("close", () => { sessionActive = false; proc = null; });
}

function sendMessage(text: string): boolean {
  if (!proc || !sessionActive) return false;
  const jsonl = JSON.stringify({
    type: "user",
    message: { role: "user", content: [{ type: "text", text }] },
  }) + "\n";
  proc.stdin!.write(jsonl);
  lastActivity = Date.now();
  return true;
}

function endSession() {
  if (proc) proc.kill();
}

// Inactivity timeout — kill idle sessions after 15 minutes
setInterval(() => {
  if (sessionActive && Date.now() - lastActivity > 15 * 60 * 1000) {
    endSession();
  }
}, 60000);
```

Key differences from the per-request patterns:
- Uses `--input-format stream-json` for bidirectional communication
- Stdin stays open — messages are newline-delimited JSON, not piped and closed
- `--append-system-prompt` injects persona/constraints without replacing base prompt
- Must manage lifecycle: start, stop, inactivity timeout

#### Action Markers

Let Claude trigger structured side-effects from within its text output. Define
a marker format in your system prompt, parse it server-side, and forward as
typed events to the frontend.

```typescript
// In your system prompt:
// "When you complete a task, emit: [ACTION] {\"type\":\"task_complete\",\"data\":{...}}"

function parseMarkers(event: any, emit: (marker: any) => void) {
  if (event.type !== "assistant" || !event.message?.content) return;
  for (const block of event.message.content) {
    if (block.type !== "text") continue;
    for (const line of block.text.split("\n")) {
      const match = line.match(/\[ACTION\]\s*(\{.*\})/);
      if (match) {
        try { emit(JSON.parse(match[1])); } catch { /* malformed marker */ }
      }
    }
  }
}
```

For **tool lifecycle events** (tool started, tool completed, Claude stopped),
prefer HTTP Hooks (see below) — they're structured, reliable, and don't
require parsing text output. Action Markers remain the right choice for
**mid-stream custom events**: things Claude emits during text generation that
don't correspond to tool calls (progress steps, status updates, UI triggers).
Hooks can't do mid-stream events; Action Markers can't do tool lifecycle
events. They're complementary.

### HTTP Hooks: Sideband Events from Claude to Your Server

Claude Code can POST structured JSON to your Loom server at lifecycle points —
before/after tool calls, when Claude finishes responding, when sessions
start/end. This gives your server reliable, typed events without parsing
stdout. HTTP hooks **supplement** stdout streaming (you still need
`stream-json` for token-by-token text), but they're more reliable for
lifecycle events because they come from Claude Code's own event system.

Configure hooks in `.claude/settings.local.json` (gitignored — these contain
localhost URLs specific to your dev environment):

```json
{
  "hooks": {
    "PostToolUse": [{
      "hooks": [{
        "type": "http",
        "url": "http://localhost:3456/hooks/tool-complete",
        "timeout": 30
      }]
    }],
    "PreToolUse": [{
      "matcher": "Bash|Write|Edit",
      "hooks": [{
        "type": "http",
        "url": "http://localhost:3456/hooks/pre-tool",
        "timeout": 120
      }]
    }],
    "Stop": [{
      "hooks": [{
        "type": "http",
        "url": "http://localhost:3456/hooks/stop",
        "timeout": 30
      }]
    }]
  }
}
```

**Important:** Hooks fire for ALL `claude -p` processes in the project, not
just ones spawned by your app. Use `session_id` from the POST body to route
events to the correct client connection. Each hook POST includes `session_id`,
`tool_name`, `tool_input`, and (for PostToolUse) `tool_response`.

#### Receiving Hook Events

Add Express endpoints to handle the POSTs and relay to the browser:

```typescript
// Map session IDs to active WebSocket/SSE connections
const sessionClients = new Map<string, Set<WebSocket>>();

app.post("/hooks/tool-complete", express.json(), (req, res) => {
  const { session_id, tool_name, tool_input, tool_response } = req.body;
  const clients = sessionClients.get(session_id);
  if (clients) {
    const event = JSON.stringify({
      type: "tool_complete",
      tool: tool_name,
      input: tool_input,
      response: tool_response
    });
    for (const ws of clients) ws.send(event);
  }
  res.json({}); // 2xx with empty body = allow, no decision
});

app.post("/hooks/stop", express.json(), (req, res) => {
  const { session_id, last_assistant_message } = req.body;
  const clients = sessionClients.get(session_id);
  if (clients) {
    for (const ws of clients) {
      ws.send(JSON.stringify({ type: "claude_stopped" }));
    }
  }
  // Return empty to allow Claude to stop normally.
  // Return {"decision": "block", "reason": "..."} to force Claude to continue.
  res.json({});
});
```

#### Interactive Permission Approval from the Browser

The most powerful HTTP hook pattern for Loom apps: let users approve or deny
tool calls from the web UI instead of being locked into `--permission-mode
dontAsk`. This turns the browser into the permission dialog.

The flow:
1. Claude decides to use a tool (e.g., `Bash "npm test"`)
2. Claude Code POSTs to your `PreToolUse` hook endpoint
3. Your server pushes the tool details to the browser via WebSocket
4. The user sees an approval UI and clicks approve or deny
5. Your server responds to the HTTP hook request with the decision
6. Claude Code proceeds or blocks based on the response

**Server endpoint:**

```typescript
// Pending permission requests, keyed by a unique request ID
const pendingPermissions = new Map<string, {
  resolve: (decision: any) => void;
  timer: NodeJS.Timeout;
}>();

app.post("/hooks/pre-tool", express.json(), (req, res) => {
  const { session_id, tool_name, tool_input } = req.body;
  const clients = sessionClients.get(session_id);

  if (!clients || clients.size === 0) {
    // No connected browser — auto-deny for safety
    return res.json({
      hookSpecificOutput: {
        hookEventName: "PreToolUse",
        permissionDecision: "deny",
        permissionDecisionReason: "No browser session connected"
      }
    });
  }

  const requestId = crypto.randomUUID();

  // Push to browser
  const event = JSON.stringify({
    type: "permission_request",
    requestId,
    tool: tool_name,
    input: tool_input
  });
  for (const ws of clients) ws.send(event);

  // Wait for user decision (timeout after 90s)
  const promise = new Promise<any>((resolve) => {
    const timer = setTimeout(() => {
      pendingPermissions.delete(requestId);
      resolve({
        hookSpecificOutput: {
          hookEventName: "PreToolUse",
          permissionDecision: "deny",
          permissionDecisionReason: "Permission request timed out"
        }
      });
    }, 90000);
    pendingPermissions.set(requestId, { resolve, timer });
  });

  promise.then((decision) => res.json(decision));
});

// Browser sends approval/denial here
app.post("/api/permission-response", express.json(), (req, res) => {
  const { requestId, approved } = req.body;
  const pending = pendingPermissions.get(requestId);
  if (!pending) return res.status(404).json({ error: "Request expired" });

  clearTimeout(pending.timer);
  pendingPermissions.delete(requestId);

  pending.resolve({
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: approved ? "allow" : "deny",
      permissionDecisionReason: approved
        ? "User approved via browser"
        : "User denied via browser"
    }
  });

  res.json({ ok: true });
});
```

**Frontend approval component:**

```javascript
// Handle permission requests from WebSocket
ws.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  if (msg.type === "permission_request") {
    showPermissionDialog(msg);
  }
};

function showPermissionDialog({ requestId, tool, input }) {
  const dialog = document.createElement("div");
  dialog.className = "permission-dialog";
  dialog.innerHTML = `
    <div class="permission-content">
      <h3>Claude wants to use: ${esc(tool)}</h3>
      <pre>${esc(JSON.stringify(input, null, 2))}</pre>
      <div class="permission-actions">
        <button onclick="respondPermission('${requestId}', true)">Allow</button>
        <button onclick="respondPermission('${requestId}', false)">Deny</button>
      </div>
    </div>
  `;
  document.body.appendChild(dialog);
}

async function respondPermission(requestId, approved) {
  await fetch("/api/permission-response", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ requestId, approved })
  });
  document.querySelector(".permission-dialog")?.remove();
}
```

**Important considerations:**
- Set `"timeout": 120` on the PreToolUse hook — the default 30s isn't enough
  for human approval. Claude Code waits for the hook response before proceeding.
- Use `--permission-mode default` (not `dontAsk`) when using this pattern, so
  Claude Code sends `PreToolUse` events for tools that need approval.
- The `session_id` in the hook POST matches the session you get from
  `--session-id` or the `system.init` event in stream-json output. Use this
  to route permission requests to the correct browser tab.
- If no browser is connected, auto-deny for safety. Don't let tool calls
  hang forever waiting for a user who isn't there.

## What to Generate

When you build the app, produce:

1. **`server.ts`** — Express server with cookie-parser, session store,
   `requireAuth` middleware, OAuth endpoints (`/api/oauth/start`,
   `/api/oauth/exchange`, `/api/health`, `/api/logout`), and your app's
   endpoint pattern(s). The exchange endpoint must fetch the user's
   profile from `https://api.anthropic.com/v1/me` and store it in
   the session. Each spawn uses `spawnEnvForUser()` to inject
   the requesting user's token.
2. **`public/index.html`** — The frontend, starting with the `<SetupScreen>`
   component (shown when no session exists) and your app's main UI
   (shown after authentication). Must include a user indicator showing
   the user's email (from `/api/health` → `user.email`) and a logout
   button that calls `POST /api/logout`. Use a fallback label like
   "CONNECTED" if the profile has no email. Protected endpoints must
   check for 401 responses and redirect to the setup screen (in-memory
   sessions are wiped on server restart). If the app has both a chat
   input and a setup screen input, use `input.setup-input` selectors
   (not `.setup-input`) to avoid CSS specificity conflicts with global
   `input[type="text"]` rules — see `references/oauth-reference.md`.
3. **`package.json`** — Dependencies (including `cookie-parser`) and start script
4. **A one-liner to run it** — so the person can verify it works immediately

For simple apps, a single `server.ts` serving a static `index.html` is ideal.
For complex UIs, scaffold a React frontend with a separate server.

After generating, offer to start the server and open it in the browser together.
Then iterate based on what the person sees.

### Deployment Verification

When deploying to a remote VM (exe.dev, etc.), verify outbound
connectivity after starting the service. Fresh VMs can have transient
DNS or network issues that cause the first OAuth exchange to fail:

```bash
# Verify the app can reach Anthropic's token endpoint
curl -s -o /dev/null -w "%{http_code}" https://platform.claude.com/v1/oauth/token
# Should return 405 (Method Not Allowed for GET) — confirms connectivity
```

If this returns a network error, wait a few seconds and retry. Don't
declare deployment complete until the VM can reach Anthropic's servers.

## The Possibility Space

The most interesting Loom apps are not chat interfaces in a browser.
They're things that couldn't exist without an agentic runtime — applications
where the backend can read, reason, and act on context that traditional APIs
can't access.

Help people think about what they actually want to make. The question isn't
"how do I put a chat box in a browser?" but "what would this look like if
there were an intelligence behind it?"
