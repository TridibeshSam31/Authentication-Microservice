# OIDC Authentication Microservice

A production-ready OIDC-compatible authentication microservice built with Node.js, Express, Prisma, and PostgreSQL.

## Features

- ✅ User Registration & Login with Argon2 password hashing
- ✅ JWT Token Issuance (ID Token & Access Token)
- ✅ JWKS Endpoint for public key discovery
- ✅ OAuth 2.0 Authorization Code Flow
- ✅ PKCE Support (S256 & plain)
- ✅ Refresh Tokens with Rotation & Reuse Detection
- ✅ UserInfo Endpoint
- ✅ OIDC Discovery (`/.well-known/openid-configuration`)
- ✅ Security Headers (Helmet)
- ✅ CORS Support
- ✅ Structured Logging (Pino)

## Prerequisites

- Node.js 20+
- Docker & Docker Compose (for PostgreSQL)

## Setup

1. **Start PostgreSQL**:
   ```bash
   docker-compose up -d
   ```

2. **Install Dependencies**:
   ```bash
   npm install
   ```

3. **Run Migrations**:
   ```bash
   npx prisma migrate dev --name init
   ```

4. **Generate Prisma Client**:
   ```bash
   npx prisma generate
   ```

5. **Start Development Server**:
   ```bash
   npm run dev
   ```

The server will start on `http://localhost:3000`.

## API Endpoints

### Authentication
- `POST /auth/register` - Register a new user
- `POST /auth/login` - Login user

### OAuth 2.0 / OIDC
- `GET /oauth/authorize` - Authorization endpoint
- `POST /oauth/token` - Token endpoint
- `GET /oauth/userinfo` - UserInfo endpoint

### Discovery
- `GET /.well-known/openid-configuration` - OIDC Discovery Document
- `GET /jwks` - JSON Web Key Set

### Health
- `GET /health` - Health check

## Example Flow

### 1. Register a User
```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password123"}'
```

### 2. Authorization Code Flow with PKCE

Generate code verifier and challenge:
```bash
# Code verifier (random string)
CODE_VERIFIER=$(openssl rand -base64 32 | tr -d "=+/" | cut -c1-43)

# Code challenge (SHA256 hash of verifier)
CODE_CHALLENGE=$(echo -n $CODE_VERIFIER | openssl dgst -sha256 -binary | base64 | tr -d "=+/" | tr "/+" "_-")
```

Get authorization code:
```bash
curl "http://localhost:3000/oauth/authorize?response_type=code&client_id=demo-client&redirect_uri=http://localhost:3000/callback&code_challenge=$CODE_CHALLENGE&code_challenge_method=S256"
```

Exchange code for tokens:
```bash
curl -X POST http://localhost:3000/oauth/token \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"authorization_code\",\"code\":\"YOUR_CODE\",\"redirect_uri\":\"http://localhost:3000/callback\",\"client_id\":\"demo-client\",\"code_verifier\":\"$CODE_VERIFIER\"}"
```

### 3. Access UserInfo
```bash
curl http://localhost:3000/oauth/userinfo \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## Environment Variables

Create a `.env` file:
```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/oidc_auth?schema=public"
PORT=3000
```

## Production Build

```bash
npm run build
npm start
```

## Tech Stack

- **Runtime**: Node.js 20+ with TypeScript
- **Framework**: Express.js
- **Database**: PostgreSQL
- **ORM**: Prisma 5
- **Auth**: jsonwebtoken, argon2
- **Validation**: Zod
- **Logging**: Pino
- **Security**: Helmet, CORS

---

## OIDC Deep Dive — The Complete Protocol Reference

> This section provides a protocol-level understanding of OIDC — every HTTP request, every cryptographic operation, and every security decision that this microservice implements.

### Why OIDC Exists

Before OIDC, every app rolled its own login system. **OAuth 2.0** solved the *authorization* problem ("can this app access my photos?") but said nothing about *identity* ("who is this user?").

**OIDC** is a standardized identity layer on top of OAuth 2.0. It adds:
- A standard **ID Token** (JWT) that tells you who the user is
- A standard **UserInfo endpoint** for fetching profile data
- A standard **Discovery document** so clients can auto-configure themselves

### The Three Actors

Every OIDC flow involves exactly three parties:

| Actor | Role | Description |
|-------|------|-------------|
| **End User** | Human | Has a browser, owns identity |
| **Relying Party (RP)** | Client App | Wants to know WHO the user is |
| **OpenID Provider (OP)** | Auth Server | Authenticates users and issues tokens — **this microservice** |

### OIDC Discovery

Before any auth flow begins, the client reads the **Discovery Document** at `/.well-known/openid-configuration`. This returns all endpoint URLs, supported scopes, grant types, and signing algorithms — enabling any OIDC-compliant client library (NextAuth, Passport.js, Spring Security) to auto-configure itself.

```json
{
  "issuer": "https://auth.yourdomain.com",
  "authorization_endpoint": "https://auth.yourdomain.com/auth/authorize",
  "token_endpoint": "https://auth.yourdomain.com/auth/token",
  "userinfo_endpoint": "https://auth.yourdomain.com/auth/userinfo",
  "jwks_uri": "https://auth.yourdomain.com/.well-known/jwks.json",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email", "offline_access"],
  "code_challenge_methods_supported": ["S256"]
}
```

### Authorization Code Flow with PKCE — Step by Step

This is the **only flow implemented** — the Implicit flow is deprecated.

#### Step 0: Client Registration

Before a client app can authenticate users, it registers with the OP:

| Field | Description |
|-------|-------------|
| `client_id` | Unique identifier for the app |
| `client_secret` | Hashed secret (confidential clients only) |
| `redirect_uris` | Whitelisted callback URLs |
| `grant_types` | Allowed grant types |
| `client_type` | `"public"` (SPA/mobile, must use PKCE) or `"confidential"` (backend server) |

#### Step 1: PKCE Setup (Client-Side)

```
code_verifier  = random_string(43-128 chars, URL-safe)
code_challenge = BASE64URL( SHA256( code_verifier ) )
```

> **Why PKCE?** Without it, an attacker who intercepts the authorization code can exchange it for tokens. With PKCE, they'd also need the `code_verifier` which never leaves the client.

#### Step 2: Authorization Request

The client redirects the user to the authorization endpoint with these parameters:

| Parameter | Purpose | Security Role |
|-----------|---------|---------------|
| `response_type=code` | Request an authorization code | Forces the secure code flow |
| `client_id` | Identifies the app | Looked up in DB |
| `redirect_uri` | Where to send the user back | **Must exactly match** registered URI |
| `scope=openid` | **Required for OIDC** | Triggers ID token issuance |
| `state` | Random string | CSRF protection |
| `nonce` | Random string embedded in ID token | Replay protection |
| `code_challenge` | SHA256 hash of PKCE verifier | Binds request to token exchange |
| `code_challenge_method=S256` | Hash algorithm | Only S256 is acceptable |

#### Step 3: User Authenticates + Consent

1. Server validates `client_id` and `redirect_uri`
2. Shows login form → verifies credentials with **argon2**
3. Shows consent screen → user approves requested scopes
4. Generates a one-time-use **authorization code** (expires in ≤ 10 minutes)
5. Redirects user back: `302 → https://myapp.com/callback?code=...&state=...`

#### Step 4: Token Exchange (Back-Channel)

The client's backend exchanges the code for tokens. The server runs a **validation gauntlet**:

1. ✅ Verify `grant_type` is `"authorization_code"`
2. ✅ Look up the authorization code — verify not expired, not used
3. ✅ Mark code as used **immediately** (prevent race conditions)
4. ✅ Verify `client_id` and `redirect_uri` match
5. ✅ **PKCE verification**: `SHA256(code_verifier) === stored code_challenge`
6. ✅ If confidential client → verify `client_secret`
7. ✅ All checks pass → **issue tokens**

### The Three Tokens

#### 1. Access Token (JWT) — Resource Authorization

| Claim | Purpose |
|-------|---------|
| `iss` | Your server (issuer) |
| `sub` | User's unique ID |
| `aud` | Which client this token is for |
| `exp` | Expiration (5–15 min) |
| `scope` | What was authorized |
| `jti` | Unique ID for revocation |

> **Lifetime: 5–15 minutes.** Short-lived because access tokens are stateless and harder to revoke.

#### 2. ID Token (JWT) — User Identity (OIDC-Specific)

Everything in the access token **plus**:

| Claim | Purpose |
|-------|---------|
| `auth_time` | When the user actually logged in |
| `nonce` | Must match what client sent (replay protection) |
| `at_hash` | Cryptographic binding to the access token |
| `name`, `email`, `picture` | Identity claims from requested scopes |

> **Consumed by the client** — never sent to resource APIs.

#### 3. Refresh Token (Opaque) — Session Continuity

- **Not a JWT** — an opaque random string, stored hashed in the database
- Enables getting new access/ID tokens without re-authentication
- **Must be revocable** (why it's not a JWT)
- Linked to a **token family** for reuse detection
- Lifetime: 7–30 days

### Refresh Token Rotation & Reuse Detection

```
[Legitimate Client]    uses RT_1  →  Gets RT_2 + new AT  ✓
[Attacker stole RT_1]  uses RT_1  →  Server sees RT_1 is REVOKED
                                     → ALARM! Token reuse detected!
                                     → Revoke ENTIRE family (RT_2 too)
                                     → Force user to re-authenticate
```

### JWKS — Decentralized Token Verification

External services verify tokens **without calling the auth server** — they fetch the public keys from `/.well-known/jwks.json` (cached, refreshed periodically).

**Verification flow:**
1. Decode JWT header → extract `kid`
2. Fetch JWKS → find key matching `kid`
3. Verify RS256 signature with public key
4. Validate claims (`iss`, `aud`, `exp`)

**Key rotation:** New keys are generated periodically. Both old and new keys are published in JWKS simultaneously until all tokens signed with the old key have expired.

### Security Invariants

| Rule | Consequence of Breaking |
|------|------------------------|
| Authorization codes are **one-time-use** | Attacker replays intercepted code |
| Authorization codes expire in **≤ 10 min** | Attacker has unlimited time to use stolen code |
| `redirect_uri` must **exactly match** | Attacker redirects code to their server |
| PKCE S256 is **mandatory** for public clients | Stolen codes are exchangeable |
| Access tokens live **≤ 15 min** | Attacker has indefinite access |
| Refresh tokens are **rotated on every use** | Attacker uses stolen RT indefinitely |
| `state` is **verified on callback** | Attacker forges auth requests (CSRF) |
| `nonce` is **in ID token and verified** | Attacker replays old ID tokens |
| JWKS serves **only public keys** | Private key exposure = forge any token |
| Passwords hashed with **argon2id** | Passwords cracked in minutes |

### Standards Implemented

- [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) — OAuth 2.0 Framework
- [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) — PKCE
- [RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517) — JSON Web Key (JWK)
- [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) — JSON Web Token (JWT)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html)

---

## License

Copyright © 2026 Tridibesh Samantroy. All Rights Reserved.

This project and all of its source code, documentation, diagrams, and assets are the exclusive intellectual property of Tridibesh Samantroy.

Unauthorized copying, reproduction, redistribution, modification, or commercial use of this project — in whole or in part — is strictly prohibited without prior written permission from the author.

This repository is made publicly visible for portfolio and demonstration purposes only. Viewing the code does not grant any license to use, copy, fork, or distribute it.

For licensing inquiries, please contact the author directly.
