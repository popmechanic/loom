# Loom Skill Reliability Overhaul — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use trycycle-executing to implement this plan task-by-task.

**Goal:** Make apps generated from the loom skill work on the first try by fixing the actual reliability gaps: incomplete code patterns (missing middleware, missing error handling, missing CORS), inaccurate stream event documentation, dead code paths, and a "What to Generate" checklist that mentions things (cookie-parser, express.static) the patterns never demonstrate.

**Architecture:** Progressive disclosure — SKILL.md stays under 500 lines as a lean orchestration doc (design conversation, architecture, safety defaults, "What to Generate" checklist, reliability checklist). All code patterns move to `references/server-patterns-reference.md` where they are fixed for completeness. The existing "The Conversation" section is preserved — it teaches Claude HOW to approach design questions. Event documentation is corrected against verified facts. Cost/budget references are removed (subscription users only).

**Tech Stack:** Markdown (SKILL.md), TypeScript code examples, `claude -p` CLI

---

### Task 1: Create `references/server-patterns-reference.md` with all code patterns

Move the code-heavy sections out of SKILL.md into a dedicated reference file. This is the structural fix that brings SKILL.md under the 500-line guideline while preserving all the patterns in a referenceable location.

**Files:**
- Create: `skills/loom/references/server-patterns-reference.md`

**Step 1: Create the reference file**

Extract these sections from SKILL.md into the new file, preserving their content exactly (we fix bugs in Task 3):

1. **Shared Utilities** — `cleanEnv()` and `createStreamParser()` (SKILL.md lines ~245-308)
2. **Pattern: REST Endpoint** — including "Don't do this" (lines ~313-373)
3. **Pattern: SSE Streaming** — including "Don't do this" (lines ~375-474)
4. **Pattern: WebSocket Session** — including "Don't do this" (lines ~476-572)
5. **Pattern: Background Job** — including "Don't do this" (lines ~574-632)
6. **Pattern: Parallel Analysis** (lines ~634-684)
7. **Stream-JSON Event Types** — table and assistant event extraction (lines ~686-735)
8. **Frontend Layer** — streaming text display, structured result rendering (lines ~737-812)
9. **Error Handling** — mental model and checklist (lines ~815-882)
10. **Input Validation** (lines ~884-907)
11. **Temp Directory Cleanup** (lines ~909-931)
12. **Handling File Uploads and Drops** (lines ~932-1016)
13. **Advanced Patterns** — extract, persistent session, action markers (lines ~1018-1161)
14. **HTTP Hooks** (lines ~1162-1381)

The reference file should begin with:

```markdown
# Loom Server Patterns Reference

Complete code patterns for building Loom apps. Read `SKILL.md` for design
guidance and architecture decisions before reaching for these patterns.

Read `cli-runtime-reference.md` for the full `claude -p` flag reference.
Read `oauth-reference.md` for authentication setup.
```

**Step 2: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/references/server-patterns-reference.md
git commit -m "Create server-patterns-reference.md from SKILL.md code sections"
```

---

### Task 2: Slim SKILL.md to orchestration document

Replace the extracted code sections in SKILL.md with brief summaries and `references/` pointers. Preserve all non-code sections: frontmatter, intro, "Why This Matters", "The Conversation" (design questions), "The Possibility Space", architecture diagram, communication patterns table, model/performance table, "What to Generate", and deployment verification.

**Files:**
- Modify: `skills/loom/SKILL.md`

**Step 1: Replace the "Building It" code-heavy sections**

Remove lines ~245-1381 (everything from "#### Safety Defaults" through "HTTP Hooks") and replace with a compact summary that points to the reference:

```markdown
### Server Patterns

The server spawns `claude -p` processes and bridges results to the frontend.
For complete, copy-pasteable patterns with "Don't do this" guards, see
`references/server-patterns-reference.md`. Here's the mental model:

**Safety Defaults (non-negotiable for every pattern):**
- `--permission-mode dontAsk` — prevents Claude from hanging on approval prompts
- `--allowedTools "Read,Bash,..."` — required with `dontAsk` or Claude has no tools
- `--max-turns N` — prevents infinite loops (5 for one-shot, 10-15 for streaming)

**Shared Utilities:** `cleanEnv()` strips nesting-detection env vars so `claude -p`
can start when your server runs inside Claude Code. `createStreamParser()` buffers
chunked stdout into complete JSON lines. Both are defined in the patterns reference.

**Choose your pattern:**

| Pattern | When | Reference Section |
|---------|------|-------------------|
| REST + JSON | One-shot, structured output | "Pattern: REST Endpoint" |
| SSE Streaming | Live text to browser | "Pattern: SSE Streaming" |
| WebSocket | Bidirectional, multi-turn | "Pattern: WebSocket Session" |
| Background Job | Long-running with polling | "Pattern: Background Job" |
| Parallel Analysis | Batch concurrent items | "Pattern: Parallel Analysis" |
| HTTP Hooks | Tool lifecycle, browser-based permission approval | "HTTP Hooks" |

**Error handling:** Every pattern handles three failure modes — stderr capture,
non-zero exit code detection, and `is_error` checks on the result event. See
the Error Surfacing Checklist in the patterns reference.

**Frontend:** Use `fetch()` + `ReadableStream` for POST-based SSE (CSRF-safe).
Buffer SSE lines with `split("\n\n")` and keep the last incomplete chunk.
See the patterns reference for complete frontend code.
```

**Step 2: Trim the frontmatter description**

Replace the current 6-line description with a shorter version. Keep the same trigger phrases but cut word count:

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

**Step 3: Verify line count**

Run: `wc -l skills/loom/SKILL.md`
Expected: Under 500 lines (target: ~300-400).

**Step 4: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/SKILL.md
git commit -m "Restructure SKILL.md — move code patterns to reference file"
```

---

### Task 3: Fix code patterns for reliability

Fix the actual bugs and reliability gaps in `server-patterns-reference.md`. These are the changes that directly affect whether generated apps work on the first try. This is the most important task in the plan.

**Files:**
- Modify: `skills/loom/references/server-patterns-reference.md`

**Step 1: Fix the SSE streaming pattern**

Make these changes to the SSE pattern:

1. **Remove the `stream_event` branch** — text arrives as complete blocks in `assistant` events. The stream may contain additional event types beyond those documented; these can be safely ignored. Replace the `stream_event` branch with a comment:
   ```typescript
   // Note: text arrives as complete blocks in "assistant" events.
   // The stream may contain additional event types — safe to ignore.
   ```

2. **Confirm `express.json()` is shown** — the SSE pattern's `app.post("/api/stream")` reads `req.body.task`. The pattern must show `app.use(express.json())` before the route. (The REST pattern shows it; the SSE pattern does not — it assumes they share the same app but never says so.) Add `app.use(express.json());` explicitly at the top of the SSE pattern's code block.

3. **Add `subtype: "error_max_turns"` detection** to the result handler. Use `subtype` (verified to exist) and `is_error` (verified). Do NOT add `stop_reason` handling — the field exists but checking `subtype` is sufficient and clearer:
   ```typescript
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
   ```

4. **Remove cost from the result event** — change `{ type: "done", cost: event.total_cost_usd }` to `{ type: "done" }`. Subscription users.

5. **Add `JSON.parse` error handling in the frontend** — the `data = JSON.parse(line.slice(6))` call throws if the SSE line is malformed (partial data, heartbeat corruption). Wrap in try/catch:
   ```javascript
   try {
     const data = JSON.parse(line.slice(6));
     if (data.type === "token") output.textContent += data.text;
     else if (data.type === "done") { /* stream complete */ }
     else if (data.type === "error") output.textContent += `\n[Error: ${data.message}]`;
   } catch { /* malformed SSE data — skip */ }
   ```

6. **Remove cost display from the frontend** — change `else if (data.type === "done") document.getElementById("cost").textContent = ...` to `else if (data.type === "done") { /* stream complete */ }`.

**Step 2: Fix the WebSocket pattern**

1. Remove `let lineBuf = "";` (dead variable, never used — `createStreamParser` handles buffering).
2. Remove the `stream_event` branch — same as SSE.
3. Add `subtype: "error_max_turns"` detection to the result handler.

**Step 3: Fix the Background Job pattern**

1. Remove `let lineBuf = "";` (dead variable, never used).
2. Add `subtype: "error_max_turns"` detection to the result handler — set status to `"incomplete"` instead of `"complete"`.

**Step 4: Fix the REST pattern — remove duplicate `cleanEnv()` call**

Remove `const env = cleanEnv();` (standalone line) — the inline `env: cleanEnv()` in `execFileSync` options already handles it.

**Step 5: Fix the `extract()` function — stderr collection bug**

The current `extract()` collects stderr with `for await (const chunk of proc.stderr)` AFTER `for await (const chunk of proc.stdout)`, which deadlocks if stderr fills its buffer. Replace with concurrent collection:

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

  let stdout = "";
  let stderr = "";
  proc.stdout.on("data", (chunk) => { stdout += chunk.toString(); });
  proc.stderr.on("data", (chunk) => { stderr += chunk.toString(); });

  try {
    const exitCode = await new Promise<number | null>(r => proc.on("close", r));
    if (exitCode !== 0) {
      throw new Error(`Extraction failed (exit ${exitCode}): ${stderr.slice(0, 200)}`);
    }
    const wrapper = JSON.parse(stdout);
    if (wrapper.is_error) throw new Error(wrapper.result);
    if (wrapper.structured_output) return wrapper.structured_output as T;
    return JSON.parse(wrapper.result) as T;
  } finally {
    clearTimeout(timer);
  }
}
```

Changes: collect stderr concurrently via `.on("data")` instead of post-close `for await`; add `is_error` check; remove the fallback `return wrapper as T` which masked parse errors.

**Step 6: Add CORS note to Express patterns**

After the REST pattern's import block, add a comment:

```typescript
// If your frontend is served from a different origin (e.g., Vite dev server
// on port 5173, API on port 3456), you need CORS headers:
// import cors from "cors";
// app.use(cors({ origin: "http://localhost:5173" }));
//
// For single-origin apps (express.static serves the frontend), CORS is not needed.
```

**Step 7: Add `express.static` setup to the patterns**

After the shared utilities section, before the first pattern, add a "Server Setup" block showing the boilerplate that all patterns share:

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

This addresses the review finding that cookie-parser and express.static are mentioned in "What to Generate" but never demonstrated in patterns.

**Step 8: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/references/server-patterns-reference.md
git commit -m "Fix code patterns — remove dead code, fix stderr bug, add CORS/static/middleware"
```

---

### Task 4: Fix stream event documentation

The event types table documents event types that don't exist in `claude -p` output and misses ones that do. Correct it against verified facts.

**Files:**
- Modify: `skills/loom/references/server-patterns-reference.md` — Stream-JSON Event Types section
- Modify: `skills/loom/references/cli-runtime-reference.md` — Event sequence section

**Step 1: Replace the event types table in server-patterns-reference.md**

Replace the existing table and surrounding text with:

```markdown
#### Stream-JSON Event Types

When using `--output-format stream-json --verbose`, Claude emits these event
types as newline-delimited JSON:

| Event Type | Shape | What It Means | Forward? |
|------------|-------|---------------|----------|
| `system` | `{type:"system", subtype:"init", session_id, model, tools, ...}` | Session started | Optional (extract session_id) |
| `assistant` | `{type:"assistant", message:{content:[{type:"text",text:"..."}, {type:"tool_use",...}]}}` | Complete message with text and/or tool calls | Yes (primary text delivery) |
| `user` | `{type:"user", message:{content:[{type:"tool_result",...}]}}` | Tool result (auto-generated) | Optional (show tool output) |
| `result` | `{type:"result", subtype:"success"|"error_max_turns", is_error, session_id, num_turns, usage}` | Session complete | Yes (done signal) |

The stream may contain additional event types not listed here (internal
events, rate limit signals, etc.). These can be safely ignored — only the
four types above are needed for app development.

**Extended thinking models** (e.g., haiku) emit `thinking` blocks in
`assistant` events before `text` blocks. These are internal reasoning and
should not be forwarded to the frontend.
```

**Step 2: Add max-turns detection guidance**

After the event types table, add:

```markdown
**Detecting max-turns exhaustion:** When `--max-turns` is exceeded, the `result`
event has `subtype: "error_max_turns"` and `is_error: false`. Check `subtype`
to detect this:

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

**Step 3: Update cli-runtime-reference.md event sequence**

Replace the event sequence example (lines ~96-101) with:

```
{"type":"system","subtype":"init","session_id":"...","model":"claude-sonnet-4-6","tools":[...]}
{"type":"assistant","message":{"content":[{"type":"text","text":"..."}]}}
{"type":"result","subtype":"success","is_error":false,"session_id":"...","num_turns":1}
```

Remove the `stream_event` line from the example — it does not appear in actual output.

**Step 4: Remove `total_cost_usd` jq example from cli-runtime-reference.md**

Remove the line `claude -p --output-format json "query" | jq '.total_cost_usd'` from the json output section (line ~87). The field exists in the output but is not useful for subscription users and was a source of confusion.

Keep `total_cost_usd` listed in the `result` event shape documentation — it's accurate — but don't promote it as something to extract.

**Step 5: Update the streaming section Node.js example in cli-runtime-reference.md**

The Node.js stream parsing example (lines ~258-265) uses `event.type === 'stream_event'` with `event.event?.delta?.text`. This accessor does work for extracting text when present. Update the example to handle the primary `assistant` event type and add a note about additional events:

```javascript
const readline = require('readline');
for await (const line of readline.createInterface({ input: process.stdin })) {
  const event = JSON.parse(line);
  if (event.type === 'assistant' && event.message?.content) {
    for (const block of event.message.content) {
      if (block.type === 'text') process.stdout.write(block.text);
    }
  }
  // The stream may contain additional event types — safe to ignore.
}
```

**Step 6: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/references/server-patterns-reference.md skills/loom/references/cli-runtime-reference.md
git commit -m "Fix event documentation against verified stream-json facts"
```

---

### Task 5: Fix cli-runtime-reference.md — add missing flags, remove cost guidance

**Files:**
- Modify: `skills/loom/references/cli-runtime-reference.md`

**Step 1: Remove `maxBudgetUsd` from the Agent SDK TypeScript example**

In the TypeScript SDK example (line ~380), remove the line `    maxBudgetUsd: 10.00,` from the options object. This flag controls API dollar spend and has no meaningful effect for subscription users. The skill targets subscription users exclusively.

**Step 2: Reframe model selection comment**

In the Safety Controls section (line ~409-410), change `# Model selection for cost` to `# Model selection` and reframe the comments as speed/quality rather than cost:

```bash
# Model selection
claude -p --model haiku "quick extraction"    # Fastest
claude -p --model sonnet "standard task"       # Balanced
claude -p --model opus "complex reasoning"     # Best quality
```

**Step 3: Add `--effort` to the Safety Controls section**

After the model selection block, add:

```bash
# Reasoning effort
claude -p --effort high "complex analysis task"   # Maximum reasoning
claude -p --effort low "simple extraction"          # Fast, less reasoning
```

**Step 4: Add `--add-dir` to the Tool Configuration section**

After the fine-grained allow/deny examples, add:

```bash
# Expand filesystem access beyond working directory
claude -p --add-dir /data/shared --add-dir /tmp/uploads "analyze files"
```

**Step 5: Add missing flags to the Quick Reference table**

Add these rows to the Quick Reference table:

```markdown
| Effort control | `claude -p --effort high "query"` |
| Extra directories | `claude -p --add-dir /path "query"` |
```

Do NOT add `--max-budget-usd`. It controls API dollar spend and is meaningless for subscription users. If someone needs it, it's in `claude --help`.

**Step 6: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/references/cli-runtime-reference.md
git commit -m "Add missing CLI flags, remove subscription-irrelevant cost guidance"
```

---

### Task 6: Fix remaining cost references and add reliability checklist to SKILL.md

**Files:**
- Modify: `skills/loom/SKILL.md`

**Step 1: Fix "Safety Defaults" line**

If the text "no human sitting at a terminal to approve tool use or notice runaway costs" survived the restructuring, change it to "no human sitting at a terminal to approve tool use".

**Step 2: Verify no remaining cost references**

Run: `grep -n "cost\|total_cost_usd\|budget\|billing" skills/loom/SKILL.md`
Expected: No matches.

**Step 3: Add a first-run reliability checklist to the "What to Generate" section**

After the "What to Generate" numbered list (after item 4 "A one-liner to run it"), add:

```markdown
### First-Run Reliability Checklist

Before showing the app to the user, verify these common failure points:

**Server:**
- [ ] `express.json()` middleware is applied before any route that reads `req.body`
- [ ] `cookieParser()` middleware is applied (required for session cookies)
- [ ] `express.static("public")` serves the frontend (or CORS is configured for separate origins)
- [ ] `cleanEnv()` is called on every `spawn`/`execFileSync` — without it, Claude processes fail silently inside Claude Code
- [ ] `--permission-mode dontAsk` is paired with `--allowedTools` or `--tools` — without allowed tools, Claude can reason but can't act (silent failure)
- [ ] `--max-turns` is set on every spawn — without it, Claude can loop indefinitely
- [ ] All `JSON.parse` calls are wrapped in try/catch — killed processes emit partial JSON
- [ ] `is_error` is checked on result events before accessing `structured_output`
- [ ] `subtype === "error_max_turns"` is checked on result events — this fires with `is_error: false` so it's easy to miss
- [ ] stderr is captured on every spawn (at minimum `console.error`)
- [ ] The `gotResult` guard fires an error event if the process exits without a result
- [ ] If frontend and server are on different ports, CORS is configured
- [ ] `package.json` includes all dependencies: `express`, `cookie-parser`, `uuid` (and `cors` if needed)

**Frontend:**
- [ ] SSE buffer splits on `"\n\n"` and keeps the last incomplete chunk
- [ ] `JSON.parse` on SSE data is wrapped in try/catch
- [ ] 401 responses redirect to the OAuth setup screen (in-memory sessions are wiped on server restart)
- [ ] The "done" handler does NOT reference `data.cost` (subscription users)
```

This checklist addresses the primary reliability problem: Claude generates apps that deviate from the patterns in ways that cause silent failures. The checklist makes failure modes explicit so Claude can self-verify.

**Step 4: Align "What to Generate" list with demonstrated patterns**

Update the "What to Generate" list to match what's actually shown in the patterns. The current list mentions `cookie-parser` without showing the import. Fix item 1 to reference the Server Setup block in the patterns reference:

Change item 1 from:
```
1. **`server.ts`** — Express server with cookie-parser, session store, ...
```
to:
```
1. **`server.ts`** — Express server using the Server Setup block from
   `references/server-patterns-reference.md` (express, cookie-parser,
   express.static). Includes session store, `requireAuth` middleware,
   OAuth endpoints (`/api/oauth/start`, `/api/oauth/exchange`,
   `/api/health`, `/api/logout`), and your app's endpoint pattern(s).
   The exchange endpoint must fetch the user's profile from
   `https://api.anthropic.com/v1/me` and store it in the session.
   Each spawn uses `spawnEnvForUser()` to inject the requesting
   user's token.
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

**Step 5: Commit**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add skills/loom/SKILL.md
git commit -m "Add reliability checklist and align What to Generate with demonstrated patterns"
```

---

### Task 7: Final verification pass

**Step 1: Verify SKILL.md line count**

Run: `wc -l skills/loom/SKILL.md`
Expected: Under 500 lines.

**Step 2: Verify no broken cross-references**

Run: `grep -n "stream_event\|content_block_start\|content_block_stop\|lineBuf\|total_cost_usd" skills/loom/SKILL.md`
Expected: No matches (these have all been moved to the patterns reference or removed).

Run: `grep -n "stream_event" skills/loom/references/server-patterns-reference.md`
Expected: Only in comments explaining why it's absent (not in code that tries to handle it).

**Step 3: Verify no orphaned cost references**

Run: `grep -rn "cost\|billing\|budget" skills/loom/SKILL.md`
Expected: No matches.

Run: `grep -n "total_cost_usd" skills/loom/references/server-patterns-reference.md`
Expected: Only in the result event shape documentation (it's accurate), not in code examples that forward it to the frontend.

Run: `grep -n "max.budget\|maxBudget" skills/loom/references/cli-runtime-reference.md`
Expected: No matches.

**Step 4: Verify the patterns reference has correct event handling**

Scan all result-event handlers in the patterns reference:

Run: `grep -n "stop_reason.*max_turns\|max_turns.*stop_reason" skills/loom/references/server-patterns-reference.md`
Expected: No matches. (Max-turns is detected via `subtype`, not `stop_reason`.)

Run: `grep -n "event.total_cost_usd\|cost:" skills/loom/references/server-patterns-reference.md`
Expected: No matches in code that forwards data to the frontend.

**Step 5: Verify cookie-parser and express.static are demonstrated**

Run: `grep -n "cookieParser\|cookie-parser\|express.static" skills/loom/references/server-patterns-reference.md`
Expected: At least one match each in the Server Setup block.

**Step 6: Verify package.json dependencies are listed**

Run: `grep -n "cookie-parser" skills/loom/SKILL.md`
Expected: At least one match in the "What to Generate" section and/or reliability checklist.

**Step 7: Commit if any cleanup needed**

```bash
cd /Users/marcusestes/Websites/loom/.worktrees/loom-skill-reliability
git add -A
git commit -m "Final verification pass — fix any remaining inconsistencies"
```
