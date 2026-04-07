# Guide d'utilisation de la pagination

Ce guide explique comment utiliser le système de pagination implémenté dans le projet.

## Backend

### Utilisation basique

```typescript
import { paginationMiddleware } from './middleware/pagination'
import { paginatePrisma } from './utils/prisma-pagination'

// Route avec pagination
app.get('/api/users', paginationMiddleware, async (req, res) => {
  if (!req.pagination) {
    return res.status(400).json({ error: 'Pagination parameters required' })
  }

  const result = await paginatePrisma(
    prisma,
    'user',
    req.pagination,
    {
      include: {
        posts: true,
      },
    }
  )

  res.json(result)
})
```

### Paramètres de requête

- `page` : Numéro de page (défaut: 1)
- `pageSize` : Nombre d'éléments par page (défaut: 20, max: 100)
- `sortBy` : Champ de tri (optionnel)
- `order` : Ordre de tri - `asc` ou `desc` (défaut: `asc`)

**Exemple** :
```
GET /api/users?page=2&pageSize=50&sortBy=createdAt&order=desc
```

### Format de réponse

```json
{
  "data": [...],
  "meta": {
    "page": 2,
    "pageSize": 50,
    "total": 150,
    "totalPages": 3,
    "hasNext": true,
    "hasPrevious": true
  }
}
```

### Requêtes personnalisées

Pour des requêtes plus complexes :

```typescript
import { paginateCustomQuery } from './utils/prisma-pagination'

const result = await paginateCustomQuery(
  async (skip, take, orderBy) => {
    return prisma.user.findMany({
      skip,
      take,
      orderBy,
      where: {
        email: { contains: '@example.com' }
      },
      include: { posts: true }
    })
  },
  async () => {
    return prisma.user.count({
      where: {
        email: { contains: '@example.com' }
      }
    })
  },
  req.pagination!
)
```

## Frontend

### Utilisation avec le hook usePagination

```typescript
import { usePagination } from './hooks/usePagination'
import { fetchPaginated } from './utils/api'

function UserList() {
  const {
    data,
    meta,
    loading,
    error,
    pagination,
    setPage,
    setPageSize,
    setSort,
  } = usePagination(
    (params) => fetchPaginated<User>('/api/users', params),
    {
      initialPage: 1,
      initialPageSize: 20,
    }
  )

  if (loading) return <div>Chargement...</div>
  if (error) return <div>Erreur: {error.message}</div>

  return (
    <div>
      {data.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
      
      {meta && (
        <PaginationControls
          meta={meta}
          onPageChange={setPage}
          onPageSizeChange={setPageSize}
        />
      )}
    </div>
  )
}
```

### Utilisation avec DataTable (composant complet)

```typescript
import { DataTable, Column } from './components/organisms/DataTable'

const columns: Column<User>[] = [
  {
    key: 'name',
    label: 'Nom',
    sortable: true,
  },
  {
    key: 'email',
    label: 'Email',
    sortable: true,
  },
  {
    key: 'createdAt',
    label: 'Date',
    sortable: true,
    render: (value) => new Date(value as string).toLocaleDateString('fr-FR'),
  },
]

function UserTable() {
  return (
    <DataTable<User>
      endpoint="/api/users"
      columns={columns}
      emptyMessage="Aucun utilisateur"
    />
  )
}
```

### Composants disponibles

#### Atoms
- `Button` : Bouton de base avec variants (primary, secondary, outline, ghost)
- `Select` : Sélecteur de base avec label et gestion d'erreur

#### Molecules
- `PaginationButton` : Bouton spécialisé pour la pagination avec état actif
- `PageSizeSelector` : Sélecteur de taille de page

#### Organisms
- `PaginationControls` : Contrôles complets de pagination (navigation + sélecteur)
- `DataTable` : Tableau de données avec pagination intégrée

## Exemples complets

### Backend : Route avec pagination et tri

```typescript
app.get('/api/posts', paginationMiddleware, async (req, res) => {
  try {
    if (!req.pagination) {
      return res.status(400).json({ error: 'Pagination required' })
    }

    const result = await paginatePrisma(
      prisma,
      'post',
      req.pagination,
      {
        where: {
          published: true,
        },
        include: {
          author: true,
        },
      }
    )

    res.json(result)
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch posts' })
  }
})
```

### Frontend : Liste personnalisée avec pagination

```typescript
import { usePagination } from './hooks/usePagination'
import { PaginationControls } from './components/organisms/PaginationControls'
import { fetchPaginated } from './utils/api'

function CustomList() {
  const { data, meta, loading, setPage, setPageSize } = usePagination(
    (params) => fetchPaginated<Post>('/api/posts', params)
  )

  return (
    <div>
      <div className="grid gap-4">
        {data.map(post => (
          <div key={post.id}>
            <h3>{post.title}</h3>
            <p>{post.content}</p>
          </div>
        ))}
      </div>
      
      {meta && (
        <PaginationControls
          meta={meta}
          onPageChange={setPage}
          onPageSizeChange={setPageSize}
        />
      )}
    </div>
  )
}
```

## Bonnes pratiques

1. **Toujours utiliser le middleware** : Ajoutez `paginationMiddleware` sur toutes les routes qui retournent des listes
2. **Gérer les erreurs** : Vérifiez toujours `req.pagination` après le middleware
3. **Limiter pageSize** : Le backend limite automatiquement à 100, mais préférez 20-50 pour de meilleures performances
4. **Utiliser Suspense** : Enveloppez les composants avec pagination dans `<Suspense>` pour une meilleure UX
5. **Accessibilité** : Les composants incluent déjà les attributs ARIA nécessaires

## Migration future vers multi-tenant

Quand vous ajouterez le support multi-tenant avec `organisation_id`, vous devrez :

1. Ajouter `organisationId` dans le `where` clause :
```typescript
const result = await paginatePrisma(
  prisma,
  'user',
  req.pagination,
  {
    where: {
      organisationId: req.organisationId, // À extraire du middleware d'auth
    },
  }
)
```

2. Le middleware d'authentification devra extraire `organisationId` et l'ajouter à `req`

## Support

Pour toute question ou problème, consultez :

- [ADR-001 — Pagination](../architecture/ADR-001-pagination-implementation.md) : décisions architecturales
- [Guide Filtres et tri](./Guide-Filters.md) : combiner pagination et filtres sur les listes
- Code source : `backend/src/utils/pagination.ts` et `frontend/src/hooks/usePagination.ts`
