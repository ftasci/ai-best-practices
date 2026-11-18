## Status

> **Backend .NET**: The API used by the backoffice is **ASP.NET Core** in `backend/` (routes `/api/...`, `/auth/...`). The `backend_node/` folder is an archive (not used by default compose).

**State**: implemented and validated locally (admin access per `media.*` permissions)

## Goal

Document the backoffice **Media** feature allowing:

- storing **images in the database** (PostgreSQL binary `bytea`)
- reusing them across the site (custom pages, banners, etc.) via a **stable public URL** (`GET /api/public/media/:id`)

The goal is to ease knowledge sharing between AI agents and the team (dev / QA) by explaining:

- the Prisma data model (`Media` + `MediaBlob`)
- formats and limits (SVG included, max 5MB)
- public vs admin backend endpoints (permission-protected)
- front wiring (UI route under Configuration, API service)

## High-level overview

1. An **admin upload** creates a `Media` record (metadata) + `MediaBlob` (bytes).
2. The **public front** or any client loads the image via **`GET /api/public/media/:id`** (no auth).
3. Management operations (list, upload, update metadata, delete) go through **`/api/admin/media`** with `requirePermission('media.*')`.
4. The admin list does **not** load bytes in bulk: only metadata is returned; binary is served on demand.

## Prisma data model (DB)

File (node): `backend_node/prisma/schema.prisma`  
File (.NET): versioned PostgreSQL schema `backend/sql/migrations/`; Dapper persistence `backend/Data/`; ports `backend/Application/Abstractions/Persistence/`

### Entities

- `Media`
  - `id` (cuid)
  - `name` (display label / logical name)
  - `folder` (optional, to organize the library)
  - `alt` (optional, accessibility)
  - `mimeType` (e.g. `image/png`)
  - `sizeBytes`
  - `width`, `height` (optional, inferred server-side when possible)
  - `createdAt`, `updatedAt`
  - `createdById` (optional, link to `User` for audit)
  - 1–1 relation: `blob MediaBlob?`
- `MediaBlob`
  - `mediaId` (PK, FK to `Media`, `onDelete: Cascade`)
  - `bytes` (`Bytes` / `bytea`)
  - `createdAt`, `updatedAt`

### Why split `Media` and `MediaBlob`

- Enables fast lists (`findMany` on `Media` without loading large binary fields).
- Public endpoint reads only `mimeType` + `bytes` for a given `id`.

## Accepted formats and constraints

- **Allowed**: `image/png`, `image/jpeg`, `image/webp`, `image/gif`, `image/svg+xml`
- **Max size**: **5 MB** per file (multer `limits.fileSize`)

## Permission codes (backend contract)

Codes added in seed (in addition to Users/Roles/Permissions):

- `media.read`
- `media.write`
- `media.delete`

These codes protect endpoints via `requirePermission(code)` (same mechanism as Admin Users/Roles/Permissions docs).

## Default seed

File (node): `backend_node/prisma/seed.ts`  
File (.NET): `backend/Infrastructure/BootstrapSeedHostedService.cs`

- Upsert permissions including `media.read`, `media.write`, `media.delete`
- Role `admin`: all permissions (including media)
- Role `viewer`: read-only extended with `media.read` (plus `users.read`, `roles.read`, `permissions.read`)

Command (node):

- `cd backend_node`
- `npm run prisma:seed`

Command (.NET): seed on startup via `BootstrapSeedHostedService`; schema via `backend/sql/migrations/`.

## AuthN/AuthZ (backend)

File (node): `backend_node/src/index.ts`  
File (.NET): `AdminMediaController.cs`, `PublicMediaController.cs` under `backend/Presentation/Controllers/`; `IAdminMediaRepository` / persistence under `backend/Data/`; handlers under `backend/Application/`.

Same behavior as other admin features (see `docs/features/BO-Feature-Admin-Users-Roles-Permissions.md`):

- `POST /auth/login` → UUID token
- `Authorization: Bearer <token>` for `/api/admin/*`
- `requireAuth` + `requirePermission`
- Dev note: in-memory token map (`tokenToUserId`)

## Backend endpoints (public vs admin)

File (node): `backend_node/src/index.ts`  
File (.NET): `AdminMediaController.cs`, `PublicMediaController.cs` under `backend/Presentation/Controllers/`; `IAdminMediaRepository` / persistence under `backend/Data/`; handlers under `backend/Application/`.

### Public (binary read)

- `GET /api/public/media/:id`
  - Response: binary body
  - Headers: `Content-Type` = `mimeType`, `Cache-Control`, `ETag` (basic on `updatedAt`)

Typical site usage:

- `<img src="https://<api>/api/public/media/<id>" alt="..." />`
- Optional: query `?v=<updatedAt>` to bust browser cache after an update.

### Admin (protected CRUD)

All endpoints protected by `requirePermission`:

Media

- `GET /api/admin/media` → `media.read` (paginated list, **without** bytes)
- `POST /api/admin/media` → `media.write`
  - `multipart/form-data`
  - file field: `file`
  - optional fields: `name` (label without extension recommended; default = filename **without** extension), `folder`, `alt`
- `PUT /api/admin/media/:id` → `media.write` (metadata: `name`, `folder`, `alt`); if `name` is sent and empty after trim → **400** (`Name cannot be empty`)
- `DELETE /api/admin/media/:id` → `media.delete`

## Backoffice (React front)

### UI routes

File: `backoffice/src/App.tsx`

- `/configuration/media` → `MediaPage`

### Sidebar menu

File: `backoffice/src/components/organisms/AppSidebar/AppSidebar.tsx`

- **Configuration** section includes **Media** → `/configuration/media`

### AuthContext (token storage)

File: `backoffice/src/contexts/AuthContext.tsx`

- Token in `localStorage` (`backoffice_session`), reused by admin calls

### API services

File: `backoffice/src/services/mediaApi.ts`

- `GET /api/admin/media` with `Authorization: Bearer <token>`
- `POST /api/admin/media` as `FormData` (field `file` + metadata)
- `PUT` / `DELETE` on `/api/admin/media/:id`
- Helper `publicMediaUrl(id, updatedAt?)` to build public display / copy URL

### Media picker modal (reusable)

The backoffice reuses a **media picker modal** across several features (custom pages, banner, etc.) to choose an existing asset.

Expected behavior:

- modal shows the media library (list/grid) to pick an existing item
- if the library is empty or no media matches search/filter context, show an **explicit empty state**
- from that empty state, offer a primary **Add media** CTA (visible without leaving the modal)
- CTA opens the existing upload flow (upload modal or embedded step), then refreshes the library at the end
- after a successful upload, the new media must be selectable immediately in the same flow

UX goals:

- avoid dead ends in forms that depend on a media
- reduce back-and-forth between the current feature and `/configuration/media`
- keep a continuous path: **select if it exists, create otherwise, then select**

### Display name utilities

File: `backoffice/src/utils/mediaDisplayName.ts`

- `filenameStem`: filename stem (upload)
- `mediaNameStemForEdit`: name field prep for edit (strips trailing image extension if present, for legacy data)

### Toast (copy confirmation)

- `backoffice/src/hooks/useToast.ts` — single message, auto-dismiss
- `backoffice/src/components/atoms/Toast/ToastBanner.tsx` — fixed bottom-right banner (`z-[300]`), `role="status"`, `aria-live="polite"`
- On **Copy URL** (list, grid, preview modal): confirmation toast after successful copy; fallback browser (`prompt`) with adapted message

## Backoffice page

### MediaPage

File: `backoffice/src/pages/MediaPage/MediaPage.tsx`

Features:

- **list or thumbnail view**: toggle (Headless UI `Switch`) **List** / **Thumbnails**
- **list (table)**: columns preview (fixed square, `object-contain object-center`), name, folder, type, size, dimensions, actions. **Row click** (except actions column) opens **preview modal**; **Enter** / **Space** when row is focused (`tabIndex`) also open preview. **Actions** column uses `stopPropagation` so preview does not open.
- **thumbnails (grid)**: **square** thumbnails (`aspect-square`), image **fills** the block (`object-cover object-center`). **Click on image area** (full-bleed button) opens preview modal; rest of card (text, actions) does not.
- **preview modal**: large image (`object-contain`), title = name, footer with type / size / dimensions, **Copy URL**, **Close** (no dedicated eye icon in grid or list).
- **upload** (modal): multi-file; per file, **image preview** (blob URL, square) + **display name (no extension)** prefilled; shared optional folder and alt; send `name` in `multipart`; API default if absent = stem of `originalname`
- **edit** (pencil `PencilSquareIcon`): **Edit media** modal with **image preview** above the form (public URL, `object-cover object-center` in a height-limited frame), name (no extension in UI), folder, alt; **Save** calls `PUT /api/admin/media/:id` then **closes modal** and refreshes list; **Cancel** closes without saving
- row/card actions: **edit**, **copy URL** (toast), **delete**
- basic pagination (same pattern as other list pages)

## Quick debug / test

1. Sign in with an account that has `media.write` (or `isAdmin`).
2. Go to `/configuration/media`, upload a PNG/JPEG (≤ 5MB).
3. Verify in the browser:
   - `GET /api/admin/media` returns rows without bytes
   - `GET /api/public/media/<id>` displays the image
4. If 403 on admin:
   - verify the user has `media.read` / `media.write` / `media.delete` in their snapshot or `isAdmin`
5. If upload rejected:
   - verify MIME type and size

## Limitations and next improvements (guidance)

- **DB storage**: convenient for MVP; at high volume consider object storage (S3) + signed URLs, keeping `Media` as reference.
- **No variants** (thumb / large): add if UI needs them (async jobs or generation at upload).
- **SVG**: supported to ease vector illustrations.
- **In-memory auth token** (dev): same limitation as rest of backoffice — migrate to JWT / persistent sessions in production.
