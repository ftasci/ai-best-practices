# SSO implementation guide — adserver project

This document describes the **actual** backoffice SSO implementation in this repository: **OpenID Connect** with **Keycloak**, **Node.js** backend (**Express**, **TypeScript**, **Prisma**), **React** front (**Vite**). It complements the functional spec [`BO-Feature-SSO-Backoffice.md`](./BO-Feature-SSO-Backoffice.md).

For a short product view, prefer [`BO-Feature-SSO-Backoffice.md`](./BO-Feature-SSO-Backoffice.md).

---

## 1. Overview

### Implemented scope

- **Protocol**: OIDC (**authorization code** flow), IdP **Keycloak**.
- **Configuration**: environment variables only — **no** per-organisation SSO configuration in the database.
- **Application session**: after OIDC success, the backend issues an **opaque** token (UUID) stored in memory; the front keeps it (e.g. `localStorage`) and sends `Authorization: Bearer`.

### Out of scope today (reference)

- **SAML 2.0**, multi-IdP federation, tables like `organisation_sso_config` / `sso_user_mapping` per tenant: not present in current code. Such an evolution would require new routes, Prisma schema, and dedicated services.

---

## 2. Architecture

```
┌─────────────────┐     HTTP redirects      ┌─────────────────┐
│   Backoffice    │ ───────────────────────► │    Backend      │
│  (React / Vite) │ ◄── redirect ?token=... ── │  Express + SSO  │
└─────────────────┘                            └────────┬────────┘
                                                       │
                        OIDC discovery, token, userinfo │
                                                       ▼
                                              ┌─────────────────┐
                                              │    Keycloak     │
                                              │  (realm / OIDC) │
                                              └─────────────────┘
```

- The **browser** does not talk to Keycloak in JavaScript for this flow: the backend builds the authorization URL and receives the **callback** with `code` and `state`.
- The backend calls Keycloak to **exchange the code** and reads **userinfo** (via `openid-client`).

### Main components (code)

| Area | Role |
|------|------|
| `backend/src/auth/keycloak-sso-routes.ts` | `/auth/sso/login`, `/callback`, `/status` routes |
| `backend/src/auth/keycloak-config.ts` | Env read, `Issuer.discover`, OIDC client, `redirect` validation |
| `backend/src/auth/sso-user.ts` | Create / update `User` (email conflict, SSO keys) |
| `backend/src/auth/sso-state-store.ts` | OAuth state (memory, TTL, single use) |
| `backend/src/auth/sso-oidc-exchange.ts` | Code exchange; optional `iss mismatch` workaround |
| `backend/src/auth/oidc.ts` | Profile normalization (`normalizeUserinfo`, `OIDC_ATTRIBUTE_MAPPING`) |
| `backend/src/auth/api-token-store.ts` | API session tokens |
| `backend/src/auth/sso-rate-limit.ts` | Rate limiting with `express-rate-limit` on init / callback |
| `backend/src/middleware/auth.ts` | Bearer → Prisma user |
| `backoffice/src/services/ssoService.ts` | SSO status + login URL |
| `backoffice/src/pages/AuthCallbackPage/` | Receives `token` / `error` |

### Relevant npm dependencies

- `openid-client` — OIDC client (discovery, authorization URL, callback, userinfo).
- `express-rate-limit` — rate limiting on SSO entry points.

---

## 3. Data model

Schema: `backend/prisma/schema.prisma`, `User` model (`users` table).

Useful SSO fields:

- `isSso`, `password` (nullable for SSO-only account).
- `ssoProvider`, `ssoSubject` with composite unique `@@unique([ssoProvider, ssoSubject])`.

`ssoProvider` is derived from the issuer URL (see `getSsoProviderKey()` in `keycloak-config.ts`) to distinguish a future IdP change without `sub` collisions across providers.

---

## 4. Authentication flow (OIDC)

1. User opens the backoffice; login page calls **`GET /auth/sso/status`** to see if the backend has minimal config (`KEYCLOAK_*`, `BACKOFFICE_URL`, non-placeholder secret).
2. Click “Sign in with SSO” → redirect to **`GET /auth/sso/login?redirect=<encoded URL>`** where `redirect` is a backoffice URL (same origin as `BACKOFFICE_URL`, else default `{BACKOFFICE_URL}/auth/callback`).
3. Backend generates **`state`**, stores it with return URL, redirects to Keycloak with `scope=openid email profile`.
4. User authenticates on Keycloak.
5. Keycloak redirects to **`GET /auth/sso/callback?code=...&state=...`** on the backend (`KEYCLOAK_REDIRECT_URI`).
6. Backend validates and consumes `state`, exchanges `code` for tokens, calls **userinfo**, normalizes profile (`normalizeUserinfo`).
7. **`findOrCreateSsoUser`**: lookup by `(ssoProvider, ssoSubject)`, else by email with conflict rules (password account → `sso_email_conflict` error).
8. Create **API session token**, redirect to URL stored in state with **`?token=<uuid>`** (or to backoffice `/auth/callback` with **`?error=`** and optional `error_description`).
9. Front stores token, calls **`GET /auth/me`**, redirects home.

---

## 5. Environment variables (backend)

| Variable | Required for SSO | Description |
|----------|-------------------|-------------|
| `KEYCLOAK_URL` | Yes | Keycloak base (often with `/auth` suffix) |
| `KEYCLOAK_REALM` | Yes | Realm name |
| `KEYCLOAK_CLIENT_ID` | Yes | Confidential client |
| `KEYCLOAK_CLIENT_SECRET` | Yes | Real secret (known placeholders rejected) |
| `KEYCLOAK_REDIRECT_URI` | Yes | Public **backend** callback URL (e.g. `http://localhost:3001/auth/sso/callback`) |
| `BACKOFFICE_URL` | Yes | Backoffice origin (trailing slash handled in code) |
| `KEYCLOAK_OIDC_STRICT_ISS` | No | If `1`, no second attempt without `iss` on `iss mismatch` |
| `API_SESSION_TTL_MS` | No | API token lifetime (default 8 h) |
| `CORS_ORIGINS` | Recommended | Must include backoffice origin |
| `OIDC_ATTRIBUTE_MAPPING` | No | JSON: userinfo claim keys for email, name, avatar, etc. (see `oidc.ts`) |

Commented example: `backend/.env.example`.

### Backoffice

- `VITE_API_URL`: backend base; if empty, current origin + Vite proxy for `/auth` (see `ssoService.ts`).

---

## 6. SSO and session endpoints

Mounted in `backend/src/index.ts`:

- **`GET /auth/sso/status`** — JSON `{ ready: boolean, reason?: string }` (no secret leakage).
- **`GET /auth/sso/login`** — 302 redirect to Keycloak.
- **`GET /auth/sso/callback`** — OIDC handling, redirect to front with `token` or typed errors (`invalid_state`, `sso_email_conflict`, `keycloak_denied`, codes mapped in `sso-callback-error.ts`, etc.).

User session (non-SSO):

- **`GET /auth/me`** and **`GET /api/auth/me`** — current user (Bearer).
- **`POST /auth/logout`** / **`POST /api/auth/logout`** — revoke token in memory store.

---

## 7. Security and good practices

### Anti open redirect

`safeRedirectAfter()` in `keycloak-config.ts` only accepts URLs whose **origin** matches `BACKOFFICE_URL`.

### State CSRF / correlation

`state` is random, stored server-side with expiration (**10 minutes**), removed on read (single use).

### Rate limiting

`ssoOidcInitRateLimiter` and `ssoOidcCallbackRateLimiter` limit abuse on `/login` and `/callback` (key per IP / client).

### Client secret

`assertKeycloakEnvForSso()` rejects empty or known placeholder secrets.

### OIDC issuer

If discovered issuer and Keycloak callback issuer differ, a **second attempt** without `iss` may run (unless `KEYCLOAK_OIDC_STRICT_ISS=1`). Callback error messages may expose detail in non-production (`shouldExposeCallbackError`).

### In-memory session

API tokens and SSO states live in **process-local** `Map`s: restart = sessions invalidated; horizontal scaling would need a shared store (Redis, etc.) — outside current implementation.

---

## 8. User profile (claims)

`normalizeUserinfo()` (`oidc.ts`):

- Requires non-empty `sub`.
- Email: configured claim (default `email`), else `preferred_username` if email-shaped, else `sso-{sub}@sso.local`.
- Name: `name` claim, or `given_name` / `family_name` concatenation, or `preferred_username`.

Mapping is overridable via **`OIDC_ATTRIBUTE_MAPPING`** (JSON).

---

## 9. Troubleshooting

| Symptom | Lead |
|---------|------|
| `ready: false` on `/auth/sso/status` | Check each required variable and secret value. |
| Network / DNS error on OIDC discovery | **Backend** must resolve `KEYCLOAK_URL` (VPN, hosts, proxy). |
| `sso_invalid_grant` / code rejected | `KEYCLOAK_REDIRECT_URI` must **exactly** match Keycloak; verify client secret. |
| `sso_iss_mismatch` | Align `KEYCLOAK_URL` with well-known issuer (with or without `/auth`). |
| `invalid_state` | Delay > TTL, multiple tabs, or backend restart between login and callback. |
| `sso_email_conflict` | Existing password account + same email: no automatic linking. |
| `sso_db_schema` | PostgreSQL schema out of date: Prisma migrations / `db push`. |
| Front does not call `/auth/sso/status` | `VITE_API_URL` or Vite proxy to backend. |

---

## 10. Tests and operations

- **Manual test**: dev Keycloak, confidential client, backend redirect URI, variables in `backend/.env`, run backend + backoffice, full flow to `/auth/me`.
- **Load / durability**: for production, plan persistence or externalization of sessions and OAuth state if multiple instances.

---

## 11. External resources

- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [OAuth 2.0](https://oauth.net/2/)
- Keycloak documentation (clients, realms, redirect URIs)

---

*Document aligned with code in `backend/` and `backoffice/`; update if routes or Prisma schema change.*
