# ADR-002: Système générique de filtres et tri dynamique

## Statut
Accepté

## Contexte
Le système de pagination est en place, mais il manque un système générique et dynamique pour les filtres et le tri qui s'adapte automatiquement selon le modèle Prisma utilisé.

## Décisions

### 1. Architecture générique basée sur la configuration
**Décision** : Système de filtres basé sur une configuration déclarative par modèle.

**Raison** :
- Réutilisable pour tous les modèles Prisma
- Type-safe avec TypeScript
- Facile à étendre avec de nouveaux modèles
- Sécurité : validation stricte des champs et opérateurs autorisés

**Structure** :
```typescript
interface ModelFilterConfig {
  model: string
  allowedFields: FilterableFieldConfig[]
}
```

### 2. Opérateurs de filtrage extensibles
**Décision** : Support de 13 opérateurs de filtrage différents selon le type de champ.

**Opérateurs supportés** :
- `eq`, `ne` : Égalité / Non égalité
- `gt`, `gte`, `lt`, `lte` : Comparaisons numériques/dates
- `contains`, `startsWith`, `endsWith` : Recherche textuelle (insensible à la casse)
- `in`, `notIn` : Dans une liste / Pas dans une liste
- `isNull`, `isNotNull` : Vérification de nullité

**Raison** :
- Couvre la plupart des cas d'usage SaaS
- Opérateurs adaptés selon le type de champ (string, number, date, boolean, enum)
- Extensible pour ajouter de nouveaux opérateurs

### 3. Validation et sécurité côté backend
**Décision** : Validation stricte des champs et opérateurs avant construction des requêtes Prisma.

**Sécurité** :
- Seuls les champs déclarés dans la configuration sont acceptés
- Seuls les opérateurs autorisés pour chaque type sont acceptés
- Validation des valeurs enum si applicable
- Parsing sécurisé des valeurs selon leur type

**Raison** :
- Prévention des injections SQL
- Contrôle d'accès granulaire aux champs
- Validation des types pour éviter les erreurs runtime

### 4. Format des query params : filters[field][operator]=value
**Décision** : Format standardisé pour les filtres dans les URLs.

**Exemple** :
```
GET /api/users?filters[email][contains]=test&filters[name][eq]=John&page=1&pageSize=20
```

**Raison** :
- Standard et lisible
- Supporte plusieurs filtres simultanés
- Facile à parser côté backend
- Compatible avec les query params standards

### 5. Architecture frontend : Atomic Design
**Décision** : Composants de filtres suivant Atomic Design.

**Structure** :
- **Atoms** : `Input` (input de base)
- **Molecules** : `FilterInput` (champ de filtre avec opérateur)
- **Organisms** : `FilterPanel` (panneau complet de filtres)

**Raison** :
- Cohérence avec l'architecture existante
- Réutilisabilité maximale
- Facile à tester et maintenir

### 6. Intégration avec pagination
**Décision** : Les filtres sont intégrés dans le système de pagination existant.

**Comportement** :
- Changement de filtres → Reset à la page 1
- Les filtres sont inclus dans la clé de pagination pour éviter les requêtes dupliquées
- Les filtres sont envoyés avec les paramètres de pagination

**Raison** :
- Expérience utilisateur cohérente
- Performance optimale (une seule requête)
- API unifiée

### 7. Configuration par défaut pour modèles communs
**Décision** : Configuration par défaut pour User et Post dans `DEFAULT_FILTER_CONFIGS`.

**Raison** :
- Facilite le démarrage rapide
- Exemples de configuration
- Peut être étendu facilement

## Conséquences

### Positives
- ✅ Système générique réutilisable pour tous les modèles
- ✅ Type-safe avec TypeScript
- ✅ Sécurisé par validation stricte
- ✅ Extensible pour nouveaux opérateurs/champs
- ✅ UI intuitive avec Atomic Design
- ✅ Intégration transparente avec pagination

### Négatives
- ⚠️ Nécessite de définir la configuration pour chaque nouveau modèle
- ⚠️ Format de query params peut être verbeux pour filtres complexes

### Mitigations
- Configuration par défaut pour modèles communs
- Helper pour générer automatiquement la config depuis le schéma Prisma (futur)

## Implémentation

### Backend
1. Types TypeScript (`types/filters.ts`)
2. Utilitaires de validation et parsing (`utils/filters.ts`)
3. Middleware Express (`middleware/filters.ts`)
4. Intégration avec pagination Prisma (`utils/prisma-pagination.ts`)

### Frontend
1. Types TypeScript (`types/filters.ts`)
2. Composants Atomic Design :
   - `Input` (atom)
   - `FilterInput` (molecule)
   - `FilterPanel` (organism)
3. Intégration dans `usePagination` et `DataTable`

## Exemple d'utilisation

### Backend
```typescript
app.get('/api/users', 
  paginationMiddleware,
  filtersMiddleware('User'),
  async (req, res) => {
    const result = await paginatePrisma(
      prisma,
      'user',
      { ...req.pagination, filters: req.filters || [] },
      { include: { posts: true } },
      req.filterConfig
    )
    res.json(result)
  }
)
```

### Frontend
```typescript
<DataTable<User>
  endpoint="/api/users"
  columns={columns}
  filterConfig={{
    fields: [
      { field: 'email', label: 'Email', type: 'string' },
      { field: 'name', label: 'Nom', type: 'string' },
    ]
  }}
/>
```

## Références
- Guide pratique : [Guide Filtres et tri](../guides/Guide-Filters.md)
- [Prisma Filtering](https://www.prisma.io/docs/concepts/components/prisma-client/filtering-and-sorting)
- [Express Query Parameters](https://expressjs.com/en/api.html#req.query)
- [Atomic Design Methodology](https://atomicdesign.bradfrost.com/)

## Date
2026-01-23
