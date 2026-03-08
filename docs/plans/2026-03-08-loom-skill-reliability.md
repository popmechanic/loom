# Loom Skill Reliability Overhaul — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use trycycle-executing to implement this plan task-by-task.

**Goal:** Restructure the loom skill so apps generated from it work on the first try — fix the actual reliability problems (wrong event shapes, dead code paths, missing middleware/error handling), restructure via progressive disclosure to stay under the 500-line SKILL.md guideline, and correct documentation against verified `claude -p` output.

**Architecture:** The current 1,438-line SKILL.md becomes a ~400-line orchestration document covering design conversation, architecture, safety defaults, and the "What to Generate" checklist. All code patterns (REST, SSE, WebSocket, Background Job, Parallel, Advanced) move to a new `references/server-patterns-reference.md`. The event types table and all stream-json documentation are corrected against verified `claude -p` output. The narrative/design conversation sections are preserved — they serve an important skill-design purpose (setting degrees of freedom for how Claude approaches design questions).

**Tech Stack:** Markdown (SKILL.md), TypeScript code examples, `claude -p` CLI

---

### Task 1: Verify actual `claude -p` stream-json event shapes

Before changing any documentation, capture ground truth from `claude -p --output-format stream-json --verbose`. This task produces a verified reference that all subsequent tasks use.

**Files:**
- No files modified — this is a verification-only task

**Step 1: Run a simple no-tools query**

```bash
echo "Say hello" | claude -p --output-format stream-json --verbose --tools "" --no-session-persistence --max-turns 1 2>/dev/null > /tmp/loom-verify-events.txt
```

**Step 2: Run a tool-using query that hits max-turns**

```bash
echo "List every file recursively and analyze each one" | claude -p --output-format stream-json --verbose --no-session-persistence --max-turns 1 --permission-mode bypassPermissions --model haiku 2>/dev/null > /tmp/loom-verify-maxturns.txt
```

**Step 3: Document verified event shapes**

From the test output, confirm these facts (all verified during plan writing):

1. **No `stream_event` tokens are emitted** — text arrives as complete blocks in `assistant` events. The `stream_event` code path in current examples is dead code.
2. **No `content_block_start` or `content_block_stop` events** exist in `claude -p` output.
3. **No separate `tool_result` event type** — tool results arrive as `user` events with `tool_use_result` field.
4. **`result` event shape**: `{type:"result", subtype:"success"|"error_max_turns", stop_reason:"end_turn"|"tool_use", is_error:boolean, total_cost_usd:number, session_id, usage, num_turns}`.
5. **Max-turns detection**: When `--max-turns` is exceeded, `subtype` is `"error_max_turns"`, `is_error` is `false`, `stop_reason` is `"tool_use"` (not `"max_turns"`).
6. **`rate_limit_event`** exists but is internal — not useful to forward.
7. **`system` event subtypes include**: `init`, `hook_started`, `hook_response`.

These facts override any assumptions in the skill. If re-verification shows different results, update the plan accordingly.

**Step 4: No commit needed — this is input for subsequent tasks**

---

### Task 2: Create `references/server-patterns-reference.md` with all code patterns

Move the code-heavy sections out of SKILL.md into a dedicated reference file. This is the structural fix that brings SKILL.md under the 500-line guideline.

**Files:**
- Create: `skills/loom/references/server-patterns-reference.md`

**Step 1: Create the reference file**

Extract these sections from SKILL.md into the new file, preserving their content exactly (we fix bugs in later tasks):

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
git add skills/loom/references/server-patterns-reference.md
git commit -m "Create server-patterns-reference.md from SKILL.md code sections"
```

---

### Task 3: Slim SKILL.md to orchestration document

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

Replace the current 6-line description with:

```yaml
---
name: loom
description: >
  Build applications where Claude Code CLI (`claude -p`) or the Agent SDK is the
  runtime — a server spawns Claude processes that power a custom interface.
---
```

**Step 3: Verify line count**

Run: `wc -l skills/loom/SKILL.md`
Expected: Under 500 lines (target: ~300-400).

**Step 4: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Restructure SKILL.md — move code patterns to reference file"
```

---

### Task 4: Fix the stream event documentation against verified output

The event types table in the patterns reference (moved from SKILL.md in Task 2) documents event types that don't exist and misses ones that do. Fix it against the verified output from Task 1.

**Files:**
- Modify: `skills/loom/references/server-patterns-reference.md` — Stream-JSON Event Types section
- Modify: `skills/loom/references/cli-runtime-reference.md` — Event sequence section

**Step 1: Replace the event types table in server-patterns-reference.md**

```markdown
#### Stream-JSON Event Types

When using `--output-format stream-json --verbose`, Claude emits these event
types as newline-delimited JSON:

| Event Type | Shape | What It Means | Forward? |
|------------|-------|---------------|----------|
| `system` | `{type:"system", subtype:"init", session_id, model, tools, ...}` | Session started | Optional (extract session_id) |
| `assistant` | `{type:"assistant", message:{content:[{type:"text",text:"..."}, {type:"tool_use",...}]}}` | Complete message with text and/or tool calls | Yes (primary text delivery) |
| `user` | `{type:"user", message:{content:[{type:"tool_result",...}]}, tool_use_result}` | Tool result (auto-generated) | Optional (show tool output) |
| `result` | `{type:"result", subtype:"success"|"error_max_turns", is_error, stop_reason, session_id, num_turns, total_cost_usd, usage}` | Session complete | Yes (done signal) |

**Events you will NOT see** (contrary to some documentation):
- No `stream_event` with incremental token deltas — text arrives as complete
  blocks in `assistant` events. The `stream_event` type is not emitted by
  current `claude -p` versions.
- No `content_block_start` or `content_block_stop` wrapper events.
- No separate `tool_result` event type — tool results are inside `user` events.

**Extended thinking models** (e.g., haiku) emit `thinking` blocks in
`assistant` events before `text` blocks. These are internal reasoning and
should not be forwarded to the frontend.
```

**Step 2: Add max-turns detection guidance**

After the event types table, add:

```markdown
**Detecting max-turns exhaustion:** When `--max-turns` is exceeded, the `result`
event has `subtype: "error_max_turns"` and `is_error: false`. The `stop_reason`
will be `"tool_use"` (not `"max_turns"`). Check `subtype` to detect this:

```typescript
if (event.type === "result") {
  gotResult = true;
  if (event.is_error) {
    // Claude flagged the run as failed
    send({ type: "error", message: event.result });
  } else if (event.subtype === "error_max_turns") {
    // Task ran out of turns — output may be incomplete
    send({ type: "warning", message: "Task incomplete — reached turn limit" });
  } else {
    send({ type: "done" });
  }
}
```

**Step 3: Update cli-runtime-reference.md event sequence**

Replace the event sequence example (lines ~96-101):

```
{"type":"system","subtype":"init","session_id":"...","model":"claude-sonnet-4-6","tools":[...]}
{"type":"assistant","message":{"content":[{"type":"text","text":"..."}]}}
{"type":"result","subtype":"success","is_error":false,"stop_reason":"end_turn","session_id":"...","num_turns":1,"total_cost_usd":0.003}
```

Remove the `stream_event` line from the example — it does not appear in actual output.

**Step 4: Remove `total_cost_usd` from the cli-runtime-reference.md jq example**

Remove the line `claude -p --output-format json "query" | jq '.total_cost_usd'` from the json output section (line ~87). The field exists in the output but is not useful for subscription users and was a source of confusion.

Keep `total_cost_usd` in the `result` event shape documentation — it's accurate — but don't promote it as something to extract.

**Step 5: Commit**

```bash
git add skills/loom/references/server-patterns-reference.md skills/loom/references/cli-runtime-reference.md
git commit -m "Fix event documentation against verified claude -p output"
```

---

### Task 5: Fix the code patterns for reliability

Fix the actual bugs and reliability gaps in the code patterns now in `server-patterns-reference.md`. These are the changes that directly affect whether generated apps work on the first try.

**Files:**
- Modify: `skills/loom/references/server-patterns-reference.md`

**Step 1: Fix the SSE streaming pattern — remove dead `stream_event` path, add missing middleware and error handling**

In the SSE pattern, make these changes:

1. **Remove the `stream_event` branch** — it never fires. Replace with a comment:
   ```typescript
   // Note: text arrives as complete blocks in "assistant" events.
   // There are no incremental "stream_event" token deltas in current claude -p output.
   ```

2. **Add `express.json()` middleware** — the `app.post("/api/stream")` handler reads `req.body.task` but the pattern doesn't show `app.use(express.json())`. Add it at the top of the pattern (it's shown in the REST pattern but not SSE).

3. **Add `JSON.parse` error handling in the frontend** — the `data = JSON.parse(line.slice(6))` call throws if the SSE line is malformed (partial data, heartbeat corruption). Wrap in try/catch:
   ```javascript
   try {
     const data = JSON.parse(line.slice(6));
     if (data.type === "token") output.textContent += data.text;
     else if (data.type === "done") { /* stream complete */ }
     else if (data.type === "error") output.textContent += `\n[Error: ${data.message}]`;
   } catch { /* malformed SSE data — skip */ }
   ```

4. **Add `subtype: "error_max_turns"` detection** to the result handler:
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

5. **Remove cost from the result event** — change `{ type: "done", cost: event.total_cost_usd }` to `{ type: "done" }`.

6. **Remove cost display from the frontend** — change `else if (data.type === "done") document.getElementById("cost").textContent = ...` to `else if (data.type === "done") { /* stream complete */ }`.

**Step 2: Fix the WebSocket pattern — remove dead code**

1. Remove `let lineBuf = "";` (dead variable, never used).
2. Remove the `stream_event` branch — same as SSE.
3. Add `subtype: "error_max_turns"` detection to the result handler.

**Step 3: Fix the Background Job pattern — remove dead code**

1. Remove `let lineBuf = "";` (dead variable, never used).
2. Add `subtype: "error_max_turns"` detection.

**Step 4: Fix the REST pattern — remove duplicate `cleanEnv()` call**

Remove `const env = cleanEnv();` (line ~327 equivalent) — the inline `env: cleanEnv()` in `execFileSync` options already handles it.

**Step 5: Fix the `extract()` function — stderr collection bug**

Replace the `extract()` function with a version that collects stderr concurrently:

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

After the REST pattern's import block, add a note:

```typescript
// If your frontend is served from a different origin (e.g., Vite dev server
// on port 5173, API on port 3456), you need CORS headers:
// import cors from "cors";
// app.use(cors({ origin: "http://localhost:5173" }));
```

This is one of the most common first-run failures: the frontend can't reach the API because CORS blocks it.

**Step 7: Commit**

```bash
git add skills/loom/references/server-patterns-reference.md
git commit -m "Fix code patterns — remove dead stream_event paths, fix stderr bug, add CORS note"
```

---

### Task 6: Add missing CLI flags to the reference

The cli-runtime-reference.md is missing flags that exist in `claude --help`.

**Files:**
- Modify: `skills/loom/references/cli-runtime-reference.md`

**Step 1: Add `auto` to the permission modes table**

```markdown
| `auto` | Semi-autonomous: auto-approve most actions, prompt for risky ones | Balanced autonomy |
```

**Step 2: Add `--effort` to the Safety Controls section**

```bash
# Reasoning effort
claude -p --effort high "complex analysis task"   # Maximum reasoning
claude -p --effort low "simple extraction"          # Fast, less reasoning
```

**Step 3: Add `--add-dir` to the Tool Configuration section**

```bash
# Expand filesystem access beyond working directory
claude -p --add-dir /data/shared --add-dir /tmp/uploads "analyze files"
```

**Step 4: Add missing flags to the Quick Reference table**

```markdown
| Effort control | `claude -p --effort high "query"` |
| Extra directories | `claude -p --add-dir /path "query"` |
| Custom settings | `claude -p --settings ./settings.json "query"` |
```

Do NOT add `--max-budget-usd` to the Quick Reference or code examples. This flag controls API dollar spend and has no meaningful effect for subscription users. The skill targets subscription users, so including it sends a mixed message. If someone needs it, it's in `claude --help`.

**Step 5: Remove `maxBudgetUsd` from the Agent SDK TypeScript example**

In the TypeScript SDK example (line ~380), remove `maxBudgetUsd: 10.00` from the options. Same reasoning: subscription users don't need this.

**Step 6: Remove the cost-related jq example and cost-model comment**

In the Safety Controls section (line ~409-410), remove or rewrite the comment `# Model selection for cost` — reframe as model selection for speed/quality:

```bash
# Model selection
claude -p --model haiku "quick extraction"    # Fastest
claude -p --model sonnet "standard task"       # Balanced
claude -p --model opus "complex reasoning"     # Best quality
```

**Step 7: Commit**

```bash
git add skills/loom/references/cli-runtime-reference.md
git commit -m "Add missing CLI flags, remove subscription-irrelevant cost guidance"
```

---

### Task 7: Fix remaining cost references in SKILL.md

Remove the few cost references that remain in the slimmed SKILL.md after Task 3.

**Files:**
- Modify: `skills/loom/SKILL.md`

**Step 1: Fix "Safety Defaults" line**

If the text "no human sitting at a terminal to approve tool use or notice runaway costs" survived the restructuring, change it to "no human sitting at a terminal to approve tool use".

**Step 2: Verify no remaining cost references**

Run: `grep -n "cost\|total_cost_usd\|budget\|billing" skills/loom/SKILL.md`
Expected: No matches.

**Step 3: Commit (only if changes were made)**

```bash
git add skills/loom/SKILL.md
git commit -m "Remove remaining cost references from SKILL.md"
```

---

### Task 8: Add a reliability checklist to SKILL.md

The primary reliability problem is that Claude generates apps that deviate from the skill's patterns in ways that cause silent failures. Rather than hoping Claude follows every pattern perfectly, add an explicit checklist that Claude should verify before declaring an app complete. This addresses the review's core finding: the plan should identify *why* generated apps don't work.

**Files:**
- Modify: `skills/loom/SKILL.md` — add to the "What to Generate" section

**Step 1: Add a pre-delivery checklist**

After the "What to Generate" numbered list (after item 4), add:

```markdown
### First-Run Reliability Checklist

Before showing the app to the user, verify these common failure points:

**Server:**
- [ ] `express.json()` middleware is applied before any route that reads `req.body`
- [ ] `cleanEnv()` is called on every `spawn`/`execFileSync` — without it, Claude processes fail silently inside Claude Code
- [ ] `--permission-mode dontAsk` is paired with `--allowedTools` or `--tools` — without allowed tools, Claude can reason but can't act (silent failure)
- [ ] `--max-turns` is set on every spawn — without it, Claude can loop indefinitely
- [ ] All `JSON.parse` calls are wrapped in try/catch — killed processes emit partial JSON
- [ ] `is_error` is checked on result events before accessing `structured_output`
- [ ] `subtype === "error_max_turns"` is checked on result events — this fires with `is_error: false` so it's easy to miss
- [ ] stderr is captured on every spawn (at minimum `console.error`)
- [ ] The `gotResult` guard fires an error event if the process exits without a result
- [ ] If frontend and server are on different ports, CORS is configured

**Frontend:**
- [ ] SSE buffer splits on `"\n\n"` and keeps the last incomplete chunk
- [ ] `JSON.parse` on SSE data is wrapped in try/catch
- [ ] 401 responses redirect to the OAuth setup screen (in-memory sessions are wiped on server restart)
- [ ] The "done" handler doesn't reference `data.cost` (subscription users)
```

**Step 2: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Add first-run reliability checklist to What to Generate section"
```

---

### Task 9: Final verification pass

**Step 1: Verify SKILL.md line count**

Run: `wc -l skills/loom/SKILL.md`
Expected: Under 500 lines.

**Step 2: Verify no broken cross-references**

Run: `grep -n "stream_event\|content_block_start\|content_block_stop\|lineBuf\|total_cost_usd" skills/loom/SKILL.md`
Expected: No matches (these have all been moved to the patterns reference or removed).

Run: `grep -n "stream_event" skills/loom/references/server-patterns-reference.md`
Expected: Only in the "Events you will NOT see" section and in comments explaining why it's absent.

**Step 3: Verify no orphaned cost references**

Run: `grep -rn "cost\|billing" skills/loom/SKILL.md`
Expected: No matches.

Run: `grep -n "total_cost_usd" skills/loom/references/server-patterns-reference.md`
Expected: Only in the result event shape documentation (it's accurate), not in code examples.

**Step 4: Verify the patterns reference has correct event handling**

Scan all result-event handlers in the patterns reference for:
- `event.subtype === "error_max_turns"` (correct check)
- No `event.stop_reason === "max_turns"` (incorrect check)
- No `event.total_cost_usd` in forwarded data

Run: `grep -n "stop_reason.*max_turns\|max_turns.*stop_reason" skills/loom/references/server-patterns-reference.md`
Expected: No matches.

**Step 5: Commit if any cleanup needed**

```bash
git add -A
git commit -m "Final verification pass — fix any remaining inconsistencies"
```
