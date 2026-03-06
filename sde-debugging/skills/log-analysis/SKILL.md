---
name: log-analysis
description: "Log parsing strategy, correlation ID tracing, distributed trace reconstruction, log query patterns for common Node.js failure modes. Use when investigating an incident from logs."
---

## Log Analysis

Logs are the primary source of truth during incident investigation. The key is filtering noise to find signal quickly.

### Context

Problem or log pattern to analyze: **$ARGUMENTS**

---

### Structured Logging Prerequisites

Effective log analysis requires structured logs (JSON). If you're using plain text logs, this is technical debt worth paying off.

```javascript
// Good: structured JSON (machine-queryable, field-filterable)
{"level":"error","requestId":"req_01H2X","userId":"usr_123","action":"payment.charge",
 "error":"Connection timeout","stripeCustomerId":"cus_abc","durationMs":5003,"timestamp":"2024-01-15T14:22:11Z"}

// Bad: unstructured text (requires regex to parse, fragile)
// ERROR: Payment failed for user 123 after 5003ms - Connection timeout

// Pino structured logging (Node.js)
const logger = pino({ level: 'info', redact: ['password', 'token', 'authorization'] });
logger.error({
  action: 'payment.charge',
  userId: user.id,
  orderId: order.id,
  durationMs: Date.now() - start,
  err  // pino.stdSerializers.err serializes Error objects properly
}, 'Payment charge failed');
```

---

### Correlation IDs

Without a correlation ID (request ID), you can't connect related log entries across services.

```javascript
// Middleware: generate or propagate correlation ID
import { randomUUID } from 'crypto';

app.use((req, res, next) => {
  // Accept from upstream service or generate new
  req.id = req.headers['x-request-id'] ?? `req_${randomUUID().replace(/-/g, '').slice(0, 20)}`;
  res.setHeader('x-request-id', req.id);

  // Attach to request logger so all logs from this request share the ID
  req.log = logger.child({ requestId: req.id });
  next();
});

// Propagate to downstream services
async function callDownstreamService(req, path) {
  return fetch(`${SERVICE_URL}${path}`, {
    headers: { 'x-request-id': req.id }  // pass correlation ID downstream
  });
}

// Now: search logs by requestId to trace a single request across all services
// CloudWatch: { $.requestId = "req_01H2X" }
// Datadog: @requestId:req_01H2X
// Elasticsearch: { "query": { "match": { "requestId": "req_01H2X" } } }
```

---

### Log Query Patterns by Failure Mode

**Error spike investigation:**
```
// Find error rate over time (CloudWatch Logs Insights)
fields @timestamp, level, errorMessage, requestId
| filter level = "error"
| stats count(*) as errorCount by bin(5m)
| sort @timestamp asc

// Find most common error messages
fields errorMessage
| filter level = "error" and @timestamp > "2024-01-15T14:00:00Z"
| stats count(*) as occurrences by errorMessage
| sort occurrences desc
| limit 20
```

**Slow request investigation:**
```
// Find slowest requests
fields @timestamp, method, path, durationMs, requestId, userId
| filter durationMs > 1000
| sort durationMs desc
| limit 50

// Find P99 latency by endpoint
fields path, durationMs
| stats pct(durationMs, 99) as p99, pct(durationMs, 95) as p95, avg(durationMs) as avg
  by path
| sort p99 desc
```

**Authentication failures (brute force detection):**
```
// Find IPs with many failed auth attempts
fields @timestamp, action, clientIp, userId, errorCode
| filter action = "auth.login" and errorCode = "INVALID_CREDENTIALS"
| stats count(*) as failures by clientIp
| sort failures desc
| limit 20
```

**Database connection issues:**
```
// Find connection pool saturation events
fields @timestamp, action, poolSize, waitingClients, durationMs, requestId
| filter action = "db.query" and waitingClients > 0
| stats avg(waitingClients), max(waitingClients), count(*) by bin(1m)
```

---

### Node.js Specific Log Patterns

```javascript
// Uncaught exceptions and unhandled rejections
process.on('uncaughtException', (err) => {
  logger.fatal({ err, type: 'uncaughtException' }, 'Uncaught exception — process will exit');
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  logger.error({ reason, type: 'unhandledRejection' }, 'Unhandled promise rejection');
  // Do NOT exit here — may be a non-critical promise
});

// Search in logs: filter type = "unhandledRejection"
// -> reveals promises that swallowed errors silently

// Memory warnings
process.on('warning', (warning) => {
  if (warning.name === 'MaxListenersExceededWarning') {
    logger.warn({ warning: warning.message }, 'EventEmitter listener leak');
  }
});
// Search: filter warning.name = "MaxListenersExceededWarning"
// -> reveals EventEmitter listener leaks (common memory leak source)

// Express request timing
app.use((req, res, next) => {
  const start = process.hrtime.bigint();
  res.on('finish', () => {
    const durationMs = Number(process.hrtime.bigint() - start) / 1_000_000;
    req.log.info({
      action: 'http.request',
      method: req.method,
      path: req.route?.path ?? req.path,
      statusCode: res.statusCode,
      durationMs: Math.round(durationMs),
      contentLength: res.getHeader('content-length')
    });
  });
  next();
});
```

---

### Distributed Trace Reconstruction

When a request fails, reconstruct its path across all services:

```
Step 1: Find the error in any service
  -> Get the requestId from the error log entry

Step 2: Search all services for that requestId
  # All services, same requestId:
  requestId = "req_01H2XYZ"

Step 3: Sort by timestamp
  14:22:10.001 - api-gateway:    Received POST /api/orders (userId: usr_123)
  14:22:10.015 - auth-service:   Token validated (userId: usr_123)
  14:22:10.020 - order-service:  Creating order (orderId: ord_456)
  14:22:10.025 - inventory-svc:  Checking stock for 3 items
  14:22:10.985 - inventory-svc:  Stock check timeout after 960ms (ERROR)
  14:22:10.990 - order-service:  Inventory check failed — order not created (ERROR)
  14:22:11.001 - api-gateway:    Returning 503 to client

Step 4: Identify where time was lost
  -> inventory-service took 960ms before timeout (normal should be < 100ms)
  -> Inventory service is the bottleneck

Step 5: Investigate inventory-service logs in that timeframe
  -> Find what was happening in inventory-service during 14:22:10.025-14:22:10.985
```

---

### Log Retention Strategy

```
Production logs:
  Debug level:  Don't log in production (too verbose, high cost)
  Info level:   7-30 days retention (normal business events)
  Warn/Error:   90 days retention (incidents, anomalies)
  Fatal:        1 year retention (crash analysis)

Audit logs (security-sensitive: who did what):
  Minimum 1 year, often legally required to be 3-7 years
  Store separately from application logs
  Immutable — cannot be modified or deleted by application
  Access controls — only authorized personnel
```

---

### Output Format

```
## Log Analysis: [Problem/Incident]

### Search Strategy
[Which log streams, time range, initial queries]

### Key Findings
[Sorted by time — what the logs tell us happened]

### Error Pattern
[Most common error messages and their distribution]

### Performance Observations
[Slow queries, slow endpoints, timeline of degradation]

### Correlation Trail
[requestId trace across services if applicable]

### Gaps in Logging
[What information was missing that would have helped]

### Root Cause Indicators
[Log evidence pointing to the root cause]
```
