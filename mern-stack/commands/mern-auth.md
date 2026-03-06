---
description: "Implement complete JWT authentication for a MERN app: register, login, refresh, logout, protected routes in Express and React"
argument-hint: "[any specific requirements, e.g. 'Google OAuth', 'email verification', 'remember me']"
---

## MERN Auth Implementation

Build production-grade JWT auth with access/refresh token rotation.

### Phase 1 — Token Strategy

Define the token architecture:
- Access token: short-lived (15min), signed with ACCESS_SECRET, stored in memory
- Refresh token: long-lived (7d), signed with REFRESH_SECRET, stored in httpOnly cookie
- Refresh token rotation: invalidate old on use, issue new one
- Token family tracking in Redis to detect reuse attacks

### Phase 2 — Backend Auth Routes

Implement the auth API:

```
POST /api/v1/auth/register   — create account, send verification email
POST /api/v1/auth/login      — validate creds, issue token pair
POST /api/v1/auth/refresh    — rotate refresh token, issue new access token
POST /api/v1/auth/logout     — invalidate refresh token
GET  /api/v1/auth/me         — return current user (authenticated)
```

For each route:
- Request validation with Joi
- Rate limiting (especially login/register)
- Proper cookie config (httpOnly, Secure, SameSite=strict)
- Consistent timing on login to prevent username enumeration

### Phase 3 — Auth Middleware

```
authenticate   — verify access token, attach req.user
authorize(...roles) — check role, return 403 if insufficient
optionalAuth   — attach user if token present, but don't require it
```

### Phase 4 — User Model

Design the User schema:
- Email (unique, lowercase index)
- Password (hashed, excluded from default queries with select: false)
- Role enum
- Email verification fields
- Last login timestamp
- refreshToken field (or store in Redis for multi-device)

### Phase 5 — React Auth Layer

Build the frontend auth:
- AuthContext with useReducer for status machine (loading/authenticated/unauthenticated)
- Axios instance with request interceptor to attach Bearer token
- Response interceptor to auto-refresh on 401 (with queuing of parallel requests)
- useAuth() hook for components
- ProtectedRoute component for react-router-dom
- Logout clears token state + calls logout endpoint

### Phase 6 — Security Checklist

Verify:
- [ ] bcrypt rounds >= 12
- [ ] Refresh token cookie is httpOnly, Secure, SameSite=strict
- [ ] Old refresh tokens invalidated on rotation
- [ ] Login endpoint rate-limited (5 req / 15min per IP)
- [ ] JWT secrets are random 32+ byte values
- [ ] Access tokens excluded from logs

### Output

Provide complete, working code for all phases with explanation of security decisions.
