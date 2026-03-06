---
name: incident-response
description: "SEV classification, on-call response checklist, escalation path, stakeholder comms templates. Use when responding to a live production incident."
---

## Incident Response

When production is on fire, you need a structured process — not improvised heroics. This framework minimizes time to resolution and prevents secondary incidents caused by rushed decisions.

### Context

Incident description: **$ARGUMENTS**

---

### Incident Response Phases

```
1. DETECT    — Something is wrong (alert, customer report, team member)
2. ASSESS    — How bad is it? (SEV classification, scope, impact)
3. RESPOND   — Take action to mitigate
4. RESOLVE   — Service restored
5. DOCUMENT  — Postmortem (within 24-48h)
```

---

### Phase 1: Detect and Classify

```
SEV-1: > 20% users affected, data loss, security breach, > $10K/hr revenue impact
       -> Wake up on-call IMMEDIATELY. Escalate to leadership in 15 minutes.

SEV-2: 5-20% users affected, core feature broken, SLO breach
       -> Page on-call. Response within 30 minutes.

SEV-3: < 5% users affected, non-critical feature, workaround exists
       -> Respond next business hour.

SEV-4: Cosmetic, minor degradation within SLO
       -> Add to backlog.

When in doubt: classify higher and downgrade if evidence shows it's less severe.
Classifying too low causes slow response. Classifying too high wastes 30 minutes of sleep.
```

---

### Phase 2: First 5 Minutes (SEV-1/SEV-2)

```bash
# 1. Join incident channel (Slack: #incidents-active or create #inc-YYYYMMDD-[description])
# 2. Announce yourself: "I'm [name], taking incident commander role"
# 3. Assign roles:
#    - Incident Commander (IC): coordinates response, makes decisions, communicates
#    - Technical Lead: investigates and implements fixes
#    - Communications: handles stakeholder updates (keeps IC focused on tech)
#
# 4. Establish facts (don't assume):
# - When did it start? (check deploy times, cron jobs, traffic patterns)
# - What exactly is failing? (error rate, which endpoints, error messages)
# - What changed recently? (deploy, config, infrastructure, traffic spike)
# - Is it getting worse, stable, or improving?
#
# 5. Check the basics first:
# - Is there a recent deploy? -> consider rollback as first action
# - Is there a cron job or batch job that started recently?
# - Is there a traffic spike? (unexpected load)
# - Is a downstream dependency down? (check status pages: Stripe, AWS, SendGrid)
```

---

### Phase 3: Investigation Checklist

```javascript
// Start with the most likely causes:

// 1. Recent deployments
const recentDeploys = await gitLog({ since: '2h ago', format: 'short' });
// Was there a deploy in the last 2 hours? If yes and it correlates with incident start -> rollback

// 2. Error logs (filter to the incident timeframe)
// CloudWatch Logs / Datadog:
// filter @timestamp > "2024-01-15T14:00:00Z"
// | filter level = "error"
// | stats count(*) by errorMessage
// | sort count(*) desc

// 3. Database health
// - Check slow query log
// - Connection pool saturation
// - Disk usage (check if table/index growth hit limits)

// 4. External dependencies
// - Check status.stripe.com, status.aws.amazon.com
// - Check circuit breaker states in metrics
// - Look for elevated error rates to specific external APIs

// 5. Infrastructure metrics
// - CPU: is any instance pegged at 100%?
// - Memory: are any instances OOMing?
// - Disk: is any disk > 85% full?
// - Network: unusual traffic patterns?

// Narrow down: is it ALL requests or a specific endpoint/user segment?
// If specific endpoint: -> narrow the code search
// If specific user segment: -> data or feature flag issue
// If all requests: -> infrastructure or shared dependency issue
```

---

### Mitigation vs Fix

**Important distinction:**
- **Mitigation:** reduces or stops user impact NOW (rollback, disable feature, increase capacity)
- **Fix:** addresses the root cause (code change, config update, data repair)

In a live incident, mitigate first. Fix when users are no longer impacted.

```
Mitigation options (fast, reversible):
  - Rollback to last known good version (fastest, safest first move)
  - Disable the broken feature via feature flag
  - Increase capacity (scale up, add replicas)
  - Redirect traffic away from broken region/AZ
  - Enable maintenance mode (last resort — customer sees a message)

Only implement a code fix mid-incident if:
  - Rollback is impossible (no previous version)
  - The fix is small, well-understood, and lower risk than the incident
  - Someone reviews the fix before deploying (even in an incident)
```

---

### Stakeholder Communication Templates

**Initial alert (within 15 minutes of SEV-1/SEV-2 detection):**
```
Subject: [INCIDENT] [Short description] — [SEV Level] — Investigating

We are investigating an issue with [service/feature].

Impact: [X% of users / all users experiencing Y]
Started: [Time UTC] (detected at [Time UTC])
Status: Investigating

We will provide updates every [15/30] minutes.

IC: [Name]
```

**Update (every 15-30 minutes):**
```
Subject: [INCIDENT UPDATE] [Short description] — [Status: Investigating/Mitigating/Monitoring]

Update at [Time UTC]:

Current status: [Brief description of current situation]
Actions taken: [What has been done]
Next steps: [What you're doing now]

ETA to resolution: [Estimate or "unknown at this time"]

Next update: [Time]
```

**Resolution:**
```
Subject: [INCIDENT RESOLVED] [Short description] — Resolved at [Time UTC]

The incident has been resolved.

Duration: [X hours Y minutes]
Root cause: [Brief description — full postmortem in 24-48h]
Resolution: [What fixed it]
Users affected: [% or count]

A full postmortem will be shared within 48 hours.

Thank you for your patience.
```

---

### Post-Incident (Within 24-48h)

```
1. Write the postmortem (see postmortem skill)
2. Schedule postmortem review meeting (< 1 week after incident)
3. Assign all action items with owners and due dates
4. Update runbook with any new information learned
5. Check monitoring: would the alert have fired sooner if thresholds were different?
6. 30-day follow-up: are action items complete?
```

---

### Output Format

```
## Incident Response: [Description]

### Incident Summary
SEV: [1-4]
Impact: [Users, services, revenue]
Timeline: [Start, detected, mitigated, resolved]

### Immediate Actions Checklist
[ ] Incident commander assigned
[ ] Incident channel created
[ ] Team assembled

### Investigation Results
[Findings from each investigation step]

### Mitigation Applied
[What was done and when to stop user impact]

### Communication Log
[Who was notified, when, with what message]

### Root Cause (Preliminary)
[Initial theory — to be confirmed in postmortem]

### Next Steps
[ ] Write postmortem
[ ] Assign action items
[ ] Update runbook
```
