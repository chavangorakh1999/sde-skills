---
name: observability-design
description: "Logs/metrics/traces trinity, golden signals (latency/traffic/errors/saturation), SLO->SLI->SLA chain, alert fatigue prevention. Use when designing monitoring for a new service or fixing alerting that's too noisy."
---

## Observability Design

You can't debug what you can't see. Observability is the ability to understand the internal state of a system from its external outputs. Three pillars: logs, metrics, traces.

### Context

Service or system to instrument: **$ARGUMENTS**

---

### The Three Pillars

| Pillar | What it answers | Tool |
|--------|----------------|------|
| **Logs** | "What happened?" — event-level detail | Winston, Pino, Datadog Logs |
| **Metrics** | "How is it performing?" — aggregated numbers | Prometheus + Grafana, Datadog Metrics |
| **Traces** | "Where is the time going?" — distributed request journey | OpenTelemetry, Jaeger, Datadog APM |

Use all three. Logs without metrics means you can't detect problems without reading log lines. Metrics without traces means you can't find *which* service is slow.

---

### Structured Logging (Node.js)

```javascript
// Use structured JSON logging — never plain text
// Pino is the fastest Node.js logger (5-10x faster than Winston)
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: ['req.headers.authorization', 'body.password', 'body.token'],  // never log secrets
  serializers: {
    err: pino.stdSerializers.err,
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res
  }
});

// Request logging middleware
app.use((req, res, next) => {
  const requestLogger = logger.child({
    requestId: req.id,  // correlation ID for tracing related logs
    method: req.method,
    path: req.route?.path ?? req.path,
    userId: req.user?.id  // if authenticated
  });
  req.log = requestLogger;  // attach to request for downstream use
  next();
});

// Log levels as signals:
// error: something broke and needs human attention
// warn:  something unexpected happened but recovered (circuit breaker triggered, retry succeeded)
// info:  significant business event (user registered, order placed, payment captured)
// debug: detailed flow information (only in development/troubleshooting)
// trace: very verbose (request/response bodies, DB query results — never in production)

// Good log entries include:
// - What happened (action/event)
// - Who it happened to (userId, orderId, sessionId)
// - When (timestamp — structured loggers add this)
// - Result (success/failure with reason)
// - Duration (for any external call)

req.log.info({
  action: 'payment.captured',
  orderId,
  amount: charge.amount,
  durationMs: Date.now() - start,
  paymentProvider: 'stripe'
});

// Bad: logger.info('Payment successful') — no context
// Bad: logger.info(JSON.stringify(order)) — dumps everything, PII risk
```

---

### Metrics (Prometheus + Node.js)

```javascript
import { Registry, Counter, Histogram, Gauge } from 'prom-client';

const registry = new Registry();

// Counter: values that only go up (requests, errors, events)
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [registry]
});

// Histogram: distribution of values (latency, request size)
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_ms',
  help: 'HTTP request duration in milliseconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [10, 25, 50, 100, 200, 500, 1000, 2000, 5000],  // ms
  registers: [registry]
});

// Gauge: values that go up and down (queue depth, active connections)
const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  registers: [registry]
});

// Instrumentation middleware
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const route = req.route?.path ?? 'unknown';
    const labels = { method: req.method, route, status_code: res.statusCode };
    httpRequestsTotal.labels(labels).inc();
    httpRequestDuration.labels(labels).observe(Date.now() - start);
  });
  next();
});

// Expose metrics endpoint for Prometheus scraping
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', registry.contentType);
  res.end(await registry.metrics());
});
```

---

### The Four Golden Signals (Google SRE)

Monitor these four for every service:

```javascript
// 1. LATENCY — how long requests take
// Instrument: histogram with P50, P95, P99 percentiles
// Alert on: P99 > 2x baseline for > 5 minutes

// 2. TRAFFIC — how much demand is coming in
// Instrument: request counter per route
// Alert on: traffic drop to < 10% of normal (potential issue upstream)

// 3. ERRORS — rate of failed requests
// Instrument: counter labeled with status_code
// Alert on: 5xx error rate > 1% for > 2 minutes

// 4. SATURATION — how full the service is
// Instrument: CPU %, memory %, queue depth, connection pool usage
// Alert on: > 80% sustained for > 10 minutes

// Prometheus query examples (PromQL):
// Error rate: sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
// P99 latency: histogram_quantile(0.99, rate(http_request_duration_ms_bucket[5m]))
// Request rate: sum(rate(http_requests_total[1m]))
```

---

### SLO / SLI / SLA Chain

```
SLI (Service Level Indicator): the metric you measure
  Example: "percentage of requests that returned HTTP 200 within 500ms"

SLO (Service Level Objective): your internal target
  Example: "99.5% of requests return 200 within 500ms, measured over 28 days"
  This is what your team commits to maintaining

SLA (Service Level Agreement): external contractual commitment
  Example: "99% uptime or customer gets credit"
  SLA should always be weaker than SLO — SLA is external, SLO is your real target

Error Budget:
  SLO = 99.5% -> Error Budget = 0.5% of requests may fail
  28-day budget = 28 * 24 * 60 = 40,320 minutes
  Allowed downtime = 40,320 * 0.005 = 201.6 minutes/month (~3.4 hours)
```

```javascript
// Error budget calculation
function calculateErrorBudget(slo, windowDays = 28) {
  const totalMinutes = windowDays * 24 * 60;
  const errorBudgetPercent = (1 - slo) * 100;
  const allowedDowntimeMinutes = totalMinutes * (1 - slo);
  return {
    slo: `${slo * 100}%`,
    errorBudgetPercent: `${errorBudgetPercent.toFixed(2)}%`,
    allowedDowntimeMinutes: Math.floor(allowedDowntimeMinutes)
  };
}

calculateErrorBudget(0.999);  // 99.9% SLO -> 43.2 minutes/month error budget
calculateErrorBudget(0.9999); // 99.99% SLO -> 4.3 minutes/month (very expensive to maintain)
```

---

### Alert Design (Prevent Fatigue)

```javascript
// Good alerts: actionable, significant, rare
// Bad alerts: noisy, automatic-dismiss, wakes people up for non-critical issues

// Rules for good alerts:
// 1. Every alert must have a runbook (what to do when it fires)
// 2. If you ignore an alert more than twice, it's misconfigured — fix it
// 3. Alert on SYMPTOMS (user impact), not CAUSES (CPU high)
//    - Bad: CPU > 80% — may be fine (batch job), may be unrelated to user impact
//    - Good: P99 latency > 2s for 5 min — users are experiencing slowness

// Alert example (Prometheus Alertmanager):
// groups:
// - name: api-server
//   rules:
//   - alert: HighErrorRate
//     expr: |
//       sum(rate(http_requests_total{status_code=~"5.."}[5m])) /
//       sum(rate(http_requests_total[5m])) > 0.01
//     for: 2m
//     labels:
//       severity: critical
//     annotations:
//       summary: "Error rate {{ $value | humanizePercentage }} on {{ $labels.route }}"
//       runbook: "https://wiki.example.com/runbooks/high-error-rate"

// Severity levels:
// P1/Critical: wake someone up NOW (site down, data loss, financial impact)
// P2/High: fix today, don't wake up (service degraded, error rate elevated)
// P3/Medium: fix this week (trends worsening, non-critical service slow)
// P4/Low: fix when convenient (minor issues, cosmetic)
```

---

### Distributed Tracing

```javascript
// OpenTelemetry — vendor-neutral instrumentation
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT  // Jaeger, Tempo, Datadog
  }),
  instrumentations: [getNodeAutoInstrumentations()]  // auto-instruments Express, pg, redis, etc.
});

sdk.start();

// Manual span creation for business-level tracing:
import { trace } from '@opentelemetry/api';
const tracer = trace.getTracer('order-service');

async function processOrder(orderId) {
  const span = tracer.startSpan('process-order', {
    attributes: { 'order.id': orderId }
  });

  try {
    const order = await fetchOrder(orderId);  // child spans created auto by HTTP instrumentation
    span.setAttribute('order.total', order.total);
    span.setAttribute('order.items_count', order.items.length);
    return await chargeAndFulfill(order);
  } catch (err) {
    span.recordException(err);
    span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
    throw err;
  } finally {
    span.end();
  }
}
```

---

### Output Format

```
## Observability Design: [Service Name]

### Log Strategy
[Key events to log, log format, PII redaction, log level guidelines]

### Metrics
| Metric | Type | Labels | Alert Threshold |

### Golden Signals
| Signal | Measurement | SLO | Alert |

### SLO Definition
[SLI formula, SLO target, error budget calculation]

### Alert Runbooks
[For each alert: what fires it, what it means, how to investigate, how to fix]

### Tracing Strategy
[What to trace, sampling rate, exporter]

### Dashboard Layout
[Key panels for the service health dashboard]
```
