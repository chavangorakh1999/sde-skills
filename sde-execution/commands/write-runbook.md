---
description: On-call runbook for a service or specific alert condition
argument-hint: "<service name or alert name>"
---

# /write-runbook -- On-Call Runbook

Generate an operational runbook. Chains on-call-runbook -> incident-response.

## Invocation

```
/write-runbook High memory alert on the auth service
/write-runbook Payment service — all critical alerts
/write-runbook HighErrorRate alert on order-service
/write-runbook                    # asks for service/alert
```

## Workflow

### Step 1: Understand the Service

Extract:
- What does the service do? (1-2 sentences)
- What are its dependencies? (DB, cache, external APIs, downstream services)
- What's the SLO?
- What team owns it?
- What are the most common failure modes?

If writing for a specific alert: what metric fires it, what's the threshold, and what does it mean?

### Step 2: Define Existing Alerts

For each alert the service has:
- Name
- Fires when: (exact metric and threshold)
- Severity: SEV 1/2/3
- Typical cause

### Step 3: Write the Runbook

Apply **on-call-runbook** skill for each alert section:

**Triage questions:** what are the first 3 questions the on-call should answer?
**Diagnostic steps:** exact commands or queries to run, in order
**Probable causes:** ranked by likelihood, with specific evidence that points to each
**Fix procedures:** exact steps for each probable cause
**Escalation:** when to page a senior engineer or wake up the team

### Step 4: Maintenance Procedures

Standard operational tasks that on-call might need:
- Zero-downtime restart
- Scale up/down
- Enable maintenance mode
- Database failover

### Step 5: Apply Incident Response Template

Apply **incident-response** skill to add:
- SEV classification guide specific to this service
- Communication template customized for this service
- Escalation path

## Output

```markdown
# [Service Name] On-Call Runbook

**Last updated:** [today]
**Owner:** [team]
**Escalation:** [who / how]

## Service Overview
[1-2 sentences, dependency map]

## SLO
[Definition]

## Alert Reference
| Alert | SEV | Fires When | Most Likely Cause |

---

## Alert: [Alert Name]

**Fires when:** [metric] > [threshold] for [duration]
**SEV:** [N]

### Triage (first 5 minutes)
1. [Check X]
2. [Check Y]

### Diagnostic Steps
```bash
# Step 1: [what to run]
# Step 2: [what to look for]
```

### Probable Causes

**A. [Most common cause]**
Evidence: [what you'll see in logs/metrics]
Fix: [exact steps]
Commands:
```bash
[commands]
```

**B. [Second most common]**
...

### Escalate If
- [Condition 1]
- [Condition 2]

---

## Maintenance Procedures

### Restart Service
```bash
[commands]
```

### Scale Up
```bash
[commands]
```

## Communication Templates
[Initial message, update, resolution templates customized for this service]
```

## Next Steps

- "Should I create the deployment plan for this service? -> `/deploy-plan [service]`"
- "Want to write a postmortem template? -> `/postmortem [incident type]`"
