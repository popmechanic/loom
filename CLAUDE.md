# Loom — Developer Guide

This repo is a Claude Code plugin that ships three skills: `loom` (deployed/multi-user web apps), `loom-local` (single-user local web tools), and `loom-desktop` (native desktop apps via ElectroBun). All three teach Claude how to build applications where `claude -p` is the runtime.

## Design Goals — What Loom Delivers to Developers

Loom's value to a developer decomposes into two *independent* advantages. An app may need one, both, or neither — the skills should help developers tell which, then deliver it well. (These were validated by analyzing real Loom-pattern apps: skylights-agents, metrc, Julian, VibesOS — each takes a different subset.)

1. **The agent loop, for free.** `claude -p` (or the Agent SDK) ships the multi-turn tool-use loop, built-in tools (Read/Write/Edit/Bash/Grep/Glob/Web*), MCP integration, resumable sessions, structured output, and a permission gate. For an agentic task, the raw-inference-API equivalent means re-implementing most of Claude Code. This advantage scales with how much real tool-using work the app does — it's the whole point for apps like Julian (filesystem memory + Bash + Agent Teams) and the VibesOS generator (multi-turn file-editing loop).

2. **Subscription-OAuth economics.** The CLI authenticates with a user's Claude subscription, so inference is free-at-point-of-use instead of metered per token. This advantage is orthogonal to agency — metric/Themis runs Claude with `--tools ""` (zero agent loop) purely to bill synthesis against a Max subscription. It's the load-bearing reason Loom prefers the CLI over the SDK (see the "Choosing the runtime: CLI vs Agent SDK" section in `skills/loom/SKILL.md`).

Two boundaries the skills must respect and teach:

- **The choice is per task, not per app.** Even deeply agentic apps drop to `--tools "" --json-schema` one-shots for extraction, classification, and routing — those calls don't use the loop and are cheaper/faster as near-raw inference (Julian's `extract.ts`, VibesOS's `riffGenerate`). Help developers wield the loop where the work is agentic and skip it where it isn't, inside the same app.

- **Subscription billing does not survive multi-tenancy.** It shines when each user brings their own subscription (one instance per person, or a user-funded desktop tool). The moment the operator absorbs inference cost for many users, the model breaks and a metered API key / proxy is correct — VibesOS's in-app AI deliberately proxies OpenRouter with a shared key for exactly this reason. The `loom` skill targets multi-user apps, so it must be honest about this ceiling: steer developers to per-user OAuth where users self-fund, and to a metered-key path (or the SDK with an API key, as skylights-agents chose) where the operator pays.

**How this directs skill work:** make the agent-loop path easy to wield well (tools, MCP, sessions, structured output, safety defaults); make per-user OAuth correct-by-default; and give developers a clear decision point for when *not* to reach for the harness — single-shot inference and operator-funded multi-tenant SaaS — so they never pay harness overhead for value they aren't getting.

## What Gets Distributed

Only `skills/` and plugin metadata ship to users. Everything else is local dev tooling.

**Distributed (tracked in git):**
- `skills/loom/` — web app skill (SKILL.md + references/)
- `skills/loom-local/` — local web app skill (SKILL.md + references/)
- `skills/loom-desktop/` — desktop app skill (SKILL.md + references/)
- `.claude-plugin/plugin.json` — plugin metadata (name, version, description)
- `.claude-plugin/marketplace.json` — single-plugin marketplace manifest (lets `/plugin marketplace add popmechanic/loom` find the plugin via `source: "./"`)
- `assets/` — logo images
- `README.md`, `LICENSE`

**Local only (gitignored):**
- `neko-chat/`, `neko-chat-desktop/`, `codelens/`, `symmetry-terminal/`, `commit-storyteller/` — example apps used for manual testing
- `loom-workspace/` — skill-creator evaluation workspace (evals, benchmarks, iteration results)
- `docs/plans/` — implementation and design plans from trycycle sessions
- `tests/` — test artifacts

## Skill Architecture

Each skill follows progressive disclosure:

```
skills/loom/
├── SKILL.md              # <500 lines — mental model, conversation guide, cross-references
└── references/
    ├── server-patterns.md    # Full server-side code patterns
    ├── advanced-patterns.md  # Structured extraction, persistent sessions, HTTP hooks
    ├── session-ux.md         # Rendering tool calls, diffs, plans, reasoning in the UI
    ├── cli-runtime-reference.md  # claude -p flag reference
    └── oauth-reference.md        # Auth implementation
```

Each skill has its own reference set (`loom-local` and `loom-desktop` differ) —
check the skill's directory rather than assuming this layout.

- `SKILL.md` is always loaded when the skill triggers — keep it concise
- `references/` files are read on demand when Claude needs implementation detail
- Reference files over 300 lines include a table of contents

## Conventions

**Writing style:** Use imperative form and explain consequences. Never use MUST/MUST NOT in all caps — if something is important, explain why.

**Reference paths:** All skills use the plain relative form `references/foo.md`
(the form shown in the official Claude Code skills docs). Do not use the
`@references/` prefix — it is not part of the documented skill-reference syntax
and was removed for consistency across the three skills.

**Cross-references:** Every reference file citation in SKILL.md includes guidance on when to read it:
```markdown
> Read `references/server-patterns.md` when implementing a specific server
> endpoint or wiring up the frontend.
```

**Descriptions:** Each skill uses two frontmatter fields. `description` is a
one-line summary of what the skill builds; `when_to_use` starts with "Use
when..." and lists triggering conditions and exclusions. Claude Code
concatenates the two into the effective trigger text (combined cap ~1,536
chars), so keep them complementary, not redundant. Do not collapse the triggers
into `description` — the split is intentional and keeps each field readable.

## Testing Changes

No automated test suite exists. To test skill changes:

1. Edit the skill in `skills/`
2. Load the plugin into a fresh session exactly as users receive it:
   `claude --plugin-dir /path/to/loom` (session-only, no install)
3. Use one of the example apps as a test vehicle (e.g., build a fresh neko-chat variant)
4. Or use the skill-creator evaluation framework in `loom-workspace/`

Useful checks:

- `claude --plugin-dir /path/to/loom plugin details loom` — verifies the plugin
  resolves and shows the component inventory plus projected token cost per
  skill (the always-on number is what every user session pays; keep it small).
- `claude plugin eval` — the CLI's built-in eval runner (`evals/**/case.yaml`
  or `prompt.md` + graders). The repo isn't laid out for it yet; consider it
  if the `loom-workspace` framework is retired.

The `loom-workspace/evals/evals.json` file contains test prompts from previous evaluation rounds.

## Verifying Runtime Claims

The skills state empirical facts about `claude -p` — flags, event shapes,
permission behavior. This repo's core failure mode is those claims drifting
from CLI reality, so before adding or changing a factual claim, verify it
against the installed CLI rather than memory or docs:

- `claude --help` is incomplete. Hidden-but-real flags exist (`--max-turns`,
  `--system-prompt-file`). Parse-test instead — unknown flags error, real ones
  print the version: `claude --max-turns 2 --version`
- Capture real stream shapes with one cheap haiku run:

  ```bash
  env -u CLAUDECODE -u CLAUDE_CODE_ENTRYPOINT claude -p --model haiku \
    --output-format stream-json --verbose --include-partial-messages \
    --permission-mode dontAsk --allowedTools Read --no-session-persistence \
    "Read package.json and summarize it" | jq -c '{type, subtype}' | sort | uniq -c
  ```

  (The `env -u` strips the nesting guards so this works from inside a Claude
  Code session.)
- Claims about hooks or permission gates need a **live listener** asserting the
  request actually arrives. An unreachable HTTP hook endpoint fails open, so a
  dead-port test "passes" identically whether the feature exists or not — this
  false negative has bitten before.
- Check `result.permission_denials` in test output — it's the precise signal
  that a run was tool-limited rather than model-limited.

## Plugin Versioning

Bump the version in `.claude-plugin/plugin.json` when publishing changes —
that file is the single source of truth for the current version (don't restate
it here; a copy goes stale).

## Important Distinction

Plugin users only see the skills — they never see this file, the example apps, the docs, or the workspace. All guidance that affects skill behavior must live in `SKILL.md` or `references/`, never here. This file is exclusively for developers working on the plugin itself.
