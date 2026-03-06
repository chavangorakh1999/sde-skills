---
name: performance-profiling
description: "Hotspot identification, complexity analysis (Big-O in practice), memory profiling approach, flame graph reading for Node.js. Use when diagnosing slow endpoints or high memory usage."
---

## Performance Profiling

You cannot optimize what you cannot measure. Profile first, optimize second. Never guess.

### Context

Performance problem to diagnose: **$ARGUMENTS**

---

### Step 1: Characterize the Problem

Before profiling, define what "slow" means:

```
What is the symptom?
- Slow HTTP response? (which endpoint, which percentile: p50/p99?)
- High CPU? (sustained or spikes?)
- High memory? (growing over time — leak? or just high baseline?)
- High I/O? (DB slow, external API slow, disk?)
- High error rate? (timeouts, OOM kills?)

What's the baseline?
- Normal: p50 = 50ms, p99 = 200ms
- Today: p50 = 50ms, p99 = 2000ms  <- only p99 is affected (tail latency issue)
  vs
- Today: p50 = 500ms, p99 = 2000ms <- all requests slow (systemic issue)

When did it start?
- Gradual degradation? (likely data growth, memory leak, index missing)
- After a deploy? (code regression)
- At a specific time of day? (traffic spike, cron job, batch job)
```

---

### Step 2: Node.js CPU Profiling

```bash
# Option 1: V8 built-in profiler (production-safe, low overhead)
node --prof app.js
# Run load test or reproduce the slow scenario
# Then process the profile
node --prof-process isolate-0x*.log > profile.txt

# Option 2: Clinic.js (best DX for Node.js profiling)
npm install -g clinic
clinic doctor -- node app.js    # general diagnostics
clinic flame  -- node app.js    # flame graph (CPU hotspots)
clinic bubbleprof -- node app.js # async bottlenecks

# Option 3: 0x (interactive flame graph)
npm install -g 0x
0x -o node app.js
# Open generated HTML flame graph in browser
```

**Reading a flame graph:**
- X-axis: time (wider = more time spent)
- Y-axis: call stack (bottom = root, top = leaf)
- Look for: wide flat bars near the top of stacks (leaf functions consuming CPU)
- Look for: deep stacks that are wide (function calling many things)
- Colors: warm (red/orange) = on-CPU, cold (blue) = off-CPU (I/O wait)

---

### Step 3: Algorithmic Complexity Analysis (Big-O in Practice)

```javascript
// O(n) — fine for most cases
const activeUsers = users.filter(u => u.isActive);

// O(n²) — catastrophic at scale (1M users = 1 trillion operations)
function findDuplicates(arr) {
  const duplicates = [];
  for (let i = 0; i < arr.length; i++) {
    for (let j = i + 1; j < arr.length; j++) {  // nested loop = O(n²)
      if (arr[i] === arr[j]) duplicates.push(arr[i]);
    }
  }
  return duplicates;
}

// O(n) fix: use a Set
function findDuplicates(arr) {
  const seen = new Set();
  const duplicates = new Set();
  for (const item of arr) {
    if (seen.has(item)) duplicates.add(item);
    else seen.add(item);
  }
  return [...duplicates];
}

// Common O(n²) traps in JavaScript:
// - Array.includes() or Array.find() inside a loop -> use Map or Set
// - String concatenation in a loop -> use array.join()
// - Sorting inside a loop -> sort once, use outside the loop
// - DOM operations in a loop (not Node.js, but React/frontend)

// Real-world scale thresholds:
// O(1):    irrelevant to scale
// O(log n): 1M items -> ~20 operations (binary search, balanced tree lookup)
// O(n):    1M items -> 1M operations (acceptable for most)
// O(n log n): 1M items -> ~20M ops (sort algorithms — fine)
// O(n²):  1M items -> 1 trillion operations (unacceptable at scale)
// O(2^n): 30 items -> 1 billion operations (exponential — always a bug)
```

---

### Step 4: Node.js Memory Profiling

```javascript
// Detect a memory leak: watch RSS over time
setInterval(() => {
  const { rss, heapUsed, heapTotal } = process.memoryUsage();
  console.log({
    rss: `${(rss / 1024 / 1024).toFixed(1)} MB`,
    heapUsed: `${(heapUsed / 1024 / 1024).toFixed(1)} MB`,
    heapTotal: `${(heapTotal / 1024 / 1024).toFixed(1)} MB`
  });
}, 5000);

// If heapUsed grows monotonically over time without ever decreasing -> memory leak
// If heapUsed fluctuates but doesn't grow long-term -> GC is working, baseline is high
```

```bash
# Heap snapshot with Chrome DevTools
# In app.js:
const v8 = require('v8');
const fs = require('fs');

// Trigger via endpoint or signal:
process.on('SIGUSR2', () => {
  const snapshot = v8.writeHeapSnapshot();
  console.log(`Heap snapshot written to ${snapshot}`);
});

# Send signal to get snapshot
kill -USR2 <PID>

# Load .heapsnapshot file in Chrome DevTools -> Memory tab
# Sort by "Retained Size" to find what's holding memory
# Look for: arrays growing without bound, EventEmitter listeners not removed,
#           closures retaining large objects, circular references
```

**Common Node.js memory leaks:**
```javascript
// 1. EventEmitter listeners not removed
const emitter = new EventEmitter();
function handler() { /* ... */ }
emitter.on('event', handler);
// Fix: always call emitter.off('event', handler) when done
// or use emitter.once() for one-time handlers

// 2. Global arrays growing
global.requestLog = [];
app.use((req) => global.requestLog.push(req));  // never cleared -> OOM

// 3. Closures over large objects
function createHandler(largeData) {
  return function handler() {
    // handler closes over largeData — keeps it alive as long as handler exists
    return largeData.something;
  };
}

// 4. timers/intervals not cleared
const interval = setInterval(fn, 1000);
// If you never clearInterval(interval) -> fn and everything it closes over stays alive

// 5. Unbounded caches
const cache = new Map();
cache.set(key, value);  // never evicted -> grows forever
// Fix: use an LRU cache with a size limit: npm install lru-cache
```

---

### Step 5: Database Query Analysis

```javascript
// PostgreSQL: EXPLAIN ANALYZE
// (Node.js — pg driver)

async function analyzeQuery(db, sql, params) {
  const result = await db.query(`EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`, params);
  console.log(JSON.stringify(result.rows[0]['QUERY PLAN'], null, 2));
}

// What to look for in EXPLAIN ANALYZE:
// "Seq Scan" on large table -> missing index
// "Nested Loop" with high row estimates -> join strategy problem
// "actual rows" >> "rows" estimate -> stale table statistics (run ANALYZE)
// "actual time" >> expected -> look at node-level timing

// Slow query log (PostgreSQL)
// postgresql.conf:
// log_min_duration_statement = 100  # log queries taking > 100ms

// Mongoose: enable query logging
mongoose.set('debug', true);

// N+1 detection: if you see the same query pattern N times in a request,
// you have an N+1 problem
// Fix: .populate() in Mongoose, JOIN in SQL, DataLoader in GraphQL
```

---

### Step 6: HTTP Endpoint Profiling

```javascript
// Add timing middleware to every request
app.use((req, res, next) => {
  const start = process.hrtime.bigint();

  res.on('finish', () => {
    const durationMs = Number(process.hrtime.bigint() - start) / 1_000_000;
    logger.info({
      method: req.method,
      path: req.route?.path ?? req.path,
      statusCode: res.statusCode,
      durationMs: Math.round(durationMs),
      requestId: req.id
    });
  });

  next();
});

// Load test with autocannon (Node.js)
// npm install -g autocannon
// autocannon -c 100 -d 30 http://localhost:3000/api/users
// -c 100: 100 concurrent connections
// -d 30: 30 seconds
// Reports: latency (p50/p97.5/p99), throughput, error rate
```

---

### Output Format

```
## Performance Analysis: [Endpoint/Service]

### Symptom
[What is slow, at which percentile, compared to baseline]

### Profiling Results
[Flame graph hotspots, heap snapshot findings, query analysis]

### Root Causes Found

1. [Root cause #1 — file.js:line]
   Complexity: O(n²) -> Fix: O(n) with Map lookup
   Impact estimate: ~200ms savings at P99

2. [Root cause #2]
   ...

### Optimizations (prioritized by impact)
| Change | Effort | Expected Improvement |

### Before/After Measurements
[Benchmark results if available]

### What NOT to Optimize
[Things that look expensive but aren't actually bottlenecks at current scale]
```
