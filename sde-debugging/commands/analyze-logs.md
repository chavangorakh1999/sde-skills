---
description: Log analysis strategy and query patterns for the described failure mode
argument-hint: "<describe the failure mode or symptom to investigate>"
---

# /analyze-logs -- Log Analysis

Log analysis strategy and query patterns for a specific failure mode. Chains log-analysis -> performance-debug.

## Invocation

```
/analyze-logs p99 latency spike on /checkout endpoint — Node.js + PostgreSQL
/analyze-logs Auth service throwing 500s intermittently — need to find pattern
/analyze-logs Memory growing 200MB/hour — find what's accumulating
/analyze-logs                    # asks for failure mode description
```

## Workflow

### Step 1: Define the Investigation Goal

Clarify:
- What symptom are we investigating?
- What logging stack? (CloudWatch, Datadog, ELK, Loki, Pino to stdout)
- What time range? (start and end of anomaly + 30 min before as baseline)
- What service(s) are involved?

### Step 2: Design the Query Strategy

Apply **log-analysis** skill.

Structure the investigation as layers:
1. **Signal from noise** — filter to the affected timeframe and service
2. **Error distribution** — which errors, how many, trending up/down
3. **Pattern isolation** — is it specific endpoint, user, IP, tenant?
4. **Correlation ID trace** — follow a specific failing request end to end
5. **Root cause evidence** — find the first error in the chain

### Step 3: Produce Queries

Write concrete log queries for the logging platform:

**CloudWatch Logs Insights:**
```
# Error distribution
fields @timestamp, errorMessage, requestId
| filter level = "error" and @timestamp > "START_TIME"
| stats count(*) as cnt by errorMessage
| sort cnt desc | limit 20

# Latency by endpoint
fields path, durationMs
| filter action = "http.request" and @timestamp between START and END
| stats pct(durationMs, 99) as p99, avg(durationMs) as avg by path
| sort p99 desc
```

**Datadog Log Query:**
```
service:order-service status:error @http.status_code:5*
@requestId:<correlation-id>
@durationMs:>1000 @path:/checkout
```

**Elasticsearch (Kibana):**
```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "error" } },
        { "range": { "@timestamp": { "gte": "START", "lte": "END" } } }
      ]
    }
  },
  "aggs": {
    "error_types": { "terms": { "field": "errorMessage.keyword" } }
  }
}
```

### Step 4: Correlation ID Investigation

If distributed tracing is set up, trace a specific failing request:
1. Find a failing request ID from the error log
2. Search ALL services for that request ID
3. Reconstruct the call chain with timestamps
4. Find where the error originated

### Step 5: Interpret Findings

After running queries:
- What pattern emerges? (time-based, input-based, service-based)
- Is there a correlation with any variable? (deploy, traffic, external service)
- What's the first event in the failure chain?

Apply **performance-debug** skill if the issue is latency-related.

### Step 6: Identify Logging Gaps

What information was missing that would have made this easier?
- Missing structured fields (e.g., no `userId` in error logs)
- Missing correlation IDs
- No timing instrumentation
- Logs too verbose (signal buried in noise) or too sparse (missing context)

## Output

```
## Log Analysis Plan: [Failure Mode]

### Investigation Scope
Time range: [start] to [end]
Services: [list]
Platform: [CloudWatch / Datadog / ELK]

### Query Sequence

#### Query 1: Error Volume (Signal from Noise)
Purpose: Confirm error rate and time correlation
```[platform query]```
What to look for: [interpretation guidance]

#### Query 2: Error Distribution
Purpose: Find the most common error types
```[platform query]```
What to look for: [interpretation guidance]

#### Query 3: Pattern Isolation
Purpose: Is it specific endpoint/user/IP?
```[platform query]```

#### Query 4: Correlation ID Trace
Purpose: Follow a single failing request
```[platform query with example request ID]```

### Interpretation Guide
[How to read the results and what patterns indicate which root causes]

### What to Report Back
[The specific numbers/findings that will confirm or eliminate each hypothesis]

### Logging Improvements
[Gaps found that should be addressed]
```
