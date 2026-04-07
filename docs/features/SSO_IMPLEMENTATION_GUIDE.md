# Guide d’implémentation SSO — projet adserver

Ce document décrit **l’implémentation réelle** du SSO backoffice dans ce dépôt : **OpenID Connect** avec **Keycloak**, backend **Node.js** (**Express**, **TypeScript**, **Prisma**), front **React** (**Vite**). Il complète la fiche fonctionnelle [`BO-Feature-SSO-Backoffice.md`](./BO-Feature-SSO-Backoffice.md).

Pour une vue produit courte, privilégier [`BO-Feature-SSO-Backoffice.md`](./BO-Feature-SSO-Backoffice.md).

---

## 1. Vue d’ensemble

### Périmètre implémenté

- **Protocole** : OIDC (flux **authorization code**), IdP **Keycloak**.
- **Configuration** : variables d’environnement uniquement — **pas** de configuration SSO par organisation en base.
- **Session applicative** : après succès OIDC, le backend émet un **jeton opaque** (UUID) stocké en mémoire ; le front le conserve (ex. `localStorage`) et l’envoie en `Authorization: Bearer`.

### Hors périmètre actuel (référence)

- **SAML 2.0**, fédération multi-IdP, tables type `organisation_sso_config` / `sso_user_mapping` par tenant : non présents dans le code actuel. Une évolution dans ce sens impliquerait de nouvelles routes, schéma Prisma et services dédiés.

---

## 2. Architecture

```
┌─────────────────┐     redirections HTTP      ┌─────────────────┐
│   Backoffice    │ ─────────────────────────► │    Backend      │
│  (React / Vite) │ ◄── redirect ?token=... ── │  Express + SSO  │
└─────────────────┘                            └────────┬────────┘
                                                        │
                        découverte OIDC, token, userinfo │
                                                        ▼
                                               ┌─────────────────┐
                                               │    Keycloak     │
                                               │  (realm / OIDC) │
                                               └─────────────────┘
```

- Le **navigateur** ne parle pas à Keycloak en JavaScript applicatif pour ce flux : le backend construit l’URL d’autorisation et reçoit le **callback** avec `code` et `state`.
- Le backend appelle Keycloak pour **échanger le code** et lit le **userinfo** (via `openid-client`).

### Composants principaux (code)

| Zone | Rôle |
|------|------|
| `backend/src/auth/keycloak-sso-routes.ts` | Routes `/auth/sso/login`, `/callback`, `/status` |
| `backend/src/auth/keycloak-config.ts` | Lecture env, `Issuer.discover`, client OIDC, validation `redirect` |
| `backend/src/auth/sso-user.ts` | Création / mise à jour `User` (conflit e-mail, clés SSO) |
| `backend/src/auth/sso-state-store.ts` | State OAuth (mémoire, TTL, usage unique) |
| `backend/src/auth/sso-oidc-exchange.ts` | Échange du code ; contournement optionnel `iss mismatch` |
| `backend/src/auth/oidc.ts` | Normalisation du profil (`normalizeUserinfo`, `OIDC_ATTRIBUTE_MAPPING`) |
| `backend/src/auth/api-token-store.ts` | Jetons de session API |
| `backend/src/auth/sso-rate-limit.ts` | Rate limiting `express-rate-limit` sur init / callback |
| `backend/src/middleware/auth.ts` | Bearer → utilisateur Prisma |
| `backoffice/src/services/ssoService.ts` | Statut SSO + URL de login |
| `backoffice/src/pages/AuthCallbackPage/` | Réception `token` / `error` |

### Dépendances npm pertinentes

- `openid-client` — client OIDC (découverte, authorization URL, callback, userinfo).
- `express-rate-limit` — limitation du débit sur les entrées SSO.

---

## 3. Modèle de données

Schéma : `backend/prisma/schema.prisma`, modèle `User` (table `users`).

Champs SSO utiles :

- `isSso`, `password` (nullable pour compte SSO pur).
- `ssoProvider`, `ssoSubject` avec contrainte unique composée `@@unique([ssoProvider, ssoSubject])`.

Le `ssoProvider` est dérivé de l’URL d’issuer (voir `getSsoProviderKey()` dans `keycloak-config.ts`) pour distinguer un futur changement d’IdP sans collision de `sub` entre fournisseurs.

---

## 4. Flux d’authentification (OIDC)

1. L’utilisateur ouvre le backoffice ; la page login appelle **`GET /auth/sso/status`** pour savoir si le backend a une config minimale (`KEYCLOAK_*`, `BACKOFFICE_URL`, secret non placeholder).
2. Clic « Se connecter avec SSO » → redirection vers **`GET /auth/sso/login?redirect=<URL encodée>`** où `redirect` est une URL du backoffice (même origine que `BACKOFFICE_URL`, sinon valeur par défaut `{BACKOFFICE_URL}/auth/callback`).
3. Le backend génère un **`state`**, le stocke avec l’URL de retour, redirige vers Keycloak avec `scope=openid email profile`.
4. L’utilisateur s’authentifie sur Keycloak.
5. Keycloak redirige vers **`GET /auth/sso/callback?code=...&state=...`** sur le backend (`KEYCLOAK_REDIRECT_URI`).
6. Le backend valide et consomme le `state`, échange le `code` contre des tokens, appelle **userinfo**, normalise le profil (`normalizeUserinfo`).
7. **`findOrCreateSsoUser`** : recherche par `(ssoProvider, ssoSubject)`, sinon par e-mail avec règles de conflit (compte mot de passe → erreur `sso_email_conflict`).
8. Création d’un **jeton de session API**, redirection vers l’URL stockée dans le state avec **`?token=<uuid>`** (ou vers `/auth/callback` du backoffice avec **`?error=`** et éventuellement `error_description`).
9. Le front enregistre le jeton, appelle **`GET /auth/me`**, redirige vers l’accueil.

---

## 5. Variables d’environnement (backend)

| Variable | Obligatoire SSO | Description |
|----------|-----------------|-------------|
| `KEYCLOAK_URL` | Oui | Base Keycloak (souvent avec suffixe `/auth`) |
| `KEYCLOAK_REALM` | Oui | Nom du realm |
| `KEYCLOAK_CLIENT_ID` | Oui | Client confidentiel |
| `KEYCLOAK_CLIENT_SECRET` | Oui | Secret réel (les placeholders connus sont rejetés) |
| `KEYCLOAK_REDIRECT_URI` | Oui | URL publique du callback **backend** (ex. `http://localhost:3001/auth/sso/callback`) |
| `BACKOFFICE_URL` | Oui | Origine du backoffice (sans slash final géré côté code) |
| `KEYCLOAK_OIDC_STRICT_ISS` | Non | Si `1`, pas de seconde tentative sans `iss` lors d’un `iss mismatch` |
| `API_SESSION_TTL_MS` | Non | Durée de vie du jeton API (défaut 8 h) |
| `CORS_ORIGINS` | Recommandé | Doit inclure l’origine du backoffice |
| `OIDC_ATTRIBUTE_MAPPING` | Non | JSON : clés des claims userinfo pour email, nom, avatar, etc. (voir `oidc.ts`) |

Exemple commenté : `backend/.env.example`.

### Backoffice

- `VITE_API_URL` : base du backend ; si vide, utilisation de l’origine courante + proxy Vite pour `/auth` (voir `ssoService.ts`).

---

## 6. Endpoints SSO et session

Montage dans `backend/src/index.ts` :

- **`GET /auth/sso/status`** — JSON `{ ready: boolean, reason?: string }` (pas de fuite de secret).
- **`GET /auth/sso/login`** — redirect 302 Keycloak.
- **`GET /auth/sso/callback`** — traitement OIDC, redirect vers le front avec `token` ou erreurs typées (`invalid_state`, `sso_email_conflict`, `keycloak_denied`, codes mappés dans `sso-callback-error.ts`, etc.).

Session utilisateur (hors SSO) :

- **`GET /auth/me`** et **`GET /api/auth/me`** — utilisateur courant (Bearer).
- **`POST /auth/logout`** / **`POST /api/auth/logout`** — révocation du jeton côté store mémoire.

---

## 7. Sécurité et bonnes pratiques

### Anti open redirect

`safeRedirectAfter()` dans `keycloak-config.ts` n’accepte que des URLs dont **l’origine** correspond à `BACKOFFICE_URL`.

### State CSRF / corrélation

Le `state` est aléatoire, stocké côté serveur avec expiration (**10 minutes**), supprimé à la lecture (usage unique).

### Rate limiting

`ssoOidcInitRateLimiter` et `ssoOidcCallbackRateLimiter` limitent les abus sur `/login` et `/callback` (clé par IP / client).

### Secret client

`assertKeycloakEnvForSso()` refuse un secret vide ou égal à des placeholders documentés.

### Issuer OIDC

En cas de décalage entre l’issuer découvert et celui renvoyé par Keycloak au callback, une **deuxième tentative** sans paramètre `iss` peut être effectuée (sauf si `KEYCLOAK_OIDC_STRICT_ISS=1`). Les messages d’erreur côté callback peuvent exposer un détail en non-production (`shouldExposeCallbackError`).

### Session en mémoire

Les jetons API et les states SSO sont dans des `Map` **process-local** : redémarrage = sessions invalidées ; montée en charge horizontale nécessiterait un store partagé (Redis, etc.) — hors implémentation actuelle.

---

## 8. Profil utilisateur (claims)

`normalizeUserinfo()` (`oidc.ts`) :

- Exige un `sub` non vide.
- E-mail : claim configuré (défaut `email`), sinon `preferred_username` si format e-mail, sinon `sso-{sub}@sso.local`.
- Nom : claim `name`, ou concaténation `given_name` / `family_name`, ou `preferred_username`.

Le mapping est surchargeable via **`OIDC_ATTRIBUTE_MAPPING`** (JSON).

---

## 9. Dépannage

| Symptôme | Piste |
|----------|--------|
| `ready: false` sur `/auth/sso/status` | Vérifier chaque variable obligatoire et la valeur du secret. |
| Erreur réseau / DNS sur la découverte OIDC | Le **backend** doit résoudre `KEYCLOAK_URL` (VPN, hosts, proxy). |
| `sso_invalid_grant` / refus de code | `KEYCLOAK_REDIRECT_URI` doit **exactement** correspondre à Keycloak ; vérifier le secret client. |
| `sso_iss_mismatch` | Aligner `KEYCLOAK_URL` avec l’issuer du well-known (présence ou absence de `/auth`). |
| `invalid_state` | Délai > TTL, onglets multiples, ou redémarrage backend entre login et callback. |
| `sso_email_conflict` | Compte existant mot de passe + même e-mail : pas de liaison automatique. |
| `sso_db_schema` | Schéma PostgreSQL non à jour : migrations Prisma / `db push`. |
| Front ne joint pas `/auth/sso/status` | `VITE_API_URL` ou proxy Vite vers le backend. |

---

## 10. Tests et exploitation

- **Test manuel** : Keycloak de dev, client confidentiel, redirect URI du backend, variables dans `backend/.env`, lancer backend + backoffice, parcours complet jusqu’à `/auth/me`.
- **Charge / durabilité** : pour la production, planifier persistance ou externalisation des sessions et du state OAuth si plusieurs instances.

---

## 11. Ressources externes

- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [OAuth 2.0](https://oauth.net/2/)
- Documentation Keycloak (clients, realms, redirect URIs)

---

*Document aligné sur le code dans `backend/` et `backoffice/` ; à mettre à jour si les routes ou le schéma Prisma évoluent.*
