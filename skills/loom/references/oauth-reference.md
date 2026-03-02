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
