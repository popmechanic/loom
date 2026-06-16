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
- [Pattern: Reconnect-Safe Streaming](#pattern-reconnect-safe-streaming)
- [Pattern: Interrupting a Run](#pattern-interrupting-a-run)
- [Pattern: Validate Before Reloading a Preview](#pattern-validate-before-reloading-a-preview)
- [Pattern: Honest Progress for Slow Turns](#pattern-honest-progress-for-slow-turns)
- [Context-Window Management for Long-Lived Bridges](#context-window-management-for-long-lived-bridges)
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

## Pattern: Reconnect-Safe Streaming

The SSE and WebSocket patterns above tie one stream to one HTTP request and one
process. That's fine until the page goes away — and a browser reload (or a second
tab, or a dropped phone connection) is the single most common thing a user does.
When the request ends, the stream ends; with `res.on("close")` cleanup it can even
kill the run mid-flight. The fix is to stop streaming *from the request* and start
streaming *from a log*.

One background loop owns the Claude process and parses its output exactly once.
Every parsed event is appended to a server-side append-only log with a monotonic
id. Browsers are pure subscribers: they connect, replay everything after their
last-seen id, then receive live events. A reload reconnects, replays what it
missed, and continues — the run never noticed.

```typescript
// Builds on Server Setup + Shared Utilities (createStreamParser, spawnEnvForUser).
import { spawn, ChildProcess } from "child_process";
import { v4 as uuidv4 } from "uuid";

// One event log per run. Monotonic ids; a ring buffer bounds memory. Swap the
// array for a DB table or Redis stream when events must survive a server restart.
interface LoggedEvent { id: number; data: unknown; }
interface Run {
  proc: ChildProcess;
  events: LoggedEvent[];
  nextId: number;
  firstId: number;            // oldest id still retained — for overflow detection
  subscribers: Set<(e: LoggedEvent) => void>;
  done: boolean;
  interrupted?: boolean;      // set by the interrupt pattern below
  ownerSessionId: string;
  graceTimer?: NodeJS.Timeout;
}
const RING = 2000;
const GRACE_MS = 30_000;       // keep a run alive this long after the last client leaves
const runs = new Map<string, Run>();

function append(run: Run, data: unknown) {
  const evt = { id: run.nextId++, data };
  run.events.push(evt);
  if (run.events.length > RING) { run.events.shift(); run.firstId = run.events[0].id; }
  for (const send of run.subscribers) send(evt);
}

// Start a run: spawn once, parse once, append to the log. Returns a runId.
app.post("/api/runs", requireAuth, async (req: any, res) => {
  try { await refreshSessionIfNeeded(req.userSession); }
  catch { return res.status(401).json({ error: "Session expired — please sign in again" }); }

  const runId = uuidv4();
  // detached: true so process.kill(-pid) can later target the whole tool subtree.
  const proc = spawn("claude", [
    "-p", "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--permission-mode", "dontAsk", "--allowedTools", "Read,Glob,Grep,Bash",
    "--max-turns", "15", "--model", "sonnet", "--no-session-persistence",
    req.body.task
  ], { detached: true, env: spawnEnvForUser(req.userSession) });

  const run: Run = {
    proc, events: [], nextId: 1, firstId: 1, subscribers: new Set(),
    done: false, ownerSessionId: req.cookies[SESSION_COOKIE_NAME],
  };
  runs.set(runId, run);
  res.json({ runId });   // client immediately opens GET /api/runs/:id/events

  const parse = createStreamParser((event) => {
    if (event.type === "stream_event" && event.event?.delta?.text) {
      append(run, { type: "token", text: event.event.delta.text });
    } else if (event.type === "assistant" && event.message?.content) {
      for (const block of event.message.content) {
        if (block.type === "tool_use") append(run, { type: "tool", name: block.name });
      }
    } else if (event.type === "result") {
      // check subtype before is_error — error_max_turns is a warning, not a failure
      if (event.subtype === "error_max_turns") append(run, { type: "warning", message: "Task incomplete — reached turn limit" });
      else if (event.is_error) append(run, { type: "error", message: event.result });
      else append(run, { type: "done" });
    }
  });
  proc.stdout.on("data", parse);
  proc.stderr.on("data", (c) => { const m = c.toString().trim(); if (m) console.error(`[claude stderr] ${m}`); });
  proc.on("close", (code) => {
    // A user interrupt is not a failure — the interrupt pattern already logged it.
    if (run.nextId === 1 && !run.interrupted) {
      append(run, { type: "error", message: `Claude exited with code ${code} before producing output` });
    }
    run.done = true;
    append(run, { type: "closed" });
  });
});

// Subscribe: replay after the cursor, then stream live. Survives reloads.
app.get("/api/runs/:id/events", requireAuth, (req: any, res) => {
  const run = runs.get(req.params.id);
  if (!run) return res.status(404).end();
  if (run.ownerSessionId !== req.cookies[SESSION_COOKIE_NAME]) return res.status(403).end();

  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");
  res.flushHeaders();

  // Cursor: ?after=<id>, or the Last-Event-ID header the browser auto-resends on
  // reconnect (it remembers the last `id:` it saw). Either way, replay the gap.
  const after = Number(req.query.after ?? req.headers["last-event-id"] ?? 0);

  const write = (evt: LoggedEvent) => {
    if (res.writableEnded) return;
    try { res.write(`id: ${evt.id}\ndata: ${JSON.stringify(evt.data)}\n\n`); } catch { /* client gone */ }
  };

  // If the cursor predates the oldest retained event, the ring buffer overflowed and
  // the client missed events. Tell it explicitly so it can refetch full state rather
  // than silently rendering a hole.
  if (after > 0 && after < run.firstId - 1) {
    try { res.write(`data: ${JSON.stringify({ type: "resync" })}\n\n`); } catch {}
  }
  for (const evt of run.events) if (evt.id > after) write(evt);

  if (run.done) return res.end();   // nothing live left — replay was the whole story

  run.subscribers.add(write);
  if (run.graceTimer) { clearTimeout(run.graceTimer); run.graceTimer = undefined; }

  const heartbeat = setInterval(() => {
    try { res.write(`:heartbeat\n\n`); } catch { clearInterval(heartbeat); }
  }, 5000);

  res.on("close", () => {
    clearInterval(heartbeat);
    run.subscribers.delete(write);
    // Last client left: don't kill the process — keep it alive briefly so a refresh
    // reconnects to the same run instead of starting over.
    if (run.subscribers.size === 0 && !run.done) {
      run.graceTimer = setTimeout(() => {
        if (run.subscribers.size === 0 && !run.proc.killed) {
          try { process.kill(-run.proc.pid!, "SIGTERM"); } catch { run.proc.kill(); }
        }
      }, GRACE_MS);
    }
  });
});
```

**Why this is the default for anything non-trivial:** the per-request SSE pattern is
fine for short one-shots, but the moment a run takes long enough that a user might
reload, switch tabs, or lose signal, the event log is what keeps the experience
whole. It also gives you multi-tab for free (every tab is just another subscriber)
and a natural place to persist transcripts.

**The four load-bearing pieces:**
- **Monotonic ids + cursor replay** — the client sends its last-seen id (via `?after=`
  or the `Last-Event-ID` header) and gets exactly the gap. This is what makes reloads
  seamless.
- **Grace period before teardown** — without it, the `close` that fires on reload kills
  the process before the reload's new connection arrives. With it, the run survives the
  blink between disconnect and reconnect.
- **Resync marker on overflow** — a bounded ring buffer can age out events a slow client
  hasn't seen yet. Emitting a `resync` event turns a silent data hole into a recoverable
  "refetch full state" signal.
- **Snapshot-then-live (optional upgrade)** — for stateful apps, send a full state
  snapshot first, then live deltas. The snapshot *is* the catch-up cache, so subscribers
  never need to replay from event zero.

**Don't do this:**
- Don't reuse the per-request `res` as the stream owner. The owner is the background
  parser writing to the log; `res` objects come and go as clients connect and reconnect.
- Don't kill the process on the first `res.on("close")` — that defeats the entire
  pattern. Only the grace timer (or an explicit interrupt) kills it.
- Don't forget the `ownerSessionId` check on the subscribe endpoint — a run id is a
  capability; without the check, any authenticated user could read another user's stream.

**Frontend:** consume `/api/runs/:id/events` with `fetch()` + `ReadableStream` (or
`EventSource` for GET) exactly as in [Frontend Integration](#frontend-integration),
but persist the last `id:` seen (e.g. in `sessionStorage`) and pass it as `?after=` on
reconnect. On a `{type:"resync"}` event, clear local state and refetch the run's current
state. Store the `runId` (in the URL or `sessionStorage`) so a full page reload can
re-attach to the in-flight run.

---

## Pattern: Interrupting a Run

A real Claude Code session stops on Ctrl+C. A web app should give the user the same
power — a Stop button that actually halts a runaway or wrong-headed agent. Shipping
an app with no interrupt (or a button that silently does nothing) is a real UX
regression, not a minor omission: the user's only recourse becomes closing the tab.

Interrupt by signalling the process *group*, not just the direct child. Send `SIGINT`
first so Claude can finish the current tool call and emit a final `result`; escalate to
`SIGKILL` only if it ignores the signal (a stuck `Bash` grandchild can keep the group
alive). This builds on the run registry from the Reconnect-Safe Streaming pattern.

```typescript
app.post("/api/runs/:id/interrupt", requireAuth, (req: any, res) => {
  const run = runs.get(req.params.id);
  if (!run) return res.status(404).json({ error: "No such run" });
  if (run.ownerSessionId !== req.cookies[SESSION_COOKIE_NAME]) return res.status(403).json({ error: "Forbidden" });
  if (run.done) return res.json({ ok: true, alreadyDone: true });

  // Mark it interrupted BEFORE signalling, so the close handler treats the non-zero
  // exit as expected and the UI shows "Stopped", not "Error".
  run.interrupted = true;
  append(run, { type: "interrupted" });

  // detached: true (set at spawn) makes -pid address the whole group, so SIGINT
  // reaches Bash/MCP tool subtrees too — not just the claude process.
  try { process.kill(-run.proc.pid!, "SIGINT"); } catch { run.proc.kill("SIGINT"); }

  // Escalate if SIGINT is ignored within a grace window.
  const force = setTimeout(() => {
    if (!run.proc.killed) {
      try { process.kill(-run.proc.pid!, "SIGKILL"); } catch { run.proc.kill("SIGKILL"); }
    }
  }, 5000);
  run.proc.on("close", () => clearTimeout(force));

  res.json({ ok: true });
});
```

**Interrupt vs failure.** The distinction matters for the UI. A user stop and a model
overload both end in a non-zero exit, but they should not look the same to the person
watching. Setting `run.interrupted` and emitting an `interrupted` event lets the
frontend render "Stopped" cleanly, while the `close` handler in the streaming pattern
skips its "exited before producing output" error for interrupted runs.

**Frontend:** a Stop button (shown while the run is live) `POST`s to
`/api/runs/:id/interrupt`. The client doesn't wait on the response to update — the
`interrupted` event already arriving over the event stream is the source of truth, so
the button reacts the same whether the user has one tab open or three.

**WebSocket variant:** if you're on the WebSocket session pattern instead, the same
logic applies — on an `{action:"interrupt"}` message, `process.kill(-activeProc.pid!, "SIGINT")`
the active process and let the `close` handler null it out so the next turn respawns.

---

## Pattern: Validate Before Reloading a Preview

Apps that generate code or markup and render it live (code-generating agents, visual
builders, live-preview editors) need to guard the preview reload. When Claude writes or
edits a rendered file, a mid-stream or truncated write produces malformed output. Without
a validation step that bad output goes straight into the preview iframe — white-screening
it or worse, causing the page to throw — instead of degrading gracefully.

**The fix:** after each `tool_result` event that targets the rendered file, syntax-check
the file before swapping it into the preview. If it passes, emit `preview_reload`; if not,
keep the last-known-good render and emit `preview_reload_failed` with the error. Also
mtime-gate it so back-to-back tool calls on different files don't produce spurious reloads.

Derived from: `tool_result` events (the stream event that signals a Write/Edit completed).
Source: VibesOS `ws.ts` / `claude-bridge.ts`.

```typescript
import { statSync, readFileSync } from "fs";
import ts from "typescript";

const PREVIEW_FILE = "app/index.tsx"; // the file driving the live preview

let lastMtime = 0;
let lastGoodSource = ""; // last source that validated successfully

function validateAndMaybeReload(send: (payload: unknown) => void) {
  let mtime: number;
  try {
    mtime = statSync(PREVIEW_FILE).mtimeMs;
  } catch {
    return; // file doesn't exist yet — skip
  }
  if (mtime === lastMtime) return; // no real change
  lastMtime = mtime;

  let source: string;
  try {
    source = readFileSync(PREVIEW_FILE, "utf8");
  } catch (err) {
    send({ type: "preview_reload_failed", error: "Could not read file" });
    return;
  }

  // Syntax-check with the TypeScript compiler (or swap in any transpiler).
  // transpileModule doesn't do type-checking — just catches parse-level errors
  // (unclosed tags, truncated JSON, missing braces) quickly.
  try {
    const result = ts.transpileModule(source, {
      compilerOptions: { jsx: ts.JsxEmit.React, module: ts.ModuleKind.ESNext },
    });
    if (result.diagnostics && result.diagnostics.length > 0) {
      const msg = result.diagnostics.map((d) => ts.flattenDiagnosticMessageText(d.messageText, "\n")).join("; ");
      send({ type: "preview_reload_failed", error: msg });
      return;
    }
    lastGoodSource = source;
    send({ type: "preview_reload" }); // client swaps the iframe src / hot-reloads
  } catch (err: any) {
    // Hard parse failure — keep last-known-good, tell the client
    send({ type: "preview_reload_failed", error: err.message ?? "Parse error" });
  }
}

// Hook into the stream parser. Call validateAndMaybeReload after each tool_result
// whose tool_name is "Write" or "Edit".
const parse = createStreamParser((event) => {
  // ... token/result handling unchanged ...
  if (
    event.type === "tool_result" &&
    (event.tool_name === "Write" || event.tool_name === "Edit")
  ) {
    validateAndMaybeReload(send);
  }
});
```

**Why mtime-gating matters:** Claude can call Write several times in one turn. Without
the mtime check, each `tool_result` triggers a reload attempt — including ones where the
file content didn't actually change. The mtime gate fires exactly once per real change.

**Fallback on parse failure:** `lastGoodSource` gives you the last successfully validated
content. Serve it as a fallback (e.g., write it to a `.last-good.tsx` sidecar) so the
preview stays functional while Claude is mid-edit. The `preview_reload_failed` event lets
the frontend show a non-blocking indicator ("Saving…") rather than a blank screen.

---

## Pattern: Honest Progress for Slow Turns

When Claude is doing a single large Write with no streaming text, or is thinking hard
before speaking, the frontend can go silent for 10–90 seconds with no feedback. A
spinner that says "Working…" for the whole duration erodes trust — users can't tell if
the app is alive. The right answer is a progress signal that reflects genuine engine
activity but never lies by claiming completion.

**Four pieces:**

1. **Asymptotic ratcheted progress value** — never decreases, never reaches 100% until
   the `result` event arrives, caps near 95% mid-run so it visibly stalls rather than
   jumping to done.
2. **Activity signals** — derive "underway" from tool-use count and `input_json_delta`
   byte accumulation so the bar moves meaningfully while a large Write is building.
3. **Silence-escalating status labels** — show "Thinking…" by default, escalate to
   "Thinking hard…" at ~45 s without text tokens, then "Still working…" at ~90 s.
4. **Hard-silence timeout** — a safety backstop distinct from `--max-turns`: kill the
   process if it has emitted nothing for N seconds. This catches hangs that
   `--max-turns` never reaches (the process is alive but stuck, not taking turns).

Source: VibesOS `event-translator.ts`, `claude-bridge.ts`.

```typescript
// Track progress state alongside the stream parser.
interface ProgressState {
  value: number;       // 0–1; shown as (value * 100)%
  toolsUsed: number;
  bytesAccumulated: number;
  lastActivityMs: number;
  silenceTimer?: NodeJS.Timeout;
  statusLabel: string;
}

function calcProgress(state: ProgressState): number {
  // Ratchet: never decrease, approach 95% asymptotically based on known signals.
  // Base: 5% just for starting.
  // +30% spread across first 5 tool calls (encourages early movement).
  // +30% for byte accumulation (large writes need visible progress).
  // Cap at 0.95 — the final 5% only unlocks on the result event.
  const toolContrib = Math.min(state.toolsUsed / 5, 1) * 0.3;
  const byteContrib = Math.min(state.bytesAccumulated / 50_000, 1) * 0.3;
  const raw = 0.05 + toolContrib + byteContrib;
  return Math.max(state.value, Math.min(raw, 0.95)); // ratchet: never decrease
}

function updateSilenceLabel(state: ProgressState, send: (p: unknown) => void) {
  const silentMs = Date.now() - state.lastActivityMs;
  if (silentMs > 90_000) {
    state.statusLabel = "Still working…";
  } else if (silentMs > 45_000) {
    state.statusLabel = "Thinking hard…";
  } else {
    state.statusLabel = "Thinking…";
  }
  send({ type: "progress", value: state.value, label: state.statusLabel });
}

const HARD_SILENCE_MS = 120_000; // kill process if nothing at all for 2 minutes

function startHardSilenceTimer(proc: ReturnType<typeof spawn>, send: (p: unknown) => void) {
  return setTimeout(() => {
    send({ type: "error", message: "Claude stopped responding — process killed" });
    try { process.kill(-proc.pid!, "SIGKILL"); } catch { proc.kill(); }
  }, HARD_SILENCE_MS);
}

// Integrate into createStreamParser:
const progressState: ProgressState = {
  value: 0, toolsUsed: 0, bytesAccumulated: 0,
  lastActivityMs: Date.now(), statusLabel: "Thinking…",
};

// Poll silence labels every 10 s.
const silencePoll = setInterval(() => updateSilenceLabel(progressState, send), 10_000);
let hardSilenceTimer = startHardSilenceTimer(proc, send);

const parse = createStreamParser((event) => {
  // Reset hard-silence timer on any activity.
  clearTimeout(hardSilenceTimer);
  hardSilenceTimer = startHardSilenceTimer(proc, send);
  progressState.lastActivityMs = Date.now();
  progressState.statusLabel = "Thinking…"; // reset escalation on any event

  if (event.type === "stream_event") {
    if (event.event?.delta?.text) {
      send({ type: "token", text: event.event.delta.text });
    }
    // input_json_delta: tool input arriving incrementally (large Write filling in)
    if (event.event?.delta?.type === "input_json_delta") {
      progressState.bytesAccumulated += (event.event.delta.partial_json ?? "").length;
    }
  } else if (event.type === "assistant" && event.message?.content) {
    for (const block of event.message.content) {
      if (block.type === "tool_use") {
        progressState.toolsUsed++;
        send({ type: "tool", name: block.name });
      }
    }
  } else if (event.type === "result") {
    clearInterval(silencePoll);
    clearTimeout(hardSilenceTimer);
    send({ type: "progress", value: 1, label: "Done" }); // final unlock to 100%
    if (event.subtype === "error_max_turns") {
      send({ type: "warning", message: "Task incomplete — reached turn limit" });
    } else if (event.is_error) {
      send({ type: "error", message: event.result });
    } else {
      send({ type: "done" });
    }
  }

  progressState.value = calcProgress(progressState);
  send({ type: "progress", value: progressState.value, label: progressState.statusLabel });
});
```

**Frontend:** render progress value as a bar width (`style="width: ${(value*100).toFixed(1)}%"`)
and the label as a status line. The ratchet guarantees the bar only moves forward,
removing the unsettling "progress went backwards" effect that naive time-based estimates
produce. On the `done` event, snap value to 1 and animate to full-width.

**The hard-silence timeout vs `--max-turns`:** `--max-turns` limits how many agent turns
Claude takes, but a process can hang inside a single turn (a stuck Bash command, a
network call that never returns). The hard-silence timer kills the process by wall-clock
silence, which `--max-turns` never catches. Set it generously (2–5 minutes) to avoid
false positives on legitimately slow but live operations.

---

## Context-Window Management for Long-Lived Bridges

A long-lived `claude -p` process (persistent session, duplex bridge) eventually fills its
context window. When it does, the next turn fails rather than answering. Track `input_tokens`
from the `result` event against `contextWindow − maxOutputTokens` from `modelUsage` or
known model limits; at around 70%, kill the process and respawn it, replaying a windowed
history prelude as the first user message.

Tie the history replay to the respawn itself — not to a conversation-switch event. An
untracked respawn (crash, OOM, accidental restart) answers with no context unless you
replay unconditionally on every fresh process.

For the full pattern (append-only event log, projection tables, atomic resume-pointer
writes) see `references/advanced-patterns.md#persistent-session-long-lived-process`.
Source: metrc `budget.ts`.

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
