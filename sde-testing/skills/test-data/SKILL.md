---
name: test-data
description: "Test data management: factory functions, faker.js, fixture files, database seeding, test isolation, and avoiding flaky tests from shared state. Use when designing test data strategy for a project."
---

## Test Data Management

### Context

Test data problem or strategy question: **$ARGUMENTS**

---

### Factory Functions (Preferred)

```javascript
// tests/factories/userFactory.js
import { faker } from '@faker-js/faker';
import bcrypt from 'bcrypt';
import { User } from '../../src/models/User.js';

// Factory: creates in DB, returns document
export async function createUser(overrides = {}) {
  return User.create({
    email: faker.internet.email({ provider: 'example.com' }).toLowerCase(),
    password: await bcrypt.hash('Test1234!', 10),  // fast rounds for tests
    displayName: faker.person.fullName(),
    role: 'user',
    emailVerified: true,
    createdAt: faker.date.past({ years: 1 }),
    ...overrides,
  });
}

// Builder: returns plain object (useful for API body payloads)
export function buildUserPayload(overrides = {}) {
  return {
    email: faker.internet.email({ provider: 'example.com' }).toLowerCase(),
    password: 'Test1234!',
    displayName: faker.person.fullName(),
    ...overrides,
  };
}

// Scenario factories — named for what they represent
export async function createAdmin(overrides = {}) {
  return createUser({ role: 'admin', ...overrides });
}

export async function createUnverifiedUser(overrides = {}) {
  return createUser({ emailVerified: false, verificationToken: faker.string.uuid(), ...overrides });
}

export async function createUserWithPosts(postCount = 3) {
  const user = await createUser();
  const posts = await Promise.all(
    Array.from({ length: postCount }, () => createPost({ author: user._id }))
  );
  return { user, posts };
}
```

---

### Faker.js Patterns

```javascript
import { faker } from '@faker-js/faker';

// Seed faker for reproducible tests in CI
// faker.seed(12345);  // same seed = same data every run

// Useful generators:
faker.internet.email()                    // alice.smith@gmail.com
faker.internet.email({ provider: 'example.com' })  // bounded domain — no real emails
faker.person.fullName()                   // Alice Smith
faker.lorem.sentence()                    // random sentence
faker.lorem.paragraphs(2)                 // 2 paragraphs
faker.number.int({ min: 1, max: 100 })    // random integer
faker.date.past({ years: 1 })             // date in last year
faker.date.future({ years: 1 })           // date in next year
faker.string.uuid()                       // valid UUID
faker.datatype.boolean()                  // true/false
faker.helpers.arrayElement(['a', 'b', 'c'])  // random item

// Building arrays:
const tags = faker.helpers.multiple(faker.word.noun, { count: 3 });
```

---

### Fixture Files (for Static Reference Data)

```javascript
// tests/fixtures/countries.json — stable reference data that doesn't change
[
  { code: 'US', name: 'United States', currency: 'USD' },
  { code: 'GB', name: 'United Kingdom', currency: 'GBP' }
]

// tests/fixtures/payloads/validCreateUserPayload.json
{
  "email": "alice@example.com",
  "password": "Test1234!",
  "displayName": "Alice Smith"
}

// When to use fixtures vs factories:
// Fixtures:  static reference data, complex JSON structures, mock API responses
// Factories: DB records, dynamically generated test data with random fields
```

---

### Database Seeding (Dev/Staging)

```javascript
// scripts/seed.js — populate dev/staging database
import mongoose from 'mongoose';
import { faker } from '@faker-js/faker';
import { config } from '../src/config/index.js';
import { User } from '../src/models/User.js';
import { Post } from '../src/models/Post.js';

async function seed() {
  await mongoose.connect(config.mongoUri);
  console.log('Connected to DB');

  // Clear existing data (dev only!)
  if (config.env !== 'production') {
    await User.deleteMany({});
    await Post.deleteMany({});
    console.log('Cleared existing data');
  }

  // Create admin user (known credentials for dev)
  const admin = await User.create({
    email: 'admin@example.com',
    password: await bcrypt.hash('Admin1234!', 12),
    displayName: 'Admin User',
    role: 'admin',
    emailVerified: true,
  });

  // Create regular users
  const users = await Promise.all(
    Array.from({ length: 20 }, async () => User.create({
      email: faker.internet.email({ provider: 'example.com' }).toLowerCase(),
      password: await bcrypt.hash('Test1234!', 12),
      displayName: faker.person.fullName(),
      role: 'user',
      emailVerified: true,
    }))
  );

  // Create posts for each user
  for (const user of users) {
    const postCount = faker.number.int({ min: 1, max: 10 });
    await Promise.all(
      Array.from({ length: postCount }, () => Post.create({
        title: faker.lorem.sentence(),
        content: faker.lorem.paragraphs(faker.number.int({ min: 1, max: 5 })),
        author: user._id,
        status: faker.helpers.arrayElement(['draft', 'published', 'published', 'published']),
        createdAt: faker.date.past({ years: 2 }),
      }))
    );
  }

  console.log(`Seeded: 1 admin, ${users.length} users`);
  await mongoose.disconnect();
}

seed().catch(err => {
  console.error('Seed failed:', err);
  process.exit(1);
});

// package.json: "seed": "node scripts/seed.js"
```

---

### Test Isolation Patterns

```javascript
// Pattern 1: afterEach cleanup (recommended for unit/integration)
afterEach(async () => {
  await User.deleteMany({});
  await Post.deleteMany({});
});

// Pattern 2: transactions (roll back instead of delete — faster)
// Only if your DB supports transactions (MongoDB 4+ with replica set)
beforeEach(async () => {
  session = await mongoose.startSession();
  session.startTransaction();
});

afterEach(async () => {
  await session.abortTransaction();
  session.endSession();
});

// Pattern 3: separate DB per test file (parallel test runs)
// MongoMemoryServer creates isolated instance per file automatically

// Anti-pattern: shared test data
let userId;  // DON'T: shared state between tests
beforeAll(async () => {
  const user = await createUser();
  userId = user._id;  // tests that modify this user break each other
});

// Better:
it('works correctly', async () => {
  const user = await createUser();  // create fresh data per test
  // ...
});
```

---

### Avoiding Flaky Tests

```javascript
// Flaky cause 1: async timing — use waitFor, not setTimeout
// BAD:
await new Promise(r => setTimeout(r, 1000));  // arbitrary wait
expect(element).toBeVisible();

// GOOD:
await waitFor(() => expect(element).toBeVisible(), { timeout: 5000 });

// Flaky cause 2: test order dependency
// BAD: test B assumes test A ran first
it('A creates user', async () => { /* ... */ });
it('B updates user created by A', async () => { /* relies on A */ });

// GOOD: each test is self-contained
it('B updates user', async () => {
  const user = await createUser();  // creates its own data
  // ...
});

// Flaky cause 3: environment-dependent data
// BAD:
const users = await User.find({});
expect(users).toHaveLength(3);  // breaks if another test adds users

// GOOD:
const testUser = await createUser({ email: 'specific@example.com' });
const found = await User.findOne({ email: 'specific@example.com' });
expect(found).toBeDefined();

// Flaky cause 4: date/time dependencies
// BAD:
it('creates post with today's date', async () => {
  const post = await createPost();
  expect(post.createdAt.toDateString()).toBe(new Date().toDateString());  // fails at midnight!
});

// GOOD: use fake timers or approximate comparison
jest.useFakeTimers().setSystemTime(new Date('2024-01-15T12:00:00Z'));
```
