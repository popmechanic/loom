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
5. [Token Lifecycle](#token-lifecycle)
6. [Frontend Setup Screen](#frontend-setup-screen)
7. [Integration Notes](#integration-notes)

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
                                             │  Writes tokens to
                                             ▼
                                       ~/.claude/.credentials.json
                                             │
                                             │  Claude CLI auto-discovers
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

Exchanges an authorization code for tokens using the stored PKCE verifier.

- Validates state exists in pendingPKCE map (returns 400 if expired/missing)
- Strips `#state` suffix from code (handles callback URL fragments)
- POSTs to Anthropic's token endpoint with the code, verifier, and other params
- On success, calls `writeCredentials()` with the returned tokens
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
      return res.status(502).json({ error: `Token exchange failed (${resp.status}): ${err}` });
    }

    const tokens = await resp.json();
    writeCredentials(tokens.access_token, tokens.refresh_token, tokens.expires_in);
    res.json({ ok: true });
  } catch (err) {
    res.status(500).json({ error: "Token exchange failed" });
  }
});
```
