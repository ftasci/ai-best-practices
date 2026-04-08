# Pagination usage guide

This guide explains how to use the pagination system implemented in the project.

## Backend

### Basic usage

```typescript
import { paginationMiddleware } from './middleware/pagination'
import { paginatePrisma } from './utils/prisma-pagination'

// Paginated route
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

### Request parameters

- `page`: Page number (default: 1)
- `pageSize`: Items per page (default: 20, max: 100)
- `sortBy`: Sort field (optional)
- `order`: Sort order — `asc` or `desc` (default: `asc`)

**Example**:
```
GET /api/users?page=2&pageSize=50&sortBy=createdAt&order=desc
```

### Response shape

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

### Custom queries

For more complex queries:

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

### Using the usePagination hook

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

  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>

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

### Using DataTable (full component)

```typescript
import { DataTable, Column } from './components/organisms/DataTable'

const columns: Column<User>[] = [
  {
    key: 'name',
    label: 'Name',
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
    render: (value) => new Date(value as string).toLocaleDateString('en-GB'),
  },
]

function UserTable() {
  return (
    <DataTable<User>
      endpoint="/api/users"
      columns={columns}
      emptyMessage="No users"
    />
  )
}
```

### Available components

#### Atoms
- `Button`: Base button with variants (primary, secondary, outline, ghost)
- `Select`: Base select with label and error handling

#### Molecules
- `PaginationButton`: Pagination-specific button with active state
- `PageSizeSelector`: Page size selector

#### Organisms
- `PaginationControls`: Full pagination controls (navigation + selector)
- `DataTable`: Data table with built-in pagination

## Full examples

### Backend: Route with pagination and sorting

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

### Frontend: Custom list with pagination

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

## Good practices

1. **Always use the middleware**: Add `paginationMiddleware` on all routes that return lists
2. **Handle errors**: Always check `req.pagination` after the middleware
3. **Limit pageSize**: The backend caps at 100, but prefer 20–50 for better performance
4. **Use Suspense**: Wrap paginated components in `<Suspense>` for better UX
5. **Accessibility**: Components already include necessary ARIA attributes

## Future migration to multi-tenant

When you add multi-tenant support with `organisation_id`, you will need to:

1. Add `organisationId` to the `where` clause:
```typescript
const result = await paginatePrisma(
  prisma,
  'user',
  req.pagination,
  {
    where: {
      organisationId: req.organisationId, // From auth middleware
    },
  }
)
```

2. The authentication middleware must extract `organisationId` and attach it to `req`

## Support

For questions or issues, see:

- [ADR-001 — Pagination](../architecture/ADR-001-pagination-implementation.md): architectural decisions
- [Filters guide](./Guide-Filters.md): combining pagination and filters on lists
- Source: `backend/src/utils/pagination.ts` and `frontend/src/hooks/usePagination.ts`
