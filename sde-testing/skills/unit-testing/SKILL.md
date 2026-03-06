---
name: unit-testing
description: "Unit testing patterns with Jest: test structure, mocking, spying, async tests, snapshot testing, and coverage thresholds. Use when writing or improving unit tests for JavaScript/Node.js code."
---

## Unit Testing with Jest

### Context

Unit to test or testing problem: **$ARGUMENTS**

---

### Test Anatomy (AAA Pattern)

```javascript
// Arrange → Act → Assert
describe('userService.calculateTier', () => {
  it('returns gold tier for spend >= 1000', () => {
    // Arrange
    const user = { totalSpend: 1500, joinDate: new Date('2023-01-01') };

    // Act
    const tier = userService.calculateTier(user);

    // Assert
    expect(tier).toBe('gold');
  });

  it('returns silver tier for spend between 500 and 999', () => {
    const user = { totalSpend: 750, joinDate: new Date('2023-01-01') };
    expect(userService.calculateTier(user)).toBe('silver');
  });

  it('returns bronze tier for spend < 500', () => {
    const user = { totalSpend: 100, joinDate: new Date() };
    expect(userService.calculateTier(user)).toBe('bronze');
  });
});
```

---

### Describe Block Organization

```javascript
// Group by function, then by scenario
describe('CartService', () => {
  describe('addItem', () => {
    it('adds new item to empty cart');
    it('increments quantity when item already in cart');
    it('throws when quantity exceeds stock');
  });

  describe('removeItem', () => {
    it('removes item from cart');
    it('throws NotFoundError when item not in cart');
  });

  describe('calculateTotal', () => {
    it('sums all items');
    it('applies discount code correctly');
    it('returns 0 for empty cart');
  });
});
```

---

### Mocking Modules

```javascript
// jest.mock — replace entire module
jest.mock('../repositories/userRepository.js');
import { userRepository } from '../repositories/userRepository.js';

// After jest.mock(), all exported functions become jest.fn()
// userRepository.findById is now a jest.fn()

describe('UserService.getById', () => {
  beforeEach(() => {
    jest.clearAllMocks();  // reset call counts and implementations between tests
  });

  it('returns mapped user when found', async () => {
    const mockUser = { _id: '123', email: 'alice@example.com', role: 'user' };
    userRepository.findById.mockResolvedValue(mockUser);

    const result = await userService.getById('123');

    expect(userRepository.findById).toHaveBeenCalledWith('123');
    expect(result).toEqual({ id: '123', email: 'alice@example.com', role: 'user' });
  });

  it('throws NotFoundError when repository returns null', async () => {
    userRepository.findById.mockResolvedValue(null);

    await expect(userService.getById('123'))
      .rejects.toMatchObject({ code: 'NOT_FOUND' });
  });

  it('propagates unexpected repository errors', async () => {
    userRepository.findById.mockRejectedValue(new Error('connection lost'));

    await expect(userService.getById('123'))
      .rejects.toThrow('connection lost');
  });
});
```

---

### Spying (Partial Mocks)

```javascript
// jest.spyOn — mock one method, keep rest real
import { emailService } from '../services/emailService.js';

describe('UserService.register', () => {
  it('sends welcome email after successful registration', async () => {
    // Spy on just sendWelcome, don't mock the whole module
    const spy = jest.spyOn(emailService, 'sendWelcome')
      .mockResolvedValue({ messageId: 'abc' });

    await userService.register({ email: 'alice@example.com', password: 'Test1234!' });

    expect(spy).toHaveBeenCalledWith(
      expect.objectContaining({ email: 'alice@example.com' })
    );

    spy.mockRestore();  // restore original implementation
  });
});
```

---

### Async Testing

```javascript
// Always use async/await for async tests
it('handles concurrent requests correctly', async () => {
  // Run assertions after all promises resolve
  const [result1, result2] = await Promise.all([
    userService.getById('1'),
    userService.getById('2'),
  ]);

  expect(result1.id).toBe('1');
  expect(result2.id).toBe('2');
});

// Testing rejection — two patterns:
it('throws on invalid input', async () => {
  // Pattern 1: rejects.toThrow
  await expect(userService.getById('')).rejects.toThrow('ID is required');

  // Pattern 2: try/catch with assertions
  try {
    await userService.getById('');
    fail('Expected error was not thrown');
  } catch (err) {
    expect(err.message).toBe('ID is required');
  }
});

// Fake timers for setTimeout/setInterval/Date
it('expires cache after TTL', async () => {
  jest.useFakeTimers();

  const cache = new Cache({ ttl: 60_000 });
  cache.set('key', 'value');

  jest.advanceTimersByTime(61_000);

  expect(cache.get('key')).toBeNull();

  jest.useRealTimers();
});
```

---

### Parameterized Tests

```javascript
// test.each — avoid duplicating test structure
describe('validateEmail', () => {
  test.each([
    ['alice@example.com', true],
    ['alice+tag@example.co.uk', true],
    ['not-an-email', false],
    ['@nodomain.com', false],
    ['', false],
  ])('validateEmail(%s) returns %s', (email, expected) => {
    expect(validateEmail(email)).toBe(expected);
  });
});

// Object form for readability
test.each([
  { spend: 1500, expected: 'gold' },
  { spend: 750,  expected: 'silver' },
  { spend: 100,  expected: 'bronze' },
])('tier calculation: spend=$spend → $expected', ({ spend, expected }) => {
  expect(userService.calculateTier({ totalSpend: spend })).toBe(expected);
});
```

---

### Coverage Configuration

```javascript
// jest.config.js
export default {
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/config/**',      // don't cover config files
    '!src/migrations/**',  // don't cover migrations
  ],
  coverageThresholds: {
    global: {
      lines: 80,
      branches: 75,
      functions: 85,
      statements: 80,
    },
    // Higher bar for critical modules:
    './src/services/': {
      lines: 90,
    },
  },
  coverageReporters: ['text', 'html', 'lcov'],  // lcov for CI integration
};
```

---

### Test Quality Anti-Patterns

```javascript
// ANTI-PATTERN: testing implementation, not behavior
it('sets this._cache[key] in the internal map', () => {
  cache.set('key', 'val');
  expect(cache._cache.has('key')).toBe(true);  // testing internals
});
// Better:
it('returns value after setting', () => {
  cache.set('key', 'val');
  expect(cache.get('key')).toBe('val');
});

// ANTI-PATTERN: meaningless assertions
expect(result).toBeDefined();  // truthy check adds no value
expect(result).not.toBeNull(); // same

// Better: assert the actual shape
expect(result).toEqual({ id: '123', email: 'alice@example.com' });

// ANTI-PATTERN: test titles that just repeat the assertion
it('returns true');  // useless
// Better:
it('returns true when user has verified email and active subscription');
```
