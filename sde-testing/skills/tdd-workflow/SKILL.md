---
name: tdd-workflow
description: "Test-Driven Development workflow: Red-Green-Refactor cycle, when to apply TDD, starting with the simplest case, triangulation, and common pitfalls. Use when practicing TDD or introducing it to a team."
---

## Test-Driven Development

### Context

Feature or component to develop with TDD: **$ARGUMENTS**

---

### The Red-Green-Refactor Cycle

```
RED:    Write a failing test that describes the desired behavior
GREEN:  Write the MINIMUM code to make the test pass (no more)
REFACTOR: Clean up the code without breaking tests

Key discipline: Never write production code without a failing test first.
                Never refactor without green tests.
```

---

### Worked Example: Password Validator

```javascript
// Step 1: RED — write the simplest failing test
// password-validator.test.js
describe('validatePassword', () => {
  it('returns valid for a strong password', () => {
    expect(validatePassword('Abcdef1!')).toEqual({ valid: true, errors: [] });
  });
});

// Run: jest — fails with "ReferenceError: validatePassword is not defined" ✓

// Step 2: GREEN — minimum code to pass (don't over-engineer)
// password-validator.js
export function validatePassword(password) {
  return { valid: true, errors: [] };  // hardcoded — just enough to pass
}
// Run: jest — passes ✓

// Step 3: RED — add next case (triangulation forces real implementation)
it('fails when password has no uppercase letter', () => {
  const result = validatePassword('abcdef1!');
  expect(result.valid).toBe(false);
  expect(result.errors).toContain('Must contain at least one uppercase letter');
});
// Run: fails ✓

// Step 4: GREEN — implement the uppercase check
export function validatePassword(password) {
  const errors = [];
  if (!/[A-Z]/.test(password)) {
    errors.push('Must contain at least one uppercase letter');
  }
  return { valid: errors.length === 0, errors };
}
// Run: passes ✓

// Step 5: RED — add more cases one at a time
it('fails when password has no number', () => {
  const result = validatePassword('Abcdefg!');
  expect(result.valid).toBe(false);
  expect(result.errors).toContain('Must contain at least one number');
});

it('fails when password has no special character', () => {
  const result = validatePassword('Abcdef12');
  expect(result.valid).toBe(false);
  expect(result.errors).toContain('Must contain at least one special character');
});

it('fails when password is shorter than 8 characters', () => {
  const result = validatePassword('Ab1!');
  expect(result.valid).toBe(false);
  expect(result.errors).toContain('Must be at least 8 characters');
});

// Step 6: GREEN — implement all rules
export function validatePassword(password) {
  const errors = [];

  if (password.length < 8) errors.push('Must be at least 8 characters');
  if (!/[A-Z]/.test(password)) errors.push('Must contain at least one uppercase letter');
  if (!/[0-9]/.test(password)) errors.push('Must contain at least one number');
  if (!/[^A-Za-z0-9]/.test(password)) errors.push('Must contain at least one special character');

  return { valid: errors.length === 0, errors };
}

// Step 7: REFACTOR — constants, clarity, no behavior changes
const RULES = [
  { test: (p) => p.length >= 8, error: 'Must be at least 8 characters' },
  { test: (p) => /[A-Z]/.test(p), error: 'Must contain at least one uppercase letter' },
  { test: (p) => /[0-9]/.test(p), error: 'Must contain at least one number' },
  { test: (p) => /[^A-Za-z0-9]/.test(p), error: 'Must contain at least one special character' },
];

export function validatePassword(password) {
  const errors = RULES.filter(r => !r.test(password)).map(r => r.error);
  return { valid: errors.length === 0, errors };
}
// Run: all tests still pass ✓
```

---

### TDD for API Endpoints

```javascript
// Start with the integration test — define the contract first
describe('POST /api/v1/users', () => {
  it('creates user and returns 201 with user data', async () => {
    const res = await request(app)
      .post('/api/v1/users')
      .send({ email: 'alice@example.com', password: 'Test1234!', displayName: 'Alice' });

    expect(res.status).toBe(201);
    expect(res.body.data).toMatchObject({
      email: 'alice@example.com',
      displayName: 'Alice',
    });
    expect(res.body.data.password).toBeUndefined();
  });
});
// RED — route doesn't exist yet

// Now work inside-out: implement the route → controller → service → repo
// Each layer gets its own unit tests as you implement it

// 1. Route file (thin)
router.post('/', validate(createUserSchema), asyncHandler(userController.create));

// 2. Controller unit test → implement controller
describe('userController.create', () => {
  it('returns 201 with user data', async () => {
    const mockUser = { id: '123', email: 'alice@example.com' };
    jest.spyOn(userService, 'createUser').mockResolvedValue(mockUser);

    const req = { body: { email: 'alice@example.com', password: 'Test1234!', displayName: 'Alice' } };
    const res = { status: jest.fn().mockReturnThis(), json: jest.fn() };

    await userController.create(req, res);

    expect(res.status).toHaveBeenCalledWith(201);
    expect(res.json).toHaveBeenCalledWith({ data: mockUser });
  });
});

// 3. Service unit test → implement service
// 4. Repo unit test → implement repo
// 5. Run integration test again — GREEN
```

---

### When TDD Works Best

```
High value:
  - Business logic with many edge cases (calculators, validators, state machines)
  - Pure functions with clear inputs/outputs
  - Algorithms (sorting, searching, transformation)
  - Protocol implementation (parsers, serializers)

Medium value:
  - API endpoints (start with integration test spec)
  - Event handlers
  - Data transformers

Lower value / consider testing after:
  - Simple CRUD with no logic
  - UI components (design is discovered, not specified)
  - Exploratory code you'll likely delete
```

---

### Triangulation

```
Never trust a test that passes on the first implementation.
Use triangulation: write ≥ 2 tests that force you to implement the real behavior.

Example:
  Test 1: calculateTax(100) returns 10       // could pass with "return 10"
  Test 2: calculateTax(200) returns 20       // forces real implementation
  Test 3: calculateTax(0) returns 0          // tests edge case

The three tests together prove the function works, not just for one case.
```

---

### Common TDD Pitfalls

```
1. Writing too much code in GREEN step
   → Resist. Write only what makes the test pass. Refactor after.

2. Skipping the refactor step
   → Accumulates debt. TDD without refactor = tests for messy code.

3. Writing tests after code
   → Tests tend to mirror implementation, not specify behavior. Write test first.

4. Testing private methods
   → Sign that unit is too large. Test through the public API.

5. Mocking everything
   → Tests that mock too heavily verify wiring, not behavior.
   → Prefer real collaborators in unit tests when fast/deterministic.

6. One assertion per test taken too literally
   → "One behavior per test" is better guidance. Multiple assertions for one behavior = fine.
```
