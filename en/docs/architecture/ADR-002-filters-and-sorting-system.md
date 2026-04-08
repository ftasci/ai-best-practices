# ADR-002: Generic dynamic filters and sorting system

## Status
Accepted

## Context
Pagination is in place, but a generic, dynamic filter and sort system that adapts automatically to the Prisma model used is missing.

## Decisions

### 1. Generic architecture based on configuration
**Decision**: Filter system based on declarative per-model configuration.

**Rationale**:
- Reusable across all Prisma models
- Type-safe with TypeScript
- Easy to extend with new models
- Security: strict validation of allowed fields and operators

**Structure**:
```typescript
interface ModelFilterConfig {
  model: string
  allowedFields: FilterableFieldConfig[]
}
```

### 2. Extensible filter operators
**Decision**: Support 13 filter operators depending on field type.

**Supported operators**:
- `eq`, `ne`: Equality / inequality
- `gt`, `gte`, `lt`, `lte`: Numeric/date comparisons
- `contains`, `startsWith`, `endsWith`: Text search (case-insensitive)
- `in`, `notIn`: In list / not in list
- `isNull`, `isNotNull`: Null checks

**Rationale**:
- Covers most SaaS use cases
- Operators suited to field type (string, number, date, boolean, enum)
- Extensible for new operators

### 3. Backend validation and security
**Decision**: Strict validation of fields and operators before building Prisma queries.

**Security**:
- Only fields declared in configuration are accepted
- Only operators allowed for each type are accepted
- Enum values validated when applicable
- Values parsed safely according to type

**Rationale**:
- Prevents SQL injection
- Fine-grained field access control
- Type validation to avoid runtime errors

### 4. Query param format: filters[field][operator]=value
**Decision**: Standardized format for filters in URLs.

**Example**:
```
GET /api/users?filters[email][contains]=test&filters[name][eq]=John&page=1&pageSize=20
```

**Rationale**:
- Standard and readable
- Supports multiple simultaneous filters
- Easy to parse on the backend
- Compatible with standard query params

### 5. Frontend architecture: Atomic Design
**Decision**: Filter components follow Atomic Design.

**Structure**:
- **Atoms**: `Input` (base input)
- **Molecules**: `FilterInput` (field with operator)
- **Organisms**: `FilterPanel` (full filter panel)

**Rationale**:
- Consistent with existing architecture
- Maximum reuse
- Easier to test and maintain

### 6. Integration with pagination
**Decision**: Filters integrate with the existing pagination system.

**Behavior**:
- Filter changes → reset to page 1
- Filters included in pagination key to avoid duplicate requests
- Filters sent with pagination parameters

**Rationale**:
- Consistent user experience
- Optimal performance (single request)
- Unified API

### 7. Default configuration for common models
**Decision**: Default configuration for User and Post in `DEFAULT_FILTER_CONFIGS`.

**Rationale**:
- Faster onboarding
- Configuration examples
- Easy to extend

## Consequences

### Positive
- ✅ Generic reusable system for all models
- ✅ Type-safe with TypeScript
- ✅ Secured by strict validation
- ✅ Extensible for new operators/fields
- ✅ Intuitive UI with Atomic Design
- ✅ Seamless integration with pagination

### Negative
- ⚠️ Configuration must be defined for each new model
- ⚠️ Query param format can be verbose for complex filters

### Mitigations
- Default configuration for common models
- Helper to auto-generate config from Prisma schema (future)

## Implementation

### Backend
1. TypeScript types (`types/filters.ts`)
2. Validation and parsing utilities (`utils/filters.ts`)
3. Express middleware (`middleware/filters.ts`)
4. Integration with Prisma pagination (`utils/prisma-pagination.ts`)

### Frontend
1. TypeScript types (`types/filters.ts`)
2. Atomic Design components:
   - `Input` (atom)
   - `FilterInput` (molecule)
   - `FilterPanel` (organism)
3. Integration in `usePagination` and `DataTable`

## Usage example

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
      { field: 'name', label: 'Name', type: 'string' },
    ]
  }}
/>
```

## References
- Practical guide: [Filters guide](../guides/Guide-Filters.md)
- [Prisma Filtering](https://www.prisma.io/docs/concepts/components/prisma-client/filtering-and-sorting)
- [Express Query Parameters](https://expressjs.com/en/api.html#req.query)
- [Atomic Design Methodology](https://atomicdesign.bradfrost.com/)

## Date
2026-01-23
