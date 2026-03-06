---
description: Resilience pattern recommendations matched to identified failure modes with monitoring plan
argument-hint: "<service name and its dependencies>"
---

# /design-resilience -- Resilience Design

Map failure modes to resilience patterns. Chains resilience-patterns -> observability-design.

## Invocation

```
/design-resilience Payment service calling Stripe, Twilio, and SendGrid
/design-resilience Order processing service that calls inventory, auth, and notifications
/design-resilience API gateway with 12 downstream microservices
/design-resilience                    # asks for service description
```

## Workflow

### Step 1: Map the Dependencies

Extract or ask:
- What external services/APIs does this service call?
- What internal services does it call?
- What databases does it use?
- What message queues does it produce/consume?
- What's the expected SLO for this service?

### Step 2: Enumerate Failure Modes

For each dependency:
- What happens if it's unavailable? (completely down)
- What happens if it's slow? (50th-percentile 10s, p99 30s)
- What happens if it returns errors intermittently?
- What happens if our service crashes mid-operation?

Apply **resilience-patterns** skill to categorize:
- Timeout needed? (all external calls)
- Retry safe? (idempotent operations only)
- Circuit breaker needed? (repeatedly failing, want to fail fast)
- Bulkhead needed? (don't let one dependency exhaust all resources)
- Fallback possible? (stale data, degraded response, queue for retry)

### Step 3: Design Pattern Configuration

For each dependency with a failure risk:

```
Dependency: [Name]
Failure mode: [Unavailable / Slow / Intermittent errors]
Pattern(s): [Timeout + Retry + Circuit Breaker | Fallback | Bulkhead]
Timeout: [Xms — explain how you determined this value]
Retry: [X attempts, Y initial delay, exponential backoff, jitter]
Circuit breaker: [open after X failures in Ys, half-open after Zs]
Fallback: [What to return/do if all else fails]
Bulkhead: [Max X concurrent calls to this dependency]
```

### Step 4: Degraded Mode Behavior

Define what the service does when each dependency is down:
- What features still work? (must work even in degraded mode)
- What features fail gracefully? (return sensible error, not crash)
- What features can't work at all? (accept this, communicate it clearly)

This forces you to think about which dependencies are truly critical vs. optional.

### Step 5: Health Check Design

Apply **observability-design**:
- Liveness endpoint: is the process alive?
- Readiness endpoint: can it serve traffic? (checks all critical dependencies)
- Which dependencies go into readiness vs. just metrics?

### Step 6: Observability

What to monitor to detect failures before users do:
- Circuit breaker state (CLOSED/OPEN/HALF_OPEN) — alert when OPEN
- Fallback activation rate — alert when > X%
- Timeout rate per dependency — alert when trending up
- Retry storm detection — alert when retry rate > normal QPS

### Step 7: Graceful Shutdown

Ensure the service drains cleanly on SIGTERM:
- Stop accepting new requests
- Complete in-flight requests (max 30s grace period)
- Close downstream connections cleanly

## Output

```
## Resilience Design: [Service Name]

### Dependency Map
[List of dependencies with their criticality: critical / degradable / optional]

### Failure Mode Analysis

| Dependency | Failure | User Impact | Patterns Applied |

### Pattern Configuration

#### [Dependency 1]
Timeout: [Xms with rationale]
Retry: [attempts, delays, which errors to retry]
Circuit Breaker: [thresholds and configuration]
Fallback: [what happens when all patterns are exhausted]

#### [Dependency 2]
...

### Degraded Mode Definition
| Dependency Down | Features Still Work | Features Fail Gracefully | Features Down |

### Health Endpoints
[/health/live and /health/ready implementation]

### Graceful Shutdown
[SIGTERM handling, drain period, cleanup steps]

### Monitoring Alerts
| Alert | Condition | Severity | Runbook |

### Code Skeleton
[Key resilience code patterns for this specific service]
```

## Next Steps

- "Want to write an ADR for these resilience decisions? -> `/write-adr`"
- "Should I write runbooks for these failure scenarios? -> `/write-runbook [service]`"
- "Want to set up the observability stack? -> ask about observability-design"
