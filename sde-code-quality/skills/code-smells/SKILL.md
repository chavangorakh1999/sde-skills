---
name: code-smells
description: "Identify and prescribe fixes for: god class, feature envy, long parameter list, shotgun surgery, data clumps, primitive obsession, dead code. Use when code feels wrong but you can't articulate why."
---

## Code Smells

Code smells are symptoms of deeper design problems — not bugs, but signals that the code will become harder to change. Each smell has a name (from Fowler's catalog), a symptom, and a proven fix.

### Context

Code to analyze: **$ARGUMENTS**

---

### God Class / God Object

**Symptom:** A class with 500+ lines, 20+ methods, responsible for everything.

```javascript
// Example: UserManager that does everything
class UserManager {
  async register() { ... }
  async login() { ... }
  async resetPassword() { ... }
  async updateProfile() { ... }
  async uploadAvatar() { ... }
  async sendNotification() { ... }
  async generateReport() { ... }
  async calculateMetrics() { ... }
  async exportData() { ... }
  // ... 15 more methods
}

// Why it's a problem:
// - Single point of merge conflicts (everyone touches it)
// - Can't test parts in isolation
// - Any change risks breaking unrelated behavior

// Fix: extract classes by business concept
// UserAuthService, UserProfileService, UserNotificationService,
// UserAnalyticsService — each with clear, narrow responsibility
```

---

### Feature Envy

**Symptom:** A method that is more interested in the data of another class than its own.

```javascript
// Bad: OrderProcessor reaches into Order's guts to calculate discount
class OrderProcessor {
  calculateDiscount(order) {
    if (order.customer.subscriptionTier === 'premium'
        && order.items.length > 3
        && order.createdAt.getMonth() === 11) {  // December
      return order.subtotal * 0.15;
    }
    return 0;
  }
}
// This method uses Order data, Customer data — it belongs ON Order

// Fix: move the method to the class it envies
class Order {
  calculateDiscount() {
    if (this.customer.subscriptionTier === 'premium'
        && this.items.length > 3
        && this.isHolidaySeason()) {
      return this.subtotal * 0.15;
    }
    return 0;
  }

  isHolidaySeason() {
    return this.createdAt.getMonth() === 11;
  }
}
```

---

### Long Parameter List

**Symptom:** A function with 4+ parameters.

```javascript
// Bad
function createReport(userId, startDate, endDate, format, includeArchived, tenantId, groupBy) {
  // ...
}

// Problems: easy to pass args in wrong order, hard to read call sites,
// adding another param requires updating every caller

// Fix 1: parameter object
function createReport({ userId, startDate, endDate, format, includeArchived = false, tenantId, groupBy = 'day' }) { }

// Fix 2: builder pattern (for complex construction)
class ReportBuilder {
  forUser(userId) { this.userId = userId; return this; }
  between(startDate, endDate) { this.startDate = startDate; this.endDate = endDate; return this; }
  asFormat(format) { this.format = format; return this; }
  build() { return new Report(this); }
}

const report = new ReportBuilder()
  .forUser(userId)
  .between(startDate, endDate)
  .asFormat('pdf')
  .build();
```

---

### Shotgun Surgery

**Symptom:** Making one change requires editing many files.

```javascript
// Bad: notification logic spread across 8 files
// When "add push notification" comes, you edit:
// UserService.js, OrderService.js, PaymentService.js,
// AuthService.js, config.js, constants.js, logger.js, types.ts

// Fix: centralize the concept in one place
// All notification sending goes through NotificationService
// One change to add push: edit NotificationService + add PushChannel
// Other services just call notificationService.send(user, 'order-placed', data)
```

---

### Data Clumps

**Symptom:** The same group of data items always appear together.

```javascript
// Bad: first/last name, city/state/zip always travel together
function createUser(firstName, lastName, email, street, city, state, zip) { }
function updateAddress(userId, street, city, state, zip) { }
function validateAddress(street, city, state, zip) { }

// Fix: create value objects for cohesive data
class FullName {
  constructor(first, last) {
    if (!first || !last) throw new Error('First and last name required');
    this.first = first;
    this.last = last;
  }
  get full() { return `${this.first} ${this.last}`; }
}

class Address {
  constructor({ street, city, state, zip }) {
    this.street = street; this.city = city;
    this.state = state; this.zip = zip;
  }
  validate() { /* validation logic lives here */ }
}

function createUser(name, email, address) { }  // clean signature
```

---

### Primitive Obsession

**Symptom:** Using primitives (string, number, boolean) where a domain concept should exist.

```javascript
// Bad: Money as a plain number
const price = 9.99;
const total = price * 1.1;  // floating-point arithmetic on money = bugs
// 0.1 + 0.2 = 0.30000000000000004 in JavaScript

// Fix: Money value object
class Money {
  constructor(amountInCents, currency = 'USD') {
    if (!Number.isInteger(amountInCents)) throw new Error('Amount must be integer cents');
    this.amountInCents = amountInCents;
    this.currency = currency;
  }

  add(other) {
    if (this.currency !== other.currency) throw new Error('Currency mismatch');
    return new Money(this.amountInCents + other.amountInCents, this.currency);
  }

  multiply(factor) {
    return new Money(Math.round(this.amountInCents * factor), this.currency);
  }

  format() {
    return new Intl.NumberFormat('en-US', { style: 'currency', currency: this.currency })
      .format(this.amountInCents / 100);
  }
}

// Bad: UserId as string (any string accepted)
async function getUser(userId) { }  // what format? UUID? email? integer?

// Fix: strong type
class UserId {
  constructor(value) {
    if (!isValidUUID(value)) throw new Error(`Invalid user ID: ${value}`);
    this.value = value;
  }
}

// Bad: status as magic string
user.status = 'active';  // what are valid values? Who knows?

// Fix: enum-like object
const UserStatus = Object.freeze({
  PENDING:   'pending',
  ACTIVE:    'active',
  SUSPENDED: 'suspended',
  DELETED:   'deleted'
});
user.status = UserStatus.ACTIVE;
```

---

### Dead Code

**Symptom:** Code that is never executed, called, or imported.

```javascript
// Types:
// - Commented-out code (just delete it — git history keeps it)
// - Unreachable code after return/throw
// - Functions/methods never called (search for references)
// - Variables assigned but never read
// - Feature flags that are always false/true (clean them up)
// - TODO comments from 2 years ago that never happened

// Detection:
// ESLint: no-unused-vars, no-unreachable, no-dead-code
// TypeScript: unused variables and imports flagged by compiler
// Istanbul/nyc: coverage report shows uncovered lines

// Dead feature flag example:
if (process.env.FEATURE_NEW_CHECKOUT === 'true') {
  // New checkout — always true since January, never cleaned up
  // Old path below is dead code
} else {
  oldCheckout();  // dead — never reached
}
// Fix: remove the flag, keep the new checkout, delete the old code
```

---

### Divergent Change

**Symptom:** One class changes for many different reasons.

```javascript
// Bad: UserReport changes when:
// - Report format changes (CSV, PDF, Excel)
// - User data model changes (add phone number)
// - Report delivery method changes (email, S3, download)
// - Report filters change (by status, date range, region)

// These are four independent axes of variation — four reasons to change
// Fix: separate formatters (CsvFormatter, PdfFormatter),
//      a data fetcher (UserReportDataService),
//      and a delivery service (ReportDeliveryService)
```

---

### Comments That Explain What (Not Why)

```javascript
// Bad: comment restates the code
// Increment i by 1
i++;

// Also bad: comment covers up bad code
// Check if user is authorized to access this resource and if the token has not expired
// and if the user's account is not suspended
if (user && user.token && !user.tokenExpired && user.status !== 'suspended' && user.roles.includes('admin')) {

// Fix the code, not the comment
const isAuthorizedAdmin = user?.status === 'active'
  && !user.tokenExpired
  && user.roles.includes('admin');

if (isAuthorizedAdmin) { ... }
// Now no comment needed — the code explains itself

// Good comments: explain WHY, not WHAT
// Skip the first result — API always returns a header row we don't need
const dataRows = results.slice(1);
```

---

### Output Format

```
## Code Smell Analysis: [File/Module]

### Smells Detected

#### [Smell Name] — [file.js:line-range]
Symptom: [What you see in the code]
Problem: [Why this makes the code worse]
Refactoring: [Named refactoring from Fowler's catalog]
Fix:
```js
[Before / After if space allows]
```

### Priority Order
1. [Most impactful smell to fix first — highest change frequency or largest blast radius]
2. ...

### Quick Wins (< 30 min each)
[Dead code removal, magic numbers, rename variables]
```
