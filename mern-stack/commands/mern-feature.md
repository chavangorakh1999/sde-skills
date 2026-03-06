---
description: "Build a full MERN feature end-to-end: schema → API → service → React component with state management"
argument-hint: "[feature description, e.g. 'user profile with avatar upload']"
---

## MERN Feature Builder

Build a complete, production-quality MERN feature from database to UI.

### Phase 1 — Schema & Model

Define the Mongoose schema for the feature:
- Identify entities and their relationships (embed vs reference)
- Add required indexes (compound, sparse, TTL as appropriate)
- Include timestamps, soft-delete if needed
- Add schema-level validation and virtuals

### Phase 2 — Repository Layer

Create the data access layer:
- CRUD operations with proper error mapping
- Lean queries (`.lean()`) for read-heavy operations
- Cursor-based pagination for lists
- Map Mongoose errors to domain errors (duplicate key → ConflictError)

### Phase 3 — Service Layer

Implement business logic:
- Validate business rules that go beyond schema validation
- Orchestrate cross-entity operations
- Emit domain events via eventBus for side effects (emails, analytics)
- Keep services free of HTTP concepts (req/res)

### Phase 4 — Express Routes & Controller

Wire up the API:
- RESTful route structure
- Joi request validation middleware
- Authentication + authorization middleware
- Controller: delegate to service, map result to HTTP response
- Use asyncHandler wrapper

### Phase 5 — React Integration

Build the UI:
- React Query hooks for server state (useQuery, useMutation)
- Optimistic updates where UX benefits
- Loading, error, and empty states
- Form handling with React Hook Form + Zod
- Component broken into container + presentational parts

### Phase 6 — Tests

Write the test layer:
- Unit tests for service business logic
- Integration tests for API endpoints with Supertest
- React Testing Library tests for key user interactions

### Output

Provide:
1. **File structure** — paths for all new files
2. **Complete code** for each file with inline comments
3. **Test examples** for service + one API route + one component
4. **Migration notes** — any existing code that needs updating
