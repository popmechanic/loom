---
name: loom
description: >
  Build server-based web applications where Claude Code CLI (`claude -p`) is
  the backend runtime — a Node/Express server spawns Claude processes to power
  a custom browser interface with streaming output.
when_to_use: >
  Use when building a deployed or multi-user web app on a server, handling
  per-user Anthropic OAuth, rate limiting, or persistent sessions. Triggers:
  "build an app that uses Claude", "Claude as backend/runtime", "wrap claude -p
  in a server", "streaming Claude output to browser". NOT for local single-user
  tools (use loom-local), desktop apps (use loom-desktop), or direct Anthropic
  API usage.
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
events or UI-driven permission approval (see `references/advanced-patterns.md#http-hooks`).

When a run can outlive the page — long tasks, or users who refresh or open a
second tab — don't tie the stream to a single request. A plain per-request SSE
stream dies on reload (the most common thing a user does), losing the run. Put a
server-side event log between Claude and the browser so clients can reconnect and
replay what they missed (see `references/server-patterns.md#pattern-reconnect-safe-streaming`).

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
| Multi-step workflow with state | Session-based (`--session-id` first turn, `--resume <id>` after), or a persistent duplex bridge (`references/advanced-patterns.md#persistent-session-long-lived-process`) |
| Concurrent analysis of multiple items | Parallel `claude -p` processes |
| Person steers while Claude works | Persistent stdin bridge: write user messages mid-run; stop with an interrupt (`references/server-patterns.md#pattern-interrupting-a-run`) |

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
  files to a temp directory and pass the path (see `references/server-patterns.md#handling-file-uploads-and-drops`).
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

For richer session UIs — live tool calls with running/done status, streaming
tool input progress, diffs, plan/todo checklists, reasoning panels, and a
context-window meter — see `references/session-ux.md`.

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
- **Securing tool access**: An approval gate alone is not enough — some built-in tools
  can bypass it. Narrow the surface with `--allowedTools` and name dangerous ones in
  `--disallowedTools` too (see `references/advanced-patterns.md#securing-tool-access-a-gate-is-not-enough`).
- **Multi-tenancy**: If multiple users, each needs isolated sessions (see the multi-user
  pattern in `references/oauth-reference.md`). Each user authenticates independently,
  gets a session cookie, and their Claude processes receive their own token via env var.
  Consider separate working directories per user if they have persistent file operations.

### 6. Model and performance

| Need | Model | Why |
|------|-------|-----|
| Fast responses (<3s) | `haiku` | Classification, extraction, routing (Haiku 4.5) |
| Good quality, reasonable speed | `sonnet` | Default for most apps (Sonnet 4.6) |
| Best reasoning | `opus` | Complex analysis, code generation (Opus 4.8) |
| Hardest agentic work | `fable` | Most capable (Fable 5) — enable when available on your plan |
| Reliability | `--fallback-model sonnet,haiku` | Auto-fallback on overload (comma-separated, tried in order) |

Reasoning depth is a separate axis from model choice: `--effort` takes
`low | medium | high | xhigh | max`. `xhigh` is Claude Code's own default for
coding/agentic work; use `low`/`medium` for routing and extraction. The `fable`
alias resolves to Anthropic's most capable model — the CLI recognizes it, so
swap it in for `opus` on the hardest tasks once it's available on your plan.

For web UIs, perceived speed matters. Use streaming to show partial results
immediately, even when using slower models.

## Building It

Default to **Node.js/TypeScript** with **Express** for the server and plain
**HTML/CSS/JS** or **React** for the frontend, unless the person prefers otherwise.

### Choosing the runtime: CLI vs Agent SDK

Loom spawns the `claude -p` CLI rather than calling the Claude Agent SDK
(`@anthropic-ai/claude-agent-sdk`) as a library, and the reason is authentication.
The CLI authenticates with the user's Claude **subscription** via OAuth: `claude
setup-token` mints a one-year `CLAUDE_CODE_OAUTH_TOKEN` scoped to inference, and the
server injects each user's token into their own spawned process (see Authentication
Setup). No API key, no per-call billing — every user brings their own subscription.

The Agent SDK, used as a library, authenticates only with `ANTHROPIC_API_KEY`, and
Anthropic's policy is explicit that third-party products built on the SDK should not
offer claude.ai/subscription login. (The SDK can reach a subscription only by
pointing `pathToClaudeCodeExecutable` at a real `claude` binary running under the
user's own `HOME` — i.e. driving the CLI anyway — which is fine for a single-user
local tool but not a deployed multi-user server.)

So for a deployed Loom app, use the CLI. Reach for the SDK only when the app is
deliberately API-key-based — an internal or billed tool where you own the key and
per-call cost is acceptable. Everything the SDK is praised for has a CLI equivalent
that keeps subscription OAuth:

| SDK convenience | CLI equivalent (keeps subscription OAuth) |
|-----------------|-------------------------------------------|
| `canUseTool` browser approval | `PreToolUse` HTTP hook → browser (`references/advanced-patterns.md#http-hooks`) |
| Queued steering / mid-turn input | Duplex `--input-format stream-json` on stdin (`references/advanced-patterns.md#persistent-session-long-lived-process`) |
| `sessionStore` / resume | `--session-id` first turn, `--resume <id>` after, plus your own transcript store |
| Programmatic `mcpServers` | `--mcp-config` / `--strict-mcp-config` |
| Typed partial-message stream | `--include-partial-messages` |
| Structured output + retry | `--json-schema` |

### Authentication Setup

Before any server pattern works, each user needs valid Anthropic
credentials. For multi-user apps, tokens are stored in an in-memory session
store — NOT in a shared file. Your server should:

1. Use `requireAuth` middleware on protected endpoints
2. Show the OAuth setup screen if no valid session cookie exists
3. **After token exchange, fetch the user's profile** from `https://api.anthropic.com/v1/me`
   using the new `access_token` as a Bearer token, and store `{name, email}` in the
   session. Without this, the frontend can't show who's logged in — it falls back to
   a generic "CONNECTED" label. This is the most commonly omitted step.
4. Call `refreshSessionIfNeeded()` before spawning Claude processes
5. Inject the user's token via `CLAUDE_CODE_OAUTH_TOKEN` env var on each spawn

**Always log OAuth errors server-side.** The exchange endpoint calls
Anthropic's token endpoint over HTTPS — this is the most failure-prone
path (DNS issues on fresh VMs, transient network errors, expired codes).
Both the `!resp.ok` branch and the `catch` block must `console.error`
the actual error, not silently return a generic message. Without this,
`journalctl` shows nothing when the exchange fails and you're debugging
blind.

> Read `references/oauth-reference.md` for the complete implementation —
> PKCE utilities, server endpoints, session store, `requireAuth` middleware,
> token refresh, and the ready-to-use React `<SetupScreen>` component.

### The Server Layer

The server's job is simple: receive HTTP requests, spawn Claude, return results.

Read `references/cli-runtime-reference.md` for the full `claude -p` flag reference.

#### Safety Defaults

Every pattern runs Claude in a server — no human sitting at a terminal
to approve tool use. Three flags are non-negotiable:

**`--permission-mode dontAsk`** — In a server context, there's nobody to click
"approve." Without this flag, Claude hangs forever waiting for interactive
input. `dontAsk` auto-denies any tool not in `--allowedTools`, which is exactly
what you want: predictable, unattended execution.

**Critical:** Pair `--permission-mode dontAsk` with `--allowedTools` or `--tools`.
Without allowed tools, `dontAsk` gives Claude no tools at all — it can reason
but can't act, and the failure is silent (no error, just missing results).

**`--max-turns`** — Prevents
conversational loops where Claude keeps trying approaches that won't work.
`5` for one-shot tasks, `10-15` for streaming, `20` for multi-turn sessions.

**`--max-budget-usd`** (recommended for unattended runs) — A hard dollar ceiling
per invocation. `--max-turns` bounds the loop count; `--max-budget-usd` bounds the
spend. For a server running Claude processes nobody is watching, set both so a
runaway task can't burn turns *or* budget. Print-mode only — silently ignored
outside `-p`.

Every pattern also handles three failure modes:

- **stderr** — Claude writes warnings, errors, and diagnostics here. Always
  capture it; it's your best debugging signal when something goes wrong.
- **Non-zero exit codes** — Model overloaded, permission denied, timeout hit.
  `execFileSync` throws; `spawn` emits a `close` event.
- **Malformed output** — A killed or timed-out process may emit partial JSON.
  Always wrap JSON.parse in try/catch and check `parsed.is_error` before
  using `structured_output`.

#### Shared Utilities

Three helpers used by every pattern: `cleanEnv()` (remove nesting guards),
`createStreamParser()` (buffer stdout into JSON lines), and
`spawnEnvForUser()` (inject OAuth token into spawn env).

> See `references/server-patterns.md#shared-utilities` for the full
> implementations with explanatory prose.

#### Patterns at a Glance

| Pattern | When to Use | Reference |
|---------|-------------|-----------|
| REST + JSON | One-shot requests, data extraction | `references/server-patterns.md#pattern-rest-endpoint` |
| SSE Streaming | Streaming text to browser | `references/server-patterns.md#pattern-sse-streaming` |
| WebSocket | Bidirectional, multi-turn | `references/server-patterns.md#pattern-websocket-session` |
| Background Job | Long-running tasks | `references/server-patterns.md#pattern-background-job-with-progress` |
| Parallel | Batch analysis | `references/server-patterns.md#pattern-parallel-analysis` |
| Reconnect-Safe Stream | Survive browser reload, multi-tab, runs that outlive a request | `references/server-patterns.md#pattern-reconnect-safe-streaming` |
| Interrupt | Stop an in-flight run from the UI | `references/server-patterns.md#pattern-interrupting-a-run` |
| Structured Extraction | Fast async data extraction (Haiku) | `references/advanced-patterns.md#structured-extraction-async-haiku` |
| Persistent Session | Long-lived process, lower latency | `references/advanced-patterns.md#persistent-session-long-lived-process` |
| Action Markers | Mid-stream structured events | `references/advanced-patterns.md#action-markers` |
| HTTP Hooks | Tool lifecycle events, browser permission approval | `references/advanced-patterns.md#http-hooks` |
| Validate Preview | Avoid white-screening on mid-stream writes | `references/server-patterns.md#pattern-validate-before-reloading-a-preview` |
| Honest Progress | Show truthful progress when tokens aren't flowing | `references/server-patterns.md#pattern-honest-progress-for-slow-turns` |
| Durable Sessions | Survive a restart; persist transcript + session_id | `references/advanced-patterns.md#durable-sessions-survive-a-restart` |

> Read `references/server-patterns.md` when implementing a specific server
> endpoint or wiring up the frontend. Read `references/advanced-patterns.md`
> when the basic patterns aren't enough for your use case.

### Stream-JSON Event Types

When using `--output-format stream-json --verbose --include-partial-messages`,
Claude emits newline-delimited JSON events:

| Event Type | Shape | Forward? |
|------------|-------|----------|
| `system` | `{type:"system", subtype:"init", session_id, model, tools}` | Optional (extract session_id) |
| `stream_event` | `{type:"stream_event", event:{delta:{text:"..."}}}` | Yes (live text) |
| `assistant` | `{type:"assistant", message:{content:[...]}}` | Tool use only (text already streamed) |
| `tool_result` | `{type:"tool_result", tool_name, content, is_error}` | Optional |
| `compact` | `{type:"compact"}` | No (internal) |
| `rate_limit_event` | `{type:"rate_limit_event", rate_limit_info:{...}}` | No (log it) |
| `result` | `{type:"result", subtype:"success"|"error_max_turns", is_error}` | Yes (done signal) |

> See `references/server-patterns.md#stream-json-event-types` for complete notes,
> extended thinking behavior, code samples for extracting text/tool use, and
> max-turns detection.

### Frontend Integration

Use `fetch()` + `ReadableStream` for POST-based SSE (CSRF-safe). Parse `data:`
lines, dispatch on event type (`token`, `done`, `error`). For quick prototyping,
`EventSource` works for GET-based SSE.

> See `references/server-patterns.md#frontend-integration` for the complete
> streaming text display and structured result rendering code.

### Error Handling

Every pattern in `references/server-patterns.md` handles errors inline — you
won't find a separate error handling block to copy-paste because it doesn't
belong in one. Here's the mental model behind the three failure modes:

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

> See `references/server-patterns.md#error-surfacing-checklist` for the
> three-channel checklist with code samples.

## What to Generate

When you build the app, produce:

1. **`server.ts`** — Express server using the Express baseline from
   `references/server-patterns.md#server-setup`
   (express, cookie-parser, express.static). Includes session store,
   `requireAuth` middleware, OAuth endpoints (`/api/oauth/start`,
   `/api/oauth/exchange`, `/api/health`, `/api/logout`), and your app's
   endpoint pattern(s). The exchange endpoint must fetch the user's
   profile from `https://api.anthropic.com/v1/me` and store it in
   the session. Each spawn uses `spawnEnvForUser()` to inject the
   requesting user's token.
2. **`public/index.html`** — The frontend, starting with the `<SetupScreen>`
   component (shown when no session exists) and your app's main UI
   (shown after authentication). Include a user indicator showing
   the user's email (from `/api/health` → `user.email`) and a logout
   button that calls `POST /api/logout`. Use a fallback label like
   "CONNECTED" if the profile has no email. Protected endpoints check
   for 401 responses and redirect to the setup screen (in-memory
   sessions are wiped on server restart). If the app has both a chat
   input and a setup screen input, use `input.setup-input` selectors
   (not `.setup-input`) to avoid CSS specificity conflicts with global
   `input[type="text"]` rules — see `references/oauth-reference.md`.
3. **`package.json`** — Dependencies (`express`, `cookie-parser`, `uuid`,
   plus `ws` and `cookie` if using WebSockets, and `cors` if frontend/server
   are separate origins) and a start script
4. **A one-liner to run it** — so the person can verify it works immediately

### First-Run Reliability Checklist

These are the silent-failure modes — things that break with NO error message.
The patterns in `references/server-patterns.md` demonstrate correct handling
for each, but they're easy to miss or deviate from when generating a new app.

- [ ] `--include-partial-messages` is on every streaming spawn — without it, text dumps as a single block instead of streaming token-by-token
- [ ] Text is forwarded from `stream_event` only, NOT from `assistant` text blocks — otherwise every token appears twice
- [ ] `spawnEnvForUser()` is called on every `spawn`/`execFileSync` — this removes nesting guards AND injects the user's OAuth token; bare `cleanEnv()` omits the token and causes silent auth failure
- [ ] `--permission-mode dontAsk` is paired with `--allowedTools` or `--tools` — without allowed tools, Claude produces an empty result with NO error
- [ ] `subtype === "error_max_turns"` is checked **before** `is_error` on result events — max-turns sets `is_error: true` (verified on CLI v2.1.x), so an `if (is_error) … else if (subtype === …)` ordering makes the max-turns branch dead code and surfaces "incomplete" as a hard error
- [ ] `express.json()` middleware is applied before any route that reads `req.body` — without it, `req.body` is `undefined` and the spawn gets an empty prompt
- [ ] 401 responses in the frontend redirect to the setup screen — in-memory sessions are wiped on server restart, and a 401 fed to the SSE parser fails silently
- [ ] `cookie-parser` middleware is applied before any route that reads `req.cookies` — without it, `requireAuth` sees `undefined` and every request returns 401
- [ ] Profile fetch happens in the `/api/oauth/exchange` handler after token exchange — without it, the frontend shows "CONNECTED" instead of the user's email
- [ ] `/api/health` does NOT use `requireAuth` — it must return `{needsSetup: true}` for unauthenticated users, not 401
- [ ] WebSocket upgrade validates the session cookie from `req.headers.cookie` — Express middleware does not run on WebSocket handshakes

For simple apps, a single `server.ts` serving a static `index.html` is ideal.
For complex UIs, scaffold a React frontend with a separate server. For the
frontend itself, a no-build-step approach (UMD React + Babel-in-browser, or
plain HTML/JS) matches the thin-bridge ethos and ships faster — reach for a
bundled build only when the UI complexity justifies it. All five reference apps
shipped no-build frontends.

When deploying behind a trusted reverse proxy (exe.dev, Cloudflare Access) that
handles user identity, the proxy-header auth alternative avoids the full OAuth
flow for the user-identity layer — see `references/oauth-reference.md#reverse-proxy-header-auth`.

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
