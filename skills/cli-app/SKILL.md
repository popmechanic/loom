---
name: cli-app
description: >
  Build web applications that use Claude Code CLI (`claude -p`) as the runtime —
  a server spawns Claude processes that read files, run commands, and return
  structured output, while a custom frontend renders results as rich UI
  (dashboards, charts, timelines, annotated views). Use when a user wants a web
  interface where Claude is the backend: analytics dashboards, monitoring tools,
  code review UIs, data explorers, or any app calling `claude -p` instead of a
  traditional API. Triggers: "build an app that uses Claude", "web dashboard
  powered by Claude", "wrap claude -p", "Claude as backend", or any web app
  needing Claude's agentic capabilities (file access, tool use, streaming)
  through a purpose-built interface. NOT for Anthropic API apps, chat replicas,
  or standard web apps without an AI runtime.
---

# CLI App: Web Interfaces to Claude Code

You're helping someone build a web application where Claude Code is the runtime —
not a helper writing React components, but the actual backend engine that powers
the application's intelligence. The web frontend talks to a server that spawns
`claude -p` processes or uses the Agent SDK, streaming results back to the browser.

This is a different posture than normal web development. Normally, the backend
is a database and some business logic. Here, the backend is Claude — an agent
that can read files, run commands, search code, and reason about complex tasks.
The frontend's job is to give that agent a purpose-built interface that makes
its capabilities useful for a specific domain.

## Why This Matters

Most "AI-powered" web apps just wrap a chat API. They put a text box on screen,
send messages to an LLM, and show the response. That's a chat replica.

Claude Code is not a chat API. It's an agentic runtime with filesystem access,
tool use, multi-turn sessions, structured output, and streaming. Building a web
interface to Claude Code means designing UIs that expose these capabilities in
ways that make sense for specific use cases — not just conversations, but
workflows, dashboards, pipelines, and interactive tools.

The question to help users explore is: **what would you build if your backend
could read files, run code, search the web, coordinate multiple agents, and
stream its reasoning to the browser in real time?**

## The Architecture

Every Claude Code web app has the same basic shape:

```
┌──────────────┐     HTTP/WS      ┌──────────────┐    stdio/SDK    ┌──────────────┐
│   Browser    │ ◄──────────────► │  Node Server  │ ◄────────────► │  claude -p   │
│  (React/HTML)│                  │  (Express/etc) │                │  (Agent SDK) │
└──────────────┘                  └──────────────┘                └──────────────┘
     UI layer                      Bridge layer                    Runtime layer
```

**Browser**: The custom UI. Could be anything — a dashboard, a Kanban board,
a code review tool, a data explorer, a monitoring panel. Whatever makes sense
for the use case.

**Node Server**: The bridge. Receives requests from the browser, spawns Claude
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

### 1. What does the user see and do?

Get concrete about the interface, not the AI. What's the screen layout? What does
the user click? What appears when Claude is working? What does the final output
look like?

Good: "A split-pane view — file tree on the left, code on the right, and a
review panel at the bottom that fills in as Claude analyzes each file."

Bad: "An AI that reviews code." (Too vague — no interface design.)

Ask: "Walk me through the screen. What does someone see when they open this app?
What do they do first? What happens next?"

### 2. What's the interaction model?

How does the user's action translate to a Claude invocation?

| Interaction | Claude Pattern |
|-------------|----------------|
| Click a button, get a result | One-shot `claude -p`, REST response |
| Watch progress in real-time | Stream-JSON → SSE to browser |
| Multi-step workflow with state | Session-based (`--session-id` + `--continue`) |
| Concurrent analysis of multiple items | Parallel `claude -p` processes |
| User steers while Claude works | Bidirectional streaming via WebSocket |

### 3. What does Claude actually do?

Map each user action to what Claude needs behind the scenes:

- **What tools?** Read-only analysis (`Read,Glob,Grep`) vs. modification (`Write,Edit,Bash`)
  vs. no tools at all (`--tools ""` for pure reasoning)
- **What data?** Files on the server? User-uploaded content? Piped from other services?
- **What output shape?** Free text for display? Structured JSON for rendering UI components?
  Both (use `--json-schema` for structured + `result` for narrative)?

### 4. How should the output render?

This is where web interfaces shine over CLIs. You can render Claude's structured
output as rich UI:

- JSON schema with `issues: [{file, line, severity, message}]` → render as an
  annotated code view with inline markers
- Schema with `{nodes: [], edges: []}` → render as an interactive graph
- Schema with `{sections: [{title, content, status}]}` → render as a progress dashboard
- Stream-JSON events → animate a typing indicator, progress bar, or live log

Design the JSON schema to match the UI components you want to render. The schema
IS your API contract between Claude and the frontend.

### 5. What are the safety boundaries?

Web apps add security concerns that CLIs don't have:

- **Auth**: Who can access this? Do you need user accounts?
- **Sandboxing**: Claude has filesystem access — what directory should it be scoped to?
- **Cost**: Each request costs money — do you need rate limiting? Budget caps (`--max-budget-usd`)?
- **Permissions**: Use the tightest `--permission-mode` and `--allowedTools` that work
- **Multi-tenancy**: If multiple users, each needs isolated sessions and working directories

### 6. Model and performance

| Need | Model | Why |
|------|-------|-----|
| Fast responses (<3s) | `haiku` | Classification, extraction, routing |
| Good quality, reasonable speed | `sonnet` | Default for most web apps |
| Best reasoning | `opus` | Complex analysis, code generation |
| Reliability | `--fallback-model haiku` | Auto-fallback on overload |

For web UIs, perceived speed matters. Use streaming to show partial results
immediately, even when using slower models.

## Building It

Default to **Node.js/TypeScript** with **Express** for the server and plain
**HTML/CSS/JS** or **React** for the frontend, unless the user prefers otherwise.

### The Server Layer

The server's job is simple: receive HTTP requests, spawn Claude, return results.

Read `references/cli-runtime-reference.md` for the full `claude -p` flag reference.
Here are the server patterns to reach for:

#### Pattern: REST Endpoint (One-Shot)

User clicks a button → server calls Claude → returns JSON.

```typescript
import express from "express";
import { execSync } from "child_process";

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

  const result = execSync(
    `claude -p --model sonnet --output-format json --json-schema '${schema}' --tools "" --no-session-persistence`,
    { input: `${task}\n\n${content}`, encoding: "utf-8", timeout: 60000 }
  );

  const { structured_output, total_cost_usd } = JSON.parse(result);
  res.json({ ...structured_output, cost: total_cost_usd });
});
```

#### Pattern: SSE Streaming

User triggers a task → server streams Claude's output token-by-token.

```typescript
app.get("/api/stream", (req, res) => {
  const { task } = req.query;

  res.setHeader("Content-Type", "text/event-stream");
  res.setHeader("Cache-Control", "no-cache");
  res.setHeader("Connection", "keep-alive");

  const cleanEnv = Object.fromEntries(
    Object.entries(process.env).filter(([k]) => !k.startsWith("CLAUDE"))
  );

  const proc = spawn("claude", [
    "-p", "--output-format", "stream-json", "--verbose",
    "--model", "sonnet", "--no-session-persistence",
    task as string
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
      } catch {}
    }
  });

  proc.on("close", () => res.end());
  req.on("close", () => proc.kill());
});
```

#### Pattern: WebSocket Session

Persistent connection with multi-turn conversation and streaming.

```typescript
import { WebSocketServer } from "ws";
import { spawn } from "child_process";
import { v4 as uuidv4 } from "uuid";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws) => {
  const sessionId = uuidv4();

  ws.on("message", (raw) => {
    const { action, payload } = JSON.parse(raw.toString());

    const args = [
      "-p", "--output-format", "stream-json", "--verbose",
      "--session-id", sessionId, "--continue",
      "--model", "sonnet"
    ];

    const cleanEnv = Object.fromEntries(
    Object.entries(process.env).filter(([k]) => !k.startsWith("CLAUDE"))
  );

    const proc = spawn("claude", [...args, payload.prompt], { env: cleanEnv });

    proc.stdout.on("data", (chunk) => {
      for (const line of chunk.toString().split("\n").filter(Boolean)) {
        try {
          const event = JSON.parse(line);
          if (event.event?.delta?.text) {
            ws.send(JSON.stringify({ type: "token", text: event.event.delta.text }));
          } else if (event.type === "result") {
            ws.send(JSON.stringify({ type: "done", result: event }));
          }
        } catch {}
      }
    });
  });
});
```

#### Pattern: Background Job with Progress

Long-running task that reports progress to the browser.

```typescript
const jobs = new Map<string, { status: string; result?: any }>();

app.post("/api/jobs", (req, res) => {
  const jobId = uuidv4();
  jobs.set(jobId, { status: "running" });
  res.json({ jobId });

  // Run Claude in background
  const cleanEnv = Object.fromEntries(
    Object.entries(process.env).filter(([k]) => !k.startsWith("CLAUDE"))
  );

  const proc = spawn("claude", [
    "-p", "--output-format", "stream-json", "--verbose",
    "--model", "sonnet", "--max-turns", "20", "--max-budget-usd", "5",
    "--permission-mode", "bypassPermissions",
    "--tools", "Read,Bash,Glob,Grep",
    "--no-session-persistence"
  ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv });

  proc.stdin.write(req.body.task);
  proc.stdin.end();

  proc.stdout.on("data", (chunk) => {
    for (const line of chunk.toString().split("\n").filter(Boolean)) {
      try {
        const event = JSON.parse(line);
        if (event.type === "result") {
          jobs.set(jobId, { status: "complete", result: event });
        }
      } catch {}
    }
  });
});

app.get("/api/jobs/:id", (req, res) => {
  const job = jobs.get(req.params.id);
  res.json(job || { status: "not_found" });
});
```

#### Pattern: Parallel Analysis

Analyze multiple items concurrently, aggregate results.

```typescript
app.post("/api/batch", async (req, res) => {
  const { items, task } = req.body;

  const results = await Promise.all(
    items.map((item: string) => new Promise<any>((resolve) => {
      const result = execSync(
        `claude -p --model haiku --output-format json --json-schema '${schema}' --tools "" --no-session-persistence`,
        { input: `${task}\n\n${item}`, encoding: "utf-8", timeout: 30000 }
      );
      resolve(JSON.parse(result).structured_output);
    }))
  );

  res.json({ results });
});
```

### The Frontend Layer

The frontend renders Claude's output as purpose-built UI, not chat bubbles.

#### Streaming Text Display

```javascript
const source = new EventSource(`/api/stream?task=${encodeURIComponent(task)}`);
const output = document.getElementById("output");

source.onmessage = (e) => {
  const data = JSON.parse(e.data);
  if (data.type === "token") {
    output.textContent += data.text;
  } else if (data.type === "done") {
    source.close();
    document.getElementById("cost").textContent = `$${data.cost.toFixed(4)}`;
  }
};
```

#### Structured Result Rendering

```javascript
// Claude returns { findings: [{title, severity, description}], summary }
// Render as a rich card layout, not a chat message
function renderFindings(data) {
  return data.findings.map(f => `
    <div class="finding finding--${f.severity}">
      <span class="badge">${f.severity}</span>
      <h3>${f.title}</h3>
      <p>${f.description}</p>
    </div>
  `).join("");
}
```

## What to Generate

When you build the app, produce:

1. **`server.ts`** — Express server with the appropriate endpoint pattern(s)
2. **`public/index.html`** — The frontend (inline styles/scripts for simplicity,
   or a small React app for complex UIs)
3. **`package.json`** — Dependencies and start script
4. **A one-liner to run it** — so the user can verify it works immediately

For simple apps, a single `server.ts` serving a static `index.html` is ideal.
For complex UIs, scaffold a React frontend with a separate server.

After generating, offer to start the server and open it in the browser together.
Then iterate based on what the user sees.

## Thinking Beyond Chat Replicas

The most interesting Claude Code web apps are NOT chat interfaces in a browser.
They're interfaces that couldn't exist without an agentic runtime:

- A **code review dashboard** that shows a file tree, highlights issues inline,
  and lets you click "fix" to have Claude apply the fix — all in the browser
- A **data explorer** where you upload a CSV and get an interactive dashboard
  with charts, filters, and natural language queries — Claude generates the
  visualization code on the fly
- A **incident commander** that monitors logs, correlates alerts, and shows
  a live timeline with suggested runbook actions
- A **migration planner** that reads your codebase, shows a dependency graph,
  and lets you drag-and-drop the migration order while Claude estimates risk
- A **test lab** where you paste a function and Claude generates a battery of
  edge cases, runs them, and shows pass/fail results in a test runner UI
- A **documentation studio** that reads your source files and generates a
  navigable doc site, with Claude filling in explanations for complex sections
- A **deploy review board** that shows the diff, Claude's safety analysis,
  staging test results, and a big green "Ship It" button

Help users think in this direction. The question isn't "how do I put a chat
box in a browser?" but "what interface would make this workflow 10x better
if it had an intelligent agent behind it?"
