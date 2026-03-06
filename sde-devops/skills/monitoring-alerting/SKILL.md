---
name: monitoring-alerting
description: "Monitoring and alerting for Node.js apps: Prometheus metrics, Grafana dashboards, CloudWatch, PagerDuty alerting, golden signals, and SLO-based alerting. Use when designing or improving observability."
---

## Monitoring & Alerting

### Context

Monitoring problem or system to instrument: **$ARGUMENTS**

---

### The Four Golden Signals

```
Latency:       How long do requests take? (p50, p95, p99)
Traffic:       How much load? (requests/sec)
Errors:        What fraction of requests fail? (5xx rate, error rate)
Saturation:    How full is the system? (CPU, memory, queue depth)

Alert on: Error rate rising, Latency p95 > SLO threshold, Saturation > 80%
```

---

### Prometheus Metrics in Node.js

```javascript
// npm install prom-client
// src/metrics/index.js
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client';

export const registry = new Registry();

// Collect Node.js default metrics (CPU, memory, event loop lag, GC)
collectDefaultMetrics({ register: registry, prefix: 'nodejs_' });

// HTTP request duration (most important metric)
export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],  // seconds
  registers: [registry],
});

// HTTP request count
export const httpRequestTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [registry],
});

// Business metrics
export const userRegistrations = new Counter({
  name: 'user_registrations_total',
  help: 'Total user registrations',
  registers: [registry],
});

export const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Current number of active connections',
  registers: [registry],
});

// Queue depth (for BullMQ)
export const queueDepth = new Gauge({
  name: 'queue_depth',
  help: 'Number of jobs in queue',
  labelNames: ['queue_name', 'status'],
  registers: [registry],
});
```

---

### Metrics Middleware

```javascript
// src/middlewares/metricsMiddleware.js
import { httpRequestDuration, httpRequestTotal } from '../metrics/index.js';

export function metricsMiddleware(req, res, next) {
  const startTime = process.hrtime.bigint();

  res.on('finish', () => {
    const durationMs = Number(process.hrtime.bigint() - startTime) / 1e9;

    // Normalize route to avoid high cardinality (e.g., /users/:id not /users/123)
    const route = req.route?.path ?? req.path;

    const labels = {
      method: req.method,
      route,
      status_code: res.statusCode.toString(),
    };

    httpRequestDuration.observe(labels, durationMs);
    httpRequestTotal.inc(labels);
  });

  next();
}

// Expose metrics endpoint for Prometheus scraping
router.get('/metrics', async (req, res) => {
  // Restrict to internal network only
  if (!req.ip.startsWith('10.') && req.ip !== '127.0.0.1') {
    return res.status(403).end();
  }
  res.set('Content-Type', registry.contentType);
  res.end(await registry.metrics());
});
```

---

### Grafana Dashboard Queries

```promql
# Request rate (RPS)
rate(http_requests_total[5m])

# Error rate (5xx fraction)
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# Latency percentiles
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Latency by route (heatmap)
histogram_quantile(0.95,
  sum by (route, le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)

# Event loop lag (from default metrics)
nodejs_eventloop_lag_mean_seconds

# Memory usage
nodejs_heap_size_used_bytes / nodejs_heap_size_total_bytes * 100
```

---

### Alerting Rules (Prometheus AlertManager)

```yaml
# alerts/api.yml
groups:
  - name: api-alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate: {{ $value | humanizePercentage }}"
          description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes. SLO is 99.9% success rate."
          runbook: https://runbooks.example.com/high-error-rate

      # High latency
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency > 2s: {{ $value | humanizeDuration }}"

      # Service down
      - alert: ServiceDown
        expr: up{job="api"} == 0
        for: 1m
        labels:
          severity: critical
          pagerduty: true
        annotations:
          summary: "API service is down"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          nodejs_heap_size_used_bytes / nodejs_heap_size_total_bytes > 0.9
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Node.js heap usage > 90%"
```

---

### CloudWatch (AWS)

```javascript
// src/utils/cloudwatch.js — custom metrics for AWS
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

const cw = new CloudWatchClient({ region: process.env.AWS_REGION });

export async function emitMetric(name, value, unit = 'Count', dimensions = []) {
  if (process.env.NODE_ENV !== 'production') return;  // only emit in prod

  try {
    await cw.send(new PutMetricDataCommand({
      Namespace: 'MyApp/API',
      MetricData: [{
        MetricName: name,
        Value: value,
        Unit: unit,
        Timestamp: new Date(),
        Dimensions: dimensions,
      }]
    }));
  } catch (err) {
    logger.warn({ err }, 'failed to emit CloudWatch metric');
  }
}

// Usage:
await emitMetric('UserRegistrations', 1, 'Count', [
  { Name: 'Environment', Value: process.env.NODE_ENV }
]);
await emitMetric('RequestLatency', durationMs, 'Milliseconds', [
  { Name: 'Route', Value: '/api/v1/users' }
]);
```

---

### SLO-Based Alerting

```
Define SLOs first, then set alerts to warn before budget burns out.

SLO: 99.9% success rate over 30 days
Error budget: 0.1% * 30 * 24 * 60 = 43.2 minutes/month

Burn rate alert: alert when error budget burns X times faster than normal
  15x burn rate → alert fires in 1 hour   (critical, page on-call)
  6x burn rate  → alert fires in 6 hours  (warning, Slack)
  1x burn rate  → alert fires in 3 days   (informational, ticket)

Prometheus multi-burn-rate:
  - alert: SLOBurnRateCritical
    expr: |
      (
        job:http_errors:rate1h / job:http_requests:rate1h > 15 * (1 - 0.999)
      ) and (
        job:http_errors:rate5m / job:http_requests:rate5m > 15 * (1 - 0.999)
      )
```
