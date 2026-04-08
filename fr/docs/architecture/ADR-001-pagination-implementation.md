# ADR-001: Implémentation de la pagination

## Statut
Accepté

## Contexte
Le projet est une coquille vide pour créer des SaaS multi-tenant. Il est nécessaire d'implémenter une base solide pour la pagination au niveau backend et frontend avant de développer les fonctionnalités métier.

## Décisions

### 1. Type de pagination : Offset-based
**Décision** : Utilisation de la pagination offset-based (page/limit) plutôt que cursor-based.

**Raison** :
- Plus simple à implémenter et comprendre
- Permet la navigation directe vers une page spécifique
- Suffisant pour la plupart des cas d'usage SaaS
- Peut être étendu vers cursor-based plus tard si nécessaire

**Alternatives considérées** :
- Cursor-based : Meilleure performance pour très grandes listes, mais plus complexe et ne permet pas la navigation directe

### 2. Taille de page par défaut : 20
**Décision** : Taille de page par défaut de 20 éléments.

**Raison** :
- Bon équilibre entre performance et UX
- Standard dans de nombreuses applications SaaS
- Configurable par l'utilisateur (10, 20, 50, 100)

### 3. Architecture backend : Middleware + Utilitaires
**Décision** : Séparation en middleware de validation et utilitaires réutilisables.

**Structure** :
- `middleware/pagination.ts` : Parse et valide les query params
- `utils/pagination.ts` : Fonctions utilitaires (calcul métadonnées, validation)
- `utils/prisma-pagination.ts` : Helpers spécifiques Prisma

**Raison** :
- Réutilisabilité maximale
- Séparation des responsabilités
- Facilite les tests unitaires
- Type-safe avec TypeScript

### 4. Format de réponse standardisé
**Décision** : Format uniforme pour toutes les réponses paginées.

```typescript
{
  data: T[],
  meta: {
    page: number,
    pageSize: number,
    total: number,
    totalPages: number,
    hasNext: boolean,
    hasPrevious: boolean
  }
}
```

**Raison** :
- Cohérence API
- Facilite le développement frontend
- Métadonnées complètes pour l'UI

### 5. Architecture frontend : Atomic Design
**Décision** : Structure basée sur Atomic Design.

**Structure** :
- **Atoms** : `Button`, `Select` (composants de base réutilisables)
- **Molecules** : `PaginationButton`, `PageSizeSelector` (composants composés)
- **Organisms** : `PaginationControls`, `DataTable` (composants complexes)
- **Hooks** : `usePagination` (logique réutilisable)

**Raison** :
- Organisation claire et scalable
- Réutilisabilité maximale
- Facilite la maintenance
- Aligné avec les meilleures pratiques React

### 6. Hook React 19 : usePagination avec transitions
**Décision** : Hook personnalisé utilisant `useTransition` pour les changements de pagination.

**Raison** :
- Utilise les fonctionnalités modernes de React 19
- Transitions non-bloquantes pour meilleure UX
- Support de Suspense pour les états de chargement
- API simple et intuitive

### 7. Support du tri et filtrage
**Décision** : Support de base pour `sortBy` et `order` (asc/desc).

**Raison** :
- Besoin courant dans les tableaux de données
- Facilite l'exploration des données
- Peut être étendu avec filtres métier plus tard

**Sécurité** : Validation stricte des champs de tri pour éviter l'injection SQL.

### 8. Multi-tenancy : Base unique avec organisation_id (futur)
**Décision** : Préparer pour base unique avec `organisation_id` plutôt que séparation par schéma/base.

**Raison** :
- Plus simple à gérer opérationnellement
- Meilleure performance (pas de connexions multiples)
- Scalabilité horizontale plus facile
- Peut migrer vers séparation par schéma PostgreSQL si besoin d'isolation renforcée

**Note** : Non implémenté pour l'instant, mais architecture préparée pour cette évolution.

## Conséquences

### Positives
- ✅ Base solide et réutilisable pour la pagination
- ✅ Code type-safe avec TypeScript
- ✅ Architecture scalable et maintenable
- ✅ Composants réutilisables suivant Atomic Design
- ✅ Performance optimisée avec React 19 transitions

### Négatives
- ⚠️ Offset-based peut être moins performant sur très grandes listes (>100K éléments)
- ⚠️ Nécessite d'ajouter le middleware sur chaque route paginée

### Mitigations
- Pour très grandes listes : Migrer vers cursor-based si nécessaire
- Middleware : Créer un helper pour appliquer facilement sur plusieurs routes

## Références
- Guide pratique : [Guide Pagination](../guides/Guide-Pagination.md)
- [Prisma Pagination](https://www.prisma.io/docs/concepts/components/prisma-client/pagination)
- [React 19 useTransition](https://react.dev/reference/react/useTransition)
- [Atomic Design Methodology](https://atomicdesign.bradfrost.com/)

## Date
2026-01-23
