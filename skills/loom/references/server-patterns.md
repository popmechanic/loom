# Server Patterns Reference

Core server-side code patterns for Loom web apps — shared utilities, communication patterns, streaming event handling, error management, and frontend integration. Read this when implementing a specific server endpoint or wiring up the frontend.

## Table of Contents

- [Shared Utilities](#shared-utilities)
  - [cleanEnv()](#cleanenv)
  - [createStreamParser()](#createstreamparser)
  - [spawnEnvForUser()](#spawnenvforuser)
- [Server Setup](#server-setup)
- [Pattern: REST Endpoint](#pattern-rest-endpoint)
- [Pattern: SSE Streaming](#pattern-sse-streaming)
- [Pattern: WebSocket Session](#pattern-websocket-session)
- [Pattern: Background Job with Progress](#pattern-background-job-with-progress)
- [Pattern: Parallel Analysis](#pattern-parallel-analysis)
- [Stream-JSON Event Types](#stream-json-event-types)
- [Frontend Integration](#frontend-integration)
  - [Streaming Text Display](#streaming-text-display)
  - [Structured Result Rendering](#structured-result-rendering)
- [Error Surfacing Checklist](#error-surfacing-checklist)
- [Input Validation](#input-validation)
- [Temp Directory Cleanup](#temp-directory-cleanup)
- [Handling File Uploads and Drops](#handling-file-uploads-and-drops)

---

## Shared Utilities

Every pattern below uses these three helpers. Define them once at the top of
your server file.

### cleanEnv()

Remove nesting guards so `claude -p` can start.

Used internally by `spawnEnvForUser()` below — do not call directly at spawn sites.

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

### createStreamParser()

Buffer stdout chunks into complete JSON lines.

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

### spawnEnvForUser()

Inject the user's OAuth token into the spawn environment.

Every spawn needs the user's `CLAUDE_CODE_OAUTH_TOKEN` in the environment —
without it, `claude -p` starts but can't authenticate with Anthropic's servers.
This wraps `cleanEnv()` with the token injection. See
`references/oauth-reference.md` for the full session store, `requireAuth`
middleware, and `refreshSessionIfNeeded()` that provide `req.userSession`.

```typescript
// UserSession interface is defined in references/oauth-reference.md (session store)
function spawnEnvForUser(session: UserSession): NodeJS.ProcessEnv {
  const env = cleanEnv();
  env.CLAUDE_CODE_OAUTH_TOKEN = session.accessToken;
  return env;
}
```

---

## Server Setup

Every pattern below assumes this baseline Express setup. Define it once at
the top of your server file, alongside the shared utilities.

```typescript
import express from "express";
import cookieParser from "cookie-parser";  // npm i cookie-parser @types/cookie-parser
import path from "path";

const app = express();
app.use(express.json());
app.use(cookieParser());
app.use(express.static(path.join(__dirname, "public")));

// Auth infrastructure — see references/oauth-reference.md for implementations:
// - requireAuth middleware (validates session cookie on protected routes)
// - refreshSessionIfNeeded() (call before every spawn)
// - spawnEnvForUser() (injects CLAUDE_CODE_OAUTH_TOKEN into spawn env)
// - OAuth endpoints: /api/oauth/start, /api/oauth/exchange, /api/health, /api/logout

// If frontend and server run on different ports (e.g., Vite dev + Express API):
// import cors from "cors";
// app.use(cors({ origin: "http://localhost:5173" }));

const PORT = process.env.PORT || 3456;
const server = app.listen(PORT, () => console.log(`Server running on http://localhost:${PORT}`));
```

---

## Pattern: REST Endpoint

Someone triggers an action → server calls Claude → returns JSON.

```typescript
// Uses Server Setup block — app, express.json(), etc. already defined.
import { execFileSync } from "child_process";

app.post("/api/analyze", requireAuth, async (req: any, res) => {
  try {
    await refreshSessionIfNeeded(req.userSession);
  } catch {
    return res.status(401).json({ error: "Session expired — please sign in again" });
  }
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
      "--permission-mode", "dontAsk",
      "--json-schema", schema, "--tools", "", "--no-session-persistence"
    ], { input: `${task}\n\n${content}`, encoding: "utf-8", timeout: 60000, env: spawnEnvForUser(req.userSession) });

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
- Don't skip env setup — always use `spawnEnvForUser()` before spawning.
  This calls `cleanEnv()` internally (removing nesting guards) and injects
  the user's OAuth token. Do NOT strip all `CLAUDE_*` vars.

---

## Pattern: SSE Streaming

Someone triggers a task → server streams Claude's output token-by-token.

Use `app.post` for CSRF safety — task data stays in the body instead of the URL.
(`app.get` works for quick prototyping with `EventSource`, but `EventSource` only
supports GET, so switch to `fetch()` + `ReadableStream` on the client for POST.)

```typescript
app.post("/api/stream", requireAuth, async (req: any, res) => {
  try {
    await refreshSessionIfNeeded(req.userSession);
  } catch {
    return res.status(401).json({ error: "Session expired — please sign in again" });
  }
  const { task } = req.body;

  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.flushHeaders();

  // Guard SSE writes: if the client disconnected mid-stream, res.write throws
  // (and crashes the process) from inside async stream callbacks. Swallow it.
  const send = (payload: unknown) => {
    if (res.writableEnded) return;
    try { res.write(`data: ${JSON.stringify(payload)}\n\n`); } catch { /* client gone */ }
  };

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
  ], {
    // detached: true puts claude in its own process group so process.kill(-pid)
    // kills the whole Bash/MCP tool subtree, not just the direct child.
    detached: true,
    env: spawnEnvForUser(req.userSession)
  });

  let gotResult = false;

  const parse = createStreamParser((event) => {
    // With --include-partial-messages, tokens arrive as stream_event deltas.
    // The assistant event re-delivers the complete text — DO NOT forward it
    // or the frontend will display every token twice.
    if (event.type === "stream_event" && event.event?.delta?.text) {
      send({ type: "token", text: event.event.delta.text });
    } else if (event.type === "assistant" && event.message?.content) {
      // Only forward tool_use — text was already streamed via stream_event.
      for (const block of event.message.content) {
        if (block.type === "tool_use") {
          send({ type: "tool", name: block.name });
        }
      }
    } else if (event.type === "result") {
      gotResult = true;
      if (event.subtype === "error_max_turns") {
        // is_error: true; check subtype first — error_max_turns is a turn-limit warning, not a hard failure
        send({ type: "warning", message: "Task incomplete — reached turn limit" });
      } else if (event.is_error) {
        send({ type: "error", message: event.result });
      } else {
        send({ type: "done" });
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
      send({ type: "error", message: code !== 0
        ? `Claude exited with code ${code}`
        : "Process finished without producing a result" });
    }
    res.end();
  });

  proc.on("error", (err) => {
    clearInterval(heartbeat);
    send({ type: "error", message: err.message });
    res.end();
  });

  // IMPORTANT: Use res.on("close"), NOT req.on("close").
  // req "close" fires prematurely on POST-based SSE endpoints, killing the
  // spawned process before it can respond. res "close" fires only when the
  // client actually disconnects.
  res.on("close", () => {
    clearInterval(heartbeat);
    if (!proc.killed) {
      // Kill the process group so Bash/MCP tool subtrees die too.
      // Bare proc.kill() only kills the direct child; tool subprocesses survive.
      try { process.kill(-proc.pid!, "SIGTERM"); } catch { proc.kill(); }
    }
  });
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

---

## Pattern: WebSocket Session

Persistent connection with multi-turn conversation and streaming.

```typescript
import { WebSocketServer } from "ws";
import { spawn, ChildProcess } from "child_process";
import { v4 as uuidv4 } from "uuid";
import { parse as parseCookie } from "cookie";

// Attach to the Express HTTP server, not a standalone port
const wss = new WebSocketServer({ noServer: true });

// Authenticate on upgrade — Express middleware doesn't run on WebSocket handshakes
// SESSION_COOKIE_NAME and sessions are from the session store in oauth-reference.md
server.on("upgrade", (req, socket, head) => {
  const cookies = parseCookie(req.headers.cookie || "");
  const sessionId = cookies[SESSION_COOKIE_NAME];
  const session = sessionId ? sessions.get(sessionId) : null;
  if (!session) {
    socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n");
    socket.destroy();
    return;
  }
  wss.handleUpgrade(req, socket, head, (ws) => {
    (ws as any).userSession = session;
    wss.emit("connection", ws, req);
  });
});

wss.on("connection", (ws: any) => {
  const userSession: UserSession = ws.userSession;
  const sessionId = uuidv4();
  let activeProc: ChildProcess | null = null;
  let isFirstTurn = true;

  ws.on("message", async (raw) => {
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

    try {
      await refreshSessionIfNeeded(userSession);
    } catch {
      ws.send(JSON.stringify({ type: "error", message: "Session expired" }));
      ws.close();
      return;
    }

    const proc = spawn("claude", [
      "-p", "--output-format", "stream-json", "--verbose",
      "--include-partial-messages",
      "--permission-mode", "dontAsk", "--allowedTools", "Read,Write,Edit,Bash,Glob,Grep",
      "--max-turns", "20",
      ...sessionArgs,
      "--model", "sonnet",
      payload.prompt
    ], {
      // detached: true puts claude in its own process group so process.kill(-pid)
      // kills the whole Bash/MCP tool subtree, not just the direct child.
      detached: true,
      env: spawnEnvForUser(userSession)
    });

    isFirstTurn = false;

    proc.stdin.end();
    activeProc = proc;
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
        if (event.subtype === "error_max_turns") {
          // is_error: true; check subtype first — error_max_turns is a turn-limit warning, not a hard failure
          ws.send(JSON.stringify({ type: "warning", message: "Task incomplete — reached turn limit" }));
        } else if (event.is_error) {
          ws.send(JSON.stringify({ type: "error", message: event.result }));
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
    if (activeProc) {
      // Kill the process group so Bash/MCP tool subtrees die too.
      try { process.kill(-activeProc.pid!, "SIGTERM"); } catch { activeProc.kill(); }
    }
  });
});
```

**Don't do this:**
- Same buffering and `is_error` rules as SSE — stream chunks split
  across `data` events, and results can carry `is_error: true`.
- Don't forget to null `activeProc` in both `close` and `error` handlers —
  stale references prevent cleanup on WebSocket disconnect.

---

## Pattern: Background Job with Progress

Long-running task that reports progress.

```typescript
const TIMEOUT_MS = 120_000; // 2 minutes — adjust to your workload
const JOB_TTL_MS = 5 * 60_000; // keep finished jobs for 5 minutes

interface Job {
  status: string;
  result?: any;
  error?: string;
  proc?: ReturnType<typeof spawn>;   // stored so callers can cancel
  ownerSessionId: string;            // guards /api/jobs/:id
  finishedAt?: number;               // for TTL sweep
}
const jobs = new Map<string, Job>();

// Periodically delete finished jobs so the map doesn't grow unbounded.
setInterval(() => {
  const now = Date.now();
  for (const [id, job] of jobs) {
    if (job.finishedAt && now - job.finishedAt > JOB_TTL_MS) {
      jobs.delete(id);
    }
  }
}, 60_000);

app.post("/api/jobs", requireAuth, async (req: any, res) => {
  try {
    await refreshSessionIfNeeded(req.userSession);
  } catch {
    return res.status(401).json({ error: "Session expired — please sign in again" });
  }
  const jobId = uuidv4();
  // Owner id = the session cookie value (requireAuth already validated it).
  // req.session does not exist — there is no express-session middleware.
  const ownerSessionId = req.cookies[SESSION_COOKIE_NAME] as string;

  const proc = spawn("claude", [
    "-p", "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--model", "sonnet", "--max-turns", "20",
    "--permission-mode", "dontAsk",
    "--tools", "Read,Bash,Glob,Grep",
    "--no-session-persistence"
  ], {
    stdio: ["pipe", "pipe", "pipe"],
    // detached: spawn in its own process group so proc.kill(-pid) kills the
    // whole Bash/MCP tool subtree, not just the direct child.
    detached: true,
    env: spawnEnvForUser(req.userSession)
  });

  jobs.set(jobId, { status: "running", proc, ownerSessionId });
  res.json({ jobId });

  // Guard stdin writes — without this, an EPIPE (write after stdin closed)
  // crashes the entire server process instead of being handled locally.
  proc.stdin.on("error", () => {});
  proc.stdin.write(req.body.task);
  proc.stdin.end();

  // Kill the job (and its tool subtree) if it exceeds the timeout.
  const killTimer = setTimeout(() => {
    const job = jobs.get(jobId);
    if (job?.status === "running") {
      jobs.set(jobId, { ...job, status: "failed", error: "Timeout", finishedAt: Date.now() });
      try { process.kill(-proc.pid!, "SIGTERM"); } catch { proc.kill(); }
    }
  }, TIMEOUT_MS);

  let stderrBuf = "";

  const parse = createStreamParser((event) => {
    if (event.type === "result") {
      const job = jobs.get(jobId)!;
      if (event.subtype === "error_max_turns") {
        // is_error: true; check subtype first — error_max_turns is a turn-limit warning, not a hard failure
        jobs.set(jobId, { ...job, status: "incomplete", result: event, finishedAt: Date.now() });
      } else if (event.is_error) {
        jobs.set(jobId, { ...job, status: "failed", error: event.result, finishedAt: Date.now() });
      } else {
        jobs.set(jobId, { ...job, status: "complete", result: event, finishedAt: Date.now() });
      }
    }
  });

  proc.stdout.on("data", parse);

  proc.stderr.on("data", (chunk) => {
    stderrBuf += chunk.toString();
  });

  proc.on("close", (code) => {
    clearTimeout(killTimer);
    const job = jobs.get(jobId);
    if (job?.status === "running") {
      jobs.set(jobId, { ...job, status: "failed", error: stderrBuf.trim() || `Exit code ${code ?? "unknown"}`, finishedAt: Date.now() });
    }
  });

  proc.on("error", (err) => {
    clearTimeout(killTimer);
    const job = jobs.get(jobId);
    jobs.set(jobId, { ...(job ?? { ownerSessionId }), status: "failed", error: err.message, finishedAt: Date.now() });
  });
});

app.get("/api/jobs/:id", requireAuth, (req: any, res) => {
  const job = jobs.get(req.params.id);
  if (!job) return res.json({ status: "not_found" });
  // Prevent job-ID enumeration: only the session that created the job can read it.
  if (job.ownerSessionId !== req.cookies[SESSION_COOKIE_NAME]) {
    return res.status(403).json({ error: "Forbidden" });
  }
  // Omit proc handle from the response (it's not serializable).
  const { proc: _proc, ...safe } = job;
  res.json(safe);
});
```

**Don't do this:**
- Don't check `code !== 0` in the `close` handler — also check that no result
  was received. A zero exit code without a `result` event means Claude
  finished without producing output (e.g., exceeded turn limits).

---

## Pattern: Parallel Analysis

Analyze multiple items concurrently, aggregate results. Uses `spawn` so each
Claude process runs in its own child process — truly parallel.

```typescript
app.post("/api/batch", requireAuth, async (req: any, res) => {
  try {
    await refreshSessionIfNeeded(req.userSession);
  } catch {
    return res.status(401).json({ error: "Session expired — please sign in again" });
  }
  const { items, task } = req.body;
  const TIMEOUT_MS = 30000;
  const MAX_ITEMS = 20;
  const CONCURRENCY = 4;
  if (!Array.isArray(items) || items.length === 0) {
    return res.status(400).json({ error: "items must be a non-empty array" });
  }
  if (items.length > MAX_ITEMS) {
    return res.status(400).json({ error: `Too many items (max ${MAX_ITEMS})` });
  }

  // Bounded pool: at most CONCURRENCY claude processes alive at once.
  async function mapWithConcurrency<T, R>(arr: T[], limit: number, fn: (item: T, i: number) => Promise<R>) {
    const results: PromiseSettledResult<R>[] = new Array(arr.length);
    let next = 0;
    await Promise.all(Array.from({ length: Math.min(limit, arr.length) }, async () => {
      while (next < arr.length) {
        const i = next++;
        try { results[i] = { status: "fulfilled", value: await fn(arr[i], i) }; }
        catch (reason) { results[i] = { status: "rejected", reason } as PromiseSettledResult<R>; }
      }
    }));
    return results;
  }

  const outcomes = await mapWithConcurrency(items, CONCURRENCY, (item: string) => new Promise<any>((resolve, reject) => {
      const proc = spawn("claude", [
        "-p", "--model", "haiku", "--output-format", "json",
        "--permission-mode", "dontAsk",
        "--json-schema", schema, "--tools", "", "--no-session-persistence"
      ], { stdio: ["pipe", "pipe", "pipe"], detached: true, env: spawnEnvForUser(req.userSession) });

      const killProc = () => {
        try { process.kill(-proc.pid!, "SIGTERM"); } catch { proc.kill(); }
      };
      const timeout = setTimeout(() => { killProc(); reject(new Error("Timeout")); }, TIMEOUT_MS);
      let stdout = "";
      let stderr = "";

      proc.stdout.on("data", (chunk) => { stdout += chunk.toString(); });
      proc.stderr.on("data", (chunk) => { stderr += chunk.toString(); });

      // Register close listener BEFORE writing to stdin
      proc.on("close", (code) => {
        clearTimeout(timeout);
        try {
          const parsed = JSON.parse(stdout);
          // check subtype before is_error so error_max_turns surfaces as a warning
          if (parsed.subtype === "error_max_turns") {
            reject(new Error("Task incomplete — reached turn limit"));
          } else if (parsed.is_error) {
            reject(new Error(parsed.result));
          } else {
            resolve(parsed.structured_output);
          }
        } catch (e) {
          reject(new Error(stderr.trim() || `Exit code ${code}`));
        }
      });

      proc.on("error", (err) => { clearTimeout(timeout); reject(err); });

      // Guard stdin writes — without this, EPIPE crashes the server process
      proc.stdin.on("error", () => {});
      proc.stdin.write(`${task}\n\n${item}`);
      proc.stdin.end();
    }));

  const results = outcomes.map((o, i) =>
    o.status === "fulfilled"
      ? { item: items[i], data: o.value }
      : { item: items[i], error: o.reason.message }
  );

  res.json({ results });
});
```

---

## Stream-JSON Event Types

When using `--output-format stream-json --verbose --include-partial-messages`,
Claude emits newline-delimited JSON events. The full event sequence is:

```
system (init) → stream_event (message_start) → stream_event (content_block_start)
→ stream_event (content_block_delta) ×N → stream_event (content_block_stop)
→ assistant (complete block) → stream_event (message_delta) → stream_event (message_stop)
→ rate_limit_event → result
```

Without `--include-partial-messages`, only `system`, `assistant`, and `result`
events appear — no token-level streaming. Always use `--include-partial-messages`
for streaming patterns.

| Event Type | Shape | What It Means | Forward? |
|------------|-------|---------------|----------|
| `system` | `{type:"system", subtype:"init", session_id, model, tools, ...}` | Session started | Optional (extract session_id) |
| `stream_event` | `{type:"stream_event", event:{type:"content_block_delta", delta:{text:"..."}}}` | Incremental token (requires `--include-partial-messages`) | Yes (live text) |
| `assistant` | `{type:"assistant", message:{content:[{type:"text",text:"..."}, {type:"tool_use",...}], stop_reason:"end_turn"|"tool_use"|null}}` | Complete message with text and/or tool calls | Tool use only (text already streamed via `stream_event`) |
| `tool_result` | `{type:"tool_result", tool_name, content, is_error}` | Tool execution completed (only appears when tools are used) | Optional (show tool output or detect tool failures via `is_error`) |
| `compact` | `{type:"compact"}` | Context window compacted (only in long sessions) | No (internal) |
| `rate_limit_event` | `{type:"rate_limit_event", rate_limit_info:{status, utilization, rateLimitType, isUsingOverage, resetsAt}}` | Rate limit status update | No (but log it — if `utilization` is high, consider adding delays between spawns) |
| `result` | `{type:"result", subtype:"success"|"error_max_turns", is_error, stop_reason:"end_turn"|"max_turns", session_id, num_turns, duration_ms, total_cost_usd}` | Session complete | Yes (done signal) |

**Notes on specific fields:**
- `stop_reason` appears on both `assistant` events (per-message) and `result` events
  (per-session). On `result`, verified values: `"end_turn"` for normal completion.
  Use `subtype` for programmatic branching (`"error_max_turns"` vs `"success"`);
  `stop_reason` is available if you need finer distinctions.
- `total_cost_usd` is present on `result` even for subscription users. It reflects
  internal accounting but is not meaningful for billing — do not surface it in the UI.
- `rate_limit_event` appears between the last `assistant` and `result` events.
  It's informational — no action needed unless `utilization` is consistently high,
  in which case add a small delay between concurrent spawns to avoid hitting limits.
- `tool_result` and `compact` are carried forward from the existing documentation.
  They were not re-verified (the 2026-03-08 tests used `--tools ""`
  and short prompts, so neither event would have appeared). Their shapes are
  presumed accurate — they almost certainly exist when tools are active or
  context compaction triggers.

**Extended thinking models** (e.g., haiku-4.5) work correctly with this
approach. With `--include-partial-messages`, extended thinking models emit
`content_block_delta` events for thinking content, but these have no
`delta.text` field (they carry `delta.thinking` instead) — the
`event.event?.delta?.text` check returns `undefined` and naturally skips them.
Real text tokens stream normally via `stream_event` for all models.
The `assistant` event contains the full thinking block (for logging) followed
by the text block — but since text was already streamed, only forward
`tool_use` from `assistant` events. Do NOT fall back to forwarding
`assistant` text blocks — this is unnecessary with `--include-partial-messages`
and would cause duplicate text.

**Extracting text and tool use:**

```typescript
// With --include-partial-messages: text arrives via stream_event, not assistant.
// Only extract tool_use from assistant events.
if (event.type === "stream_event" && event.event?.delta?.text) {
  // Token-by-token text — forward to frontend
} else if (event.type === "assistant" && event.message?.content) {
  for (const block of event.message.content) {
    if (block.type === "tool_use") {
      // Claude is calling a tool: block.name, block.input
      // Forward to frontend for progress indication
    }
    // DO NOT forward block.type === "text" — it duplicates stream_event tokens
  }
}
```

**Detecting max-turns exhaustion:** When `--max-turns` is exceeded, the `result`
event has `subtype: "error_max_turns"` and `is_error: true`; check `subtype` first.
It's easy to treat it as a hard error when it's really a "incomplete / hit turn limit"
warning:

```typescript
if (event.type === "result") {
  gotResult = true;
  if (event.subtype === "error_max_turns") {
    // is_error: true; check subtype first — surface as an "incomplete / hit turn limit" warning, not a hard error
    send({ type: "warning", message: "Task incomplete — reached turn limit" });
  } else if (event.is_error) {
    send({ type: "error", message: event.result });
  } else {
    send({ type: "done" });
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

---

## Frontend Integration

### Streaming Text Display

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
  let gotResult = false; // mirror the server-side guard

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split("\n\n");
    buffer = lines.pop() ?? ""; // keep incomplete chunk; ?? "" avoids rare undefined
    for (const line of lines) {
      if (!line.startsWith("data: ")) continue;
      try {
        const data = JSON.parse(line.slice(6));
        if (data.type === "token") output.textContent += data.text;
        else if (data.type === "done") { gotResult = true; /* stream complete */ }
        else if (data.type === "warning") { gotResult = true; output.textContent += `\n[Warning: ${data.message}]`; }
        else if (data.type === "error") { gotResult = true; output.textContent += `\n[Error: ${data.message}]`; }
      } catch { /* malformed SSE data — skip */ }
    }
  }

  // If the stream ended without a done/warning/error, the server closed early
  // (crash, network drop, or uncaught exception). Surface it rather than silently stopping.
  if (!gotResult) {
    output.textContent += "\n[Error: Stream ended before task completed — please retry]";
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

### Structured Result Rendering

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

---

## Error Surfacing Checklist

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
   check whether Claude flagged the run as failed. Check `subtype` before `is_error`
   so `error_max_turns` surfaces as a warning, not a hard error:
   ```typescript
   if (event.type === "result") {
     gotResult = true;
     if (event.subtype === "error_max_turns") {
       // is_error: true; check subtype first — surface as an "incomplete / hit turn limit" warning
       res.write(`data: ${JSON.stringify({ type: "warning", message: "Task incomplete — reached turn limit" })}\n\n`);
     } else if (event.is_error) {
       res.write(`data: ${JSON.stringify({ type: "error", message: event.result })}\n\n`);
     } else {
       res.write(`data: ${JSON.stringify({ type: "done", data: event.structured_output })}\n\n`);
     }
   }
   ```

If any of these three is missing, the app will fail silently in at least
one failure mode.

---

## Input Validation

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

---

## Temp Directory Cleanup

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

---

## Handling File Uploads and Drops

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
      // Use path.basename to strip any directory components from f.name.
      // Without this, f.name = "../../etc/x" would escape tempDir.
      const safeName = path.basename(f.name) || "upload";
      const filePath = path.join(tempDir, safeName);
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
