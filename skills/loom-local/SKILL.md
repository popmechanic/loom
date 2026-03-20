---
name: loom-local
description: >
  Use when building a local web application that needs Claude Code CLI
  (`claude -p`) as its runtime, using Bun as the server layer. The Bun server
  spawns Claude processes and streams results to the browser — no authentication,
  no deployment, no multi-tenancy. Triggers: "local Claude app", "Bun server
  with Claude", "claude -p web app", "local Claude-powered tool", "build a
  local app that uses Claude", or any local web app needing Claude's agentic
  capabilities through a browser interface. NOT for deployed/multi-user apps
  (use loom), desktop apps (use loom-desktop), or direct Anthropic API usage.
---

# Loom Local: Applications on the Claude Code Runtime (Bun)

Build local web applications where Claude Code CLI (`claude -p`) is the
runtime — not a helper writing React components, but the actual engine powering
the application's intelligence. The interface talks to a Bun server that spawns
`claude -p` processes, streaming results back to the browser.

No authentication, no deployment, no multi-tenancy. The user's machine already
has Claude Code authenticated — the server is just a bridge between browser
and CLI.

## Why This Matters

Most "AI-powered" web apps just wrap a chat API. They put a text box on screen,
send messages to an LLM, and show the response. That's a chat replica.

Claude Code is not a chat API. It's an agentic runtime with filesystem access,
tool use, multi-turn sessions, structured output, and streaming. Building an
interface on top of it means designing something that exposes these capabilities
in ways that make sense — not just conversations, but whatever interaction
paradigm fits the task.

The question to explore: **what would you build if your backend could read
files, run code, search the web, coordinate multiple agents, and stream its
reasoning to the browser in real time?**

## The Architecture

Every app has the same basic shape:

```
┌──────────────┐     HTTP/WS      ┌──────────────┐    stdio     ┌──────────────┐
│   Interface  │ ◄──────────────► │  Bun Server   │ ◄──────────► │  claude -p   │
│  (HTML/React)│                  │ (Bun.serve()) │              │              │
└──────────────┘                  └──────────────┘              └──────────────┘
     UI layer                      Bridge layer                  Runtime layer
```

**Interface**: The custom UI. Whatever makes sense for what's being built.

**Bun Server**: The bridge. Receives requests from the interface, spawns Claude
processes, parses output, streams results back. Handles routing, session
management, and the mapping between web concepts and Claude invocations.

**Claude Runtime**: The intelligence. `claude -p` with the right flags. Reading
files, running commands, generating structured output. Running locally, Claude
inherits the machine's credentials automatically — no token injection needed.

### Communication Patterns

| Pattern | When to Use | How |
|---------|-------------|-----|
| **REST + JSON** | One-shot requests, data extraction | POST → `claude -p --output-format json` → JSON response |
| **SSE (Server-Sent Events)** | Streaming text to browser | `claude -p --output-format stream-json --verbose` → SSE stream |
| **WebSocket** | Bidirectional, multi-turn sessions | WS connection → `claude -p --input-format stream-json` |
| **Background job** | Long-running tasks | Queue → `claude -p` process → poll for result |
| **Parallel** | Batch analysis of multiple items | Concurrent `claude -p` spawns → aggregate results |

For most apps, start with **REST + SSE**: REST for triggering tasks, SSE for
streaming progress and results. Add WebSockets only when true bidirectional
communication is needed (e.g., the user can interrupt or steer while Claude
is working).

## The Conversation

Walk through these design questions before building. Cover them naturally —
don't dump them all at once.

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
  entirely — for character personas, branded assistants, or apps where Claude should
  NOT inherit the user's CLAUDE.md settings. `--append-system-prompt` adds to the
  default prompt (preserving user settings, skills, etc.) — for when Claude should
  still act as a general assistant with extra instructions.
- **What data?** Files on disk? User-uploaded content? Piped from other services?
  Claude's Read tool natively handles images and PDFs — save uploaded binary files
  to a temp directory and pass the path.
- **What output shape?** Free text for display? Structured JSON for rendering UI
  components? Both (use `--json-schema` for structured + `result` for narrative)?

### 4. How should the output render?

This is where custom interfaces shine over CLIs. Render Claude's structured
output as rich UI:

- JSON schema with typed arrays → cards, lists, timelines, visualizations
- Schema with nodes and edges → interactive graph
- Schema with sections and status → progress view
- Stream-JSON events → animated progress indicator or live log

Design the JSON schema to match the UI components to render. The schema IS the
API contract between Claude and the frontend.

### 5. What are the safety boundaries?

Local apps simplify security but don't eliminate it:

- **Sandboxing**: Claude has filesystem access — scope the working directory.
  Use `--add-dir` to expand access when needed.
- **Permissions**: Use the tightest `--permission-mode` and `--allowedTools` that
  work. Prefer `dontAsk` over `bypassPermissions`.
- **Turn limits**: Set `--max-turns` to prevent runaway loops. `5` for one-shot,
  `10-15` for streaming, `20` for multi-turn.
- **Cost control**: Use `--max-budget-usd` to cap spending on expensive tasks.

### 6. Model and performance

| Need | Flag | Why |
|------|------|-----|
| Fast responses (<3s) | `--model haiku` | Classification, extraction, routing |
| Good quality, reasonable speed | `--model sonnet` | Default for most apps |
| Best reasoning | `--model opus` | Complex analysis, code generation |
| Reliability | `--fallback-model haiku` | Auto-fallback on overload |
| Control reasoning depth | `--effort high` or `--effort low` | Tune thinking vs. speed |

For web UIs, perceived speed matters. Use streaming to show partial results
immediately, even when using slower models.

## Building It

Default to **Bun** with **TypeScript** for the server and plain **HTML/CSS/JS**
or **React** for the frontend. Zero external dependencies — `Bun.serve()` handles
HTTP, static files, and WebSockets natively.

> Read `references/server-patterns.md` for the complete server setup, shared
> utilities (`cleanEnv()`, `createStreamParser()`), and all five communication
> pattern implementations with full code.

### Safety Defaults

Every pattern runs Claude from a server — no human at a terminal to approve
tool use. Two flags are non-negotiable:

**`--permission-mode dontAsk`** — Nobody to click "approve." Without this flag,
Claude hangs forever waiting for interactive input.

**Critical:** Pair `--permission-mode dontAsk` with `--allowedTools` or `--tools`.
Without allowed tools, `dontAsk` gives Claude no tools at all — it can reason
but can't act, and the failure is silent (no error, just missing results).

**`--max-turns`** — Prevents conversational loops where Claude keeps trying
approaches that won't work.

Every pattern also handles three failure modes:

- **stderr** — Warnings, errors, diagnostics. Always capture it.
- **Non-zero exit codes** — Model overloaded, permission denied, timeout hit.
- **Malformed output** — A killed process may emit partial JSON. Always wrap
  `JSON.parse` in try/catch and check `is_error` before using `structured_output`.

### Patterns at a Glance

| Pattern | When to Use | Reference |
|---------|-------------|-----------|
| REST + JSON | One-shot requests, data extraction | `references/server-patterns.md#pattern-rest-endpoint` |
| SSE Streaming | Streaming text to browser | `references/server-patterns.md#pattern-sse-streaming` |
| WebSocket | Bidirectional, multi-turn | `references/server-patterns.md#pattern-websocket-session` |
| Background Job | Long-running tasks | `references/server-patterns.md#pattern-background-job-with-progress` |
| Parallel | Batch analysis | `references/server-patterns.md#pattern-parallel-analysis` |

> Read `references/server-patterns.md` when implementing a specific server
> endpoint or wiring up the frontend.

### Stream-JSON Event Types

When using `--output-format stream-json --verbose --include-partial-messages`,
Claude emits newline-delimited JSON events:

| Event Type | Shape | Forward? |
|------------|-------|----------|
| `system` | `{type:"system", subtype:"init", session_id, model, tools}` | Optional (extract session_id) |
| `stream_event` | `{type:"stream_event", event:{delta:{text:"..."}}}` | Yes (live text) |
| `assistant` | `{type:"assistant", message:{content:[...]}}` | Tool use only (text already streamed) |
| `tool_result` | `{type:"tool_result", tool_name, content, is_error}` | Optional |
| `result` | `{type:"result", subtype:"success"|"error_max_turns", is_error}` | Yes (done signal) |

**Max-turns detection:** The `result` event has `subtype: "error_max_turns"`
with `is_error: false`. Check `subtype` — easy to miss because `is_error` is false.

**Extended thinking models** work correctly. Thinking tokens have
`delta.thinking` instead of `delta.text` — the `event?.delta?.text` check
naturally skips them.

## What to Generate

When building the app, produce:

1. **`server.ts`** — Bun server using the `Bun.serve()` baseline from
   `references/server-patterns.md#server-setup`, with static file serving,
   endpoint pattern(s), and WebSocket handlers if needed.
2. **`public/index.html`** — The frontend UI. Use the streaming or structured
   rendering patterns from `references/server-patterns.md#frontend-integration`.
3. **A one-liner to run it** — `bun run server.ts` — so the person can verify
   it works immediately.

For simple apps, a single `server.ts` serving a static `index.html` is ideal.
For complex UIs, scaffold a React frontend with a separate server.

After generating, offer to start the server and open it in the browser. Then
iterate based on what the person sees.

### First-Run Reliability Checklist

These are silent-failure modes — things that break with NO error message:

- [ ] `--include-partial-messages` is on every streaming spawn — without it, text dumps as a single block
- [ ] Text is forwarded from `stream_event` only, NOT from `assistant` text blocks — otherwise every token appears twice
- [ ] `cleanEnv()` is called on every `Bun.spawn`/`Bun.spawnSync` — without it, Claude refuses to start inside a Claude Code session
- [ ] `--permission-mode dontAsk` is paired with `--allowedTools` or `--tools` — without allowed tools, Claude produces an empty result with NO error
- [ ] `subtype === "error_max_turns"` is checked on result events — this fires with `is_error: false`, so unchecked it looks like success
- [ ] Request body is parsed with `await req.json()` before accessing fields — without it, the spawn gets an empty prompt
- [ ] `stdout` chunks are buffered into complete JSON lines before parsing — TCP delivers arbitrary chunk boundaries

> Read `references/cli-runtime-reference.md` for the full `claude -p` flag
> reference — input methods, output formats, structured output, tool
> configuration, session management, and gotchas.

## The Possibility Space

The most interesting apps built this way are not chat interfaces in a browser.
They're things that couldn't exist without an agentic runtime — applications
where the backend can read, reason, and act on context that traditional APIs
can't access.

The question isn't "how do I put a chat box in a browser?" but "what would
this look like if there were an intelligence behind it?"
