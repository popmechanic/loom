# Claude CLI Runtime Reference (Local)

A focused reference for `claude -p` (print mode) flags used in local Bun
apps. For the complete reference including MCP servers, hooks, the Agent SDK,
and custom agent definitions, see the
[Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code).

---

## Input Methods

Five ways to feed input to `claude -p`:

```bash
# Command line argument
claude -p "What is 2+2?"

# Stdin pipe (context + instruction)
cat data.csv | claude -p "Summarize this data"
git diff | claude -p "Review these changes"

# Stdin redirect
claude -p < prompt.txt

# Here-doc (multi-line prompts)
claude -p <<'EOF'
You are a code reviewer. Review this code:
def add(a, b): return a + b
Focus on: error handling, edge cases.
EOF

# Stream-JSON input (programmatic, real-time)
echo '{"type":"user","message":{"role":"user","content":"Hello"}}' | \
  claude -p --input-format stream-json --output-format stream-json --verbose
```

Stream input REQUIRES stream output. Argument + stdin can be combined
(stdin is context, argument is instruction).

---

## Output Formats

Controlled by `--output-format`:

### text (default)
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

Add `--include-partial-messages` for token-level streaming:
```bash
claude -p --output-format stream-json --verbose --include-partial-messages "Write a poem"
```

Without `--include-partial-messages`, text arrives only as complete `assistant`
blocks — no token-by-token streaming.

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

---

## Tool Configuration

```bash
# Disable all tools (pure LLM, no file access)
claude -p --tools "" "Just chat with me"

# Whitelist specific tools
claude -p --tools "Bash,Read,Glob,Grep" "List files"

# Fine-grained allow/deny with patterns
claude -p --allowedTools "Bash(git:*)" "commit changes"
claude -p --disallowedTools "Write,Edit,Bash" "analyze this code"
claude -p --allowedTools "Bash(git *)" "Bash(npm run *)" "Read(/src/**)" "Edit(/src/**)"

# Expand filesystem access beyond working directory
claude -p --add-dir /data/shared --add-dir /tmp/uploads "analyze files"
```

### Tool permission rule syntax
- `Bash` — all bash commands
- `Bash(npm test)` — exact command
- `Bash(npm run *)` — prefix match with word boundary
- `Read(/src/**)` — gitignore-style path pattern
- `WebFetch(domain:example.com)` — domain-scoped fetch

---

## Permission Modes

`--permission-mode <mode>`:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `default` | Prompt on first use | Interactive |
| `plan` | Read-only, then implement | Analysis first |
| `acceptEdits` | Auto-approve file modifications | Trusted editing |
| `dontAsk` | Auto-deny unless explicitly allowed | Server / automation |
| `bypassPermissions` | Skip all checks | CI/CD, fully constrained tools |

Shorthand: `--dangerously-skip-permissions` = `--permission-mode bypassPermissions`

---

## Session Management

```bash
# Ephemeral (stateless)
claude -p --no-session-persistence "stateless task"

# Continue most recent session
claude -p --continue "follow up question"

# Resume specific session
claude -p --resume "session-uuid" "continue work"

# Fixed session ID (multi-turn conversations)
SESSION="$(uuidgen)"
# First turn: --session-id creates the session
echo "My name is Alice" | claude -p --session-id $SESSION --output-format json
# Subsequent turns: --resume continues that specific session
echo "What's my name?" | claude -p --resume $SESSION

# Fork a session
claude --resume session-id --fork-session -p "try alternative approach"
```

Do NOT combine `--session-id` with `--continue` or `--resume` — this errors
unless `--fork-session` is also specified.

---

## Essential Flags

| Flag | Purpose |
|------|---------|
| `--model haiku\|sonnet\|opus` | Model selection |
| `--fallback-model haiku` | Auto-fallback on overload |
| `--effort high\|low` | Control reasoning depth |
| `--max-turns N` | Limit agentic iterations |
| `--max-budget-usd N` | Stop after spending limit |
| `--system-prompt "..."` | Replace entire system prompt |
| `--append-system-prompt "..."` | Add to default system prompt |
| `--permission-mode dontAsk` | Auto-deny unapproved tools |
| `--no-session-persistence` | Don't save session to disk |
| `--add-dir /path` | Expand filesystem access |

---

## Quick Reference

| Goal | Command |
|------|---------|
| Simple query | `echo "hi" \| claude -p` |
| JSON response | `claude -p --output-format json "query"` |
| Streaming | `claude -p --output-format stream-json --verbose "query"` |
| Token streaming | `claude -p --output-format stream-json --verbose --include-partial-messages "query"` |
| Structured data | `claude -p --output-format json --json-schema '{...}' "query"` |
| No tools | `claude -p --tools "" "query"` |
| Custom persona | `claude -p --system-prompt "You are..." "query"` |
| Append persona | `claude -p --append-system-prompt "Also do X" "query"` |
| Fast model | `claude -p --model haiku "query"` |
| Best quality | `claude -p --model opus "query"` |
| With fallback | `claude -p --model sonnet --fallback-model haiku "query"` |
| Turn limit | `claude -p --max-turns 10 "query"` |
| Cost limit | `claude -p --max-budget-usd 5 "query"` |
| Effort control | `claude -p --effort high "query"` |
| Extra directories | `claude -p --add-dir /path "query"` |
| Ephemeral | `claude -p --no-session-persistence "query"` |
| Continue session | `claude -p --continue "follow up"` |
| Resume session | `claude -p --resume UUID "continue"` |

---

## Gotchas

1. `--output-format stream-json` **requires** `--verbose`
2. `--input-format stream-json` **requires** `--output-format stream-json`
3. `structured_output` is separate from `result` in JSON output
4. **Nesting guard**: Remove `CLAUDECODE` and `CLAUDE_CODE_ENTRYPOINT` from the
   child's environment (that's what `cleanEnv()` does). Do NOT filter all
   `CLAUDE*` vars.
5. **`dontAsk` without `--allowedTools`** = silent failure. Claude can reason but
   can't act. The result is empty with NO error.
6. **Stdout is chunked** — Buffer lines before parsing JSON (split on `\n`, keep
   the last incomplete fragment). Use `TextDecoder({ stream: true })` for UTF-8
   safety.
7. **`--session-id` + `--continue` errors** — Use `--session-id` on the first
   turn only, then `--resume <id>` on subsequent turns.
8. **Claude reads images and PDFs natively** — Save uploaded files to a temp
   directory and pass the file path in the prompt. Clean up temp files after the
   Claude process exits.
9. **`--include-partial-messages` required for token streaming** — Without this
   flag, `stream-json` delivers text only as complete `assistant` blocks.
10. Context caching reduces cost on repeated runs — don't invalidate KV cache
    with dynamic timestamps at prompt start.
