---
description: Blameless postmortem with 5-Why analysis, impact metrics, and SMART action items
argument-hint: "<incident description>"
---

# /postmortem -- Blameless Postmortem

Write a complete blameless postmortem. Chains root-cause-analysis -> postmortem.

## Invocation

```
/postmortem Checkout service returned 503s for 45 minutes during peak traffic on Black Friday
/postmortem Database primary failed, replica promotion took 8 minutes, all writes failed
/postmortem Memory leak caused auth service to restart every 6 hours over 3 days
/postmortem                    # asks for incident description
```

## Workflow

### Step 1: Gather Incident Data

Ask for:
- What was the symptom? (error rate, latency, feature broken)
- When did it start and end? (UTC timestamps)
- Who was affected? (% users, which features, revenue impact)
- What was the sequence of events? (deploy, cron job, traffic spike, alert, etc.)
- What was done to resolve it?
- Do you have relevant logs or metrics to reference?

### Step 2: Reconstruct the Timeline

Work through the event sequence chronologically:
- What normal baseline looked like before the incident
- First sign of anomaly (even if not immediately noticed)
- When the alert fired (and if it should have fired earlier)
- Investigation steps and findings
- Mitigation actions and their effects
- Resolution

### Step 3: Quantify Impact

Apply **postmortem** skill:
- Duration (minutes of degradation)
- Users affected (% or absolute number)
- Error rate (peak, sustained)
- Revenue estimate (failed transactions × average order value, if applicable)
- SLA breach? (if yes, customer credits owed)

### Step 4: Run 5 Whys

Apply **root-cause-analysis** skill:
- Start with the immediate symptom
- Ask "why?" at each level
- Stop at a systemic fix, not human error
- May produce 2-3 parallel root cause chains

### Step 5: Identify Contributing Factors

Separate from root causes — conditions that made it worse or enabled it:
- Missing monitoring (late detection)
- No rollback capability (slow recovery)
- Insufficient runbook (long time to identify fix)
- Alert fatigue (alert existed but was ignored)

### Step 6: Write SMART Action Items

For each root cause and major contributing factor:
- Specific action (not "improve monitoring")
- Assigned owner (one person, not "team")
- Due date (within 2-4 weeks for most items)
- Verification criteria (how do we know it's done?)

### Step 7: What Went Well

Honest assessment — what worked? Safety mechanisms that held? Instruments that helped? Fast response? These reinforce good practices.

## Output

```markdown
# Postmortem: [Title]

**Date:** [Incident date]
**Duration:** [Start] to [End] ([X hours Y minutes])
**Severity:** SEV-[N]
**Status:** Draft / In Review / Accepted

## Impact
- Users affected: [X% or N users]
- Error rate: [peak X%, sustained Y%]
- Duration: [N minutes]
- Revenue impact: [$X estimated]
- SLA breach: Yes/No

## Timeline

| Time (UTC) | Event |
|------------|-------|
...

## Root Cause Analysis

### 5 Whys
Why 1: [Symptom] -> [Immediate cause]
Why 2: -> [Underlying cause]
Why 3: -> [Root cause]

**Root cause:** [Systemic issue]

### Contributing Factors
1. ...
2. ...

## What Went Well
- ...

## What Went Poorly
- ...

## Action Items

| # | Action | Root Cause Addressed | Owner | Due | Status |
|---|--------|---------------------|-------|-----|--------|

## Lessons Learned
1. ...
```

## Next Steps

- "Should I create tickets for the action items? -> `/write-tickets [action items]`"
- "Want to design a runbook for this scenario? -> `/write-runbook [service]`"
- "Should I improve the monitoring? -> ask about observability-design"
