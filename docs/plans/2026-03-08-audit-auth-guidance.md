# Audit Auth Guidance for Login with Anthropic

> **For Claude:** REQUIRED SUB-SKILL: Use trycycle-executing to implement this plan task-by-task.

## Goal

Improve the reliability of the "Login with Anthropic" feature in the Loom skill so that Claude following these patterns consistently produces a working authentication interface. The core problem: SKILL.md's code patterns use `cleanEnv()` (no token) while the oauth-reference.md defines `spawnEnvForUser()` (with token), creating a disconnect that causes silent auth failures.

## Problem Analysis

The Loom skill's auth guidance is split across three files (SKILL.md, oauth-reference.md, cli-runtime-reference.md). The reliability failures are:

1. **`cleanEnv()` vs `spawnEnvForUser()` confusion** -- SKILL.md's code patterns all use `cleanEnv()` directly, which produces an env without `CLAUDE_CODE_OAUTH_TOKEN`. The oauth-reference.md defines `spawnEnvForUser()` which calls `cleanEnv()` then injects the token. But the SKILL.md patterns never use `spawnEnvForUser()` -- they use `cleanEnv()` bare. Claude following SKILL.md's patterns will produce servers where auth tokens are never injected.

2. **Scattered auth guidance across 3 files** -- SKILL.md's "Authentication Setup" section says "see oauth-reference.md" but the code patterns use `cleanEnv()` without auth integration. A reader following SKILL.md's SSE/WebSocket/REST patterns gets a working server structure but with no auth wiring.

3. **Cookie parser dependency implicit** -- SKILL.md's "Server Setup" block shows `app.use(cookieParser())` but never mentions `cookie-parser` as a required npm dependency. The package.json mention in "What to Generate" is too far away.

4. **Profile fetch after token exchange sometimes omitted** -- The profile fetch is buried in bullet point #3 of the "Authentication Setup" section. Claude sometimes skips it, leaving `session.profile` undefined.

5. **401 bidirectional flow not well-documented** -- The oauth-reference.md has a "401 Handling" section but it doesn't describe the complete round-trip: server restart, stale cookie, 401, setup screen re-initialization, re-auth, return to app. The neko-chat implementation does this correctly.

## Worktree

All edits target: `/Users/marcusestes/Websites/loom/.worktrees/audit-auth-guidance/`

## Changed Files

- `skills/loom/SKILL.md`
- `skills/loom/references/oauth-reference.md`

## Tasks

### Task 1: Add `spawnEnvForUser()` as a shared utility and replace all `cleanEnv()` calls in SKILL.md patterns

**Why:** Every server pattern in SKILL.md currently shows `env: cleanEnv()` -- this produces an environment WITHOUT the user's OAuth token. The auth token is what makes `claude -p` work with a user's Anthropic subscription. The fix is to add `spawnEnvForUser()` as a third shared utility (alongside `cleanEnv()` and `createStreamParser()`) and then update every spawn site to use it unconditionally.

**Files:** `skills/loom/SKILL.md`

**Changes:**

**1a.** In the **Shared Utilities** section, change the intro text from "Every pattern below uses these two helpers" to "Every pattern below uses these three helpers". Then after the `createStreamParser()` code block and its `TextDecoder` explanation paragraph, add:

Old text:
```
#### Server Setup
```

New text (insert before `#### Server Setup`):
```
**`spawnEnvForUser()`** -- Inject the user's OAuth token into the spawn environment.

Every spawn needs the user's `CLAUDE_CODE_OAUTH_TOKEN` in the environment --
without it, `claude -p` starts but can't authenticate with Anthropic's servers.
This wraps `cleanEnv()` with the token injection. See
`references/oauth-reference.md` for the full session store, `requireAuth`
middleware, and `refreshSessionIfNeeded()` that provide `req.userSession`.

```typescript
function spawnEnvForUser(session: UserSession): NodeJS.ProcessEnv {
  const env = cleanEnv();
  env.CLAUDE_CODE_OAUTH_TOKEN = session.accessToken;
  return env;
}
```

#### Server Setup
```

Do NOT include any "local-only" caveat or conditional language suggesting `cleanEnv()` is an alternative for some apps. `spawnEnvForUser()` is always the right function to use at spawn sites.

**1b.** In the **Server Setup** section code block, add a comment after `app.use(cookieParser());`:

Old:
```typescript
app.use(express.static(path.join(__dirname, "public")));
```

New:
```typescript
app.use(express.static(path.join(__dirname, "public")));

// Auth infrastructure -- see references/oauth-reference.md for implementations:
// - requireAuth middleware (validates session cookie on protected routes)
// - refreshSessionIfNeeded() (call before every spawn)
// - spawnEnvForUser() (injects CLAUDE_CODE_OAUTH_TOKEN into spawn env)
// - OAuth endpoints: /api/oauth/start, /api/oauth/exchange, /api/health, /api/logout
```

**1c.** In the **REST Endpoint** pattern, change:

Old:
```typescript
app.post("/api/analyze", (req, res) => {
```

New:
```typescript
app.post("/api/analyze", requireAuth, async (req: any, res) => {
  await refreshSessionIfNeeded(req.userSession);
```

And change:
```typescript
    ], { input: `${task}\n\n${content}`, encoding: "utf-8", timeout: 60000, env: cleanEnv() });
```

To:
```typescript
    ], { input: `${task}\n\n${content}`, encoding: "utf-8", timeout: 60000, env: spawnEnvForUser(req.userSession) });
```

**1c-ii.** Also in the REST pattern's **"Don't do this"** block, update the env cleanup bullet:

Old:
```
- Don't skip env cleanup — always use `cleanEnv()` before spawning.
  Do NOT strip all `CLAUDE_*` vars; some are required for auth.
```

New:
```
- Don't skip env setup — always use `spawnEnvForUser()` before spawning.
  This calls `cleanEnv()` internally (removing nesting guards) and injects
  the user's OAuth token. Do NOT strip all `CLAUDE_*` vars.
```

**1d.** In the **SSE Streaming** pattern, change:

Old:
```typescript
app.post("/api/stream", (req, res) => {
  const { task } = req.body;
```

New:
```typescript
app.post("/api/stream", requireAuth, async (req: any, res) => {
  await refreshSessionIfNeeded(req.userSession);
  const { task } = req.body;
```

And change:
```typescript
  ], { env: cleanEnv() });
```

To:
```typescript
  ], { env: spawnEnvForUser(req.userSession) });
```

**1e.** In the **WebSocket Session** pattern, this requires a different approach because WebSocket upgrades bypass Express middleware. Replace the pattern's opening to validate auth during the upgrade handshake:

Old:
```typescript
import { WebSocketServer } from "ws";
import { spawn, ChildProcess } from "child_process";
import { v4 as uuidv4 } from "uuid";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws) => {
  const sessionId = uuidv4();
  let activeProc: ChildProcess | null = null;
  let isFirstTurn = true;
```

New:
```typescript
import { WebSocketServer } from "ws";
import { spawn, ChildProcess } from "child_process";
import { v4 as uuidv4 } from "uuid";
import { parse as parseCookie } from "cookie";

// Attach to the Express HTTP server, not a standalone port
const wss = new WebSocketServer({ noServer: true });

// Authenticate on upgrade — Express middleware doesn't run on WebSocket handshakes
server.on("upgrade", (req, socket, head) => {
  const cookies = parseCookie(req.headers.cookie || "");
  const sessionId = cookies[SESSION_COOKIE_NAME];
  const session = sessionId ? sessions.get(sessionId) : null;
  if (!session) {
    socket.write("HTTP/1.1 401 Unauthorized\r\n\r\n");
    socket.destroy();
    return;
  }
  wss.handleUpgrade(req, socket, head, (ws) => {
    (ws as any).userSession = session;
    wss.emit("connection", ws, req);
  });
});

wss.on("connection", (ws: any) => {
  const userSession: UserSession = ws.userSession;
  const sessionId = uuidv4();
  let activeProc: ChildProcess | null = null;
  let isFirstTurn = true;
```

And change the spawn env:

Old:
```typescript
    ], { env: cleanEnv() });
```

New:
```typescript
    ], { env: spawnEnvForUser(userSession) });
```

Also add a refresh call before the spawn:

Old:
```typescript
    const proc = spawn("claude", [
```

New:
```typescript
    await refreshSessionIfNeeded(userSession);

    const proc = spawn("claude", [
```

And change the `ws.on("message")` handler from synchronous to async:

Old:
```typescript
  ws.on("message", (raw) => {
```

New:
```typescript
  ws.on("message", async (raw) => {
```

**1f.** In the **Background Job** pattern, change:

Old:
```typescript
app.post("/api/jobs", (req, res) => {
```

New:
```typescript
app.post("/api/jobs", requireAuth, async (req: any, res) => {
  await refreshSessionIfNeeded(req.userSession);
```

And change:
```typescript
  ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv() });
```

To:
```typescript
  ], { stdio: ["pipe", "pipe", "pipe"], env: spawnEnvForUser(req.userSession) });
```

**1g.** In the **Parallel Analysis** pattern, change:

Old:
```typescript
app.post("/api/batch", async (req, res) => {
```

New:
```typescript
app.post("/api/batch", requireAuth, async (req: any, res) => {
  await refreshSessionIfNeeded(req.userSession);
```

And change:
```typescript
      ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv() });
```

To:
```typescript
      ], { stdio: ["pipe", "pipe", "pipe"], env: spawnEnvForUser(req.userSession) });
```

**1h.** In the **Structured Extraction** async helper function, add a `session` parameter:

Old:
```typescript
async function extract<T>(prompt: string, schema: object, timeoutMs = 30000): Promise<T> {
```

New:
```typescript
async function extract<T>(prompt: string, schema: object, session: UserSession, timeoutMs = 30000): Promise<T> {
```

And change:
```typescript
  ], { stdio: ["pipe", "pipe", "pipe"], env: cleanEnv() });
```

To:
```typescript
  ], { stdio: ["pipe", "pipe", "pipe"], env: spawnEnvForUser(session) });
```

**1i.** In the **Persistent Session** pattern, change:

Old:
```typescript
  proc = spawn("claude", args, {
    stdio: ["pipe", "pipe", "pipe"],
    env: cleanEnv(),
  });
```

New:
```typescript
  proc = spawn("claude", args, {
    stdio: ["pipe", "pipe", "pipe"],
    env: spawnEnvForUser(session),
  });
```

And update the `startSession` function signature to accept a session:

Old:
```typescript
function startSession(systemPrompt?: string) {
```

New:
```typescript
function startSession(session: UserSession, systemPrompt?: string) {
```

**Verification:** Search SKILL.md for `env: cleanEnv()` -- should appear ONLY inside the `cleanEnv()` function definition itself. Every spawn site should use `spawnEnvForUser()`. Also search for `env: spawnEnvForUser` to confirm all spawn sites are updated.

### Task 2: Add cookie-parser dependency note at point of use in SKILL.md

**Why:** The Server Setup block shows `app.use(cookieParser())` but a reader building incrementally may not realize this requires `npm i cookie-parser`.

**Files:** `skills/loom/SKILL.md`

**Changes:**

In the **Server Setup** code block, change:

Old:
```typescript
import cookieParser from "cookie-parser";
```

New:
```typescript
import cookieParser from "cookie-parser";  // npm i cookie-parser @types/cookie-parser
```

**Verification:** The dependency is now visible at point of use.

### Task 3: Strengthen profile fetch guidance in SKILL.md

**Why:** The profile fetch is critical for the frontend to show who's logged in, but it's buried in bullet point #3 of the "Authentication Setup" section. Claude sometimes skips it, leaving `session.profile` undefined.

**Files:** `skills/loom/SKILL.md`

**Changes:**

In the **Authentication Setup** section, change item #3 from:

Old:
```
3. After token exchange, fetch the user's profile from `https://api.anthropic.com/v1/me`
   and store it in the session — the frontend needs this to show who's logged in
```

New:
```
3. **After token exchange, fetch the user's profile** from `https://api.anthropic.com/v1/me`
   using the new `access_token` as a Bearer token, and store `{name, email}` in the
   session. Without this, the frontend can't show who's logged in -- it falls back to
   a generic "CONNECTED" label. This is the most commonly omitted step.
```

**Verification:** The emphasis and "most commonly omitted" callout makes it harder to skip.

### Task 4: Replace 401 Handling section in oauth-reference.md with complete round-trip documentation

**Why:** The current "401 Handling" section shows only the frontend guard snippet. It doesn't describe the complete cycle: server restart, stale cookie, 401 on protected endpoint, setup screen re-initialization, re-auth, return to app. The neko-chat implementation does this correctly (its `send()` function checks `response.status === 401`, hides the main UI, shows the setup screen, and calls `initSetup()`).

**Files:** `skills/loom/references/oauth-reference.md`

**Changes:**

Replace the entire section from `## 401 Handling` through the end of the "without this guard" paragraph (just before `## CSS Specificity Note`). The old section starts with "## 401 Handling" and ends with "which fails silently -- the user sees nothing happen."

Replace with the following content (note: the ASCII diagram uses indented-text formatting, not fenced code blocks, to avoid nested-fence rendering issues):

Old text to find and replace (starts at "## 401 Handling" and goes through "which fails silently --- the user sees nothing happen."):

New replacement:

    ## 401 Handling: Session Expiry Round-Trip

    In-memory sessions don't survive server restarts. When the server redeploys
    or crashes, every user's session is gone. Their browser still has the
    `loom_session` cookie, but the server returns 401 for any authenticated
    endpoint.

    ### The Full Cycle

        Server restart --> sessions wiped
                |
        Browser sends request with stale cookie
                |
        requireAuth returns 401  <-- also triggers on session TTL expiry
                |
        Frontend catches 401 BEFORE parsing response body
                |
        Frontend hides main app UI, shows setup screen
                |
        User re-authenticates via OAuth flow
                |
        New session created, new cookie set
                |
        Setup screen calls onComplete / reloads
                |
        Main app UI restored with fresh session

    ### Frontend Guard (Required on Every Protected Fetch)

    Every `fetch()` to a protected endpoint must check for 401 **before**
    attempting to parse the response as SSE, JSON, or any other format.
    A 401 response body is JSON (`{"error":"Not authenticated"}`) -- if
    this reaches the SSE parser, it fails silently and the user sees nothing.

    ```javascript
    const response = await fetch("/api/chat", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ message }),
    });

    if (response.status === 401) {
      // Session expired or server restarted -- show setup screen
      // Hide the main app UI and re-initialize the setup flow
      document.getElementById('setup-screen').style.display = 'flex';
      document.querySelector('.main-app').style.display = 'none';
      initSetup();  // Re-initialize OAuth event listeners
      return;
    }

    // Only now parse as SSE stream
    const reader = response.body.getReader();
    ```

    ### Key Implementation Details

    - **Re-initialize the setup screen**, don't just show/hide it. The OAuth
      state variables (PKCE state, code input value) need to be fresh. Call
      `initSetup()` or remount the `<SetupScreen>` component.
    - **Clear any in-progress UI state** (streaming indicators, disabled inputs)
      before showing the setup screen.
    - **After re-auth, restore the app view** -- the setup screen's `onComplete`
      callback should hide the setup screen and show the main app, then focus
      the primary input.
    - **The `/api/health` endpoint does NOT require auth** -- it checks for a
      session cookie but returns `{needsSetup: true}` instead of 401 when
      the session is missing. This is intentional: the health check is used
      for initial page load to determine which screen to show.

**Verification:** The section now describes the complete server-restart-to-recovery cycle, includes the code guard, and explains setup screen re-initialization. The ASCII diagram uses indented code block (4-space indent) to avoid nested code fence issues.

### Task 5: Update First-Run Reliability Checklist for auth

**Why:** The existing checklist has one item that says "`cleanEnv()` is called on every `spawn`/`execFileSync`" -- this now contradicts the patterns (which use `spawnEnvForUser()`). That item must be replaced, not just supplemented. Additionally, new auth-specific checklist items are needed.

**Files:** `skills/loom/SKILL.md`

**Changes:**

**5a.** Replace the existing `cleanEnv()` checklist item:

Old:
```
- [ ] `cleanEnv()` is called on every `spawn`/`execFileSync` — without it, Claude processes fail silently when the server runs inside Claude Code
```

New:
```
- [ ] `spawnEnvForUser()` is called on every `spawn`/`execFileSync` — this removes nesting guards AND injects the user's OAuth token; bare `cleanEnv()` omits the token and causes silent auth failure
```

**5b.** After the existing list (last item is about "401 responses in the frontend redirect to the setup screen"), add these new items:

```markdown
- [ ] `cookie-parser` middleware is applied before any route that reads `req.cookies` -- without it, `requireAuth` sees `undefined` and every request returns 401
- [ ] Profile fetch happens in the `/api/oauth/exchange` handler after token exchange -- without it, the frontend shows "CONNECTED" instead of the user's email
- [ ] `/api/health` does NOT use `requireAuth` -- it must return `{needsSetup: true}` for unauthenticated users, not 401
- [ ] WebSocket upgrade validates the session cookie from `req.headers.cookie` -- Express middleware does not run on WebSocket handshakes
```

**Verification:** The old contradictory `cleanEnv()` item is gone. The replacement and new items cover the auth-specific silent failure modes without internal contradictions.

### Task 6: Update the Shared Utilities intro count and cleanEnv description

**Why:** After adding `spawnEnvForUser()` as a third utility, the intro text that says "two helpers" needs to say "three helpers." Also, the `cleanEnv()` description should note that it is an internal building block -- not for direct use at spawn sites -- to eliminate any remaining ambiguity.

**Files:** `skills/loom/SKILL.md`

**Changes:**

Change:
```
Every pattern below uses these two helpers. Define them once at the top of
your server file.
```

To:
```
Every pattern below uses these three helpers. Define them once at the top of
your server file.
```

And in the `cleanEnv()` description, change:

Old:
```
**`cleanEnv()`** -- Remove nesting guards so `claude -p` can start.
```

New:
```
**`cleanEnv()`** -- Remove nesting guards so `claude -p` can start. Used internally by `spawnEnvForUser()` below -- do not call directly at spawn sites.
```

**Verification:** The utility count matches and `cleanEnv()` is clearly marked as internal.

## Execution Notes

- All changes are markdown edits -- no runtime code to test.
- The neko-chat implementation in the main repo should NOT be modified -- it is the working reference.
- Do NOT attempt to run `claude -p` or spawn any Claude processes. This is a documentation-only audit.
- After editing, verify consistency by searching SKILL.md:
  - `env: cleanEnv()` should appear ZERO times outside of function definitions
  - `env: spawnEnvForUser(` should appear at every spawn site
  - `requireAuth` should appear on every route handler that spawns Claude
  - `refreshSessionIfNeeded` should appear before every spawn call
  - The string `cleanEnv()` should NOT appear in any "Don't do this" bullet or checklist item as the recommended function -- only `spawnEnvForUser()` should appear in those advisory positions
