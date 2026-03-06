---
description: "Create a comprehensive test plan for a feature or system: coverage targets, test types, tooling, and CI integration"
argument-hint: "[feature or system to test, e.g. 'payment processing service']"
---

## Test Plan

Create a structured test plan covering all layers of the testing pyramid.

### Phase 1 — Risk Analysis

Identify what to test and why:
- **Critical paths**: what must never break? (auth, payments, data integrity)
- **Complex logic**: what has the most branches and edge cases?
- **Integration points**: external APIs, databases, message queues
- **User-facing flows**: what does the user directly experience?

Map risks to test types:
```
High risk + complex logic → unit tests + integration tests
User-facing flows → E2E tests
External integrations → mocks + contract tests
```

### Phase 2 — Test Inventory

List all tests to write, organized by layer:

**Unit Tests** (fast, isolated, 70% of test suite)
- List each service/function to unit test
- Identify key edge cases per function
- Estimate: ~2-5 tests per function

**Integration Tests** (medium speed, 20% of test suite)
- List each API endpoint
- For each: happy path + validation error + auth error
- Estimate: ~3-5 tests per endpoint

**E2E Tests** (slow, 10% of test suite)
- List critical user journeys only (3-10 total)
- One test per journey
- Use existing auth state where possible

### Phase 3 — Tooling & Setup

Specify the test infrastructure:
- **Unit/Integration**: Jest + MongoMemoryServer + Supertest + @faker-js/faker
- **E2E**: Playwright with Page Object Model
- **Contract** (if applicable): Pact + Pact Broker
- **Coverage**: Istanbul (built into Jest) with thresholds

Provide setup code for:
- jest.config.js with projects configuration
- Global setup/teardown
- Test helper file structure

### Phase 4 — Coverage Targets

Define thresholds per module:
```
Overall: 80% line coverage
Services (business logic): 90%
Controllers (HTTP layer): 75%
Utils/helpers: 85%
```

Identify coverage gaps and prioritize filling them:
- Untested error paths
- Untested conditional branches
- Untested middleware behavior

### Phase 5 — CI Integration

Design the CI pipeline:
```yaml
on: [push, pull_request]

jobs:
  unit-tests:         fast, run on every push
  integration-tests:  medium, run on every push
  e2e-tests:          slow, run on merge to main
  coverage-report:    blocks merge if thresholds not met
```

### Phase 6 — Test Data Strategy

Define how test data is managed:
- Factory functions for DB records
- Builder functions for API payloads
- Fixture files for static reference data
- Seeding strategy for dev/staging

### Output

Provide:
1. **Risk matrix** — what to test and why (prioritized)
2. **Complete test inventory** — all tests listed by category
3. **Setup code** — jest.config.js, helper files, CI config
4. **Coverage configuration** with per-module thresholds
5. **Estimated effort** — number of test files and approximate test count
