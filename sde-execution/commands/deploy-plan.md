---
description: Full deployment checklist with rollback triggers and monitoring hooks
argument-hint: "<service and key changes>"
---

# /deploy-plan -- Deployment Plan

Create a complete deployment checklist. Chains deployment-plan -> on-call-runbook.

## Invocation

```
/deploy-plan Payment service — first production release, zero-downtime required
/deploy-plan Auth service — adding JWT refresh token logic, DB migration included
/deploy-plan Feature release: new checkout flow behind feature flag, 5% canary
/deploy-plan                    # asks for service and change description
```

## Workflow

### Step 1: Understand the Release

Extract:
- What service and version?
- What changed? (code changes, DB migrations, config changes, new dependencies)
- Zero-downtime required?
- Any external dependencies or coordinated changes?
- Who is the on-call engineer for this deploy?
- What's the rollback strategy?

### Step 2: Risk Assessment

Classify the risk:
- **Low:** no DB changes, pure logic change, already battle-tested pattern
- **Medium:** DB migration (backward-compatible), new external dependency, performance-sensitive area
- **High:** breaking DB change, first time this service handles production traffic, complex migration

Risk level drives: canary percentage (higher risk = lower starting canary), hold time at each stage.

### Step 3: Pre-Deploy Checklist

Apply **deployment-plan** skill.

Generate a specific checklist for this deploy:
- Code review approvals
- Staging verification steps
- DB migration backward-compatibility check
- Rollback migration availability
- On-call availability confirmation
- Feature flag state verification

### Step 4: Deployment Steps

Define the exact sequence:
1. When to run migrations (before or after code deploy?)
2. Canary percentage plan (1% -> 5% -> 25% -> 100%)
3. Hold duration at each percentage
4. Go/no-go criteria at each gate

### Step 5: Smoke Tests

Write specific smoke tests for this deploy:
- Which endpoints to hit manually
- What responses to verify
- Which metrics to check on dashboard

### Step 6: Rollback Triggers and Procedure

Define exactly:
- Which metrics trigger rollback (with thresholds)
- The step-by-step rollback procedure (with commands)
- Time estimate for rollback (must be < 5 minutes)

### Step 7: Post-Deploy Monitoring

What to watch in the 24 hours after deployment.

### Step 8: Runbook Update

Apply **on-call-runbook** skill — does the existing runbook need updating for this release?
If this introduces new alerts or new failure modes, document them now.

## Output

```
## Deployment Plan: [Service] v[Version]

### Risk Level: [Low / Medium / High]
[Reason for classification]

### Pre-Deploy Checklist
[ ] ...

### Database Changes
[Migration description, rollback migration, timing]

### Deployment Sequence
[ ] T-15: [Checks]
[ ] T-0:  Run migrations
[ ] T+0:  Deploy canary (X%)
[ ] T+5m: Verify metrics -> go/no-go
[ ] T+5m: Increase to Y%
...

### Smoke Tests
[ ] GET /health/ready -> 200
[ ] [Specific business logic test]
[ ] Check [metric] on [dashboard link]

### Rollback Triggers
| Metric | Threshold | Action |

### Rollback Procedure
1. [Step 1 — time estimate]
2. ...
Total rollback time: < X minutes

### Post-Deploy Monitoring (24h)
[What to watch]

### On-Call: [Name]
[Contact info / escalation path]
```

## Next Steps

- "Need a runbook for the new alerts? -> `/write-runbook [service]`"
- "Should I write release notes? -> `/release-notes [commits]`"
