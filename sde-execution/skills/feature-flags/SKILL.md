---
name: feature-flags
description: "Flag lifecycle (build -> launch -> clean up), targeting rules, gradual rollout percentages, flag debt prevention. Use when releasing risky features or planning a phased rollout."
---

## Feature Flags

Feature flags decouple deployment from release. Deploy code to 100% of servers; release features to 0% of users. Flip the flag to release. Flip it back to rollback — no deploy required.

### Context

Feature to flag: **$ARGUMENTS**

---

### Flag Lifecycle

```
1. BUILD    — Flag created as OFF. Code deployed. No users see the feature.
2. TEST     — Flag ON for internal users only. QA and team verify.
3. CANARY   — Flag ON for 1-5% of users. Monitor metrics.
4. ROLLOUT  — Gradually increase: 10% -> 25% -> 50% -> 100%.
5. LAUNCH   — Flag ON for 100% of users. Ready for cleanup.
6. CLEANUP  — Remove the flag and the conditional code. (MUST DO)
```

Step 6 is the most important and most skipped. Flag debt (old flags never cleaned up) creates a maintenance nightmare.

---

### Flag Types

```javascript
// 1. Boolean flag (simple on/off)
if (await flags.isEnabled('new-checkout-flow', user)) {
  return newCheckoutFlow(req, res);
} else {
  return legacyCheckoutFlow(req, res);
}

// 2. Percentage rollout (same user always gets same experience)
async function isEnabled(flagName, userId, percentage) {
  // Hash user ID + flag name to get a deterministic bucket (0-99)
  const bucket = Math.abs(hashCode(`${flagName}:${userId}`)) % 100;
  return bucket < percentage;
}

// 3. User targeting (specific users/groups)
const rules = {
  'beta-feature': {
    userIds: ['usr_123', 'usr_456'],        // explicit users
    emails: ['@example.com'],               // domain-based
    percentage: 10,                          // random 10%
    attributes: { plan: 'enterprise' }      // attribute-based
  }
};

// 4. Kill switch (default ON, can turn OFF for emergency)
if (await flags.isEnabled('stripe-payments', user)) {
  await stripe.charge(amount);
} else {
  return res.status(503).json({ error: 'Payments temporarily unavailable' });
}
```

---

### Implementation Patterns

```javascript
// Simple Redis-backed flag (for teams without LaunchDarkly/Split/Unleash)

class FeatureFlagService {
  constructor(redis, config) {
    this.redis = redis;
    this.config = config;  // fallback if Redis unavailable
  }

  async isEnabled(flagName, userId = null) {
    try {
      const flagConfig = await this.redis.get(`flag:${flagName}`);
      if (!flagConfig) return false;

      const { enabled, percentage, userIds } = JSON.parse(flagConfig);
      if (!enabled) return false;

      // Explicit user override
      if (userId && userIds?.includes(userId)) return true;

      // Percentage rollout (deterministic per user)
      if (percentage !== undefined && userId) {
        const bucket = this.getUserBucket(flagName, userId);
        return bucket < percentage;
      }

      return enabled;
    } catch (err) {
      // Flag service failure: default to false (safe default)
      logger.warn({ flagName, err }, 'Flag service error — defaulting to false');
      return false;
    }
  }

  getUserBucket(flagName, userId) {
    // Deterministic: same flag + user always returns same bucket
    const hash = crypto.createHash('md5')
      .update(`${flagName}:${userId}`)
      .digest('hex');
    return parseInt(hash.slice(0, 8), 16) % 100;
  }
}

// Managed solutions to consider:
// LaunchDarkly — full-featured, expensive ($75+/month)
// Unleash — self-hosted, free, solid
// Split.io — similar to LaunchDarkly
// Growthbook — open-source, experiment-focused
// Simple Redis key — fine for teams not needing targeting rules
```

---

### Flag Debt Prevention

```javascript
// Rule 1: Every flag has an owner and expiry date
// Flag registry (Notion, Google Sheet, Jira):
// | Flag Name | Owner | Created | Expected Cleanup Date | Status |
// | new-checkout | @alice | 2024-01-10 | 2024-02-10 | Rolling out 50% |

// Rule 2: Flag code should be easy to remove
// Bad: flag spread across 10 files
// Good: flag at the entry point, flags clean single branches

// Rule 3: Cleanup is a ticket in the sprint BEFORE launch
// When you create the feature ticket, create the cleanup ticket now:
// "Remove new-checkout flag and dead code" — schedule it 2 sprints out

// Rule 4: Flag names follow a convention
// Convention: [team]-[feature]-[v2 if replacement]
// Examples: checkout-new-flow, auth-mfa-v2, payment-stripe-v3

// Code that's easy to clean up:
if (await flags.isEnabled('new-dashboard')) {
  return <NewDashboard />;
}
return <OldDashboard />;
// Cleanup: delete flag, delete the if block, delete OldDashboard component

// Code that's hard to clean up (flag in multiple places):
// scattered conditionals in 5 files -> cleanup requires searching every file
// Fix: wrap in a single adapter function
async function getDashboardComponent(user) {
  if (await flags.isEnabled('new-dashboard', user.id)) return NewDashboard;
  return OldDashboard;
}
// Cleanup: remove the function, use NewDashboard directly
```

---

### Monitoring Flags

```javascript
// Track which flag variant each user received (for debugging and rollback analysis)
const flagValue = await flags.isEnabled('new-checkout', user.id);
logger.info({ action: 'flag.evaluated', flagName: 'new-checkout', flagValue, userId: user.id });

// Compare metrics between flag variants
// Query: conversion rate by flag variant
// Datadog: avg(conversion_rate) by {flag:new-checkout}
// Answer: "New checkout converts 12% vs old checkout 10% -> flag is working"

// Alerting: if a flag flip causes error rate to jump, circuit breaker on the flag:
const errorRate = await getErrorRate('new-checkout');
if (errorRate > 0.05 && flagPercentage > 0) {
  await flags.setPercentage('new-checkout', 0);  // auto-rollback
  alert.send('new-checkout flag auto-disabled: error rate ' + errorRate);
}
```

---

### Output Format

```
## Feature Flag Plan: [Feature Name]

### Flag Configuration
Name: [flag-name]
Type: [Boolean / Percentage / Targeting]
Default (when flag is absent): false
Owner: [name]
Expected cleanup: [date]

### Rollout Plan
| Phase | Percentage | Duration | Success Criteria |

### Targeting Rules (if applicable)
[Which users/groups get early access]

### Monitoring Plan
[Which metrics to compare between flag variants]

### Cleanup Plan
[When and how to remove the flag and old code]

### Rollback Procedure
[How to revert: set flag to 0%, time estimate]
```
