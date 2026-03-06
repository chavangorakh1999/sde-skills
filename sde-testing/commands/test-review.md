---
description: "Review existing tests for quality, completeness, and anti-patterns"
argument-hint: "[test file or test suite to review]"
---

## Test Review

Evaluate tests for quality, completeness, and maintainability.

### Phase 1 — Structure & Organization

Check the test structure:
- [ ] Tests organized in `describe` blocks matching production code structure
- [ ] Test names describe behavior, not implementation ("returns 401 when unauthenticated", not "auth middleware test")
- [ ] Tests follow AAA (Arrange, Act, Assert) pattern
- [ ] No test logic inside `beforeAll/beforeEach` that's specific to one test

Flag:
```javascript
// BAD — test logic in beforeAll
beforeAll(async () => {
  await createSpecificUserForOneTest();  // should be in the test itself
});

// GOOD — test-specific setup in the test
it('updates user profile', async () => {
  const user = await createUser();  // arranged here, used here
  // ...
});
```

### Phase 2 — Test Independence

Verify isolation:
- [ ] Tests don't depend on execution order
- [ ] Tests don't share mutable state via `let` variables populated in `beforeAll`
- [ ] Each test creates its own data
- [ ] DB is cleared between tests (afterEach, not afterAll)
- [ ] Mocks are reset between tests (jest.clearAllMocks in beforeEach)

### Phase 3 — Assertion Quality

Check assertions:
- [ ] Assertions are specific (not `toBeDefined()` or `toBeTruthy()`)
- [ ] Expected values are hardcoded, not derived from the same logic under test
- [ ] Error cases assert the error type/message, not just that an error was thrown
- [ ] Async rejections use `await expect(...).rejects.toThrow(...)` (not try/catch with no assertion)
- [ ] Mutation tests verify DB state, not just response (call `findById` after)

```javascript
// BAD — too vague
expect(result).toBeDefined();
expect(error).toBeInstanceOf(Error);

// GOOD — specific
expect(result).toEqual({ id: '123', email: 'alice@example.com', role: 'user' });
await expect(service.getById('999')).rejects.toMatchObject({
  code: 'NOT_FOUND',
  message: 'User not found'
});
```

### Phase 4 — Mock Quality

Review mocking:
- [ ] Only external/slow dependencies mocked (not everything)
- [ ] Mocks return realistic data shapes
- [ ] Tests verify mock was called with correct arguments where important
- [ ] `mockRestore()` called after `spyOn` or cleanup in `afterEach`
- [ ] No `jest.mock()` of the module under test (test the real thing)

### Phase 5 — Coverage Assessment

Evaluate completeness:
- [ ] Every exported function has at least one test
- [ ] Happy path tested
- [ ] Error paths tested (not found, validation error, auth failure)
- [ ] Boundary conditions tested (empty array, zero, max values)
- [ ] Critical business rules have multiple tests from different angles

Identify gaps:
- What branches are untested?
- What error paths are missing tests?
- What side effects are not verified?

### Phase 6 — Performance & Reliability

Check for flakiness sources:
- [ ] No `setTimeout` delays waiting for async operations (use `waitFor` or proper awaiting)
- [ ] No date/time assertions without controlled time
- [ ] No assertions on collection length without isolated data
- [ ] No `test.only` or `test.skip` committed

### Output Format

```
## Test Review: [file/suite]

### Summary
Coverage: [estimated %] | Total tests: X | Issues: CRITICAL: X, MAJOR: X, MINOR: X

### CRITICAL (fix before merge)
- [TestName]: [Issue] → [Fix]

### MAJOR (should fix)
- [TestName]: [Issue] → [Fix]

### MINOR (nice to have)
- [TestName]: [Suggestion]

### Missing Test Cases
- [ ] Test for [scenario] — [why it matters]

### Good Patterns Found
- [What was done well]

### Refactored Examples
[Before/after for the most impactful fixes]
```
