---
name: express-api
description: "Express.js REST API: router organization, controller pattern, async error handling, request validation with Joi/Zod, consistent response formats. Use when building Express API endpoints."
---

## Express.js REST API

Production-grade Express APIs follow consistent patterns: one router per resource, controllers that only handle HTTP concerns, services for business logic, and unified error handling.

### Context

API to build: **$ARGUMENTS**

---

### App Setup

```javascript
// app.js — configure Express (no listen here)
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import compression from 'compression';
import mongoSanitize from 'express-mongo-sanitize';
import rateLimit from 'express-rate-limit';
import { config } from './config/index.js';
import { apiRouter } from './api/routes/index.js';
import { errorHandler } from './api/middlewares/errorHandler.js';
import { requestLogger } from './api/middlewares/requestLogger.js';
import { generateRequestId } from './utils/generateRequestId.js';

const app = express();

// Security middleware (must come early)
app.use(helmet());  // sets security headers
app.use(cors({
  origin: config.cors.origins,
  credentials: true
}));

// Parse body
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: false, limit: '10mb' }));

// Sanitize MongoDB operators in request body/query (prevents NoSQL injection)
app.use(mongoSanitize());

// Compression
app.use(compression());

// Request ID + logging
app.use(generateRequestId);  // attaches req.id
app.use(requestLogger);

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 500,
  standardHeaders: true,
  legacyHeaders: false
});
app.use('/api', limiter);

// Routes
app.use('/api/v1', apiRouter);

// Health check (no auth required)
app.get('/health/live', (req, res) => res.json({ status: 'ok' }));
app.get('/health/ready', async (req, res) => {
  // check DB + cache connection
  res.json({ status: 'ready' });
});

// Error handler MUST be last middleware (4 params)
app.use(errorHandler);

export { app };

// server.js — start listening
import { app } from './app.js';
import { connectDatabase } from './config/database.js';
import { config } from './config/index.js';

connectDatabase().then(() => {
  app.listen(config.node.port, () => {
    console.log(`Server running on port ${config.node.port}`);
  });
});
```

---

### Router Organization

```javascript
// api/routes/index.js — root router
import { Router } from 'express';
import authRouter from './auth.routes.js';
import userRouter from './user.routes.js';
import postRouter from './post.routes.js';

const router = Router();

router.use('/auth', authRouter);
router.use('/users', userRouter);
router.use('/posts', postRouter);

export { router as apiRouter };

// api/routes/user.routes.js
import { Router } from 'express';
import { UserController } from '../controllers/user.controller.js';
import { authenticate } from '../middlewares/authenticate.js';
import { authorize } from '../middlewares/authorize.js';
import { validate } from '../middlewares/validate.js';
import { updateUserSchema, getUsersSchema } from '../validators/user.validator.js';

const router = Router();
const controller = new UserController(/* inject */);

router.get('/', authenticate, authorize('admin'), validate(getUsersSchema, 'query'), controller.list);
router.get('/me', authenticate, controller.getMe);
router.patch('/me', authenticate, validate(updateUserSchema), controller.updateMe);
router.get('/:id', authenticate, controller.getById);
router.delete('/:id', authenticate, authorize('admin'), controller.delete);

export default router;
```

---

### Controller Pattern

```javascript
// api/controllers/user.controller.js
export class UserController {
  constructor(userService) {
    this.userService = userService;
    // Bind arrow functions to avoid losing 'this' context in route handlers
  }

  list = async (req, res, next) => {
    try {
      const { page = 1, limit = 20, search } = req.query;
      const result = await this.userService.list({ page: +page, limit: +limit, search });
      res.json({
        data: result.users,
        pagination: {
          total: result.total,
          page: +page,
          limit: +limit,
          totalPages: Math.ceil(result.total / +limit)
        }
      });
    } catch (err) {
      next(err);
    }
  };

  getMe = async (req, res, next) => {
    try {
      const user = await this.userService.getProfile(req.user.id);
      res.json({ data: user });
    } catch (err) {
      next(err);
    }
  };

  updateMe = async (req, res, next) => {
    try {
      const user = await this.userService.updateProfile(req.user.id, req.body);
      res.json({ data: user });
    } catch (err) {
      next(err);
    }
  };

  delete = async (req, res, next) => {
    try {
      await this.userService.softDelete(req.params.id);
      res.status(204).end();
    } catch (err) {
      next(err);
    }
  };
}
```

---

### Request Validation (Joi)

```javascript
// api/validators/user.validator.js
import Joi from 'joi';

export const updateUserSchema = Joi.object({
  displayName: Joi.string().min(2).max(50).trim(),
  bio: Joi.string().max(500).trim().allow('', null),
  avatarUrl: Joi.string().uri().allow('', null)
}).min(1);  // at least one field must be provided

export const getUsersSchema = Joi.object({
  page:   Joi.number().integer().min(1).default(1),
  limit:  Joi.number().integer().min(1).max(100).default(20),
  search: Joi.string().max(100).trim().allow('')
});

// api/middlewares/validate.js
export function validate(schema, property = 'body') {
  return async (req, res, next) => {
    try {
      const validated = await schema.validateAsync(req[property], {
        abortEarly: false,    // collect all errors, not just first
        stripUnknown: true    // remove unknown fields
      });
      req[property] = validated;  // replace with validated + sanitized data
      next();
    } catch (err) {
      if (err.isJoi) {
        return res.status(400).json({
          error: {
            code: 'VALIDATION_ERROR',
            message: 'Validation failed',
            details: err.details.map(d => ({
              field: d.path.join('.'),
              message: d.message
            }))
          }
        });
      }
      next(err);
    }
  };
}
```

---

### Error Handling

```javascript
// utils/errors.js — typed errors for different scenarios
export class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;  // vs programming errors
  }
}

export class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(message, 404, 'NOT_FOUND');
  }
}

export class ConflictError extends AppError {
  constructor(message) {
    super(message, 409, 'CONFLICT');
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Access denied') {
    super(message, 403, 'FORBIDDEN');
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Authentication required') {
    super(message, 401, 'UNAUTHORIZED');
  }
}

// api/middlewares/errorHandler.js — MUST have 4 parameters
export function errorHandler(err, req, res, next) {
  // Log all errors (structured)
  req.log?.error({ err }, err.message) ?? console.error(err);

  // Handle Mongoose validation errors
  if (err.name === 'ValidationError') {
    return res.status(400).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Validation failed',
        details: Object.values(err.errors).map(e => ({ field: e.path, message: e.message }))
      }
    });
  }

  // Handle Mongoose duplicate key error
  if (err.code === 11000) {
    const field = Object.keys(err.keyValue)[0];
    return res.status(409).json({
      error: { code: 'DUPLICATE_KEY', message: `${field} already exists` }
    });
  }

  // Handle JWT errors
  if (err.name === 'JsonWebTokenError') {
    return res.status(401).json({ error: { code: 'INVALID_TOKEN', message: 'Invalid token' } });
  }
  if (err.name === 'TokenExpiredError') {
    return res.status(401).json({ error: { code: 'TOKEN_EXPIRED', message: 'Token expired' } });
  }

  // Handle operational errors (our AppError subclasses)
  if (err.isOperational) {
    return res.status(err.statusCode).json({
      error: { code: err.code, message: err.message }
    });
  }

  // Programming errors (bugs) — don't expose details
  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'Something went wrong',
      requestId: req.id
    }
  });
}
```

---

### Response Format

```javascript
// Consistent response envelope
// Success (single resource): { data: { ... } }
// Success (collection): { data: [...], pagination: { ... } }
// Error: { error: { code, message, details?, requestId } }

// Example responses:
// GET /api/v1/users/123
{ "data": { "id": "...", "email": "alice@example.com", "displayName": "Alice" } }

// GET /api/v1/users?page=1&limit=20
{
  "data": [{ "id": "...", "email": "..." }, ...],
  "pagination": { "total": 142, "page": 1, "limit": 20, "totalPages": 8 }
}

// POST /api/v1/users (201 Created)
{ "data": { "id": "...", "email": "..." } }

// DELETE /api/v1/users/123 (204 No Content)
// (empty body)

// Error (400 Bad Request)
{ "error": { "code": "VALIDATION_ERROR", "message": "Validation failed",
             "details": [{ "field": "email", "message": "Invalid email format" }] } }
```

---

### Output Format

```
## Express API: [Resource/Feature]

### Route Definition
[routes file]

### Controller
[controller with all handlers]

### Validation Schemas
[Joi/Zod schemas for request body and query params]

### Response Examples
[Success and error response JSON for each endpoint]

### Security Checklist
[ ] All state-changing routes require authentication
[ ] Authorization checked (not just authentication)
[ ] Input sanitized with express-mongo-sanitize
[ ] Rate limiting applied
[ ] No sensitive data in responses
```
