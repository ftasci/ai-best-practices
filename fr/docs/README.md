# Documentation — carte du savoir

Ce fichier est le **point d’entrée** pour humains et agents (IA) : il indique **où trouver quoi** et dans quel ordre charger le contexte. Les chemins sont stables pour le partage de contexte entre sessions et contributeurs.

---

## 1. Ordre de lecture recommandé

| Priorité | Fichier / dossier | Rôle |
|----------|-------------------|------|
| 1 | [`../agent/agent-principal.md`](../agent/agent-principal.md) | Règles globales du projet, stack, conventions, checklists |
| 2 | **Cette carte** (`docs/README.md`) | Index des docs métier et techniques |
| 3 | [`business/`](business/) — vision, glossaire, historique des besoins | Contexte *produit* partagé (Git), avant le détail technique |
| 4 | Docs du **périmètre** de la tâche (ADR, guide, feature — voir ci‑dessous) | Contexte ciblé, évite de tout charger |

Pour une tâche précise : lire l’agent spécialisé depuis `agent/` si besoin (React, API, UX, etc.), puis **uniquement** les documents listés pour la zone touchée.

---

## 2. Contexte métier / produit (vision & historique des besoins)

Emplacement : **`docs/business/`** — voir [`business/README.md`](business/README.md).

| Document | Rôle |
|----------|------|
| [`business/VISION.md`](business/VISION.md) | Objectif global, utilisateurs, périmètre, critères de succès |
| [`business/GLOSSAIRE.md`](business/GLOSSAIRE.md) | Termes métier et définitions |
| [`business/HISTORIQUE-BESOINS.md`](business/HISTORIQUE-BESOINS.md) | Journal daté des changements de besoins (récent en haut) |

**Règle** : toute évolution notable du *quoi* ou du *pourquoi* métier → entrée dans `HISTORIQUE-BESOINS.md` + mise à jour de `VISION.md` ou du glossaire si besoin. Le détail livrable reste dans `docs/features/` (et le *pourquoi technique* dans les ADR).

**Multi‑machine / multi‑agent** : ce dossier est versionné ; après édition, **commit + push** pour propager le contexte. En session IA, citer les chemins lus (convention déjà en section « Directives agents »).

---

## 3. Directives agents (comportement & qualité)

Emplacement : **`agent/`** à la racine du dépôt.

| Fichier | Usage |
|---------|--------|
| `agent-principal.md` | Référence principale ; liens vers les autres |
| `agent-software-engineer.md` | Exécution, tests, qualité |
| `agent-react.md` | Frontend React |
| `agent-api-architect.md` | Couches API, DTOs, résilience |
| `agent-sys-architecture.md` | Architecture système, ADR |
| `agent-ux-ui.md` | UX, parcours, accessibilité |
| `agent-github.md` | Git / PR (si utilisé) |

**Convention multi‑agent** : en début de tâche, indiquer explicitement les chemins déjà consultés (ex. « contexte : `agent/agent-principal.md`, `docs/business/VISION.md`, `docs/architecture/ADR-002-…` ») pour aligner le prochain intervenant.

---

## 4. Décisions d’architecture (ADR)

Emplacement : **`docs/architecture/`**

| Document | Sujet |
|----------|--------|
| `ADR-001-pagination-implementation.md` | Pagination offset, défauts, format de réponse |
| `ADR-002-filters-and-sorting-system.md` | Filtres / tri génériques, sécurité, opérateurs |

**Règle** : toute décision structurante (stack, auth, schéma majeur, pattern API) → **nouvel ADR numéroté** ici, pas seulement un commentaire dans le code.

---

## 5. Guides techniques (mode d’emploi développeur)

Emplacement : **`docs/guides/`** — nommage : `Guide-<Objet>.md`

| Document | Sujet |
|----------|--------|
| [`guides/Guide-Pagination.md`](guides/Guide-Pagination.md) | Middleware, Prisma, format de réponse |
| [`guides/Guide-Filters.md`](guides/Guide-Filters.md) | Query params, config modèles, routes |

**Règle** : le guide décrit le **comment** ; l’ADR le **pourquoi**. Si le code change, mettre à jour le guide **dans le même changement** quand c’est visible pour les consommateurs de l’API.

---

## 6. Fonctionnalités produit

Emplacement : **`docs/features/`** — conserver les conventions de nommage existantes (`BO-Feature-*.md`, etc.)

| Document | Sujet |
|----------|--------|
| [`features/BO-Feature-SSO-Backoffice.md`](features/BO-Feature-SSO-Backoffice.md) | SSO / OIDC Keycloak — spec, env, endpoints |
| [`features/BO-Feature-Media.md`](features/BO-Feature-Media.md) | Médias (backoffice) |
| [`features/SSO_IMPLEMENTATION_GUIDE.md`](features/SSO_IMPLEMENTATION_GUIDE.md) | Implémentation SSO dans le dépôt (complète la fiche SSO) |

**Règle** : nouvelle feature significative → **`docs/features/BO-Feature-<NOM>.md`** (objectif, périmètre, données, endpoints, hors périmètre). Guide d’implémentation détaillé à part si la fiche devient trop longue.

---

## 7. Où placer les futurs documents

| Type | Emplacement |
|------|-------------|
| ADR | `docs/architecture/ADR-00N-titre-kebab-case.md` |
| Guide technique | `docs/guides/Guide-<Objet>.md` |
| Feature backoffice | `docs/features/BO-Feature-<NOM>.md` |
| Feature frontend (public) | `docs/features/FR-Feature-<NOM>.md` |
| JTBD / parcours UX | `docs/ux/<feature>-jtbd.md`, `-journey.md`, `-flow.md` (créer `docs/ux/` si besoin) |
| Vision / glossaire / historique besoins | `docs/business/` (`VISION.md`, `GLOSSAIRE.md`, `HISTORIQUE-BESOINS.md`) |
| Rapport stack / versions | `docs/report/` (maintenu ou régénéré volontairement) |

Éviter la **duplication** : un document **référence**, les autres **pointent** avec un lien Markdown.

---

## 8. Checklist « contexte minimal » (humain ou agent)

Avant d’implémenter une modification :

- [ ] `agent/agent-principal.md` à jour dans ta lecture si la tâche touche stack ou conventions
- [ ] `docs/business/` (vision, glossaire, historique) si la tâche touche le périmètre produit ou une règle métier
- [ ] ADR concerné(s) si tu changes une décision déjà enregistrée (ou prévoir un nouvel ADR)
- [ ] Guide (`docs/guides/`) ou fiche `docs/features/` mis à jour si le comportement ou l’API change
- [ ] En fin de tâche : une phrase dans la PR ou le message de commit sur ce qui a été documenté (ou « doc inchangée, pas d’impact »)

---

## 9. Fichier vivant

Cette carte doit être **ajustée** quand de nouvelles catégories de documents apparaissent (ex. OpenAPI publié, runbooks ops). Préférer **éditer ce README** plutôt que multiplier les « index » ailleurs.
