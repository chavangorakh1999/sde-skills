---
name: express-middleware
description: "Express middleware patterns: error handling middleware, request validation, rate limiting, logging, auth middleware, custom middleware composition. Use when building or debugging Express middleware chains."
---

## Express Middleware

Middleware is the backbone of Express. Understanding execution order, error propagation, and composition separates clean Express apps from spaghetti.

### Context

Middleware problem or pattern: **$ARGUMENTS**

---

### Middleware Execution Model

```javascript
// Middleware executes in the order it's registered
// Must call next() to pass control — or end the request

app.use((req, res, next) => {
  console.log('1: runs first');
  next();          // pass to next middleware
});

app.use((req, res, next) => {
  console.log('2: runs second');
  next();
});

app.get('/path', (req, res) => {
  res.send('3: route handler runs last');
});

// Order matters — register middleware BEFORE routes that need it
// Exception: error-handling middleware goes AFTER routes
```

---

### Request Validation Middleware

```javascript
// middlewares/validate.js — Joi schema validation
import Joi from 'joi';
import { ValidationError } from '../errors/index.js';

export function validate(schema, source = 'body') {
  return (req, res, next) => {
    const { error, value } = schema.validate(req[source], {
      abortEarly: false,   // collect all errors, not just first
      stripUnknown: true   // remove fields not in schema
    });

    if (error) {
      const details = error.details.map(d => ({
        field: d.path.join('.'),
        message: d.message
      }));
      return next(new ValidationError('Validation failed', details));
    }

    req[source] = value;  // replace with sanitized/coerced values
    next();
  };
}

// Usage in routes:
import Joi from 'joi';
import { validate } from '../middlewares/validate.js';

const createUserSchema = Joi.object({
  email: Joi.string().email().lowercase().required(),
  password: Joi.string().min(8).max(72).required(),
  displayName: Joi.string().trim().min(2).max(50).required()
});

router.post('/users',
  validate(createUserSchema),
  userController.create
);

// Validate query params:
const listUsersSchema = Joi.object({
  page: Joi.number().integer().min(1).default(1),
  limit: Joi.number().integer().min(1).max(100).default(20),
  role: Joi.string().valid('admin', 'user', 'moderator')
});

router.get('/users',
  validate(listUsersSchema, 'query'),
  userController.list
);
```

---

### Authentication Middleware

```javascript
// middlewares/authenticate.js
import { verifyAccessToken } from '../utils/jwt.js';

export async function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: { code: 'MISSING_TOKEN' } });
  }

  const token = authHeader.slice(7);

  try {
    const payload = verifyAccessToken(token);
    req.user = { id: payload.sub, role: payload.role };
    next();
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: { code: 'TOKEN_EXPIRED' } });
    }
    return res.status(401).json({ error: { code: 'INVALID_TOKEN' } });
  }
}

// middlewares/authorize.js — RBAC
export function authorize(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: { code: 'UNAUTHENTICATED' } });
    }
    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: { code: 'INSUFFICIENT_PERMISSIONS' } });
    }
    next();
  };
}

// Usage:
router.delete('/users/:id',
  authenticate,
  authorize('admin'),
  userController.delete
);

// Route-level ownership check
export function requireOwnership(getResourceUserId) {
  return async (req, res, next) => {
    try {
      const resourceUserId = await getResourceUserId(req);
      if (req.user.role !== 'admin' && req.user.id !== resourceUserId.toString()) {
        return res.status(403).json({ error: { code: 'FORBIDDEN' } });
      }
      next();
    } catch (err) {
      next(err);
    }
  };
}
```

---

### Rate Limiting

```javascript
// middlewares/rateLimiter.js
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redisClient } from '../config/redis.js';

// General API rate limit
export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,
  standardHeaders: true,      // Return rate limit info in headers
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.sendCommand(args),
  }),
  keyGenerator: (req) => req.user?.id ?? req.ip,  // per-user if authenticated
  handler: (req, res) => {
    res.status(429).json({
      error: { code: 'RATE_LIMIT_EXCEEDED', retryAfter: res.getHeader('Retry-After') }
    });
  }
});

// Stricter limit for auth endpoints
export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,                     // 5 login attempts per 15 min
  skipSuccessfulRequests: true,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.sendCommand(args),
  }),
});

// Apply:
app.use('/api/v1', apiLimiter);
app.use('/api/v1/auth/login', authLimiter);
```

---

### Structured Request Logging

```javascript
// middlewares/requestLogger.js
import { randomUUID } from 'crypto';
import { logger } from '../config/logger.js';

export function requestLogger(req, res, next) {
  // Attach correlation ID for log tracing
  req.correlationId = req.headers['x-correlation-id'] ?? randomUUID();
  res.setHeader('x-correlation-id', req.correlationId);

  // Child logger with request context
  req.log = logger.child({
    correlationId: req.correlationId,
    method: req.method,
    path: req.path,
    userId: req.user?.id,  // may be undefined at this point — set after auth
  });

  const startTime = Date.now();

  res.on('finish', () => {
    req.log.info({
      statusCode: res.statusCode,
      durationMs: Date.now() - startTime,
      contentLength: res.getHeader('content-length'),
    }, 'request completed');
  });

  req.log.info('request received');
  next();
}

// Register early (before routes):
app.use(requestLogger);
// Register auth middleware after — it will populate req.user
// For logging userId, re-attach after auth or log in route handler
```

---

### Error Handling Middleware

```javascript
// middlewares/errorHandler.js
import { logger } from '../config/logger.js';

// Custom error classes
export class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;  // vs. programming errors
  }
}

export class ValidationError extends AppError {
  constructor(message, details) {
    super(message, 400, 'VALIDATION_ERROR');
    this.details = details;
  }
}

export class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

export class ConflictError extends AppError {
  constructor(message) { super(message, 409, 'CONFLICT'); }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') { super(message, 401, 'UNAUTHORIZED'); }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') { super(message, 403, 'FORBIDDEN'); }
}

// The error handler middleware — MUST have 4 params for Express to recognize it
export function errorHandler(err, req, res, next) {
  // Mongoose validation error
  if (err.name === 'ValidationError' && err.errors) {
    const details = Object.values(err.errors).map(e => ({
      field: e.path,
      message: e.message
    }));
    return res.status(400).json({
      error: { code: 'VALIDATION_ERROR', message: 'Validation failed', details }
    });
  }

  // Mongoose duplicate key
  if (err.code === 11000) {
    const field = Object.keys(err.keyPattern)[0];
    return res.status(409).json({
      error: { code: 'CONFLICT', message: `${field} already exists` }
    });
  }

  // Operational errors (our AppError subclasses)
  if (err.isOperational) {
    const body = {
      error: { code: err.code, message: err.message }
    };
    if (err.details) body.error.details = err.details;
    return res.status(err.statusCode).json(body);
  }

  // Programming errors — log and return generic message
  (req.log ?? logger).error({
    err,
    stack: err.stack,
    correlationId: req.correlationId,
  }, 'unhandled error');

  res.status(500).json({
    error: { code: 'INTERNAL_ERROR', message: 'An unexpected error occurred' }
  });
}

// Register AFTER all routes:
app.use(errorHandler);
```

---

### Middleware Composition

```javascript
// Compose multiple middleware into one
export function compose(...middlewares) {
  return middlewares;  // Express accepts arrays
}

// Usage:
const adminOnly = compose(authenticate, authorize('admin'));
router.delete('/users/:id', adminOnly, userController.delete);

// Conditional middleware
export function unless(middleware, ...paths) {
  return (req, res, next) => {
    if (paths.some(path => req.path.startsWith(path))) {
      return next();
    }
    return middleware(req, res, next);
  };
}

// Apply auth everywhere except public routes:
app.use(unless(authenticate, '/api/v1/auth', '/api/v1/health'));
```

---

### Output Format

```
## Middleware Analysis: [Problem/Feature]

### Issues Found
[For each: what middleware, what's wrong, what it causes]

### Implementation
[Complete middleware code with registration order]

### Middleware Stack Order
[Numbered list: 1. corsMiddleware 2. requestLogger 3. bodyParser ...]
```
