---
name: sprint-planning
description: "Capacity math (velocity x availability), dependency mapping, risk identification, commitment guard heuristics. Use when planning a sprint or quarterly roadmap."
---

## Sprint Planning

Good sprint planning turns a backlog into a credible commitment. The key: don't commit to more than you can actually deliver, and surface risks before the sprint starts.

### Context

Sprint to plan: **$ARGUMENTS**

---

### Step 1: Calculate Real Capacity

```javascript
// Don't use raw working days — use focused engineering hours

const sprintDays = 10;  // 2-week sprint
const engineersOnTeam = 4;

// Dedductions from each engineer's day:
const meetingsHours = 1.5;     // standups, sprint ceremonies, 1:1s
const codeReviewHours = 0.5;  // reviewing others' PRs
const slackInterruptHours = 0.5; // context switching, ad-hoc questions

const focusedHoursPerDay = 8 - meetingsHours - codeReviewHours - slackInterruptHours;
// = 5.5 focused hours per engineer per day

// Team capacity for this sprint:
const fullTeamCapacity = sprintDays * engineersOnTeam * focusedHoursPerDay;
// = 10 × 4 × 5.5 = 220 focused hours

// Adjustments:
const ptoAndHolidays = 3;  // engineer-days of PTO this sprint
const capacityReduction = ptoAndHolidays * focusedHoursPerDay;
// = 3 × 5.5 = 16.5 hours reduction

const adjustedCapacity = fullTeamCapacity - capacityReduction;
// = 220 - 16.5 = 203.5 focused hours

// If team uses story points:
// capacity in points = adjustedCapacity / hoursPerPoint
// hoursPerPoint = (previous 3-sprint avg hours) / (previous 3-sprint avg points)
```

---

### Step 2: Account for Overhead

```
Reserve capacity for recurring obligations:
- Tech debt payoff: 15-20% of sprint (prevent accumulation)
- Bug fixes (not pre-identified): ~10% (always some incoming)
- Code review + helping others: already in per-engineer calc above
- Sprint ceremonies (planning, retro, demo): 1-2 hours per person, subtract from first/last days

Committed capacity = adjustedCapacity × 0.75 (leave 25% buffer for overhead + unknowns)
```

---

### Step 3: Map Dependencies

Before committing to sprint scope, check:
- Does any story depend on another team? (API change, data migration, infra work)
- Does any story depend on an external party? (vendor API access, design mockups)
- Are stories in the right sequence? (story B requires story A to be merged first)

```
Dependency map example:
Story A: Add MFA backend      -> no dependencies
Story B: Add MFA frontend     -> depends on Story A (needs API)
Story C: Enforce MFA policy   -> depends on Story B (needs full flow working)
Story D: MFA recovery codes   -> can start in parallel with B (independent)

Sprint risk: if Story A slips 3 days, Story B can't start -> Story C can't start
Mitigation: start Story A on day 1, Story D can parallelize
```

---

### Step 4: Risk Identification

For each story, flag risks:
- **Unclear requirements** — needs product clarification before starting
- **External dependency** — blocked by another team, vendor, or design
- **High complexity** — first time using technology, large change
- **Integration risk** — changes a shared component many services use
- **Data risk** — involves a migration or schema change

Scale risk: Low / Medium / High. High risk stories get explicit mitigation plans.

---

### Step 5: Commitment Heuristics

```
Green: commit if:
- All requirements clear
- No blocking dependencies
- Team has done similar work before
- Estimate has low variance (High - Low < 2x Low)

Yellow: commit with caveat if:
- Requirements mostly clear (1-2 open questions resolvable this sprint)
- Dependency exists but blocked team is aware and has committed to unblock on day 1
- Medium complexity with mitigation plan

Red: don't commit, put in backlog:
- Requirements unclear (still in product discovery)
- Blocking external dependency not confirmed
- Estimate is speculative (never done this before, no spike done)
- Story size > half the sprint capacity on its own

Rule of thumb: don't commit to more than 80% of capacity
The 20% is absorbed by: scope creep, unexpected bugs, review iterations, meetings that run long
```

---

### Step 6: Sprint Goal

Before locking the sprint, write the sprint goal:

```
Bad sprint goal: "Complete stories A, B, C, D, E, and F"
(this is a task list, not a goal)

Good sprint goal: "Users can register, log in, and enable two-factor authentication by end of sprint"
(this is user value, testable, guides priority if things slip)

If a story doesn't contribute to the sprint goal, question whether it should be in the sprint.
```

---

### Output Format

```
## Sprint Plan: Sprint [N] ([Date range])

### Capacity
| Engineer | Available Days | PTO/Holidays | Focused Hours |
|----------|---------------|-------------|--------------|
| Total | | | X hours |

Committed capacity: [Y hours after overhead buffer]

### Sprint Goal
[One sentence describing what users can do at end of sprint]

### Committed Stories

| Story | Points/Days | Owner | Dependencies | Risk |
|-------|------------|-------|-------------|------|

**Total commitment:** [X points / Y days]
**Capacity utilization:** [Z% of adjusted capacity]

### Dependency Map
[Which stories block which]

### Risks
| Risk | Story | Probability | Mitigation |

### Not in Sprint (Next Sprint Candidates)
[Stories considered but deferred, with reason]
```
