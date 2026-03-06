---
description: Break an epic into INVEST-ready technical stories with acceptance criteria
argument-hint: "<epic description>"
---

# /write-tickets -- Technical Story Breakdown

Break an epic into technical stories with INVEST criteria, acceptance criteria, and dependency sequence. Chains tech-stories -> estimation.

## Invocation

```
/write-tickets Implement multi-factor authentication for all admin users
/write-tickets Add real-time notifications — WebSocket, push, email fallback
/write-tickets Migrate user data from legacy MySQL to PostgreSQL
/write-tickets                    # asks for epic description
```

## Workflow

### Step 1: Understand the Epic

Extract:
- What is the user-facing goal?
- What are the main functional requirements?
- Any non-functional requirements (performance, security, scale)?
- Any explicit out-of-scope items?
- Dependencies: does this require changes to other teams' services?

### Step 2: Identify Stories Using Vertical Slices

Apply **tech-stories** skill.

Each story should be: deployable independently, deliver testable value, completable in 1-3 days.

**Vertical slice** = includes everything needed for the feature to work end-to-end, even if it's a simplified version.

Bad: "Backend for MFA" (not deployable without frontend)
Good: "TOTP setup — admin can enable MFA via authenticator app" (full vertical slice)

**Technical enablers** = infrastructure work with no direct user-facing value:
- DB migrations
- New shared library
- New shared component

Write enablers as separate tickets, sequenced before the stories that need them.

### Step 3: Sequence and Dependencies

Draw the dependency graph. Mark which stories can be parallel.

### Step 4: Write Each Ticket

For each story and enabler, apply the full technical ticket template:
- Context (why)
- Scope (in / out)
- Technical approach (how — brief)
- Acceptance criteria (testable)
- Definition of done

### Step 5: Estimate

Apply **estimation** skill for each ticket:
- Three-point estimate (L/M/H)
- Flag risk factors

### Step 6: Sanity Check INVEST

For each ticket, verify:
- [ ] I: Independent (can be deployed alone)
- [ ] N: Negotiable (scope could be trimmed if needed)
- [ ] V: Valuable (delivers something testable)
- [ ] E: Estimable (team can size it)
- [ ] S: Small (< 3 days, otherwise split)
- [ ] T: Testable (acceptance criteria are verifiable)

## Output

```
## Story Breakdown: [Epic Name]

### Summary
[X stories, Y enablers, Z-W total estimated days]

### Dependency Sequence
Enabler A -> Story 1 -> Story 3
Enabler B (parallel with A) -> Story 2 (parallel with Story 1)

### Parallelization
[Stories that can be worked simultaneously]

---

## Enabler A: [Title]
**Context:** [Why this is needed]
**Scope:** In: [...] Out: [...]
**Technical approach:** [Brief]
**Acceptance criteria:**
- [ ] ...
**Estimate:** L: X, M: Y, H: Z (PERT: W days)

---

## Story 1: [Title]
[Full ticket template]
**Estimate:** L: X, M: Y, H: Z (PERT: W days)

---
[Continue for each story]
---

### Total Estimate
| Story | PERT | Risk |
...
**Total:** [X-Y days range]
```

## Next Steps

- "Want to plan the sprint? -> `/estimate [scope]`"
- "Should I write a deployment plan for these? -> `/deploy-plan [service]`"
