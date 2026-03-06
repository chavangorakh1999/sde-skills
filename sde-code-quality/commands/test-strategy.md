---
description: Testing pyramid plan with example test cases at each level — unit, integration, E2E
argument-hint: "<feature or component description>"
---

# /test-strategy -- Test Strategy Planning

Plan what tests to write for a feature, at what level, with example test cases. Chains testing-strategy -> security-review (for test coverage of security paths).

## Invocation

```
/test-strategy User registration, email verification, and login flow
/test-strategy Payment service calling Stripe, with retry and DLQ
/test-strategy React DataTable component with server-side pagination
/test-strategy                    # asks what to test
```

## Workflow

### Step 1: Understand What's Being Built

Extract:
- What does this feature do?
- What are the success paths?
- What are the error paths?
- What external dependencies does it have? (DB, external APIs, queues)
- What's the risk level? (financial, auth, data loss — higher risk = more tests)

### Step 2: Risk Assessment

Higher risk -> higher coverage -> more test levels:

```
Risk factors that increase test investment:
- Financial transactions (payments, billing, refunds)
- Authentication and authorization
- Data that can't be recovered if wrong (deletions, migrations)
- Public API (external consumers break on regressions)
- Features used by many users

Risk factors that decrease test investment:
- Internal admin tools
- Low-traffic features
- Already covered by integration or E2E tests
```

### Step 3: Define the Testing Pyramid

Apply **testing-strategy** skill:

**Unit test candidates** (pure logic, no I/O):
- Input validation logic
- Business rule calculations
- State machine transitions
- Error message generation
- Data transformation/mapping functions

**Integration test candidates** (component + real dependencies):
- API endpoints (HTTP -> service -> DB -> response)
- Repository methods with real DB
- Message queue consumer with real queue
- Cache invalidation with real Redis

**E2E test candidates** (full user journey in running app):
- Happy path for critical user journeys only
- Smoke test: can users complete the core action?

### Step 4: Write Example Test Cases

For each level, write 3-5 concrete example test cases:

**Unit:**
```javascript
// validateRegistration
it('rejects email without @ symbol')
it('rejects password shorter than 8 characters')
it('accepts valid email and password')
it('trims whitespace from email')
```

**Integration:**
```javascript
// POST /api/v1/users
it('creates user and returns 201 with user data')
it('returns 409 when email already exists')
it('returns 400 for invalid email format')
it('hashes password before storing (does not store plaintext)')
it('sends welcome email after successful registration')
```

**E2E:**
```javascript
// Playwright
it('user can register, receive verification email, verify, and log in')
it('shows error when registering with existing email')
```

### Step 5: Security Test Cases

Apply security-review lens — which security-sensitive paths need explicit test coverage?

```javascript
// Auth tests (always write these)
it('rejects login with wrong password — returns 401, not 404')  // no user enumeration
it('rate limits login after 10 failed attempts')
it('invalidates session on logout')
it('cannot access protected endpoint without token')
it('cannot access another user\'s resource with valid token')  // horizontal escalation
```

### Step 6: Test Data Strategy

How to set up and tear down test data?
- **Factories**: programmatic creation for integration tests (`createUser(overrides)`)
- **Fixtures**: static JSON for unit tests
- **Builders**: for complex objects with many optional fields

```javascript
// Factory example (integration tests)
async function createUser(overrides = {}) {
  return User.create({
    email: `test-${Date.now()}@example.com`,
    passwordHash: await bcrypt.hash('password123', 10),
    displayName: 'Test User',
    ...overrides
  });
}
```

### Step 7: CI Integration

Which tests run where?
- PR checks: unit + integration (must be < 3 minutes total)
- Main branch merge: unit + integration + critical E2E
- Pre-production deploy: full E2E suite
- Smoke test (post-deploy): 3-5 key journeys only

## Output

```
## Test Strategy: [Feature Name]

### Risk Assessment
[Risk level: Low / Medium / High — with rationale]

### Pyramid Recommendation
Unit: X tests covering [what]
Integration: Y tests covering [what]
E2E: Z tests covering [what]

### Unit Tests

| Test | Input | Expected Output |
|------|-------|-----------------|

### Integration Tests

| Endpoint/Operation | Scenario | Expected Result |
|--------------------|----------|-----------------|

### E2E Tests

| Journey | Steps | Assertion |
|---------|-------|-----------|

### Security Test Coverage

| Attack Scenario | Test |

### Test Data Setup
[Factory/fixture approach]

### Coverage Target
[% and which modules need highest coverage]

### CI Plan
[PR gates / merge gates / deployment smoke tests]
```

## Next Steps

- "Want me to generate the actual tests? -> `/write-tests [code]`"
- "Want me to review existing tests? -> `/test-review [test file]`"
