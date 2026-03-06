---
name: postmortem
description: "Blameless postmortem: impact quantification, timeline reconstruction, 5-Why RCA, contributing factors, SMART action items. Use after any significant incident."
---

## Blameless Postmortem

A postmortem is a learning document, not a blame document. The goal: understand what happened, quantify the impact, find the systemic root cause, and implement fixes that prevent recurrence.

### Context

Incident to document: **$ARGUMENTS**

---

### Blameless Culture

The blameless postmortem, pioneered by Google SRE, operates on a key insight: when engineers fear blame, they hide information, underreport incidents, and avoid risk — all of which make systems worse. When engineers feel safe, they report honestly, which makes systems better.

**Rules:**
- Name systems, processes, and tools — never people
- "The deploy process lacked automated rollback" — good
- "Alice forgot to test before deploying" — bad, also wrong (Alice was set up to fail by a process that allowed deployment without automated testing)
- Assume everyone did their best with the information and tools they had

---

### Postmortem Template

```markdown
# Postmortem: [Incident Title]

**Date:** [Date of incident]
**Duration:** [Start time] to [End time] ([X hours Y minutes])
**Severity:** SEV-1 / SEV-2 / SEV-3 / SEV-4
**Authors:** [Names — but we don't blame in the document body]
**Status:** In Review | Accepted | Action Items Complete

---

## Impact

**Users affected:** [% of users or absolute count]
**Error rate:** [% of requests failed, peak and sustained]
**Revenue impact:** [$X estimated lost revenue or transactions failed]
**SLA breach:** [Yes/No — if yes, customer credits owed]
**Services affected:** [List of services impacted]

---

## Timeline

All times UTC.

| Time | Event |
|------|-------|
| 14:00 | Deploy of version 2.4.1 to production |
| 14:03 | Error rate begins rising (missed — no alert set) |
| 14:18 | First customer support ticket received |
| 14:22 | On-call engineer paged via PagerDuty |
| 14:35 | Root cause identified: N+1 query in login handler |
| 14:41 | Rollback to version 2.4.0 initiated |
| 14:47 | Service fully recovered, error rate normal |

---

## Root Cause Analysis

[Use 5 Whys — see root-cause-analysis skill]

**Root cause:** [Systemic issue, not human error]

**Contributing factors:**
1. [Factor 1 — condition that made this possible]
2. [Factor 2 — condition that made it worse or harder to detect]
3. [Factor 3 — condition that slowed recovery]

---

## What Went Well

[Honest assessment — systems or processes that worked as designed]
- Alert fired within 5 minutes of SLO breach (once detected)
- Rollback procedure completed in < 8 minutes
- On-call engineer had runbook and didn't need to escalate

---

## What Went Poorly

[Systems or processes that failed, without blaming individuals]
- Error rate monitor existed but threshold was set too high (5% not 0.5%)
- No automated smoke test after deploy (N+1 query would have been caught)
- Deploy ran during high-traffic hour (no deployment window policy)

---

## Action Items

| # | Action | Addresses | Owner | Due | Status |
|---|--------|-----------|-------|-----|--------|
| 1 | Lower error rate alert threshold to 0.5% | Late detection | | 2024-01-22 | Open |
| 2 | Add post-deploy smoke test for auth endpoint | No smoke test | | 2024-01-22 | Open |
| 3 | Add deployment window policy (no deploys 12-15 UTC) | Deploy timing | | 2024-01-29 | Open |
| 4 | Add N+1 query count assertion to auth integration tests | Root cause | | 2024-01-22 | Open |

---

## Lessons Learned

[2-3 key takeaways for the broader team]
1. Post-deploy smoke tests are cheap insurance for high-traffic endpoints
2. Alert thresholds should be based on SLO error budget, not rough guesses
3. Every deployment is an implicit experiment — treat it like one
```

---

### SEV Classification

Use consistently so "SEV-1" means the same thing to everyone:

```
SEV-1 (Critical):
  - All users affected or > 20% of traffic failing
  - Data loss or corruption occurring
  - Security breach in progress
  - Revenue loss > $10K/hour
  Response: Wake up on-call, escalate to CTO/VP within 15 min

SEV-2 (Major):
  - > 5% of users affected or core feature broken for segment
  - Service degraded with SLO breach
  - No data loss, workaround available
  Response: Page on-call, response within 30 minutes

SEV-3 (Minor):
  - < 5% of users affected, edge cases
  - Non-critical feature broken
  - SLO not breached
  Response: Respond during business hours

SEV-4 (Low):
  - Cosmetic issues, minor UX problems
  - Performance slightly degraded but within SLO
  Response: Add to backlog, fix in normal sprint
```

---

### Output Format

Produce the complete postmortem document, then:

```
## Review Checklist

Before marking this postmortem as Accepted:
[ ] Timeline is accurate (verified with logs, not memory)
[ ] Root cause is a systemic issue, not a person
[ ] Every action item has an owner, due date, and is SMART
[ ] Impact is quantified (% users, revenue, duration)
[ ] "What went well" section is honest (not just negative)
[ ] Action items reviewed and assigned in team meeting

## Follow-Up Dates
- Action items review: [Date]
- 30-day check: are action items complete?
```
