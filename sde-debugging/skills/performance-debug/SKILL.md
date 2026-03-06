---
name: performance-debug
description: "Latency profiling workflow, p99 spike investigation, throughput bottleneck isolation, identifying queuing vs processing delays in Node.js. Use when an endpoint or service is unexpectedly slow."
---

## Performance Debugging

The methodology for debugging performance: measure precisely, isolate the bottleneck, prove the cause, then fix. Never optimize without profiling first.

### Context

Performance problem to investigate: **$ARGUMENTS**

---

### Step 1: Define "Slow" Precisely

```javascript
// Before diagnosing, characterize the symptom:

// 1. Which metric is degraded?
//    P50 high: most requests are slow (systemic issue)
//    P99 high, P50 fine: tail latency issue (rare-but-slow requests)
//    Both high: everything is slow

// 2. Which endpoints are affected?
//    All endpoints: infrastructure, shared dependency, or resource exhaustion
//    Specific endpoint: logic bug, query, or external call in that path

// 3. When did it start?
//    After deploy: code regression
//    Gradual increase: data growth hitting complexity cliff, memory leak, missing index
//    Sudden spike: traffic spike, noisy neighbor, external service slowness

// 4. Is it correlated with any variable?
//    Time of day: cron job, batch process, traffic pattern
//    User segment: specific user type, tenant, or geography
//    Request size: large payloads hitting O(n) logic

// Collect: P50, P95, P99 for the affected endpoint before and after degradation
```

---

### Step 2: Isolate the Layer

```javascript
// Breakdown of a typical Node.js/Express request timeline:
//
// Client -> Network -> Load Balancer -> Express handler -> Service layer -> DB query -> Response
//           ?ms         1-2ms           1-5ms (framework)   ?ms           ?ms
//
// Add timing to each layer:

async function getUserHandler(req, res, next) {
  const timings = {};
  const start = Date.now();

  try {
    // Layer 1: Input validation
    const t0 = Date.now();
    const validatedId = validateUserId(req.params.id);
    timings.validation = Date.now() - t0;

    // Layer 2: Cache check
    const t1 = Date.now();
    const cached = await cache.get(`user:${validatedId}`);
    timings.cacheCheck = Date.now() - t1;

    if (cached) {
      timings.total = Date.now() - start;
      req.log.info({ action: 'getUser', timings, cacheHit: true });
      return res.json(JSON.parse(cached));
    }

    // Layer 3: Database query
    const t2 = Date.now();
    const user = await userRepo.findById(validatedId);
    timings.dbQuery = Date.now() - t2;

    // Layer 4: Business logic + serialization
    const t3 = Date.now();
    const response = sanitizeUser(user);
    timings.serialization = Date.now() - t3;

    timings.total = Date.now() - start;
    req.log.info({ action: 'getUser', timings, cacheHit: false });

    res.json(response);
  } catch (err) {
    next(err);
  }
}

// After logging this for 1000 requests, you know exactly where the time is going
// Most common finding: 95% of latency is in a single layer (usually DB)
```

---

### Database Query Debugging

```javascript
// Step 1: Log all slow queries
// In Knex.js:
knex.on('query-response', (response, query, builder) => {
  const durationMs = /* calculate */ ;
  if (durationMs > 100) {
    logger.warn({ action: 'slow_query', sql: query.sql, durationMs });
  }
});

// Step 2: EXPLAIN ANALYZE the slow queries
const plan = await db.raw(`
  EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
  SELECT u.*, COUNT(o.id) as order_count
  FROM users u
  LEFT JOIN orders o ON o.user_id = u.id
  WHERE u.status = 'active'
  GROUP BY u.id
  ORDER BY u.created_at DESC
  LIMIT 20
`);
console.log(JSON.stringify(plan.rows[0]['QUERY PLAN'], null, 2));

// What to look for in EXPLAIN output:
// "Seq Scan" on large table (> 1K rows): missing index
// "actual rows" >> "rows" estimate: stale statistics (run ANALYZE table_name)
// "Nested Loop" with many rows: potentially expensive, may need different join strategy
// "Sort" without index: ORDER BY without supporting index -> filesort
// "Hash" with large batches count: insufficient work_mem

// Quick win: check if index exists for the filter/join/sort columns
const indexes = await db.raw(`
  SELECT indexname, indexdef
  FROM pg_indexes
  WHERE tablename = 'orders'
`);

// Add missing index (CONCURRENTLY to avoid table lock in production):
await db.raw('CREATE INDEX CONCURRENTLY idx_orders_user_status ON orders(user_id, status)');
```

---

### Node.js Event Loop Debugging

```javascript
// The event loop is single-threaded. If something blocks it, ALL requests are slow.

// Detect event loop lag:
const { monitorEventLoopDelay } = require('perf_hooks');
const histogram = monitorEventLoopDelay({ resolution: 10 });
histogram.enable();

// Check every 5 seconds
setInterval(() => {
  const lagMs = histogram.mean / 1e6;  // nanoseconds to milliseconds
  if (lagMs > 10) {
    logger.warn({ eventLoopLagMs: lagMs }, 'Event loop delay detected');
  }
  histogram.reset();
}, 5000);

// Common causes of event loop blocking:
// 1. JSON.parse/stringify of very large objects (> 1MB)
// 2. Synchronous crypto operations (use async versions)
// 3. Large array operations (sort, filter) on CPU-intensive data
// 4. bcrypt with work factor too high (12 is fine, 15+ starts to lag)
// 5. fs.readFileSync, execSync in request handlers
// 6. Regex on large input (catastrophic backtracking)

// Test if your regex can backtrack catastrophically:
// const regex = /(a+)+$/;
// regex.test('aaaaaaaaaaaaaaaaaaaaaaab');  // Takes minutes with 20+ 'a's
// Fix: use atomic groups, possessive quantifiers, or limit input length

// CPU-intensive work: move to worker threads
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

function runInWorker(fn, data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(`
      const { parentPort, workerData } = require('worker_threads');
      // Run CPU-intensive work here, send result back
      const result = (${fn.toString()})(workerData);
      parentPort.postMessage(result);
    `, { eval: true, workerData: data });

    worker.on('message', resolve);
    worker.on('error', reject);
  });
}
```

---

### External Service Latency

```javascript
// When a third-party API is slow, it's not "your" performance problem,
// but it IS your user's problem. Measure and protect against it.

// Track latency by external service:
async function callWithMetrics(serviceName, fn) {
  const start = Date.now();
  try {
    const result = await fn();
    const durationMs = Date.now() - start;
    metrics.histogram(`external_call_duration_ms`, durationMs, { service: serviceName, status: 'success' });
    return result;
  } catch (err) {
    const durationMs = Date.now() - start;
    metrics.histogram(`external_call_duration_ms`, durationMs, { service: serviceName, status: 'error' });
    throw err;
  }
}

// Usage:
const charge = await callWithMetrics('stripe', () => stripe.charges.create(data));

// If Stripe P99 is 3s and your timeout is 5s:
// -> P99 latency of your endpoint is at least 3s (Stripe adds 3s in the worst case)
// -> Consider circuit breaker to fail fast after threshold
// -> Consider async flow: accept order, charge asynchronously
```

---

### Output Format

```
## Performance Debug Report: [Endpoint/Service]

### Symptom
Metric: [P50/P95/P99 before and after]
When started: [timestamp or "gradual"]
Scope: [All endpoints / specific endpoint / specific user segment]

### Layer Breakdown
| Layer | Avg Duration | P99 Duration | % of Total |

### Root Cause
[Specific finding: which line of code, which query, which external call]

### Evidence
[Timing logs, EXPLAIN ANALYZE output, profiling results]

### Fix
[Code change, index addition, architecture change]

### Expected Improvement
[P99 latency after fix: estimated X ms vs current Y ms]

### Prevention
[What to add to prevent this class of problem in future: indexes, query tests, timeouts]
```
