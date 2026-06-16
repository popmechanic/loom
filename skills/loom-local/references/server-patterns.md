# Server Patterns Reference (Bun)

Core server-side code patterns for Loom Local apps — shared utilities,
communication patterns, streaming event handling, and frontend integration.
All patterns use `Bun.serve()` with zero external dependencies.

## Table of Contents

- [Shared Utilities](#shared-utilities)
  - [cleanEnv()](#cleanenv)
  - [createStreamParser()](#createstreamparser)
- [Server Setup](#server-setup)
- [Pattern: REST Endpoint](#pattern-rest-endpoint)
- [Pattern: SSE Streaming](#pattern-sse-streaming)
- [Pattern: WebSocket Session](#pattern-websocket-session)
- [Pattern: Reconnect-Safe Streaming](#pattern-reconnect-safe-streaming)
- [Pattern: Interrupting a Run](#pattern-interrupting-a-run)
- [Pattern: Background Job with Progress](#pattern-background-job-with-progress)
- [Pattern: Parallel Analysis](#pattern-parallel-analysis)
- [Multi-Turn via Long-Lived Process](#multi-turn-via-long-lived-process)
- [Frontend Integration](#frontend-integration)
  - [Streaming Text Display](#streaming-text-display)
  - [Structured Result Rendering](#structured-result-rendering)
- [Error Surfacing Checklist](#error-surfacing-checklist)

---

## Shared Utilities

Every pattern below uses these two helpers. Define them once at the top of
the server file.

### cleanEnv()

Remove nesting guards so `claude -p` can start.

When the server runs inside Claude Code (which it often does during
development), two environment variables — `CLAUDECODE` and
`CLAUDE_CODE_ENTRYPOINT` — tell Claude it's already running and block nested
processes. Remove exactly these two. Do NOT filter all `CLAUDE*` vars — that
kills auth tokens and feature flags.

If the server runs inside **cmux** (the Claude terminal app), additional
`CMUX_*` surface identifiers trigger nesting detection. These are terminal-state
vars, not auth — safe to remove.

```typescript
function cleanEnv(): Record<string, string | undefined> {
  const env = { ...process.env };
  delete env.CLAUDECODE;
  delete env.CLAUDE_CODE_ENTRYPOINT;
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
gets silently discarded.

```typescript
function createStreamParser(onEvent: (event: any) => void) {
  const decoder = new TextDecoder();
  let buffer = "";

  return (chunk: Uint8Array) => {
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

---

## Server Setup

Every pattern below assumes this baseline `Bun.serve()` setup. Define it once
at the top of the server file, alongside the shared utilities.

> **Loopback is not a security boundary.** Binding `127.0.0.1` keeps this off
> the LAN, but any process or browser tab on this machine can still reach the
> port — and this server runs `claude -p` with `Bash` and `dontAsk`, i.e.
> arbitrary local command execution. Fine for a single-user local tool; if you
> need a hard boundary, add an `Origin` check or a random per-launch token, and
> never port-forward it.

```typescript
import { resolve } from "path";
const PORT = process.env.PORT !== undefined ? Number(process.env.PORT) : 3456;
const PUBLIC_ROOT = resolve("./public");

const server = Bun.serve({
  port: PORT,
  hostname: "127.0.0.1",
  async fetch(req) {
    const url = new URL(req.url);

    // Static files
    if (url.pathname === "/" || !url.pathname.startsWith("/api/")) {
      const rel = url.pathname === "/" ? "index.html" : decodeURIComponent(url.pathname).slice(1);
      const filePath = resolve(PUBLIC_ROOT, rel);
      if (filePath !== PUBLIC_ROOT && !filePath.startsWith(PUBLIC_ROOT + "/")) {
        return new Response("Forbidden", { status: 403 });
      }
      const file = Bun.file(filePath);
      if (await file.exists()) return new Response(file);
      return new Response("Not found", { status: 404 });
    }

    // API routes (add patterns below)
    return new Response("Not found", { status: 404 });
  },
});

console.log(`Server running on http://127.0.0.1:${PORT}`);
```

---

## Pattern: REST Endpoint

Someone triggers an action → server calls Claude → returns JSON.

```typescript
// Inside the fetch handler, add:
if (url.pathname === "/api/analyze" && req.method === "POST") {
  const { content, task } = await req.json();

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

  const proc = Bun.spawnSync(["claude",
    "-p", "--model", "sonnet", "--output-format", "json",
    "--permission-mode", "dontAsk",
    "--json-schema", schema, "--tools", "", "--no-session-persistence"
  ], { stdin: new TextEncoder().encode(`${task}\n\n${content}`), env: cleanEnv() });

  if (proc.exitCode !== 0) {
    const stderr = proc.stderr.toString().trim();
    console.error(`[claude] exit ${proc.exitCode}, stderr: ${stderr}`);
    return Response.json({ error: stderr || "Claude process failed" }, { status: 502 });
  }

  try {
    const parsed = JSON.parse(proc.stdout.toString());
    if (parsed.is_error) {
      return Response.json({ error: parsed.result }, { status: 502 });
    }
    return Response.json(parsed.structured_output);
  } catch {
    return Response.json({ error: "Malformed Claude output" }, { status: 502 });
  }
}
```

**Don't do this:**
- Don't read `structured_output` without checking `is_error` first — when
  Claude hits a tool failure, `structured_output` is `null`.
- Don't skip `cleanEnv()` — without it, Claude refuses to start when the server
  runs inside Claude Code.

---

## Pattern: SSE Streaming

Someone triggers a task → server streams Claude's output token-by-token.

```typescript
if (url.pathname === "/api/stream" && req.method === "POST") {
  const { task } = await req.json();

  const proc = Bun.spawn(["claude",
    "-p", "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--permission-mode", "dontAsk", "--allowedTools", "Read,Glob,Grep,Bash",
    "--max-turns", "15",
    "--model", "sonnet", "--no-session-persistence",
    task
  ], { stdout: "pipe", stderr: "pipe", env: cleanEnv() });

  let gotResult = false;

  const stream = new ReadableStream({
    async start(controller) {
      const encode = (data: object) =>
        controller.enqueue(new TextEncoder().encode(`data: ${JSON.stringify(data)}\n\n`));

      const parse = createStreamParser((event) => {
        if (event.type === "stream_event" && event.event?.delta?.text) {
          encode({ type: "token", text: event.event.delta.text });
        } else if (event.type === "assistant" && event.message?.content) {
          for (const block of event.message.content) {
            if (block.type === "tool_use") {
              encode({ type: "tool", name: block.name });
            }
          }
        } else if (event.type === "result") {
          gotResult = true;
          if (event.subtype === "error_max_turns") {
            encode({ type: "warning", message: "Task incomplete — reached turn limit" });
          } else if (event.is_error) {
            encode({ type: "error", message: event.result });
          } else {
            encode({ type: "done" });
          }
        }
      });

      const stderrPromise = new Response(proc.stderr).text();
      const reader = proc.stdout.getReader();
      try {
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          parse(value);
        }
      } finally {
        if (!gotResult) {
          encode({ type: "error", message: "Process finished without producing a result" });
        }
        controller.close();
      }

      const stderrText = await stderrPromise;
      if (stderrText.trim()) console.error(`[claude stderr] ${stderrText.trim()}`);
    },
    cancel() {
      proc.kill();
    }
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      "Connection": "keep-alive",
    },
  });
}
```

**Don't do this:**
- Don't forward text from `assistant` events when using `--include-partial-messages`
  — the same text was already delivered token-by-token via `stream_event`. Only
  forward `tool_use` blocks from `assistant` events, or text will appear twice.
- Don't omit `--include-partial-messages` — without it, text arrives as a
  single complete block instead of token-by-token streaming.
- Don't skip the `gotResult` guard — if Claude exits without emitting a
  `result` event, the frontend gets no signal that something went wrong.

---

## Pattern: WebSocket Session

Persistent connection with multi-turn conversation and streaming. Bun has
built-in WebSocket support — no `ws` package needed.

```typescript
// Replace the Bun.serve() call with this expanded version:
const server = Bun.serve({
  port: PORT,
  hostname: "127.0.0.1",
  async fetch(req) {
    const url = new URL(req.url);

    // WebSocket upgrade
    if (url.pathname === "/ws") {
      const upgraded = server.upgrade(req, { data: { sessionId: crypto.randomUUID() } });
      if (!upgraded) return new Response("WebSocket upgrade failed", { status: 400 });
      return undefined as any;
    }

    // Static files
    if (url.pathname === "/" || !url.pathname.startsWith("/api/")) {
      const rel = url.pathname === "/" ? "index.html" : decodeURIComponent(url.pathname).slice(1);
      const filePath = resolve(PUBLIC_ROOT, rel);
      if (filePath !== PUBLIC_ROOT && !filePath.startsWith(PUBLIC_ROOT + "/")) {
        return new Response("Forbidden", { status: 403 });
      }
      const file = Bun.file(filePath);
      if (await file.exists()) return new Response(file);
      return new Response("Not found", { status: 404 });
    }

    // API routes
    return new Response("Not found", { status: 404 });
  },
  websocket: {
    open(ws) {
      (ws.data as any).isFirstTurn = true;
      (ws.data as any).activeProc = null;
    },
    async message(ws, raw) {
      const data = ws.data as any;
      if (data.activeProc) { ws.send(JSON.stringify({ type: "error", message: "busy" })); return; }
      let parsed; try { parsed = JSON.parse(raw as string); } catch { ws.send(JSON.stringify({ type: "error", message: "bad json" })); return; }
      const { payload } = parsed;

      // First turn: --session-id creates the session
      // Subsequent turns: --resume continues that specific session
      const sessionArgs = data.isFirstTurn
        ? ["--session-id", data.sessionId]
        : ["--resume", data.sessionId];

      let proc;
      try {
        proc = Bun.spawn(["claude",
          "-p", "--output-format", "stream-json", "--verbose",
          "--include-partial-messages",
          "--permission-mode", "dontAsk", "--allowedTools", "Read,Write,Edit,Bash,Glob,Grep",
          "--max-turns", "20",
          ...sessionArgs,
          "--model", "sonnet",
          payload.prompt
        ], { stdout: "pipe", stderr: "pipe", env: cleanEnv() });

        data.isFirstTurn = false;
        data.activeProc = proc;
        let gotResult = false;

        const parse = createStreamParser((event) => {
          if (event.type === "stream_event" && event.event?.delta?.text) {
            ws.send(JSON.stringify({ type: "token", text: event.event.delta.text }));
          } else if (event.type === "assistant" && event.message?.content) {
            for (const block of event.message.content) {
              if (block.type === "tool_use") {
                ws.send(JSON.stringify({ type: "tool", name: block.name, input: block.input }));
              }
            }
          } else if (event.type === "result") {
            gotResult = true;
            if (event.subtype === "error_max_turns") {
              ws.send(JSON.stringify({ type: "warning", message: "Task incomplete — reached turn limit" }));
            } else if (event.is_error) {
              ws.send(JSON.stringify({ type: "error", message: event.result }));
            } else {
              ws.send(JSON.stringify({ type: "done" }));
            }
          }
        });

        const stderrPromise = new Response(proc.stderr).text();
        const reader = proc.stdout.getReader();
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          parse(value);
        }

        if (!gotResult) {
          ws.send(JSON.stringify({ type: "error", message: "Process finished without producing a result" }));
        }

        const stderrText = await stderrPromise;
        if (stderrText.trim()) console.error(`[claude stderr] ${stderrText.trim()}`);
      } catch (e) {
        ws.send(JSON.stringify({ type: "error", message: String(e) }));
      } finally {
        if (proc && data.activeProc === proc) { proc.kill(); data.activeProc = null; }
      }
    },
    close(ws) {
      const data = ws.data as any;
      if (data.activeProc) data.activeProc.kill();
    },
  },
});
```

**Don't do this:**
- Don't combine `--session-id` with `--resume` — use `--session-id` on the
  first turn only, then `--resume <id>` on subsequent turns.
- Don't allow concurrent subprocess spawns per connection — reject messages
  while one is active.
- Don't forget to kill the subprocess when the WebSocket closes.

---

## Pattern: Reconnect-Safe Streaming

A plain per-request SSE stream dies on browser reload — the client gets half
a run and misses everything that happened while the page was reloading. The
fix: decouple the `claude -p` process lifetime from any browser connection.

**How it works:** A background loop owns the process and appends events to an
in-memory ring buffer with monotonic ids. Browsers subscribe and ask "give me
everything after id N" — by sending `?after=<id>` or the standard
`Last-Event-ID` header. They replay the backlog and then stream live. A 5s
heartbeat keeps connections from timing out. When the last subscriber
disconnects, start a grace-period timer (e.g., 30s) before killing the
process — a reload re-attaches instead of losing the run. Emit a synthetic
`resync` event when the cursor predates the oldest retained event so the client
knows to do a full reload instead of assuming it's up to date.

```typescript
const EVENT_LOG_SIZE = 500; // ring buffer capacity

interface LogEntry {
  id: number;
  data: object;
}

let nextId = 1;
const eventLog: LogEntry[] = [];
let activeProc: ReturnType<typeof Bun.spawn> | null = null;
let clientCount = 0;
let gracePeriodTimer: ReturnType<typeof setTimeout> | null = null;
const GRACE_MS = 30_000;

function appendEvent(data: object) {
  const entry: LogEntry = { id: nextId++, data };
  eventLog.push(entry);
  if (eventLog.length > EVENT_LOG_SIZE) eventLog.shift();
  return entry;
}

function eventsAfter(cursor: number): LogEntry[] {
  if (eventLog.length === 0) return [];
  const oldest = eventLog[0].id;
  // cursor predates the ring buffer — client needs resync
  if (cursor > 0 && cursor < oldest) {
    return [{ id: nextId - 1, data: { type: "resync" } }];
  }
  return eventLog.filter(e => e.id > cursor);
}

async function startRun(prompt: string) {
  if (activeProc) return; // already running

  // Detached process group so SIGINT reaches the whole tree
  activeProc = Bun.spawn(["claude",
    "-p", "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--permission-mode", "dontAsk", "--allowedTools", "Read,Glob,Grep,Bash",
    "--max-turns", "20", "--model", "sonnet", "--no-session-persistence",
    prompt
  ], {
    stdout: "pipe", stderr: "pipe",
    env: cleanEnv(),
    // detached: true is not a Bun.spawn option — use the pid for process group
  });

  const parse = createStreamParser((event) => {
    if (event.type === "stream_event" && event.event?.delta?.text) {
      appendEvent({ type: "token", text: event.event.delta.text });
    } else if (event.type === "assistant" && event.message?.content) {
      for (const block of event.message.content) {
        if (block.type === "tool_use") appendEvent({ type: "tool", name: block.name });
      }
    } else if (event.type === "result") {
      if (event.subtype === "error_max_turns") {
        appendEvent({ type: "warning", message: "Task incomplete — reached turn limit" });
      } else if (event.is_error) {
        appendEvent({ type: "error", message: event.result });
      } else {
        appendEvent({ type: "done" });
      }
    }
  });

  (async () => {
    const reader = activeProc!.stdout.getReader();
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      parse(value);
    }
    activeProc = null;
  })();
}

// SSE subscriber endpoint
if (url.pathname === "/api/run-stream" && req.method === "GET") {
  const cursor = Number(req.headers.get("last-event-id") || url.searchParams.get("after") || "0");

  if (clientCount === 0 && gracePeriodTimer) {
    clearTimeout(gracePeriodTimer);
    gracePeriodTimer = null;
  }
  clientCount++;

  const HEARTBEAT_MS = 5_000;
  let heartbeatTimer: ReturnType<typeof setInterval>;

  const stream = new ReadableStream({
    start(controller) {
      const enc = new TextEncoder();
      const send = (id: number, data: object) => {
        controller.enqueue(enc.encode(`id: ${id}\ndata: ${JSON.stringify(data)}\n\n`));
      };

      // Replay backlog
      for (const e of eventsAfter(cursor)) send(e.id, e.data);

      // Heartbeat
      heartbeatTimer = setInterval(() => {
        controller.enqueue(enc.encode(`: heartbeat\n\n`));
      }, HEARTBEAT_MS);

      // Drain new events by polling (simple approach — swap for a push-based
      // notifier in high-throughput apps)
      let lastSent = cursor;
      const poll = setInterval(() => {
        for (const e of eventsAfter(lastSent)) {
          send(e.id, e.data);
          lastSent = e.id;
        }
      }, 50);

      (stream as any)._cleanup = () => {
        clearInterval(heartbeatTimer);
        clearInterval(poll);
        clientCount--;
        if (clientCount === 0 && activeProc) {
          gracePeriodTimer = setTimeout(() => {
            activeProc?.kill();
            activeProc = null;
          }, GRACE_MS);
        }
      };
    },
    cancel() {
      (stream as any)._cleanup?.();
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      "Connection": "keep-alive",
    },
  });
}
```

The key insight: the process runs in the background regardless of how many
browser tabs are watching. Reload the page and you re-attach to the same run.

---

## Pattern: Interrupting a Run

Shipping no interrupt is a real UX regression. When Claude is doing a long
agentic run, users need a way to stop it.

The correct approach: send `SIGINT` to the process group, not just the process
itself. `SIGINT` lets Claude finish the current tool call and emit a clean
`result` event before exiting. If the process ignores it for ~5s, escalate to
`SIGKILL`. Disambiguate a user interrupt from a real failure in the UI — show
"Stopped" not "Error".

To reach the whole process group, spawn Claude with a dedicated process group:

```typescript
// In Node.js / Bun, Bun.spawn does not expose detached option directly.
// Use child_process from Bun's Node compat layer for process-group spawning:
import { spawn } from "child_process";

interface RunHandle {
  proc: ReturnType<typeof spawn>;
  pid: number;
  interrupted: boolean;
}

let activeRun: RunHandle | null = null;

function spawnDetached(args: string[]): RunHandle {
  const proc = spawn("claude", args, {
    stdio: ["ignore", "pipe", "pipe"],
    detached: true, // own process group — SIGINT reaches children
    env: cleanEnv() as NodeJS.ProcessEnv,
  });
  return { proc, pid: proc.pid!, interrupted: false };
}

// POST /api/interrupt — stop the active run
if (url.pathname === "/api/interrupt" && req.method === "POST") {
  if (!activeRun) return Response.json({ ok: false, reason: "no active run" });

  const run = activeRun;
  run.interrupted = true;
  appendEvent({ type: "status", message: "Stopping…" });

  try {
    // Negative pid targets the whole process group
    process.kill(-run.pid, "SIGINT");
  } catch {
    // process already gone — nothing to do
  }

  // Escalate to SIGKILL after 5s if it hasn't exited
  const killTimer = setTimeout(() => {
    try { process.kill(-run.pid, "SIGKILL"); } catch { /* already gone */ }
  }, 5_000);

  run.proc.once("exit", () => {
    clearTimeout(killTimer);
    // Emit a disambiguated event so the UI shows "Stopped" not "Error"
    appendEvent({ type: run.interrupted ? "interrupted" : "done" });
    activeRun = null;
  });

  return Response.json({ ok: true });
}
```

In the streaming handler, the `interrupted` flag lets the parser emit a
distinct `{ type: "interrupted" }` event instead of `{ type: "error" }`.
The frontend checks for `interrupted` and shows "Stopped by user" rather than
an error state.

**Don't do this:**
- Don't use `proc.kill()` alone — it sends `SIGTERM` to the top-level process
  and leaves tool subprocesses (e.g., a `Bash` command) running as orphans.
- Don't treat `SIGINT` as instant — give the process a few seconds to clean up
  before escalating to `SIGKILL`.

---

## Pattern: Background Job with Progress

Long-running task that reports progress via polling.

```typescript
const TIMEOUT_MS = 60000;
const JOB_TTL_MS = 5 * 60 * 1000; // evict finished jobs after 5 minutes
const jobs = new Map<string, { status: string; result?: any; error?: string; proc?: any; timeoutId?: any; finishedAt?: number }>();

// POST /api/jobs — start a job
if (url.pathname === "/api/jobs" && req.method === "POST") {
  const { task } = await req.json();
  const jobId = crypto.randomUUID();

  // Evict stale finished jobs
  const now = Date.now();
  for (const [id, job] of jobs) {
    if (job.finishedAt && now - job.finishedAt > JOB_TTL_MS) jobs.delete(id);
  }

  const proc = Bun.spawn(["claude",
    "-p", "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--model", "sonnet", "--max-turns", "20",
    "--permission-mode", "dontAsk",
    "--allowedTools", "Read,Bash,Glob,Grep",
    "--no-session-persistence"
  ], { stdout: "pipe", stderr: "pipe", stdin: "pipe", env: cleanEnv() });

  proc.stdin.write(task);
  proc.stdin.end();

  const timeoutId = setTimeout(() => {
    proc.kill();
    const job = jobs.get(jobId);
    if (job?.status === "running") {
      jobs.set(jobId, { status: "failed", error: "Timed out", finishedAt: Date.now() });
    }
  }, TIMEOUT_MS);

  jobs.set(jobId, { status: "running", proc, timeoutId });

  // Process in background
  (async () => {
    const stderrPromise = new Response(proc.stderr).text();

    const parse = createStreamParser((event) => {
      if (event.type === "result") {
        clearTimeout(timeoutId);
        if (event.subtype === "error_max_turns") {
          jobs.set(jobId, { status: "incomplete", result: event, finishedAt: Date.now() });
        } else if (event.is_error) {
          jobs.set(jobId, { status: "failed", error: event.result, finishedAt: Date.now() });
        } else {
          jobs.set(jobId, { status: "complete", result: event, finishedAt: Date.now() });
        }
      }
    });

    const reader = proc.stdout.getReader();
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      parse(value);
    }

    const stderrText = await stderrPromise;
    if (jobs.get(jobId)?.status === "running") {
      jobs.set(jobId, { status: "failed", error: stderrText.trim() || "Process exited unexpectedly", finishedAt: Date.now() });
    } else if (stderrText.trim()) {
      console.error(`[claude stderr] ${stderrText.trim()}`);
    }
  })();

  return Response.json({ jobId });
}

// GET /api/jobs/:id — poll status
if (url.pathname.startsWith("/api/jobs/") && req.method === "GET") {
  const jobId = url.pathname.split("/api/jobs/")[1];
  const job = jobs.get(jobId);
  if (!job) return Response.json({ status: "not_found" });
  const { proc: _proc, timeoutId: _tid, ...safe } = job;
  return Response.json(safe);
}
```

---

## Pattern: Parallel Analysis

Analyze multiple items concurrently, aggregate results.

```typescript
if (url.pathname === "/api/batch" && req.method === "POST") {
  const { items, task } = await req.json();
  const TIMEOUT_MS = 30000;
  const MAX_ITEMS = 20;
  const CONCURRENCY = 4;

  if (!Array.isArray(items) || items.length === 0) {
    return new Response(JSON.stringify({ error: "items must be a non-empty array" }), { status: 400 });
  }
  if (items.length > MAX_ITEMS) {
    return new Response(JSON.stringify({ error: `Too many items (max ${MAX_ITEMS})` }), { status: 400 });
  }

  const schema = JSON.stringify({
    type: "object",
    properties: { analysis: { type: "string" }, score: { type: "number" } },
    required: ["analysis", "score"]
  });

  function analyzeItem(item: string): Promise<any> {
    return new Promise<any>((resolve, reject) => {
      const proc = Bun.spawn(["claude",
        "-p", "--model", "haiku", "--output-format", "json",
        "--permission-mode", "dontAsk",
        "--json-schema", schema, "--tools", "", "--no-session-persistence"
      ], { stdout: "pipe", stderr: "pipe", stdin: "pipe", env: cleanEnv() });

      const timeout = setTimeout(() => { proc.kill(); reject(new Error("Timeout")); }, TIMEOUT_MS);

      proc.stdin.write(`${task}\n\n${item}`);
      proc.stdin.end();

      (async () => {
        try {
          const stdout = await new Response(proc.stdout).text();
          const stderr = await new Response(proc.stderr).text();
          const exitCode = await proc.exited;
          clearTimeout(timeout);

          if (exitCode !== 0) {
            reject(new Error(stderr.trim() || `Exit code ${exitCode}`));
            return;
          }
          const parsed = JSON.parse(stdout);
          if (parsed.is_error) reject(new Error(parsed.result));
          else resolve(parsed.structured_output);
        } catch (e) {
          clearTimeout(timeout);
          reject(e);
        }
      })();
    });
  }

  async function mapWithConcurrency<T, R>(
    arr: T[],
    limit: number,
    fn: (item: T) => Promise<R>
  ): Promise<PromiseSettledResult<R>[]> {
    const results: PromiseSettledResult<R>[] = new Array(arr.length);
    let nextIndex = 0;

    async function worker() {
      while (nextIndex < arr.length) {
        const i = nextIndex++;
        try {
          results[i] = { status: "fulfilled", value: await fn(arr[i]) };
        } catch (e: any) {
          results[i] = { status: "rejected", reason: e };
        }
      }
    }

    await Promise.all(Array.from({ length: Math.min(limit, arr.length) }, worker));
    return results;
  }

  const outcomes = await mapWithConcurrency(items, CONCURRENCY, analyzeItem);

  const results = outcomes.map((o, i) =>
    o.status === "fulfilled"
      ? { item: items[i], data: o.value }
      : { item: items[i], error: (o as PromiseRejectedResult).reason.message }
  );

  return Response.json({ results });
}
```

---

## Multi-Turn via Long-Lived Process

The WebSocket Session pattern above uses `--resume` to continue a session on
each new turn. A cleaner alternative for apps that need ongoing multi-turn
interaction or mid-run steering: keep one `claude -p --input-format stream-json`
process alive for the whole session and write each user turn as a newline-delimited
JSON message to stdin:

```typescript
import { spawn } from "child_process";

const proc = spawn("claude", [
  "-p", "--output-format", "stream-json", "--verbose",
  "--include-partial-messages",
  "--input-format", "stream-json",
  "--permission-mode", "dontAsk", "--allowedTools", "Read,Write,Edit,Bash,Glob,Grep",
  "--max-turns", "50", "--model", "sonnet",
], {
  stdio: ["pipe", "pipe", "pipe"],
  env: cleanEnv() as NodeJS.ProcessEnv,
});

function sendTurn(text: string) {
  const msg = JSON.stringify({ type: "user", message: { role: "user", content: text } });
  proc.stdin.write(msg + "\n");
}
```

Multi-turn and mid-run steering are the same write: call `sendTurn()` from a
WebSocket message handler, an HTTP endpoint, or a keyboard event. This avoids
the cold-start overhead of respawning and the path-traversal risk of managing
session IDs across turns.

The process stays alive until the app is done or explicitly killed. Pair this
with the Reconnect-Safe Streaming pattern to survive page reloads while the
long-lived process is running.

---

## Frontend Integration

### Streaming Text Display

`fetch()` + `ReadableStream` supports POST (CSRF-safe, keeps task data out
of URLs):

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
    buffer = lines.pop();
    for (const line of lines) {
      if (!line.startsWith("data: ")) continue;
      try {
        const data = JSON.parse(line.slice(6));
        if (data.type === "token") output.textContent += data.text;
        else if (data.type === "done") { /* stream complete */ }
        else if (data.type === "warning") output.textContent += `\n[Warning: ${data.message}]`;
        else if (data.type === "error") output.textContent += `\n[Error: ${data.message}]`;
      } catch { /* malformed SSE data — skip */ }
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

**Note:** `EventSource` puts the full prompt in the URL query string, which
appears in server logs and browser history. For any prompt that may contain
sensitive input, use the `fetch()` + POST approach above instead.

### Structured Result Rendering

```javascript
const esc = s => String(s).replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));

async function getAnalysis(content) {
  const res = await fetch("/api/analyze", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ content, task: "Analyze this code" })
  });
  const { findings, summary } = await res.json();

  document.getElementById("summary").textContent = summary;
  const list = document.getElementById("findings");
  list.innerHTML = findings.map(f =>
    `<div class="finding ${esc(f.severity)}"><strong>${esc(f.title)}</strong><p>${esc(f.description)}</p></div>`
  ).join("");
}
```

**Don't do this:**
- Don't use raw `innerHTML` with unescaped model output — in a same-origin
  app with Bash tools enabled, XSS in the page is remote code execution.

---

## Error Surfacing Checklist

Three failure channels, all of which must be handled:

**stderr** — Claude writes diagnostics here before the process exits. Always
capture it (`console.error` at minimum). This is the primary debugging signal
when a request fails silently.

**Non-zero exit codes** — Model overloaded (503 from upstream), permission
denied (tool not in `--allowedTools`), or process killed by timeout. For
`Bun.spawnSync`, check `proc.exitCode`. For `Bun.spawn`, `await proc.exited`.

**Malformed output** — Happens when a process is killed mid-stream (timeout,
client disconnect). Stdout contains partial JSON that won't parse. Always wrap
`JSON.parse` in try/catch, and always check `parsed.is_error` before reaching
for `structured_output`.

---

## Tool Flag Reference

Two flags control which tools Claude can use:

- `--tools <list>` — selects which built-in tools exist in this invocation;
  `--tools ""` disables all built-in tools entirely.
- `--allowedTools <list>` — auto-approves the listed tools so Claude can use
  them without a permission prompt; tools not in the list still exist but
  require approval (or are blocked in `--permission-mode dontAsk`).

Use `--tools ""` for pure-analysis tasks where no file access is needed. Use
`--allowedTools` to specify which tools Claude may act on in agentic tasks.
