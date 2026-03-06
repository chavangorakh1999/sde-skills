---
description: Full PR review with CRITICAL/MAJOR/MINOR/NIT findings and concrete fixes across correctness, security, performance, resilience, design, testability, and readability
argument-hint: "<paste diff or file>"
---

# /review-code -- PR Code Review

Systematic review across 7 dimensions. Every finding includes severity, location, problem, and a concrete fix. Chains pr-review + security-review.

## Invocation

```
/review-code [paste git diff or file contents]
/review-code                    # asks for code to review
```

## Workflow

### Step 1: Get the Code

If code is not provided, ask: "Please paste the diff, file, or describe what you'd like reviewed."

Accept: git diff output, single file contents, function/class snippet.

Note: the larger the diff, the more likely to miss subtle issues. If diff > 500 lines, ask if there's a specific area to focus on.

### Step 2: Understand Intent

Before reviewing, briefly state your understanding:
- "This looks like a new user registration endpoint."
- "This is a refactoring of the payment service."

If you misunderstand, the user can correct before you spend time on the wrong review.

### Step 3: Security Pass (First)

Apply **security-review** skill first — security issues are blockers.

Check:
- [ ] Injection vulnerabilities (SQL, NoSQL, command, template)
- [ ] Authentication: is auth checked on every state-changing route?
- [ ] Authorization: horizontal privilege escalation possible?
- [ ] Secrets: any hardcoded tokens, keys, passwords?
- [ ] Sensitive data in logs or responses (passwords, tokens, PII)
- [ ] Mass assignment: is req.body spread directly onto model?
- [ ] Rate limiting on sensitive endpoints (login, registration, password reset)

### Step 4: Correctness Pass

- Async/await correctness (missing await, forEach + async, unhandled promises)
- Logic errors (off-by-one, wrong comparisons, null dereferencing)
- Race conditions on shared state
- Error handling (try/catch, rejection handling, error propagation)

### Step 5: Performance Pass

- N+1 queries
- Missing pagination on collection endpoints
- Missing DB indexes (infer from query patterns)
- Synchronous operations blocking the event loop
- Unnecessary data fetching (SELECT * when only a few fields needed)

### Step 6: Resilience Pass

- External calls: do they have timeouts?
- Third-party API calls: is there error handling and retry logic?
- Message queue consumers: is there a DLQ?
- Graceful shutdown handling?

### Step 7: Design Pass

Apply **solid-principles** and **code-smells** lenses:
- SRP violations (class doing too many things)
- Missing dependency injection (hardcoded dependencies)
- Feature envy (method using another class's data more than its own)
- Long parameter lists

### Step 8: Testability Pass

- Are dependencies injectable for mocking?
- Are functions doing too many things to test in isolation?
- Is there any untestable code (hardcoded dates, globals, side effects)?

### Step 9: Readability Pass

- Intention-revealing names
- Magic numbers/strings
- Comment quality (explains why, not what)
- Consistent patterns with the rest of the codebase

### Step 10: Compile Findings and Summarize

Produce findings in the standard format. Conclude with merge recommendation.

## Output

```
## Code Review: [Feature/File Name]

### Summary
[2-3 sentences: overall quality, number of blocking issues, merge recommendation]

### Findings

[CRITICAL] path/file.js:42 — Security: SQL Injection
  Problem: req.query.id concatenated directly into query string
  Fix: db.query('SELECT * FROM users WHERE id = $1', [req.query.id])

[MAJOR] services/order.js:88 — Resilience: No timeout on Stripe call
  Problem: stripe.charges.create() has no timeout — hangs indefinitely on Stripe slowness
  Fix: await Promise.race([stripe.charges.create(data), timeout(5000)])

[MINOR] controllers/user.js:15 — Correctness: Missing await in loop
  Problem: items.forEach(async item => ...) — forEach ignores promises, errors swallowed
  Fix: await Promise.all(items.map(item => processItem(item)))

[NIT] utils/format.js:3 — Rename d to date

### What's Done Well
- Error handling with typed errors
- Input validation with Joi before DB queries
- Consistent response format

### Merge Recommendation
[ ] Block — fix CRITICAL issues before merge
```

## Next Steps

- "Want me to refactor the problematic sections? -> `/refactor [code]`"
- "Should I write tests for this code? -> `/write-tests [code]`"
- "Want a security deep-dive? -> `/review-code` with security focus"
