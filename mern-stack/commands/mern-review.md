---
description: "Review MERN code for security, performance, architecture, and best practices"
argument-hint: "[code to review or area to focus on, e.g. 'auth flow' or 'paste code here']"
---

## MERN Code Review

Comprehensive review across security, performance, architecture, and best practices.

### Phase 1 — Security Audit

Check every layer:

**Backend:**
- [ ] User input validated with Joi/Zod before any DB query
- [ ] `express-mongo-sanitize` active (NoSQL injection)
- [ ] Passwords stored with bcrypt (rounds >= 12)
- [ ] JWT secrets >= 32 chars, not hardcoded
- [ ] Refresh tokens in httpOnly cookies (not localStorage)
- [ ] Rate limiting on auth and expensive endpoints
- [ ] Helmet.js active with CSP configured
- [ ] CORS restricted to known origins
- [ ] No sensitive data in error responses or logs

**Frontend:**
- [ ] No `dangerouslySetInnerHTML` with user content
- [ ] Access tokens in memory (not localStorage)
- [ ] No secrets or API keys in frontend code

### Phase 2 — Performance Review

**MongoDB:**
- [ ] All query fields in queries have indexes
- [ ] Using `.lean()` on read-only queries
- [ ] No N+1 queries (use `.populate()` or batch fetch)
- [ ] Projections exclude large/unused fields
- [ ] Pagination implemented (no unbounded `.find()`)

**Node.js:**
- [ ] No CPU-blocking operations in event loop (use worker threads)
- [ ] Promises awaited correctly (no fire-and-forget in critical paths)
- [ ] Memory leaks: no accumulating globals, event listeners cleaned up
- [ ] Connection pools sized for concurrency

**React:**
- [ ] No unnecessary re-renders (check with React DevTools Profiler)
- [ ] useEffect dependencies correct (no missing deps, no object deps)
- [ ] Lists have stable `key` props (not index)
- [ ] Large lists virtualized

### Phase 3 — Architecture Review

**Backend structure:**
- [ ] Routes → Controllers → Services → Repositories — no layer skipping
- [ ] Services contain business logic, controllers are thin HTTP adapters
- [ ] No mongoose models imported directly in route handlers
- [ ] Error classes used consistently, not `new Error()` everywhere
- [ ] Config validated at startup (not `process.env.X ?? undefined` scattered)

**Frontend structure:**
- [ ] Server state in React Query (not duplicated in Zustand/Context)
- [ ] Forms use React Hook Form (not manual useState chains)
- [ ] Components split at appropriate granularity (not monolithic pages)
- [ ] API calls centralized in api/ modules (not fetch() in components)

### Phase 4 — Code Quality

- [ ] asyncHandler or express-async-errors used (no unhandled async rejections)
- [ ] Consistent error response format `{ error: { code, message } }`
- [ ] Magic numbers extracted to named constants
- [ ] Duplicate logic extracted (DRY where it adds clarity)
- [ ] No commented-out code
- [ ] Environment-specific config via env vars (not if/else on NODE_ENV)

### Phase 5 — Testing Coverage

- [ ] Service logic has unit tests (happy path + error cases)
- [ ] API routes have integration tests with Supertest
- [ ] UI components have RTL tests for key interactions
- [ ] Test factories used (not hardcoded fixture objects)
- [ ] Tests are independent (no shared mutable state between tests)

### Output Format

```
## MERN Review: [Feature/File/Area]

### CRITICAL (must fix before merge)
- [Issue]: [What's wrong] → [Fix]

### MAJOR (should fix)
- [Issue]: [What's wrong] → [Fix]

### MINOR (consider)
- [Issue]: [Suggestion]

### Positives
- [What was done well]

### Recommended Changes
[Code examples for the most important fixes]
```
