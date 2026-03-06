---
description: Phased migration plan with rollback strategy, success criteria, and monitoring plan
argument-hint: "<from technology/pattern -> to technology/pattern>"
---

# /plan-migration -- Migration Planning

Create a safe, phased migration plan with clear rollback triggers at each stage. Chains migration-plan -> resilience-patterns.

## Invocation

```
/plan-migration Monolithic Rails app -> separate Node.js microservices over 6 months
/plan-migration MongoDB 5.0 -> 7.0 in production with zero downtime
/plan-migration REST API -> GraphQL for the mobile app consumer
/plan-migration Self-hosted Redis -> Redis Cluster with 5-node setup
/plan-migration                    # asks for migration description
```

## Workflow

### Step 1: Understand the Migration

Extract:
- What are we migrating FROM? (current state)
- What are we migrating TO? (target state)
- Why? (technical reason, business driver, compliance)
- What's the timeline? (hard deadline or flexible?)
- What's the risk tolerance? (can we have minutes of downtime? zero downtime required?)
- What's the blast radius? (one service? whole platform? customer-facing?)

### Step 2: Pre-Migration Assessment

Before designing the plan:
- [ ] List all services/clients that depend on what's changing
- [ ] Identify the hardest part (what's most likely to fail?)
- [ ] Establish baseline metrics (what's normal today?)
- [ ] Map out rollback options (can we revert each phase independently?)

### Step 3: Design the Phases

Apply **migration-plan** skill:

Phase 0: Shadow mode (if applicable — build new, run parallel, compare)
Phase 1: Canary (1-5% of traffic or data)
Phase 2: Gradual rollout (10% -> 25% -> 50%)
Phase 3: Full cutover (100%)
Phase 4: Cleanup (remove old infrastructure)

For each phase:
- What changes?
- What's the success criteria?
- How long to hold before proceeding?
- What's the rollback trigger?
- How to rollback (steps + time estimate)?

### Step 4: Resilience Design

Apply **resilience-patterns** for the transition period:
- Dual-write pattern (write to both old and new during transition)
- Feature flags for routing traffic
- Circuit breaker if new system is unreliable
- Fallback to old system on failure

### Step 5: Data Migration (if applicable)

If data must be migrated:
- Backfill strategy (batch size, rate limiting, validation)
- Verification: compare old vs new data counts, checksums
- Rollback: can we un-migrate data? (often the hardest part)
- Cleanup: when and how to delete old data

### Step 6: Define Rollback Triggers

Quantitative triggers that initiate immediate rollback:
- Error rate > baseline + X%
- Latency P99 > X ms
- Data consistency check fails
- Any data loss detected

### Step 7: Monitoring Plan

Which metrics to watch at each phase, compared to baseline.

## Output

```
## Migration Plan: [FROM] -> [TO]

### Executive Summary
[What's migrating, why, timeline, risk level]

### Pre-Migration Checklist
- [ ] Baseline metrics captured
- [ ] Rollback tested in staging
- [ ] ...

### Phases

#### Phase 0: Preparation (Week 1-2)
What: [Description]
Success criteria: [How we know it's done]
Rollback: N/A

#### Phase 1: Shadow Mode / Canary (Week 3)
What: [Description — traffic %]
Hold period: [Duration to monitor]
Success criteria: [Metrics targets]
Rollback trigger: [Specific thresholds]
Rollback procedure: [Steps to revert, estimated time]

#### Phase 2: Gradual Rollout (Week 4-5)
...

#### Phase 3: Full Cutover (Week 6)
...

#### Phase 4: Cleanup (Week 7-8)
...

### Rollback Triggers (Any Phase)
| Metric | Threshold | Action |

### Data Migration
[Backfill plan, validation, rollback]

### Monitoring Plan
[Dashboards, metrics, comparison period]

### Risk Register
| Risk | Probability | Impact | Mitigation |

### Go/No-Go Decision Points
[Who approves proceeding at each phase]
```

## Next Steps

- "Should I write an ADR for this migration? -> `/write-adr [decision]`"
- "Want to design the resilience patterns? -> `/design-resilience [service]`"
