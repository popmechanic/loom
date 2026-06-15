# Advanced Patterns Reference

Advanced server-side patterns for Loom web apps — async extraction, persistent sessions, mid-stream markers, and HTTP hooks for tool lifecycle events. Read this when the basic communication patterns (REST, SSE, WebSocket) aren't enough for your use case.

## Table of Contents

- [Structured Extraction (Async Haiku)](#structured-extraction-async-haiku)
- [Persistent Session (Long-Lived Process)](#persistent-session-long-lived-process)
- [Action Markers](#action-markers)
- [HTTP Hooks](#http-hooks)
  - [Configuration](#configuration)
  - [Receiving Hook Events](#receiving-hook-events)
  - [Interactive Permission Approval from the Browser](#interactive-permission-approval-from-the-browser)

---

## Structured Extraction (Async Haiku)

For lightweight data extraction tasks — form field suggestions, entity
parsing, classification — spawn a one-shot Haiku process with no tools
and a JSON schema. Async with a timeout kill, unlike the synchronous
`execFileSync` REST pattern.

```typescript
async function extract<T>(prompt: string, schema: object, session: UserSession, timeoutMs = 30000): Promise<T> {
  const proc = spawn("claude", [
    "-p", "--model", "haiku", "--output-format", "json",
    "--json-schema", JSON.stringify(schema),
    "--tools", "", "--no-session-persistence",
    "--permission-mode", "dontAsk"
  ], { stdio: ["pipe", "pipe", "pipe"], env: spawnEnvForUser(session) });

  const timer = setTimeout(() => proc.kill(), timeoutMs);

  let stdout = "";
  let stderr = "";
  proc.stdout.on("data", (chunk) => { stdout += chunk.toString(); });
  proc.stderr.on("data", (chunk) => { stderr += chunk.toString(); });

  // Register close listener BEFORE writing to stdin — a fast process
  // could exit before the listener is attached otherwise.
  const exitCode = new Promise<number | null>(r => proc.on("close", r));

  proc.stdin.write(prompt);
  proc.stdin.end();

  try {
    const code = await exitCode;
    if (code !== 0) {
      throw new Error(`Extraction failed (exit ${code}): ${stderr.slice(0, 200)}`);
    }
    const wrapper = JSON.parse(stdout);
    if (wrapper.is_error) throw new Error(wrapper.result);
    if (wrapper.structured_output) return wrapper.structured_output as T;
    try { return JSON.parse(wrapper.result) as T; } catch {
      throw new Error("Claude returned no structured_output and result was not valid JSON");
    }
  } finally {
    clearTimeout(timer);
  }
}
```

---

## Persistent Session (Long-Lived Process)

Instead of spawning a new process per request, keep one Claude process alive
and send messages via JSONL on stdin. Lower latency, continuous context, no
session serialization overhead. The tradeoff is lifecycle management.

```typescript
import { spawn, ChildProcess } from "child_process";

// Map<sessionId, { proc, lastActivity }> — one entry per user session.
// Never share a proc across sessions: each user's CLAUDE_CODE_OAUTH_TOKEN is
// injected at spawn time, so a shared proc would run all requests under
// whichever user started it first.
const sessions = new Map<string, { proc: ChildProcess; lastActivity: number }>();

function buildArgs(systemPrompt?: string): string[] {
  const args = [
    "-p",
    "--input-format", "stream-json",
    "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",
    "--permission-mode", "dontAsk",
    "--allowedTools", "Read,Write,Edit,Bash,Glob,Grep",
  ];
  if (systemPrompt) args.push("--append-system-prompt", systemPrompt);
  return args;
}

function getOrStart(session: UserSession, systemPrompt?: string): { proc: ChildProcess; lastActivity: number } {
  let s = sessions.get(session.id);
  if (!s || s.proc.exitCode !== null) {
    const proc = spawn("claude", buildArgs(systemPrompt), {
      stdio: ["pipe", "pipe", "pipe"],
      env: spawnEnvForUser(session),
    });
    s = { proc, lastActivity: Date.now() };
    sessions.set(session.id, s);

    const parse = createStreamParser((event) => {
      if (event.type === "stream_event" && event.event?.delta?.text) {
        broadcast(session.id, { type: "token", text: event.event.delta.text });
      } else if (event.type === "assistant" && event.message?.content) {
        for (const block of event.message.content) {
          if (block.type === "tool_use") {
            broadcast(session.id, { type: "tool", name: block.name, input: block.input });
          }
        }
      } else if (event.type === "result") {
        if (event.subtype === "error_max_turns") {
          broadcast(session.id, { type: "warning", message: "Task incomplete — reached turn limit" });
        } else if (event.is_error) {
          broadcast(session.id, { type: "error", message: event.result });
        } else {
          broadcast(session.id, { type: "done" });
        }
      }
    });
    proc.stdout!.on("data", parse);
    proc.stderr!.on("data", (chunk) => console.error(`[claude:${session.id}] ${chunk}`));
    proc.on("close", () => { sessions.delete(session.id); });
  }
  return s;
}

function sendMessage(session: UserSession, text: string): boolean {
  const s = sessions.get(session.id);
  if (!s || s.proc.exitCode !== null) return false;
  const jsonl = JSON.stringify({
    type: "user",
    message: { role: "user", content: [{ type: "text", text }] },
  }) + "\n";
  s.proc.stdin!.on("error", () => {});
  s.proc.stdin!.write(jsonl);
  s.lastActivity = Date.now();
  return true;
}

function endSession(session: UserSession) {
  const s = sessions.get(session.id);
  if (s) { s.proc.kill(); sessions.delete(session.id); }
}

// Inactivity timeout — kill idle sessions after 15 minutes
setInterval(() => {
  const now = Date.now();
  for (const [id, s] of sessions) {
    if (now - s.lastActivity > 15 * 60 * 1000) {
      s.proc.kill();
      sessions.delete(id);
    }
  }
}, 60000);
```

Key differences from the per-request patterns:
- Uses `--input-format stream-json` for bidirectional communication
- Stdin stays open — messages are newline-delimited JSON, not piped and closed
- `--append-system-prompt` injects persona/constraints without replacing base prompt
- Lifecycle management required: start, stop, inactivity timeout
- `getOrStart` is lazy — call it before `sendMessage` to ensure the process exists
- Each session gets its own spawned process with its own `CLAUDE_CODE_OAUTH_TOKEN`; never share a process across users

---

## Action Markers

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

---

## HTTP Hooks

Claude Code can POST structured JSON to your Loom server at lifecycle points —
before/after tool calls, when Claude finishes responding, when sessions
start/end. This gives your server reliable, typed events without parsing
stdout. HTTP hooks **supplement** stdout streaming (you still need
`stream-json` for token-by-token text), but they're more reliable for
lifecycle events because they come from Claude Code's own event system.

### Configuration

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

### Receiving Hook Events

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

### Interactive Permission Approval from the Browser

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
