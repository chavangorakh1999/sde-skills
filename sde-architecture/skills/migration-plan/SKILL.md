---
name: migration-plan
description: "Safe migrations for DB/service/language/framework: strangler fig pattern, phased cutover, feature flags, rollback triggers and procedures. Use when planning a risky migration."
---

## Migration Planning

Migrations fail when they're treated as a single big-bang change. Safe migrations are: phased, reversible at each phase, and verified with metrics before proceeding.

### Context

Migration to plan: **$ARGUMENTS**

---

### Migration Principles

1. **Strangler Fig** — build the new alongside the old, route traffic gradually
2. **Dark Launch** — run both old and new code in parallel, compare outputs
3. **Feature Flags** — control who gets new vs old behavior
4. **Rollback at each phase** — define rollback triggers before starting
5. **Measure before cutting over** — never switch 100% without error rate confirmation

---

### Phase Structure for Any Migration

```
Phase 0: Preparation (not visible to users)
  - Build and test the new implementation
  - Set up feature flag / routing capability
  - Establish baseline metrics (error rate, latency, conversion)
  - Write rollback procedure
  - Define success criteria and rollback triggers

Phase 1: Shadow Mode / Dark Launch (0% traffic, internal only)
  - Run new code on production traffic, discard results
  - Compare new vs old output in logs
  - Fix divergences

Phase 2: Canary (1-5% of traffic)
  - Route small % to new implementation
  - Monitor metrics vs baseline
  - Hold for 24-48h to catch edge cases

Phase 3: Gradual Rollout (10% -> 25% -> 50% -> 100%)
  - Increase at each stage, hold to verify
  - Each stage: confirm error rate < baseline + 0.1%

Phase 4: Cleanup
  - Remove old implementation
  - Remove feature flag
  - Update documentation

Rollback at any phase: revert feature flag to 0% — instant
```

---

### Database Migration (Zero-Downtime)

The hardest migrations. Databases are slow to change and every step must work with both old and new application code simultaneously.

```javascript
// Example: Adding a NOT NULL column to a users table

// Step 1 (safe to deploy anytime): Add column as nullable
// ALTER TABLE users ADD COLUMN display_name VARCHAR(100);
// Application: reads display_name if present, falls back to old column

// Step 2: Backfill existing rows in batches (don't lock the table)
// Batch update to avoid long table lock:
const BATCH_SIZE = 1000;
let lastId = 0;

while (true) {
  const result = await db.query(
    `UPDATE users
     SET display_name = COALESCE(full_name, email)
     WHERE id > $1 AND display_name IS NULL
     LIMIT $2
     RETURNING id`,
    [lastId, BATCH_SIZE]
  );

  if (result.rows.length === 0) break;
  lastId = result.rows[result.rows.length - 1].id;
  await sleep(100);  // rate limit to avoid overwhelming DB
}

// Step 3 (after backfill complete): Add NOT NULL constraint
// PostgreSQL: use a constraint check, not DDL change, to validate without long lock
// ALTER TABLE users ADD CONSTRAINT users_display_name_not_null CHECK (display_name IS NOT NULL) NOT VALID;
// ALTER TABLE users VALIDATE CONSTRAINT users_display_name_not_null;  -- validates in background

// Step 4: Update application to always write display_name
// Step 5: Once all deploys have new code: ALTER TABLE users ALTER COLUMN display_name SET NOT NULL;
// Step 6: Drop the old column (after confirming it's no longer read)
```

**Schema migration rules:**
- Never drop a column that's still read by deployed code
- Never add a NOT NULL column without a default in a single migration
- Always batch large UPDATE/DELETE operations
- Use CONCURRENTLY for index creation: `CREATE INDEX CONCURRENTLY ...`

---

### Service Migration (Monolith to Microservices)

Using the Strangler Fig pattern:

```javascript
// Step 1: Identify the seam — a bounded context with clear inputs/outputs
// Example: extracting "notifications" from a monolith

// Step 2: Build the new service alongside the monolith
// NotificationService with its own database, deployed independently

// Step 3: Dual-write — monolith writes to both old path and new service
async function sendNotification(userId, type, data) {
  // Write to old path (monolith's notification queue)
  await this.notificationQueue.add({ userId, type, data });

  // Also write to new service (async, non-blocking)
  notificationService.send(userId, type, data).catch(err => {
    logger.warn('New notification service failed', { err, userId });
    // Don't throw — old path is still the source of truth
  });
}

// Step 4: Read from new service (shadow read — compare with old)
async function getNotifications(userId) {
  const [oldResult, newResult] = await Promise.allSettled([
    oldNotificationStore.get(userId),
    notificationService.get(userId)
  ]);

  if (oldResult.status === 'fulfilled' && newResult.status === 'fulfilled') {
    const differ = !deepEqual(oldResult.value, newResult.value);
    if (differ) logger.warn('Notification divergence', { userId });  // investigate
  }

  return oldResult.value;  // still using old — just monitoring new
}

// Step 5: Flip reads to new service (keep old as backup)
// Step 6: Stop dual-write to old path
// Step 7: Decommission old notification code
```

---

### Feature Flag Strategy

```javascript
// Use feature flags to control every phase of migration
// Flip without deploy — instant rollback

// Simple flag with LaunchDarkly / custom Redis flag
async function getUser(id) {
  const useNewUserService = await flags.getBooleanValue('new-user-service', {
    userId: currentUser.id,
    percentage: 10  // 10% canary
  });

  if (useNewUserService) {
    return newUserService.findById(id);
  }
  return legacyUserService.findById(id);
}

// Rollback: set 'new-user-service' flag to false in LaunchDarkly
// Instant for all users, no deploy required

// Simple homemade flag (Redis-backed, percentage rollout):
class FeatureFlag {
  constructor(redis) { this.redis = redis; }

  async isEnabled(flagName, userId) {
    const percentage = await this.redis.get(`flag:${flagName}:pct`) ?? 0;
    // Deterministic: same user always gets same experience
    const bucket = Math.abs(hashCode(userId + flagName)) % 100;
    return bucket < percentage;
  }
}
```

---

### Rollback Triggers

Define BEFORE starting the migration. If any of these occur, rollback immediately:

```
Rollback triggers for [migration name]:
- Error rate > 0.5% (baseline was 0.1%) on affected endpoints
- P99 latency > 2x baseline for > 5 minutes
- Data consistency check fails (old vs new output diverges > X%)
- Any data loss or corruption detected
- Downstream service degradation (cascade)

Rollback procedure:
1. Set feature flag 'migration-name' to 0% (immediate)
2. Verify error rate returns to baseline within 2 minutes
3. If data was mutated: run compensation script [link to script]
4. Alert on-call team via PagerDuty channel [link]
5. Post-mortem within 24h
```

---

### Output Format

```
## Migration Plan: [From] -> [To]

### Executive Summary
[What's changing, why, timeline, risk level]

### Phase Plan

| Phase | What Changes | Traffic % | Success Criteria | Rollback |
|-------|-------------|-----------|------------------|----------|

### Technical Steps
[Detailed step-by-step for each phase]

### Rollback Triggers
[Metrics thresholds that trigger immediate rollback]

### Rollback Procedure
[Step-by-step: how to revert each phase in < 5 minutes]

### Data Migration
[If data must be migrated: batch strategy, validation, cleanup]

### Feature Flags
[Flag names, values per phase]

### Monitoring Plan
[Which dashboards to watch, which metrics to compare vs baseline]
```
