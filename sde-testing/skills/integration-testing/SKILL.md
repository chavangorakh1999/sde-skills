---
name: integration-testing
description: "Integration testing for Node.js APIs: Supertest setup, database integration with MongoMemoryServer, test isolation, seed data patterns, and testing middleware. Use when writing API integration tests."
---

## Integration Testing

Integration tests verify the collaboration between layers — routes, middleware, services, and the database work together correctly.

### Context

API or integration point to test: **$ARGUMENTS**

---

### When to Write Integration Tests (vs Unit)

```
Unit tests:        business logic in isolation (services, utils)
Integration tests: HTTP layer + middleware + DB working together

Write integration tests when:
- Testing authentication/authorization middleware
- Testing input validation + error response format
- Testing database query correctness
- Testing multiple services interacting

Don't write integration tests for:
- Pure business logic (belongs in unit tests — faster)
- Frontend component behavior
```

---

### Test Infrastructure

```javascript
// tests/helpers/server.js — shared Express app
import { createApp } from '../../src/app.js';

// Create a fresh app instance for tests
export const app = createApp();

// tests/helpers/db.js — in-memory MongoDB
import { MongoMemoryServer } from 'mongodb-memory-server';
import mongoose from 'mongoose';

let mongod;

export async function connectTestDb() {
  mongod = await MongoMemoryServer.create();
  const uri = mongod.getUri();
  await mongoose.connect(uri);
}

export async function disconnectTestDb() {
  await mongoose.connection.dropDatabase();
  await mongoose.connection.close();
  await mongod.stop();
}

export async function clearCollections(...models) {
  if (models.length === 0) {
    // Clear all collections
    for (const key of Object.keys(mongoose.connection.collections)) {
      await mongoose.connection.collections[key].deleteMany({});
    }
  } else {
    await Promise.all(models.map(m => m.deleteMany({})));
  }
}

// Global test setup — jest.setup.js
import { connectTestDb, disconnectTestDb } from './helpers/db.js';
beforeAll(connectTestDb);
afterAll(disconnectTestDb);
```

---

### Test Factories

```javascript
// tests/helpers/factories.js
import { faker } from '@faker-js/faker';
import bcrypt from 'bcrypt';
import { User } from '../../src/models/User.js';
import { Post } from '../../src/models/Post.js';
import { signAccessToken } from '../../src/utils/jwt.js';

export async function createUser(overrides = {}) {
  const user = await User.create({
    email: faker.internet.email().toLowerCase(),
    password: await bcrypt.hash('Test1234!', 10),
    displayName: faker.person.fullName(),
    role: 'user',
    emailVerified: true,
    ...overrides,
  });
  return user;
}

export async function createPost(authorId, overrides = {}) {
  return Post.create({
    title: faker.lorem.sentence(),
    content: faker.lorem.paragraphs(2),
    author: authorId,
    status: 'published',
    ...overrides,
  });
}

// Helper: create user + access token in one call
export async function createAuthenticatedUser(overrides = {}) {
  const user = await createUser(overrides);
  const token = signAccessToken({ sub: user._id.toString(), role: user.role });
  return { user, token, authHeader: `Bearer ${token}` };
}
```

---

### Request Helper

```javascript
// tests/helpers/request.js
import request from 'supertest';
import { app } from './server.js';

export function api(token) {
  const agent = request(app);
  const withAuth = (method) => (url) =>
    agent[method](url).set('Authorization', `Bearer ${token}`);

  return {
    get: (url) => agent.get(url),
    post: (url) => agent.post(url),
    patch: (url) => agent.patch(url),
    delete: (url) => agent.delete(url),
    authGet: withAuth('get'),
    authPost: withAuth('post'),
    authPatch: withAuth('patch'),
    authDelete: withAuth('delete'),
  };
}
```

---

### Writing Integration Tests

```javascript
// tests/routes/posts.test.js
import { clearCollections } from '../helpers/db.js';
import { createUser, createAuthenticatedUser, createPost } from '../helpers/factories.js';
import { api } from '../helpers/request.js';
import { Post } from '../../src/models/Post.js';

afterEach(() => clearCollections());

describe('POST /api/v1/posts', () => {
  it('creates a post and returns 201', async () => {
    const { token } = await createAuthenticatedUser();

    const res = await api(token).authPost('/api/v1/posts').send({
      title: 'My First Post',
      content: 'Content here...',
    });

    expect(res.status).toBe(201);
    expect(res.body.data).toMatchObject({
      title: 'My First Post',
      status: 'draft',
    });
    expect(res.body.data.id).toBeDefined();

    // Verify DB persistence
    const inDb = await Post.findById(res.body.data.id);
    expect(inDb).not.toBeNull();
    expect(inDb.title).toBe('My First Post');
  });

  it('returns 400 for missing required fields', async () => {
    const { token } = await createAuthenticatedUser();

    const res = await api(token).authPost('/api/v1/posts').send({
      content: 'No title provided',
    });

    expect(res.status).toBe(400);
    expect(res.body.error.code).toBe('VALIDATION_ERROR');
    expect(res.body.error.details).toContainEqual(
      expect.objectContaining({ field: 'title' })
    );
  });

  it('returns 401 when not authenticated', async () => {
    const res = await api().post('/api/v1/posts').send({
      title: 'Post',
      content: 'Content',
    });

    expect(res.status).toBe(401);
  });
});

describe('GET /api/v1/posts', () => {
  it('returns paginated posts', async () => {
    const { user, token } = await createAuthenticatedUser();
    await Promise.all([
      createPost(user._id, { title: 'Post 1' }),
      createPost(user._id, { title: 'Post 2' }),
      createPost(user._id, { title: 'Post 3' }),
    ]);

    const res = await api(token).authGet('/api/v1/posts?limit=2&page=1');

    expect(res.status).toBe(200);
    expect(res.body.data).toHaveLength(2);
    expect(res.body.pagination).toMatchObject({
      page: 1, limit: 2, total: 3, totalPages: 2
    });
  });

  it('only returns published posts for non-admin users', async () => {
    const { user, token } = await createAuthenticatedUser();
    await createPost(user._id, { status: 'published', title: 'Published' });
    await createPost(user._id, { status: 'draft', title: 'Draft' });

    const res = await api(token).authGet('/api/v1/posts');

    expect(res.body.data).toHaveLength(1);
    expect(res.body.data[0].title).toBe('Published');
  });
});

describe('DELETE /api/v1/posts/:id', () => {
  it('allows author to delete their own post', async () => {
    const { user, token } = await createAuthenticatedUser();
    const post = await createPost(user._id);

    const res = await api(token).authDelete(`/api/v1/posts/${post._id}`);

    expect(res.status).toBe(204);
    expect(await Post.findById(post._id)).toBeNull();
  });

  it('prevents deleting another user post', async () => {
    const { user: author } = await createAuthenticatedUser();
    const { token: otherToken } = await createAuthenticatedUser();
    const post = await createPost(author._id);

    const res = await api(otherToken).authDelete(`/api/v1/posts/${post._id}`);

    expect(res.status).toBe(403);
    expect(await Post.findById(post._id)).not.toBeNull();
  });

  it('allows admin to delete any post', async () => {
    const { user: author } = await createAuthenticatedUser();
    const { token: adminToken } = await createAuthenticatedUser({ role: 'admin' });
    const post = await createPost(author._id);

    const res = await api(adminToken).authDelete(`/api/v1/posts/${post._id}`);

    expect(res.status).toBe(204);
  });
});
```

---

### Testing Middleware

```javascript
// Test rate limiter behavior
describe('Rate limiting', () => {
  it('blocks after exceeding limit', async () => {
    const { token } = await createAuthenticatedUser();

    // Make max allowed requests
    for (let i = 0; i < 5; i++) {
      await api(token).authPost('/api/v1/auth/login').send({
        email: 'alice@example.com', password: 'wrong'
      });
    }

    // Next request should be rate limited
    const res = await api(token).authPost('/api/v1/auth/login').send({
      email: 'alice@example.com', password: 'wrong'
    });

    expect(res.status).toBe(429);
    expect(res.body.error.code).toBe('RATE_LIMIT_EXCEEDED');
  });
});
```

---

### Test Isolation Rules

```
1. Each test must be independent — don't rely on data from other tests
2. Use afterEach(clearCollections) — not afterAll (leaves dirty state for parallel tests)
3. Create exactly the data each test needs (use factories)
4. Don't share mutable state between tests (shared let variables = trouble)
5. Don't test multiple behaviors in one test — one assertion focus per test
```
