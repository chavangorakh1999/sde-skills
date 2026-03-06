---
name: on-call-runbook
description: "Runbook template: symptom -> probable cause -> diagnostic steps -> fix procedure -> escalation for each alert. Use when creating on-call documentation for a service."
---

## On-Call Runbook

A runbook is the operational guide for an alert: what it means, how to diagnose it, and how to fix it. A good runbook means a 2am alert doesn't require waking up an expert.

### Context

Service or alert to document: **$ARGUMENTS**

---

### Runbook Structure

```markdown
# [Service Name] On-Call Runbook

**Last updated:** [Date]
**Owner:** [Team]
**Escalation:** [Who to call if runbook doesn't resolve it]
**Postmortems:** [Link to past postmortems for context]

---

## Service Overview

[1-2 sentences: what does this service do, what does it depend on, what depends on it]

**Architecture:**
[Client] -> [Service] -> [Database, Cache, External APIs]

**Critical dependencies:**
- PostgreSQL: stores [what]
- Redis: used for [what]
- Stripe API: used for [what]

**SLO:** 99.9% of requests return 2xx within 500ms (28-day window)

---

## Alert Reference

| Alert Name | SEV | Likely Cause | Quick Fix |

---

## Alert: HighErrorRate

**Fires when:** HTTP 5xx error rate > 1% for 5 minutes
**Expected baseline:** < 0.1% error rate
**SEV:** 2

### Triage

```bash
# Step 1: Check error logs (last 15 minutes)
# CloudWatch Logs Insights:
fields @timestamp, level, errorMessage, requestId, path
| filter level = "error" and @timestamp > ago(15m)
| stats count(*) as cnt by errorMessage
| sort cnt desc | limit 10

# Step 2: Check recent deployments
git log --oneline --since="2h ago"
# Was there a deploy in the last 2 hours? -> consider rollback

# Step 3: Check downstream services
# Stripe status: https://status.stripe.com
# AWS status: https://health.aws.amazon.com
# Check circuit breaker state in metrics dashboard
```

### Probable Causes and Fixes

**A. Bad deploy (most common)**
- Evidence: errors started after a deploy
- Fix: rollback to previous image tag
- Command: `kubectl rollout undo deployment/[service]` or re-deploy previous tag
- Time: 2-5 minutes

**B. Database connection exhausted**
- Evidence: error messages contain "connection pool" or "ECONNREFUSED"
- Fix: restart service to reset connections (short term); investigate root cause (long term)
- Check: `SELECT count(*) FROM pg_stat_activity WHERE datname = '[db]';`
- If > 80 connections: pool is exhausted
- Fix: restart pods to reset pool; add PgBouncer if recurring

**C. Downstream service (e.g., Stripe) unavailable**
- Evidence: errors from external API calls (Stripe, SendGrid, etc.)
- Check: downstream service status page
- Fix: circuit breaker should handle this; if not, disable the affected feature via flag
- Communication: update status page, notify customers if extended

**D. Disk full**
- Evidence: "ENOSPC" or "no space left on device" in errors
- Check: `df -h` on the affected instance
- Fix: delete old logs/temp files; increase disk; rotate logs

### Escalate If:
- Error rate > 5% for > 10 minutes
- Evidence of data loss or corruption
- Root cause not identified after 30 minutes
- Off-hours and you need database access

---

## Alert: HighLatency

**Fires when:** P99 request duration > 2000ms for 5 minutes
**Expected baseline:** P99 < 500ms
**SEV:** 3 (2 if revenue-impacting)

### Triage

```bash
# Step 1: Identify which endpoints are slow
fields path, durationMs
| filter @timestamp > ago(15m)
| stats pct(durationMs, 99) as p99 by path
| sort p99 desc | limit 10

# Step 2: Check DB query times
# Look for: slow_query in logs, or check pg_stat_statements
SELECT query, mean_exec_time, calls, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

# Step 3: Check external API latency
# Look for: action="external_call" and high durationMs in logs
```

### Probable Causes and Fixes

**A. Slow database queries**
- Evidence: high durationMs on DB-heavy endpoints; slow_query log entries
- Fix: EXPLAIN ANALYZE the slow query; add index if missing
- Immediate: `CREATE INDEX CONCURRENTLY ...` (non-blocking)
- Verify: query time should drop within 1-2 minutes of index creation

**B. N+1 query problem (after code change)**
- Evidence: many identical queries in logs for a single request
- Fix: rollback the code change that introduced it
- Long-term: add query count assertion to tests

**C. External service slow (Stripe, etc.)**
- Evidence: durationMs for external call > 1000ms
- Fix: circuit breaker should fail fast; if not working, check circuit breaker config
- Immediate: none (can't make Stripe faster); add to status page

**D. Memory pressure / GC pauses**
- Evidence: P99 high but P50 fine; intermittent not sustained
- Check: memory metrics — is heap growing?
- Fix: increase memory allocation; investigate memory leak

---

## Alert: ServiceDown

**Fires when:** Service health check fails for 3 consecutive checks (90 seconds)
**SEV:** 1

### Immediate Actions (first 5 minutes)

1. Check if it's all instances or one: `kubectl get pods -n [namespace]`
2. If pod is OOMKilled: `kubectl describe pod [pod-name]` -> increase memory
3. If CrashLoopBackOff: `kubectl logs [pod-name] --previous` -> look for fatal error
4. Try to restart: `kubectl rollout restart deployment/[service]`
5. If restart fails: rollback to previous version

### Communication

Send within 5 minutes:
"We are experiencing a service outage for [service]. Impact: [users affected].
Engineering team is actively investigating. Next update in 15 minutes."

---

## Maintenance Procedures

### Zero-Downtime Restart
```bash
# Kubernetes: rolling restart (replaces pods one at a time)
kubectl rollout restart deployment/[service] -n [namespace]

# Verify rollout status
kubectl rollout status deployment/[service] -n [namespace]
```

### Scale Up (if traffic spike)
```bash
kubectl scale deployment/[service] --replicas=10 -n [namespace]
```

### Emergency DB Read-Only Mode
[Step-by-step to enable read-only mode if writes must be suspended]
```

---

### Output Format

Produce a complete runbook document for the service/alert, following the template above. Include:
1. Service overview and dependency map
2. SLO definition
3. One section per alert with: fires when, triage steps, probable causes, fixes, escalation
4. Maintenance procedures for common operational tasks
5. Escalation contacts and when to use them
