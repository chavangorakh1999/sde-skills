---
name: pr-review
description: "Structured PR review across 7 dimensions: correctness, security, performance, resilience, design, testability, readability. Produces severity-tagged findings. Use when reviewing a pull request or preparing code for review."
---

## PR Review Framework

Systematic code review across 7 dimensions. Each finding includes severity, location, problem, and concrete fix. Not just "this looks wrong" — always say why and how to fix it.

### Context

Code or diff to review: **$ARGUMENTS**

If code is not provided, ask the user to paste the diff or file.

---

### Review Dimensions

Work through each dimension systematically. Don't stop at the first issue — complete all 7.

---

#### Dimension 1: Correctness

Does the code do what it claims to do?

```javascript
// Check for:
// - Off-by-one errors in loops, slice, indexOf
// - Async/await misuse (missing await, swallowed promises)
// - Race conditions (multiple async operations on shared state)
// - Null/undefined dereferencing without guard
// - Type coercion bugs (== vs ===, falsy pitfalls)
// - Integer overflow (JavaScript: safe to Number.MAX_SAFE_INTEGER = 2^53-1)
// - Incorrect regular expressions
// - Logic errors in conditionals (AND vs OR, negation)

// Example bug:
async function processItems(items) {
  items.forEach(async (item) => {  // BUG: forEach ignores returned promises
    await processItem(item);       // these run concurrently, errors are swallowed
  });
}

// Fix:
await Promise.all(items.map(item => processItem(item)));
// or sequentially: for (const item of items) { await processItem(item); }
```

---

#### Dimension 2: Security

Does the code open attack vectors?

```javascript
// OWASP Top 10 checklist:

// A1 - Injection
// SQL: parameterized queries always
db.query(`SELECT * FROM users WHERE id = '${userId}'`);  // CRITICAL: SQL injection
db.query('SELECT * FROM users WHERE id = $1', [userId]); // correct

// NoSQL injection (MongoDB)
User.find({ username: req.body.username });  // MAJOR: $where injection possible
User.find({ username: sanitize(req.body.username) }); // use express-mongo-sanitize

// A2 - Broken Authentication
// - JWT: verify signature AND expiry AND audience
// - Passwords: bcrypt with work factor >= 12, never log
// - Timing attacks: use crypto.timingSafeEqual for token comparison

// A3 - Sensitive Data Exposure
// - Secrets in code? env vars only
// - PII in logs? redact
// - Response includes password hash? Strip it
// - HTTP (not HTTPS) in redirects?

// A5 - Broken Access Control
// - Can user A access user B's resource? Check authorization, not just authentication
// - Missing auth middleware on routes?
router.delete('/users/:id', deleteUser);  // CRITICAL: no auth middleware!
router.delete('/users/:id', authenticate, authorize('admin'), deleteUser); // correct

// A6 - Security Misconfiguration
// - Helmet.js missing? Default Express exposes X-Powered-By header
// - CORS: allow all origins (*) in production?
// - Stack traces in 500 responses?

// A9 - Dependencies
// npm audit —severity high
// Outdated packages with known CVEs
```

---

#### Dimension 3: Performance

Will this be fast enough under load?

```javascript
// N+1 queries — most common performance bug
const posts = await Post.find({ status: 'published' });
for (const post of posts) {
  post.author = await User.findById(post.authorId);  // N queries!
}
// Fix: .populate() in Mongoose, JOIN in SQL, or DataLoader for GraphQL

// Synchronous I/O in async context
const data = fs.readFileSync('file.txt');  // blocks the event loop
const data = await fs.promises.readFile('file.txt'); // correct

// Missing database indexes (infer from query patterns)
// O(n) operations that could be O(1) with right data structure
// Unneeded data transfer (SELECT * when only 2 columns needed)
// Missing pagination on unbounded results
// Regex on large strings without anchors (catastrophic backtracking risk)

// Memory-heavy patterns in Node.js:
// - Accumulating all results in memory before sending (stream instead)
// - Creating large buffers unnecessarily
// - Closure over large objects (prevents GC)
```

---

#### Dimension 4: Resilience

Does the code handle failure gracefully?

```javascript
// External calls without timeout
await fetch(externalServiceUrl);  // MAJOR: hangs indefinitely on slow response
await fetch(externalServiceUrl, { signal: AbortSignal.timeout(5000) }); // 5s timeout

// No retry logic for transient failures
// No circuit breaker for repeatedly-failing dependencies
// Uncaught promise rejections
// Missing error handling in Express middleware (next(err) not called)
// Missing DLQ for message queue consumers
// Missing graceful shutdown handler

// Express error middleware not at end of chain:
app.use(router);
app.use(errorHandler);  // must be LAST, with 4 params: (err, req, res, next)

// Uncaught exception handlers missing:
process.on('uncaughtException', (err) => { logger.fatal(err); process.exit(1); });
process.on('unhandledRejection', (reason) => { logger.error(reason); });
```

---

#### Dimension 5: Design

Does the structure make sense?

```javascript
// Single Responsibility: does this function/class do one thing?
// Open/Closed: can behavior be extended without modifying existing code?
// Dependency Injection: are dependencies hardcoded or injected (testable)?
// Proper abstraction: right level of detail, no leaky implementation details
// Cohesion: related things together, unrelated things separated

// Example violation: a "UserService" that also sends emails and calls Stripe
class UserService {
  async register(data) {
    const user = await this.createUser(data);
    await this.sendWelcomeEmail(user);     // SRP violation: email concerns here
    await this.createStripeCustomer(user); // SRP violation: payment concerns here
    return user;
  }
}

// Better: UserService.register -> emits 'user.registered' event
// EmailService listens and sends email
// BillingService listens and creates Stripe customer
```

---

#### Dimension 6: Testability

Can this code be tested in isolation?

```javascript
// Hard-coded dependencies make testing impossible:
async function getUser(id) {
  return require('../db').query('SELECT * FROM users WHERE id = $1', [id]);
  // Can't test without a real DB connection
}

// Dependency injection makes it testable:
async function getUser(db, id) {
  return db.query('SELECT * FROM users WHERE id = $1', [id]);
  // Test: pass a mock db object
}

// Or use a repository pattern:
class UserRepository {
  constructor(db) { this.db = db; }
  async findById(id) { return this.db.query('...', [id]); }
}
// In tests: new UserRepository(mockDb)

// Other testability issues:
// - Functions do too many things (can't test one path without triggering others)
// - Global state mutation (tests interfere with each other)
// - No clear inputs/outputs (side effects everywhere)
// - Dates/random values hardcoded (should be injectable)
```

---

#### Dimension 7: Readability

Will the next engineer understand this in 6 months?

```javascript
// Naming: intention-revealing names
const d = new Date();                    // bad
const orderCreatedAt = new Date();       // good

const arr = users.filter(u => u.a > 0); // bad
const activeUsers = users.filter(u => u.subscriptionCount > 0); // good

// Magic numbers/strings — use named constants
if (retries > 3) { ... }                         // bad: what is 3?
const MAX_RETRY_ATTEMPTS = 3;
if (retries > MAX_RETRY_ATTEMPTS) { ... }        // good

// Comment the "why", not the "what"
// bad: // increment i by 1
// good: // skip the first result (API always returns a header row)

// Function length: if you can't see the whole function without scrolling, split it
// Nesting depth: > 3 levels deep is a smell (early returns, extract methods)
// Inconsistent patterns: mixing callbacks and async/await in the same file
```

---

### Severity Scale

```
[CRITICAL] — Must fix before merge. Security vulnerability, data loss, crash bug.
[MAJOR]    — Should fix before merge. Performance issue, resilience gap, design problem.
[MINOR]    — Fix soon. Readability, minor correctness, missing test coverage.
[NIT]      — Optional. Style, naming, trivial readability.
```

---

### Output Format

```
## PR Review: [File/Feature Name]

### Summary
[2-3 sentences: overall assessment, critical count, merge recommendation]

### Findings

[CRITICAL] path/to/file.js:42 — [Category]: [Short description]
  Problem: [What's wrong and why it matters]
  Fix:     [Concrete code or approach to fix it]

[MAJOR] path/to/file.js:88 — [Category]: [Short description]
  Problem: ...
  Fix:     ...

[MINOR] path/to/file.js:15 — [Category]: [Short description]
  Problem: ...
  Fix:     ...

[NIT] path/to/file.js:7 — Rename `d` to `createdAt`

### What's Done Well
[1-3 things that are good — balanced reviews build trust]

### Merge Recommendation
[ ] Approve — no blocking issues
[ ] Approve with minor fixes — address [NIT] and [MINOR] items
[ ] Request changes — address [MAJOR] items before merge
[ ] Block — [CRITICAL] issues must be resolved
```
