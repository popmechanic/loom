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
- [Pattern: Background Job with Progress](#pattern-background-job-with-progress)
- [Pattern: Parallel Analysis](#pattern-parallel-analysis)
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

```typescript
const PORT = Number(process.env.PORT) || 3456;

const server = Bun.serve({
  port: PORT,
  async fetch(req) {
    const url = new URL(req.url);

    // Static files
    if (url.pathname === "/" || !url.pathname.startsWith("/api/")) {
      const filePath = url.pathname === "/"
        ? "./public/index.html"
        : `./public${url.pathname}`;
      const file = Bun.file(filePath);
      if (await file.exists()) return new Response(file);
      return new Response("Not found", { status: 404 });
    }

    // API routes (add patterns below)
    return new Response("Not found", { status: 404 });
  },
});

console.log(`Server running on http://localhost:${PORT}`);
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
          if (event.is_error) {
            encode({ type: "error", message: event.result });
          } else if (event.subtype === "error_max_turns") {
            encode({ type: "warning", message: "Task incomplete — reached turn limit" });
          } else {
            encode({ type: "done" });
          }
        }
      });

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

      const stderrText = await new Response(proc.stderr).text();
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
      const filePath = url.pathname === "/"
        ? "./public/index.html"
        : `./public${url.pathname}`;
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
      const { payload } = JSON.parse(raw as string);

      if (data.activeProc) {
        ws.send(JSON.stringify({ type: "error", message: "Processing in progress" }));
        return;
      }

      // First turn: --session-id creates the session
      // Subsequent turns: --resume continues that specific session
      const sessionArgs = data.isFirstTurn
        ? ["--session-id", data.sessionId]
        : ["--resume", data.sessionId];

      const proc = Bun.spawn(["claude",
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
          if (event.is_error) {
            ws.send(JSON.stringify({ type: "error", message: event.result }));
          } else if (event.subtype === "error_max_turns") {
            ws.send(JSON.stringify({ type: "warning", message: "Task incomplete — reached turn limit" }));
          } else {
            ws.send(JSON.stringify({ type: "done" }));
          }
        }
      });

      const reader = proc.stdout.getReader();
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        parse(value);
      }

      data.activeProc = null;
      if (!gotResult) {
        ws.send(JSON.stringify({ type: "error", message: "Process finished without producing a result" }));
      }

      const stderrText = await new Response(proc.stderr).text();
      if (stderrText.trim()) console.error(`[claude stderr] ${stderrText.trim()}`);
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

## Pattern: Background Job with Progress

Long-running task that reports progress via polling.

```typescript
const jobs = new Map<string, { status: string; result?: any; error?: string }>();

// POST /api/jobs — start a job
if (url.pathname === "/api/jobs" && req.method === "POST") {
  const { task } = await req.json();
  const jobId = crypto.randomUUID();
  jobs.set(jobId, { status: "running" });

  const proc = Bun.spawn(["claude",
    "-p", "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--model", "sonnet", "--max-turns", "20",
    "--permission-mode", "dontAsk",
    "--tools", "Read,Bash,Glob,Grep",
    "--no-session-persistence"
  ], { stdout: "pipe", stderr: "pipe", stdin: "pipe", env: cleanEnv() });

  proc.stdin.write(task);
  proc.stdin.end();

  // Process in background
  (async () => {
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

    const reader = proc.stdout.getReader();
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      parse(value);
    }

    if (jobs.get(jobId)?.status === "running") {
      const stderrText = await new Response(proc.stderr).text();
      jobs.set(jobId, { status: "failed", error: stderrText.trim() || "Process exited unexpectedly" });
    }
  })();

  return Response.json({ jobId });
}

// GET /api/jobs/:id — poll status
if (url.pathname.startsWith("/api/jobs/") && req.method === "GET") {
  const jobId = url.pathname.split("/api/jobs/")[1];
  const job = jobs.get(jobId);
  return Response.json(job || { status: "not_found" });
}
```

---

## Pattern: Parallel Analysis

Analyze multiple items concurrently, aggregate results.

```typescript
if (url.pathname === "/api/batch" && req.method === "POST") {
  const { items, task } = await req.json();
  const TIMEOUT_MS = 30000;

  const schema = JSON.stringify({
    type: "object",
    properties: { analysis: { type: "string" }, score: { type: "number" } },
    required: ["analysis", "score"]
  });

  const outcomes = await Promise.allSettled(
    items.map((item: string) => new Promise<any>((resolve, reject) => {
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
    }))
  );

  const results = outcomes.map((o, i) =>
    o.status === "fulfilled"
      ? { item: items[i], data: o.value }
      : { item: items[i], error: o.reason.message }
  );

  return Response.json({ results });
}
```

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

### Structured Result Rendering

```javascript
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
    `<div class="finding ${f.severity}">
       <strong>${f.title}</strong>
       <p>${f.description}</p>
     </div>`
  ).join("");
}
```

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
