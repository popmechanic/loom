# Test Plan: Audit Auth Guidance for Login with Anthropic

## Strategy Reconciliation

The agreed strategy -- medium-fidelity structural analysis with grep/search-based verification, no runtime `claude -p` testing -- holds as-is. The implementation plan targets only markdown documentation files (SKILL.md, oauth-reference.md), and all changes are textual patterns that can be verified by searching for the presence or absence of specific strings. No strategy changes are needed.

**Harness:** All tests use grep/ripgrep searches against the edited files. The observation surface is the text content of:
- `/Users/marcusestes/Websites/loom/.worktrees/audit-auth-guidance/skills/loom/SKILL.md`
- `/Users/marcusestes/Websites/loom/.worktrees/audit-auth-guidance/skills/loom/references/oauth-reference.md`

**Source of truth:** The implementation plan at `docs/plans/2026-03-08-audit-auth-guidance.md`, the oauth-reference.md (defines the canonical auth functions), and the neko-chat reference implementation (demonstrates the correct 401 round-trip pattern).

---

## Test Plan

### 1. All spawn sites use spawnEnvForUser(), not cleanEnv()

- **Name:** Every Claude spawn site injects the user's OAuth token via spawnEnvForUser()
- **Type:** invariant
- **Harness:** grep search
- **Preconditions:** SKILL.md has been edited per Tasks 1a-1i
- **Actions:**
  1. Search SKILL.md for `env: cleanEnv()` -- should return zero matches
  2. Search SKILL.md for `env: spawnEnvForUser(` -- should return 7 matches (REST, SSE, WebSocket, Background Job, Parallel Analysis, Structured Extraction, Persistent Session)
- **Expected outcome:** Zero `env: cleanEnv()` at spawn sites (source: plan Task 1, verification section). Seven `env: spawnEnvForUser(` matches (source: plan Tasks 1c through 1i enumerate exactly 7 spawn sites).
- **Interactions:** None -- pure text search.

### 2. Every route handler that spawns Claude uses requireAuth middleware

- **Name:** Protected routes enforce authentication before spawning Claude
- **Type:** invariant
- **Harness:** grep search
- **Preconditions:** SKILL.md has been edited per Tasks 1c-1g
- **Actions:**
  1. Search SKILL.md for lines matching `app.post.*requireAuth` -- should find all route patterns (analyze, stream, jobs, batch)
  2. Verify the WebSocket pattern authenticates via the `server.on("upgrade")` handler instead (Express middleware doesn't run on WebSocket handshakes)
- **Expected outcome:** 4 `app.post` routes with `requireAuth` (REST, SSE, Background Job, Parallel Analysis). WebSocket uses `server.on("upgrade")` with cookie validation (source: plan Task 1e).
- **Interactions:** The WebSocket pattern diverges from the Express middleware approach -- this is intentional and documented in the plan.

### 3. refreshSessionIfNeeded() is called before every spawn

- **Name:** Token refresh runs before every Claude subprocess to prevent mid-session auth failures
- **Type:** invariant
- **Harness:** grep search
- **Preconditions:** SKILL.md has been edited per Tasks 1c-1g
- **Actions:**
  1. Search SKILL.md for `refreshSessionIfNeeded` in code blocks (not documentation text) -- should appear before each spawn
  2. Count handler-level occurrences: REST, SSE, WebSocket, Background Job, Parallel Analysis = 5
- **Expected outcome:** 5 `await refreshSessionIfNeeded(` calls in route handlers/patterns (source: plan Tasks 1c, 1d, 1e, 1f, 1g). Structured Extraction and Persistent Session patterns are called from within handlers that already refreshed, so they may not have their own refresh -- this is acceptable.
- **Interactions:** The refresh function is defined in oauth-reference.md, not SKILL.md -- cross-file dependency.

### 4. spawnEnvForUser() utility is documented as third shared helper

- **Name:** Shared Utilities section documents three helpers, with spawnEnvForUser() as the third
- **Type:** scenario
- **Harness:** grep search
- **Preconditions:** SKILL.md has been edited per Tasks 1a and 6
- **Actions:**
  1. Search for "these three helpers" in SKILL.md -- should exist exactly once
  2. Search for "these two helpers" -- should exist zero times
  3. Search for the `spawnEnvForUser()` function definition block (with `function spawnEnvForUser`)
  4. Verify `cleanEnv()` is marked as internal ("do not call directly at spawn sites")
- **Expected outcome:** "three helpers" appears once (source: plan Task 6). `spawnEnvForUser` function definition exists in Shared Utilities section (source: plan Task 1a). `cleanEnv()` description includes internal warning (source: plan Task 6).
- **Interactions:** None.

### 5. cleanEnv() never appears in advisory/recommended positions

- **Name:** cleanEnv() is not recommended as the function to use -- only spawnEnvForUser() is
- **Type:** invariant
- **Harness:** grep search with context analysis
- **Preconditions:** SKILL.md has been edited per all tasks
- **Actions:**
  1. Search SKILL.md for all occurrences of `cleanEnv`
  2. Classify each: function definition, internal reference within spawnEnvForUser, or advisory text
  3. Verify no occurrence recommends cleanEnv() for direct use at spawn sites
- **Expected outcome:** `cleanEnv` appears only in: its own function definition, the "do not call directly" description, inside `spawnEnvForUser()` definition, the "Don't do this" bullet (explaining spawnEnvForUser calls it internally), and the checklist item (warning that bare cleanEnv is wrong). Source: plan Tasks 1c-ii, 5a, 6.
- **Interactions:** None.

### 6. cookie-parser dependency is noted at point of use

- **Name:** The cookie-parser import includes an npm install comment
- **Type:** boundary
- **Harness:** grep search
- **Preconditions:** SKILL.md has been edited per Task 2
- **Actions:**
  1. Search SKILL.md for `cookie-parser.*npm i` or `npm i cookie-parser`
- **Expected outcome:** The import line reads `import cookieParser from "cookie-parser";  // npm i cookie-parser @types/cookie-parser` (source: plan Task 2).
- **Interactions:** None.

### 7. Profile fetch guidance is strengthened with emphasis and "most commonly omitted" callout

- **Name:** Authentication Setup item #3 emphasizes the profile fetch as the most commonly omitted step
- **Type:** scenario
- **Harness:** grep search
- **Preconditions:** SKILL.md has been edited per Task 3
- **Actions:**
  1. Search SKILL.md for "most commonly omitted"
  2. Search for "**After token exchange, fetch the user's profile**" (bold emphasis)
  3. Search for "Bearer token" in the same context
- **Expected outcome:** Item #3 in Authentication Setup is bold-emphasized and includes "most commonly omitted step" callout (source: plan Task 3).
- **Interactions:** None.

### 8. oauth-reference.md has complete 401 round-trip documentation

- **Name:** The 401 Handling section documents the full server-restart-to-recovery cycle
- **Type:** scenario
- **Harness:** grep search
- **Preconditions:** oauth-reference.md has been edited per Task 4
- **Actions:**
  1. Search for "Session Expiry Round-Trip" heading
  2. Search for "The Full Cycle" subsection
  3. Search for "Frontend Guard" subsection with code example
  4. Search for "Key Implementation Details" subsection
  5. Verify the old "showSetupScreen()" one-liner is replaced with the expanded initSetup() pattern
  6. Verify "CSS Specificity Note" section still exists after the 401 section
- **Expected outcome:** The section includes the ASCII flow diagram, the expanded frontend guard code block (with initSetup(), hide/show logic), and the Key Implementation Details bullets. The CSS Specificity Note section is preserved. Source: plan Task 4, neko-chat reference (initSetup pattern).
- **Interactions:** The frontend streaming text display in SKILL.md still references `showSetupScreen()` -- this is intentional (SKILL.md shows the minimal pattern; oauth-reference.md shows the complete implementation).

### 9. First-Run Reliability Checklist has correct auth items

- **Name:** Checklist items cover all auth-specific silent failure modes without contradictions
- **Type:** integration
- **Harness:** grep search
- **Preconditions:** SKILL.md has been edited per Task 5
- **Actions:**
  1. Search for `spawnEnvForUser()` in checklist items -- should exist
  2. Search for "cleanEnv()" in checklist items -- should appear only as a warning ("bare cleanEnv() omits the token")
  3. Search for "cookie-parser" in checklist items -- should exist
  4. Search for "Profile fetch" in checklist items -- should exist
  5. Search for "health.*requireAuth" or "health.*NOT.*requireAuth" in checklist items -- should exist
  6. Search for "WebSocket upgrade" in checklist items -- should exist
  7. Count total checklist items (lines matching `^- \[ \]`) -- should be 11 (7 original + 4 new, with 1 replaced)
- **Expected outcome:** 11 checklist items total. The old `cleanEnv()` item is replaced by `spawnEnvForUser()` item. Four new items added (cookie-parser, profile fetch, health endpoint, WebSocket upgrade). Source: plan Tasks 5a, 5b.
- **Interactions:** The checklist items reference patterns and functions defined elsewhere in SKILL.md and oauth-reference.md.

### 10. WebSocket pattern uses noServer mode with upgrade authentication

- **Name:** WebSocket pattern authenticates via HTTP upgrade, not standalone port
- **Type:** integration
- **Harness:** grep search
- **Preconditions:** SKILL.md has been edited per Task 1e
- **Actions:**
  1. Search for `noServer: true` in the WebSocket pattern
  2. Search for `server.on("upgrade"` in the WebSocket pattern
  3. Search for `parseCookie` import
  4. Verify the old `port: 8080` standalone server is gone
  5. Search for `ws.on("message", async` (was synchronous, now async for refreshSessionIfNeeded)
- **Expected outcome:** WebSocket pattern uses `{ noServer: true }`, authenticates during upgrade handshake via cookie parsing, rejects unauthenticated connections with 401, and attaches `userSession` to the ws object. Source: plan Task 1e.
- **Interactions:** Depends on the `cookie` npm package (not `cookie-parser`). The `parse as parseCookie` import is a different package than the Express middleware.

### 11. Server Setup block includes auth infrastructure comment

- **Name:** Server Setup code block documents the auth functions that need to be implemented
- **Type:** boundary
- **Harness:** grep search
- **Preconditions:** SKILL.md has been edited per Task 1b
- **Actions:**
  1. Search for "Auth infrastructure" comment in SKILL.md
  2. Verify it mentions requireAuth, refreshSessionIfNeeded, spawnEnvForUser, and the 4 OAuth endpoints
- **Expected outcome:** A comment block after `express.static` lists all auth components and references oauth-reference.md. Source: plan Task 1b.
- **Interactions:** None.

---

## Coverage Summary

### Covered

- **All 7 spawn sites** verified to use `spawnEnvForUser()` instead of `cleanEnv()` (Test 1)
- **All 4 Express route patterns** verified to use `requireAuth` middleware (Test 2)
- **WebSocket auth** verified to use upgrade-time cookie validation (Tests 2, 10)
- **Token refresh** verified before every spawn (Test 3)
- **Utility documentation** verified for three helpers, internal marking (Tests 4, 5)
- **Dependency visibility** verified for cookie-parser (Test 6)
- **Profile fetch emphasis** verified (Test 7)
- **401 round-trip documentation** verified as complete (Test 8)
- **Checklist completeness** verified with correct count and no contradictions (Test 9)
- **Server Setup auth comment** verified (Test 11)

### Explicitly Excluded (per agreed strategy)

- **Runtime testing of `claude -p`**: Cannot spawn Claude processes from within a subagent (nesting guard). Risk: the documented patterns could have TypeScript syntax errors or runtime behavior differences. Mitigated by the fact that all patterns are consistent with the working neko-chat reference implementation.
- **Frontend rendering verification**: No browser testing. The 401 guard code and setup screen re-initialization are documented but not tested in a real browser. Risk: low -- the code patterns match the working neko-chat implementation.
- **Cross-file link integrity**: We verify that SKILL.md references oauth-reference.md, but we don't verify that all function names match exactly between files. Risk: minimal -- both files were edited in the same session with the same function names.
