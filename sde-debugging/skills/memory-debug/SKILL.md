---
name: memory-debug
description: "Memory leak detection in Node.js: heap snapshot analysis, GC pressure diagnosis, reference retention patterns, EventEmitter leaks. Use when a Node.js service grows in memory over time."
---

## Memory Debugging (Node.js)

A Node.js memory leak means objects that should be garbage collected are being kept alive by a hidden reference. The process grows indefinitely and eventually OOM-kills.

### Context

Memory issue to diagnose: **$ARGUMENTS**

---

### Step 1: Confirm It's a Leak

Not all memory growth is a leak. Distinguish:

```javascript
// Monitor memory over time — add to your app or run from CLI
setInterval(() => {
  const { heapUsed, heapTotal, rss, external } = process.memoryUsage();
  const mb = (n) => `${(n / 1024 / 1024).toFixed(1)} MB`;
  console.log({
    heapUsed: mb(heapUsed),
    heapTotal: mb(heapTotal),
    rss: mb(rss),          // resident set size — total memory used by process
    external: mb(external)  // memory used by C++ objects bound to JS objects
  });
}, 10_000);

// Leak pattern: heapUsed grows monotonically over hours/days, never decreases
// Normal: heapUsed fluctuates (GC cleans up), but stays roughly bounded
// GC pressure: heapUsed frequently near heapTotal (GC working hard but keeping up)

// Baseline: in development, idle process with no traffic
// Load test: run traffic, watch memory grow
// If memory grows proportional to traffic and then stabilizes: not a leak (working set)
// If memory grows and NEVER stabilizes: leak
```

---

### Step 2: Take Heap Snapshots

```javascript
// Method 1: V8 heap snapshot via code
const v8 = require('v8');

// Add endpoint to trigger snapshot on demand (protect with auth!)
app.post('/admin/heap-snapshot', authenticate, requireAdmin, (req, res) => {
  const filename = v8.writeHeapSnapshot();
  res.json({ filename });
});

// Method 2: Send signal to process
// In app.js:
process.on('SIGUSR2', () => {
  const filename = v8.writeHeapSnapshot();
  console.log(`Heap snapshot: ${filename}`);
});

// Trigger: kill -USR2 <PID>

// Method 3: Node.js --inspect (Chrome DevTools)
// node --inspect app.js
// Open Chrome: chrome://inspect
// Memory tab -> Take Heap Snapshot

// Method 4: Clinic.js (easiest)
// npm install -g clinic
// clinic heapprofiler -- node app.js
// Run load test while it runs
// Clinic generates visual report
```

---

### Step 3: Analyze Heap Snapshots

```
In Chrome DevTools (Memory tab):

1. Take snapshot BEFORE load
2. Run load test
3. Take snapshot AFTER load
4. Take snapshot after 10 minutes idle (GC should clean up)
5. If snapshot 4 >> snapshot 1: leak confirmed

To find the leak:
- In snapshot 3 or 4, view "Summary" mode
- Sort by "Retained Size" (largest first)
- Look for: arrays, objects, strings growing unexpectedly

Common findings:
  Array: 500MB retained -> find which array holds what
  (closure): large retained size -> closure is holding objects alive
  EventEmitter: many instances -> listeners not removed

- Click on a suspicious object -> "Retainers" tab shows what's keeping it alive
  Example: MyClass -> listenerArray -> EventEmitter -> (root)
  -> EventEmitter isn't being cleaned up, holding MyClass instances
```

---

### Common Memory Leak Patterns

```javascript
// 1. EVENTLISTENER LEAK — most common
class DatabaseConnectionPool {
  constructor(eventBus) {
    this.eventBus = eventBus;
    // LEAK: adds listener but never removes it
    this.eventBus.on('config-change', this.handleConfigChange.bind(this));
  }

  handleConfigChange(config) {
    // handles config changes
  }
  // If instances are created multiple times, listeners accumulate
}

// Fix: always provide a remove method
class DatabaseConnectionPool {
  constructor(eventBus) {
    this.eventBus = eventBus;
    this._boundHandler = this.handleConfigChange.bind(this);
    this.eventBus.on('config-change', this._boundHandler);
  }

  destroy() {
    this.eventBus.off('config-change', this._boundHandler);  // remove listener
  }
}

// Detect: Node.js warns when > defaultMaxListeners (10) listeners on one event
const emitter = new EventEmitter();
emitter.setMaxListeners(20);  // increase if you genuinely need many listeners
// Or: emitter.on('newListener', ...) to track additions

// 2. UNBOUNDED CACHE / ACCUMULATOR
class RequestLogger {
  constructor() {
    this.requests = [];  // LEAK: never cleared
  }

  log(req) {
    this.requests.push(req);  // grows forever
  }
}

// Fix: use LRU cache with max size, or circular buffer
import LRU from 'lru-cache';
const cache = new LRU({ max: 1000 });  // never exceeds 1000 entries

// 3. TIMER NOT CLEARED
class HealthChecker {
  start() {
    // LEAK: if HealthChecker is replaced without calling stop(), interval keeps running
    this.interval = setInterval(() => this.check(), 5000);
  }

  stop() {
    clearInterval(this.interval);  // must call this when done
  }
}

// 4. CLOSURE OVER LARGE OBJECT
function processLargeFile(buffer) {
  // LEAK: handler closes over the 100MB buffer
  const handler = (event) => {
    console.log(buffer.length);  // buffer kept alive as long as handler exists
  };
  eventEmitter.on('done', handler);
  // ... process ...
  // If handler is never removed, 100MB buffer stays in memory
}

// Fix: extract only what you need
function processLargeFile(buffer) {
  const bufferLength = buffer.length;  // extract primitive — GC can free the buffer
  const handler = (event) => {
    console.log(bufferLength);
  };
  eventEmitter.once('done', handler);  // once() removes after first call
}

// 5. CIRCULAR REFERENCES (rare in modern Node.js — V8 handles these)
// WeakMap/WeakRef: use when you want cache that doesn't prevent GC
const objectMetadata = new WeakMap();
// Key will be GC'd when no other references exist, automatically removes from WeakMap
```

---

### GC Pressure Diagnosis

```javascript
// High GC pressure: frequent GCs consuming CPU, but memory stays bounded
// vs.
// Memory leak: memory grows despite GC

// Monitor GC via performance hooks
const { PerformanceObserver, constants } = require('perf_hooks');

const gcObserver = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {  // GC taking > 50ms is significant
      logger.warn({
        gcType: entry.detail.kind,
        durationMs: entry.duration
      }, 'Significant GC pause');
    }
  }
});
gcObserver.observe({ entryTypes: ['gc'] });

// If GC pauses are long and frequent:
// - Too many short-lived allocations (allocation churn) -> object pooling
// - Large old-generation objects -> avoid keeping large objects long-lived
// - Increase --max-old-space-size if heap is legitimately large

// Node.js memory flags:
// --max-old-space-size=4096   increase heap limit to 4GB (default ~1.5GB for 64-bit)
// --expose-gc                 expose global.gc() for manual GC trigger in tests
```

---

### Output Format

```
## Memory Debug Report: [Service]

### Symptoms
RSS growth rate: [MB/hour]
Heap growth rate: [MB/hour]
OOM events: [frequency if any]

### Heap Snapshot Analysis
[What the snapshot showed: top retained objects, retainer chain]

### Leak Found
Type: [EventEmitter leak / Unbounded cache / Timer / Closure]
Location: [file.js:line]
Root cause: [Why the reference is being kept]

### Fix
[Code change with before/after]

### Verification
[How to confirm the fix worked: memory should stabilize at X MB after load test]

### Prevention
[Lint rules, code review checklist items, or patterns to adopt]
```
