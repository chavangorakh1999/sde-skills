---
name: design-patterns
description: "GoF patterns with JavaScript implementation examples: when to use, when to avoid, what problem each actually solves. Use when you recognize a recurring design problem and want a proven solution."
---

## Design Patterns

Design patterns are solutions to recurring problems in software design — they're vocabulary, not templates. Know the problem each pattern solves, not just how to implement it.

### Context

Problem or pattern to explore: **$ARGUMENTS**

---

### Creational Patterns

#### Factory / Factory Method

**Problem:** Creating objects without specifying the exact class.

```javascript
// When to use: creating objects based on a config/type, hiding construction complexity,
//              centralizing object creation for testing

// Simple factory (not in GoF, but very common)
class NotificationFactory {
  static create(type, config) {
    switch (type) {
      case 'email': return new EmailNotification(config);
      case 'sms':   return new SmsNotification(config);
      case 'push':  return new PushNotification(config);
      default: throw new Error(`Unknown notification type: ${type}`);
    }
  }
}

// Usage
const notification = NotificationFactory.create(user.preferredNotification, config);
await notification.send(user, message);

// When to avoid: if there's only one type, or if switch doesn't grow
```

#### Builder

**Problem:** Constructing complex objects step by step.

```javascript
// When to use: many optional constructor parameters, multi-step construction,
//              readable test data setup

class QueryBuilder {
  constructor() {
    this._table = null;
    this._conditions = [];
    this._orderBy = null;
    this._limit = null;
  }

  from(table) { this._table = table; return this; }
  where(condition) { this._conditions.push(condition); return this; }
  orderBy(field, direction = 'ASC') { this._orderBy = `${field} ${direction}`; return this; }
  limit(n) { this._limit = n; return this; }

  build() {
    if (!this._table) throw new Error('Table is required');
    let query = `SELECT * FROM ${this._table}`;
    if (this._conditions.length) query += ` WHERE ${this._conditions.join(' AND ')}`;
    if (this._orderBy) query += ` ORDER BY ${this._orderBy}`;
    if (this._limit) query += ` LIMIT ${this._limit}`;
    return query;
  }
}

const query = new QueryBuilder()
  .from('users')
  .where('status = $1')
  .where('created_at > $2')
  .orderBy('created_at', 'DESC')
  .limit(20)
  .build();
```

#### Singleton

**Problem:** Ensure a class has only one instance.

```javascript
// When to use: shared connection pools, configuration, loggers
// When NOT to use: for everything (Singleton is often a global variable in disguise)

// Module-level singleton (Node.js module system caches modules — this is free)
// db.js
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export default pool;

// Every import gets the same pool instance — no Singleton class needed
// import pool from './db'; — same pool everywhere

// Class-based singleton (if you need explicit control)
class Config {
  static #instance = null;

  static getInstance() {
    if (!Config.#instance) Config.#instance = new Config();
    return Config.#instance;
  }

  #constructor() {
    this.settings = loadSettingsFromEnv();
  }
}
// Warning: Singletons make testing hard. Prefer dependency injection.
```

---

### Structural Patterns

#### Adapter

**Problem:** Make incompatible interfaces work together.

```javascript
// When to use: integrating third-party libraries, migrating between implementations

// You have code that depends on this interface:
// { send(to, subject, body): Promise<void> }

// But the new email provider has this API:
// provider.messages.create({ from, to, subject, html })

class SendGridAdapter {
  constructor(sendgrid, fromEmail) {
    this.sendgrid = sendgrid;
    this.fromEmail = fromEmail;
  }

  async send(to, subject, body) {
    await this.sendgrid.request({
      method: 'POST',
      url: '/v3/mail/send',
      data: {
        from: { email: this.fromEmail },
        to: [{ email: to }],
        subject,
        content: [{ type: 'text/html', value: body }]
      }
    });
  }
}

// All existing code that uses mailer.send(to, subject, body) works unchanged
const mailer = new SendGridAdapter(sendgridClient, 'noreply@example.com');
```

#### Decorator

**Problem:** Add behavior to objects dynamically without subclassing.

```javascript
// When to use: cross-cutting concerns (logging, caching, timing, auth)
//              that don't belong in the core logic

// Core repository
class UserRepository {
  async findById(id) {
    return db.query('SELECT * FROM users WHERE id = $1', [id]);
  }
}

// Decorator: adds caching without changing UserRepository
class CachedUserRepository {
  constructor(inner, cache, ttl = 3600) {
    this.inner = inner;
    this.cache = cache;
    this.ttl = ttl;
  }

  async findById(id) {
    const key = `user:${id}`;
    const cached = await this.cache.get(key);
    if (cached) return JSON.parse(cached);

    const user = await this.inner.findById(id);
    if (user) await this.cache.setex(key, this.ttl, JSON.stringify(user));
    return user;
  }
}

// Decorator: adds timing
class TimedUserRepository {
  constructor(inner, metrics) { this.inner = inner; this.metrics = metrics; }

  async findById(id) {
    const start = Date.now();
    try {
      return await this.inner.findById(id);
    } finally {
      this.metrics.histogram('user_repo_find_by_id_ms', Date.now() - start);
    }
  }
}

// Compose decorators
const userRepo = new TimedUserRepository(
  new CachedUserRepository(new UserRepository(), redis),
  metrics
);
// Each decorator does one thing; the core stays clean
```

---

### Behavioral Patterns

#### Observer / Event Emitter

**Problem:** Define a one-to-many dependency so that when one object changes, its dependents are notified automatically.

```javascript
// Node.js EventEmitter is Observer built-in
import { EventEmitter } from 'events';

class OrderService extends EventEmitter {
  async place(orderData) {
    const order = await Order.create(orderData);
    this.emit('order.placed', order);  // notify all listeners
    return order;
  }
}

const orderService = new OrderService();

// Each listener handles its own concern independently
orderService.on('order.placed', async (order) => {
  await inventoryService.reserve(order.items);
});

orderService.on('order.placed', async (order) => {
  await emailService.sendConfirmation(order);
});

orderService.on('order.placed', async (order) => {
  await analyticsService.track('order_placed', { orderId: order.id, total: order.total });
});

// When to use: decoupling producers from consumers, cross-cutting reactions to domain events
// When to avoid: when you need guaranteed delivery (use a real message queue instead)
// Warning: unhandled EventEmitter errors can crash the process — always handle 'error' event
orderService.on('error', (err) => logger.error('OrderService error', err));
```

#### Strategy

**Problem:** Define a family of algorithms, encapsulate each one, and make them interchangeable.

```javascript
// When to use: multiple ways to do the same thing, switching algorithms at runtime

class PaymentProcessor {
  constructor(strategy) { this.strategy = strategy; }

  async processPayment(amount, metadata) {
    return this.strategy.charge(amount, metadata);
  }
}

class StripeStrategy {
  async charge(amount, { customerId }) {
    return stripe.paymentIntents.create({ amount, currency: 'usd', customer: customerId });
  }
}

class PayPalStrategy {
  async charge(amount, { paypalOrderId }) {
    return paypal.orders.capture(paypalOrderId);
  }
}

// Runtime strategy selection
const strategy = user.preferredPayment === 'paypal'
  ? new PayPalStrategy()
  : new StripeStrategy();

const processor = new PaymentProcessor(strategy);
await processor.processPayment(order.total, paymentMetadata);
```

#### Command

**Problem:** Encapsulate a request as an object, enabling queuing, logging, and undo.

```javascript
// When to use: job queues, undo/redo, operation logging, retry logic

class TransferMoneyCommand {
  constructor({ fromAccountId, toAccountId, amount, idempotencyKey }) {
    this.fromAccountId = fromAccountId;
    this.toAccountId = toAccountId;
    this.amount = amount;
    this.idempotencyKey = idempotencyKey;
  }

  async execute(accountService) {
    return accountService.transfer(
      this.fromAccountId,
      this.toAccountId,
      this.amount,
      this.idempotencyKey
    );
  }

  async undo(accountService) {
    // Reverse: transfer back
    return accountService.transfer(
      this.toAccountId,
      this.fromAccountId,
      this.amount,
      `undo-${this.idempotencyKey}`
    );
  }
}

// Command is now serializable for job queues
await queue.add('transfer-money', command);
```

#### Repository

**Problem:** Abstract the data access layer from business logic.

```javascript
// When to use: always (for anything beyond trivial scripts)
// Benefits: testability, swap storage layer, consistent querying interface

class UserRepository {
  constructor(db) { this.db = db; }

  async findById(id) {
    const { rows } = await this.db.query(
      'SELECT id, email, display_name, created_at FROM users WHERE id = $1 AND deleted_at IS NULL',
      [id]
    );
    return rows[0] ?? null;
  }

  async findByEmail(email) {
    const { rows } = await this.db.query(
      'SELECT * FROM users WHERE email = $1 AND deleted_at IS NULL',
      [email]
    );
    return rows[0] ?? null;
  }

  async save(user) {
    const { rows } = await this.db.query(
      `INSERT INTO users (id, email, display_name, password_hash)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (id) DO UPDATE SET display_name = $3, updated_at = NOW()
       RETURNING *`,
      [user.id, user.email, user.displayName, user.passwordHash]
    );
    return rows[0];
  }

  async softDelete(id) {
    await this.db.query(
      'UPDATE users SET deleted_at = NOW() WHERE id = $1',
      [id]
    );
  }
}
```

---

### Anti-Patterns to Name and Avoid

```javascript
// 1. Anemic Domain Model: classes with only getters/setters, no behavior
//    Fix: move logic into the domain objects

// 2. Service Locator: global registry of services (hidden dependencies)
//    Fix: inject dependencies explicitly

// 3. Singleton everywhere: using Singleton as a global variable substitute
//    Fix: dependency injection, module-level instances

// 4. Gold Plating: implementing features "just in case" (YAGNI violation)
//    Fix: build for today's requirements only

// 5. Premature abstraction: creating interfaces before you have 2+ implementations
//    "Rule of Three": abstract when you have 3 similar things, not 1
```

---

### Output Format

```
## Design Pattern Analysis: [Problem/Code]

### Pattern Identified / Recommended
[Pattern name, category (Creational/Structural/Behavioral)]

### Problem It Solves
[Concrete description of the problem in this specific context]

### Implementation
```js
[Code example]
```

### Tradeoffs
- Pro: [benefit]
- Con: [cost / when it over-engineers]

### When to Avoid This Pattern Here
[If there's a simpler alternative that fits better]
```
