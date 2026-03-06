---
name: deployment-plan
description: "Release checklist: pre-deploy (DB migration, rollback plan), deploy (canary/blue-green), smoke tests, rollback triggers, post-deploy monitoring. Use when planning a production deployment."
---

## Deployment Plan

Production deployments are experiments. Treat them like one: have a hypothesis (new code is better), control the blast radius (canary), and have a rollback if the hypothesis is wrong.

### Context

Service or feature to deploy: **$ARGUMENTS**

---

### Pre-Deployment Checklist

```
Code readiness:
[ ] All acceptance criteria verified in staging
[ ] Code review approved by 2+ engineers
[ ] No CRITICAL or MAJOR findings from last security review
[ ] Test suite passing (unit + integration + smoke)
[ ] npm audit clean (no high/critical vulnerabilities)
[ ] Feature flags configured (if rolling out gradually)

Database:
[ ] Migrations reviewed — are they backward-compatible? (app v_new must work with DB v_old during deploy)
[ ] Migration is zero-downtime? (no long locks, uses CONCURRENTLY for indexes)
[ ] Rollback migration written and tested (can we undo the DB change?)
[ ] Migration timed in staging (if > 30s on production data size, plan carefully)
[ ] Backup taken (or confirm automated backup is recent)

Rollback:
[ ] Previous version identified and tagged
[ ] Rollback procedure documented and tested
[ ] Time estimate for rollback: < 5 minutes

Communication:
[ ] Stakeholders notified (especially if downtime window)
[ ] On-call engineer aware and available
[ ] Incident channel pre-created (#deploy-YYYYMMDD-service)
```

---

### Zero-Downtime Deploy Patterns

```
Blue-Green:
  - Maintain two identical environments (blue = current, green = new)
  - Deploy to green, test it, flip load balancer to green
  - Blue stays up for instant rollback (just flip LB back)
  - Cost: double infrastructure during deploy
  - Best for: high-risk deploys where instant rollback is critical

Canary Deploy:
  - Route small % of traffic to new version
  - Monitor for X minutes at each percentage
  - Gradually increase: 1% -> 5% -> 25% -> 100%
  - Rollback: route 0% to new version (instant)
  - Best for: code changes, most common pattern

Rolling Update:
  - Replace instances one at a time (old still serving while new comes up)
  - No extra infrastructure needed
  - Rollback: slower (must roll back each instance)
  - Kubernetes default: maxSurge: 1, maxUnavailable: 0
  - Best for: low-risk deploys, stateless services

Feature Flag Deploy:
  - Deploy code to 100% of instances, but feature is OFF by default
  - Turn on gradually via flag for % of users
  - Instant rollback: flip flag to 0%
  - Best for: large features, risky functionality
```

---

### Deployment Checklist

```
T-15 minutes:
[ ] Confirm no active incidents (don't deploy into a fire)
[ ] Confirm on-call is available
[ ] Pull baseline metrics snapshot (current p99, error rate, QPS)

T-0 (deployment start):
[ ] Run DB migrations (if any) — verify they complete successfully
[ ] Deploy canary (1-5% of instances/traffic)
[ ] Monitor for 5 minutes: error rate, latency, any new error messages

T+5 minutes (after canary):
[ ] Error rate: equal to baseline? (pass) or elevated? (STOP, investigate)
[ ] P99 latency: within 10% of baseline? (pass) or significantly higher? (STOP)
[ ] Any new error messages in logs? (check error patterns for new strings)
[ ] Application starts correctly? (health/ready endpoint returns 200)

If all pass: increase to 25%, hold 5 min -> 50% -> 100%
If any fail: ROLLBACK IMMEDIATELY (don't debug in canary state)

T+30 minutes (after full rollout):
[ ] Same metrics check
[ ] Check dependent services (did anything downstream start failing?)
[ ] Check business metrics (conversion rate, order success rate — if applicable)
[ ] Smoke test critical user journeys manually
```

---

### Rollback Procedure

```
When to rollback:
- Error rate > 1% (baseline was 0.1%)
- P99 latency > 2x baseline for > 5 minutes
- Any CRITICAL error in logs not present before deploy
- Any downstream service degradation that started with this deploy
- Your gut says something is wrong

How to rollback:
Option A (fastest): Feature flag to 0% (< 1 minute, if feature-flagged)
Option B (fast): Canary to 0% and route all traffic to previous version (< 2 minutes)
Option C (safe): Re-deploy previous Docker image tag
Option D (last resort): Manually revert DB migration (have this script ready)

NEVER:
- Debug in production while serving degraded traffic to users
- Take more than 5 minutes to decide to rollback when metrics are bad
- Deploy a fix forward without reverting (fix-forward is slower than rollback)
```

---

### Post-Deploy Monitoring

```
First hour:
  - Error rate dashboard (target: = baseline)
  - P99 latency (target: within 10% of baseline)
  - New error message patterns (zero tolerance for new CRITICAL errors)
  - Downstream service health

First 24 hours:
  - Memory growth rate (Node.js: should be stable, not growing)
  - CPU usage trends
  - Database query latency
  - Cache hit rate (if caching was changed)

First week:
  - Business metrics (if feature-flagged to % users, compare cohorts)
  - Error budget burn rate (is SLO still met?)
  - Customer support ticket volume (often lags by 1-2 days)
```

---

### Output Format

```
## Deployment Plan: [Service/Feature]

### Deployment Strategy
[Canary / Blue-green / Rolling / Feature flag — with % plan]

### Pre-Deploy Checklist
[Full checklist items for this specific deploy]

### Database Changes
[Migration description, estimated time, rollback migration]

### Deployment Steps
[Step-by-step for this specific deploy]

### Smoke Tests
[What to test manually after deploy, what queries to run]

### Rollback Triggers
| Metric | Threshold | Action |

### Rollback Procedure
[Step-by-step, time estimate]

### On-Call Contact
[Who is available and how to reach them]

### Post-Deploy Monitoring
[Dashboard links, what to watch for, for how long]
```
