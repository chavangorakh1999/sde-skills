---
name: refactoring
description: "Safe refactoring patterns with step-by-step sequences: extract method/class, decompose conditional, replace magic number, introduce parameter object. Use when cleaning up code without changing behavior."
---

## Refactoring

Refactoring changes code structure without changing behavior. The key word is **safe**: each step must leave the tests green. Refactor in small, verifiable moves — not one giant rewrite.

### Context

Code to refactor: **$ARGUMENTS**

---

### The Golden Rule

**Never refactor and add features simultaneously.** Two separate commits:
1. Refactor: move code around, rename, extract — behavior unchanged, tests pass
2. Feature: add new behavior on the cleaner foundation

---

### Step 1: Establish a Safety Net

Before touching code, you need tests that catch regressions.

```javascript
// If tests don't exist, write characterization tests first:
// Characterization test = "document what the code currently does"
// (even if what it does is weird or wrong — that's a separate fix)

describe('UserRegistrationService (characterization)', () => {
  it('currently hashes password with MD5 (known bug — do not fix here)', () => {
    const service = new UserRegistrationService();
    const user = service.register({ email: 'a@b.com', password: 'pass123' });
    expect(user.passwordHash).toMatch(/^[a-f0-9]{32}$/);  // MD5 format
  });
  // ... document other current behaviors
});

// Run tests before starting. Note which pass. Refactoring is complete only
// when all previously-passing tests still pass.
```

---

### Extract Method

The most common refactoring. When a function does more than one thing, extract the parts.

```javascript
// Before: one big function doing many things
async function processOrder(orderId) {
  const order = await Order.findById(orderId);
  if (!order) throw new Error('Order not found');

  // Validate order
  if (order.items.length === 0) throw new Error('Empty order');
  if (order.total < 0) throw new Error('Invalid total');
  const hasStock = await Promise.all(order.items.map(item =>
    Inventory.check(item.productId, item.quantity)
  ));
  if (hasStock.some(s => !s)) throw new Error('Out of stock');

  // Charge payment
  const paymentIntent = await stripe.paymentIntents.create({
    amount: Math.round(order.total * 100),
    currency: 'usd',
    customer: order.customerId
  });

  // Update order
  order.status = 'paid';
  order.paymentIntentId = paymentIntent.id;
  order.paidAt = new Date();
  await order.save();

  // Send confirmation
  await emailService.send(order.customerEmail, 'order-confirmation', { order });
}

// After: extract each responsibility into a named method
async function processOrder(orderId) {
  const order = await findOrder(orderId);
  await validateOrder(order);
  const paymentIntent = await chargeOrder(order);
  await markOrderPaid(order, paymentIntent);
  await sendConfirmation(order);
}

async function findOrder(orderId) {
  const order = await Order.findById(orderId);
  if (!order) throw new NotFoundError(`Order ${orderId} not found`);
  return order;
}

async function validateOrder(order) {
  if (order.items.length === 0) throw new ValidationError('Order has no items');
  if (order.total < 0) throw new ValidationError('Order total is invalid');
  const stockResults = await Promise.all(
    order.items.map(item => Inventory.check(item.productId, item.quantity))
  );
  if (stockResults.some(inStock => !inStock)) {
    throw new ValidationError('One or more items are out of stock');
  }
}

// ... chargeOrder, markOrderPaid, sendConfirmation each in ~5-10 lines
```

**Steps:**
1. Identify a coherent chunk of code with a clear purpose
2. Extract it into a new function with an intention-revealing name
3. Pass in what it needs, return what it produces
4. Run tests — must still pass

---

### Extract Class

When a class has grown multiple unrelated responsibilities.

```javascript
// Before: UserService doing too many things
class UserService {
  async register(data) { ... }
  async login(email, password) { ... }
  async updateProfile(userId, data) { ... }
  async forgotPassword(email) { ... }  // email concern
  async resetPassword(token, password) { ... }  // email concern
  async sendVerificationEmail(userId) { ... }  // email concern
  async verifyEmail(token) { ... }  // email concern
}

// After: separate auth concerns from profile concerns from email concerns
class UserService {
  async register(data) { ... }    // core user lifecycle
  async updateProfile(userId, data) { ... }
}

class AuthService {
  async login(email, password) { ... }
  async resetPassword(token, password) { ... }
}

class EmailVerificationService {
  async sendVerification(userId) { ... }
  async verify(token) { ... }
  async forgotPassword(email) { ... }
}
```

**Steps:**
1. Identify which methods belong together (same data, same responsibility)
2. Create the new class, move the methods
3. Inject a reference to the new class where the old class used to call those methods
4. Run tests

---

### Decompose Conditional

Complex if/else logic into named predicates or strategy pattern.

```javascript
// Before: dense conditional logic
function calculateDiscount(user, order) {
  if (user.subscriptionTier === 'premium' && order.total > 100 && !order.hasDigitalItems) {
    return order.total * 0.15;
  } else if (user.loyaltyPoints > 1000 && order.total > 50) {
    return order.total * 0.10;
  } else if (user.referralCode && order.isFirstOrder) {
    return order.total * 0.20;
  }
  return 0;
}

// After: named predicates + early returns + clear structure
function calculateDiscount(user, order) {
  if (qualifiesForPremiumDiscount(user, order)) return order.total * 0.15;
  if (qualifiesForLoyaltyDiscount(user, order)) return order.total * 0.10;
  if (qualifiesForReferralDiscount(user, order)) return order.total * 0.20;
  return 0;
}

function qualifiesForPremiumDiscount(user, order) {
  return user.subscriptionTier === 'premium'
    && order.total > 100
    && !order.hasDigitalItems;
}

function qualifiesForLoyaltyDiscount(user, order) {
  return user.loyaltyPoints > 1000 && order.total > 50;
}

function qualifiesForReferralDiscount(user, order) {
  return Boolean(user.referralCode) && order.isFirstOrder;
}
```

---

### Introduce Parameter Object

When a function takes many related parameters, group them.

```javascript
// Before: many parameters
async function createReport(startDate, endDate, userId, tenantId, format, includeArchived) {
  // ...
}

// After: parameter object
async function createReport({ startDate, endDate, userId, tenantId, format, includeArchived = false }) {
  // ...
}

// Callers become readable:
await createReport({
  startDate: new Date('2024-01-01'),
  endDate: new Date('2024-12-31'),
  userId: currentUser.id,
  tenantId: currentTenant.id,
  format: 'pdf'
  // includeArchived defaults to false
});
```

---

### Replace Magic Numbers/Strings

```javascript
// Before
if (retries > 3) { ... }
if (user.tier === 2) { ... }
setTimeout(fn, 300000);

// After
const MAX_RETRY_ATTEMPTS = 3;
const USER_TIERS = Object.freeze({ FREE: 0, BASIC: 1, PREMIUM: 2 });
const SESSION_TIMEOUT_MS = 5 * 60 * 1000;  // 5 minutes, expressed as math

if (retries > MAX_RETRY_ATTEMPTS) { ... }
if (user.tier === USER_TIERS.PREMIUM) { ... }
setTimeout(fn, SESSION_TIMEOUT_MS);
```

---

### Refactoring Sequence

For any significant refactoring:

1. **Ensure tests exist** (write characterization tests if needed)
2. **Run tests — note starting state** (which pass, which fail)
3. **Small move**: extract, rename, or reorganize one thing
4. **Run tests** — must still pass
5. **Commit the step** (small, focused commits are reviewable)
6. **Repeat** steps 3-5

Never accumulate multiple refactoring moves before testing. If tests break, you know exactly which move caused it.

---

### Output Format

```
## Refactoring Plan: [File/Class Name]

### Current Issues
[Code smells identified, with references to specific line numbers]

### Refactoring Steps

#### Step 1: [Refactoring name]
Before:
```js
[original code]
```
After:
```js
[refactored code]
```
Why: [rationale]

#### Step 2: ...

### Test Strategy
[What tests to write or verify before starting]

### Risk Assessment
[What could break, how to detect it]
```
