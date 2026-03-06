---
name: testing-strategy
description: "Unit/integration/contract/E2E pyramid ratios, what to test at each level, coverage targets, mutation testing. Use when planning what tests to write for a feature or evaluating an existing test suite."
---

## Testing Strategy

The testing pyramid is a guide to how many tests to write at each level — not a rule. The right ratio depends on your system's risk profile, speed requirements, and team size.

### Context

Feature or system to define test strategy for: **$ARGUMENTS**

---

### The Testing Pyramid

```
          /\
         /  \
        / E2E \          Few — expensive, slow, brittle, catch integration bugs
       /--------\
      /Integration\      Some — medium cost, catch contract violations
     /--------------\
    /   Unit Tests   \   Many — cheap, fast, catch logic bugs in isolation
   /------------------\
```

**Typical ratios:**
- Unit: 70%, Integration: 20%, E2E: 10%
- But: a microservice with no UI might be 40%/55%/5%
- An API gateway with little logic might be 20%/70%/10%

**The honeycomb model (for services):** Integration tests at the center (service tests calling real dependencies with test instances), fewer unit tests (for pure logic), minimal E2E.

---

### Unit Tests

Test one function/class in complete isolation. Dependencies are mocked or stubbed.

**What to unit test:**
- Business logic (calculations, transformations, state machines)
- Validation functions
- Pure utility functions
- Error conditions (validate error thrown, error message, error type)

**What NOT to unit test:**
- Database queries (test at integration level with real DB)
- HTTP routing (test at integration level)
- Trivial getters/setters

```javascript
// Good unit test: pure business logic, no dependencies
// calculateOrderDiscount.test.js
import { calculateOrderDiscount } from './calculateOrderDiscount';

describe('calculateOrderDiscount', () => {
  describe('premium subscriber discount', () => {
    it('applies 15% for premium subscribers with orders over $100', () => {
      const order = { total: 150, hasDigitalItems: false };
      const user = { subscriptionTier: 'premium', loyaltyPoints: 0 };
      expect(calculateOrderDiscount(user, order)).toBe(22.5);  // 150 * 0.15
    });

    it('does not apply premium discount to digital orders', () => {
      const order = { total: 150, hasDigitalItems: true };
      const user = { subscriptionTier: 'premium', loyaltyPoints: 0 };
      expect(calculateOrderDiscount(user, order)).toBe(0);
    });
  });

  it('applies 10% for users with >1000 loyalty points on orders over $50', () => {
    const order = { total: 75, hasDigitalItems: false };
    const user = { subscriptionTier: 'basic', loyaltyPoints: 1500 };
    expect(calculateOrderDiscount(user, order)).toBe(7.5);
  });

  it('returns 0 for orders that qualify for no discount', () => {
    const order = { total: 10, hasDigitalItems: false };
    const user = { subscriptionTier: 'free', loyaltyPoints: 0 };
    expect(calculateOrderDiscount(user, order)).toBe(0);
  });
});
```

---

### Integration Tests

Test a component against real dependencies (real DB, real cache, real message queue — running locally/in CI).

**What to integration test:**
- API endpoints (HTTP request -> DB -> response)
- Repository methods with real database queries
- Message queue consumer + DB interactions
- Cache invalidation logic

```javascript
// API integration test with Supertest + real Express app
// POST /api/users endpoint test
import request from 'supertest';
import { app } from '../app';
import { db } from '../db';

beforeAll(async () => {
  await db.migrate.latest();  // run migrations
});

afterEach(async () => {
  await db('users').truncate();  // clean state between tests
});

afterAll(async () => {
  await db.destroy();
});

describe('POST /api/v1/users', () => {
  it('creates a user and returns 201 with user data', async () => {
    const response = await request(app)
      .post('/api/v1/users')
      .send({ email: 'alice@example.com', password: 'password123', displayName: 'Alice' })
      .expect(201);

    expect(response.body).toMatchObject({
      email: 'alice@example.com',
      displayName: 'Alice'
    });
    expect(response.body).toHaveProperty('id');
    expect(response.body).not.toHaveProperty('passwordHash');  // never expose

    // Verify it's actually in the DB
    const dbUser = await db('users').where({ email: 'alice@example.com' }).first();
    expect(dbUser).toBeDefined();
  });

  it('returns 409 if email already exists', async () => {
    await request(app).post('/api/v1/users')
      .send({ email: 'alice@example.com', password: 'pass', displayName: 'Alice' });

    const response = await request(app)
      .post('/api/v1/users')
      .send({ email: 'alice@example.com', password: 'different', displayName: 'Alice2' })
      .expect(409);

    expect(response.body.error.code).toBe('EMAIL_ALREADY_EXISTS');
  });

  it('returns 400 for invalid email format', async () => {
    const response = await request(app)
      .post('/api/v1/users')
      .send({ email: 'not-an-email', password: 'pass', displayName: 'Alice' })
      .expect(400);

    expect(response.body.error.code).toBe('VALIDATION_ERROR');
  });
});
```

---

### E2E Tests

Test the full system from a user's perspective. Use real browser or real HTTP client against a running deployment.

**What to E2E test:**
- Critical user journeys (sign up, log in, core feature, check out)
- Smoke tests after deployment (is the app alive? can users log in?)
- Edge cases that span multiple services

```javascript
// Playwright E2E test
// tests/e2e/registration.spec.ts
import { test, expect } from '@playwright/test';

test.describe('User Registration', () => {
  test('user can register and access dashboard', async ({ page }) => {
    await page.goto('/register');

    await page.fill('[data-testid="email-input"]', 'alice@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePass123!');
    await page.fill('[data-testid="display-name-input"]', 'Alice');
    await page.click('[data-testid="register-button"]');

    // Wait for navigation to dashboard
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('[data-testid="welcome-message"]'))
      .toContainText('Welcome, Alice');
  });

  test('shows error for duplicate email', async ({ page }) => {
    // Seed: create user first
    await createUser('existing@example.com');

    await page.goto('/register');
    await page.fill('[data-testid="email-input"]', 'existing@example.com');
    await page.fill('[data-testid="password-input"]', 'pass');
    await page.click('[data-testid="register-button"]');

    await expect(page.locator('[data-testid="error-message"]'))
      .toContainText('Email already registered');
    // Still on register page
    await expect(page).toHaveURL('/register');
  });
});

// Selector strategy:
// data-testid="..." — preferred (stable, not tied to CSS or text)
// role="button" with name — good for accessibility + tests
// CSS classes — avoid (break on refactoring)
// Text content — fragile (copy changes break tests)
```

---

### Coverage Targets

```
Line/branch coverage:
- < 40%: high risk, hard to refactor safely
- 60-70%: acceptable minimum for production code
- 80-85%: good coverage, sustainable
- > 90%: high, investigate if it's meaningful or just noise
- 100%: usually not worth it; diminishing returns on last 10-15%

What matters more than % coverage:
- Are the CRITICAL paths covered? (auth, payments, data mutations)
- Are error paths covered? (network failure, invalid input, not found)
- Are business rules covered? (discount calculation, access control)

Coverage tools:
- Jest: built-in (--coverage flag)
- Istanbul/nyc: for non-Jest setups
- V8 coverage: native Node.js, faster than Istanbul
```

---

### Mutation Testing

Coverage tells you what was executed; mutation testing tells you if your tests actually catch bugs.

```bash
# Stryker (Node.js mutation testing)
npm install --save-dev @stryker-mutator/core @stryker-mutator/jest-runner

# stryker.config.js
module.exports = {
  testRunner: 'jest',
  mutate: ['src/**/*.js', '!src/**/*.test.js'],
  thresholds: { high: 80, low: 60, break: 50 }
};

npx stryker run

# Stryker makes small changes (mutations) to your code:
# - changes > to >= (boundary mutation)
# - changes && to || (logical mutation)
# - removes return values
# If a mutant SURVIVES (tests still pass), your tests didn't catch the bug
# Mutation score = killed mutants / total mutants
# Target: > 75% mutation score for critical modules
```

---

### Output Format

```
## Test Strategy: [Feature/System]

### Risk Assessment
[What failure modes are most costly? What's the blast radius of a bug here?]

### Pyramid Recommendation
Unit: X%  Integration: Y%  E2E: Z%
Rationale: [why this ratio for this system]

### Unit Tests
[What to test, example test cases, mock strategy]

### Integration Tests
[What to test, DB/dependency setup, cleanup strategy]

### E2E Tests
[Critical user journeys to cover, selector strategy]

### Coverage Targets
[Line/branch targets, which modules need higher coverage]

### Test Data Strategy
[Fixtures / factories / builders — see test-data skill]

### CI Integration
[Which tests run on PR? On merge? On deploy?]
```
