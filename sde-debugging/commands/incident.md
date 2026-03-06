---
description: Incident response checklist and stakeholder comms template for a live production incident
argument-hint: "<incident description>"
---

# /incident -- Live Incident Response

Structured response for a live production incident. Chains incident-response -> postmortem template.

## Invocation

```
/incident Database primary is unresponsive — all writes failing — SEV-1
/incident Checkout page returning 500 errors — 30% of users affected
/incident Memory leak causing auth service restarts every 2 hours
/incident                    # asks for description
```

## Workflow

### Step 1: Classify Severity

Apply **incident-response** SEV classification immediately:

```
SEV-1: > 20% users, data loss, security breach, > $10K/hr revenue
  -> WAKE UP on-call NOW. Leadership notified in 15 min.

SEV-2: 5-20% users, core feature broken, SLO breach
  -> Page on-call. Respond within 30 min.

SEV-3: < 5% users, non-critical feature, workaround exists
  -> Business hours response.
```

When in doubt: classify higher and downgrade when you have more information.

### Step 2: Activate Response

Produce the immediate checklist:

```
[ ] Create incident channel: #inc-YYYYMMDD-short-description
[ ] Announce: "Incident Commander: [name]. Status: investigating."
[ ] Assign roles: IC, Technical Lead, Communications
[ ] Post initial stakeholder message (< 15 min from detection)
```

### Step 3: Establish Facts (First 5 Minutes)

Produce a list of quick investigation steps for this specific incident:
- What changed recently? (recent deploys, config changes, cron jobs)
- What exactly is failing? (endpoint, error message, error rate)
- Is it getting worse or stable?
- Any external dependencies down? (check status pages)

### Step 4: Investigation Checklist

Tailored to the incident type:
- **503/500 errors:** check error logs, check downstream service health, check DB connections
- **Memory/CPU:** check metrics, recent deploys, cron job timing
- **Latency spike:** check DB slow queries, external API latency, traffic spike
- **Data issues:** check recent migrations, data pipelines, external data source

### Step 5: Mitigation Options

Present options in order of: fastest/safest first

1. Rollback to last known good version (if recent deploy correlates with incident)
2. Disable broken feature via feature flag
3. Scale up resources (if capacity issue)
4. Enable maintenance mode (last resort)

For each: estimated time, risk of mitigation, rollback-from-mitigation option.

### Step 6: Communication Templates

Produce tailored stakeholder messages:

**Initial (< 15 min):**
```
We are investigating an issue with [service].
Impact: [X% of users experiencing Y]
Started: [time UTC]
IC: [name] | Status: Investigating
Next update: in 15 minutes
```

**Updates (every 15-30 min):**
```
Update: [time UTC]
Status: [Investigating / Identified / Mitigating / Monitoring]
Actions taken: [what you did]
Current state: [what's happening now]
ETA: [estimate or "unknown"]
Next update: [time]
```

**Resolution:**
```
RESOLVED at [time UTC]
Duration: [X min]
Impact: [% users, features affected]
Root cause: [brief — full postmortem in 24-48h]
Resolution: [what fixed it]
```

### Step 7: Post-Incident Plan

After resolution, produce:
- Postmortem owner and due date (within 48h)
- Action items to investigate further
- Monitoring improvements to prevent late detection

## Output

```
## Incident Response: [Title]

### Classification
SEV: [1-4]
Impact: [users, services, revenue]
Detection: [how it was found, when]

### Immediate Actions
[ ] Incident channel created
[ ] Team assembled
[ ] IC assigned
[ ] Initial comms sent

### Investigation Steps (next 30 minutes)
1. [Check X]
2. [Check Y]
3. [Check Z]

### Mitigation Options
1. [Option A — fastest] — Risk: low — Time: 2 min
2. [Option B] — Risk: medium — Time: 10 min

### Communication

**Initial message:**
[Template]

**Update template:**
[Template]

**Resolution template:**
[Template]

### Post-Incident
[ ] Postmortem owner: ___
[ ] Postmortem due: ___
[ ] Immediate action items identified: ___
```

## Next Steps (after resolution)

- "Write the postmortem -> `/postmortem [incident description]`"
- "Create tickets for action items -> `/write-tickets [action items]`"
