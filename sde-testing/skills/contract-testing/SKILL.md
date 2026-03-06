---
name: contract-testing
description: "Contract testing with Pact: consumer-driven contracts, provider verification, preventing breaking API changes. Use when building microservices or APIs consumed by multiple clients."
---

## Contract Testing

Contract tests verify that a consumer and provider can communicate — without requiring both services to be running simultaneously.

### Context

Service pair or contract testing problem: **$ARGUMENTS**

---

### Why Contract Testing

```
Problem: Service A calls Service B's API.
         Service B changes an endpoint.
         Service A breaks in production.

Integration tests catch this — but require both services running.
E2E tests catch this — but are slow and flaky.
Contract tests catch this — fast, isolated, run in CI.

Contract testing flow:
1. Consumer writes a test that records expected interactions → produces a pact file
2. Provider verifies the pact file against its real implementation
3. CI fails if provider changes break any consumer contract
```

---

### Consumer Side (Pact JS)

```javascript
// npm install @pact-foundation/pact
// consumer/tests/userService.pact.test.js

import { Pact, Matchers } from '@pact-foundation/pact';
import path from 'path';
import { userApiClient } from '../src/clients/userApiClient.js';

const { like, eachLike, email, string, integer } = Matchers;

const provider = new Pact({
  consumer: 'frontend-app',
  provider: 'user-service',
  port: 4000,
  log: path.resolve(__dirname, '../logs', 'pact.log'),
  dir: path.resolve(__dirname, '../pacts'),  // pact files written here
  logLevel: 'error',
});

describe('User API contract', () => {
  beforeAll(() => provider.setup());
  afterEach(() => provider.verify());
  afterAll(() => provider.finalize());

  describe('GET /users/:id', () => {
    it('returns user when found', async () => {
      await provider.addInteraction({
        state: 'user 123 exists',
        uponReceiving: 'a request for user 123',
        withRequest: {
          method: 'GET',
          path: '/users/123',
          headers: { Authorization: like('Bearer some-token') },
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            data: {
              id: like('123'),
              email: email('alice@example.com'),  // matches email format
              displayName: string('Alice Smith'),  // matches string type
              role: like('user'),
              createdAt: like('2024-01-01T00:00:00.000Z'),
            }
          }
        }
      });

      // Now make the real call to the mock provider
      const result = await userApiClient.getUser('123');

      expect(result).toMatchObject({
        id: expect.any(String),
        email: expect.stringMatching(/@/),
        displayName: expect.any(String),
      });
    });

    it('returns 404 when user not found', async () => {
      await provider.addInteraction({
        state: 'user 999 does not exist',
        uponReceiving: 'a request for non-existent user 999',
        withRequest: {
          method: 'GET',
          path: '/users/999',
        },
        willRespondWith: {
          status: 404,
          body: {
            error: {
              code: like('NOT_FOUND'),
              message: like('User not found'),
            }
          }
        }
      });

      await expect(userApiClient.getUser('999'))
        .rejects.toMatchObject({ status: 404 });
    });
  });

  describe('POST /users', () => {
    it('creates a user', async () => {
      await provider.addInteraction({
        state: 'no user with email alice@example.com exists',
        uponReceiving: 'a request to create a user',
        withRequest: {
          method: 'POST',
          path: '/users',
          headers: { 'Content-Type': 'application/json' },
          body: {
            email: 'alice@example.com',
            password: like('Test1234!'),
            displayName: like('Alice Smith'),
          }
        },
        willRespondWith: {
          status: 201,
          body: {
            data: {
              id: like('123'),
              email: like('alice@example.com'),
              displayName: like('Alice Smith'),
            }
          }
        }
      });

      const result = await userApiClient.createUser({
        email: 'alice@example.com',
        password: 'Test1234!',
        displayName: 'Alice Smith',
      });

      expect(result.id).toBeDefined();
    });
  });
});
```

---

### Provider Verification

```javascript
// npm install @pact-foundation/pact
// provider/tests/pact.verify.test.js

import { Verifier } from '@pact-foundation/pact';
import path from 'path';
import { app } from '../src/app.js';
import { connectTestDb, disconnectTestDb } from './helpers/db.js';
import { createUser } from './helpers/factories.js';

describe('User Service — Pact Verification', () => {
  let server;

  beforeAll(async () => {
    await connectTestDb();
    await new Promise(resolve => {
      server = app.listen(3001, resolve);
    });
  });

  afterAll(async () => {
    await disconnectTestDb();
    await new Promise(resolve => server.close(resolve));
  });

  it('verifies all consumer pacts', async () => {
    const verifier = new Verifier({
      provider: 'user-service',
      providerBaseUrl: 'http://localhost:3001',

      // Load pact files from consumer (committed to repo or fetched from broker)
      pactUrls: [
        path.resolve(__dirname, '../../frontend-app/pacts/frontend-app-user-service.json')
      ],

      // Set up provider state before each interaction
      stateHandlers: {
        'user 123 exists': async () => {
          await createUser({ _id: '000000000000000000000123', displayName: 'Alice' });
        },
        'user 999 does not exist': async () => {
          // No setup needed — user doesn't exist
        },
        'no user with email alice@example.com exists': async () => {
          await User.deleteOne({ email: 'alice@example.com' });
        },
      },

      // Add auth header so provider middleware passes
      requestFilter: (req, res, next) => {
        req.headers.authorization = `Bearer ${generateTestToken()}`;
        next();
      },

      publishVerificationResult: process.env.CI === 'true',
      providerVersion: process.env.GIT_SHA ?? '1.0.0',
    });

    await verifier.verifyProvider();
  });
});
```

---

### Pact Broker (CI Integration)

```yaml
# docker-compose.pact-broker.yml — run locally or use pactflow.io
services:
  pact-broker:
    image: pactfoundation/pact-broker
    ports: ["9292:9292"]
    environment:
      PACT_BROKER_DATABASE_URL: postgres://pact:pact@postgres/pact
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: pact
      POSTGRES_DB: pact

# Publish pacts from consumer CI:
# npx pact-broker publish ./pacts --broker-base-url http://localhost:9292 --consumer-app-version $GIT_SHA

# Verify can-i-deploy before provider releases:
# npx pact-broker can-i-deploy --pacticipant user-service --version $GIT_SHA --to-environment production
```

---

### Lightweight Alternative: JSON Schema Contracts

```javascript
// If Pact is too heavy, validate response shape with JSON Schema
import Ajv from 'ajv';
import addFormats from 'ajv-formats';

const ajv = new Ajv({ allErrors: true });
addFormats(ajv);

const userSchema = {
  type: 'object',
  required: ['data'],
  properties: {
    data: {
      type: 'object',
      required: ['id', 'email', 'displayName', 'role'],
      properties: {
        id: { type: 'string' },
        email: { type: 'string', format: 'email' },
        displayName: { type: 'string', minLength: 1 },
        role: { type: 'string', enum: ['user', 'admin', 'moderator'] },
        createdAt: { type: 'string', format: 'date-time' },
      },
      additionalProperties: true,  // allow extra fields (additive changes are ok)
    }
  }
};

const validate = ajv.compile(userSchema);

// In your integration test:
it('returns user matching the contract', async () => {
  const res = await request(app).get('/users/123').set(...);
  const valid = validate(res.body);
  if (!valid) console.error(validate.errors);
  expect(valid).toBe(true);
});
```

---

### When to Use Contract Testing

```
Good fit:
  - Multiple consumers calling the same API (frontend + mobile + partner)
  - Microservices with independent deployment cycles
  - Teams that can't easily run all services together

Not worth it:
  - Single frontend + single backend (integration tests suffice)
  - Rarely-changing APIs with a single consumer
  - Teams where all services deploy together
```
