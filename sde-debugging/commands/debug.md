---
description: Structured debugging — hypotheses, elimination tests, root cause, fix
argument-hint: "<symptom description>"
---

# /debug -- Structured Debugging

Systematic debugging from symptom to root cause to fix. Chains root-cause-analysis -> relevant deep-dive skill.

## Invocation

```
/debug Users report intermittent login failures, only on mobile, only in Safari
/debug Memory grows 200MB/hour and service restarts every 6 hours
/debug POST /checkout returning 200 but not creating orders 10% of the time
/debug p99 latency on /search jumped from 200ms to 3000ms after yesterday's deploy
/debug                    # asks for symptom description
```

## Workflow

### Step 1: Define the Symptom Precisely

Never debug a vague feeling. Force specificity:
- What exactly is failing? (error message, wrong value, slow response, crash)
- What's the frequency? (always, 10%, once)
- What's the scope? (all users, specific users, specific input)
- When did it start? (correlate with events: deploys, config changes, traffic)
- Is it getting worse, stable, or sporadic?

### Step 2: Form Hypotheses

Before touching anything, list 3-5 hypotheses ranked by probability:
- "Most likely: N+1 query introduced in recent deploy"
- "Less likely: Redis cache eviction causing fallback to slow path"
- "Unlikely but worth checking: Safari-specific CORS issue"

Good hypotheses are:
- Specific and falsifiable (you can test them)
- Ranked (test most likely first)
- Based on evidence (symptom characteristics)

### Step 3: Design Elimination Tests

For each hypothesis, define: "If this is true, I expect to see X. Let me check."

```
Hypothesis: N+1 query
Test: Enable query logging and look for repeated identical queries in one request
Evidence for: See "SELECT * FROM users WHERE id = ?" running 50 times per request
Evidence against: Query log shows only 2-3 unique queries per request

Hypothesis: Safari CORS issue
Test: Check browser console for CORS errors; test same request in Chrome
Evidence for: CORS error in Safari console, works in Chrome
Evidence against: Same error in Chrome too (not Safari-specific)
```

Eliminate hypotheses with evidence, don't just look for confirmation.

### Step 4: Investigate

Apply the appropriate deep-dive skill based on the symptom:
- Slow latency: **performance-debug** skill
- Memory growth: **memory-debug** skill
- Error spike: **log-analysis** skill
- Production incident: **incident-response** skill

### Step 5: Isolate the Root Cause

```
Isolation techniques:
- Binary search: disable half the changes since last good state
- Narrow scope: reproduce in isolation (single request, minimal data)
- Instrument: add timing/logging at each layer, find where time/errors come from
- Reproduce locally: can you trigger it locally? If yes, set a breakpoint
- Check git history: what changed between last good state and now?

// Minimal reproduction is a superpower:
// Instead of debugging in production, find the smallest possible test case
// that reproduces the bug. Once you can reproduce reliably, you can fix reliably.
```

### Step 6: Verify the Fix

Don't just fix and deploy. Verify:
1. Does the fix address the root cause you identified? (not just the symptom)
2. Does it reproduce? Run the fix against the repro case.
3. Any edge cases? (what if input is null, empty, max length?)
4. Does it break anything else? (run the test suite)
5. What monitoring proves it's fixed in production? (check after deploy)

### Step 7: Prevent Recurrence

After fixing, note: "How do we make sure this can't happen again?"
- Test that would have caught it
- Monitoring that would have detected it earlier
- Code review checklist item
- Process change

## Output

```
## Debug Session: [Symptom]

### Symptom Definition
[Precise description with frequency, scope, first occurrence]

### Hypotheses (ranked)
1. [Most likely — why]
2. [Less likely — why]
3. ...

### Investigation

#### Testing Hypothesis 1
Evidence found: [What you looked at]
Result: Confirmed / Eliminated / Need more data

#### Testing Hypothesis 2
...

### Root Cause
[Specific: file.js:line, query, or external factor]
[Why it causes the symptom]

### Fix
[Code change or configuration change]

### Verification
[How to confirm the fix worked in production]

### Prevention
[Test / monitoring / process change to prevent recurrence]
```

## Next Steps

- "Should I write a postmortem? -> `/postmortem [incident description]`"
- "Want to review the fix code? -> `/review-code [fixed code]`"
