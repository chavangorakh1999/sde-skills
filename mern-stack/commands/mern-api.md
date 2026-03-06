---
description: "Design and implement a RESTful Express API: route structure, validation, error handling, pagination, and OpenAPI documentation"
argument-hint: "[resource or domain, e.g. 'blog posts with comments and tags']"
---

## Express API Design

Design and implement a complete, production-quality REST API.

### Phase 1 — Resource Modeling

Define the resource hierarchy:
- Map entities to URL paths
- Identify sub-resources vs top-level resources
- Define HTTP methods for each endpoint
- Determine which resources need nested routes

Standard RESTful pattern:
```
GET    /resources          — list with pagination
POST   /resources          — create
GET    /resources/:id      — get one
PUT    /resources/:id      — replace (rare)
PATCH  /resources/:id      — partial update
DELETE /resources/:id      — delete
```

### Phase 2 — Request Validation Schemas

Write Joi schemas for all request bodies and query parameters:
- Body schemas: required fields, types, string lengths, email format, enums
- Query schemas: pagination (page, limit with defaults), filters, sort
- Param schemas: validate :id is valid MongoDB ObjectId

```javascript
const objectIdSchema = Joi.string().regex(/^[0-9a-fA-F]{24}$/).required();
```

### Phase 3 — Controller + Service Architecture

Implement the layered architecture:

**Routes** — register middleware, delegate to controller
```javascript
router.get('/', validate(listSchema, 'query'), asyncHandler(controller.list));
```

**Controller** — HTTP layer only, no business logic
```javascript
async list(req, res) {
  const result = await service.list(req.query, req.user);
  res.json({ data: result.items, pagination: result.pagination });
}
```

**Service** — all business logic, no HTTP concepts

### Phase 4 — Pagination

Implement cursor-based pagination for production, offset for simplicity:

**Offset** (simple, less efficient):
```javascript
const { page = 1, limit = 20 } = query;
const [items, total] = await Promise.all([
  Model.find(filter).sort(sort).skip((page - 1) * limit).limit(limit).lean(),
  Model.countDocuments(filter)
]);
return { items, pagination: { page, limit, total, totalPages: Math.ceil(total / limit) } };
```

**Cursor-based** (scalable):
Use `_id` or a sorted field as cursor, return `nextCursor` in response.

### Phase 5 — Consistent Response Format

Standardize all responses:
```json
// Success list:
{ "data": [...], "pagination": { "page": 1, "limit": 20, "total": 100 } }

// Success single:
{ "data": { "id": "...", ... } }

// Error:
{ "error": { "code": "VALIDATION_ERROR", "message": "...", "details": [...] } }
```

### Phase 6 — Security Layer

Apply to all routes:
- Rate limiting (general + strict for write operations)
- Authentication middleware on protected routes
- Authorization checks (ownership, roles)
- Request size limits: `express.json({ limit: '10kb' })`

### Phase 7 — OpenAPI Documentation

Generate API spec:
```yaml
# Document each endpoint with:
paths:
  /resources:
    get:
      summary: List resources
      parameters: [page, limit, filter params]
      responses:
        200: { description: Success, schema: ... }
        401: { description: Unauthorized }
```

Use `swagger-jsdoc` + `swagger-ui-express` to serve docs at `/api/docs`.

### Output

1. **Route structure** — complete file listing
2. **Router file** with all routes, middleware chain
3. **Validation schemas** for all endpoints
4. **Controller + service** complete implementations
5. **Response format** examples for success and all error cases
6. **Security middleware** configuration
7. **OpenAPI snippet** for 2-3 key endpoints
