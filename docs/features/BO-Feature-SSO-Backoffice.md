## Status

**Backend**: **Node.js** API (Express, TypeScript, Prisma) in `backend/`. The backoffice (Vite + React) consumes `/auth/...` and `/api/...` routes exposed by this server.

**State**: implemented — OIDC **Keycloak** (authorization code flow), server-side API session (opaque token), user provisioning in the database.

## Goal

Document **SSO** authentication for the **backoffice** via **OpenID Connect** (Keycloak), alongside any local flows.

- Compared to a generic multi-tenant model: **no** `organisation_sso_config` table, no per-organisation OIDC/SAML init. **Single** IdP configuration via environment variables (`KEYCLOAK_*`, `BACKOFFICE_URL`).
- **First SSO login**: create or update a `User` with `isSso: true`, `password: null`, `ssoProvider` / `ssoSubject` keys; business “rights” scope (roles, permissions) depends on the rest of the product (outside this file’s strict scope).

## Data model (Prisma)

File: `backend/prisma/schema.prisma` (`users` table).

| Field | Role |
|-------|------|
| `password` | `String?` — `null` for **SSO-only** accounts (no bcrypt hash). |
| `isSso` | `Boolean` — `true` if the account is linked to the Keycloak OIDC flow. |
| `ssoProvider` | `String?` — identifier derived from the issuer (see `getSsoProviderKey()` in `keycloak-config.ts`). |
| `ssoSubject` | `String?` — Keycloak `sub` claim. |
| Constraint | `@@unique([ssoProvider, ssoSubject])` — one provider / subject pair = one user. |

## Environment variables

### Backend — `backend/.env`

| Variable | Role |
|----------|------|
| `KEYCLOAK_URL` | Keycloak base URL (e.g. `https://host/auth`) |
| `KEYCLOAK_REALM` | Realm name |
| `KEYCLOAK_CLIENT_ID` | Confidential OIDC client |
| `KEYCLOAK_CLIENT_SECRET` | Client secret (placeholder values like `YOUR_SECRET_HERE` are rejected) |
| `KEYCLOAK_REDIRECT_URI` | Exact **backend** callback URL (e.g. `http://localhost:3001/auth/sso/callback`) — must match a **Valid Redirect URI** in the Keycloak client |
| `BACKOFFICE_URL` | Backoffice origin (e.g. `http://localhost:3002`) — final redirects and validation of the `redirect` parameter (anti open redirect) |
| `KEYCLOAK_OIDC_STRICT_ISS` | Optional: `1` to disable the second attempt without `iss` on `iss mismatch` (see `sso-oidc-exchange.ts`) |
| `API_SESSION_TTL_MS` | Optional — API session token lifetime (default 8 h) |
| `CORS_ORIGINS` | Comma-separated allowed origins (must include the backoffice origin) |
| `OIDC_ATTRIBUTE_MAPPING` | Optional — JSON to map userinfo claims (email, name, etc.); see `normalizeUserinfo` in `backend/src/auth/oidc.ts` |

### Backoffice — `backoffice/.env`

| Variable | Role |
|----------|------|
| `VITE_API_URL` | Backend URL (e.g. `http://localhost:3001`). If empty, the app uses the current origin and relies on the **Vite proxy** to prefix `/auth` to the backend. |

The browser does not talk to Keycloak directly for this flow: everything goes through the backend (redirects and callback).

## Backend endpoints

Prefix mounted in `backend/src/index.ts`: `/auth/sso` (router `createKeycloakSsoRouter`).

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/auth/sso/status` | Whether minimal SSO config is present (`ready`, `reason` on error). No secret; used by the backoffice before showing the SSO button. |
| `GET` | `/auth/sso/login` | Redirects to Keycloak (authorization code). Optional query: `redirect` — absolute backoffice URL for return (same origin as `BACKOFFICE_URL`). Default: `{BACKOFFICE_URL}/auth/callback`. |
| `GET` | `/auth/sso/callback` | OIDC callback: code exchange, userinfo read, user provisioning, API session token creation, redirect to `redirect` with `?token=...` or to `/auth/callback` with `?error=...`. |
| `GET` | `/auth/me`, `/api/auth/me` | Current user (`Authorization: Bearer <token>`). JSON: selected user fields (id, email, name, image, `ssoProvider`, `isSso`, `createdAt`). |
| `POST` | `/auth/logout` / `/api/auth/logout` | Revokes the token server-side (in-memory store). |

## Business behavior

1. **OIDC scopes**: `openid email profile`. Email is preferred; fallback `preferred_username` if email-shaped; otherwise `sso-{sub}@sso.local` to satisfy `User.email` uniqueness.
2. **Email conflict**: if a user already exists with the same email **without** SSO (`isSso` false) **with** a password — callback returns an error (`sso_email_conflict`); no automatic local/SSO merge.
3. **SSO linking**: if email already exists as SSO or without password conflict, update `ssoProvider` / `ssoSubject` / profile.
4. **Backoffice access**: `RequireAuth` checks session (token on client, validity via `/auth/me`). No `hasBackofficeAccess` field in current `/auth/me` response.
5. **Session**: opaque token (UUID) in an in-memory `Map` (`api-token-store.ts`); backend process restart invalidates sessions. Keycloak OIDC client cached after first success (`getKeycloakOpenIdClient`).
6. **OAuth state**: stored in memory with 10-minute TTL (`sso-state-store.ts`), single use (`consumeSsoState`).

## Front (backoffice)

- Route `/auth/callback`: reads `token` or `error`, stores token (`API_TOKEN_STORAGE_KEY`), calls `refreshUser()` → `GET /auth/me`, redirects to `/`.
- Login page: calls `GET /auth/sso/status`; “Sign in with SSO” → `ssoService.initiateSsoLogin()` → `${API}/auth/sso/login?redirect=.../auth/callback`.
- Files: `backoffice/src/contexts/AuthContext.tsx`, `backoffice/src/pages/AuthCallbackPage/AuthCallbackPage.tsx`, `backoffice/src/routes/RequireAuth.tsx`, `backoffice/src/pages/LoginPage/LoginPage.tsx`, `backoffice/src/services/ssoService.ts`.

## Keycloak configuration (reminder)

- Client: **confidential**, **Standard flow** (authorization code).
- **Valid redirect URIs**: must include **exactly** `KEYCLOAK_REDIRECT_URI` (**backend** URL, not the front).

## Troubleshooting

### SSO unavailable or login error

The backend uses OIDC discovery (`Issuer.discover` on `{KEYCLOAK_URL}/realms/{KEYCLOAK_REALM}`). In development, JSON error on `/auth/sso/login` may expose a technical message.

Common causes:

1. **IdP host unreachable** (DNS / VPN): URL must resolve from the machine running the **backend**, not only from the browser.
2. **Missing or placeholder variables**: `KEYCLOAK_REDIRECT_URI`, `KEYCLOAK_CLIENT_SECRET` non-placeholder, `BACKOFFICE_URL`.
3. **TLS certificate**: internal PKI — handle at OS / Node level (proxy, trust store).
4. **`iss` mismatch**: Keycloak URL (`KEYCLOAK_URL`, with or without `/auth`) inconsistent with issuer from well-known — see logs and `KEYCLOAK_OIDC_STRICT_ISS`.
5. After fixes, restart the backend for a clean OIDC client if caching is an issue.

### Backoffice cannot reach `/auth/sso/status`

Check `VITE_API_URL` or Vite proxy configuration to the backend.

## Main backend files

- `backend/src/auth/keycloak-sso-routes.ts` — `/auth/sso/*` routes
- `backend/src/auth/keycloak-config.ts` — env, issuer, client, `safeRedirectAfter`
- `backend/src/auth/sso-user.ts` — `findOrCreateSsoUser`
- `backend/src/auth/sso-state-store.ts`, `backend/src/auth/sso-oidc-exchange.ts`, `backend/src/auth/sso-callback-error.ts`, `backend/src/auth/sso-rate-limit.ts`
- `backend/src/auth/oidc.ts` — `normalizeUserinfo` / `OIDC_ATTRIBUTE_MAPPING`
- `backend/src/auth/api-token-store.ts` — session tokens
- `backend/src/middleware/auth.ts` — Bearer + user loading
- `backend/src/index.ts` — router mount and `/auth/me`

## Related documentation

- Technical and security guide: [`SSO_IMPLEMENTATION_GUIDE.md`](./SSO_IMPLEMENTATION_GUIDE.md)
