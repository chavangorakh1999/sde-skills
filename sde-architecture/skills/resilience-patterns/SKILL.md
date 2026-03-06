---
name: resilience-patterns
description: "Circuit breaker, bulkhead, timeout, retry with exponential backoff + jitter, fallback, health check patterns. Use when designing a service that calls external dependencies or when debugging cascading failures."
---

## Resilience Patterns

Distributed systems fail. The goal is not to prevent failure — it's to prevent cascading failure. Each pattern here addresses a specific failure mode.

### Context

Service or failure mode to address: **$ARGUMENTS**

---

### Timeout

**Problem:** Calls to external services hang indefinitely, exhausting thread pool or event loop.

```javascript
// Without timeout: one slow Stripe call blocks the event loop for minutes
await stripe.charges.create(chargeData);

// With timeout (fetch / axios):
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch(url, { signal: controller.signal });
  clearTimeout(timeoutId);
  return response;
} catch (err) {
  if (err.name === 'AbortError') throw new TimeoutError('Payment service timed out');
  throw err;
}

// With axios:
const response = await axios.post(url, data, { timeout: 5000 });

// With node-fetch / undici:
import { fetch } from 'undici';
const response = await fetch(url, { signal: AbortSignal.timeout(5000) });

// Timeout values by service type:
// Internal microservice: 100-500ms (same datacenter, should be fast)
// Third-party API (Stripe, Twilio): 3-10 seconds (network + processing)
// File uploads/downloads: proportional to expected size (don't use fixed timeout)
// Database queries: 2-30 seconds depending on query complexity
```

---

### Retry with Exponential Backoff + Jitter

**Problem:** Transient failures (network blip, 503 from overloaded service) that would succeed on retry.

```javascript
async function retryWithBackoff(fn, options = {}) {
  const {
    maxAttempts = 3,
    initialDelayMs = 200,
    maxDelayMs = 10_000,
    factor = 2,
    retryableErrors = [408, 429, 500, 502, 503, 504]
  } = options;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      const isRetryable = retryableErrors.includes(err.statusCode);
      const isLastAttempt = attempt === maxAttempts;

      if (!isRetryable || isLastAttempt) throw err;

      // Exponential backoff: 200ms, 400ms, 800ms...
      const baseDelay = Math.min(initialDelayMs * Math.pow(factor, attempt - 1), maxDelayMs);
      // Full jitter: random between 0 and baseDelay
      // Prevents thundering herd when many clients retry simultaneously
      const jitterDelay = Math.random() * baseDelay;

      await sleep(jitterDelay);
    }
  }
}

const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));

// Usage
const charge = await retryWithBackoff(
  () => stripe.charges.create(chargeData),
  { maxAttempts: 3, initialDelayMs: 500 }
);

// WARNING: Only retry idempotent operations or operations with idempotency keys
// Never blindly retry POST /payments without an idempotency key
// Retrying a non-idempotent write can double-charge a customer
```

---

### Circuit Breaker

**Problem:** Continuous retries to a failing service amplify load, slowing recovery and wasting resources.

```javascript
class CircuitBreaker {
  constructor(fn, options = {}) {
    this.fn = fn;
    this.state = 'CLOSED';  // CLOSED = normal, OPEN = rejecting, HALF_OPEN = testing
    this.failureCount = 0;
    this.lastFailureTime = null;

    this.failureThreshold = options.failureThreshold ?? 5;   // open after 5 failures
    this.recoveryTimeout = options.recoveryTimeout ?? 30_000; // try again after 30s
    this.successThreshold = options.successThreshold ?? 2;   // close after 2 successes
    this.halfOpenSuccesses = 0;
  }

  async execute(...args) {
    if (this.state === 'OPEN') {
      const timeSinceFailure = Date.now() - this.lastFailureTime;
      if (timeSinceFailure < this.recoveryTimeout) {
        throw new CircuitOpenError('Circuit breaker is OPEN — service unavailable');
      }
      // Try one probe request
      this.state = 'HALF_OPEN';
      this.halfOpenSuccesses = 0;
    }

    try {
      const result = await this.fn(...args);
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  onSuccess() {
    if (this.state === 'HALF_OPEN') {
      this.halfOpenSuccesses++;
      if (this.halfOpenSuccesses >= this.successThreshold) {
        this.state = 'CLOSED';
        this.failureCount = 0;
      }
    } else {
      this.failureCount = 0;
    }
  }

  onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
    } else if (this.state === 'HALF_OPEN') {
      this.state = 'OPEN';  // probe failed — back to OPEN
    }
  }
}

// Usage
const stripeCircuitBreaker = new CircuitBreaker(
  (data) => stripe.charges.create(data),
  { failureThreshold: 5, recoveryTimeout: 30_000 }
);

// When circuit is OPEN: throw error immediately without calling Stripe
// Downstream can use fallback (queue for later retry, show user error)

// Production: use a battle-tested library like 'opossum' (Node.js)
import CircuitBreaker from 'opossum';
const breaker = new CircuitBreaker(stripeChargeFunction, {
  timeout: 5000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000
});
breaker.fallback(() => ({ queued: true, message: 'Payment queued for retry' }));
```

---

### Bulkhead

**Problem:** One slow/failing downstream service exhausts all connection capacity, starving other services.

```javascript
// Bulkhead: separate resource pools for different downstream services
// If Stripe is slow, it can only exhaust its own pool, not the one for the DB

import { Semaphore } from 'async-mutex';

class BulkheadPool {
  constructor(maxConcurrent) {
    this.semaphore = new Semaphore(maxConcurrent);
  }

  async execute(fn) {
    const [, release] = await this.semaphore.acquire();
    try {
      return await fn();
    } finally {
      release();
    }
  }
}

const stripePool    = new BulkheadPool(10);   // max 10 concurrent Stripe calls
const sendgridPool  = new BulkheadPool(5);    // max 5 concurrent email sends
const externalApiPool = new BulkheadPool(20); // max 20 concurrent external API calls

// Now even if Stripe has 10 concurrent slow requests, DB and email pools are unaffected

// HTTP connection pool bulkhead (Node.js HTTP agent):
import { Agent } from 'http';
const stripeAgent = new Agent({ maxSockets: 10 });  // only 10 sockets to Stripe
const dbAgent = new Agent({ maxSockets: 20 });      // separate pool for DB calls
```

---

### Fallback

**Problem:** When a dependency fails, the system should degrade gracefully rather than propagate the failure.

```javascript
// Fallback patterns:
// 1. Return cached/stale data
async function getProductPrice(productId) {
  try {
    return await pricingService.getPrice(productId);
  } catch (err) {
    // Pricing service down — return cached price (acceptable staleness)
    const cachedPrice = await cache.get(`price:${productId}`);
    if (cachedPrice) return { price: JSON.parse(cachedPrice), stale: true };
    // No cache — return null and let caller decide
    return null;
  }
}

// 2. Queue for later (async fallback)
async function sendNotification(userId, message) {
  try {
    await notificationService.send(userId, message);
  } catch (err) {
    // Notification service down — queue for delivery when it recovers
    await dlq.add('failed-notification', { userId, message, failedAt: new Date() });
    logger.warn('Notification queued due to service failure', { userId });
    // Don't throw — queueing is acceptable degradation
  }
}

// 3. Default response
async function getPersonalizedRecommendations(userId) {
  try {
    return await mlService.getRecommendations(userId);
  } catch (err) {
    // ML service down — return popular items (non-personalized fallback)
    return await productService.getPopularItems({ limit: 10 });
  }
}

// Rule: fallback responses must be clearly marked as degraded
// (stale: true, source: 'fallback') so monitoring can detect service issues
```

---

### Health Checks

```javascript
// Two types:
// Liveness: "Is the process alive?" — if no, restart it
// Readiness: "Can it serve traffic?" — if no, remove from load balancer

// Express health check endpoints
app.get('/health/live', (req, res) => {
  // Simple liveness — if we can respond, we're alive
  res.json({ status: 'ok', uptime: process.uptime() });
});

app.get('/health/ready', async (req, res) => {
  const checks = await Promise.allSettled([
    db.query('SELECT 1'),              // DB is reachable
    redis.ping(),                      // Cache is reachable
  ]);

  const results = {
    database: checks[0].status === 'fulfilled' ? 'ok' : 'error',
    cache:    checks[1].status === 'fulfilled' ? 'ok' : 'error'
  };

  const isReady = Object.values(results).every(s => s === 'ok');
  res.status(isReady ? 200 : 503).json({ status: isReady ? 'ready' : 'not ready', checks: results });
});

// Kubernetes uses /health/live for livenessProbe
// Kubernetes uses /health/ready for readinessProbe
// During startup, DB migrations, or graceful shutdown: readiness returns 503
// Kubernetes stops sending traffic but doesn't restart the pod
```

---

### Graceful Shutdown

```javascript
// When pod/container receives SIGTERM, drain in-flight requests before exiting
let isShuttingDown = false;

process.on('SIGTERM', async () => {
  isShuttingDown = true;
  console.log('SIGTERM received — starting graceful shutdown');

  // 1. Stop accepting new requests
  server.close();

  // 2. Wait for in-flight requests to complete (max 30s)
  await waitForInflightRequests(30_000);

  // 3. Close downstream connections
  await db.end();
  await redis.quit();

  process.exit(0);
});

// Readiness check returns 503 during shutdown
app.get('/health/ready', (req, res) => {
  if (isShuttingDown) return res.status(503).json({ status: 'shutting down' });
  // ... normal checks
});
```

---

### Output Format

```
## Resilience Design: [Service Name]

### Failure Mode Analysis
| Dependency | Failure Mode | Probability | Impact |

### Patterns Applied

#### [Pattern Name] for [Dependency]
Configuration: [timeout/threshold/pool size values]
Code: [snippet]
Fallback: [what happens when pattern triggers]

### Degraded Mode Behavior
[What the system does when each dependency is down]

### Health Check Endpoints
[/health/live and /health/ready implementations]

### Monitoring
[What to alert on: circuit open, fallback triggered, error rate spike]
```
