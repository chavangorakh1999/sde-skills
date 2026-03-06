---
description: "Write tests for existing code: analyze the code, identify cases to test, and generate complete test file"
argument-hint: "[code or file path to test, or paste the code directly]"
---

## Write Tests

Analyze the provided code and write comprehensive tests.

### Phase 1 — Code Analysis

Read and understand the code:
- Identify all exported functions/classes/endpoints
- Map the inputs, outputs, and side effects of each
- Identify the branches (if/else, try/catch, switch cases)
- Identify external dependencies that need mocking
- Note any existing tests and fill gaps

### Phase 2 — Test Case Identification

For each function/endpoint, enumerate test cases:

**Happy path cases:**
- Normal input produces expected output
- Boundary values (min, max, empty collection)

**Error/edge cases:**
- Invalid input → validation error
- Not found → 404 / NotFoundError
- Auth failure → 401/403
- External dependency failure → graceful error handling
- Concurrent operations (if relevant)

**Branch coverage:**
- Every `if` should have a test for each branch
- Every `catch` block should have a test that triggers it

### Phase 3 — Mock Design

Identify what to mock vs what to use real:
- Real: pure logic, same-process utilities, in-memory DB
- Mock: external HTTP calls, email service, time-dependent code
- Spy: functions where you want to verify they were called but keep real behavior

Write mock setup that is minimal and representative:
```javascript
jest.mock('../repositories/userRepo.js');
userRepo.findById.mockResolvedValue({ id: '123', email: 'alice@example.com' });
```

### Phase 4 — Write the Tests

Generate the complete test file with:
- Descriptive `describe` blocks matching the code structure
- `beforeAll/afterAll` for DB setup/teardown
- `beforeEach/afterEach` for state cleanup
- Tests using AAA pattern (Arrange, Act, Assert)
- Meaningful test names: "returns X when Y given Z"
- Parameterized tests (`test.each`) for repetitive cases

### Phase 5 — Verify Coverage

Check that the test suite covers:
- [ ] All exported functions have tests
- [ ] Happy path tested for each
- [ ] All meaningful error cases tested
- [ ] All branch conditions exercised
- [ ] Async errors caught with `rejects.toThrow`

### Output

Provide:
1. **Test analysis** — what functions/endpoints you found, what cases you identified
2. **Complete test file** ready to run (correct imports, helpers, all test cases)
3. **Mocking strategy** explanation
4. **Coverage estimate** — approximate branch coverage the tests achieve
5. **Missing cases** — any edge cases that would require additional context to test
