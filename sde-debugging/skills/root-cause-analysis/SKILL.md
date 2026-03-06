---
name: root-cause-analysis
description: "5 Whys applied systematically, fishbone (Ishikawa) diagram, fault tree analysis — structured to find fixable root causes, not symptoms. Use when debugging a recurring problem or writing a postmortem."
---

## Root Cause Analysis

The goal of RCA is not to assign blame — it's to find the systemic fix that prevents recurrence. "The deploy caused the outage" is not a root cause. "Our deploy process had no automated smoke test that would have caught this regression" is a root cause.

### Context

Problem or incident to analyze: **$ARGUMENTS**

---

### 5 Whys

Start with the symptom and ask "why?" until you reach something you can change.

```
Symptom: Users cannot log in (10:00 PM - 10:47 PM, 2024-01-15)

Why 1: Why couldn't users log in?
  -> The auth service was returning 500 errors

Why 2: Why was the auth service returning 500s?
  -> The DB connection pool was exhausted (0 available connections)

Why 3: Why was the connection pool exhausted?
  -> The N+1 query in the login handler was opening 50 connections per request
     (50 requests/sec × 50 connections = 2,500 connections needed vs pool limit 100)

Why 4: Why did the N+1 query exist in production?
  -> The code was merged without a code review that would have caught it

Why 5: Why wasn't it caught in code review?
  -> We don't have a review checklist that includes N+1 query detection
     AND we don't have query count assertions in our integration tests

Root cause: Missing automated safeguards (query count assertion in tests + N+1 check in PR review)

Action items:
  1. Add N+1 query count assertion to auth service integration tests
  2. Add "N+1 query check" to PR review checklist for DB-touching code
  3. Configure PgBouncer to improve connection pool resilience (short-term mitigation)
```

**Rules for good 5 Whys:**
- Each "why" must be a fact, not an assumption — if unsure, investigate first
- Stop when you reach something you can actually change (process, tool, test)
- "Human error" is never the root cause — it's a signal that the system failed to prevent the error
- There may be multiple root causes — trace all branches

---

### Fishbone (Ishikawa) Diagram

Use when there are multiple possible causes and you need to brainstorm before narrowing down.

```
Categories to explore:

PEOPLE:         Training gaps, unclear ownership, fatigue, understaffing
PROCESS:        Missing review steps, inadequate runbooks, unclear escalation
TECHNOLOGY:     Software bugs, capacity limits, missing monitoring, dependency failures
ENVIRONMENT:    Infrastructure changes, traffic spikes, external provider issues
MEASUREMENT:    Missing metrics, incorrect alerting, slow detection

Example: Database performance degradation

People:         DBA team had two people out — change was made without experienced review
Process:        No mandatory staging performance test before production deploy
Technology:     Index was dropped in migration (side effect not noticed)
                Query planner chose different execution plan after index drop
Environment:    Traffic spike (product launch) hit same day as migration
Measurement:    Query duration metric existed but no alert on p99 > 500ms
```

---

### Fault Tree Analysis

Work backwards from the failure, branching at each AND/OR gate.

```
Top event: Service unavailable (return 503)
    |
    OR gate (any of these cause unavailability)
    |
    +-- Database unreachable
    |       |
    |       OR gate
    |       |
    |       +-- Primary DB crash (hardware failure)
    |       +-- Network partition (primary isolated from app)
    |       +-- Connection pool exhausted (application bug)
    |
    +-- All API server instances unhealthy
    |       |
    |       AND gate (requires ALL instances to fail simultaneously)
    |       |
    |       +-- Deploy pushed broken build to all instances
    |       +-- No health check before routing traffic to new instances
    |
    +-- External dependency (Stripe) unavailable
            |
            AND gate (requires both)
            |
            +-- Stripe is down
            +-- No fallback / circuit breaker in our code
```

The OR gates show single points of failure. The AND gates show where multiple things must fail simultaneously — these are harder to trigger but often exist where we feel safe ("it can't fail that way").

---

### Corrective vs. Preventive Actions

Every action item should be SMART:
- **S**pecific: "Add query count assertion to auth tests" not "improve testing"
- **M**easurable: can we verify it's done?
- **A**ctionable: someone can actually do this
- **R**elevant: directly addresses the root cause
- **T**ime-bound: deadline assigned, owner named

```
Root cause: No automated smoke test after deploy

Corrective action (fix what's broken now):
  - Manually validate auth service health before next deploy
  - Add PgBouncer connection pooling as temporary mitigation

Preventive action (prevent recurrence):
  - ACTION: Add post-deploy smoke test for /health/ready + /api/auth/status
    Owner: @alice, Due: 2024-01-22
  - ACTION: Add N+1 assertion to auth integration tests (query count < 5 per login)
    Owner: @bob, Due: 2024-01-22
  - ACTION: Add PR review checklist item: "Does this touch DB queries? Check for N+1"
    Owner: @carol, Due: 2024-01-22

What to avoid:
  - "Be more careful in code review" — not actionable, not measurable
  - "Training on N+1 queries" — helpful but doesn't prevent future mistakes
  - Blame-based actions — blameless RCA produces better fixes
```

---

### Output Format

```
## Root Cause Analysis: [Problem/Incident]

### Timeline
[Sequence of events — when detected, when investigated, when resolved]

### 5 Whys Analysis
Why 1: [Symptom] -> because [Immediate cause]
Why 2: [Immediate cause] -> because [Underlying cause]
...
Root cause: [The systemic issue you can fix]

### Contributing Factors
[Other conditions that made this worse or enabled it]

### Root Causes
1. [Root cause 1] — directly caused the failure
2. [Root cause 2] — enabled the failure to persist/spread

### Action Items

| # | Action | Type | Owner | Due Date | Status |
|---|--------|------|-------|----------|--------|

### What Prevented Earlier Detection?
[Monitoring gaps, missing alerts, no smoke tests]

### What Limited the Impact?
[If applicable: what safety mechanisms worked?]
```
