## Statut
> **Backend .NET** : l’API utilisée par le backoffice est **ASP.NET Core** dans `backend/` (routes /api/..., /auth/...). Le dossier (node) `backend_node/` est une archive (non utilisée par le compose par défaut).

Etat: implemente et valide localement (acces admin selon permissions `media.*`)

## Objectif
Documenter la fonctionnalite backoffice **Media** permettant de:
- stocker des **images en base de donnees** (binaire PostgreSQL `bytea`)
- les reutiliser partout sur le site (custom pages, banners, etc.) via une **URL publique stable** (`GET /api/public/media/:id`)

L'objectif est de faciliter le knowledge sharing entre agents AI et l'equipe (dev / QA), en expliquant:
- le modele de donnees Prisma (`Media` + `MediaBlob`)
- les formats et limites (SVG inclus, max 5MB)
- les endpoints backend public vs admin (protection par permissions)
- le wiring front (route UI sous Configuration, service API)

## Vue d'ensemble (high level)
1. Un **upload admin** cree un enregistrement `Media` (metadata) + `MediaBlob` (bytes).
2. Le **front public** ou tout client consomme l'image via **`GET /api/public/media/:id`** (pas d'auth).
3. Les operations de gestion (liste, upload, update metadata, delete) passent par **`/api/admin/media`** avec `requirePermission('media.*')`.
4. La liste admin ne charge **pas** les bytes en masse: seules les metadata sont retournees; le binaire est servi a la demande.

## Modele de donnees Prisma (DB)
Fichier (node): `backend_node/prisma/schema.prisma`
Fichier (.NET): schéma PostgreSQL versionné `backend/sql/migrations/` ; persistance Dapper `backend/Data/` ; ports `backend/Application/Abstractions/Persistence/`

### Entites
- `Media`
  - `id` (cuid)
  - `name` (libelle affichage / nom logique)
  - `folder` (optionnel, pour organiser la bibliotheque)
  - `alt` (optionnel, accessibilite)
  - `mimeType` (ex. `image/png`)
  - `sizeBytes`
  - `width`, `height` (optionnels, inferes cote serveur au mieux)
  - `createdAt`, `updatedAt`
  - `createdById` (optionnel, lien vers `User` pour audit)
  - relation 1–1: `blob MediaBlob?`
- `MediaBlob`
  - `mediaId` (PK, FK vers `Media`, `onDelete: Cascade`)
  - `bytes` (`Bytes` / `bytea`)
  - `createdAt`, `updatedAt`

### Pourquoi separer `Media` et `MediaBlob`
- Permet des listes rapides (`findMany` sur `Media` sans charger les gros champs binaires).
- Endpoint public lit uniquement `mimeType` + `bytes` pour un `id` donne.

## Formats acceptes et contraintes
- **Autorises**: `image/png`, `image/jpeg`, `image/webp`, `image/gif`, `image/svg+xml`
- **Taille max**: **5 MB** par fichier (multer `limits.fileSize`)

## Permission codes (contrat backend)
Codes ajoutes au seed (en plus des permissions Users/Roles/Permissions):
- `media.read`
- `media.write`
- `media.delete`

Ces codes sont utilises pour la protection des endpoints via `requirePermission(code)` (meme mecanisme que la doc Admin Users/Roles/Permissions).

## Seed par defaut
Fichier (node): `backend_node/prisma/seed.ts`
Fichier (.NET): `backend/Infrastructure/BootstrapSeedHostedService.cs`

- Upsert des permissions incluant `media.read`, `media.write`, `media.delete`
- Role `admin`: toutes les permissions (y compris media)
- Role `viewer`: lecture seule etendue avec `media.read` (en plus de `users.read`, `roles.read`, `permissions.read`)

Commande (node) :
- `cd backend_node`
- `npm run prisma:seed`

Commande (.NET) : seed au démarrage via `BootstrapSeedHostedService` ; schéma via `backend/sql/migrations/`.

## AuthN/AuthZ (backend)
Fichier (node): `backend_node/src/index.ts`
Fichier (.NET): `AdminMediaController.cs`, `PublicMediaController.cs` sous `backend/Presentation/Controllers/` ; `IAdminMediaRepository` / persistance sous `backend/Data/` ; handlers sous `backend/Application/`.

Le comportement est le meme que pour les autres features admin (voir `docs/features/BO-Feature-Admin-Users-Roles-Permissions.md`):
- `POST /auth/login` -> token UUID
- `Authorization: Bearer <token>` pour `/api/admin/*`
- `requireAuth` + `requirePermission`
- Note dev: map token en RAM (`tokenToUserId`)

## Endpoints backend (public vs admin)
Fichier (node): `backend_node/src/index.ts`
Fichier (.NET): `AdminMediaController.cs`, `PublicMediaController.cs` sous `backend/Presentation/Controllers/` ; `IAdminMediaRepository` / persistance sous `backend/Data/` ; handlers sous `backend/Application/`.

### Public (consultation binaire)
- `GET /api/public/media/:id`
  - reponse: corps binaire
  - headers: `Content-Type` = `mimeType`, `Cache-Control`, `ETag` (basique sur `updatedAt`)

Usage typique cote site:
- `<img src="https://<api>/api/public/media/<id>" alt="..." />`
- Optionnel: query `?v=<updatedAt>` pour invalider le cache navigateur apres mise a jour.

### Admin (CRUD protege)
Tous ces endpoints sont proteges par `requirePermission`:

Media
- `GET /api/admin/media` => `media.read` (liste paginee, **sans** bytes)
- `POST /api/admin/media` => `media.write`
  - `multipart/form-data`
  - champ fichier: `file`
  - champs optionnels: `name` (libelle sans extension recommande ; defaut = nom de fichier **sans** extension), `folder`, `alt`
- `PUT /api/admin/media/:id` => `media.write` (metadata: `name`, `folder`, `alt`) ; si `name` est envoye et vide apres trim -> **400** (`Le nom ne peut pas être vide`)
- `DELETE /api/admin/media/:id` => `media.delete`

## Backoffice (front React)

### Routes UI
Fichier: `backoffice/src/App.tsx`

- `/configuration/media` -> `MediaPage`

### Menu Sidebar
Fichier: `backoffice/src/components/organisms/AppSidebar/AppSidebar.tsx`
- Section **Configuration** contient une entree **Media** -> `/configuration/media`

### AuthContext (token stockage)
Fichier: `backoffice/src/contexts/AuthContext.tsx`
- token dans `localStorage` (`backoffice_session`), reutilise par les appels admin

### Services API
Fichier: `backoffice/src/services/mediaApi.ts`
- `GET /api/admin/media` avec `Authorization: Bearer <token>`
- `POST /api/admin/media` en `FormData` (champ `file` + metadata)
- `PUT` / `DELETE` sur `/api/admin/media/:id`
- Helper `publicMediaUrl(id, updatedAt?)` pour construire l'URL publique d'affichage / copie

### Modal de selection media (reutilisable)
Le backoffice reutilise un **modal de selection media** dans plusieurs features (pages custom, banniere, etc.) pour choisir un media existant.

Comportement attendu:
- le modal affiche la bibliotheque media (liste/grille) pour selectionner un element existant
- si la bibliotheque est vide, ou si aucun media ne correspond au contexte de recherche/filtre, afficher un **etat vide explicite**
- depuis cet etat vide, proposer une action primaire **Ajouter un media** (CTA visible sans quitter le modal)
- le CTA ouvre le flux d'upload (modal d'upload existant ou etape integree), puis rafraichit la bibliotheque a la fin
- apres upload reussi, le media nouvellement cree doit pouvoir etre selectionne immediatement dans ce meme parcours

Objectif UX:
- eviter les impasses dans les formulaires qui dependent d'un media
- reduire les allers-retours entre la feature courante et `/configuration/media`
- conserver un parcours continu: **selectionner si existe, creer sinon, puis selectionner**

### Utilitaires (nom affiche)
Fichier: `backoffice/src/utils/mediaDisplayName.ts`
- `filenameStem` : stem du nom de fichier (upload)
- `mediaNameStemForEdit` : preparation du champ nom en edition (retire une extension image en fin de chaine si presente, pour donnees historiques)

### Toast (confirmation copie)
- `backoffice/src/hooks/useToast.ts` — message unique, auto-dismiss
- `backoffice/src/components/atoms/Toast/ToastBanner.tsx` — bandeau fixe bas-droite (`z-[300]`), `role="status"`, `aria-live="polite"`
- Sur **Copier l’URL** (liste, grille, modal apercu) : toast de confirmation apres copie reussie ; en repli navigateur (`prompt`), message adapte

## Page backoffice

### MediaPage
Fichier: `backoffice/src/pages/MediaPage/MediaPage.tsx`

Fonctionnalites:
- **affichage liste ou miniatures**: interrupteur (Headless UI `Switch`) **Liste** / **Miniatures**
- **liste (tableau)** : colonnes apercu (carre fixe, `object-contain object-center`), nom, dossier, type, taille, dimensions, actions. **Clic sur la ligne** (sauf colonne actions) ouvre le **modal d’aperçu** ; **Entrée** / **Espace** lorsque la ligne est focus (`tabIndex`) ouvrent aussi l’aperçu. La colonne **Actions** utilise `stopPropagation` pour ne pas declencher l’aperçu.
- **miniatures (grille)** : vignettes **carrees** (`aspect-square`), image en **remplissage** du bloc (`object-cover object-center`). **Clic sur la zone image** (bouton plein cadre) ouvre le modal d’aperçu ; le reste de la carte (texte, actions) ne l’ouvre pas.
- **modal d’aperçu** : image en grand (`object-contain`), titre = nom, pied avec type / taille / dimensions, **Copier l’URL**, **Fermer** (pas d’icône œil dédiée dans la grille ou la liste).
- **upload** (modal) : multi-fichiers ; pour chaque fichier, **apercu image** (blob URL, carre) + champ **nom affiche (sans extension)** pre-rempli ; dossier et alt optionnels communs ; envoi `name` en `multipart` ; defaut API si absent = stem du `originalname`
- **edition** (icône **crayon** `PencilSquareIcon`) : modal **Modifier le média** avec **apercu image** au-dessus du formulaire (URL publique, `object-cover object-center` dans un cadre limite en hauteur), champs nom (sans extension en UI), dossier, alt ; **Enregistrer** appelle `PUT /api/admin/media/:id` puis **ferme le modal** et rafraîchit la liste ; **Annuler** ferme sans sauver
- actions par ligne / carte : **modifier**, **copier l’URL** (toast), **supprimer**
- pagination basique (meme pattern que les autres pages liste)

## Facons de debug / test rapide
1. Se connecter avec un compte ayant `media.write` (ou `isAdmin`).
2. Aller sur `/configuration/media`, uploader une image PNG/JPEG (<= 5MB).
3. Verifier dans le navigateur:
   - `GET /api/admin/media` retourne la ligne sans bytes
   - `GET /api/public/media/<id>` affiche l'image
4. Si 403 sur admin:
   - verifier que le user a `media.read` / `media.write` / `media.delete` dans son snapshot ou `isAdmin`
5. Si upload refuse:
   - verifier le type MIME supporte et la taille

## Limitations et prochaines ameliorations (conseils)
- **Stockage en DB**: pratique pour un MVP; en forte volumetrie, envisager object storage (S3) + URL signees, en gardant `Media` comme reference.
- **Pas de variantes** (thumb / large): a ajouter si besoin UI (jobs async ou generation a l'upload).
- **SVG**: supporte pour faciliter l'usage d'illustrations vectorielles.
- **Auth token en RAM** (dev): meme limite que le reste du backoffice — migrer vers JWT / sessions persistees en prod.
