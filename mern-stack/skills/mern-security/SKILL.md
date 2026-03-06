---
name: mern-security
description: "MERN stack security: NoSQL injection prevention, XSS, CSRF, secure headers, input sanitization, dependency auditing, secrets management. Use when securing or auditing a MERN application."
---

## MERN Security

### Context

Security concern or area to review: **$ARGUMENTS**

---

### NoSQL Injection Prevention

```javascript
// Vulnerability: user-controlled operators in MongoDB queries
// Attack: POST /login { "email": { "$gt": "" }, "password": { "$gt": "" } }
// This matches ALL users if you pass objects directly to Mongoose

// Anti-pattern:
async function login(req, res) {
  const user = await User.findOne({ email: req.body.email });  // email could be object!
}

// Fix 1: Joi/Zod schema validation strips non-string values
const loginSchema = Joi.object({
  email: Joi.string().email().required(),    // Joi rejects objects
  password: Joi.string().required()
});

// Fix 2: express-mongo-sanitize middleware (belt-and-suspenders)
import mongoSanitize from 'express-mongo-sanitize';
app.use(mongoSanitize({
  replaceWith: '_',  // replace $ and . with underscore
  onSanitize: ({ req, key }) => {
    logger.warn({ key, ip: req.ip }, 'NoSQL injection attempt blocked');
  }
}));

// Fix 3: explicit type coercion
async function login(email, password) {
  // Ensure email is string before query
  const user = await User.findOne({ email: String(email) });
  // ...
}
```

---

### XSS Prevention

```javascript
// Server-side: never return raw HTML from user input
// React prevents XSS by default — never use dangerouslySetInnerHTML with user content

// WRONG:
function Comment({ text }) {
  return <div dangerouslySetInnerHTML={{ __html: text }} />;  // XSS!
}

// CORRECT:
function Comment({ text }) {
  return <div>{text}</div>;  // React escapes automatically
}

// If you MUST render HTML (e.g., rich text), sanitize first
import DOMPurify from 'dompurify';

function RichTextContent({ html }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'b', 'i', 'em', 'strong', 'a', 'ul', 'ol', 'li'],
    ALLOWED_ATTR: ['href'],
    FORBID_SCRIPTS: true
  });
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}

// Server-side output encoding for non-React contexts
import escapeHtml from 'escape-html';
const safeText = escapeHtml(userInput);

// Content Security Policy — prevents inline scripts
app.use((req, res, next) => {
  res.setHeader('Content-Security-Policy',
    "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:;"
  );
  next();
});
```

---

### Secure HTTP Headers

```javascript
// Use Helmet.js — sets 15+ security headers
import helmet from 'helmet';

app.use(helmet({
  // Already enabled by default — these are explicit for clarity:
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],  // tighten if possible
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'https://api.stripe.com'],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
    }
  },
  crossOriginEmbedderPolicy: false,  // may break if you load cross-origin resources
}));

// CORS — restrict to known origins
import cors from 'cors';

const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') ?? [];

app.use(cors({
  origin: (origin, callback) => {
    // Allow non-browser requests (Postman, server-to-server) only in dev
    if (!origin && process.env.NODE_ENV !== 'production') return callback(null, true);
    if (allowedOrigins.includes(origin)) return callback(null, true);
    callback(new Error(`CORS: origin ${origin} not allowed`));
  },
  credentials: true,  // allow cookies
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Correlation-Id'],
}));
```

---

### CSRF Protection

```javascript
// CSRF matters when using cookie-based auth
// Method 1: SameSite cookie (simplest, covers most cases)
res.cookie('refreshToken', token, {
  httpOnly: true,
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'strict',   // 'strict' prevents cross-site requests entirely
  maxAge: 7 * 24 * 60 * 60 * 1000,
  path: '/api/auth',    // limit cookie scope
});

// Method 2: Double-submit cookie pattern (for SameSite=lax or older browsers)
import { randomBytes } from 'crypto';

// On login, set a non-httpOnly CSRF token the client can read
res.cookie('csrfToken', randomBytes(32).toString('hex'), {
  secure: process.env.NODE_ENV === 'production',
  sameSite: 'strict',
  // NOT httpOnly — client needs to read and send it
});

// Middleware to verify CSRF token on state-changing requests
function verifyCsrf(req, res, next) {
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) return next();

  const tokenFromCookie = req.cookies.csrfToken;
  const tokenFromHeader = req.headers['x-csrf-token'];

  if (!tokenFromCookie || tokenFromCookie !== tokenFromHeader) {
    return res.status(403).json({ error: { code: 'INVALID_CSRF_TOKEN' } });
  }
  next();
}
```

---

### Password Security

```javascript
// bcrypt for passwords — always
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;  // ~250ms on modern hardware; adjust up as CPUs get faster

async function hashPassword(plaintext) {
  return bcrypt.hash(plaintext, SALT_ROUNDS);
}

async function verifyPassword(plaintext, hash) {
  return bcrypt.compare(plaintext, hash);
}

// NEVER log, return, or store plaintext passwords
// NEVER use MD5, SHA1, or unsalted SHA256 for passwords

// Timing-safe comparison for tokens (prevents timing attacks)
import { timingSafeEqual } from 'crypto';

function safeCompareTokens(a, b) {
  const bufA = Buffer.from(a);
  const bufB = Buffer.from(b);
  if (bufA.length !== bufB.length) return false;
  return timingSafeEqual(bufA, bufB);
}
```

---

### Input Sanitization

```javascript
// Sanitize filenames for uploads
import path from 'path';

function sanitizeFilename(filename) {
  // Remove path traversal attempts, null bytes, special chars
  return path.basename(filename)
    .replace(/[^a-zA-Z0-9._-]/g, '_')
    .slice(0, 255);
}

// Parameterized queries — Mongoose does this by default for finds
// But be careful with $where, mapReduce, or aggregate $function — avoid user input there

// URL validation — never fetch arbitrary URLs from user input
const ALLOWED_REDIRECT_HOSTS = ['app.example.com', 'www.example.com'];

function validateRedirectUrl(url) {
  try {
    const parsed = new URL(url);
    if (!ALLOWED_REDIRECT_HOSTS.includes(parsed.hostname)) {
      throw new Error('Invalid redirect target');
    }
    return parsed.href;
  } catch {
    return '/';  // fallback to safe URL
  }
}
```

---

### Secrets Management

```javascript
// NEVER commit secrets to git
// .env.example — commit this (no real values)
// .env — gitignore this
// production — use Vault, AWS Secrets Manager, or env vars injected by platform

// Validate all secrets exist at startup
const requiredSecrets = [
  'JWT_ACCESS_SECRET',
  'JWT_REFRESH_SECRET',
  'MONGODB_URI',
  'ENCRYPTION_KEY',
];

for (const secret of requiredSecrets) {
  if (!process.env[secret]) {
    console.error(`FATAL: missing required secret: ${secret}`);
    process.exit(1);
  }
}

// Encryption at rest for sensitive fields (PII)
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');  // 32 bytes = 64 hex chars

function encrypt(text) {
  const iv = randomBytes(12);
  const cipher = createCipheriv(ALGORITHM, KEY, iv);
  const encrypted = Buffer.concat([cipher.update(text, 'utf8'), cipher.final()]);
  const tag = cipher.getAuthTag();
  return `${iv.toString('hex')}:${tag.toString('hex')}:${encrypted.toString('hex')}`;
}

function decrypt(encryptedText) {
  const [ivHex, tagHex, dataHex] = encryptedText.split(':');
  const decipher = createDecipheriv(ALGORITHM, KEY, Buffer.from(ivHex, 'hex'));
  decipher.setAuthTag(Buffer.from(tagHex, 'hex'));
  return decipher.update(Buffer.from(dataHex, 'hex')) + decipher.final('utf8');
}
```

---

### Dependency Security

```bash
# Run regularly (add to CI):
npm audit --audit-level=high
npx better-npm-audit audit

# Auto-fix safe updates:
npm audit fix

# Check for known-malicious packages:
npx package-checker  # or socket.dev

# Pin versions in production, use lockfile
# Enable Dependabot or Renovate for automated PRs
```

---

### Security Checklist

```
## MERN Security Audit

### Input/Output
- [ ] All user input validated with Joi/Zod before use
- [ ] express-mongo-sanitize installed and active
- [ ] No dangerouslySetInnerHTML with user content; DOMPurify used if unavoidable
- [ ] File uploads: type validation, size limits, S3 not local disk

### Auth
- [ ] Passwords hashed with bcrypt (rounds >= 12)
- [ ] JWT secrets are long random strings (>= 32 chars), never hardcoded
- [ ] Refresh tokens stored in httpOnly, Secure, SameSite=strict cookies
- [ ] Access tokens short-lived (15min)

### Transport
- [ ] Helmet.js active with CSP configured
- [ ] CORS restricted to known origins
- [ ] HTTPS enforced in production

### Infrastructure
- [ ] No secrets in code or git history
- [ ] npm audit passing (no high/critical)
- [ ] Principle of least privilege on DB user
- [ ] Rate limiting on auth and expensive endpoints
```
