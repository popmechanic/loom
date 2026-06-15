# Claude CLI Runtime Reference

A complete technical reference for using `claude -p` (print mode) and the Agent SDK
to build applications where Claude is the runtime.

> Verified against Claude Code 2.x (`claude --help`). Flag names and behavior are
> stable across the 2.x line — run `claude --help` to confirm against your
> installed version.

## Table of Contents

1. [Input Methods](#input-methods)
2. [Output Formats](#output-formats)
3. [Structured Output (JSON Schema)](#structured-output)
4. [Tool Configuration](#tool-configuration)
5. [Permission Modes](#permission-modes)
6. [Session Management](#session-management)
7. [Custom Agents](#custom-agents)
8. [Streaming (Bidirectional)](#streaming)
9. [MCP Servers](#mcp-servers)
10. [Hooks](#hooks)
11. [Agent SDK (Programmatic)](#agent-sdk)
12. [Safety Controls](#safety-controls)
13. [Quick Reference Table](#quick-reference)

---

## Input Methods

Five ways to feed input to `claude -p`:

### Command line argument
```bash
claude -p "What is 2+2?"
```

### Stdin pipe (context + instruction)
```bash
cat data.csv | claude -p "Summarize this data"
git diff | claude -p "Review these changes"
```

### Stdin redirect
```bash
claude -p < prompt.txt
```

### Here-doc (multi-line prompts)
```bash
claude -p <<'EOF'
You are a code reviewer. Review this code:
def add(a, b): return a + b
Focus on: error handling, edge cases.
EOF
```

### Stream-JSON input (programmatic, real-time)
```bash
echo '{"type":"user","message":{"role":"user","content":"Hello"}}' | \
  claude -p --input-format stream-json --output-format stream-json --verbose
```

Stream input REQUIRES stream output. You can combine argument + stdin
(stdin is context, argument is instruction).

---

## Output Formats

Controlled by `--output-format`:

### text (default)
Human-readable plain text.
```bash
response=$(claude -p "What is 2+2?")
claude -p "Write a haiku" > haiku.txt
```

### json
Structured response with full metadata:
```bash
claude -p --output-format json "What is 2+2?"
```
Returns: `type`, `subtype`, `is_error`, `duration_ms`, `result`,
`session_id`, `total_cost_usd`, `usage`, `modelUsage`.

Extract fields with jq:
```bash
claude -p --output-format json "query" | jq -r '.result'
```

### stream-json (real-time, requires --verbose)
Emits newline-delimited JSON events:
```bash
claude -p --output-format stream-json --verbose "Write a poem"
```

Event sequence:
```
{"type":"system","subtype":"init","session_id":"...","model":"claude-sonnet-4-6","tools":[...]}
{"type":"assistant","message":{"content":[{"type":"text","text":"..."}]}}
{"type":"result","subtype":"success","is_error":false,"stop_reason":"end_turn","session_id":"...","num_turns":1}
```

Note: With `--include-partial-messages`, additional `stream_event` events appear
between `system` and `assistant` containing token-level deltas. See the
Stream-JSON Event Types table in `SKILL.md` for the complete event sequence.

---

## Structured Output

The killer feature for data extraction and typed outputs:

```bash
claude -p --output-format json \
  --json-schema '{"type":"object","properties":{"answer":{"type":"string"},"confidence":{"type":"number"}},"required":["answer"]}' \
  "What is the capital of France?"
```

Response contains BOTH `result` (text) and `structured_output` (typed):
```json
{
  "result": "The capital of France is Paris.",
  "structured_output": { "answer": "Paris", "confidence": 1.0 }
}
```

Use cases: data extraction pipelines, form automation, type-safe agentic outputs.

---

## Tool Configuration

### Disable all tools (pure LLM, no file access)
```bash
claude -p --tools "" "Just chat with me"
```

### Whitelist specific tools
```bash
claude -p --tools "Bash,Read,Glob,Grep" "List files"
```

### Fine-grained allow/deny with patterns
```bash
# Only git commands
claude -p --allowedTools "Bash(git:*)" "commit changes"

# Read-only analysis
claude -p --disallowedTools "Write,Edit,Bash" "analyze this code"

# Complex: git + npm + read src
claude -p --allowedTools "Bash(git *)" "Bash(npm run *)" "Read(/src/**)" "Edit(/src/**)"
```

### Tool permission rule syntax
- `Bash` — all bash commands
- `Bash(npm test)` — exact command
- `Bash(npm run *)` — prefix match with word boundary
- `Read(/src/**)` — gitignore-style path pattern
- `WebFetch(domain:example.com)` — domain-scoped fetch
- `Task(worker)` — specific subagent only

### Expand filesystem access
```bash
# Expand filesystem access beyond working directory
claude -p --add-dir /data/shared --add-dir /tmp/uploads "analyze files"
```

---

## Permission Modes

`--permission-mode <mode>`:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `default` | Prompt on first use | Interactive |
| `plan` | Read-only, then implement | Analysis first |
| `auto` | Classifier decides per action — auto-approves safe ones, prompts on the rest | Interactive/assisted (can still hang in a server when it decides to prompt) |
| `acceptEdits` | Auto-approve file modifications | Trusted editing |
| `dontAsk` | Auto-deny unless explicitly allowed | Safe mode (servers) |
| `bypassPermissions` | Skip all checks | CI/CD, automation |

For unattended server contexts, prefer `dontAsk` (paired with `--allowedTools`)
over `auto` — `auto` can still decide to prompt, and there's no human to answer.

Shorthand: `--dangerously-skip-permissions` = `--permission-mode bypassPermissions`

---

## Session Management

### Ephemeral (stateless)
```bash
claude -p --no-session-persistence "stateless task"
```

### Continue most recent session
```bash
claude -p --continue "follow up question"
```

### Resume specific session
```bash
claude -p --resume "session-uuid" "continue work"
```

### Fixed session ID (multi-turn conversations)
```bash
SESSION="$(uuidgen)"
# First turn: --session-id creates the session
echo "My name is Alice" | claude -p --session-id $SESSION --output-format json
# Subsequent turns: --resume continues that specific session
echo "What's my name?" | claude -p --resume $SESSION
```

**Important:** Do NOT combine `--session-id` with `--continue` or `--resume` —
this errors unless `--fork-session` is also specified. Use `--session-id` on
the first turn to create the session, then `--resume <id>` on subsequent turns.

### Fork a session
```bash
claude --resume session-id --fork-session -p "try alternative approach"
```

Get session ID from JSON output: `jq -r '.session_id'`

---

## Custom Agents

### Define inline
```bash
claude -p --agents '{
  "reviewer": {
    "description": "Code reviewer",
    "prompt": "You are a strict code reviewer.",
    "tools": ["Read", "Grep", "Glob"],
    "model": "sonnet",
    "maxTurns": 10
  }
}' --agent reviewer "review this codebase"
```

### Agent definition fields
| Field | Required | Description |
|-------|----------|-------------|
| `description` | Yes | When to use this agent |
| `prompt` | Yes | System prompt |
| `tools` | No | Allowed tools |
| `disallowedTools` | No | Denied tools |
| `model` | No | `sonnet`, `opus`, `haiku`, `inherit` |
| `maxTurns` | No | Turn limit |
| `permissionMode` | No | Permission mode |
| `skills` | No | Skills to preload |
| `mcpServers` | No | MCP servers to connect |

---

## Streaming

### Output streaming (one-way)
```bash
claude -p --output-format stream-json --verbose "Write a story" | \
  while IFS= read -r line; do
    text=$(echo "$line" | jq -r '.event.delta.text // empty' 2>/dev/null)
    [ -n "$text" ] && printf "%s" "$text"
  done
```

### Token-level streaming (--include-partial-messages)

By default, `--output-format stream-json` delivers text only as complete
`assistant` blocks. To get token-by-token `stream_event` deltas, add
`--include-partial-messages`:

```bash
claude -p --output-format stream-json --verbose --include-partial-messages "Write a story"
```

Without this flag, streaming apps show text as a single dump instead of
typing progressively. **Required for SSE/WebSocket streaming patterns.**

### Node.js stream parsing
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

### Bidirectional streaming
```bash
echo '{"type":"user","message":{"role":"user","content":"Hello"}}' | \
  claude -p --input-format stream-json --output-format stream-json --verbose --replay-user-messages
```

`--replay-user-messages` echoes user messages back with `"isReplay": true`.

---

## MCP Servers

Load via config file:
```bash
claude --mcp-config ./mcp.json -p "query"
claude --strict-mcp-config --mcp-config ./mcp.json -p "query"
```

Config format:
```json
{
  "mcpServers": {
    "github": { "type": "http", "url": "https://api.example.com/mcp/" },
    "local-tool": {
      "type": "stdio",
      "command": "npx",
      "args": ["@some/mcp-server"],
      "env": { "API_KEY": "${MY_API_KEY}" }
    }
  }
}
```

Server types: `http`, `sse` (deprecated), `stdio`.

---

## Hooks

Hooks are shell commands that execute in response to Claude events.
Configure in `.claude/settings.json`.

### Available events
`SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`,
`PostToolUseFailure`, `PermissionRequest`, `Notification`,
`SubagentStart`, `SubagentStop`, `Stop`, `SessionEnd`

### Hook mechanics
- Input: JSON on stdin with tool details, session ID, working directory
- Exit 0: allow / continue
- Exit 2: block the action
- Stderr: feedback message sent back to Claude

### Example: audit logging
```json
{
  "hooks": {
    "PreToolUse": [{
      "type": "command",
      "command": "jq -c '{ts: now|todate, tool: .tool_name}' >> ~/claude-audit.log"
    }]
  }
}
```

### Example: block dangerous commands
```bash
#!/bin/bash
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')
if echo "$CMD" | grep -q "rm -rf"; then
  echo "Blocked: rm -rf" >&2
  exit 2
fi
exit 0
```

---

## Agent SDK

### Python
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Find and fix bugs in auth.py",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash"],
            permission_mode="acceptEdits",
            max_turns=5,
            model="sonnet"
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)

asyncio.run(main())
```

### TypeScript
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Find and fix bugs in auth.py",
  options: {
    allowedTools: ["Read", "Edit", "Bash"],
    permissionMode: "acceptEdits",
    maxTurns: 5,
    model: "sonnet"
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

### Session resumption in SDK
```python
session_id = None
async for msg in query(prompt="Read the auth module", options=...):
    if hasattr(msg, "session_id"): session_id = msg.session_id

async for msg in query(prompt="Now find callers", options=ClaudeAgentOptions(resume=session_id)):
    if hasattr(msg, "result"): print(msg.result)
```

---

## Safety Controls

```bash
# Turn limit — bound the agentic loop
claude -p --max-turns 10 "multi-step task"

# Cost ceiling — hard dollar cap per invocation (print mode only). The process
# stops once spend would exceed this amount. Pair with --max-turns for unattended runs.
claude -p --max-budget-usd 0.50 "bounded task"

# Fallback model(s) for reliability — comma-separated, tried in order on overload
claude -p --model opus --fallback-model sonnet,haiku "important task"

# Model selection (aliases resolve to the current generation)
claude -p --model haiku "quick extraction"      # Fastest, cheapest (Haiku 4.5)
claude -p --model sonnet "standard task"         # Balanced default (Sonnet 4.6)
claude -p --model opus "complex reasoning"       # Best quality (Opus 4.8)
# claude -p --model fable "hardest agentic work" # Most capable (Fable 5) — uncomment when available on your plan

# Reasoning effort: low | medium | high | xhigh | max
claude -p --effort xhigh "agentic coding task"   # Claude Code's own default for coding/agentic work
claude -p --effort low "simple extraction"       # Fast, minimal reasoning
```

`--effort` trades latency and token spend for depth. `xhigh` (between `high` and
`max`) is the setting Claude Code uses for its own coding and agentic work;
reserve `max` for correctness-critical tasks where cost doesn't matter, and drop
to `low`/`medium` for routing, classification, and extraction.

The `fable` alias resolves to Anthropic's most capable model for the heaviest
long-horizon agentic work. It may not be available on every plan or region — the
CLI recognizes the alias, so swap it in wherever you'd otherwise reach for `opus`
on the hardest tasks once it's enabled for your account.

## Isolation & Embedding Controls

When `claude -p` is the runtime inside your own server, you usually want it
isolated from the host machine's Claude Code configuration — predictable,
reproducible, and uncontaminated by whatever CLAUDE.md, plugins, or MCP servers
happen to live on the box.

```bash
# Control which settings layers load (user / project / local)
claude -p --setting-sources user,project "task"

# Only use MCP servers from --mcp-config; ignore all other MCP configuration
claude -p --strict-mcp-config --mcp-config ./mcp.json "task"

# Improve cross-user/cross-machine prompt-cache reuse: move per-machine sections
# (cwd, env info, git status) out of the system prompt into the first user message
claude -p --exclude-dynamic-system-prompt-sections "task"

# Persona/instructions from a file instead of an inline string
claude -p --system-prompt-file persona.txt "task"
claude -p --append-system-prompt-file house-rules.txt "task"
```

### `--bare` (minimal mode)

`--bare` strips Claude Code down to the model loop: it skips hooks, LSP, plugin
sync, auto-memory, background prefetches, keychain reads, and CLAUDE.md
auto-discovery (and sets `CLAUDE_CODE_SIMPLE=1`). This gives the fastest, most
reproducible startup for an embedded runtime.

**The catch for Loom apps:** under `--bare`, Anthropic auth is *strictly*
`ANTHROPIC_API_KEY` or `apiKeyHelper` (via `--settings`) — OAuth and the keychain
are **never** read. That makes `--bare` incompatible with the
`CLAUDE_CODE_OAUTH_TOKEN` multi-user pattern this skill is built around. Use it
only for single-tenant apps that authenticate with an API key. With `--bare` you
must supply context explicitly: `--system-prompt[-file]`,
`--append-system-prompt[-file]`, `--add-dir`, `--mcp-config`, `--settings`,
`--agents`.

---

## Quick Reference

| Goal | Command |
|------|---------|
| Simple chat | `echo "hi" \| claude -p` |
| JSON response | `claude -p --output-format json "query"` |
| Streaming | `claude -p --output-format stream-json --verbose "query"` |
| Structured data | `claude -p --output-format json --json-schema '{...}' "query"` |
| No tools (pure LLM) | `claude -p --tools "" "query"` |
| Full autonomy | `claude -p --permission-mode bypassPermissions "query"` |
| Ephemeral | `claude -p --no-session-persistence "query"` |
| Custom persona | `claude -p --system-prompt "You are..." "query"` |
| Append persona | `claude -p --append-system-prompt "Also do X" "query"` |
| Fast model | `claude -p --model haiku "query"` |
| Best quality | `claude -p --model opus "query"` |
| With fallback | `claude -p --model opus --fallback-model sonnet,haiku "query"` |
| Turn limit | `claude -p --max-turns 10 "query"` |
| Cost ceiling | `claude -p --max-budget-usd 0.50 "query"` |
| Continue session | `claude -p --continue "follow up"` |
| Resume session | `claude -p --resume UUID "continue"` |
| Token streaming | `claude -p --output-format stream-json --verbose --include-partial-messages "query"` |
| Effort control | `claude -p --effort xhigh "query"` (low\|medium\|high\|xhigh\|max) |
| Extra directories | `claude -p --add-dir /path "query"` |
| Minimal/isolated mode | `claude -p --bare "query"` (API-key auth only) |
| Limit settings layers | `claude -p --setting-sources user,project "query"` |
| Cache-friendly prefix | `claude -p --exclude-dynamic-system-prompt-sections "query"` |
| Persona from file | `claude -p --system-prompt-file persona.txt "query"` |

## Gotchas

1. `--output-format stream-json` **requires** `--verbose`
2. `--input-format stream-json` **requires** `--output-format stream-json`
3. MCP tools load by default — use `--strict-mcp-config` to control
4. `structured_output` is separate from `result` in JSON output
5. Context caching reduces cost on repeated runs (don't invalidate KV cache with dynamic timestamps at prompt start). For multi-user apps, add `--exclude-dynamic-system-prompt-sections` to move per-machine sections (cwd, env info, git status) out of the cached prefix so the cache is reused across users and machines.
6. **Nesting guard**: When spawning `claude -p` from within Claude Code, remove `CLAUDECODE` and `CLAUDE_CODE_ENTRYPOINT` from the child's environment — these block nested Claude processes. Do NOT filter all `CLAUDE*` vars (that kills auth tokens).
7. **`dontAsk` without `--allowedTools`** = silent failure. `dontAsk` auto-denies everything not explicitly allowed. Without `--allowedTools`, Claude has NO tools — it can reason but can't act. The result is an empty or incomplete response with NO error. Always pair `--permission-mode dontAsk` with `--allowedTools "Read,Bash,..."` or `--tools "Read,Bash,..."`.
8. **Stdout is chunked** — TCP delivers data in arbitrary chunks. Buffer lines before parsing JSON (split on `\n`, keep the last incomplete fragment). Use `TextDecoder({ stream: true })` not `chunk.toString()` for UTF-8 safety.
9. **`--session-id` + `--continue` errors** — Combining `--session-id` with `--continue` or `--resume` requires `--fork-session`. For multi-turn conversations, use `--session-id` on the first turn only, then `--resume <id>` on subsequent turns.
10. **Claude reads images and PDFs natively** — The Read tool handles PNG, JPG, JPEG, GIF, WebP, BMP, and PDF files. For apps that accept file uploads or drops, save binary files to a temp directory and pass the file path in the prompt — Claude will use its Read tool to analyze them visually. Clean up temp files after the Claude process exits.
11. **`--include-partial-messages` required for token streaming** — Without this flag, `stream-json` output delivers text only as complete `assistant` blocks (no `stream_event` tokens). Add `--include-partial-messages` to get token-by-token deltas for SSE/WebSocket streaming.
12. **`--bare` forces API-key auth** — Minimal mode never reads `CLAUDE_CODE_OAUTH_TOKEN` or the keychain (auth is strictly `ANTHROPIC_API_KEY` / `apiKeyHelper`). Don't combine it with the OAuth multi-user pattern; it's for single-tenant API-key apps. See Isolation & Embedding Controls.
13. **`--max-budget-usd` and `--no-session-persistence` are print-mode-only** — They're silently ignored outside `-p`. `--max-budget-usd` is a hard cost ceiling per invocation; pair it with `--max-turns` to bound unattended runs on both cost and loop count.
