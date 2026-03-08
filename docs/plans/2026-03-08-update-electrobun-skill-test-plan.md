# Test Plan: ElectroBun v1.15.1 Update & loom-desktop Skill Review

## Strategy reconciliation

The agreed testing strategy is: cross-reference every API name, config field, import path, and code example against the actual ElectroBun v1.15.1 source on GitHub, at Medium fidelity.

The implementation plan confirms this approach is correct. The artifact is four Markdown skill files with embedded TypeScript code examples — not executable code. There are no test harnesses to build, no APIs to call, no services to deploy. Verification is entirely textual: search the modified files for stale references and compare every factual claim against the v1.15.1 source of truth.

**No strategy changes needed.** The plan touches exactly two files (SKILL.md and electrobun-setup.md) across 13 tasks, with no new files created. The interaction surface is documentation correctness, not runtime behavior.

---

## Test Plan

### 1. After all tasks: no stale version references remain

- **Name**: All version strings reference v1.15.1, not v1.14.4 or v1.14.5-beta
- **Type**: invariant
- **Harness**: Text search (`grep`) across all files in `skills/loom-desktop/`
- **Preconditions**: All 13 tasks from the implementation plan are complete.
- **Actions**:
  1. `grep -rn 'v1\.14' skills/loom-desktop/`
  2. `grep -rn 'v1.14' skills/loom-desktop/`
- **Expected outcome**: Zero matches. Source of truth: ElectroBun v1.15.1 is the current stable release per the GitHub releases page and the implementation plan.
- **Interactions**: None.

### 2. After all tasks: no stale import path `electrobun/config` remains

- **Name**: Config import uses `"electrobun"` not `"electrobun/config"`
- **Type**: invariant
- **Harness**: Text search across all files in `skills/loom-desktop/`
- **Preconditions**: Task 1 (fix import path) is complete.
- **Actions**:
  1. `grep -rn 'electrobun/config' skills/loom-desktop/`
- **Expected outcome**: Zero matches. Source of truth: ElectroBun v1.15.1 `package.json` exports map — `"."` maps to `"./dist/api/bun/index.ts"` which re-exports `ElectrobunConfig`. The path `"electrobun/config"` is not in the exports map.
- **Interactions**: Affects both `electrobun-setup.md` (Task 1) and any code examples in SKILL.md that might reference the config import.

### 3. After all tasks: no stale template `interactive-playground` remains

- **Name**: Removed template references are gone, all 19 real templates are listed
- **Type**: invariant
- **Harness**: Text search + manual count
- **Preconditions**: Task 2 (update template table) is complete.
- **Actions**:
  1. `grep -rn 'interactive-playground' skills/loom-desktop/`
  2. Count rows in the template table in `electrobun-setup.md` (should be 19 data rows).
  3. Cross-reference template names against the `templates/` directory in ElectroBun v1.15.1 source.
- **Expected outcome**: Zero matches for `interactive-playground`. Table contains exactly 19 templates matching the subdirectories under `templates/` in the ElectroBun repo at v1.15.1. Source of truth: `https://github.com/blackboardsh/electrobun/tree/main/templates`.
- **Interactions**: Only affects `electrobun-setup.md`.

### 4. After all tasks: `copy` config uses build-level object format everywhere

- **Name**: Copy config is `build.copy: { source: dest }`, not `views.mainview.copy: [...]`
- **Type**: invariant
- **Harness**: Text search
- **Preconditions**: Tasks 3 and 4 (fix copy config format) are complete.
- **Actions**:
  1. `grep -rn 'copy: \[' skills/loom-desktop/`
  2. Visually inspect every occurrence of `copy` in config examples in both SKILL.md and electrobun-setup.md. Verify `copy` is at the `build` level as `{ "source": "dest" }`.
- **Expected outcome**: Zero matches for array-format `copy: [`. All copy examples show `build.copy` as an object mapping source paths to destination paths. Source of truth: ElectroBun v1.15.1 `ElectrobunConfig` type definition in `src/shared/types.ts` — `copy` is typed as `{ [sourcePath: string]: string }` at the `build` level.
- **Interactions**: Affects SKILL.md (Gotcha #10) and electrobun-setup.md (Configuration section).

### 5. After all tasks: all `Bun.spawn` calls use `CLAUDE_BIN` not `"claude"`

- **Name**: Every spawn call in code examples uses CLAUDE_BIN for reliability
- **Type**: invariant
- **Harness**: Text search with context
- **Preconditions**: Task 6 (CLAUDE_BIN consistency) is complete.
- **Actions**:
  1. Search SKILL.md for all occurrences of `Bun.spawn` and `Bun.spawnSync`.
  2. For each match, verify the first argument in the args array is `CLAUDE_BIN`, not the string `"claude"`.
  3. Exception: the `resolveClaudePath()` function itself (Gotcha #13 and Shared Utilities) legitimately references the string `"claude"` as a fallback return value and in `which claude` — these are not spawn calls.
- **Expected outcome**: Every `Bun.spawn(["claude"` and `Bun.spawnSync(["claude"` is replaced with `Bun.spawn([CLAUDE_BIN` / `Bun.spawnSync([CLAUDE_BIN`. The only remaining `"claude"` strings should be in: (a) the `resolveClaudePath()` function's fallback `return "claude"`, (b) shell commands like `which claude`, (c) prose text. Source of truth: The skill's own Shared Utilities section defines `CLAUDE_BIN = resolveClaudePath()` and states "use `CLAUDE_BIN` in all `Bun.spawn()` calls."
- **Interactions**: Affects Pattern 1 (sync), Pattern 2 (streaming), and the startup CLI check in SKILL.md.

### 6. After all tasks: `--continue` is not referenced as a session flag in Pattern 3

- **Name**: Session management uses `--resume`, not `--continue`
- **Type**: regression
- **Harness**: Text search with context
- **Preconditions**: Task 7 (fix stale `--continue` reference) is complete.
- **Actions**:
  1. `grep -n '\-\-continue' skills/loom-desktop/SKILL.md`
  2. For any matches, verify they are NOT in the context of `spawnClaude` session management or the Pattern 3 "Don't do this" section.
  3. Verify the Pattern 3 note says `--resume <id>` for follow-up turns.
  4. Verify the "Don't do this" bullets reference `--resume`, not `--continue`.
- **Expected outcome**: The Pattern 3 note (after the conversational handler) references `--resume <id>` for follow-up turns. The "Don't do this" bullets reference `--resume`. Any remaining `--continue` references in the file are in the cli-runtime-reference.md context (which correctly documents `--continue` as "resume most recent session" — a different use case). Source of truth: `cli-runtime-reference.md` Session Management section — `--session-id` on first turn, `--resume <id>` on subsequent turns. `--continue` resumes the *most recent* session, which is wrong for multi-session apps.
- **Interactions**: Affects SKILL.md only. The cli-runtime-reference.md correctly documents both `--continue` and `--resume` as separate features.

### 7. User builds a new Loom desktop app following the updated skill

- **Name**: A user starting from scratch can follow the skill to scaffold and understand the full stack
- **Type**: scenario
- **Harness**: Manual walkthrough reading the skill files in order
- **Preconditions**: All tasks complete. Reading the files as a user would.
- **Actions**:
  1. Read SKILL.md from top. Verify the version reference in Prerequisites (line ~53) says v1.15.1.
  2. Follow the `@references/electrobun-setup.md` link. Verify the template table shows 19 templates.
  3. In `electrobun-setup.md`, read the Configuration section. Verify the import is `from "electrobun"` (not `"electrobun/config"`).
  4. Verify the config example includes `build.copy` at the correct level.
  5. Return to SKILL.md. Read the Shared Utilities section. Verify `CLAUDE_BIN` is defined and the note says to use it everywhere.
  6. Read Pattern 1 (Synchronous). Verify the `Bun.spawnSync` call uses `CLAUDE_BIN`.
  7. Read Pattern 2 (Streaming). Verify the `Bun.spawn` call uses `CLAUDE_BIN`.
  8. Read Pattern 3 (Conversational). Verify the note references `--resume` for follow-up turns.
  9. Read the "Don't do this" bullets after Pattern 3. Verify they reference `--resume`.
  10. Read Gotcha #7. Verify it says v1.15.1 and mentions new APIs.
  11. Read Gotcha #10 (Missing index.html). Verify the `copy` directive uses `build.copy` object format.
  12. Read the startup CLI check. Verify it uses `CLAUDE_BIN`.
  13. Read the What to Generate checklist. Verify it lists `resolveClaudePath()` and does not list `deriveAndSendRPC()`.
- **Expected outcome**: Every step passes. A user following the skill end-to-end encounters no stale references, no incorrect import paths, no wrong config formats, and no string `"claude"` in spawn calls. Source of truth: All assertions trace to the ElectroBun v1.15.1 source and the skill's own internal consistency (CLAUDE_BIN defined in Shared Utilities, referenced everywhere else).
- **Interactions**: Exercises the full interaction surface: both modified files, all code examples, all prose references.

### 8. New v1.15.1 APIs are documented in electrobun-setup.md

- **Name**: Key new APIs (GlobalShortcut, Screen, Session, BuildConfig, ContextMenu) are mentioned
- **Type**: scenario
- **Harness**: Text search + manual review
- **Preconditions**: Task 8 (add new APIs) is complete.
- **Actions**:
  1. Read electrobun-setup.md Key Concepts section.
  2. Verify subsections exist for: GlobalShortcut, Screen, Session, BuildConfig, Context Menus.
  3. Verify each subsection has a brief code example with the correct import path (`from "electrobun/bun"`).
  4. Cross-reference each API name against the ElectroBun v1.15.1 source exports in `src/api/bun/index.ts`.
  5. Verify the new config options list mentions: `urlSchemes`, `useAsar`, `bundleWGPU`, `targets`, `scripts`, `runtime`.
- **Expected outcome**: All five API subsections exist with correct imports. The config options list covers the six items. Source of truth: ElectroBun v1.15.1 `src/api/bun/index.ts` for API exports; `src/shared/types.ts` for config type definition.
- **Interactions**: Only affects electrobun-setup.md. The SKILL.md Gotcha #7 also mentions these APIs in passing.

### 9. Gotcha #7 version status is updated and mentions new APIs

- **Name**: Gotcha #7 reflects v1.15.1 status with new API summary
- **Type**: integration
- **Harness**: Manual review of SKILL.md Gotcha #7
- **Preconditions**: Task 5 (update version references) is complete.
- **Actions**:
  1. Read Gotcha #7 in SKILL.md.
  2. Verify title says "ElectroBun version status" (not "beta status").
  3. Verify version says v1.15.1.
  4. Verify the "New in v1.15.x" line mentions: `--watch` race condition fixed, WebGPU/WGPU, GpuWindow, GlobalShortcut, Screen, Session, Socket, BuildConfig.
- **Expected outcome**: All items present and correctly named. Source of truth: ElectroBun v1.15.1 changelog/release notes and the implementation plan's enumeration of new APIs.
- **Interactions**: Cross-references with electrobun-setup.md (Task 8) which documents these APIs in detail.

### 10. Process crash recovery gotcha exists and is correct

- **Name**: Gotcha #16 provides a retry pattern for transient Claude process failures
- **Type**: integration
- **Harness**: Manual review of SKILL.md
- **Preconditions**: Task 10 (add error recovery pattern) is complete.
- **Actions**:
  1. Read Gotcha #16 in SKILL.md.
  2. Verify the `spawnWithRetry` function: calls `spawnClaude`, checks `proc.exited`, retries with exponential backoff.
  3. Verify the note says not to retry on user-initiated aborts (SIGTERM).
  4. Verify the function signature references `spawnClaude` from Pattern 2.
- **Expected outcome**: Gotcha #16 exists, the retry function is syntactically valid TypeScript, and the guidance about SIGTERM is present. Source of truth: The skill's own Pattern 2 `spawnClaude` function (the retry wrapper delegates to it). The retry pattern itself follows standard exponential backoff best practices.
- **Interactions**: References `spawnClaude` from Pattern 2. If spawnClaude's signature changes, this gotcha's code would need updating.

### 11. What to Generate checklist is updated

- **Name**: Checklist lists `resolveClaudePath()` and does not list `deriveAndSendRPC()`
- **Type**: boundary
- **Harness**: Text search
- **Preconditions**: Task 12 (update checklist) is complete.
- **Actions**:
  1. Read the "What to Generate" checklist in SKILL.md.
  2. Verify the `claude-manager.ts` line includes `resolveClaudePath()`.
  3. Verify `deriveAndSendRPC()` is NOT in the checklist.
- **Expected outcome**: The checklist line reads: `resolveClaudePath()`, `cleanEnv()`, `createStreamParser()`, `spawnClaude()`, `abort()`, heartbeat. Source of truth: The Shared Utilities section defines `resolveClaudePath()` as a required helper; `deriveAndSendRPC()` is inlined in `spawnClaude()` in the implementation (Pattern 2), so listing it separately in the checklist is misleading.
- **Interactions**: None.

### 12. `--watch` documentation is accurate for v1.15.1

- **Name**: The `--watch` flag is documented as working (race condition fixed)
- **Type**: boundary
- **Harness**: Manual review
- **Preconditions**: Task 9 (verify --watch docs) is complete (no changes expected).
- **Actions**:
  1. Read lines 107-113 of `electrobun-setup.md`.
  2. Verify `--watch` is documented as working ("Add `--watch` for auto-rebuild on source changes").
  3. Verify there is no warning about race conditions or instability.
- **Expected outcome**: `--watch` is presented as a working feature with no caveats. Source of truth: ElectroBun commit `7a119ebe` fixed the `--watch` race condition in v1.14.5-beta.0; v1.15.1 includes this fix.
- **Interactions**: None.

### 13. Unmodified files (cli-runtime-reference.md, rpc-schema-reference.md) are untouched

- **Name**: Files not in the implementation plan's scope are not accidentally modified
- **Type**: regression
- **Harness**: Git diff
- **Preconditions**: All tasks complete.
- **Actions**:
  1. `git diff HEAD~N -- skills/loom-desktop/references/cli-runtime-reference.md` (where N covers all implementation commits).
  2. `git diff HEAD~N -- skills/loom-desktop/references/rpc-schema-reference.md`.
- **Expected outcome**: Both diffs are empty. These files are not in scope for this change. Source of truth: The implementation plan explicitly lists only SKILL.md and electrobun-setup.md as modified files.
- **Interactions**: If any task accidentally edits these files, it indicates scope creep.

### 14. Internal consistency: `--session-id` + `--resume` pattern matches across SKILL.md and cli-runtime-reference.md

- **Name**: Session management patterns in SKILL.md are consistent with the CLI reference
- **Type**: integration
- **Harness**: Manual cross-reference
- **Preconditions**: Tasks 6 and 7 are complete.
- **Actions**:
  1. Read Pattern 2's `spawnClaude` args construction (SKILL.md, around line 556-562). Note the session logic: `--session-id` on first turn, `--resume` on follow-up.
  2. Read Pattern 3's note (SKILL.md). Verify it describes the same logic.
  3. Read cli-runtime-reference.md Session Management section (lines 180-216). Verify it documents: `--session-id` creates, `--resume` continues, combining `--session-id` + `--continue`/`--resume` errors without `--fork-session`.
  4. Verify no contradiction between the two files.
- **Expected outcome**: Both files agree that `--session-id` is for creating sessions and `--resume <id>` is for continuing them. The cli-runtime-reference correctly notes that combining `--session-id` with `--continue`/`--resume` requires `--fork-session`. Source of truth: cli-runtime-reference.md (the canonical CLI flag reference).
- **Interactions**: Cross-file consistency check between SKILL.md and cli-runtime-reference.md.

### 15. The CLAUDE_BIN usage note exists after the Shared Utilities section

- **Name**: A prominent note tells users to use CLAUDE_BIN everywhere
- **Type**: boundary
- **Harness**: Manual review
- **Preconditions**: Task 6, Step 4 is complete.
- **Actions**:
  1. Read SKILL.md after the `const CLAUDE_BIN = resolveClaudePath();` line.
  2. Verify a note exists that says to use `CLAUDE_BIN` in every `Bun.spawn()` and `Bun.spawnSync()` call.
- **Expected outcome**: The note is present immediately after the CLAUDE_BIN definition. Source of truth: The skill's own architecture — `resolveClaudePath()` exists specifically because macOS GUI apps don't inherit shell PATH, and its purpose is defeated if any spawn call uses the string `"claude"` instead.
- **Interactions**: None.

---

## Coverage Summary

### Covered

- **Version references**: All `v1.14.x` strings (Tests 1, 9)
- **Import paths**: `electrobun/config` → `electrobun` (Test 2)
- **Template table**: 4 → 19 templates, stale names removed (Test 3)
- **Config format**: `copy` array → object at build level (Test 4)
- **CLAUDE_BIN consistency**: All spawn calls use the variable (Tests 5, 15)
- **Session flags**: `--continue` → `--resume` in Pattern 3 (Tests 6, 14)
- **End-to-end user walkthrough**: Full skill read-through for coherence (Test 7)
- **New API documentation**: Five new APIs + six config options (Test 8)
- **Gotcha updates**: Version status, crash recovery (Tests 9, 10)
- **Checklist accuracy**: resolveClaudePath added, deriveAndSendRPC removed (Test 11)
- **--watch documentation**: Confirmed accurate (Test 12)
- **Regression protection**: Unmodified files untouched (Test 13)
- **Cross-file consistency**: Session patterns match CLI reference (Test 14)

### Explicitly excluded per agreed strategy

- **Compile-testing every code example**: The code examples are embedded in Markdown, not in a runnable project. Compiling them would require scaffolding a full ElectroBun project and is beyond Medium fidelity. **Risk**: A code example could have a syntax error or type mismatch that text search won't catch. Mitigated by: the code examples are copied or adapted from working patterns, and the changes in this plan are targeted edits (variable names, flag names), not wholesale rewrites.
- **Low-level WGPU FFI API documentation**: The strategy explicitly excluded exhaustive WGPU docs. New WGPU-related templates are listed in the table, but we don't document the WGPU API surface. **Risk**: A user building a WGPU app gets less guidance. Mitigated by: the templates themselves contain working code, and the skill points to the ElectroBun docs for API details.
- **Cross-referencing every ElectroBun API name against source**: We verify the new APIs mentioned (GlobalShortcut, Screen, etc.) but don't audit every existing API name already in the skill. **Risk**: An existing API reference could be stale from a prior version. Mitigated by: the existing skill was written against v1.14.4, and no existing APIs were removed or renamed in v1.15.1 (additive release).
- **Runtime testing of the skill in actual Claude sessions**: The skill is a prompt document, not executable code. Testing it means having Claude use it to build a real app. This is integration testing at the product level and is out of scope for this plan. **Risk**: The skill could guide Claude incorrectly despite being factually correct (e.g., ordering issues, ambiguous instructions). Mitigated by: the scenario test (Test 7) does a manual walkthrough of the user path.
