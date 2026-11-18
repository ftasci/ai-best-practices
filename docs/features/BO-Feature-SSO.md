# Backoffice feature — SSO (OIDC / Keycloak)

## Status

**Backend**: **Node.js** API (Express, TypeScript, Prisma) in `backend/`. The backoffice (Vite + React) consumes `/auth/...` and `/api/...` routes exposed by this server.

**State**: implemented — OIDC **Keycloak** (authorization code flow), server-side API session (opaque token), user provisioning in the database.

## Goal

Provide **SSO** authentication for the **backoffice** via **OpenID Connect** (Keycloak), alongside any local flows.

- **Single** IdP configuration via environment variables (`KEYCLOAK_*`, `BACKOFFICE_URL`) — no per-organisation SSO configuration in the database.
- **First SSO login**: create or update a `User` with `isSso: true`, `password: null`, `ssoProvider` / `ssoSubject` keys. Business rights (roles, permissions) depend on the rest of the product and are outside this document's scope.

## Scope

### In scope

- **Protocol**: OIDC, **authorization code** flow, IdP **Keycloak**.
- **Configuration**: environment variables only.
- **Application session**: after OIDC success, the backend issues an **opaque** token (UUID) stored in memory; the front keeps it (`localStorage`) and sends `Authorization: Bearer`.

### Out of scope

- **SAML 2.0**, multi-IdP federation, per-tenant tables such as `organisation_sso_config` / `sso_user_mapping`. Such an evolution would require new routes, Prisma schema changes, and dedicated services.
- Shared session store (Redis, etc.) for horizontal scaling — see [Security](#security).

## Architecture

```text
┌─────────────────┐      HTTP redirects       ┌─────────────────┐
│   Backoffice    │ ────────────────────────► │     Backend     │
│  (React / Vite) │ ◄── redirect ?token=... ─ │  Express + SSO  │
└─────────────────┘                           └────────┬────────┘
                                                       │
                       OIDC discovery, token, userinfo │
                                                       ▼
                                              ┌─────────────────┐
                                              │    Keycloak     │
                                              │  (realm / OIDC) │
                                              └─────────────────┘
```

- The **browser** does not talk to Keycloak in JavaScript: the backend builds the authorization URL and receives the **callback** with `code` and `state`.
- The backend exchanges the code and reads **userinfo** via `openid-client`.

**npm dependencies**: `openid-client` (discovery, authorization URL, callback, userinfo), `express-rate-limit` (rate limiting on SSO entry points).

## Authentication flow

1. The login page calls **`GET /auth/sso/status`** to check that the backend has minimal config (`KEYCLOAK_*`, `BACKOFFICE_URL`, non-placeholder secret) before showing the SSO button.
2. "Sign in with SSO" → `ssoService.initiateSsoLogin()` → **`GET /auth/sso/login?redirect=<encoded URL>`**. `redirect` must share the origin of `BACKOFFICE_URL`, otherwise the default `{BACKOFFICE_URL}/auth/callback` is used.
3. The backend generates **`state`**, stores it with the return URL, and redirects to Keycloak with `scope=openid email profile`.
4. The user authenticates on Keycloak.
5. Keycloak redirects to **`GET /auth/sso/callback?code=...&state=...`** on the backend (`KEYCLOAK_REDIRECT_URI`).
6. The backend validates and consumes `state`, exchanges `code` for tokens, calls **userinfo**, and normalizes the profile (`normalizeUserinfo`).
7. **`findOrCreateSsoUser`** provisions the user (see [Business rules](#business-rules)).
8. The backend creates an **API session token** and redirects to the URL stored in state with **`?token=<uuid>`** — or to backoffice `/auth/callback` with **`?error=`** (and optional `error_description`).
9. The front route `/auth/callback` stores the token (`API_TOKEN_STORAGE_KEY`), calls `refreshUser()` → **`GET /auth/me`**, and redirects to `/`.

## Business rules

### User provisioning

1. **Lookup**: by `(ssoProvider, ssoSubject)` first, then by email.
2. **Email conflict**: if a user exists with the same email, **without** SSO (`isSso: false`) and **with** a password → error `sso_email_conflict`. No automatic local/SSO merge.
3. **SSO linking**: if the email exists as an SSO account or without a password, update `ssoProvider` / `ssoSubject` / profile.

### Profile claims

`normalizeUserinfo()` (`backend/src/auth/oidc.ts`):

- Requires a non-empty `sub`.
- **Email**: configured claim (default `email`), else `preferred_username` if email-shaped, else `sso-{sub}@sso.local` (to satisfy `User.email` uniqueness).
- **Name**: `name` claim, else `given_name` + `family_name`, else `preferred_username`.

Claim keys (email, name, avatar, etc.) can be overridden with **`OIDC_ATTRIBUTE_MAPPING`** (JSON).

### Session and backoffice access

- Opaque token (UUID) in an in-memory `Map` (`api-token-store.ts`), lifetime `API_SESSION_TTL_MS` (default 8 h). A backend restart invalidates all sessions.
- `RequireAuth` checks the session (token on client, validity via `/auth/me`). There is no `hasBackofficeAccess` field in the current `/auth/me` response.
- The Keycloak OIDC client is cached after the first successful discovery (`getKeycloakOpenIdClient`).

## Data model (Prisma)

File: `backend/prisma/schema.prisma`, `User` model (`users` table).

| Field | Role |
|-------|------|
| `password` | `String?` — `null` for **SSO-only** accounts (no bcrypt hash). |
| `isSso` | `Boolean` — `true` if the account is linked to the Keycloak OIDC flow. |
| `ssoProvider` | `String?` — derived from the issuer URL (`getSsoProviderKey()` in `keycloak-config.ts`), so a future IdP change cannot cause `sub` collisions across providers. |
| `ssoSubject` | `String?` — Keycloak `sub` claim. |
| Constraint | `@@unique([ssoProvider, ssoSubject])` — one provider / subject pair = one user. |

## Environment variables

### Backend — `backend/.env`

Commented example: `backend/.env.example`.

| Variable | Required | Role |
|----------|----------|------|
| `KEYCLOAK_URL` | Yes | Keycloak base URL, often with `/auth` suffix (e.g. `https://host/auth`) |
| `KEYCLOAK_REALM` | Yes | Realm name |
| `KEYCLOAK_CLIENT_ID` | Yes | Confidential OIDC client |
| `KEYCLOAK_CLIENT_SECRET` | Yes | Real secret — empty or placeholder values (e.g. `YOUR_SECRET_HERE`) are rejected |
| `KEYCLOAK_REDIRECT_URI` | Yes | Exact **backend** callback URL (e.g. `http://localhost:3001/auth/sso/callback`) — must match a **Valid Redirect URI** in the Keycloak client |
| `BACKOFFICE_URL` | Yes | Backoffice origin (e.g. `http://localhost:3002`, trailing slash handled) — final redirects and `redirect` validation (anti open redirect) |
| `CORS_ORIGINS` | Recommended | Comma-separated allowed origins; must include the backoffice origin |
| `KEYCLOAK_OIDC_STRICT_ISS` | No | `1` disables the second attempt without `iss` on `iss mismatch` (see `sso-oidc-exchange.ts`) |
| `API_SESSION_TTL_MS` | No | API session token lifetime (default 8 h) |
| `OIDC_ATTRIBUTE_MAPPING` | No | JSON mapping of userinfo claims (see [Profile claims](#profile-claims)) |

### Backoffice — `backoffice/.env`

| Variable | Role |
|----------|------|
| `VITE_API_URL` | Backend URL (e.g. `http://localhost:3001`). If empty, the app uses the current origin and relies on the **Vite proxy** to forward `/auth` to the backend (see `ssoService.ts`). |

## Endpoints

Mounted in `backend/src/index.ts`; SSO routes under `/auth/sso` (router `createKeycloakSsoRouter`).

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/auth/sso/status` | `{ ready: boolean, reason?: string }` — whether minimal SSO config is present. No secret leakage. |
| `GET` | `/auth/sso/login` | 302 redirect to Keycloak. Optional query `redirect` (see [flow](#authentication-flow), step 2). |
| `GET` | `/auth/sso/callback` | OIDC callback: code exchange, userinfo, provisioning, session token, redirect with `?token=` or typed `?error=` (`invalid_state`, `sso_email_conflict`, `keycloak_denied`, others mapped in `sso-callback-error.ts`). |
| `GET` | `/auth/me`, `/api/auth/me` | Current user (`Authorization: Bearer <token>`): id, email, name, image, `ssoProvider`, `isSso`, `createdAt`. |
| `POST` | `/auth/logout`, `/api/auth/logout` | Revokes the token in the in-memory store. |

## Security

- **Anti open redirect**: `safeRedirectAfter()` (`keycloak-config.ts`) only accepts URLs whose **origin** matches `BACKOFFICE_URL`.
- **OAuth state (CSRF / correlation)**: random, stored server-side with a **10-minute** TTL, single use (`sso-state-store.ts`, `consumeSsoState`).
- **Rate limiting**: `ssoOidcInitRateLimiter` and `ssoOidcCallbackRateLimiter` limit abuse on `/login` and `/callback` (keyed per IP / client).
- **Client secret**: `assertKeycloakEnvForSso()` rejects empty or known placeholder secrets.
- **OIDC issuer**: if the discovered issuer and the callback issuer differ, a **second attempt** without `iss` may run (unless `KEYCLOAK_OIDC_STRICT_ISS=1`). Callback errors may expose details outside production (`shouldExposeCallbackError`).
- **In-memory state**: API tokens and SSO states live in **process-local** `Map`s. Restart = sessions invalidated; running multiple instances requires a shared store (Redis, etc.), which is not implemented.

## Keycloak configuration

- Client: **confidential**, **Standard flow** (authorization code).
- **Valid redirect URIs**: must include **exactly** `KEYCLOAK_REDIRECT_URI` (the **backend** URL, not the front).

## Troubleshooting

The backend uses OIDC discovery (`Issuer.discover` on `{KEYCLOAK_URL}/realms/{KEYCLOAK_REALM}`). In development, the JSON error on `/auth/sso/login` may expose a technical message.

| Symptom | Lead |
|---------|------|
| `ready: false` on `/auth/sso/status` | Check each required variable and that the secret is not a placeholder. |
| Network / DNS error on OIDC discovery | The **backend** host (not just the browser) must resolve `KEYCLOAK_URL` — VPN, hosts, proxy. |
| TLS certificate error | Internal PKI — handle at OS / Node level (proxy, trust store). |
| `sso_invalid_grant` / code rejected | `KEYCLOAK_REDIRECT_URI` must **exactly** match Keycloak; verify the client secret. |
| `sso_iss_mismatch` | Align `KEYCLOAK_URL` (with or without `/auth`) with the well-known issuer; see logs and `KEYCLOAK_OIDC_STRICT_ISS`. |
| `invalid_state` | Delay > TTL, multiple tabs, or backend restart between login and callback. |
| `sso_email_conflict` | Existing password account with the same email: no automatic linking. |
| `sso_db_schema` | PostgreSQL schema out of date: run Prisma migrations / `db push`. |
| Backoffice cannot reach `/auth/sso/status` | Check `VITE_API_URL` or the Vite proxy to the backend. |
| Still failing after config fixes | Restart the backend to drop the cached OIDC client. |

## Tests and operations

- **Manual test**: dev Keycloak, confidential client, backend redirect URI, variables in `backend/.env`; run backend + backoffice and complete the full flow to `/auth/me`.
- **Production**: plan persistence or externalization of sessions and OAuth state before running multiple instances.

## Code map

### Backend

| File | Role |
|------|------|
| `backend/src/index.ts` | Router mount, `/auth/me` |
| `backend/src/auth/keycloak-sso-routes.ts` | `/auth/sso/login`, `/callback`, `/status` routes |
| `backend/src/auth/keycloak-config.ts` | Env read, `Issuer.discover`, OIDC client, `safeRedirectAfter` |
| `backend/src/auth/sso-user.ts` | `findOrCreateSsoUser` (email conflict, SSO keys) |
| `backend/src/auth/sso-state-store.ts` | OAuth state (memory, TTL, single use) |
| `backend/src/auth/sso-oidc-exchange.ts` | Code exchange; optional `iss mismatch` workaround |
| `backend/src/auth/sso-callback-error.ts` | Callback error codes |
| `backend/src/auth/sso-rate-limit.ts` | Rate limiting on init / callback |
| `backend/src/auth/oidc.ts` | `normalizeUserinfo`, `OIDC_ATTRIBUTE_MAPPING` |
| `backend/src/auth/api-token-store.ts` | API session tokens |
| `backend/src/middleware/auth.ts` | Bearer → Prisma user |

### Backoffice

| File | Role |
|------|------|
| `backoffice/src/services/ssoService.ts` | SSO status + login URL |
| `backoffice/src/pages/LoginPage/LoginPage.tsx` | SSO button (shown when status is ready) |
| `backoffice/src/pages/AuthCallbackPage/AuthCallbackPage.tsx` | Receives `token` / `error` |
| `backoffice/src/contexts/AuthContext.tsx` | Token storage, `refreshUser()` |
| `backoffice/src/routes/RequireAuth.tsx` | Route guard |

## References

- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [OAuth 2.0](https://oauth.net/2/)
- Keycloak documentation (clients, realms, redirect URIs)

---

*Document aligned with code in `backend/` and `backoffice/`; update if routes or Prisma schema change.*
