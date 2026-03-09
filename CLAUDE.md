# Loom — Developer Guide

This repo is a Claude Code plugin that ships two skills: `loom` (web apps) and `loom-desktop` (native desktop apps via ElectroBun). Both teach Claude how to build applications where `claude -p` is the runtime.

## What Gets Distributed

Only `skills/` and plugin metadata ship to users. Everything else is local dev tooling.

**Distributed (tracked in git):**
- `skills/loom/` — web app skill (SKILL.md + references/)
- `skills/loom-desktop/` — desktop app skill (SKILL.md + references/)
- `.claude-plugin/plugin.json` — plugin metadata (name, version, description)
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
    ├── cli-runtime-reference.md  # claude -p flag reference
    └── oauth-reference.md        # Auth implementation
```

- `SKILL.md` is always loaded when the skill triggers — keep it concise
- `references/` files are read on demand when Claude needs implementation detail
- Reference files over 300 lines include a table of contents

## Conventions

**Writing style:** Use imperative form and explain consequences. Never use MUST/MUST NOT in all caps — if something is important, explain why.

**Reference paths:** `loom` uses `references/foo.md`, `loom-desktop` uses `@references/foo.md` (the `@` prefix is a Claude Code convention for skill-relative paths). Be consistent within each skill.

**Cross-references:** Every reference file citation in SKILL.md includes guidance on when to read it:
```markdown
> Read `references/server-patterns.md` when implementing a specific server
> endpoint or wiring up the frontend.
```

**Descriptions:** Start with "Use when..." and list triggering conditions. Never summarize what the skill does in the description — that causes Claude to follow the description instead of reading the full skill.

## Testing Changes

No automated test suite exists. To test skill changes:

1. Edit the skill in `skills/`
2. Use one of the example apps as a test vehicle (e.g., build a fresh neko-chat variant)
3. Or use the skill-creator evaluation framework in `loom-workspace/`

The `loom-workspace/evals/evals.json` file contains test prompts from previous evaluation rounds.

## Plugin Versioning

Bump the version in `.claude-plugin/plugin.json` when publishing changes. Current: `0.0.2`.

## Important Distinction

Plugin users only see the skills — they never see this file, the example apps, the docs, or the workspace. All guidance that affects skill behavior must live in `SKILL.md` or `references/`, never here. This file is exclusively for developers working on the plugin itself.
