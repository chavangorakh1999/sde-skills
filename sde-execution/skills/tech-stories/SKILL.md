---
name: tech-stories
description: "Break epics into INVEST-ready technical tickets: context, scope boundaries, acceptance criteria, definition of done. Use when planning a sprint or breaking down a feature for a team."
---

## Technical Story Writing

Well-written technical tickets reduce estimation variance, prevent scope creep, and give engineers clear done criteria. Bad tickets either block engineers with ambiguity or balloon into 3-week monsters.

### Context

Epic or feature to break down: **$ARGUMENTS**

---

### INVEST Criteria

Every ticket should be:

- **I**ndependent — can be developed and deployed without waiting for another ticket
- **N**egotiable — scope can be adjusted without destroying the value
- **V**aluable — delivers something testable/reviewable, not just "half a feature"
- **E**stimable — team can size it; if not, it needs more discovery
- **S**mall — completable in 1-3 days of focused work; if not, split it
- **T**estable — clear acceptance criteria that can be verified

---

### Technical Ticket Template

```markdown
## [Short, specific title — verb + noun: "Add rate limiting to auth endpoints"]

### Context
[Why are we doing this? What problem does it solve?
Link to the epic, ADR, or product spec if relevant.
1-3 sentences max.]

### Scope
**In scope:**
- [Specific thing 1]
- [Specific thing 2]

**Out of scope (do not do this sprint):**
- [Thing that might be tempting but isn't in scope]
- [Related improvement to do separately]

### Technical Approach
[Brief description of the implementation approach — not full design, just enough for estimation.
For example: "Add express-rate-limit middleware with Redis store, apply to /auth/* routes"]

### Acceptance Criteria
- [ ] [Specific, testable criterion 1]
- [ ] [Specific, testable criterion 2]
- [ ] [Performance criterion: "Rate limiting adds < 5ms latency per request"]
- [ ] [Error handling: "Returns 429 with Retry-After header when limit exceeded"]

### Definition of Done
- [ ] Code reviewed and approved
- [ ] Unit tests written and passing
- [ ] Integration test covers the happy path and rate-limit-exceeded case
- [ ] No new security vulnerabilities (npm audit clean)
- [ ] Documentation updated (if public API affected)
- [ ] Deployed to staging and verified

### Dependencies
[List any other tickets this depends on, or any external dependencies]

### Notes / Open Questions
[Assumptions made, design questions to resolve before starting, gotchas]

### Estimate
Story points: [1/2/3/5/8] or T-shirt: [XS/S/M/L/XL]
```

---

### How to Break an Epic

```
Example epic: "Implement multi-factor authentication for all admin users"

Step 1: Identify the vertical slices (each slice delivers end-user value)
  Slice 1: TOTP setup flow (admin can enable MFA via authenticator app)
  Slice 2: TOTP verification at login (admin must enter TOTP to log in)
  Slice 3: MFA enforcement (all admin accounts require MFA by deadline)
  Slice 4: Recovery codes (admin can generate and use backup codes)

Step 2: Identify technical enablers (infrastructure needed, no user-facing value alone)
  Enabler A: Add MFA fields to users table (migration)
  Enabler B: Create TOTP utility module (speakeasy integration)

Step 3: Identify dependent sequence (what must come before what)
  Enabler A -> Enabler B -> Slice 1 -> Slice 2 -> Slice 3 -> Slice 4

Step 4: Write each as a ticket (see template above)
Step 5: Estimate each ticket independently
Step 6: Identify which can be parallelized (Slice 1 and Slice 4 share no code if backend is ready)
```

---

### Story Splitting Patterns

When a ticket is too big (> 5 days), split it:

```
1. By workflow step
   "Implement checkout" -> "Add cart review page" + "Add payment step" + "Add confirmation page"

2. By happy path vs edge cases
   "Handle payment" -> "Charge card (happy path)" + "Handle payment failure + retry" + "Handle refund"

3. By data complexity
   "Migrate users" -> "Migrate users without subscriptions" + "Migrate users with active subscriptions"

4. By performance tier
   "Implement search" -> "Basic search (correct, not optimized)" + "Add Redis caching" + "Add Elasticsearch"

5. By environment
   "Deploy feature" -> "Deploy to staging + internal test" + "Canary to 5% production" + "Full rollout"

Anti-pattern: splitting by frontend/backend is a bad split if it creates blocking dependency.
Better: full vertical slice (even if it's just one endpoint + one component).
```

---

### Output Format

For each story in the epic:

```
## Story Breakdown: [Epic Name]

### Summary
X stories identified, Y enablers, estimated Z-W total days

### Dependency Sequence
[Linear order if dependencies exist]

### Parallelization Opportunities
[Which stories can be worked simultaneously]

### Stories

---
[Full ticket template for each story]
---
```
