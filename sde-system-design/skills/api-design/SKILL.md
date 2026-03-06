---
name: api-design
description: "REST/GraphQL/gRPC API contracts with versioning, auth, pagination, error formats, and rate limiting. Use when designing a new API or reviewing an existing one for production readiness."
---

## API Design

Design APIs that are consistent, versioned, secure, and easy to consume. A good API is a product — its consumers are developers, and their DX matters.

### Context

API to design: **$ARGUMENTS**

---

### Step 1: Choose the Right Protocol

**REST** — default choice. Use when:
- Public API or external integrations
- CRUD resources with clear noun-based structure
- Team is familiar, tooling is abundant

**GraphQL** — use when:
- Multiple consumers with wildly different data needs (mobile vs web)
- Over-fetching is a real problem (network-constrained clients)
- Rapid iteration on frontend data requirements
- Warning: n+1 query problem requires DataLoader; not great for file uploads; harder to cache

**gRPC** — use when:
- Internal service-to-service communication
- Low latency, high throughput (binary protocol, HTTP/2 multiplexing)
- Strong typing across polyglot services (Protobuf schema)
- Warning: not browser-native (needs grpc-web proxy); harder to debug

---

### Step 2: URL Structure (REST)

Resources are nouns, not verbs. HTTP methods are the verbs.

```
# Good
GET    /api/v1/users           — list users
POST   /api/v1/users           — create user
GET    /api/v1/users/:id       — get user
PUT    /api/v1/users/:id       — replace user (full update)
PATCH  /api/v1/users/:id       — partial update
DELETE /api/v1/users/:id       — delete user

GET    /api/v1/users/:id/posts — get posts belonging to user

# Bad (verbs in URLs)
POST /api/v1/createUser
GET  /api/v1/getUser?id=123
POST /api/v1/users/delete
```

**Versioning:**
- URL path (`/v1/`) — most visible, easy to route, breaks bookmarks
- Header (`API-Version: 2024-01`) — cleaner URLs, harder to test in browser
- Query param (`?version=2`) — worst option, pollutes queries
- Recommendation: URL path for public APIs, header for internal APIs

---

### Step 3: Request/Response Contract

Define explicit contracts. No "just look at the code" — document each endpoint.

```javascript
// POST /api/v1/users
// Creates a new user account

// Request
{
  "email": "alice@example.com",       // required, unique, max 255 chars
  "password": "...",                   // required, min 8 chars (never log or return)
  "displayName": "Alice",             // required, max 50 chars
  "role": "member"                    // optional, enum: ["member", "admin"], default: "member"
}

// Success Response — 201 Created
{
  "id": "usr_01H2X...",               // stable, opaque identifier
  "email": "alice@example.com",
  "displayName": "Alice",
  "role": "member",
  "createdAt": "2024-01-15T10:30:00Z" // ISO 8601 UTC
}

// Never return: password hash, internal IDs if using UUIDs, PII you don't need to expose
```

**ID strategy:**
- UUID v4: random, no info leakage, globally unique
- ULID/KSUID: sortable + random (better for DB index locality than UUID v4)
- Prefixed IDs (`usr_`, `ord_`): self-documenting, easier debugging
- Auto-increment integers: avoid for public APIs (exposes count, easy to enumerate)

**Timestamps:** always ISO 8601 UTC (`2024-01-15T10:30:00Z`), never Unix epoch in responses (unreadable)

---

### Step 4: Error Format

Consistent error responses are as important as consistent success responses.

```javascript
// Standard error envelope
{
  "error": {
    "code": "VALIDATION_ERROR",        // machine-readable, snake_case
    "message": "Validation failed",    // human-readable summary
    "details": [                       // optional array for field-level errors
      {
        "field": "email",
        "message": "Invalid email format"
      },
      {
        "field": "password",
        "message": "Must be at least 8 characters"
      }
    ],
    "requestId": "req_01H2X..."        // correlate with server logs
  }
}

// HTTP Status codes to use consistently:
// 200 OK              — success with body
// 201 Created         — resource created, include Location header
// 204 No Content      — success, no body (DELETE)
// 400 Bad Request     — client error, validation failure
// 401 Unauthorized    — not authenticated (missing/invalid token)
// 403 Forbidden       — authenticated but not authorized
// 404 Not Found       — resource doesn't exist
// 409 Conflict        — unique constraint violation, optimistic lock failure
// 422 Unprocessable   — semantically invalid (business rule violation)
// 429 Too Many Req    — rate limited, include Retry-After header
// 500 Internal Error  — server error (never expose stack traces)
```

---

### Step 5: Authentication

```javascript
// JWT Bearer Token (most common for REST APIs)
// Header: Authorization: Bearer eyJhbGci...

// Validate on every request:
// 1. Signature valid (HMAC SHA-256 or RSA)
// 2. Not expired (exp claim)
// 3. Audience matches (aud claim) — prevent token reuse across services
// 4. Issuer matches (iss claim)

// Express.js middleware pattern:
const authenticate = async (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: { code: 'MISSING_TOKEN' } });

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET, {
      audience: 'sde-skills-api',
      issuer: 'sde-skills-auth'
    });
    req.user = payload;
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: { code: 'TOKEN_EXPIRED' } });
    }
    return res.status(401).json({ error: { code: 'INVALID_TOKEN' } });
  }
};
```

**API Keys** (for service-to-service or developer API access):
- Store hashed (SHA-256), never plaintext
- Prefix with service identifier: `sk_live_...`, `sk_test_...`
- Include key ID so you can revoke without knowing the key value

---

### Step 6: Pagination

```javascript
// Cursor-based (recommended for large, real-time datasets)
// GET /api/v1/posts?cursor=eyJpZCI6MTIzfQ&limit=20

// Response
{
  "data": [ /* 20 posts */ ],
  "pagination": {
    "nextCursor": "eyJpZCI6MTQzfQ",  // null if no more results
    "hasMore": true
  }
}

// Cursor implementation (opaque base64 of {id, timestamp})
// Stable across inserts/deletes; works with time-series data
// Cannot jump to page N directly

// Offset-based (OK for admin interfaces, static data)
// GET /api/v1/users?page=3&pageSize=20
// Suffers from "page drift" on live data (inserts shift pages)

// Keyset pagination (efficient for sorted, indexed columns)
// GET /api/v1/posts?after_id=143&limit=20
// SELECT * FROM posts WHERE id > 143 ORDER BY id LIMIT 20
// Index-friendly, no OFFSET scan penalty
```

---

### Step 7: Rate Limiting

```javascript
// Response headers (always include these)
// X-RateLimit-Limit: 1000        — requests allowed per window
// X-RateLimit-Remaining: 847     — requests left in current window
// X-RateLimit-Reset: 1705318800  — Unix epoch when window resets
// Retry-After: 47                — seconds until client may retry (on 429)

// Tiers example
const rateLimits = {
  anonymous:    { rpm: 60,    burst: 10  },
  authenticated: { rpm: 1000,  burst: 100 },
  premium:      { rpm: 10000, burst: 1000 }
};

// Strategies:
// Token bucket     — smooth traffic, allows bursts up to bucket size
// Fixed window     — simple, can be gamed at window boundary (double requests)
// Sliding window   — accurate, more memory (log of each request timestamp)
// Sliding window counter — approximation, low memory, good accuracy

// Apply rate limits per:
// - IP (for anonymous)
// - User ID (for authenticated)
// - API key (for developer APIs)
// - Endpoint (stricter for expensive operations like /search)
```

---

### Step 8: Idempotency

```javascript
// For POST operations that must not double-execute (payments, sends, creates):
// Client sends Idempotency-Key header on first attempt
// Server stores (key -> response) for 24h
// Identical key returns same response without re-executing

// POST /api/v1/payments
// Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

// Implementation:
const idempotencyMiddleware = async (req, res, next) => {
  const key = req.headers['idempotency-key'];
  if (!key) return next();  // idempotency is optional on this endpoint

  const cached = await redis.get(`idem:${key}`);
  if (cached) {
    const { status, body } = JSON.parse(cached);
    return res.status(status).json(body);  // replay stored response
  }

  // Intercept response to cache it
  const originalJson = res.json.bind(res);
  res.json = (body) => {
    redis.setex(`idem:${key}`, 86400, JSON.stringify({ status: res.statusCode, body }));
    return originalJson(body);
  };
  next();
};
```

---

### Output Format

```
## API Design: [Resource/Service Name]

### Protocol Choice
[REST / GraphQL / gRPC with rationale]

### Endpoints
| Method | Path | Description | Auth |

### Request/Response Contracts
[For each endpoint: request body, success response, error responses]

### Error Codes
[Full list of machine-readable error codes]

### Auth Mechanism
[JWT / API Key / OAuth2 — details]

### Pagination Strategy
[Cursor / offset / keyset — with example]

### Rate Limits
[Tiers and limits]

### Breaking Change Policy
[How you'll handle versioning and deprecation]
```
