---
description: RESTful API contract with auth, rate limits, pagination, error schema, and versioning
argument-hint: "<resource name and key operations>"
---

# /design-api -- API Contract Design

Design a complete, production-ready API contract. Chains api-design skill with a security review lens.

## Invocation

```
/design-api User resource — registration, login, profile, soft delete
/design-api Product catalog — CRUD, search, filtering, cursor pagination
/design-api Payment — initiate, capture, refund with idempotency
/design-api                    # asks what resource to design
```

## Workflow

### Step 1: Understand the Resource

Extract:
- What resource(s) are we designing?
- Who are the consumers? (web, mobile, third-party developers?)
- What operations are needed? (CRUD + custom actions)
- Any existing APIs this must be consistent with?
- Authentication model? (JWT, API key, OAuth2, session)

### Step 2: Design Endpoints

Apply **api-design** skill:

For each endpoint:
- HTTP method + path (RESTful noun-based URLs)
- Authentication required? (yes/no/scope)
- Request body schema (with field types, required/optional, validation rules)
- Success response (status code + body)
- Error responses (status codes + machine-readable error codes)
- Rate limit tier

```
POST /api/v1/users
PUT  /api/v1/users/:id
GET  /api/v1/users/:id
GET  /api/v1/users?search=alice&cursor=...&limit=20
DELETE /api/v1/users/:id
```

### Step 3: Error Contract

Define the full error code list for this resource — not just HTTP status codes:

```
VALIDATION_ERROR      — 400, field-level details in `details` array
EMAIL_ALREADY_EXISTS  — 409, email taken
INVALID_CREDENTIALS   — 401, wrong email/password (don't say which is wrong)
TOKEN_EXPIRED         — 401, refresh token
TOKEN_INVALID         — 401, bad signature
NOT_FOUND             — 404, resource doesn't exist
FORBIDDEN             — 403, authenticated but not authorized
RATE_LIMIT_EXCEEDED   — 429, include Retry-After
INTERNAL_ERROR        — 500, never expose details to client
```

### Step 4: Pagination

Choose the right pagination strategy:
- **Cursor**: real-time feeds, large datasets, unknown total count
- **Keyset**: sorted by indexed column, no OFFSET penalty
- **Offset**: admin views, static data, user expects "page 3 of 10"

### Step 5: Security Review

Check the API design against:
- [ ] All state-changing endpoints require authentication
- [ ] Authorization: can user A modify user B's resource? (horizontal privilege escalation)
- [ ] No sensitive data in URLs (tokens, passwords, SSNs)
- [ ] No internal IDs exposed if sequential (use UUIDs or opaque cursors)
- [ ] Rate limiting on auth endpoints (prevent brute force)
- [ ] Idempotency keys on non-idempotent POST operations (payments, sends)
- [ ] Response does not include password hashes, internal fields, PII not needed by client

### Step 6: OpenAPI Sketch

Produce a skeleton OpenAPI 3.0 spec for the key endpoints:

```yaml
openapi: 3.0.0
info:
  title: [Resource] API
  version: 1.0.0
paths:
  /api/v1/[resource]:
    post:
      summary: Create [resource]
      requestBody: ...
      responses: ...
```

### Step 7: Offer Next Steps

- "Should I design the data model behind this API? -> `/model-data [domain]`"
- "Want a security review of this API? -> `/review-code [api file]`"
- "Should I write tests for this API? -> `/test-plan [api description]`"
- "Want the full system design? -> `/design [system name]`"

## Output

```
## API Design: [Resource Name]

### Base URL & Versioning
[Base URL, versioning strategy, deprecation policy]

### Authentication
[Mechanism, token format, refresh strategy]

### Endpoints

#### POST /api/v1/[resource]
**Auth:** Required
**Rate Limit:** [tier]
**Request:**
  ```json
  { "field": "type — description" }
  ```
**Response 201:**
  ```json
  { ... }
  ```
**Errors:** [codes]

[Repeat for each endpoint]

### Error Code Reference
| Code | HTTP Status | Description |

### Pagination
[Strategy with example request/response]

### Rate Limits
| Tier | Limit | Window |

### Security Checklist
[Checked items]

### OpenAPI Skeleton
[YAML]
```
