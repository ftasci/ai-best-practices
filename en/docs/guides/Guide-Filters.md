# Filters and sorting usage guide

This guide explains how to use the generic filter and sort system implemented in the project.

## Overview

The filter system is **generic and dynamic**: it adapts automatically to the Prisma model used. It integrates with the existing pagination system.

## Backend

### Filter configuration for a model

Define configuration in `backend/src/utils/filters.ts`:

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
  // Add other models...
}
```

### Using in a route

```typescript
import { paginationMiddleware } from './middleware/pagination'
import { filtersMiddleware } from './middleware/filters'
import { paginatePrisma } from './utils/prisma-pagination'

app.get(
  '/api/users',
  paginationMiddleware,
  filtersMiddleware('User'), // Model name
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
      req.filterConfig // Pass config for validation
    )

    res.json(result)
  }
)
```

### Query param format

Filters are sent as: `filters[field][operator]=value`

**Examples**:
```
# Simple filter
GET /api/users?filters[email][eq]=test@example.com

# Contains operator
GET /api/users?filters[name][contains]=John

# Multiple filters (implicit AND)
GET /api/users?filters[email][contains]=test&filters[name][eq]=John

# Filters with pagination and sort
GET /api/users?filters[email][contains]=test&page=1&pageSize=20&sortBy=createdAt&order=desc
```

### Available operators

| Operator | Description | Supported types |
|----------|-------------|-----------------|
| `eq` | Equals | All |
| `ne` | Not equals | All |
| `gt` | Greater than | number, date |
| `gte` | Greater or equal | number, date |
| `lt` | Less than | number, date |
| `lte` | Less or equal | number, date |
| `contains` | Contains (case-insensitive) | string |
| `startsWith` | Starts with | string |
| `endsWith` | Ends with | string |
| `in` | In list | string, number, enum |
| `notIn` | Not in list | string, number, enum |
| `isNull` | Is null | All |
| `isNotNull` | Is not null | All |

### Supported field types

- `string`: Text (textual operators available)
- `number`: Number (comparison operators available)
- `boolean`: Boolean (eq, ne only)
- `date`: Date (comparison operators available)
- `enum`: Enumeration (eq, ne, in, notIn)

## Frontend

### Filter configuration

Define filter configuration for your model:

```typescript
import { FilterConfig } from './types/filters'

const userFilterConfig: FilterConfig = {
  fields: [
    {
      field: 'email',
      label: 'Email',
      type: 'string',
      placeholder: 'Search by email...',
    },
    {
      field: 'name',
      label: 'Name',
      type: 'string',
      placeholder: 'Search by name...',
    },
    {
      field: 'createdAt',
      label: 'Created at',
      type: 'date',
    },
    {
      field: 'published',
      label: 'Published',
      type: 'boolean',
    },
  ],
}
```

### Using with DataTable

```typescript
import { DataTable } from './components/organisms/DataTable'

<DataTable<User>
  endpoint="/api/users"
  columns={columns}
  filterConfig={userFilterConfig}
  showFilters={true}
/>
```

### Using with usePagination

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

// Update filters
setFilters([
  { field: 'email', operator: 'contains', value: 'test' },
])
```

### Using with FilterPanel

```typescript
import { FilterPanel } from './components/organisms/FilterPanel'

<FilterPanel
  config={filterConfig}
  filters={pagination.filters || []}
  onFiltersChange={setFilters}
/>
```

## Full examples

### Example 1: Simple email filter

**Backend**: Route configured with `filtersMiddleware('User')`

**Frontend**:
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

**Result**: Users can filter by email with operators: equals, contains, starts with, etc.

### Example 2: Multiple filters

**Backend**: Same configuration

**Frontend**:
```typescript
const filterConfig: FilterConfig = {
  fields: [
    { field: 'email', label: 'Email', type: 'string' },
    { field: 'name', label: 'Name', type: 'string' },
    { field: 'createdAt', label: 'Created at', type: 'date' },
  ],
}
```

**Result**: Users can add several filters at once (implicit AND).

### Example 3: Enum filter

**Backend**:
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

**Frontend**:
```typescript
const filterConfig: FilterConfig = {
  fields: [
    {
      field: 'status',
      label: 'Status',
      type: 'enum',
      enumValues: ['draft', 'published', 'archived'],
    },
  ],
}
```

**Result**: Dropdown with available enum values.

## Good practices

1. **Security**: Always use `filtersMiddleware` to validate filters
2. **Configuration**: Declare all filterable fields in config
3. **Types**: Use the correct type per field (string, number, date, etc.)
4. **Operators**: Keep default operators unless you need something specific
5. **UX**: Provide clear labels and placeholders for users
6. **Performance**: Filters are applied server-side, not client-side

### Extending for new models

To add a new model:

1. **Backend**: Add config in `DEFAULT_FILTER_CONFIGS`
2. **Backend**: Use `filtersMiddleware('ModelName')` in routes
3. **Frontend**: Create a matching `FilterConfig`
4. **Frontend**: Pass config to `DataTable` or `FilterPanel`

## Support

For questions or issues, see:

- [ADR-002 — Filters and sorting](../architecture/ADR-002-filters-and-sorting-system.md): architectural decisions
- [Pagination guide](./Guide-Pagination.md): `page`, `pageSize`, sorting parameters
- Source: `backend/src/utils/filters.ts` and `frontend/src/components/organisms/FilterPanel`
