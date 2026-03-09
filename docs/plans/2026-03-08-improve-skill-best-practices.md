# Improve Loom Skills to Follow Best Practices — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use trycycle-executing to implement this plan task-by-task.

**Goal:** Refactor `skills/loom/SKILL.md` (1632 lines) and `skills/loom-desktop/SKILL.md` (2026 lines) to be under 500 lines each by extracting content to reference files, improving metadata descriptions for triggering, and following skill writing best practices (imperative form, "why" over rigid MUSTs, progressive disclosure).

**Architecture:** Each SKILL.md becomes a concise orchestration document — mental model, conversation guide, key patterns (summarized with cross-references), what to generate, and gotchas. Detailed code patterns, event type docs, and feature-specific content move to well-organized reference files with tables of contents. No content is deleted — it's relocated to references.

**Tech Stack:** Markdown files only. No code changes.

---

## Analysis: What stays, what moves

### loom/SKILL.md (1632 → ~420 lines target)

**Keep in SKILL.md (core guidance):**
- Metadata (9 lines) — rewrite description for trigger optimization
- Intro + Why This Matters + Architecture (lines 1-81, ~81 lines)
- The Conversation (lines 82-172, ~90 lines)
- Building It intro + Auth Setup summary (lines 173-203, ~10 lines condensed, point to oauth-reference)
- Safety Defaults summary (lines 212-240, ~10 lines condensed)
- Shared Utilities summary (lines 242-328, ~5 lines condensed — code stays in reference)
- Pattern summaries (REST, SSE, WS, Background, Parallel) — ~30 lines total summarizing when to use each, then cross-ref
- Stream-JSON Event Types — just the event table (~15 lines)
- Error Handling mental model (lines 960-982, ~20 lines — keep this, it's guidance not code)
- What to Generate (lines 1556-1606, ~50 lines) — keep, but update internal "above/below" refs to point to reference files
- First-Run Reliability Checklist (lines 1584-1601, ~18 lines) — keep, but update "inline patterns above" to reference file cross-ref
- Deployment Verification (lines 1608-1621, ~14 lines) — keep
- The Possibility Space (lines 1623-1632, ~10 lines) — keep

All frontend code (Streaming Text Display, Structured Result Rendering) and all server code samples move to references. SKILL.md provides prose summaries with cross-references only.

**Extract to references (two files, split by concern):**
- `references/server-patterns.md` (~650 lines): Shared Utilities (cleanEnv, createStreamParser, spawnEnvForUser), Server Setup, core communication patterns (REST, SSE, WebSocket, Background Job, Parallel), Stream-JSON Event Types (notes, extended thinking, code samples), Error Surfacing Checklist, Input Validation, Temp Directory Cleanup, Handling File Uploads and Drops, Frontend Integration (Streaming Text Display, Structured Result Rendering)
- `references/advanced-patterns.md` (~350 lines): Structured Extraction (Async Haiku), Persistent Session (Long-Lived Process), Action Markers, HTTP Hooks (config, receiving events, interactive permission approval with full server + frontend code)

**Existing references stay as-is:**
- `references/cli-runtime-reference.md` (479 lines) — already well-extracted, has TOC
- `references/oauth-reference.md` (693 lines) — already well-extracted, has TOC

### loom-desktop/SKILL.md (2026 → ~460 lines target)

**Keep in SKILL.md (core guidance):**
- Metadata (13 lines) — rewrite description for trigger optimization
- Intro + Why Desktop + Prerequisites (lines 1-55, ~55 lines)
- Architecture: Thin Bridge (lines 56-115, ~60 lines)
- Project Setup (lines 116-134, ~18 lines — already references electrobun-setup.md)
- The Conversation (lines 135-211, ~76 lines)
- Building It intro (lines 212-216, ~5 lines)
- Shared Utilities summary (lines 217-328, ~15 lines condensed, code → reference)
- Event Type Mapping table (lines 330-343, ~14 lines — keep, it's a quick lookup)
- deriveAndSendRPC summary (lines 345-423, ~5 lines condensed, code → reference)
- Pattern summaries (Sync, Stream, Conversational, Background) — ~40 lines total
- Desktop Features summaries (lines 1265-1490, ~30 lines condensed)
- Distribution summary (lines 1491-1661, ~20 lines condensed)
- Gotchas headlines + 1-line descriptions (lines 1662-1979, ~80 lines condensed from ~318)
- What to Generate (lines 1980-2010, ~30 lines)
- The Possibility Space (lines 2011-2026, ~15 lines)

**Extract to references:**
- `references/desktop-patterns.md` (~700 lines): All Bun-side and webview-side code patterns (Sync, Streaming including full spawnClaude, Conversational, Background), deriveAndSendRPC function, all webview React components
- `references/desktop-features.md` (~250 lines): File Drag-and-Drop, Native File Dialogs, System Tray, Native Menus, Local File Access Configuration
- `references/distribution.md` (~250 lines): Build, Sharing, Claude CLI dep, App Icon, DMG Icon, Code Signing & Notarization, Auto-Updates
- `references/gotchas.md` (~350 lines): All 16 gotchas with full detail and code samples

**Existing references stay as-is:**
- `references/cli-runtime-reference.md` (460 lines) — already well-extracted, has TOC
- `references/electrobun-setup.md` (280 lines) — already well-extracted
- `references/rpc-schema-reference.md` (396 lines) — already well-extracted, has TOC

---

## Writing style audit

The user requested "imperative form, examples, explaining 'why' over rigid MUSTs" as a key goal. The existing prose is largely well-written in this style already — it explains rationale, uses conversational tone, and includes examples. However, a few instances need fixing:

**Known violations to fix during rewrite:**

In `loom/SKILL.md`:
- Line 223: "you MUST pair `--permission-mode dontAsk` with `--allowedTools`" → rewrite to explain why (it stays in server-patterns.md — fix there)
- Lines that say "Every plan MUST start with" or similar rigid phrasing → scan and soften

**Rule for the executor:** When writing or copying prose into SKILL.md or reference files, apply these transformations:
1. Replace "MUST"/"MUST NOT" with explanations of what happens if you don't (e.g., "Without `--allowedTools`, `dontAsk` gives Claude no tools at all — it can reason but can't act, and the failure is silent")
2. Use imperative form for instructions ("Use `cleanEnv()` to remove nesting guards" not "You should use `cleanEnv()`")
3. Keep "Don't do this" blocks — they explain "why" by showing consequences, which is the right pattern

The existing "Don't do this" blocks in both skills are already excellent examples of explaining "why" (they show what breaks and how). These should be preserved as-is in reference files.

---

## Verification Criteria

1. **Line counts**: Both SKILL.md under 500 lines
2. **Metadata**: Each description under ~120 words with key trigger phrases optimized per best practices
3. **Reference integrity**: Every `references/*.md` path mentioned in SKILL.md exists; no orphan reference files
4. **Section heading diff**: No major sections silently dropped — all headings from original appear as either kept sections or cross-references
5. **Content balance**: Total line count across all files per skill stays roughly the same (content moved, not deleted)
6. **TOC**: Reference files over 300 lines include a table of contents
7. **Cross-references**: Each reference file is cross-referenced from SKILL.md with "read this when..." guidance
8. **No stale internal refs**: No "above"/"below" references in SKILL.md that point to content now in reference files
9. **Prose style**: No "MUST"/"MUST NOT" — use imperative form and explain consequences instead

---

### Task 1: Create `skills/loom/references/server-patterns.md`

**Files:**
- Create: `skills/loom/references/server-patterns.md`

**Step 1: Create the file**

Extract the following sections from `skills/loom/SKILL.md` into a new reference file with a table of contents:

1. **Shared Utilities** (lines 242-328) — `cleanEnv()`, `createStreamParser()`, `spawnEnvForUser()` — full code with explanatory prose
2. **Server Setup** (lines 329-356) — Express baseline — full code
3. **Pattern: REST Endpoint** (lines 358-416) — full code + "Don't do this"
4. **Pattern: SSE Streaming** (lines 417-526) — full code + "Don't do this"
5. **Pattern: WebSocket Session** (lines 527-654) — full code + "Don't do this"
6. **Pattern: Background Job with Progress** (lines 655-721) — full code + "Don't do this"
7. **Pattern: Parallel Analysis** (lines 722-775) — full code
8. **Stream-JSON Event Types** (lines 777-878) — event table, notes, extended thinking discussion, code samples for extracting text/tool use and detecting max-turns, tool tracking
9. **Error Surfacing Checklist** (lines 983-1030) — checklist with code
10. **Input Validation** (lines 1031-1055) — safePath helper + prompt defense
11. **Temp Directory Cleanup** (lines 1056-1078) — withTempDir helper
12. **Handling File Uploads and Drops** (lines 1079-1163) — tiers table, server/frontend code
13. **Frontend Integration** — Streaming Text Display (lines 883-937, the `fetch` + `ReadableStream` POST pattern and the simpler `EventSource` GET pattern), Structured Result Rendering (lines 939-958, the `esc()` helper and `renderResults()`)

**Prose style:** When copying content, fix any "MUST" language. Specifically, line 223 ("you MUST pair `--permission-mode dontAsk` with `--allowedTools`") should be rewritten as: "Pair `--permission-mode dontAsk` with `--allowedTools` or `--tools`. Without allowed tools, `dontAsk` gives Claude no tools at all — it can reason but can't act, and the failure is silent (no error, just missing results)."

Include a table of contents at the top. Introduce the file with one sentence: "Core server-side code patterns for Loom web apps — shared utilities, communication patterns, streaming event handling, error management, and frontend integration. Read this when implementing a specific server endpoint or wiring up the frontend."

**Step 2: Verify the file**

Run: `wc -l skills/loom/references/server-patterns.md`
Expected: ~600-700 lines

**Step 3: Verify no "MUST" language**

Run: `grep -n 'MUST' skills/loom/references/server-patterns.md`
Expected: No matches

**Step 4: Commit**

```bash
git add skills/loom/references/server-patterns.md
git commit -m "Extract server patterns to reference file for loom skill"
```

---

### Task 1b: Create `skills/loom/references/advanced-patterns.md`

**Files:**
- Create: `skills/loom/references/advanced-patterns.md`

**Step 1: Create the file**

Extract from `skills/loom/SKILL.md`:

1. **Structured Extraction (Async Haiku)** (lines 1170-1218) — extract() helper with timeout, error handling, and fallback parsing
2. **Persistent Session (Long-Lived Process)** (lines 1220-1302) — full code (startSession, sendMessage, endSession, inactivity timeout) + key differences list from per-request patterns
3. **Action Markers** (lines 1303-1334) — system prompt pattern for structured side-effects + when to use Action Markers vs HTTP Hooks (complementary roles)
4. **HTTP Hooks** (lines 1335-1555) — `.claude/settings.local.json` configuration, receiving hook events (Express endpoints, session routing), Interactive Permission Approval from the Browser (full flow, server endpoint with pending permissions map and timeout, frontend approval component, important considerations about timeout, permission-mode, session_id routing)

These patterns appear in production but aren't needed for every app. They're separated from the core server patterns because most apps start with REST + SSE and only reach for these when the basic patterns aren't enough.

Include a table of contents at the top. Intro: "Advanced server-side patterns for Loom web apps — async extraction, persistent sessions, mid-stream markers, and HTTP hooks for tool lifecycle events. Read this when the basic communication patterns (REST, SSE, WebSocket) aren't enough for your use case."

**Step 2: Verify the file**

Run: `wc -l skills/loom/references/advanced-patterns.md`
Expected: ~350-420 lines

**Step 3: Verify no "MUST" language**

Run: `grep -n 'MUST' skills/loom/references/advanced-patterns.md`
Expected: No matches

**Step 4: Commit**

```bash
git add skills/loom/references/advanced-patterns.md
git commit -m "Extract advanced patterns to reference file for loom skill"
```

---

### Task 2: Rewrite `skills/loom/SKILL.md` to under 500 lines

**Files:**
- Modify: `skills/loom/SKILL.md`

**Step 1: Replace SKILL.md with condensed version**

Rewrite the file with these sections:

1. **Metadata** — Rewrite the YAML front matter description. Descriptions should start with "Use when..." and describe only triggering conditions — never summarize what the skill does (that causes Claude to follow the description instead of reading the full skill). Revised description:

   ```yaml
   description: >
     Use when building a web application that needs Claude Code CLI (`claude -p`)
     or the Agent SDK as its backend runtime — a server that spawns Claude
     processes to power a custom browser interface. Triggers: "build an app that
     uses Claude", "Claude as backend/runtime", "Claude-powered web app", "wrap
     claude -p in a server", "streaming Claude output to browser", or any web app
     needing Claude's agentic capabilities through a purpose-built interface. NOT
     for direct Anthropic API usage, simple chat replicas, or desktop apps (use
     loom-desktop).
   ```

   Changes from original: Restructured to start with "Use when..." per skill description best practices. Lists triggering conditions and user phrases, not workflow steps. Added "Claude-powered web app" and "streaming Claude output to browser" as natural triggers. Added explicit "NOT for desktop apps (use loom-desktop)" cross-reference. ~85 words (well under 120).

2. **Intro + Why This Matters + Architecture** (lines 11-81) — Keep verbatim. This is mental model content, not implementation detail. (~71 lines)

3. **The Conversation** (lines 82-172) — Keep verbatim. Design guidance is the core skill. (~90 lines)

4. **Building It** — Section header + 2-line intro (lines 173-176). (~4 lines)

5. **Authentication Setup** — Condense to ~8 lines of prose summary explaining the pattern (OAuth, session store, per-user tokens). End with cross-reference:

   > Read `references/oauth-reference.md` for the complete implementation — PKCE
   > utilities, server endpoints, session store, `requireAuth` middleware, token
   > refresh, and the ready-to-use React `<SetupScreen>` component.

6. **The Server Layer** — Replace with a "Server Patterns" section containing:
   - Safety Defaults prose (~12 lines): the 3 non-negotiable flags paragraph plus the 3 failure modes paragraph. This is critical guidance, not code.
   - Shared Utilities 3-line summary naming `cleanEnv()`, `createStreamParser()`, `spawnEnvForUser()` with cross-ref.
   - "Patterns at a Glance" table listing each pattern with when to use:

     | Pattern | When to Use | Reference |
     |---------|-------------|-----------|
     | REST + JSON | One-shot requests, data extraction | `references/server-patterns.md#pattern-rest-endpoint` |
     | SSE Streaming | Streaming text to browser | `references/server-patterns.md#pattern-sse-streaming` |
     | WebSocket | Bidirectional, multi-turn | `references/server-patterns.md#pattern-websocket-session` |
     | Background Job | Long-running tasks | `references/server-patterns.md#pattern-background-job-with-progress` |
     | Parallel | Batch analysis | `references/server-patterns.md#pattern-parallel-analysis` |
     | Structured Extraction | Fast async data extraction (Haiku) | `references/advanced-patterns.md#structured-extraction-async-haiku` |
     | Persistent Session | Long-lived process, lower latency | `references/advanced-patterns.md#persistent-session-long-lived-process` |
     | Action Markers | Mid-stream structured events | `references/advanced-patterns.md#action-markers` |
     | HTTP Hooks | Tool lifecycle events, browser permission approval | `references/advanced-patterns.md#http-hooks` |

   (~35 lines total for this section)

7. **Stream-JSON Event Types** — Keep just the compact event table (the 7 rows from lines 793-802), without the extended notes, code samples, or thinking model discussion. Add one cross-ref line:

   > See `references/server-patterns.md#stream-json-event-types` for complete notes,
   > extended thinking behavior, code samples for extracting text/tool use, and
   > max-turns detection.

   (~15 lines)

8. **Frontend Integration** — Replace the inline code with a 5-line prose summary: "Use `fetch()` + `ReadableStream` for POST-based SSE (CSRF-safe). Parse `data:` lines, dispatch on event type (`token`, `done`, `error`). For quick prototyping, `EventSource` works for GET-based SSE." Then cross-ref:

   > See `references/server-patterns.md#frontend-integration` for the complete
   > streaming text display and structured result rendering code.

   (~8 lines)

9. **Error Handling** — Keep the mental model paragraph (3 failure modes: stderr, non-zero exit, malformed output, ~22 lines of prose). Remove the Error Surfacing Checklist code. **Update stale internal reference:**
    - Line 962: "Every pattern above handles errors inline" → "Every pattern in `references/server-patterns.md` handles errors inline"

   Cross-ref:

   > See `references/server-patterns.md#error-surfacing-checklist` for the
   > three-channel checklist with code samples.

   (~24 lines)

10. **What to Generate** — Keep content from lines 1556-1606, but **update stale internal references:**
    - Line 1560: "using the Server Setup block above" → "using the Express baseline from `references/server-patterns.md#server-setup`"
    - Any other "above"/"below" references that now point to content in reference files

    (~50 lines)

11. **First-Run Reliability Checklist** — Keep content from lines 1584-1601, but **update the lead-in:**
    - Line 1587: "The inline patterns above demonstrate correct handling for each" → "The patterns in `references/server-patterns.md` demonstrate correct handling for each"

    (~18 lines)

12. **Deployment Verification** — Keep verbatim (lines 1608-1621). (~14 lines)

13. **The Possibility Space** — Keep verbatim (lines 1623-1632). (~10 lines)

**Prose style pass:** After assembling the condensed file, scan for and fix:
- Any remaining "MUST"/"MUST NOT" → rewrite as imperative + consequence
- Any remaining "above"/"below" that reference content now in reference files → update to cross-ref the specific reference file and section

**Approximate line count breakdown:**

| Section | Lines |
|---------|-------|
| Metadata | 12 |
| Intro + Why + Architecture | 71 |
| The Conversation | 90 |
| Building It + Auth summary | 14 |
| Server Patterns (safety + utils + table) | 35 |
| Stream-JSON Event Types (table + ref) | 15 |
| Frontend Integration (summary + ref) | 8 |
| Error Handling (prose + ref) | 24 |
| What to Generate | 50 |
| First-Run Checklist | 18 |
| Deployment Verification | 14 |
| The Possibility Space | 10 |
| Blank lines, headings, markdown | ~50 |
| **Total** | **~411** |

This leaves ~89 lines of headroom below the 500-line limit.

**Step 2: Verify line count**

Run: `wc -l skills/loom/SKILL.md`
Expected: 380-450 lines

**Step 3: Verify no broken references**

Run a script to check that every `references/*.md` path mentioned in SKILL.md exists:
```bash
grep -oP 'references/[a-z-]+\.md' skills/loom/SKILL.md | sort -u | while read f; do [ -f "skills/loom/$f" ] && echo "OK: $f" || echo "MISSING: $f"; done
```
Expected: All OK, no MISSING (should find: cli-runtime-reference.md, oauth-reference.md, server-patterns.md, advanced-patterns.md)

**Step 4: Verify no orphan reference files**

```bash
ls skills/loom/references/*.md | while read f; do basename "$f" | xargs -I{} sh -c 'grep -q "{}" skills/loom/SKILL.md && echo "REFERENCED: {}" || echo "ORPHAN: {}"'; done
```
Expected: All REFERENCED

**Step 5: Verify no stale "above"/"below" references**

```bash
grep -n '\babove\b\|\bbelow\b' skills/loom/SKILL.md
```
Review each match. Allowed: references to content that is still in SKILL.md (e.g., "the Communication Patterns table above" if the table is still in the file). Disallowed: references to code patterns, utilities, or sections that have moved to reference files. Fix any that point to moved content.

**Step 6: Verify prose style**

```bash
grep -n 'MUST' skills/loom/SKILL.md
```
Expected: No matches

**Step 7: Verify section coverage**

Compare the original section headings against the new file — every heading should either be present in SKILL.md or have a cross-reference to the reference file where it now lives. Specifically verify these moved sections have cross-references:
- Shared Utilities → server-patterns
- Server Setup → server-patterns
- REST/SSE/WS/Background/Parallel patterns → server-patterns (via table)
- Stream-JSON Event Types details → server-patterns
- Error Surfacing Checklist → server-patterns
- Input Validation → server-patterns
- Temp Directory Cleanup → server-patterns
- Handling File Uploads → server-patterns
- Streaming Text Display → server-patterns
- Structured Result Rendering → server-patterns
- Structured Extraction → advanced-patterns (via table)
- Persistent Session → advanced-patterns (via table)
- Action Markers → advanced-patterns (via table)
- HTTP Hooks → advanced-patterns (via table)

**Step 8: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Condense loom SKILL.md to ~420 lines, cross-ref server and advanced patterns"
```

---

### Task 3: Create `skills/loom-desktop/references/desktop-patterns.md`

**Files:**
- Create: `skills/loom-desktop/references/desktop-patterns.md`

**Step 1: Create the file**

Extract from `skills/loom-desktop/SKILL.md`:

1. **Shared Utilities** (lines 217-328) — `resolveClaudePath()`, `cleanEnv()`, `createStreamParser()` — full code with explanatory prose
2. **`deriveAndSendRPC()`** (lines 345-423) — full function with state tracking
3. **Pattern 1: Synchronous** (lines 427-507) — Bun-side handler + "Don't do this"
4. **Pattern 2: Streaming** (lines 508-885) — full spawnClaude(), RPC handlers, webview-side React component (with module-level callbacks pattern), startup CLI check
5. **Pattern 3: Conversational** (lines 886-1091) — session management, conversational handler, Chat UI component, "Don't do this"
6. **Pattern 4: Background** (lines 1092-1263) — task registry, task list handler, system tray integration, completion notification, webview task list component

**Prose style:** Fix any "above"/"below" references that become stale after extraction. Specifically:
- Line 1391 "Already shown in Pattern 4 above" — update to "Already shown in Pattern 4" (it's in the same file now) or reference the section by name
- Line 1738 "The `deriveAndSendRPC` function above" — update to reference the section directly

Include a table of contents. Intro sentence: "Complete Bun-side and webview-side code patterns for Loom desktop apps. Read this when implementing a specific interaction pattern."

**Step 2: Verify**

Run: `wc -l skills/loom-desktop/references/desktop-patterns.md`
Expected: ~700-800 lines

**Step 3: Commit**

```bash
git add skills/loom-desktop/references/desktop-patterns.md
git commit -m "Extract desktop patterns to reference file for loom-desktop skill"
```

---

### Task 4: Create `skills/loom-desktop/references/desktop-features.md`

**Files:**
- Create: `skills/loom-desktop/references/desktop-features.md`

**Step 1: Create the file**

Extract from `skills/loom-desktop/SKILL.md`:

1. **File Drag-and-Drop** (lines 1270-1337) — webview drop handler (FileReader approach), Bun-side prompt building, explanation of why File.path doesn't work
2. **Native File Dialogs** (lines 1338-1388) — open file via Utils.openFileDialog(), save results via Bun.write()
3. **System Tray** (lines 1389-1398) — key points (brief, most code is in Pattern 4 — cross-ref `desktop-patterns.md#pattern-4-background` for the full implementation)
4. **Native Menus** (lines 1399-1467) — ApplicationMenu setup + event handling + accelerator notes
5. **Local File Access Configuration** (lines 1468-1490) — tools/permissions table + bypassPermissions rationale

Include a table of contents. Intro: "Desktop-specific features for Loom apps: file handling, native UX, and access configuration. Read this when adding native platform features to your app."

**Step 2: Verify**

Run: `wc -l skills/loom-desktop/references/desktop-features.md`
Expected: ~250-300 lines

**Step 3: Commit**

```bash
git add skills/loom-desktop/references/desktop-features.md
git commit -m "Extract desktop features to reference file for loom-desktop skill"
```

---

### Task 5: Create `skills/loom-desktop/references/distribution.md`

**Files:**
- Create: `skills/loom-desktop/references/distribution.md`

**Step 1: Create the file**

Extract from `skills/loom-desktop/SKILL.md`:

1. **Build** (lines 1493-1512) — build command, environments table, platform targeting
2. **Sharing the Build** (lines 1513-1527) — DMG not ZIP, xattr workaround
3. **Claude CLI as External Dependency** (lines 1528-1534) — don't bundle, startup check
4. **App Icon** (lines 1535-1564) — sips commands, config
5. **DMG Icon** (lines 1565-1577) — iconutil + DeRez/Rez workflow
6. **Code Signing & Notarization** (lines 1578-1626) — full 7-step setup
7. **Auto-Updates** (lines 1627-1661) — BSDIFF patching, Updater API

Intro: "Building, signing, and distributing Loom desktop apps. Read this when preparing your app for distribution or when setting up code signing."

**Step 2: Verify**

Run: `wc -l skills/loom-desktop/references/distribution.md`
Expected: ~200-250 lines

**Step 3: Commit**

```bash
git add skills/loom-desktop/references/distribution.md
git commit -m "Extract distribution guide to reference file for loom-desktop skill"
```

---

### Task 6: Create `skills/loom-desktop/references/gotchas.md`

**Files:**
- Create: `skills/loom-desktop/references/gotchas.md`

**Step 1: Create the file**

Extract all 16 gotchas from `skills/loom-desktop/SKILL.md` (lines 1662-1979) with their full detail and code samples. Keep the numbering and full explanations.

**Prose style:** Fix any stale "above"/"below" references and "MUST" language. Specifically:
- Line 1668 "**Must remove:**" → "**Remove:**" (imperative form — the explanation that follows already says why)
- Line 1672 "**Must NEVER remove:**" → "**Do not remove:**" (the next sentence already explains the consequence: "kills authentication and causes silent failures")
- Line 1738 "The `deriveAndSendRPC` function above handles both patterns" → "The `deriveAndSendRPC()` function in `desktop-patterns.md#deriveandsendrpc` handles both patterns"

Include a table of contents listing all 16 gotchas by number and title. Intro: "Critical issues from real-world Loom desktop development. Read these before building — they'll save you hours of debugging."

**Step 2: Verify**

Run: `wc -l skills/loom-desktop/references/gotchas.md`
Expected: ~330-370 lines

**Step 3: Commit**

```bash
git add skills/loom-desktop/references/gotchas.md
git commit -m "Extract gotchas to reference file for loom-desktop skill"
```

---

### Task 7: Rewrite `skills/loom-desktop/SKILL.md` to under 500 lines

**Files:**
- Modify: `skills/loom-desktop/SKILL.md`

**Step 1: Replace SKILL.md with condensed version**

Rewrite keeping:

1. **Metadata** — Rewrite the YAML front matter description. Descriptions should start with "Use when..." and describe only triggering conditions — never summarize what the skill does. Revised description:

   ```yaml
   description: >
     Use when building a native desktop application that needs Claude Code CLI
     (`claude -p`) as its runtime, using ElectroBun (Bun + system webview). The
     Bun process spawns Claude directly via typed RPC — no HTTP server, no auth,
     no SSE formatting. Triggers: "desktop app with Claude", "native Claude tool",
     "ElectroBun Claude app", "local AI agent", "Claude on the dock", or any
     desktop application needing Claude's agentic capabilities through a native
     interface. NOT for web apps (use loom), direct Anthropic API, or chat
     replicas.
   ```

   Changes from original: Restructured to start with "Use when..." per skill description best practices. Lists triggering conditions, not workflow steps. More concise (~88 words from 112), removed redundant triggers, added "Claude on the dock". Explicitly cross-refs loom for web.

2. **Intro + Why Desktop + Prerequisites** (lines 14-55) — Keep verbatim. (~42 lines)

3. **Architecture: Thin Bridge** (lines 56-115) — Keep verbatim including diagram and table. (~60 lines)

4. **Project Setup** (lines 116-134) — Keep, already concise with cross-refs to electrobun-setup.md and cli-runtime-reference.md. (~18 lines)

5. **The Conversation** (lines 135-211) — Keep verbatim. (~76 lines)

6. **Building It** intro — Keep 3 lines (lines 212-216).

7. **Shared Utilities** — 5-line summary listing the four helpers (`resolveClaudePath()`, `cleanEnv()`, `createStreamParser()`, `deriveAndSendRPC()`), with cross-ref to `references/desktop-patterns.md#shared-utilities`.

8. **Event Type Mapping** — Keep the mapping table (lines 330-343). (~14 lines)

9. **Patterns** — Replace with a "Patterns at a Glance" section: table listing each pattern with when to use and cross-ref. Include a 3-line summary of the Streaming pattern since it's the default. (~20 lines)

   | Pattern | When to Use | Reference |
   |---------|-------------|-----------|
   | Synchronous | Quick extraction, classification | `references/desktop-patterns.md#pattern-1-synchronous` |
   | Streaming | Primary pattern — tokens flow to UI | `references/desktop-patterns.md#pattern-2-streaming` |
   | Conversational | Multi-turn with context retention | `references/desktop-patterns.md#pattern-3-conversational` |
   | Background | Long tasks, tray minimization | `references/desktop-patterns.md#pattern-4-background` |

10. **Desktop Features** — Replace with a summary table: feature name, 1-line description, cross-ref. (~15 lines)

    | Feature | Description | Reference |
    |---------|-------------|-----------|
    | File Drag-and-Drop | Drop files for Claude to analyze (use FileReader, not File.path) | `references/desktop-features.md#file-drag-and-drop` |
    | Native File Dialogs | Open/save files via Utils.openFileDialog() | `references/desktop-features.md#native-file-dialogs` |
    | System Tray | Minimize during long tasks, show progress | `references/desktop-features.md#system-tray` |
    | Native Menus | App menu bar with keyboard shortcuts | `references/desktop-features.md#native-menus` |
    | File Access Config | Permission modes and tools lists for desktop | `references/desktop-features.md#local-file-access-configuration` |

11. **Distribution** — Replace with 5-line summary + cross-ref to `references/distribution.md`. Key points: `bunx electrobun build --env=stable`, always DMG never ZIP, Claude CLI is an external dependency, code signing requires Apple Developer account. (~8 lines)

12. **Gotchas** — Replace with numbered list of 1-line descriptions (title + core fix), ~16 entries. Keep the top 4 most critical gotchas with 2-3 lines of detail since they bite everyone: (1) env cleaning, (2) stream buffering, (3) macOS PATH, (4) File.path. Cross-ref to `references/gotchas.md` for full detail with code. (~45 lines)

13. **What to Generate** (lines 1980-2010) — Keep verbatim. (~30 lines)

14. **The Possibility Space** (lines 2011-2026) — Keep verbatim. (~15 lines)

**Prose style pass:** After assembling the condensed file:
- Scan for any "MUST"/"MUST NOT" → rewrite as imperative + consequence
- Scan for any "above"/"below" references pointing to content now in reference files → update to cross-ref the specific reference file

**Approximate line count breakdown:**

| Section | Lines |
|---------|-------|
| Metadata | 13 |
| Intro + Why Desktop + Prerequisites | 42 |
| Architecture: Thin Bridge | 60 |
| Project Setup | 18 |
| The Conversation | 76 |
| Building It + Shared Utils summary | 10 |
| Event Type Mapping | 14 |
| Patterns at a Glance | 20 |
| Desktop Features table | 15 |
| Distribution summary | 8 |
| Gotchas condensed | 45 |
| What to Generate | 30 |
| The Possibility Space | 15 |
| Blank lines, headings, markdown | ~50 |
| **Total** | **~416** |

This leaves ~84 lines of headroom below the 500-line limit.

**Step 2: Verify line count**

Run: `wc -l skills/loom-desktop/SKILL.md`
Expected: 390-470 lines

**Step 3: Verify no broken references**

```bash
grep -oP 'references/[a-z-]+\.md' skills/loom-desktop/SKILL.md | sort -u | while read f; do [ -f "skills/loom-desktop/$f" ] && echo "OK: $f" || echo "MISSING: $f"; done
```
Expected: All OK (should find: cli-runtime-reference.md, electrobun-setup.md, rpc-schema-reference.md, desktop-patterns.md, desktop-features.md, distribution.md, gotchas.md)

**Step 4: Verify no orphan reference files**

```bash
ls skills/loom-desktop/references/*.md | while read f; do basename "$f" | xargs -I{} sh -c 'grep -q "{}" skills/loom-desktop/SKILL.md && echo "REFERENCED: {}" || echo "ORPHAN: {}"'; done
```
Expected: All REFERENCED

**Step 5: Verify no stale "above"/"below" references**

```bash
grep -n '\babove\b\|\bbelow\b' skills/loom-desktop/SKILL.md
```
Review each match. Fix any that point to content now in reference files.

**Step 6: Verify prose style**

```bash
grep -n 'MUST' skills/loom-desktop/SKILL.md
```
Expected: No matches

**Step 7: Verify section coverage**

Confirm all original major headings are either present in SKILL.md or have a cross-reference. Specifically verify:
- Shared Utilities → desktop-patterns
- deriveAndSendRPC → desktop-patterns
- All 4 patterns → desktop-patterns (via table)
- All 5 desktop features → desktop-features (via table)
- All 7 distribution sections → distribution
- All 16 gotchas → gotchas (1-line summaries in SKILL.md + cross-ref)

**Step 8: Commit**

```bash
git add skills/loom-desktop/SKILL.md
git commit -m "Condense loom-desktop SKILL.md to ~420 lines, cross-ref extracted content"
```

---

### Task 8: Final verification

**Files:**
- Read: all modified files

**Step 1: Run line counts**

```bash
wc -l skills/loom/SKILL.md skills/loom/references/*.md skills/loom-desktop/SKILL.md skills/loom-desktop/references/*.md
```

Expected:
- `skills/loom/SKILL.md` < 500
- `skills/loom-desktop/SKILL.md` < 500
- All reference files exist
- Loom total: original ~2804 lines (SKILL 1632 + cli-ref 479 + oauth-ref 693) → new ~2800 lines (SKILL ~420 + cli-ref 479 + oauth-ref 693 + server-patterns ~650 + advanced-patterns ~380)
- Loom-desktop total: original ~3162 lines (SKILL 2026 + cli-ref 460 + electrobun 280 + rpc-schema 396) → new ~3100 lines (SKILL ~420 + cli-ref 460 + electrobun 280 + rpc-schema 396 + desktop-patterns ~750 + desktop-features ~270 + distribution ~220 + gotchas ~350)

**Step 2: Verify reference integrity**

For each skill, check that every `references/*.md` path in SKILL.md resolves and that there are no orphaned reference files.

**Step 3: Verify section headings**

For each skill, confirm all original major headings are either present in SKILL.md or cross-referenced from it.

**Step 4: Check TOC on large reference files**

Any reference file over 300 lines should have a table of contents at the top. Expected to need TOCs:
- `skills/loom/references/server-patterns.md` (~650 lines) — needs TOC
- `skills/loom/references/advanced-patterns.md` (~380 lines) — needs TOC
- `skills/loom-desktop/references/desktop-patterns.md` (~750 lines) — needs TOC
- `skills/loom-desktop/references/gotchas.md` (~350 lines) — needs TOC

**Step 5: Verify metadata descriptions**

Count words in both SKILL.md description blocks. Both should be under 120 words.

```bash
head -12 skills/loom/SKILL.md | sed -n '/description:/,/^---/p' | head -n -1 | wc -w
head -13 skills/loom-desktop/SKILL.md | sed -n '/description:/,/^---/p' | head -n -1 | wc -w
```

**Step 6: Verify no stale internal references (final pass)**

```bash
grep -n '\babove\b\|\bbelow\b' skills/loom/SKILL.md skills/loom-desktop/SKILL.md
```
Review each match against the file structure. Every "above"/"below" should reference content that is still in the same file.

**Step 7: Verify prose style (final pass)**

```bash
grep -rn 'MUST' skills/loom/SKILL.md skills/loom/references/*.md skills/loom-desktop/SKILL.md skills/loom-desktop/references/*.md
```
Expected: No matches. If any found, rewrite as imperative + consequence.

**Step 8: Commit any fixes from verification**

```bash
git add -A
git commit -m "Fix any issues found during verification"
```

---

## Remember
- Exact file paths always
- Complete code in plan (not "add validation")
- Exact commands with expected output
- DRY, YAGNI, TDD, frequent commits
- Content is MOVED to references, never deleted
- Every reference cross-link includes guidance on WHEN to read it
- Reference files over 300 lines get a table of contents
- Descriptions are REWRITTEN per best practices, not kept as-is
- Update "above"/"below" references when content moves to reference files
- No "MUST" — use imperative form and explain consequences
