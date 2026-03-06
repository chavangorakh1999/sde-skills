---
name: mern-architecture
description: "MERN project structure, layered architecture (routes -> controllers -> services -> repositories), folder conventions, env setup, monorepo vs separated repos. Use when starting a new MERN project or restructuring an existing one."
---

## MERN Architecture

A well-structured MERN project makes code predictable, testable, and maintainable. The key: clear layers with one-way dependencies, consistent naming, and explicit boundaries between concerns.

### Context

Project to structure: **$ARGUMENTS**

---

### Project Structure

```
my-mern-app/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   ├── index.js          # Centralized config (reads env vars)
│   │   │   ├── database.js       # Mongoose connection
│   │   │   └── redis.js          # Redis client (if used)
│   │   │
│   │   ├── api/
│   │   │   ├── routes/           # Express router definitions
│   │   │   │   ├── index.js      # Root router (mounts all sub-routers)
│   │   │   │   ├── auth.routes.js
│   │   │   │   └── user.routes.js
│   │   │   │
│   │   │   ├── controllers/      # HTTP layer: parse request, call service, format response
│   │   │   │   ├── auth.controller.js
│   │   │   │   └── user.controller.js
│   │   │   │
│   │   │   ├── middlewares/      # Express middleware
│   │   │   │   ├── authenticate.js
│   │   │   │   ├── authorize.js
│   │   │   │   ├── validate.js   # Joi/Zod request validation
│   │   │   │   └── errorHandler.js
│   │   │   │
│   │   │   └── validators/       # Joi/Zod schemas
│   │   │       ├── auth.validator.js
│   │   │       └── user.validator.js
│   │   │
│   │   ├── services/             # Business logic (no HTTP, no DB)
│   │   │   ├── auth.service.js
│   │   │   └── user.service.js
│   │   │
│   │   ├── repositories/         # Data access layer (DB queries only)
│   │   │   ├── user.repository.js
│   │   │   └── session.repository.js
│   │   │
│   │   ├── models/               # Mongoose models and schemas
│   │   │   ├── User.model.js
│   │   │   └── Session.model.js
│   │   │
│   │   ├── utils/                # Pure utility functions
│   │   │   ├── jwt.js
│   │   │   ├── hash.js
│   │   │   └── errors.js         # Custom error classes
│   │   │
│   │   └── app.js                # Express app setup (no listen())
│   │
│   ├── server.js                 # Entry point: starts listening
│   ├── .env                      # Never commit
│   ├── .env.example              # Commit this — template without secrets
│   └── package.json
│
└── frontend/
    ├── src/
    │   ├── api/                  # API call functions (axios/fetch wrappers)
    │   │   ├── apiClient.js      # Configured axios instance with interceptors
    │   │   ├── auth.api.js
    │   │   └── user.api.js
    │   │
    │   ├── components/           # Reusable UI components
    │   │   ├── common/           # Button, Input, Modal, etc.
    │   │   └── features/         # Feature-specific: UserCard, OrderList, etc.
    │   │
    │   ├── pages/                # Page-level components (route destinations)
    │   │   ├── Dashboard.jsx
    │   │   ├── Login.jsx
    │   │   └── Register.jsx
    │   │
    │   ├── hooks/                # Custom React hooks
    │   │   ├── useAuth.js
    │   │   └── usePagination.js
    │   │
    │   ├── store/                # State management (Zustand/Redux)
    │   │   ├── auth.store.js
    │   │   └── user.store.js
    │   │
    │   ├── utils/                # Frontend utilities
    │   │   └── formatDate.js
    │   │
    │   └── App.jsx
    └── package.json
```

---

### Layered Architecture

Each layer has one job. Dependencies flow ONE way: routes -> controllers -> services -> repositories -> models.

```javascript
// routes/user.routes.js — registers endpoints only
import { Router } from 'express';
import { authenticate } from '../middlewares/authenticate.js';
import { validate } from '../middlewares/validate.js';
import { updateProfileSchema } from '../validators/user.validator.js';
import { UserController } from '../controllers/user.controller.js';

const router = Router();
const controller = new UserController(/* inject dependencies */);

router.get('/me', authenticate, controller.getProfile);
router.patch('/me', authenticate, validate(updateProfileSchema), controller.updateProfile);

export default router;

// controllers/user.controller.js — HTTP layer: request parsing + response formatting
export class UserController {
  constructor(userService) { this.userService = userService; }

  getProfile = async (req, res, next) => {
    try {
      const user = await this.userService.getProfile(req.user.id);
      res.json({ data: user });
    } catch (err) {
      next(err);  // delegate to error handler middleware
    }
  };

  updateProfile = async (req, res, next) => {
    try {
      const user = await this.userService.updateProfile(req.user.id, req.body);
      res.json({ data: user });
    } catch (err) {
      next(err);
    }
  };
}

// services/user.service.js — business logic (no HTTP, no raw DB)
export class UserService {
  constructor(userRepo) { this.userRepo = userRepo; }

  async getProfile(userId) {
    const user = await this.userRepo.findById(userId);
    if (!user) throw new NotFoundError('User not found');
    return sanitizeUser(user);  // strip sensitive fields
  }

  async updateProfile(userId, updates) {
    const { displayName, bio, avatarUrl } = updates;  // whitelist
    return this.userRepo.update(userId, { displayName, bio, avatarUrl });
  }
}

// repositories/user.repository.js — data access only
export class UserRepository {
  async findById(id) { return User.findById(id).lean(); }
  async findByEmail(email) { return User.findOne({ email }).lean(); }
  async create(data) { return User.create(data); }
  async update(id, data) { return User.findByIdAndUpdate(id, data, { new: true }).lean(); }
}
```

---

### Environment Configuration

```javascript
// config/index.js — centralized, validated config
import 'dotenv/config';

function required(name) {
  const value = process.env[name];
  if (!value) throw new Error(`Required environment variable ${name} is not set`);
  return value;
}

export const config = {
  node: {
    env: process.env.NODE_ENV ?? 'development',
    port: parseInt(process.env.PORT ?? '3000', 10)
  },
  db: {
    uri: required('MONGODB_URI')
  },
  jwt: {
    accessSecret: required('JWT_ACCESS_SECRET'),
    refreshSecret: required('JWT_REFRESH_SECRET'),
    accessExpiresIn: process.env.JWT_ACCESS_EXPIRES_IN ?? '15m',
    refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN ?? '30d'
  },
  redis: {
    url: process.env.REDIS_URL  // optional
  },
  cors: {
    origins: (process.env.CORS_ORIGINS ?? 'http://localhost:3000').split(',')
  }
};

// .env.example (commit this)
// MONGODB_URI=mongodb://localhost:27017/myapp
// JWT_ACCESS_SECRET=change-this-in-production
// JWT_REFRESH_SECRET=change-this-in-production-too
// CORS_ORIGINS=http://localhost:3000
// REDIS_URL=redis://localhost:6379
```

---

### Monorepo vs Separated Repos

```
Monorepo (single repo, multiple packages):
  Pros: atomic commits across frontend/backend, shared types, single CI pipeline
  Cons: CI takes longer, teams may conflict on root config
  When: team is small (< 10 engineers), strong coupling between frontend and backend
  Tools: npm workspaces, pnpm workspaces, Turborepo

Separated repos:
  Pros: independent deployments, independent versioning, teams don't conflict
  Cons: API contract drift, harder to develop locally (need both running)
  When: frontend and backend are owned by different teams

Recommendation for new MERN projects: start with a single repo,
two top-level directories (backend/ and frontend/). You can split later.
```

---

### Output Format

```
## MERN Architecture: [Project Name]

### Project Structure
[Full directory tree with annotations]

### Layer Responsibilities
[Table: Layer, Responsibility, What It Does NOT Do]

### Naming Conventions
[File, variable, function naming decisions]

### Environment Config
[.env.example with all required variables]

### Dependency Injection
[How services and repositories are wired together]

### Monorepo or Separate Repos?
[Recommendation with rationale]
```
