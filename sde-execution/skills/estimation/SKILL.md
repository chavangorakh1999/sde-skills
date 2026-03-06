---
name: estimation
description: "Three-point estimation (optimistic/expected/pessimistic), confidence intervals, risk-adjusted sizing, cone of uncertainty. Use when estimating engineering effort for features or projects."
---

## Engineering Estimation

Software estimates are wrong. The goal of estimation is not precision — it's to communicate uncertainty honestly and make better decisions about scope and timeline.

### Context

Work to estimate: **$ARGUMENTS**

---

### Three-Point Estimation (PERT)

For any non-trivial task, estimate three values:

```
L = Low (optimistic)  — best case: everything works, no surprises
M = Mid (expected)    — most likely: typical friction and unknowns
H = High (pessimistic) — worst case: significant unexpected complexity

PERT estimate = (L + 4M + H) / 6
Standard deviation = (H - L) / 6

Example: Implement Redis caching for product catalog
L = 2 days  (we've done this before, no unknowns)
M = 4 days  (some integration issues, cache invalidation logic)
H = 8 days  (cache consistency bugs, Redis cluster issues, write-through complexity)

PERT estimate = (2 + 4×4 + 8) / 6 = (2 + 16 + 8) / 6 = 26/6 = 4.3 days
Standard dev  = (8 - 2) / 6 = 1 day

Commit to: M = 4 days (the expected estimate)
Confidence interval: 3-6 days (PERT ± 2σ)
```

---

### Risk Factors

Adjust estimates upward for:

```
First time using this technology/library:    +25-50% (learning curve)
Integration with undocumented external API:  +25-50% (surprises guaranteed)
Modifying complex legacy code:               +50-100% (understanding takes time)
No existing tests to run:                    +25% (manual verification cost)
Requires coordinating with another team:     +25-50% (scheduling, waiting)
Unclear requirements:                        stop estimating, get requirements clear
Performance requirements to meet:            +30-50% (profiling, optimization cycles)
Security requirements (auth, encryption):    +20-40% (careful implementation + review)
```

---

### The Cone of Uncertainty

Estimates become more accurate as you learn more:

```
Phase of work               Accuracy range
----------------------------------------------
Initial concept             4x — 0.25x (8x range)
Approved product definition 2x — 0.5x  (4x range)
Requirements complete       1.5x — 0.7x (2x range)
UI design complete          1.25x — 0.8x (1.5x range)
Detailed design complete    1.1x — 0.9x (1.2x range)

In practice: never commit to a date from a first-pass estimate.
First estimate: "probably 2-6 weeks"
After design: "4-6 weeks"
After first week of implementation: "5.5-6.5 weeks"
```

---

### Story Point Calibration

If your team uses story points (Fibonacci: 1, 2, 3, 5, 8, 13):

```
Reference story: a simple CRUD endpoint with validation and tests
              -> this is "3 points" (baseline)

Then calibrate:
1 point:  ~4 hours — trivial change, no integration
2 points: ~1 day — small feature, clear requirements, known patterns
3 points: ~2 days — medium feature, some complexity (BASELINE)
5 points: ~3-4 days — complex feature, some unknowns
8 points: ~1 week — large feature, significant unknowns, needs design
13 points: too big, split it

Story points ≠ hours — they represent complexity + uncertainty + effort
Don't convert to hours in planning. Use yesterday's weather:
  "We deliver ~30 points per sprint on average" -> planning capacity
```

---

### Velocity and Yesterday's Weather

```javascript
// The best predictor of future velocity is recent past velocity

const sprintHistory = [28, 31, 26, 33, 29, 27, 32];
const avgVelocity = sprintHistory.reduce((a, b) => a + b) / sprintHistory.length;
// = 29.4 points per sprint

// Planning: commit to avgVelocity × 0.9 (10% buffer)
// = 26.5 -> commit to 26 points

// Velocity adjustments:
// New team member ramping: -10% per junior added, -5% per senior
// Planned vacation days: reduce proportionally (team_days / sprint_days)
// Major technical debt work planned: -20-30%

// If you don't track story points: use raw engineering-hours
// average hours in a sprint per engineer = (working days × 6 focused hours)
// subtract meetings, interruptions, code review time: ~4 focused hours/day
// 2-week sprint × 5 days × 4 hours = 40 focused hours per engineer
```

---

### What Breaks Estimates

```
1. Unclear requirements — scope discovered during implementation
   Mitigation: always clarify with product before estimating

2. External dependencies not under your control (API docs wrong, partner team delay)
   Mitigation: spike external integrations before estimating the full story

3. Underestimated code quality work (tests, documentation, PR review iterations)
   Mitigation: always include review, test, and deploy time in estimates

4. Environmental problems (broken dev environment, flaky CI, wrong staging data)
   Mitigation: track and report as waste; fix the environment

5. Estimation of total = sum of parts + integration testing time
   Mitigation: add 15-20% "integration tax" for tasks with many moving parts

6. "It's just like X" — no two tasks are exactly alike
   Mitigation: be specific about what's different; estimate the difference
```

---

### Output Format

```
## Estimation: [Feature/Task]

### Stories

| Story | L (opt) | M (expected) | H (pessimistic) | PERT | Risk Factors |
|-------|---------|-------------|----------------|------|-------------|

### Summary

Total PERT estimate: [X days]
Confidence range:   [Y-Z days]
Commit estimate:    [M days total]

### Risk Register
| Risk | Probability | Impact on Estimate | Mitigation |

### Assumptions Made
[List assumptions — if any are wrong, the estimate changes]

### What's Not Included
[Scope that was excluded from the estimate]
```
