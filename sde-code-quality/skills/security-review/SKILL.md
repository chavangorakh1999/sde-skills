---
name: security-review
description: "OWASP Top 10 applied to JavaScript/Node.js code, secrets audit, dependency vulnerability scan guidance, secure defaults. Use when reviewing code for security issues or hardening an existing service."
---

## Security Review

Systematic security analysis of JavaScript/Node.js code. Every finding includes attack scenario, impact, and remediation. Security is not a checklist item — it's a threat model applied to real code.

### Context

Code or service to review: **$ARGUMENTS**

---

### OWASP Top 10 — Node.js/JavaScript Lens

#### A1: Injection

**SQL Injection**
```javascript
// CRITICAL — Never concatenate user input into queries
app.get('/users', async (req, res) => {
  const users = await db.query(
    `SELECT * FROM users WHERE name = '${req.query.name}'`  // VULNERABLE
  );
});

// Fix: parameterized queries always
const users = await db.query(
  'SELECT * FROM users WHERE name = $1',
  [req.query.name]  // driver handles escaping
);

// With ORMs (Sequelize, Prisma): use the ORM methods, never raw() with user input
// Sequelize raw query (safe):
User.findAll({ where: { name: req.query.name } }); // parameterized automatically
```

**NoSQL Injection (MongoDB)**
```javascript
// CRITICAL — User input as query operators
app.post('/login', async (req, res) => {
  const user = await User.findOne({
    username: req.body.username,  // attacker sends: { "$gt": "" }
    password: req.body.password   // bypasses password check entirely
  });
});

// Fix 1: express-mongo-sanitize middleware (strips $ and . from user input)
app.use(mongoSanitize());

// Fix 2: explicit string coercion + Joi/Zod validation before querying
const schema = Joi.object({ username: Joi.string().max(50).required() });
const { username } = await schema.validateAsync(req.body);
```

**Command Injection**
```javascript
// CRITICAL — Never pass user input to shell commands
const { exec } = require('child_process');
exec(`ping ${req.query.host}`, callback);  // attacker sends: "8.8.8.8; rm -rf /"

// Fix: use execFile (args array, no shell interpretation)
const { execFile } = require('child_process');
execFile('ping', ['-c', '4', validatedHost], callback);

// Or use a library that wraps the binary safely (e.g., ping npm package)
```

---

#### A2: Broken Authentication

```javascript
// JWT — common mistakes
// CRITICAL: not verifying algorithm (alg:none attack)
jwt.verify(token, secret);  // verify checks algorithm by default — good
jwt.verify(token, secret, { algorithms: ['HS256'] });  // explicit — better

// CRITICAL: not checking expiry (exp claim)
// jwt.verify() checks exp by default — don't disable with ignoreExpiration: true

// MAJOR: weak secret (< 32 bytes)
const JWT_SECRET = 'secret123';  // brute-forceable
const JWT_SECRET = crypto.randomBytes(32).toString('hex');  // 256 bits — good

// MAJOR: timing attack on token comparison
if (token === storedToken) { ... }  // vulnerable: string comparison short-circuits
if (crypto.timingSafeEqual(Buffer.from(token), Buffer.from(storedToken))) { ... }

// Password hashing
// CRITICAL: MD5, SHA1, SHA256 — fast algorithms, NOT for passwords
const hash = crypto.createHash('sha256').update(password).digest('hex');  // WRONG

// Fix: bcrypt, scrypt, argon2
const hash = await bcrypt.hash(password, 12);  // work factor 12 minimum
const valid = await bcrypt.compare(password, hash);

// Session management
// MAJOR: session secret in code
app.use(session({ secret: 'my-session-secret' }));  // WRONG

// Fix: from environment, cryptographically random
app.use(session({
  secret: process.env.SESSION_SECRET,  // 32+ bytes random
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',  // HTTPS only in prod
  sameSite: 'strict',
  maxAge: 1000 * 60 * 60 * 24  // 24 hours
}));
```

---

#### A3: Sensitive Data Exposure

```javascript
// Secrets in code — CRITICAL
const API_KEY = 'sk-live-abc123...';  // never in source code

// Fix: dotenv + .gitignore + secret manager
require('dotenv').config();
const API_KEY = process.env.STRIPE_SECRET_KEY;

// Secrets in logs — MAJOR
logger.info({ user: req.body });  // logs password if body has it
logger.info({ user: { email: req.body.email } });  // log only what you need

// PII in error responses
res.status(500).json({ error: err.message, stack: err.stack });  // CRITICAL: stack trace exposed
res.status(500).json({ error: 'Internal server error', requestId: req.id });  // safe

// HTTPS enforcement
// Check: are there any HTTP redirects to HTTP? Is HSTS set?
app.use((req, res, next) => {
  if (req.protocol === 'http') return res.redirect(301, `https://${req.hostname}${req.url}`);
  next();
});

// Helmet for security headers
const helmet = require('helmet');
app.use(helmet());  // sets X-Frame-Options, X-XSS-Protection, HSTS, etc.
```

---

#### A5: Broken Access Control

```javascript
// Horizontal privilege escalation — most common auth bug
app.get('/users/:id/orders', authenticate, async (req, res) => {
  // CRITICAL: user can request any user's orders by changing :id
  const orders = await Order.find({ userId: req.params.id });
  res.json(orders);
});

// Fix: always authorize against the authenticated user's context
app.get('/users/:id/orders', authenticate, async (req, res) => {
  if (req.user.id !== req.params.id && !req.user.isAdmin) {
    return res.status(403).json({ error: { code: 'FORBIDDEN' } });
  }
  const orders = await Order.find({ userId: req.params.id });
  res.json(orders);
});

// Missing auth on routes — CRITICAL
app.delete('/admin/users/:id', deleteUser);  // no auth!
app.delete('/admin/users/:id', authenticate, requireRole('admin'), deleteUser);

// RBAC check pattern
function requireRole(role) {
  return (req, res, next) => {
    if (!req.user.roles.includes(role)) {
      return res.status(403).json({ error: { code: 'INSUFFICIENT_PERMISSIONS' } });
    }
    next();
  };
}

// Mass assignment protection
// CRITICAL: spreading req.body directly onto a model
const user = await User.findByIdAndUpdate(req.params.id, req.body);
// Attacker can send: { "role": "admin", "isAdmin": true }

// Fix: whitelist allowed fields
const { displayName, bio, avatarUrl } = req.body;  // explicit field extraction
const user = await User.findByIdAndUpdate(req.params.id, { displayName, bio, avatarUrl });
```

---

#### A6: Security Misconfiguration

```javascript
// CORS — too permissive
app.use(cors());  // allows ALL origins — fine for public APIs, wrong for user-data APIs

// Fix: explicit allowed origins
app.use(cors({
  origin: process.env.NODE_ENV === 'production'
    ? ['https://app.example.com']
    : ['http://localhost:3000'],
  credentials: true,  // only if you need cookies/auth headers cross-origin
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));

// Rate limiting — missing on sensitive endpoints
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 10,                    // 10 attempts per IP per 15 minutes
  message: { error: { code: 'RATE_LIMIT_EXCEEDED' } },
  standardHeaders: true,      // return RateLimit headers
  legacyHeaders: false
});

app.post('/auth/login', loginLimiter, loginHandler);

// Error handling — exposing internals
app.use((err, req, res, next) => {
  console.error(err);  // MAJOR: logs full error including request context
  res.status(500).json({ message: err.message });  // MAJOR: exposes internal error
});

// Fix:
app.use((err, req, res, next) => {
  logger.error({ err, requestId: req.id });  // structured log, not console.error
  const status = err.statusCode ?? 500;
  const message = status < 500 ? err.message : 'Internal server error';
  res.status(status).json({ error: { code: err.code ?? 'INTERNAL_ERROR', message, requestId: req.id } });
});
```

---

#### A9: Dependency Vulnerabilities

```bash
# Run these in CI
npm audit --audit-level=high    # fail build on high/critical CVEs
npm outdated                    # identify stale packages

# Automated tools:
# Snyk: snyk test (integrates with GitHub, Slack)
# Dependabot: auto-PRs for dependency updates (free on GitHub)
# Socket.dev: supply chain analysis beyond CVE databases

# Common vulnerable packages to watch:
# - lodash < 4.17.21 (prototype pollution)
# - axios < 0.21.2 (SSRF)
# - jsonwebtoken < 9.0.0 (algorithm confusion)
# - express < 4.18.0 (various)
# - mongoose < 7.x (prototype pollution, ReDoS)

# Lock files: always commit package-lock.json or yarn.lock
# Pin exact versions in production Docker images: npm ci (not npm install)
```

---

### Security Review Output Format

```
## Security Review: [Service/Component]

### Critical Findings (fix before deploy)
[CRITICAL] file.js:42 — SQL Injection via unsanitized query parameter
  Attack: attacker sends ?id=1' OR '1'='1 to dump all users
  Impact: full database read, potential write access
  Fix: parameterize the query: db.query('... WHERE id = $1', [req.query.id])

### Major Findings (fix this sprint)
[MAJOR] ...

### Minor Findings (fix next sprint)
[MINOR] ...

### Dependency Audit
[ ] npm audit output summary
[ ] Known vulnerable packages

### Security Headers Checklist
[ ] helmet() middleware applied
[ ] CORS configured for known origins only
[ ] Rate limiting on auth and sensitive endpoints
[ ] HTTPS enforced in production

### Secrets Audit
[ ] No secrets in source code
[ ] .env in .gitignore
[ ] Environment variables in secret manager
```
