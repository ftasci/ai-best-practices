# Guide d'utilisation du système de filtres et tri

Ce guide explique comment utiliser le système générique de filtres et tri implémenté dans le projet.

## Vue d'ensemble

Le système de filtres est **générique et dynamique**, ce qui signifie qu'il s'adapte automatiquement selon le modèle Prisma utilisé. Il est intégré avec le système de pagination existant.

## Backend

### Configuration des filtres pour un modèle

Définissez la configuration dans `backend/src/utils/filters.ts` :

```typescript
export const DEFAULT_FILTER_CONFIGS: Record<string, ModelFilterConfig> = {
  User: {
    model: 'User',
    allowedFields: [
      { field: 'id', type: 'string' },
      { field: 'email', type: 'string' },
      { field: 'name', type: 'string' },
      { field: 'createdAt', type: 'date' },
      { field: 'updatedAt', type: 'date' },
    ],
  },
  // Ajoutez d'autres modèles...
}
```

### Utilisation dans une route

```typescript
import { paginationMiddleware } from './middleware/pagination'
import { filtersMiddleware } from './middleware/filters'
import { paginatePrisma } from './utils/prisma-pagination'

app.get(
  '/api/users',
  paginationMiddleware,
  filtersMiddleware('User'), // Spécifiez le nom du modèle
  async (req, res) => {
    if (!req.pagination) {
      return res.status(400).json({ error: 'Pagination required' })
    }

    const result = await paginatePrisma(
      prisma,
      'user',
      {
        ...req.pagination,
        filters: req.filters || [],
      },
      {
        include: { posts: true },
      },
      req.filterConfig // Passe la config pour validation
    )

    res.json(result)
  }
)
```

### Format des query params

Les filtres sont envoyés au format : `filters[field][operator]=value`

**Exemples** :
```
# Filtre simple
GET /api/users?filters[email][eq]=test@example.com

# Filtre avec opérateur "contient"
GET /api/users?filters[name][contains]=John

# Plusieurs filtres (AND implicite)
GET /api/users?filters[email][contains]=test&filters[name][eq]=John

# Filtre avec pagination et tri
GET /api/users?filters[email][contains]=test&page=1&pageSize=20&sortBy=createdAt&order=desc
```

### Opérateurs disponibles

| Opérateur | Description | Types supportés |
|-----------|-------------|------------------|
| `eq` | Égal à | Tous |
| `ne` | Différent de | Tous |
| `gt` | Plus grand que | number, date |
| `gte` | Plus grand ou égal | number, date |
| `lt` | Plus petit que | number, date |
| `lte` | Plus petit ou égal | number, date |
| `contains` | Contient (insensible à la casse) | string |
| `startsWith` | Commence par | string |
| `endsWith` | Se termine par | string |
| `in` | Dans une liste | string, number, enum |
| `notIn` | Pas dans une liste | string, number, enum |
| `isNull` | Est null | Tous |
| `isNotNull` | N'est pas null | Tous |

### Types de champs supportés

- `string` : Texte (opérateurs textuels disponibles)
- `number` : Nombre (opérateurs de comparaison disponibles)
- `boolean` : Booléen (eq, ne uniquement)
- `date` : Date (opérateurs de comparaison disponibles)
- `enum` : Énumération (eq, ne, in, notIn)

## Frontend

### Configuration des filtres

Définissez la configuration des filtres pour votre modèle :

```typescript
import { FilterConfig } from './types/filters'

const userFilterConfig: FilterConfig = {
  fields: [
    {
      field: 'email',
      label: 'Email',
      type: 'string',
      placeholder: 'Rechercher par email...',
    },
    {
      field: 'name',
      label: 'Nom',
      type: 'string',
      placeholder: 'Rechercher par nom...',
    },
    {
      field: 'createdAt',
      label: 'Date de création',
      type: 'date',
    },
    {
      field: 'published',
      label: 'Publié',
      type: 'boolean',
    },
  ],
}
```

### Utilisation avec DataTable

```typescript
import { DataTable } from './components/organisms/DataTable'

<DataTable<User>
  endpoint="/api/users"
  columns={columns}
  filterConfig={userFilterConfig}
  showFilters={true}
/>
```

### Utilisation avec usePagination

```typescript
import { usePagination } from './hooks/usePagination'
import { FilterConfig } from './types/filters'

const filterConfig: FilterConfig = {
  fields: [
    { field: 'email', label: 'Email', type: 'string' },
  ],
}

const {
  data,
  meta,
  loading,
  pagination,
  setFilters,
} = usePagination(
  (params) => fetchPaginated<User>('/api/users', params),
  {
    initialFilters: [],
  }
)

// Mettre à jour les filtres
setFilters([
  { field: 'email', operator: 'contains', value: 'test' },
])
```

### Utilisation avec FilterPanel

```typescript
import { FilterPanel } from './components/organisms/FilterPanel'

<FilterPanel
  config={filterConfig}
  filters={pagination.filters || []}
  onFiltersChange={setFilters}
/>
```

## Exemples complets

### Exemple 1 : Filtre simple par email

**Backend** : Route configurée avec `filtersMiddleware('User')`

**Frontend** :
```typescript
const filterConfig: FilterConfig = {
  fields: [
    { field: 'email', label: 'Email', type: 'string' },
  ],
}

<DataTable<User>
  endpoint="/api/users"
  columns={columns}
  filterConfig={filterConfig}
/>
```

**Résultat** : L'utilisateur peut filtrer par email avec les opérateurs : égal, contient, commence par, etc.

### Exemple 2 : Filtres multiples

**Backend** : Même configuration

**Frontend** :
```typescript
const filterConfig: FilterConfig = {
  fields: [
    { field: 'email', label: 'Email', type: 'string' },
    { field: 'name', label: 'Nom', type: 'string' },
    { field: 'createdAt', label: 'Date de création', type: 'date' },
  ],
}
```

**Résultat** : L'utilisateur peut ajouter plusieurs filtres simultanément (AND implicite).

### Exemple 3 : Filtre avec enum

**Backend** :
```typescript
{
  model: 'Post',
  allowedFields: [
    {
      field: 'status',
      type: 'enum',
      enumValues: ['draft', 'published', 'archived'],
    },
  ],
}
```

**Frontend** :
```typescript
const filterConfig: FilterConfig = {
  fields: [
    {
      field: 'status',
      label: 'Statut',
      type: 'enum',
      enumValues: ['draft', 'published', 'archived'],
    },
  ],
}
```

**Résultat** : Un sélecteur dropdown avec les valeurs enum disponibles.

## Bonnes pratiques

1. **Sécurité** : Toujours utiliser `filtersMiddleware` pour valider les filtres
2. **Configuration** : Définir tous les champs filtrables dans la config
3. **Types** : Utiliser le bon type pour chaque champ (string, number, date, etc.)
4. **Opérateurs** : Laisser les opérateurs par défaut sauf besoin spécifique
5. **UX** : Fournir des labels et placeholders clairs pour les utilisateurs
6. **Performance** : Les filtres sont appliqués côté serveur, pas côté client

## Extension pour nouveaux modèles

Pour ajouter un nouveau modèle :

1. **Backend** : Ajoutez la config dans `DEFAULT_FILTER_CONFIGS`
2. **Backend** : Utilisez `filtersMiddleware('ModelName')` dans vos routes
3. **Frontend** : Créez une `FilterConfig` correspondante
4. **Frontend** : Passez la config à `DataTable` ou `FilterPanel`

## Support

Pour toute question ou problème, consultez :

- [ADR-002 — Filtres et tri](../architecture/ADR-002-filters-and-sorting-system.md) : décisions architecturales
- [Guide Pagination](./Guide-Pagination.md) : paramètres `page`, `pageSize`, tri
- Code source : `backend/src/utils/filters.ts` et `frontend/src/components/organisms/FilterPanel`
