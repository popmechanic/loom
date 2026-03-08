# Anthropic OAuth Reference

Loom apps need valid Anthropic credentials before the Claude CLI can function.
This reference implements **"Bring Your Own Claude"** — users authenticate with
their own Anthropic account through a one-time OAuth setup. No API keys to
manage, no cost on your side. Users bring their own Claude subscription.

## Table of Contents

1. [User Flow](#user-flow)
2. [Constants](#constants)
3. [PKCE Utilities](#pkce-utilities)
4. [Server Endpoints](#server-endpoints)
5. [Token Lifecycle (Session-Based)](#token-lifecycle-session-based)
6. [Integration Notes](#integration-notes)
7. [Multi-User Sessions](#multi-user-sessions)
8. [Frontend Setup Screen](#frontend-setup-screen)

---

## User Flow

The authentication is a four-step process:

1. User clicks **"Sign in with Anthropic"** in the setup screen
2. User authorizes the app on **claude.ai**
3. User copies the **authentication code** from the callback page
4. User pastes the code in the app, which exchanges it for tokens

```
┌──────────┐   GET /api/oauth/start   ┌──────────┐
│  Browser  │ ───────────────────────► │  Server  │
│  (Setup)  │ ◄─────────────────────── │          │
│           │   { authUrl, state }     │          │
└─────┬─────┘                          └──────────┘
      │
      │  Opens popup to authUrl
      ▼
┌──────────────────┐
│  claude.ai       │
│  /oauth/authorize│
│                  │
│  User authorizes │
│  and copies code │
└────────┬─────────┘
         │
         │  User pastes code
         ▼
┌──────────┐  POST /api/oauth/exchange  ┌──────────┐  POST token endpoint  ┌───────────────┐
│  Browser  │ ────────────────────────► │  Server  │ ──────────────────── │  Anthropic    │
│  (Setup)  │ ◄─────────────────────── │          │ ◄────────────────── │  OAuth        │
│           │   { ok: true }           │          │   { access_token,   │               │
└──────────┘                           │          │     refresh_token } │               │
                                       └─────┬────┘                     └───────────────┘
                                             │
                                             │  Stores tokens in session map
                                             │  + sets httpOnly cookie
                                             ▼
                                       In-memory session store
                                             │
                                             │  Server injects CLAUDE_CODE_OAUTH_TOKEN
                                             │  into claude -p env
                                             ▼
                                       claude -p starts successfully
```

### Why PKCE?

PKCE (Proof Key for Code Exchange, RFC 7636) prevents authorization code
interception attacks. The server generates a random `code_verifier` and
sends its SHA-256 hash (`code_challenge`) with the authorization request.
When exchanging the code for tokens, the server proves it initiated the
request by providing the original verifier. Anthropic's OAuth requires PKCE.

---

## Constants

```typescript
const OAUTH_CLIENT_ID = "9d1c250a-e61b-44d9-88ed-5944d1962f5e";
const OAUTH_AUTHORIZE_URL = "https://claude.ai/oauth/authorize";
const OAUTH_TOKEN_URL = "https://platform.claude.com/v1/oauth/token";
const OAUTH_REDIRECT_URI = "https://platform.claude.com/oauth/code/callback";
const OAUTH_SCOPES = "user:profile user:inference user:sessions:claude_code user:mcp_servers";
```

---

## PKCE Utilities

Pure Node.js crypto, no framework dependency:

```typescript
import { createHash, randomBytes } from "crypto";

function base64url(buf: Buffer): string {
  return buf.toString("base64").replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/g, "");
}

function generateCodeVerifier(): string {
  return base64url(randomBytes(32));
}

function generateCodeChallenge(verifier: string): string {
  return base64url(createHash("sha256").update(verifier).digest());
}
```

---

## Server Endpoints

### `GET /api/oauth/start`

Generates a PKCE challenge and returns an authorization URL for the frontend
to open in a popup.

- Generates random state (32 bytes hex) and PKCE verifier/challenge pair
- Stores `{state -> {verifier, createdAt}}` in an in-memory Map with 10-minute TTL
- Builds the authorization URL with all required parameters
- Returns `{authUrl, state}` to the frontend

```typescript
const pendingPKCE = new Map<string, { verifier: string; createdAt: number }>();
const PKCE_TTL_MS = 10 * 60 * 1000;

// Cleanup expired PKCE flows every 60s
setInterval(() => {
  const now = Date.now();
  for (const [key, val] of pendingPKCE) {
    if (now - val.createdAt > PKCE_TTL_MS) pendingPKCE.delete(key);
  }
}, 60_000);

app.get("/api/oauth/start", (req, res) => {
  const state = randomBytes(32).toString("hex");
  const verifier = generateCodeVerifier();
  const challenge = generateCodeChallenge(verifier);

  const authUrl = `${OAUTH_AUTHORIZE_URL}?` + new URLSearchParams({
    client_id: OAUTH_CLIENT_ID,
    code_challenge: challenge,
    code_challenge_method: "S256",
    redirect_uri: OAUTH_REDIRECT_URI,
    scope: OAUTH_SCOPES,
    state,
    response_type: "code",
  }).toString();

  pendingPKCE.set(state, { verifier, createdAt: Date.now() });
  res.json({ authUrl, state });
});
```

### `POST /api/oauth/exchange`

Exchanges an authorization code for tokens using the stored PKCE verifier,
then fetches the user's profile so the frontend can display their identity.

- Validates state exists in pendingPKCE map (returns 400 if expired/missing)
- Strips `#state` suffix from code (handles callback URL fragments)
- POSTs to Anthropic's token endpoint with the code, verifier, and other params
- Fetches user profile from `https://api.anthropic.com/v1/me` using the new access token
- On success, creates an in-memory session (with profile) and sets an httpOnly cookie
- Returns `{ok: true}` to the frontend

```typescript
app.post("/api/oauth/exchange", async (req, res) => {
  const { code, state } = req.body;
  if (!code || !state) return res.status(400).json({ error: "code and state required" });

  const pending = pendingPKCE.get(state);
  if (!pending) return res.status(400).json({ error: "Invalid or expired state" });
  pendingPKCE.delete(state);

  const cleanCode = code.split("#")[0]; // Strip callback fragment

  try {
    const resp = await fetch(OAUTH_TOKEN_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        grant_type: "authorization_code",
        code: cleanCode,
        redirect_uri: OAUTH_REDIRECT_URI,
        client_id: OAUTH_CLIENT_ID,
        code_verifier: pending.verifier,
        state,
      }),
    });

    if (!resp.ok) {
      const err = await resp.text();
      console.error(`[oauth] Token exchange failed (${resp.status}):`, err);
      return res.status(502).json({ error: `Token exchange failed (${resp.status}): ${err}` });
    }

    const tokens: any = await resp.json();

    // Fetch user profile so the frontend can show who's logged in
    let profile: { name?: string; email?: string } | undefined;
    try {
      const profileResp = await fetch("https://api.anthropic.com/v1/me", {
        headers: { Authorization: `Bearer ${tokens.access_token}` },
      });
      if (profileResp.ok) {
        const p: any = await profileResp.json();
        profile = { name: p.name, email: p.email };
      }
    } catch (e: any) {
      console.warn(`[oauth] Profile fetch error:`, e?.message);
    }

    // Create session instead of writing to disk
    const sessionId = randomUUID();
    sessions.set(sessionId, {
      accessToken: tokens.access_token,
      refreshToken: tokens.refresh_token,
      expiresAt: Date.now() + tokens.expires_in * 1000,
      createdAt: Date.now(),
      profile,
    });

    res.cookie(SESSION_COOKIE_NAME, sessionId, {
      httpOnly: true,
      sameSite: "lax",
      maxAge: SESSION_MAX_AGE_MS,
      path: "/",
    });
    res.json({ ok: true });
  } catch (err: any) {
    console.error(`[oauth] Exchange error:`, err?.message || err);
    res.status(500).json({ error: "Token exchange failed" });
  }
});
```

---

## Token Lifecycle (Session-Based)

### `requireAuth()` Middleware

Validates that requests include a valid session cookie. Attach to any
endpoint that needs authentication.

```typescript
function requireAuth(req: any, res: any, next: any) {
  const sessionId = req.cookies?.[SESSION_COOKIE_NAME];
  if (!sessionId) return res.status(401).json({ error: "Not authenticated" });
  const session = sessions.get(sessionId);
  if (!session) return res.status(401).json({ error: "Session expired" });
  req.userSession = session;
  next();
}
```

### `refreshSessionIfNeeded()`

Proactively refreshes the access token if it expires within 30 minutes.
Call before spawning Claude subprocesses to prevent mid-session auth failures.
Operates on the session object in memory — no file I/O.

**Important:** The refresh token rotates on every refresh — the new tokens
are stored directly in the session object.

```typescript
async function refreshSessionIfNeeded(session: UserSession): Promise<boolean> {
  const thirtyMinutes = 30 * 60 * 1000;
  if (session.expiresAt > Date.now() + thirtyMinutes) return false;

  try {
    const resp = await fetch(OAUTH_TOKEN_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        grant_type: "refresh_token",
        refresh_token: session.refreshToken,
        client_id: OAUTH_CLIENT_ID,
        scope: OAUTH_SCOPES,
      }),
    });
    if (!resp.ok) return false;

    const tokens: any = await resp.json();
    session.accessToken = tokens.access_token;
    session.refreshToken = tokens.refresh_token;
    session.expiresAt = Date.now() + tokens.expires_in * 1000;
    return true;
  } catch { return false; }
}
```

### `spawnEnvForUser()`

Injects the user's token into the environment for `claude -p`. Each user's
Claude process uses only their own Anthropic subscription.

```typescript
function spawnEnvForUser(session: UserSession): NodeJS.ProcessEnv {
  const env = cleanEnv();
  env.CLAUDE_CODE_OAUTH_TOKEN = session.accessToken;
  return env;
}
```

---

## Integration Notes

These functions fit into the server lifecycle at specific points:

- **`requireAuth` middleware** on protected endpoints — validates session cookie
- **`refreshSessionIfNeeded()`** before every `spawn("claude", ...)` call —
  prevents mid-session auth failures
- **`/api/health` endpoint** — checks session cookie to determine setup state
- **`/api/logout` endpoint** — destroys session and clears cookie

```typescript
app.get("/api/health", (req, res) => {
  const sessionId = req.cookies?.[SESSION_COOKIE_NAME];
  const session = sessionId ? sessions.get(sessionId) : null;
  res.json({ needsSetup: !session, user: session?.profile ?? null });
});

app.post("/api/logout", (req, res) => {
  const sessionId = req.cookies?.[SESSION_COOKIE_NAME];
  if (sessionId) sessions.delete(sessionId);
  res.clearCookie(SESSION_COOKIE_NAME);
  res.json({ ok: true });
});
```

---

## Multi-User Sessions

For apps with multiple users, tokens must not be written to a shared file.
Instead, store each user's tokens in a server-side session and inject them
into Claude processes via environment variable.

### Session Store

**Important:** Sessions live in memory and do not survive server restarts.
Every redeploy wipes all sessions, forcing users to re-authenticate. The
frontend must handle this gracefully — see "401 Handling" below.

```typescript
interface UserSession {
  accessToken: string;
  refreshToken: string;
  expiresAt: number;
  createdAt: number;
  profile?: { name?: string; email?: string };
}

const sessions = new Map<string, UserSession>();
const SESSION_MAX_AGE_MS = 24 * 60 * 60 * 1000;
const SESSION_COOKIE_NAME = "loom_session";

// Cleanup expired sessions every 5 minutes
setInterval(() => {
  const now = Date.now();
  for (const [id, session] of sessions) {
    if (now - session.createdAt > SESSION_MAX_AGE_MS) sessions.delete(id);
  }
}, 5 * 60 * 1000);
```

### Per-User Process Spawning

Inject the user's token via environment variable:

```typescript
function spawnEnvForUser(session: UserSession): NodeJS.ProcessEnv {
  const env = cleanEnv();
  env.CLAUDE_CODE_OAUTH_TOKEN = session.accessToken;
  return env;
}

// Usage in endpoint handler:
app.post("/api/chat", requireAuth, async (req: any, res) => {
  await refreshSessionIfNeeded(req.userSession);
  const proc = spawn("claude", [...args], {
    stdio: ["ignore", "pipe", "pipe"],
    env: spawnEnvForUser(req.userSession),
  });
  // ... handle streaming
});
```

---

## Frontend Setup Screen

### State Machine

```
States: idle → starting → idle (step 2) → exchanging → polling → complete
         ↓                   ↓                ↓
       error              error            error
```

State variables:

- `step`: 1 (authorize button) or 2 (paste code)
- `status`: `idle | starting | exchanging | polling | error`
- `state`: PKCE state string from `/api/oauth/start`
- `code`: authorization code pasted by user
- `error`: error message string

### `handleOAuthStart()`

Opens the Anthropic authorization page. The popup is opened **synchronously**
before any async work to avoid popup blockers.

```javascript
const handleOAuthStart = async () => {
  setStatus('starting');
  setError('');
  const popup = window.open('about:blank', '_blank');
  try {
    const res = await fetch('/api/oauth/start');
    const data = await res.json();
    if (!res.ok) { if (popup) popup.close(); setError(data.error); setStatus('error'); return; }
    setState(data.state);
    if (popup) popup.location.href = data.authUrl;
    else window.open(data.authUrl, '_blank');
    setStep(2);
    setStatus('idle');
  } catch (err) {
    if (popup) popup.close();
    setError(err.message);
    setStatus('error');
  }
};
```

### `handleOAuthExchange()`

Sends the authorization code to the server and polls `/api/health` until
credentials are confirmed.

```javascript
const handleOAuthExchange = async () => {
  if (!code.trim() || !state) return;
  setStatus('exchanging');
  setError('');
  try {
    const res = await fetch('/api/oauth/exchange', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ code: code.trim(), state }),
    });
    const data = await res.json();
    if (!res.ok) {
      if (data.error?.includes('expired')) { setStep(1); setCode(''); setError('Session expired. Start over.'); setStatus('idle'); }
      else { setError(data.error); setStatus('error'); }
      return;
    }
    setStatus('polling');
    for (let i = 0; i < 20; i++) {
      await new Promise(r => setTimeout(r, 1000));
      try {
        const h = await fetch('/api/health');
        const hd = await h.json();
        if (!hd.needsSetup) { onComplete(); return; }
      } catch {}
    }
    setError('Claude process did not start. Check server logs.');
    setStatus('error');
  } catch (err) { setError(err.message); setStatus('error'); }
};
```

### `<SetupScreen>` Component

A complete, self-contained React component with a two-step OAuth flow.
Neutral styling — clean and minimal, easy to customize for any app.

Props: `onComplete` callback, fired when credentials are confirmed.

```jsx
function SetupScreen({ onComplete }) {
  const [step, setStep] = React.useState(1);
  const [status, setStatus] = React.useState('idle');
  const [state, setState] = React.useState('');
  const [code, setCode] = React.useState('');
  const [error, setError] = React.useState('');

  const handleOAuthStart = async () => {
    setStatus('starting');
    setError('');
    const popup = window.open('about:blank', '_blank');
    try {
      const res = await fetch('/api/oauth/start');
      const data = await res.json();
      if (!res.ok) { if (popup) popup.close(); setError(data.error); setStatus('error'); return; }
      setState(data.state);
      if (popup) popup.location.href = data.authUrl;
      else window.open(data.authUrl, '_blank');
      setStep(2);
      setStatus('idle');
    } catch (err) {
      if (popup) popup.close();
      setError(err.message);
      setStatus('error');
    }
  };

  const handleOAuthExchange = async () => {
    if (!code.trim() || !state) return;
    setStatus('exchanging');
    setError('');
    try {
      const res = await fetch('/api/oauth/exchange', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ code: code.trim(), state }),
      });
      const data = await res.json();
      if (!res.ok) {
        if (data.error?.includes('expired')) { setStep(1); setCode(''); setError('Session expired. Start over.'); setStatus('idle'); }
        else { setError(data.error); setStatus('error'); }
        return;
      }
      setStatus('polling');
      for (let i = 0; i < 20; i++) {
        await new Promise(r => setTimeout(r, 1000));
        try {
          const h = await fetch('/api/health');
          const hd = await h.json();
          if (!hd.needsSetup) { onComplete(); return; }
        } catch {}
      }
      setError('Claude process did not start. Check server logs.');
      setStatus('error');
    } catch (err) { setError(err.message); setStatus('error'); }
  };

  const containerStyle = {
    display: 'flex', flexDirection: 'column', alignItems: 'center',
    justifyContent: 'center', minHeight: '100vh', fontFamily: 'system-ui, sans-serif',
    padding: '2rem', backgroundColor: '#fafafa',
  };
  const cardStyle = {
    backgroundColor: '#fff', borderRadius: '12px', padding: '2.5rem',
    maxWidth: '420px', width: '100%', boxShadow: '0 1px 3px rgba(0,0,0,0.1)',
  };
  const headingStyle = { margin: '0 0 0.5rem', fontSize: '1.5rem', fontWeight: 600 };
  const textStyle = { color: '#666', lineHeight: 1.6, margin: '0 0 1.5rem' };
  const buttonStyle = {
    width: '100%', padding: '0.75rem 1.5rem', fontSize: '1rem', fontWeight: 500,
    border: 'none', borderRadius: '8px', cursor: 'pointer',
    backgroundColor: '#1a1a1a', color: '#fff',
  };
  const inputStyle = {
    width: '100%', padding: '0.75rem', fontSize: '1rem', border: '1px solid #ddd',
    borderRadius: '8px', marginBottom: '1rem', boxSizing: 'border-box',
  };
  const errorStyle = {
    backgroundColor: '#fef2f2', color: '#dc2626', padding: '0.75rem',
    borderRadius: '8px', marginBottom: '1rem', fontSize: '0.875rem',
  };
  const secondaryStyle = {
    ...buttonStyle, backgroundColor: 'transparent', color: '#666',
    border: '1px solid #ddd', marginTop: '0.5rem',
  };

  return React.createElement('div', { style: containerStyle },
    React.createElement('div', { style: cardStyle },
      error && React.createElement('div', { style: errorStyle }, error),

      step === 1 ? React.createElement(React.Fragment, null,
        React.createElement('h1', { style: headingStyle }, 'Connect Your Account'),
        React.createElement('p', { style: textStyle },
          'Sign in with your Anthropic account to get started. ',
          'You\'ll be redirected to claude.ai to authorize access.'
        ),
        React.createElement('button', {
          style: { ...buttonStyle, opacity: status === 'starting' ? 0.6 : 1 },
          onClick: handleOAuthStart,
          disabled: status === 'starting',
        }, status === 'starting' ? 'Opening...' : 'Sign in with Anthropic')
      ) : React.createElement(React.Fragment, null,
        React.createElement('h1', { style: headingStyle }, 'Paste Your Code'),
        React.createElement('p', { style: textStyle },
          'After authorizing, you\'ll see a code on the Anthropic page. ',
          'Copy it and paste it below.'
        ),
        React.createElement('input', {
          style: inputStyle, type: 'text', placeholder: 'Paste authorization code here',
          value: code, onChange: (e) => setCode(e.target.value),
          disabled: status === 'exchanging' || status === 'polling',
        }),
        React.createElement('button', {
          style: { ...buttonStyle, opacity: !code.trim() || status !== 'idle' ? 0.6 : 1 },
          onClick: handleOAuthExchange,
          disabled: !code.trim() || status !== 'idle',
        }, status === 'exchanging' ? 'Connecting...' : status === 'polling' ? 'Verifying...' : 'Connect'),
        React.createElement('button', {
          style: secondaryStyle,
          onClick: () => { setStep(1); setCode(''); setError(''); },
          disabled: status === 'exchanging' || status === 'polling',
        }, 'Back')
      )
    )
  );
}
```

---

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

---

## CSS Specificity Note

If your app has both a chat input and a setup screen input, beware of
CSS specificity conflicts. A global `input[type="text"] { border: none; }`
rule (common for borderless chat inputs) has specificity `0-1-1`, which
beats a class selector like `.setup-input` at `0-1-0`. The setup input
will silently lose its border.

Fix: use `input.setup-input` (specificity `0-1-1`) for setup screen
input styles so they match or exceed the global rule.
