---
name: mern-testing
description: "Testing MERN stack apps: unit testing Express services with Jest, integration testing APIs with Supertest, React component testing with Testing Library, and MongoDB test setup. Use when writing or improving tests for a MERN app."
---

## MERN Testing

### Context

Testing problem or area: **$ARGUMENTS**

---

### Test Setup

```javascript
// jest.config.js (root — monorepo or single project)
export default {
  projects: [
    {
      displayName: 'backend',
      testMatch: ['<rootDir>/backend/**/*.test.js'],
      testEnvironment: 'node',
      transform: {}, // ESM — no transform needed with --experimental-vm-modules
    },
    {
      displayName: 'frontend',
      testMatch: ['<rootDir>/frontend/**/*.test.{js,jsx,tsx}'],
      testEnvironment: 'jsdom',
      setupFilesAfterFramework: ['<rootDir>/frontend/jest.setup.js'],
      moduleNameMapper: {
        '\\.(css|scss)$': 'identity-obj-proxy',
      },
    },
  ],
};

// package.json scripts
// "test": "node --experimental-vm-modules node_modules/.bin/jest"
// "test:watch": "node --experimental-vm-modules node_modules/.bin/jest --watch"
// "test:coverage": "node --experimental-vm-modules node_modules/.bin/jest --coverage"
```

---

### MongoDB Test Setup (MongoMemoryServer)

```javascript
// backend/tests/helpers/db.js
import { MongoMemoryServer } from 'mongodb-memory-server';
import mongoose from 'mongoose';

let mongod;

export async function connectTestDb() {
  mongod = await MongoMemoryServer.create();
  await mongoose.connect(mongod.getUri());
}

export async function disconnectTestDb() {
  await mongoose.connection.dropDatabase();
  await mongoose.connection.close();
  await mongod.stop();
}

export async function clearTestDb() {
  const collections = mongoose.connection.collections;
  for (const key of Object.keys(collections)) {
    await collections[key].deleteMany({});
  }
}

// backend/tests/helpers/factories.js — test data factories
import { faker } from '@faker-js/faker';
import { User } from '../../models/User.js';
import bcrypt from 'bcrypt';

export async function createUser(overrides = {}) {
  const defaults = {
    email: faker.internet.email().toLowerCase(),
    password: await bcrypt.hash('Test1234!', 10),
    displayName: faker.person.fullName(),
    role: 'user',
    ...overrides,
  };
  return User.create(defaults);
}

export async function createAdminUser(overrides = {}) {
  return createUser({ role: 'admin', ...overrides });
}
```

---

### Unit Testing Services

```javascript
// backend/services/__tests__/userService.test.js
import { jest } from '@jest/globals';
import { connectTestDb, disconnectTestDb, clearTestDb } from '../tests/helpers/db.js';
import { createUser } from '../tests/helpers/factories.js';
import { userService } from './userService.js';

beforeAll(connectTestDb);
afterAll(disconnectTestDb);
afterEach(clearTestDb);

describe('userService.getById', () => {
  it('returns the user when found', async () => {
    const user = await createUser({ displayName: 'Alice' });

    const result = await userService.getById(user._id.toString());

    expect(result).toMatchObject({
      displayName: 'Alice',
      email: user.email,
    });
    expect(result.password).toBeUndefined();  // password should be excluded
  });

  it('throws NotFoundError when user does not exist', async () => {
    const nonExistentId = '000000000000000000000001';

    await expect(userService.getById(nonExistentId))
      .rejects.toMatchObject({ code: 'NOT_FOUND' });
  });
});

describe('userService.updateProfile', () => {
  it('updates allowed fields', async () => {
    const user = await createUser();

    const updated = await userService.updateProfile(user._id.toString(), {
      displayName: 'New Name'
    });

    expect(updated.displayName).toBe('New Name');
  });

  it('does not allow updating email via this method', async () => {
    const user = await createUser();
    const originalEmail = user.email;

    await userService.updateProfile(user._id.toString(), {
      email: 'hacker@evil.com',
      displayName: 'Hacker'
    });

    const refetched = await User.findById(user._id);
    expect(refetched.email).toBe(originalEmail);
  });
});

// Mocking external services
describe('userService.sendVerificationEmail', () => {
  it('calls email service with correct params', async () => {
    const mockSend = jest.fn().mockResolvedValue({ messageId: 'abc' });
    jest.unstable_mockModule('../services/emailService.js', () => ({
      emailService: { sendVerificationEmail: mockSend }
    }));

    const user = await createUser();
    await userService.sendVerificationEmail(user._id.toString());

    expect(mockSend).toHaveBeenCalledWith(
      expect.objectContaining({ email: user.email })
    );
  });
});
```

---

### Integration Testing with Supertest

```javascript
// backend/routes/__tests__/users.test.js
import request from 'supertest';
import { app } from '../../app.js';
import { connectTestDb, disconnectTestDb, clearTestDb } from '../tests/helpers/db.js';
import { createUser } from '../tests/helpers/factories.js';
import { signAccessToken } from '../../utils/jwt.js';

beforeAll(connectTestDb);
afterAll(disconnectTestDb);
afterEach(clearTestDb);

// Helper: authenticated request
function authRequest(user) {
  const token = signAccessToken({ sub: user._id.toString(), role: user.role });
  return {
    get: (url) => request(app).get(url).set('Authorization', `Bearer ${token}`),
    post: (url) => request(app).post(url).set('Authorization', `Bearer ${token}`),
    patch: (url) => request(app).patch(url).set('Authorization', `Bearer ${token}`),
    delete: (url) => request(app).delete(url).set('Authorization', `Bearer ${token}`),
  };
}

describe('GET /api/v1/users/:id', () => {
  it('returns 200 with user data for authenticated request', async () => {
    const user = await createUser({ displayName: 'Alice' });
    const client = authRequest(user);

    const res = await client.get(`/api/v1/users/${user._id}`);

    expect(res.status).toBe(200);
    expect(res.body.data).toMatchObject({
      displayName: 'Alice',
      email: user.email,
    });
    expect(res.body.data.password).toBeUndefined();
  });

  it('returns 401 without token', async () => {
    const user = await createUser();

    const res = await request(app).get(`/api/v1/users/${user._id}`);

    expect(res.status).toBe(401);
  });

  it('returns 404 for non-existent user', async () => {
    const user = await createUser();
    const client = authRequest(user);

    const res = await client.get('/api/v1/users/000000000000000000000001');

    expect(res.status).toBe(404);
    expect(res.body.error.code).toBe('NOT_FOUND');
  });
});

describe('PATCH /api/v1/users/:id', () => {
  it('returns 403 when updating another user without admin role', async () => {
    const owner = await createUser();
    const requester = await createUser();
    const client = authRequest(requester);

    const res = await client
      .patch(`/api/v1/users/${owner._id}`)
      .send({ displayName: 'Hacker' });

    expect(res.status).toBe(403);
  });

  it('validates request body', async () => {
    const user = await createUser();
    const client = authRequest(user);

    const res = await client
      .patch(`/api/v1/users/${user._id}`)
      .send({ displayName: '' });  // too short

    expect(res.status).toBe(400);
    expect(res.body.error.code).toBe('VALIDATION_ERROR');
  });
});
```

---

### React Testing Library

```jsx
// frontend/components/__tests__/LoginForm.test.jsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { LoginForm } from '../LoginForm.jsx';

// Setup user-event
const user = userEvent.setup();

describe('LoginForm', () => {
  it('submits with email and password', async () => {
    const onSubmit = jest.fn().mockResolvedValue(undefined);
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'Password123!');
    await user.click(screen.getByRole('button', { name: /log in/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        email: 'alice@example.com',
        password: 'Password123!',
      });
    });
  });

  it('shows validation errors for empty submit', async () => {
    render(<LoginForm onSubmit={jest.fn()} />);

    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(await screen.findByText(/invalid email/i)).toBeInTheDocument();
    expect(screen.getByText(/password must be/i)).toBeInTheDocument();
  });

  it('shows server error on invalid credentials', async () => {
    const onSubmit = jest.fn().mockRejectedValue({ code: 'INVALID_CREDENTIALS' });
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'WrongPass123!');
    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(await screen.findByText(/invalid email or password/i)).toBeInTheDocument();
  });

  it('disables button while submitting', async () => {
    let resolve;
    const onSubmit = jest.fn(() => new Promise(r => { resolve = r; }));
    render(<LoginForm onSubmit={onSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.type(screen.getByLabelText(/password/i), 'Password123!');
    await user.click(screen.getByRole('button', { name: /log in/i }));

    expect(screen.getByRole('button', { name: /logging in/i })).toBeDisabled();
    resolve();
  });
});
```

---

### React Query Testing

```jsx
// frontend/components/__tests__/UserProfile.test.jsx
import { render, screen } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { server } from '../tests/mocks/server.js';  // MSW server
import { rest } from 'msw';
import { UserProfile } from '../UserProfile.jsx';

function renderWithQuery(ui) {
  const client = new QueryClient({
    defaultOptions: { queries: { retry: false } }  // no retries in tests
  });
  return render(
    <QueryClientProvider client={client}>{ui}</QueryClientProvider>
  );
}

// MSW handler in tests/mocks/handlers.js
// rest.get('/api/v1/users/:id', (req, res, ctx) => res(ctx.json({ data: mockUser })))

describe('UserProfile', () => {
  it('renders user data', async () => {
    renderWithQuery(<UserProfile userId="123" />);

    expect(screen.getByText(/loading/i)).toBeInTheDocument();
    expect(await screen.findByText('Alice Smith')).toBeInTheDocument();
  });

  it('renders error state', async () => {
    server.use(
      rest.get('/api/v1/users/999', (req, res, ctx) =>
        res(ctx.status(404), ctx.json({ error: { code: 'NOT_FOUND' } }))
      )
    );

    renderWithQuery(<UserProfile userId="999" />);

    expect(await screen.findByText(/not found/i)).toBeInTheDocument();
  });
});
```

---

### Coverage Targets

```
Unit (service logic):     90%+ — fast, isolated, test business rules
Integration (API routes): 80%+ — test request/response, middleware, DB
Component (UI):           70%+ — test user interactions, not implementation
E2E (critical paths):     5-10 flows — login, checkout, key user journeys

Run: jest --coverage --coverageThreshold='{"global":{"lines":80}}'
```
