---
description: Safe step-by-step refactoring plan with before/after code and test safety net
argument-hint: "<code or description of what needs refactoring>"
---

# /refactor -- Safe Code Refactoring

Produce a safe, step-by-step refactoring plan. Chains code-smells -> refactoring -> solid-principles.

## Invocation

```
/refactor This auth service has grown to 900 lines and handles 12 different things
/refactor [paste code that needs cleaning up]
/refactor The UserController has all business logic inside route handlers
```

## Workflow

### Step 1: Understand the Code

If code is pasted, read it. If only description is given, ask for the relevant file or function.

State what you understand:
- What does this code currently do?
- What problems is the user trying to solve?
- Any constraints? (can't break the API, must keep tests passing, refactoring must be done incrementally)

### Step 2: Identify Smells

Apply **code-smells** skill:
- Name each smell found (using Fowler's vocabulary)
- Rank by: how much it hurts readability/testability/maintainability
- Group related smells (often a God Class also has Long Methods and Feature Envy)

### Step 3: Check for Safety Net

Ask: "Do you have tests covering this code?"
- Yes: great, proceed
- No: recommend writing characterization tests first
  - "Before we refactor, let's write tests documenting current behavior. This takes 15 minutes and makes the refactoring safe."

### Step 4: Design the Target State

Apply **solid-principles** to design what the code should look like:
- What classes/functions should exist after the refactoring?
- What are their responsibilities?
- How do they communicate?

Draw a brief before/after comparison:
```
Before: UserService (400 lines, handles auth + profile + email + billing)
After:
  - AuthService (login, register, tokens) — 100 lines
  - UserProfileService (update, avatar) — 80 lines
  - UserNotificationService (email, preferences) — 60 lines
  - BillingService (subscription, payment) — 80 lines
```

### Step 5: Write the Step-by-Step Plan

Apply **refactoring** skill for the specific moves needed:

Each step must:
- Be independently safe (tests still pass after this step alone)
- Be small enough to review in one commit
- Use the Fowler refactoring name where applicable

```
Step 1: Extract validateRegistration() from register() [Extract Method]
Step 2: Extract UserAuthService class with register() and login() [Extract Class]
Step 3: Inject dependencies (UserRepository, Mailer) into AuthService [DI]
Step 4: Move email methods to UserNotificationService [Move Method]
Step 5: Update imports and factory/DI wiring
```

### Step 6: Show Before/After Code

For the most important steps, show the actual before/after code:

```javascript
// Before
class UserService {
  async register({ email, password }) {
    // 50 lines of mixed validation, bcrypt, DB, email sending
  }
}

// After
class UserService {
  constructor(authService, profileService) { ... }
}

class AuthService {
  constructor(userRepo, mailer) { ... }

  async register({ email, password }) {
    const validatedData = validateRegistration({ email, password });
    const existingUser = await this.userRepo.findByEmail(validatedData.email);
    if (existingUser) throw new ConflictError('Email already registered');
    const passwordHash = await bcrypt.hash(validatedData.password, 12);
    const user = await this.userRepo.create({ email, passwordHash });
    await this.mailer.sendWelcome(user);
    return sanitizeUser(user);
  }
}
```

### Step 7: Risk Assessment

For each step, note:
- What could break?
- How to detect a regression (which test to run)
- Rollback: can I revert just this step?

### Step 8: Offer Next Steps

- "Want me to help write tests before we start? -> `/test-strategy [feature]`"
- "Should I review the refactored code? -> `/review-code [refactored code]`"
- "Want to identify more smells in this file? -> `/smell-check [code]`"

## Output

```
## Refactoring Plan: [File/Service Name]

### Current State
[Code smells identified with severity]

### Target State
[Diagram or description of the after-refactoring structure]

### Safety Net
[Tests to write before starting]

### Steps

#### Step 1: [Refactoring Name]
Before: [code snippet]
After:  [code snippet]
Why: [rationale]
Risk: [what could break]

#### Step 2: ...

### Estimated Effort
[Time estimate for the full refactoring]

### Incremental Plan
[If large refactoring: sequence to do it over multiple PRs]
```
