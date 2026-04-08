## Statut

**Backend** : API **Node.js** (Express, TypeScript, Prisma) dans `backend/`. Le backoffice (Vite + React) consomme les routes `/auth/...` et `/api/...` exposées par ce serveur.

**État** : implémenté — OIDC **Keycloak** (flux authorization code), session API côté serveur (jeton opaque), provisioning utilisateur en base.

## Objectif

Documenter l’authentification **SSO** du **backoffice** via **OpenID Connect** (Keycloak), en complément d’éventuels flux locaux.

- Par rapport à un modèle multi-tenant générique : **pas** de table `organisation_sso_config`, pas de code organisation ni d’init SAML/OIDC par organisation. **Une seule** configuration IdP via variables d’environnement (`KEYCLOAK_*`, `BACKOFFICE_URL`).
- **Premier login SSO** : création ou mise à jour d’un enregistrement `User` avec `isSso: true`, `password: null`, clés `ssoProvider` / `ssoSubject` ; le périmètre « droits métier » (rôles, permissions) dépend du reste du produit (hors périmètre strict de ce fichier).

## Modèle de données (Prisma)

Fichier : `backend/prisma/schema.prisma` (table `users`).

| Champ | Rôle |
|--------|------|
| `password` | `String?` — `null` pour les comptes **uniquement SSO** (pas de hash bcrypt). |
| `isSso` | `Boolean` — `true` si le compte est lié au flux OIDC Keycloak. |
| `ssoProvider` | `String?` — identifiant dérivé de l’issuer (voir `getSsoProviderKey()` dans `keycloak-config.ts`). |
| `ssoSubject` | `String?` — claim `sub` Keycloak. |
| Contrainte | `@@unique([ssoProvider, ssoSubject])` — un couple fournisseur / sujet = un utilisateur. |

## Variables d’environnement

### Backend — `backend/.env`

| Variable | Rôle |
|----------|------|
| `KEYCLOAK_URL` | Base URL Keycloak (ex. `https://hôte/auth`) |
| `KEYCLOAK_REALM` | Nom du realm |
| `KEYCLOAK_CLIENT_ID` | Client OIDC (confidentiel) |
| `KEYCLOAK_CLIENT_SECRET` | Secret du client (les valeurs placeholder type `YOUR_SECRET_HERE` sont refusées) |
| `KEYCLOAK_REDIRECT_URI` | URL exacte du callback **backend** (ex. `http://localhost:3001/auth/sso/callback`) — doit correspondre à une **Valid Redirect URI** du client Keycloak |
| `BACKOFFICE_URL` | Origine du backoffice (ex. `http://localhost:3002`) — redirections finales et validation du paramètre `redirect` (anti open redirect) |
| `KEYCLOAK_OIDC_STRICT_ISS` | Optionnel : `1` pour désactiver la seconde tentative sans `iss` en cas d’`iss mismatch` (voir `sso-oidc-exchange.ts`) |
| `API_SESSION_TTL_MS` | Optionnel — durée de vie des jetons de session API (défaut 8 h) |
| `CORS_ORIGINS` | Liste séparée par virgules des origines autorisées (doit inclure l’origine du backoffice) |
| `OIDC_ATTRIBUTE_MAPPING` | Optionnel — JSON pour mapper les claims userinfo (email, nom, etc.) ; voir `normalizeUserinfo` dans `backend/src/auth/oidc.ts` |

### Backoffice — `backoffice/.env`

| Variable | Rôle |
|----------|------|
| `VITE_API_URL` | URL du backend (ex. `http://localhost:3001`). Si vide, l’app utilise l’origine courante et s’appuie sur le **proxy Vite** pour préfixer `/auth` vers le backend. |

Le navigateur ne parle pas directement à Keycloak pour le flux implémenté : tout passe par le backend (redirections et callback).

## Endpoints backend

Préfixe monté dans `backend/src/index.ts` : `/auth/sso` (router `createKeycloakSsoRouter`).

| Méthode | Chemin | Description |
|---------|--------|-------------|
| `GET` | `/auth/sso/status` | Indique si la config SSO minimale est présente (`ready`, `reason` si erreur). Sans secret ; utilisé par le backoffice avant d’afficher le bouton SSO. |
| `GET` | `/auth/sso/login` | Redirige vers Keycloak (authorization code). Query optionnelle : `redirect` — URL absolue du backoffice pour le retour (même origine que `BACKOFFICE_URL`). Défaut : `{BACKOFFICE_URL}/auth/callback`. |
| `GET` | `/auth/sso/callback` | Callback OIDC : échange du code, lecture userinfo, provisionnement utilisateur, création du jeton de session API, redirection vers `redirect` avec `?token=...` ou vers `/auth/callback` avec `?error=...`. |
| `GET` | `/auth/me`, `/api/auth/me` | Utilisateur courant (Bearer `Authorization: Bearer <token>`). Réponse JSON : champs utilisateur sélectionnés (id, email, name, image, `ssoProvider`, `isSso`, `createdAt`). |
| `POST` | `/auth/logout` / `/api/auth/logout` | Révoque le jeton côté serveur (store en mémoire). |

## Comportement métier

1. **Scopes OIDC** : `openid email profile`. L’email est prioritaire ; en secours, `preferred_username` si format e-mail ; sinon `sso-{sub}@sso.local` pour respecter l’unicité `User.email`.
2. **Conflit e-mail** : si un utilisateur existe déjà avec le même e-mail, **sans** SSO (`isSso` false) et **avec** mot de passe — le callback renvoie une erreur (`sso_email_conflict`) ; pas de fusion automatique compte local / SSO.
3. **Liaison SSO** : si l’e-mail existe déjà en compte SSO ou sans conflit mot de passe, mise à jour de `ssoProvider` / `ssoSubject` / profil.
4. **Accès backoffice** : le garde `RequireAuth` vérifie la présence d’une session (token stocké côté client, validité côté `/auth/me`). Pas de champ `hasBackofficeAccess` dans la réponse actuelle de `/auth/me`.
5. **Session** : jeton opaque (UUID) dans une `Map` en mémoire (`api-token-store.ts`) ; redémarrage du processus backend invalide les sessions. État OIDC Keycloak mis en cache après premier succès (`getKeycloakOpenIdClient`).
6. **State OAuth** : stocké en mémoire avec TTL 10 minutes (`sso-state-store.ts`), usage unique (`consumeSsoState`).

## Front (backoffice)

- Route `/auth/callback` : lit `token` ou `error`, enregistre le jeton (`API_TOKEN_STORAGE_KEY`), appelle `refreshUser()` → `GET /auth/me`, redirige vers `/`.
- Page login : appelle `GET /auth/sso/status` ; bouton « Se connecter avec SSO » → `ssoService.initiateSsoLogin()` → `${API}/auth/sso/login?redirect=.../auth/callback`.
- Fichiers : `backoffice/src/contexts/AuthContext.tsx`, `backoffice/src/pages/AuthCallbackPage/AuthCallbackPage.tsx`, `backoffice/src/routes/RequireAuth.tsx`, `backoffice/src/pages/LoginPage/LoginPage.tsx`, `backoffice/src/services/ssoService.ts`.

## Configuration Keycloak (rappel)

- Client : **confidential**, **Standard flow** (authorization code).
- **Valid redirect URIs** : doit inclure **exactement** `KEYCLOAK_REDIRECT_URI` (URL du **backend**, pas celle du front).

## Dépannage

### SSO indisponible ou erreur au login

Le backend utilise la découverte OIDC (`Issuer.discover` sur `{KEYCLOAK_URL}/realms/{KEYCLOAK_REALM}`). En développement, une erreur JSON sur `/auth/sso/login` peut exposer un message technique.

Causes fréquentes :

1. **Hôte IdP introuvable** (DNS / VPN) : l’URL doit être résolvable depuis la machine qui exécute le **backend**, pas seulement depuis le navigateur.
2. **Variables manquantes ou placeholder** : `KEYCLOAK_REDIRECT_URI`, `KEYCLOAK_CLIENT_SECRET` non placeholder, `BACKOFFICE_URL`.
3. **Certificat TLS** : PKI interne — à traiter au niveau OS / Node (proxy, magasin de certificats).
4. **`iss` mismatch** : URL Keycloak (`KEYCLOAK_URL`, avec ou sans `/auth`) incohérente avec l’issuer du well-known — voir logs et `KEYCLOAK_OIDC_STRICT_ISS`.
5. Après correction, redémarrer le backend pour repartir sur un client OIDC propre si le cache pose problème.

### Le backoffice n’atteint pas `/auth/sso/status`

Vérifier `VITE_API_URL` ou la configuration du proxy Vite vers le backend.

## Fichiers principaux (backend)

- `backend/src/auth/keycloak-sso-routes.ts` — routes `/auth/sso/*`
- `backend/src/auth/keycloak-config.ts` — env, issuer, client, `safeRedirectAfter`
- `backend/src/auth/sso-user.ts` — `findOrCreateSsoUser`
- `backend/src/auth/sso-state-store.ts`, `backend/src/auth/sso-oidc-exchange.ts`, `backend/src/auth/sso-callback-error.ts`, `backend/src/auth/sso-rate-limit.ts`
- `backend/src/auth/oidc.ts` — `normalizeUserinfo` / `OIDC_ATTRIBUTE_MAPPING`
- `backend/src/auth/api-token-store.ts` — jetons de session
- `backend/src/middleware/auth.ts` — Bearer + chargement utilisateur
- `backend/src/index.ts` — montage du router et `/auth/me`

## Documentation associée

- Guide technique détaillé et sécurité : [`SSO_IMPLEMENTATION_GUIDE.md`](./SSO_IMPLEMENTATION_GUIDE.md)
