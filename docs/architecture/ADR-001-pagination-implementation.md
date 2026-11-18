# ADR-001: Pagination implementation

## Status
Accepted

## Context
The project is a greenfield shell for multi-tenant SaaS. A solid pagination foundation at backend and frontend is needed before building business features.

## Decisions

### 1. Pagination type: Offset-based
**Decision**: Use offset-based pagination (page/limit) rather than cursor-based.

**Rationale**:
- Simpler to implement and understand
- Allows jumping directly to a specific page
- Sufficient for most SaaS use cases
- Can be extended to cursor-based later if needed

**Alternatives considered**:
- Cursor-based: Better performance for very large lists, but more complex and no direct page navigation

### 2. Default page size: 20
**Decision**: Default page size of 20 items.

**Rationale**:
- Good balance between performance and UX
- Common default in many SaaS apps
- User-configurable (10, 20, 50, 100)

### 3. Backend architecture: Middleware + utilities
**Decision**: Split into validation middleware and reusable utilities.

**Structure**:
- `middleware/pagination.ts`: Parse and validate query params
- `utils/pagination.ts`: Utility functions (metadata calculation, validation)
- `utils/prisma-pagination.ts`: Prisma-specific helpers

**Rationale**:
- Maximum reuse
- Clear separation of concerns
- Easier unit tests
- Type-safe with TypeScript

### 4. Standardized response shape
**Decision**: Uniform shape for all paginated responses.

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

**Rationale**:
- Consistent API
- Easier frontend development
- Complete metadata for UI

### 5. Frontend architecture: Atomic Design
**Decision**: Structure based on Atomic Design.

**Structure**:
- **Atoms**: `Button`, `Select` (base reusable components)
- **Molecules**: `PaginationButton`, `PageSizeSelector` (composed components)
- **Organisms**: `PaginationControls`, `DataTable` (complex components)
- **Hooks**: `usePagination` (reusable logic)

**Rationale**:
- Clear, scalable organization
- Maximum reuse
- Easier maintenance
- Aligned with React best practices

### 6. React 19 hook: usePagination with transitions
**Decision**: Custom hook using `useTransition` for pagination changes.

**Rationale**:
- Uses modern React 19 features
- Non-blocking transitions for better UX
- Suspense support for loading states
- Simple, intuitive API

### 7. Sorting and filtering support
**Decision**: Basic support for `sortBy` and `order` (asc/desc).

**Rationale**:
- Common need for data tables
- Easier data exploration
- Can be extended with richer business filters later

**Security**: Strict validation of sort fields to prevent SQL injection.

### 8. Multi-tenancy: Single database with organisation_id (future)
**Decision**: Plan for a single database with `organisation_id` rather than per-schema/per-database separation.

**Rationale**:
- Simpler operations
- Better performance (no multiple connections)
- Easier horizontal scaling
- Can migrate to PostgreSQL schema separation if stronger isolation is needed

**Note**: Not implemented yet, but architecture prepared for this evolution.

## Consequences

### Positive
- ✅ Solid, reusable pagination foundation
- ✅ Type-safe code with TypeScript
- ✅ Scalable, maintainable architecture
- ✅ Reusable components following Atomic Design
- ✅ Performance optimized with React 19 transitions

### Negative
- ⚠️ Offset-based can be slower on very large lists (>100K rows)
- ⚠️ Middleware must be added on each paginated route

### Mitigations
- For very large lists: migrate to cursor-based if needed
- Middleware: provide a helper to apply easily across routes

## References
- Practical guide: [Pagination guide](../guides/Guide-Pagination.md)
- [Prisma Pagination](https://www.prisma.io/docs/concepts/components/prisma-client/pagination)
- [React 19 useTransition](https://react.dev/reference/react/useTransition)
- [Atomic Design Methodology](https://atomicdesign.bradfrost.com/)

## Date
2026-01-23
