# Loom Skill Reliability Overhaul — Test Plan

## Strategy reconciliation

The agreed testing strategy was **subagent-based semantic review**: dispatch subagents to read the modified SKILL.md and cli-runtime-reference.md, and verify they teach correct patterns. This strategy was chosen because:

- There is NO runnable application code — the files being modified are markdown documentation containing teaching examples
- `claude -p` cannot be spawned from within Claude Code (nesting detection blocks it)
- The user is subscription-only (no API billing for test runs)
- All streaming behavior has been verified empirically and results are available as artifacts

The implementation plan confirms this approach is valid:
- Both modified files are markdown (SKILL.md, cli-runtime-reference.md)
- SKILL.md stays as a single comprehensive document (no structural split)
- All changes are to code patterns within markdown, event documentation tables, and prose guidance
- The verified stream outputs at `/tmp/loom-*.jsonl` serve as the ground truth

**Strategy holds as-is.** No adjustments needed. The harness is "read the file and check assertions against verified facts."

## Harness requirements

### Harness 1: Grep-based mechanical verification

**What it does:** Runs targeted grep/search commands against the modified files to verify presence/absence of specific strings, flag combinations, and patterns.

**What it exposes:** File content matching, line counts, pattern presence.

**Estimated complexity:** Zero — uses existing grep/search tools. No code to write.

**Which tests depend on it:** Tests 1-7 (all mechanical verification tests).

### Harness 2: Semantic review by reading

**What it does:** A reviewer reads the complete modified SKILL.md and cli-runtime-reference.md, then checks each assertion against the verified facts in `/tmp/loom-verified-facts.md`, `/tmp/loom-partial-test.jsonl`, and `/tmp/loom-haiku-stream.jsonl`.

**What it exposes:** Semantic correctness of documentation, internal consistency of code patterns, completeness of event handling.

**Estimated complexity:** Zero infrastructure. Requires reading and reasoning only.

**Which tests depend on it:** Tests 8-16 (all semantic/scenario tests).

---

## Test plan

### Scenario tests

**1. SSE streaming app teaches token-by-token streaming end-to-end**

- **Type:** scenario
- **Harness:** Semantic review
- **Preconditions:** Implementation plan Tasks 1-5 complete. SKILL.md modified in worktree.
- **Actions:** Read the SSE Streaming pattern section of SKILL.md. Trace the full data path: spawn args -> event handler -> SSE write -> frontend display.
- **Expected outcome:**
  - The `spawn("claude", [...])` call includes `--include-partial-messages` in its args array. *Source: verified fact — `/tmp/loom-partial-test.jsonl` shows `stream_event` tokens only appear with `--include-partial-messages`.*
  - The `createStreamParser` callback forwards text from `stream_event` events via `event.event?.delta?.text`. *Source: `/tmp/loom-partial-test.jsonl` line 8 shows `content_block_delta` with `delta.text` field nested inside `stream_event`.*
  - The `assistant` event handler forwards ONLY `tool_use` blocks, NOT `text` blocks. *Source: `/tmp/loom-partial-test.jsonl` line 27 shows the `assistant` event contains the complete text that was already streamed token-by-token — forwarding it would duplicate text.*
  - The frontend `JSON.parse(line.slice(6))` is wrapped in try/catch. *Source: plan Task 2 Step 6.*
  - The frontend does NOT display `cost` from `data.cost`. *Source: user requirement — subscription-only.*
  - The result handler checks `event.is_error` and `event.subtype === "error_max_turns"`. *Source: `/tmp/loom-verified-facts.md` — result event has `subtype` and `is_error` fields.*
- **Interactions:** Touches event documentation section (must be consistent), frontend SSE section, "Don't do this" list.

**2. WebSocket session pattern teaches multi-turn streaming with correct event filtering**

- **Type:** scenario
- **Harness:** Semantic review
- **Preconditions:** SKILL.md modified.
- **Actions:** Read the WebSocket Session pattern. Trace: spawn args -> session management -> event handler -> ws.send -> client.
- **Expected outcome:**
  - `spawn("claude", [...])` includes `--include-partial-messages`. *Source: same verified fact as Test 1.*
  - Text forwarded from `stream_event` only, not from `assistant` text blocks. *Source: same as Test 1.*
  - `tool_use` blocks forwarded WITH `block.input` (WebSocket can handle richer payloads). *Source: plan Task 1 Step 3 explicitly preserves this.*
  - Dead `let lineBuf = ""` variable is removed. *Source: plan Task 2 Step 3 — this variable is unused because `createStreamParser` handles buffering.*
  - Result handler includes `is_error` and `subtype` checks. *Source: verified facts.*
- **Interactions:** Shares event handling logic with SSE pattern — must be consistent.

**3. Background Job pattern correctly stores completion status including max-turns**

- **Type:** scenario
- **Harness:** Semantic review
- **Preconditions:** SKILL.md modified.
- **Actions:** Read the Background Job pattern. Trace: spawn -> event handler -> jobs map -> poll endpoint.
- **Expected outcome:**
  - `spawn("claude", [...])` includes `--include-partial-messages`. *Source: verified fact.*
  - Dead `let lineBuf = ""` variable is removed. *Source: plan Task 2 Step 3.*
  - Result handler distinguishes `is_error`, `subtype === "error_max_turns"`, and success. Stores separate statuses for each. *Source: verified facts — `subtype` field distinguishes these cases, and `is_error: false` for max-turns means a naive `is_error` check misses it.*
- **Interactions:** Jobs map status values must be consistent with poll endpoint expectations.

**4. Persistent Session pattern broadcasts filtered events without duplication**

- **Type:** scenario
- **Harness:** Semantic review
- **Preconditions:** SKILL.md modified.
- **Actions:** Read the Persistent Session pattern. Trace: spawn args -> event handler -> broadcast.
- **Expected outcome:**
  - `spawn("claude", [...])` includes `--include-partial-messages`. *Source: verified fact.*
  - The `createStreamParser` callback does NOT raw-broadcast every event. It filters: `stream_event` text -> broadcast as `{type:"token"}`, `assistant` -> forward only `tool_use`, `result` -> broadcast as `{type:"done"}` with error/max-turns handling. *Source: without filtering, both `stream_event` tokens AND `assistant` complete text get broadcast, causing duplicate display. Plan Task 1 Step 5.*
- **Interactions:** Uses bidirectional streaming (`--input-format stream-json`) — different from other patterns.

**5. A Claude instance reading the skill can build a working SSE app from scratch**

- **Type:** scenario
- **Harness:** Semantic review
- **Preconditions:** All tasks complete. SKILL.md finalized.
- **Actions:** Read SKILL.md from top to bottom as if you were a Claude instance receiving it as a skill. Trace the path a Claude instance would follow to build a new SSE streaming app: read "Building It" -> pick SSE pattern -> copy spawn args -> copy event handler -> copy frontend code -> generate package.json.
- **Expected outcome:**
  - Server Setup block exists and shows `express.json()`, `cookieParser()`, `express.static`. *Source: plan Task 2 Step 1. These were mentioned in "What to Generate" but never demonstrated.*
  - The "What to Generate" checklist references the Server Setup block. *Source: plan Task 5 Step 4.*
  - Reliability checklist exists and covers the silent-failure modes. *Source: plan Task 5 Step 3.*
  - No contradictory guidance exists (e.g., "handle both assistant text and stream_event" in one place while "only forward stream_event" in another). *Source: the old warning at lines 702-706 of current SKILL.md is now obsolete — verified that `--include-partial-messages` makes all models emit `stream_event` tokens.*
  - Following the patterns produces code with `cleanEnv()` on every spawn, `--permission-mode dontAsk` paired with `--allowedTools`, and `express.json()` before routes that read `req.body`. *Source: these are the silent failure modes identified in the plan.*
- **Interactions:** Exercises nearly every section of SKILL.md — this is the highest-value scenario test.

### Integration tests

**6. Event documentation table matches the verified stream output**

- **Type:** integration
- **Harness:** Semantic review + grep verification
- **Preconditions:** SKILL.md modified (Task 3 complete).
- **Actions:** Read the Stream-JSON Event Types table. Compare every event type, shape, and field against the verified outputs.
- **Expected outcome:**
  - `system` event shape includes `subtype:"init"`, `session_id`, `model`, `tools`. *Source: `/tmp/loom-partial-test.jsonl` line 5 (init event after hooks).*
  - `stream_event` shape shows `event.delta.text` for content_block_delta. *Source: `/tmp/loom-partial-test.jsonl` lines 8-26, actual field path is `event.delta.text` (the `delta` object has a `type:"text_delta"` and `text` field).*
  - `assistant` shape includes `stop_reason` field. *Source: `/tmp/loom-partial-test.jsonl` line 27, field `stop_reason: null` on the assistant message.*
  - `rate_limit_event` is documented with shape including `status`, `utilization`, `rateLimitType`, `isUsingOverage`, `resetsAt`. *Source: `/tmp/loom-partial-test.jsonl` line 31.*
  - `result` shape includes `subtype`, `is_error`, `stop_reason`, `session_id`, `num_turns`, `duration_ms`, `total_cost_usd`. *Source: `/tmp/loom-partial-test.jsonl` line 32 and `/tmp/loom-verified-facts.md`.*
  - `total_cost_usd` is noted as not meaningful for billing / not to surface in UI. *Source: user requirement — subscription-only.*
  - Table notes that `--include-partial-messages` is required for `stream_event` tokens. *Source: verified — without the flag, no `stream_event` events appear (see `/tmp/loom-stream-test.jsonl` vs `/tmp/loom-partial-test.jsonl`).*
  - `tool_result` and `compact` events are noted as carried forward / not re-verified. *Source: plan Task 3 Step 1 notes — tests used `--tools ""` and short prompts.*
- **Interactions:** This table is referenced by all four streaming patterns. Inconsistency here propagates to every generated app.

**7. cli-runtime-reference.md event sequence matches SKILL.md event table**

- **Type:** integration
- **Harness:** Semantic review + grep verification
- **Preconditions:** cli-runtime-reference.md modified (Task 3-4 complete).
- **Actions:** Read the stream-json event sequence example in cli-runtime-reference.md. Compare against the event types table in SKILL.md.
- **Expected outcome:**
  - The event sequence example uses verified event types (`system`, `assistant`, `result`). *Source: verified facts.*
  - A note explains that `stream_event` events appear with `--include-partial-messages`. *Source: plan Task 3 Step 4.*
  - The `stream_event` line from the old example is removed or contextualized. *Source: plan Task 3 Step 4 — it was misleading without `--include-partial-messages` context.*
  - The `total_cost_usd` jq extraction example is removed. *Source: plan Task 3 Step 5 — not useful for subscription users.*
  - The Node.js stream parsing example includes a note about `--include-partial-messages` being required. *Source: plan Task 3 Step 6.*
- **Interactions:** cli-runtime-reference.md and SKILL.md must agree on event types, shapes, and handling guidance.

**8. cli-runtime-reference.md has all new flags documented**

- **Type:** integration
- **Harness:** Grep verification
- **Preconditions:** cli-runtime-reference.md modified (Task 4 complete).
- **Actions:** Search for flag documentation coverage.
- **Expected outcome:**
  - `--include-partial-messages` appears in: streaming section (new subsection), Quick Reference table, and Gotchas. Minimum 3 matches. *Source: plan Task 4 Steps 1, 6, 9.*
  - `--effort` appears with `high` and `low` examples. *Source: plan Task 4 Step 4.*
  - `--add-dir` appears with usage example. *Source: plan Task 4 Step 5.*
  - `maxBudgetUsd` / `max-budget-usd` does NOT appear anywhere. *Source: plan Task 4 Step 2 — meaningless for subscription users.*
  - Quick Reference table has rows for token streaming, effort control, extra directories. *Source: plan Task 4 Step 6.*
  - "Fast/cheap" is reframed to "Fast model". *Source: plan Task 4 Step 7.*
  - Model selection comment says "Model selection" not "Model selection for cost". *Source: plan Task 4 Step 3.*
- **Interactions:** Quick Reference table is the most-referenced section for Claude instances generating `claude -p` commands.

### Invariant tests

**9. No streaming spawn exists without `--include-partial-messages`**

- **Type:** invariant
- **Harness:** Grep verification
- **Preconditions:** All tasks complete.
- **Actions:** `grep -n "stream-json" skills/loom/SKILL.md` — for every match inside a `spawn` call, verify `--include-partial-messages` appears in the same args array (within surrounding 5 lines).
- **Expected outcome:** Every `spawn` call using `--output-format stream-json` also includes `--include-partial-messages`. Minimum 4 instances (SSE, WebSocket, Background Job, Persistent Session). *Source: verified fact — without it, no token streaming.*
- **Interactions:** N/A — purely mechanical.

**10. No streaming event handler forwards `assistant` text blocks**

- **Type:** invariant
- **Harness:** Grep verification
- **Preconditions:** All tasks complete.
- **Actions:** `grep -n 'block.type === "text"' skills/loom/SKILL.md` — check that no match appears inside a `createStreamParser` callback that also has a `stream_event` handler.
- **Expected outcome:** Zero matches inside streaming handlers. The only remaining `block.type === "text"` references should be in the event documentation or in patterns that don't use streaming (REST uses `--output-format json`). *Source: with `--include-partial-messages`, forwarding `assistant` text duplicates everything.*
- **Interactions:** N/A — purely mechanical.

**11. No dead `lineBuf` variables remain**

- **Type:** invariant
- **Harness:** Grep verification
- **Preconditions:** All tasks complete.
- **Actions:** `grep -n "lineBuf" skills/loom/SKILL.md`
- **Expected outcome:** Zero matches. *Source: `createStreamParser` handles buffering internally; `lineBuf` was always dead code.*
- **Interactions:** N/A — purely mechanical.

**12. No cost/budget references in code examples or frontend patterns**

- **Type:** invariant
- **Harness:** Grep verification
- **Preconditions:** All tasks complete.
- **Actions:**
  - `grep -n "total_cost_usd" skills/loom/SKILL.md` — should only appear in the event types table documentation, with a note about not surfacing in UI.
  - `grep -n "cost\|billing\|budget" skills/loom/SKILL.md` — should match only `total_cost_usd` documentation. No cost display in frontend code, no budget flags.
  - `grep -n "maxBudget\|max.budget" skills/loom/references/cli-runtime-reference.md` — zero matches.
- **Expected outcome:** `total_cost_usd` appears only in result event shape docs with the "not meaningful for billing" note. No `data.cost`, no `$${data.cost.toFixed(4)}`, no `maxBudgetUsd`. *Source: user requirement — subscription-only.*
- **Interactions:** N/A — purely mechanical.

**13. `extract()` function has correct execution order**

- **Type:** invariant
- **Harness:** Semantic review
- **Preconditions:** Task 2 Step 4 complete.
- **Actions:** Read the `extract()` function. Check execution order.
- **Expected outcome:**
  - `proc.on("close", r)` is registered BEFORE `proc.stdin.write(prompt)`. *Source: plan Task 2 Step 4 — a very fast process could close before the listener is attached.*
  - stderr is collected concurrently via `.on("data")`, not sequentially via `for await` after stdout. *Source: plan Task 2 Step 4 — sequential collection deadlocks if stderr fills its buffer.*
  - `is_error` is checked on the parsed wrapper. *Source: verified facts — `is_error` field exists on result.*
  - `wrapper as T` fallback is retained after `JSON.parse(wrapper.result)` fails. *Source: plan review finding 5 — removing it regresses on non-JSON results.*
  - `--tools ""` is present in the args array. *Source: plan review finding 1 (retracted but flagged) — `dontAsk` without tools is silent failure.*
- **Interactions:** `extract()` is used by Structured Extraction pattern, which is a teaching example for lightweight data extraction.

### Boundary tests

**14. Extended thinking model guidance is correct**

- **Type:** boundary
- **Harness:** Semantic review
- **Preconditions:** Task 3 complete.
- **Actions:** Read the extended thinking note in the event types section. Search for `grep -n "extended thinking" skills/loom/SKILL.md`.
- **Expected outcome:**
  - The note explains that `--include-partial-messages` makes `stream_event` work for ALL models, including extended thinking. *Source: verified — `/tmp/loom-haiku-stream.jsonl` shows haiku (extended thinking) emits `thinking_delta` events (lines 8-15) followed by `text_delta` events with actual text.*
  - Thinking deltas have `type: "thinking_delta"` (no `text` field on the delta), so `event.event?.delta?.text` correctly skips them. *Source: `/tmp/loom-haiku-stream.jsonl` lines 8-15 — `delta.type` is `"thinking_delta"` with `thinking` field, not `text` field.*
  - The note does NOT say to "handle both assistant text blocks and stream_event deltas" or to "fall back" to assistant text. *Source: this guidance is obsolete with `--include-partial-messages`.*
  - The note does NOT say to forward `assistant` text blocks for extended thinking models. *Source: same — the old dual-path guidance is removed.*
- **Interactions:** This note directly affects whether generated apps work with `--model haiku`.

**15. `dontAsk` silent failure is prominently documented**

- **Type:** boundary
- **Harness:** Semantic review + grep verification
- **Preconditions:** Tasks 4-5 complete.
- **Actions:** Read the Safety Defaults section in SKILL.md. Read Gotcha 7 in cli-runtime-reference.md. Check reliability checklist.
- **Expected outcome:**
  - Safety Defaults section explains that `dontAsk` without `--allowedTools` means "no tools at all" and "the failure is silent." *Source: current SKILL.md lines 225-229 — this must be preserved.*
  - Gotcha 7 in cli-runtime-reference.md explicitly says "silent failure." *Source: plan Task 4 Step 8.*
  - Reliability checklist includes a line about pairing `dontAsk` with `--allowedTools` or `--tools`. *Source: plan Task 5 Step 3.*
- **Interactions:** Every streaming pattern uses `dontAsk` — if this guidance is unclear, every generated app is at risk.

### Regression tests

**16. Existing correct patterns are preserved**

- **Type:** regression
- **Harness:** Semantic review
- **Preconditions:** All tasks complete.
- **Actions:** Verify that correctly-functioning patterns from the current SKILL.md are not broken by the changes.
- **Expected outcome:**
  - REST pattern still uses `execFileSync` with args array (not string interpolation). *Source: current SKILL.md lines 346-350.*
  - `cleanEnv()` still strips `CLAUDECODE`, `CLAUDE_CODE_ENTRYPOINT`, and `CMUX_*` vars. *Source: current SKILL.md lines 264-279. Verified in testing session.*
  - `createStreamParser()` still uses `TextDecoder` with `{ stream: true }`. *Source: current SKILL.md lines 290-307.*
  - Heartbeat in SSE pattern (`setInterval(() => res.write(":heartbeat\n\n"), 5000)`) is preserved. *Source: current SKILL.md lines 393-395.*
  - `res.on("close")` (not `req.on("close")`) for SSE cleanup is preserved. *Source: current SKILL.md lines 451-455.*
  - `gotResult` guard in `close` handler is preserved. *Source: current SKILL.md lines 437-442.*
  - Error Surfacing Checklist section is preserved. *Source: current SKILL.md lines 838-882.*
  - OAuth guidance references and `references/oauth-reference.md` cross-references are preserved. *Source: current SKILL.md lines 183-206.*
  - File upload handling section is preserved unchanged. *Source: current SKILL.md lines 932-989.*
  - HTTP Hooks section is preserved unchanged. *Source: current SKILL.md lines 1162-1381.*
  - Safety Defaults section is preserved (minus the "runaway costs" phrase). *Source: plan Task 5 Step 1 removes "or notice runaway costs".*
- **Interactions:** These are the patterns that currently work. Breaking any of them would be a net regression.

---

## Coverage summary

### Areas covered

- **All streaming patterns** (SSE, WebSocket, Background Job, Persistent Session): flag correctness, event handling, duplicate text prevention, dead code removal, error handling
- **Event documentation**: table accuracy vs. verified stream output, extended thinking behavior, event sequence diagram
- **cli-runtime-reference.md**: new flag coverage, cost reference removal, gotchas, quick reference table
- **Code pattern reliability**: `extract()` race condition/deadlock, `express.json()` middleware, frontend error handling, Server Setup block
- **First-run reliability checklist**: existence and coverage of silent failure modes
- **Regression protection**: all existing correct patterns verified as preserved
- **Cost/budget reference removal**: comprehensive grep for any remaining references

### Areas explicitly excluded (per agreed strategy)

- **Runtime execution testing** — cannot spawn `claude -p` from within Claude Code. The verified stream outputs (`/tmp/loom-*.jsonl`) substitute as ground truth. Risk: if the verified outputs are wrong, all event-related tests are wrong. Mitigation: the outputs were generated on this machine on 2026-03-08 and manually inspected during the conversation.
- **TypeScript compilation** — the code examples are teaching patterns embedded in markdown, not standalone files. Extracting and compiling them would require building a harness to pull fenced code blocks, resolve imports, and provide type stubs. The complexity exceeds the value for a documentation improvement task. Risk: a typo in a code example survives. Mitigation: semantic review catches structural bugs (wrong field names, missing arguments) even without compilation.
- **End-to-end app generation** — generating a full app from the skill and running it would be the highest-fidelity test but requires a human to verify the output since `claude -p` cannot be called from within Claude Code. Risk: the skill could teach correct patterns that still don't produce working apps due to a subtle interaction bug. Mitigation: the scenario tests trace the full data path manually.
- **OAuth flow testing** — oauth-reference.md is not modified by this plan. Not tested. Risk: none from this change.
- **Performance testing** — not applicable. The modified artifacts are documentation files. There is no performance surface.
