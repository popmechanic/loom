# Improve Loom Skills to Follow Best Practices — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use trycycle-executing to implement this plan task-by-task.

**Goal:** Refactor `skills/loom/SKILL.md` (1632 lines) and `skills/loom-desktop/SKILL.md` (2026 lines) to be under 500 lines each by extracting content to reference files, improving metadata, and following skill writing best practices.

**Architecture:** Each SKILL.md becomes a concise orchestration document — mental model, conversation guide, key patterns (summarized with cross-references), what to generate, and gotchas. Detailed code patterns, event type docs, and feature-specific content move to well-organized reference files with tables of contents. No content is deleted — it's relocated to references.

**Tech Stack:** Markdown files only. No code changes.

---

## Analysis: What stays, what moves

### loom/SKILL.md (1632 → ~450 lines target)

**Keep in SKILL.md (core guidance):**
- Metadata (9 lines) — polish description
- Intro + Why This Matters + Architecture (lines 1-81, ~81 lines)
- The Conversation (lines 82-172, ~90 lines)
- Building It intro + Auth Setup summary (lines 173-203, ~30 lines — condense, point to oauth-reference)
- Safety Defaults summary (lines 212-240, ~10 lines condensed)
- Shared Utilities summary (lines 242-328, ~15 lines condensed — code stays in reference)
- Server Setup summary (lines 329-356, ~5 lines condensed)
- Pattern summaries (REST, SSE, WS, Background, Parallel) — ~30 lines total summarizing when to use each, then cross-ref
- Frontend Layer summary (lines 879-958, ~15 lines condensed)
- Error Handling mental model (lines 960-982, ~20 lines — keep this, it's guidance not code)
- What to Generate (lines 1556-1606, ~50 lines) — keep, this is the deliverable checklist
- First-Run Reliability Checklist (lines 1584-1601, ~18 lines) — keep, operational
- Deployment Verification (lines 1608-1621, ~14 lines) — keep
- The Possibility Space (lines 1623-1632, ~10 lines) — keep

**Extract to references:**
- `references/server-patterns.md` (~650 lines): All server-side code patterns (REST, SSE, WebSocket, Background Job, Parallel, Stream-JSON Event Types, Advanced Patterns including Structured Extraction, Persistent Session, Action Markers, HTTP Hooks + Interactive Permission)
- `references/frontend-patterns.md` (~150 lines): Streaming Text Display, Structured Result Rendering, Error Surfacing Checklist, Input Validation, Temp Directory Cleanup, Handling File Uploads and Drops
- Shared Utilities code (cleanEnv, createStreamParser, spawnEnvForUser, Server Setup) — move to `references/server-patterns.md` at the top

**Existing references stay as-is:**
- `references/cli-runtime-reference.md` (479 lines) — already well-extracted, has TOC
- `references/oauth-reference.md` (693 lines) — already well-extracted, has TOC

### loom-desktop/SKILL.md (2026 → ~480 lines target)

**Keep in SKILL.md (core guidance):**
- Metadata (13 lines) — polish description
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
- `references/desktop-features.md` (~350 lines): File Drag-and-Drop, Native File Dialogs, System Tray, Native Menus, Local File Access Configuration
- `references/distribution.md` (~250 lines): Build, Sharing, Claude CLI dep, App Icon, DMG Icon, Code Signing & Notarization, Auto-Updates
- `references/gotchas.md` (~350 lines): All 16 gotchas with full detail and code samples

**Existing references stay as-is:**
- `references/cli-runtime-reference.md` (460 lines) — already well-extracted, has TOC
- `references/electrobun-setup.md` (280 lines) — already well-extracted
- `references/rpc-schema-reference.md` (396 lines) — already well-extracted, has TOC

---

## Verification Criteria

1. **Line counts**: Both SKILL.md under 500 lines
2. **Metadata**: Each description under ~120 words with trigger phrases
3. **Reference integrity**: Every `references/*.md` path mentioned in SKILL.md exists; no orphan reference files
4. **Section heading diff**: No major sections silently dropped — all headings from original appear as either kept sections or cross-references
5. **Content balance**: Total line count across all files per skill stays roughly the same (content moved, not deleted)
6. **TOC**: Reference files over 300 lines include a table of contents
7. **Cross-references**: Each reference file is cross-referenced from SKILL.md with "read this when..." guidance

---

### Task 1: Create `skills/loom/references/server-patterns.md`

**Files:**
- Create: `skills/loom/references/server-patterns.md`

**Step 1: Create the file**

Extract the following sections from `skills/loom/SKILL.md` into a new reference file with a table of contents:

1. **Shared Utilities** (lines 242-328) — `cleanEnv()`, `createStreamParser()`, `spawnEnvForUser()` — full code
2. **Server Setup** (lines 329-356) — Express baseline — full code
3. **Pattern: REST Endpoint** (lines 358-416) — full code + "Don't do this"
4. **Pattern: SSE Streaming** (lines 417-526) — full code + "Don't do this"
5. **Pattern: WebSocket Session** (lines 527-654) — full code + "Don't do this"
6. **Pattern: Background Job with Progress** (lines 655-721) — full code + "Don't do this"
7. **Pattern: Parallel Analysis** (lines 722-775) — full code
8. **Stream-JSON Event Types** (lines 777-878) — event table, notes, code samples
9. **Error Surfacing Checklist** (lines 983-1030) — checklist with code
10. **Input Validation** (lines 1031-1055) — safePath helper + prompt defense
11. **Temp Directory Cleanup** (lines 1056-1078) — withTempDir helper
12. **Handling File Uploads and Drops** (lines 1079-1163) — tiers table, server/frontend code
13. **Structured Extraction (Async Haiku)** (lines 1170-1218) — extract() helper
14. **Persistent Session (Long-Lived Process)** (lines 1220-1302) — full code
15. **Action Markers** (lines 1303-1334) — system prompt pattern
16. **HTTP Hooks** (lines 1335-1555) — config, receiving events, interactive permission approval

Include a table of contents at the top. Introduce the file with one sentence: "Complete server-side code patterns for Loom web apps. Read this when implementing a specific communication pattern or advanced feature."

**Step 2: Verify the file**

Run: `wc -l skills/loom/references/server-patterns.md`
Expected: ~900-1000 lines

**Step 3: Commit**

```bash
git add skills/loom/references/server-patterns.md
git commit -m "Extract server patterns to reference file for loom skill"
```

---

### Task 2: Rewrite `skills/loom/SKILL.md` to under 500 lines

**Files:**
- Modify: `skills/loom/SKILL.md`

**Step 1: Replace SKILL.md with condensed version**

Rewrite the file keeping:

1. **Metadata** — Keep the current description. It's already under 120 words and has good trigger phrases.
2. **Intro + Why This Matters + Architecture** (lines 1-81) — Keep verbatim. This is mental model content, not implementation detail.
3. **The Conversation** (lines 82-172) — Keep verbatim. Design guidance is the core skill.
4. **Building It** intro + **Authentication Setup** — Condense to ~10 lines. Point to `references/oauth-reference.md` for the complete implementation.
5. **The Server Layer** — Replace with a "Patterns at a Glance" section: a table listing each pattern with when to use it and a cross-reference to `references/server-patterns.md#pattern-name`. Include the Safety Defaults paragraph (3 flags are non-negotiable) since that's critical guidance, not code.
6. **Shared Utilities** — 3-line summary listing the three helpers with a cross-ref to server-patterns.
7. **Stream-JSON Event Types** — Keep just the event table (lines 793-802) and the "extracting text and tool use" code snippet (lines 833-847). Cross-ref server-patterns for the full notes.
8. **Frontend Layer** — Keep the streaming text display code (~40 lines of JS) since it's the minimum a reader needs to wire up the frontend. Cross-ref server-patterns for structured rendering, error surfacing, file uploads.
9. **Error Handling** — Keep the mental model paragraph (3 failure modes) but remove the checklist code samples — cross-ref server-patterns.
10. **What to Generate** — Keep verbatim.
11. **First-Run Reliability Checklist** — Keep verbatim.
12. **Deployment Verification** — Keep verbatim.
13. **The Possibility Space** — Keep verbatim.

The key cross-reference pattern for each extraction:

```markdown
> Read `references/server-patterns.md#section-name` for the complete implementation
> with code samples and "don't do this" warnings.
```

**Step 2: Verify line count**

Run: `wc -l skills/loom/SKILL.md`
Expected: 400-500 lines

**Step 3: Verify no broken references**

Run a script to check that every `references/*.md` path mentioned in SKILL.md exists:
```bash
grep -oP 'references/[a-z-]+\.md' skills/loom/SKILL.md | sort -u | while read f; do [ -f "skills/loom/$f" ] && echo "OK: $f" || echo "MISSING: $f"; done
```
Expected: All OK, no MISSING

**Step 4: Verify section coverage**

Compare the original section headings against the new file — every heading should either be present or have a cross-reference.

**Step 5: Commit**

```bash
git add skills/loom/SKILL.md
git commit -m "Condense loom SKILL.md to ~450 lines, cross-ref server-patterns"
```

---

### Task 3: Create `skills/loom-desktop/references/desktop-patterns.md`

**Files:**
- Create: `skills/loom-desktop/references/desktop-patterns.md`

**Step 1: Create the file**

Extract from `skills/loom-desktop/SKILL.md`:

1. **Shared Utilities** (lines 217-328) — `resolveClaudePath()`, `cleanEnv()`, `createStreamParser()` — full code
2. **`deriveAndSendRPC()`** (lines 345-423) — full function
3. **Pattern 1: Synchronous** (lines 427-507) — Bun-side handler + "Don't do this"
4. **Pattern 2: Streaming** (lines 508-885) — full spawnClaude(), RPC handlers, webview-side React component, startup CLI check
5. **Pattern 3: Conversational** (lines 886-1091) — session management, conversational handler, Chat UI component, "Don't do this"
6. **Pattern 4: Background** (lines 1092-1263) — task registry, task list handler, system tray integration, completion notification, webview task list component

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

1. **File Drag-and-Drop** (lines 1270-1337) — webview drop handler, Bun-side prompt building
2. **Native File Dialogs** (lines 1338-1388) — open file, save results
3. **System Tray** (lines 1389-1398) — key points (brief, most code is in Pattern 4)
4. **Native Menus** (lines 1399-1467) — ApplicationMenu setup + event handling
5. **Local File Access Configuration** (lines 1468-1490) — tools/permissions table

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

1. **Build** (lines 1493-1512)
2. **Sharing the Build** (lines 1513-1527)
3. **Claude CLI as External Dependency** (lines 1528-1534)
4. **App Icon** (lines 1535-1564)
5. **DMG Icon** (lines 1565-1577)
6. **Code Signing & Notarization** (lines 1578-1626)
7. **Auto-Updates** (lines 1627-1661)

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

Include a table of contents listing all 16 gotchas. Intro: "Critical issues from real-world Loom desktop development. Read these before building — they'll save you hours of debugging."

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

1. **Metadata** — Keep current description, it's good.
2. **Intro + Why Desktop + Prerequisites** (lines 1-55) — Keep verbatim.
3. **Architecture: Thin Bridge** (lines 56-115) — Keep verbatim including diagram and table.
4. **Project Setup** (lines 116-134) — Keep, already concise with cross-refs.
5. **The Conversation** (lines 135-211) — Keep verbatim.
6. **Building It** intro — Keep 5 lines.
7. **Shared Utilities** — 5-line summary listing the three helpers, cross-ref to `references/desktop-patterns.md#shared-utilities`.
8. **Event Type Mapping** — Keep the mapping table (lines 330-343).
9. **Patterns** — Replace with a "Patterns at a Glance" section: table listing each pattern with when to use and cross-ref. Include a 3-line summary of the Streaming pattern (the core one) since it's the default.
10. **Desktop Features** — Replace with a summary table: feature name, 1-line description, cross-ref to `references/desktop-features.md#section`.
11. **Distribution** — Replace with 5-line summary + cross-ref to `references/distribution.md`.
12. **Gotchas** — Replace with numbered list of 1-line descriptions (title + core fix). Cross-ref to `references/gotchas.md` for full detail with code. Keep the top 3-4 most critical gotchas (env cleaning, stream buffering, macOS PATH, File.path) with 2-3 line detail since they bite everyone.
13. **What to Generate** (lines 1980-2010) — Keep verbatim.
14. **The Possibility Space** (lines 2011-2026) — Keep verbatim.

**Step 2: Verify line count**

Run: `wc -l skills/loom-desktop/SKILL.md`
Expected: 400-500 lines

**Step 3: Verify no broken references**

```bash
grep -oP 'references/[a-z-]+\.md' skills/loom-desktop/SKILL.md | sort -u | while read f; do [ -f "skills/loom-desktop/$f" ] && echo "OK: $f" || echo "MISSING: $f"; done
```
Expected: All OK

**Step 4: Verify section coverage**

**Step 5: Commit**

```bash
git add skills/loom-desktop/SKILL.md
git commit -m "Condense loom-desktop SKILL.md to ~470 lines, cross-ref extracted content"
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
- Total lines across all files per skill roughly match original totals (content moved not deleted)

**Step 2: Verify reference integrity**

For each skill, check that every `references/*.md` path in SKILL.md resolves and that there are no orphaned reference files.

**Step 3: Verify section headings**

For each skill, confirm all original major headings are either present in SKILL.md or cross-referenced from it.

**Step 4: Check TOC on large reference files**

Any reference file over 300 lines should have a table of contents at the top.

**Step 5: Commit any fixes from verification**

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
