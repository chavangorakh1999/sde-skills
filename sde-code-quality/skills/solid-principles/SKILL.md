---
name: solid-principles
description: "Apply SOLID, DRY, YAGNI, Law of Demeter in real JavaScript/Node.js code — with before/after examples and common misapplications. Use when reviewing code design or explaining why a design feels wrong."
---

## SOLID Principles in JavaScript

SOLID describes properties of maintainable code, not rules to follow blindly. Applied well, SOLID makes code easier to test, extend, and understand. Applied poorly, it produces unnecessary abstraction and complexity.

### Context

Code or design to evaluate: **$ARGUMENTS**

---

### S — Single Responsibility Principle

A class/function should have **one reason to change**.

"Reason to change" means business concern, not "one line of code."

```javascript
// Violation: UserController handles HTTP, business logic, and data access
class UserController {
  async register(req, res) {
    // HTTP parsing
    const { email, password } = req.body;

    // Business logic
    if (await User.exists({ email })) {
      return res.status(409).json({ error: 'Email taken' });
    }
    const hash = await bcrypt.hash(password, 12);

    // Data access
    const user = await User.create({ email, passwordHash: hash });

    // Email sending
    await mailer.send(email, 'Welcome!', welcomeTemplate(user));

    res.status(201).json({ id: user.id, email });
  }
}

// Fixed: each layer has one reason to change
class UserController {
  constructor(userService) { this.userService = userService; }

  async register(req, res, next) {
    try {
      const user = await this.userService.register(req.body);
      res.status(201).json(user);
    } catch (err) {
      next(err);  // HTTP layer only: parse request, delegate, format response
    }
  }
}

class UserService {
  constructor(userRepo, mailer) { this.userRepo = userRepo; this.mailer = mailer; }

  async register({ email, password }) {
    if (await this.userRepo.existsByEmail(email)) {
      throw new ConflictError('Email already registered');
    }
    const passwordHash = await bcrypt.hash(password, 12);
    const user = await this.userRepo.create({ email, passwordHash });
    await this.mailer.sendWelcome(user);
    return { id: user.id, email: user.email };
  }
}
// UserRepo: data access concern. Mailer: notification concern.
```

**SRP warning signs:** "and", "also", "or" in the description of what a class does.

---

### O — Open/Closed Principle

Open for extension, closed for modification. Add new behavior without editing existing code.

```javascript
// Violation: adding a new notification type requires editing NotificationService
class NotificationService {
  async send(user, type, data) {
    if (type === 'email') {
      await this.emailClient.send(user.email, data.subject, data.body);
    } else if (type === 'sms') {
      await this.smsClient.send(user.phone, data.message);
    } else if (type === 'push') {
      // Adding push requires editing this file — violates OCP
      await this.pushClient.send(user.deviceToken, data);
    }
  }
}

// Fixed: strategy pattern — new notification type = new class, no editing existing
class NotificationService {
  constructor(channels) {
    this.channels = new Map(channels.map(c => [c.name, c]));
  }

  async send(user, channelName, data) {
    const channel = this.channels.get(channelName);
    if (!channel) throw new Error(`Unknown notification channel: ${channelName}`);
    await channel.send(user, data);
  }
}

class EmailChannel {
  get name() { return 'email'; }
  async send(user, { subject, body }) {
    await emailClient.send(user.email, subject, body);
  }
}

class SmsChannel {
  get name() { return 'sms'; }
  async send(user, { message }) {
    await smsClient.send(user.phone, message);
  }
}

// Adding push: create PushChannel class, register it. No existing code changes.
const notificationService = new NotificationService([
  new EmailChannel(), new SmsChannel(), new PushChannel()
]);
```

**OCP in the real world:** You don't need OCP everywhere. Apply it at the boundaries where extension is genuinely expected. Over-applying OCP leads to unnecessary abstraction.

---

### L — Liskov Substitution Principle

Subtypes must be substitutable for their base type without breaking correctness.

```javascript
// Violation: ReadOnlyList extends Array but breaks the contract
class ReadOnlyList extends Array {
  push() { throw new Error('Cannot modify ReadOnlyList'); }
  pop()  { throw new Error('Cannot modify ReadOnlyList'); }
}

// Code that uses Array expects push/pop to work — ReadOnlyList breaks it
function addItem(list, item) { list.push(item); }  // throws for ReadOnlyList
// Liskov violation: subclass breaks assumptions of the parent type

// Fix: don't inherit Array; implement a read-only interface separately
class ReadOnlyList {
  constructor(items) { this._items = [...items]; }
  [Symbol.iterator]() { return this._items[Symbol.iterator](); }
  get length() { return this._items.length; }
  at(index) { return this._items.at(index); }
  // No push, pop — the contract doesn't include mutation
}
```

**Practical LSP test:** If you can replace a subclass with any other subclass and the calling code works correctly, LSP is satisfied.

---

### I — Interface Segregation Principle

Don't force classes to implement methods they don't use. Prefer narrow interfaces over fat ones.

```javascript
// In JavaScript (duck typing), ISP means: don't require more than you need

// Violation: UserRepository required to implement methods not all callers use
class UserRepository {
  findById(id) { ... }
  findByEmail(email) { ... }
  save(user) { ... }
  delete(id) { ... }
  bulkImport(users) { ... }  // only used by admin import tool
  generateReport() { ... }  // only used by analytics
}

// In tests, you now have to stub bulkImport and generateReport even for simple tests

// Fix: separate repositories by consumer
class UserReadRepository  { findById(id) { ... } findByEmail(email) { ... } }
class UserWriteRepository { save(user) { ... }   delete(id) { ... } }
class UserAdminRepository { bulkImport(users) { ... } generateReport() { ... } }

// Each consumer depends only on the interface it uses
class AuthService {
  constructor(userReadRepo) { this.userRepo = userReadRepo; }
  // Only needs findByEmail — doesn't know about save, delete, bulkImport
}
```

---

### D — Dependency Inversion Principle

High-level modules should not depend on low-level modules. Both should depend on abstractions. Concretely: inject dependencies rather than creating them.

```javascript
// Violation: UserService creates its own dependencies (hardcoded)
class UserService {
  async getUser(id) {
    const db = new PostgresDB(process.env.DB_URL);  // hardcoded dependency
    return db.query('SELECT * FROM users WHERE id = $1', [id]);
  }
  // Can't test without a real PostgreSQL connection
  // Can't swap to MongoDB without editing UserService
}

// Fixed: dependencies injected (from outside)
class UserService {
  constructor(userRepository) {
    this.userRepository = userRepository;  // depends on abstraction (duck type)
  }

  async getUser(id) {
    return this.userRepository.findById(id);
  }
}

// Production: inject real PostgreSQL repo
const userService = new UserService(new PostgresUserRepository(db));

// Tests: inject in-memory fake
const userService = new UserService(new InMemoryUserRepository());

// DI container example (manual, simple)
function createApp() {
  const db = new PostgresDB(config.dbUrl);
  const userRepo = new PostgresUserRepository(db);
  const mailer = new SendGridMailer(config.sendgridKey);
  const userService = new UserService(userRepo, mailer);
  const userController = new UserController(userService);
  return { userController };
}
```

---

### DRY — Don't Repeat Yourself

Every piece of knowledge should have a single, authoritative representation.

```javascript
// DRY violation: validation logic duplicated
// In registration handler:
if (password.length < 8) return res.status(400).json({ error: 'Password too short' });

// In password reset handler:
if (newPassword.length < 8) return res.status(400).json({ error: 'Password too short' });

// Fix: single validator
const validatePassword = (password) => {
  if (password.length < 8) throw new ValidationError('Password must be at least 8 characters');
};

// DRY ≠ avoid duplication at all costs
// Three identical 2-line utility functions might be fine — don't over-abstract
// The rule is "knowledge", not "code": if they'd change for different reasons,
// duplication is preferable to a coupled abstraction
```

---

### YAGNI — You Aren't Gonna Need It

Don't build for hypothetical future requirements.

```javascript
// YAGNI violation: building a plugin system nobody asked for
class NotificationService {
  registerPlugin(plugin) { this.plugins.push(plugin); }  // for future extensibility
  executePlugins(event) { this.plugins.forEach(p => p.execute(event)); }
  // ... actual notification logic buried under plugin system

// Fix: just send the notification
class NotificationService {
  async sendEmail(to, subject, body) {
    await this.emailClient.send(to, subject, body);
  }
  // When you actually need plugins, add them then
}
```

---

### Law of Demeter — "Talk to friends, not strangers"

A method should only call methods on: itself, its parameters, objects it creates, its direct fields.

```javascript
// Violation: train wreck (method chaining through unrelated objects)
const city = user.getAddress().getZipCode().getCity();
// UserController knows about Address, ZipCode, City — tightly coupled

// Fix: ask for what you need, not for objects to navigate
const city = user.getCity();  // User knows how to get its own city

// In practice: avoid chains longer than 2 dots into foreign objects
// Exception: fluent interfaces (Promise.then().catch()) and chainable builders
//            are fine — they're returning 'self', not navigating foreign objects
```

---

### Output Format

```
## SOLID Analysis: [Code/Component]

### Violations Found
[For each: principle, location, what's wrong, why it matters, concrete fix]

### What's Applied Well
[Principles correctly applied in the code]

### Refactoring Priority
1. [Most impactful fix first]
2. ...

### Over-Engineering Risk
[Any places where applying SOLID would add complexity without benefit]
```
