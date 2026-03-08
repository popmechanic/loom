# Loom Skill Reliability Overhaul — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use trycycle-executing to implement this plan task-by-task.

**Goal:** Make apps generated from the loom skill work on the first try. The primary root cause is that the skill's streaming patterns do NOT use `--include-partial-messages`, so streaming apps dump text as a single block instead of token-by-token. Secondary causes: incomplete code patterns (missing middleware, missing error handling, missing CORS), inaccurate stream event documentation, dead code paths, and a "What to Generate" checklist that mentions dependencies the patterns never demonstrate.

**Architecture:** SKILL.md remains a single comprehensive document. All bug fixes and pattern improvements are applied directly in place. No structural split — Claude Code loads SKILL.md at skill activation, and inline patterns are more reliable than cross-references to separate files. Event documentation is corrected against verified facts from testing on 2026-03-08. Cost/budget references are removed (subscription users only).

**Tech Stack:** Markdown (SKILL.md), TypeScript code examples, `claude -p` CLI

---

### Task 1: Fix the streaming bug — add `--include-partial-messages`

**THIS IS THE PRIMARY RELIABILITY FIX.** Without `--include-partial-messages`, text arrives ONLY as complete `assistant` blocks — no token-by-token streaming. The skill's SSE, WebSocket, Background Job, and Persistent Session patterns all produce apps where streaming text appears as a single dump instead of typing in progressively. This flag was verified on this machine (2026-03-08) as required for token streaming.

**Files:**
- Modify: `skills/loom/SKILL.md`

**Step 1: Add `--include-partial-messages` to ALL streaming patterns**

In every `spawn("claude", [...])` call that uses `--output-format stream-json`, add `"--include-partial-messages"` to the args array. This applies to:

1. **SSE Streaming pattern** (~line 397) — the `spawn("claude", [...])` call
2. **WebSocket Session pattern** (~line 507) — the `spawn("claude", [...])` call
3. **Background Job pattern** (~line 586) — the `spawn("claude", [...])` call
4. **Persistent Session pattern** (~line 1077) — the `spawn("claude", [...])` call

Example change for the SSE pattern:
```typescript
  const proc = spawn("claude", [
    "-p", "--output-format", "stream-json", "--verbose",
    "--include-partial-messages",                          // <-- ADD THIS
    "--permission-mode", "dontAsk", "--allowedTools", "Read,Glob,Grep,Bash",
    "--max-turns", "15",
    "--model", "sonnet", "--no-session-persistence",
    task
  ], { env: cleanEnv() });
```

**Step 2: Rewrite the SSE event handler to avoid duplicate text**

With `--include-partial-messages`, tokens arrive via `stream_event` deltas AND the complete text arrives again in the `assistant` event. The handler MUST forward text only from `stream_event` and forward only `tool_use` from `assistant`. **Remove the `block.type === "text"` branch from the `assistant` handler** — if it remains, the frontend receives every piece of text twice (once token-by-token, once as a complete dump).

Replace the entire `createStreamParser` callback in the SSE pattern (lines ~407-426) with:

```typescript
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
```

**Step 3: Apply the same event handler rewrite to the WebSocket pattern**

Replace the `createStreamParser` callback in the WebSocket pattern (~lines 523-538) with the same structure: `stream_event` for text, `assistant` for tool_use only, `result` with `is_error`/`subtype` checks. Remove `{ type: "done", result: event }` and replace with the `is_error`/`subtype` handling.

**Step 4: Apply the same event handler rewrite to the Background Job pattern**

In the Background Job pattern (~lines 600-604), update the result handler to detect `error_max_turns`:

```typescript
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
```

**Step 5: Update the "Don't do this" for SSE**

Replace the last bullet in the SSE "Don't do this" section (~line 472-474):
```
- Don't assume text arrives via `stream_event` only — models with extended
  thinking (haiku-4.5) deliver text as complete blocks in `assistant` events.
  Always handle both `assistant` text blocks and `stream_event` deltas.
```

with:
```
- Don't omit `--include-partial-messages` — without it, text arrives as a
  single complete block in `assistant` events instead of token-by-token
  `stream_event` deltas. This makes streaming apps feel broken.
- Don't forward text from `assistant` events when using `--include-partial-messages`
  — the same text was already delivered token-by-token via `stream_event`. Only
  forward `tool_use` blocks from `assistant` events, or text will appear twice.
```

**Step 6: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/SKILL.md
git commit -m "Add --include-partial-messages to all streaming patterns (primary reliability fix)"
```

---

### Task 2: Fix code patterns for reliability

Fix the actual bugs and reliability gaps in SKILL.md. These are the changes that directly affect whether generated apps work on the first try.

**Files:**
- Modify: `skills/loom/SKILL.md`

**Step 1: Add Server Setup block**

After the `createStreamParser()` utility (~line 308) and before "Pattern: REST Endpoint" (~line 313), add a "Server Setup" block showing the boilerplate that all patterns share:

```markdown
#### Server Setup

Every pattern below assumes this baseline Express setup. Define it once at
the top of your server file, alongside the shared utilities above.

```typescript
import express from "express";
import cookieParser from "cookie-parser";
import path from "path";

const app = express();
app.use(express.json());
app.use(cookieParser());
app.use(express.static(path.join(__dirname, "public")));

// If frontend and server run on different ports (e.g., Vite dev + Express API):
// import cors from "cors";
// app.use(cors({ origin: "http://localhost:5173" }));

const PORT = process.env.PORT || 3456;
app.listen(PORT, () => console.log(`Server running on http://localhost:${PORT}`));
```
```

This addresses the review finding that cookie-parser and express.static are mentioned in "What to Generate" but never demonstrated in patterns.

**Step 2: Fix the REST pattern — remove duplicate `cleanEnv()` call**

Remove `const env = cleanEnv();` (standalone line ~327) — the inline `env: cleanEnv()` in `execFileSync` options already handles it.

**Step 3: Remove `let lineBuf = ""` from WebSocket and Background Job patterns**

These are dead variables — `createStreamParser` handles buffering internally. Remove:
- WebSocket pattern: `let lineBuf = "";` (~line 520)
- Background Job pattern: `let lineBuf = "";` (~line 597)

**Step 4: Fix the `extract()` function — stderr collection deadlock and race condition**

The current `extract()` collects stderr with `for await (const chunk of proc.stderr)` AFTER `for await (const chunk of proc.stdout)`, which deadlocks if stderr fills its buffer. Replace with concurrent collection.

**Important:** Register the `close` listener immediately after `spawn`, BEFORE writing to stdin — otherwise a very fast process could close before the listener is attached. Also retain a last-resort fallback for non-JSON `result` strings to avoid throwing on successful operations that return plain text.

```typescript
async function extract<T>(prompt: string, schema: object, timeoutMs = 30000): Promise<T> {
  const proc = spawn("claude", [
    "-p", "--model", "haiku", "--output-format", "json",
    "--json-schema", JSON.stringify(schema),
    "--tools", "", "--no-session-persistence",
    "--permission-mode", "dontAsk"
  ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv() });

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
      // result is not JSON — return the wrapper itself as a last resort.
      // This handles edge cases where structured_output is absent but
      // the operation succeeded with a plain-text result.
      return wrapper as T;
    }
  } finally {
    clearTimeout(timer);
  }
}
```

Changes from current code: (1) collect stderr concurrently via `.on("data")` instead of post-close `for await`; (2) register `close` listener before `stdin.write` to prevent race condition; (3) add `is_error` check; (4) retain `wrapper as T` fallback but only after trying `JSON.parse(wrapper.result)` first.

**Step 5: Remove cost from result event forwarding**

In the SSE pattern's result handler (already rewritten in Task 1) and the Error Surfacing Checklist example (~line 871-878), remove `cost: event.total_cost_usd` from the done event. Change `{ type: "done", data: event.structured_output }` to `{ type: "done", data: event.structured_output }` (this one is fine — it forwards structured data, not cost). But change `{ type: "done", cost: event.total_cost_usd }` (line ~424) to `{ type: "done" }`. Subscription users don't need cost tracking.

**Step 6: Add `JSON.parse` error handling in the frontend SSE code**

The `data = JSON.parse(line.slice(6))` call (~line 774) throws if the SSE line is malformed. Wrap in try/catch:

```javascript
try {
  const data = JSON.parse(line.slice(6));
  if (data.type === "token") output.textContent += data.text;
  else if (data.type === "done") { /* stream complete */ }
  else if (data.type === "error") output.textContent += `\n[Error: ${data.message}]`;
} catch { /* malformed SSE data — skip */ }
```

Also remove the cost display from the frontend: change `else if (data.type === "done") document.getElementById("cost").textContent = ...` (~line 776) to `else if (data.type === "done") { /* stream complete */ }`.

**Step 7: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/SKILL.md
git commit -m "Fix code patterns — remove dead code, fix stderr deadlock, add middleware, remove cost"
```

---

### Task 3: Fix stream event documentation

The event types table documents event types that don't match verified `claude -p` output and misses ones that do. Correct against verified facts from 2026-03-08 testing.

**Files:**
- Modify: `skills/loom/SKILL.md` — Stream-JSON Event Types section
- Modify: `skills/loom/references/cli-runtime-reference.md` — Event sequence section

**Step 1: Replace the event types table in SKILL.md**

Replace the existing Stream-JSON Event Types table and surrounding text (~lines 686-735) with:

```markdown
#### Stream-JSON Event Types

When using `--output-format stream-json --verbose --include-partial-messages`,
Claude emits newline-delimited JSON events. The full event sequence is:

```
system (init) → stream_event (message_start) → stream_event (content_block_start)
→ stream_event (content_block_delta) ×N → stream_event (content_block_stop)
→ assistant (complete block) → stream_event (message_delta) → stream_event (message_stop)
→ rate_limit_event → result
```

Without `--include-partial-messages`, only `system`, `assistant`, and `result`
events appear — no token-level streaming. **Always use `--include-partial-messages`
for streaming patterns.**

| Event Type | Shape | What It Means | Forward? |
|------------|-------|---------------|----------|
| `system` | `{type:"system", subtype:"init", session_id, model, tools, ...}` | Session started | Optional (extract session_id) |
| `stream_event` | `{type:"stream_event", event:{type:"content_block_delta", delta:{text:"..."}}}` | Incremental token (requires `--include-partial-messages`) | Yes (live text) |
| `assistant` | `{type:"assistant", message:{content:[{type:"text",text:"..."}, {type:"tool_use",...}], stop_reason:"end_turn"|"tool_use"|null}}` | Complete message with text and/or tool calls | Tool use only (text already streamed via `stream_event`) |
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

**Extended thinking models** (e.g., haiku-4.5) work correctly with this
approach. With `--include-partial-messages`, extended thinking models emit
`content_block_delta` events for thinking content, but these have EMPTY
`delta.text` — the `event.event?.delta?.text` check naturally skips them.
Real text tokens stream normally via `stream_event` for all models.
The `assistant` event contains the full thinking block (for logging) followed
by the text block — but since text was already streamed, only forward
`tool_use` from `assistant` events. Do NOT fall back to forwarding
`assistant` text blocks — this is unnecessary with `--include-partial-messages`
and would cause duplicate text.
```

**Step 2: Update the "Extracting text and tool use" example**

Replace the existing assistant extraction example (~lines 710-722) to match the new `--include-partial-messages` approach:

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

**Step 3: Add max-turns detection guidance**

After the event types section, add:

```markdown
**Detecting max-turns exhaustion:** When `--max-turns` is exceeded, the `result`
event has `subtype: "error_max_turns"` and `is_error: false`. Check `subtype`
to detect this — it's easy to miss because `is_error` is false:

```typescript
if (event.type === "result") {
  gotResult = true;
  if (event.is_error) {
    send({ type: "error", message: event.result });
  } else if (event.subtype === "error_max_turns") {
    send({ type: "warning", message: "Task incomplete — reached turn limit" });
  } else {
    send({ type: "done" });
  }
}
```
```

**Step 4: Update cli-runtime-reference.md event sequence**

Replace the event sequence example (lines ~96-101) with:

```
{"type":"system","subtype":"init","session_id":"...","model":"claude-sonnet-4-6","tools":[...]}
{"type":"assistant","message":{"content":[{"type":"text","text":"..."}]}}
{"type":"result","subtype":"success","is_error":false,"stop_reason":"end_turn","session_id":"...","num_turns":1}
```

Add a note after:
```
Note: With `--include-partial-messages`, additional `stream_event` events appear
between `system` and `assistant` containing token-level deltas. See the
Stream-JSON Event Types table in `SKILL.md` for the complete event sequence.
```

Remove the old `stream_event` line from the example — it is misleading without the `--include-partial-messages` flag context.

**Step 5: Remove `total_cost_usd` jq example from cli-runtime-reference.md**

Remove the line `claude -p --output-format json "query" | jq '.total_cost_usd'` from the json output section (line ~87). The field exists in the output but is not useful for subscription users. Keep `total_cost_usd` listed in the `result` event shape documentation (line ~81) — it's accurate — but don't promote extracting it.

**Step 6: Update the Node.js stream parsing example in cli-runtime-reference.md**

The Node.js stream parsing example (lines ~258-265) uses `event.type === 'stream_event'` with `event.event?.delta?.text`. This works only with `--include-partial-messages`. Add a note about the flag requirement:

```javascript
// Note: stream_event tokens require --include-partial-messages flag.
// Without it, text arrives only in assistant events.
const readline = require('readline');
for await (const line of readline.createInterface({ input: process.stdin })) {
  const event = JSON.parse(line);
  if (event.type === 'stream_event' && event.event?.delta?.text) {
    process.stdout.write(event.event.delta.text);
  }
  // The stream may contain additional event types — safe to ignore.
}
```

**Step 7: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/SKILL.md skills/loom/references/cli-runtime-reference.md
git commit -m "Fix event documentation against verified stream-json facts"
```

---

### Task 4: Fix cli-runtime-reference.md — add missing flags, remove cost guidance

**Files:**
- Modify: `skills/loom/references/cli-runtime-reference.md`

**Step 1: Add `--include-partial-messages` to the streaming section**

After the output streaming bash example (line ~255), add:

```markdown
### Token-level streaming (--include-partial-messages)

By default, `--output-format stream-json` delivers text only as complete
`assistant` blocks. To get token-by-token `stream_event` deltas, add
`--include-partial-messages`:

```bash
claude -p --output-format stream-json --verbose --include-partial-messages "Write a story"
```

Without this flag, streaming apps show text as a single dump instead of
typing progressively. **Required for SSE/WebSocket streaming patterns.**
```

**Step 2: Remove `maxBudgetUsd` from the Agent SDK TypeScript example**

In the TypeScript SDK example (line ~380), remove the line `    maxBudgetUsd: 10.00,` from the options object. This flag controls API dollar spend and has no meaningful effect for subscription users.

**Step 3: Reframe model selection comment**

In the Safety Controls section (lines ~409-410), change `# Model selection for cost` to `# Model selection` and reframe as speed/quality:

```bash
# Model selection
claude -p --model haiku "quick extraction"    # Fastest
claude -p --model sonnet "standard task"       # Balanced
claude -p --model opus "complex reasoning"     # Best quality
```

**Step 4: Add `--effort` to the Safety Controls section**

After the model selection block, add:

```bash
# Reasoning effort
claude -p --effort high "complex analysis task"   # Maximum reasoning
claude -p --effort low "simple extraction"          # Fast, less reasoning
```

**Step 5: Add `--add-dir` to the Tool Configuration section**

After the fine-grained allow/deny examples, add:

```bash
# Expand filesystem access beyond working directory
claude -p --add-dir /data/shared --add-dir /tmp/uploads "analyze files"
```

**Step 6: Add missing flags to the Quick Reference table**

Add these rows to the Quick Reference table:

```markdown
| Token streaming | `claude -p --output-format stream-json --verbose --include-partial-messages "query"` |
| Effort control | `claude -p --effort high "query"` |
| Extra directories | `claude -p --add-dir /path "query"` |
```

Do NOT add `--max-budget-usd`. It controls API dollar spend and is meaningless for subscription users.

**Step 7: Reframe the "Fast/cheap" row in Quick Reference**

Change:
```markdown
| Fast/cheap | `claude -p --model haiku "query"` |
```
to:
```markdown
| Fast model | `claude -p --model haiku "query"` |
```

**Step 8: Document `dontAsk` + empty result behavior**

In the Gotchas section, update item 7 to be more prominent about the silent failure:

```markdown
7. **`dontAsk` without `--allowedTools`** = silent failure. `dontAsk` auto-denies everything not explicitly allowed. Without `--allowedTools`, Claude has NO tools — it can reason but can't act. The result is an empty or incomplete response with NO error. Always pair `--permission-mode dontAsk` with `--allowedTools "Read,Bash,..."` or `--tools "Read,Bash,..."`.
```

**Step 9: Add `--include-partial-messages` to Gotchas**

Add a new gotcha:

```markdown
11. **`--include-partial-messages` required for token streaming** — Without this flag, `stream-json` output delivers text only as complete `assistant` blocks (no `stream_event` tokens). Add `--include-partial-messages` to get token-by-token deltas for SSE/WebSocket streaming.
```

**Step 10: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/references/cli-runtime-reference.md
git commit -m "Add --include-partial-messages flag, --effort, --add-dir; remove cost guidance"
```

---

### Task 5: Add reliability checklist, fix "What to Generate", trim frontmatter

**Files:**
- Modify: `skills/loom/SKILL.md`

**Step 1: Fix "Safety Defaults" cost reference**

Change "no human sitting at a terminal to approve tool use or notice runaway costs" (~line 218) to "no human sitting at a terminal to approve tool use".

**Step 2: Trim the frontmatter description**

Replace the current 6-line description (lines 2-13) with a shorter version. Keep the same trigger phrases but cut word count:

```yaml
---
name: loom
description: >
  Build applications where Claude Code CLI (`claude -p`) or the Agent SDK is the
  runtime — a server spawns Claude processes that power a custom interface.
  Triggers: "build an app that uses Claude", "Claude as backend/runtime",
  "wrap claude -p", or any app needing Claude's agentic capabilities through
  a purpose-built interface. NOT for Anthropic API apps or chat replicas.
---
```

**Step 3: Add first-run reliability checklist**

After the "What to Generate" numbered list (after item 4 "A one-liner to run it"), add:

```markdown
### First-Run Reliability Checklist

These are the silent-failure modes — things that break with NO error message.
The inline patterns above demonstrate correct handling for each, but they're
easy to miss or deviate from when generating a new app.

- [ ] `--include-partial-messages` is on every streaming spawn — without it, text dumps as a single block instead of streaming token-by-token
- [ ] Text is forwarded from `stream_event` only, NOT from `assistant` text blocks — otherwise every token appears twice
- [ ] `cleanEnv()` is called on every `spawn`/`execFileSync` — without it, Claude processes fail silently when the server runs inside Claude Code
- [ ] `--permission-mode dontAsk` is paired with `--allowedTools` or `--tools` — without allowed tools, Claude produces an empty result with NO error
- [ ] `subtype === "error_max_turns"` is checked on result events — this fires with `is_error: false`, so unchecked it looks like success
- [ ] `express.json()` middleware is applied before any route that reads `req.body` — without it, `req.body` is `undefined` and the spawn gets an empty prompt
- [ ] 401 responses in the frontend redirect to the setup screen — in-memory sessions are wiped on server restart, and a 401 fed to the SSE parser fails silently
```

This checklist covers only the failure modes that produce no error — the ones where
the app appears to work but produces wrong or missing output. Patterns that fail
loudly (missing stderr capture, no `gotResult` guard) are already obvious from the
inline code and don't need a checklist entry.

**Step 4: Align "What to Generate" list with demonstrated patterns**

Update item 1 to reference the Server Setup block:

Change item 1 from:
```
1. **`server.ts`** — Express server with cookie-parser, session store, ...
```
to:
```
1. **`server.ts`** — Express server using the Server Setup block above
   (express, cookie-parser, express.static). Includes session store,
   `requireAuth` middleware, OAuth endpoints (`/api/oauth/start`,
   `/api/oauth/exchange`, `/api/health`, `/api/logout`), and your app's
   endpoint pattern(s). The exchange endpoint must fetch the user's
   profile from `https://api.anthropic.com/v1/me` and store it in
   the session. Each spawn uses `spawnEnvForUser()` to inject the
   requesting user's token.
```

Change item 3 from:
```
3. **`package.json`** — Dependencies (including `cookie-parser`) and start script
```
to:
```
3. **`package.json`** — Dependencies (`express`, `cookie-parser`, `uuid`,
   and `cors` if frontend/server are separate origins) and a start script
```

**Step 5: Remove any remaining cost/budget references**

Run: `grep -n "cost\|total_cost_usd\|budget\|billing" skills/loom/SKILL.md`
Expected: Only the `total_cost_usd` field in the result event shape documentation (it's accurate and labelled as not-for-UI). No matches in code examples, "What to Generate", or frontend patterns. Fix any that remain.

**Step 6: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/SKILL.md
git commit -m "Add reliability checklist, trim frontmatter, align What to Generate with patterns"
```

---

### Task 6: Final verification pass

**Step 1: Verify `--include-partial-messages` is in all streaming spawns**

Run: `grep -n "stream-json" skills/loom/SKILL.md`
Every match that is inside a `spawn` call should have `--include-partial-messages` in the same args array (check surrounding lines).

Run: `grep -c "include-partial-messages" skills/loom/SKILL.md`
Expected: At least 4 matches (SSE, WebSocket, Background Job, Persistent Session).

**Step 2: Verify no duplicate text forwarding**

Run: `grep -n 'block.type === "text"' skills/loom/SKILL.md`
Expected: Zero matches inside any `createStreamParser` callback that also has a `stream_event` handler. The only remaining `block.type === "text"` references should be in the event types documentation or in patterns that do NOT use `--include-partial-messages` (REST pattern uses `--output-format json`, not streaming).

**Step 3: Verify dead code is removed**

Run: `grep -n "lineBuf" skills/loom/SKILL.md`
Expected: No matches (dead variable removed from WebSocket and Background Job patterns).

**Step 4: Verify `extract()` registers close listener before stdin.write**

Visually confirm in the `extract()` function that `proc.on("close", r)` appears BEFORE `proc.stdin.write(prompt)`.

**Step 5: Verify cost references are appropriate**

Run: `grep -n "total_cost_usd" skills/loom/SKILL.md`
Expected: Only in the result event shape documentation row, with the note about not surfacing it in UI. Not in any code that forwards data to the frontend.

Run: `grep -rn "cost\|billing\|budget" skills/loom/SKILL.md`
Expected: Only the `total_cost_usd` documentation. No cost display in frontend code, no budget flags.

**Step 6: Verify `rate_limit_event` is documented**

Run: `grep -n "rate_limit_event" skills/loom/SKILL.md`
Expected: At least 2 matches (in the event sequence diagram and in the event types table).

**Step 7: Verify `stop_reason` is documented**

Run: `grep -n "stop_reason" skills/loom/SKILL.md`
Expected: At least 1 match in the result event shape and/or the notes below the event table.

**Step 8: Verify cookie-parser and express.static are demonstrated**

Run: `grep -n "cookieParser\|cookie-parser\|express.static" skills/loom/SKILL.md`
Expected: At least one match each in the Server Setup block.

**Step 9: Verify `--include-partial-messages` is in cli-runtime-reference.md**

Run: `grep -n "include-partial-messages" skills/loom/references/cli-runtime-reference.md`
Expected: At least 3 matches (streaming section, quick reference, gotchas).

**Step 10: Verify no `maxBudgetUsd` or `max-budget-usd` remain**

Run: `grep -rn "maxBudget\|max.budget" skills/loom/references/cli-runtime-reference.md`
Expected: No matches.

**Step 11: Verify extended thinking note is updated**

Run: `grep -n "extended thinking" skills/loom/SKILL.md`
Expected: The note should mention that `--include-partial-messages` makes `stream_event` work for ALL models including extended thinking, and that thinking deltas have empty text. Should NOT say to "handle both `assistant` text blocks and `stream_event` deltas" or to "fall back" to assistant text — that guidance is obsolete.

**Step 12: Commit if any cleanup needed**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add -A
git commit -m "Final verification pass — fix any remaining inconsistencies"
```
