---
description: Three-point estimate with confidence intervals, risk flags, and sprint planning implications
argument-hint: "<scope description>"
---

# /estimate -- Engineering Effort Estimation

Generate a structured estimate with three-point range, risks, and capacity implications. Chains estimation -> sprint-planning.

## Invocation

```
/estimate Migrate from REST to GraphQL for our public API — Node.js backend
/estimate Implement full-text search with Elasticsearch — currently using LIKE queries
/estimate Add WebSocket real-time notifications to existing Express app
/estimate                    # asks for scope description
```

## Workflow

### Step 1: Understand the Scope

Clarify before estimating:
- What specifically needs to be built? (ask if vague)
- What's already done? (existing code to build on vs. greenfield)
- What are the quality requirements? (tests required? performance target?)
- Are there dependencies? (another team's API, design mockups, data migration?)
- Is this being done by one engineer or a team?
- What's the team's experience with the technologies involved?

If scope is unclear: don't estimate, clarify first. "A rough estimate on unclear scope is a precise answer to the wrong question."

### Step 2: Break Into Components

Decompose the work into estimable chunks:
- Database changes
- Backend API changes
- Frontend changes
- Testing
- Infrastructure/DevOps
- Documentation

Each component gets its own three-point estimate.

### Step 3: Apply Three-Point Estimation

Apply **estimation** skill for each component:
```
L = Low (optimistic — everything goes right)
M = Mid (expected — typical friction and discoveries)
H = High (pessimistic — significant unexpected complexity)

PERT = (L + 4M + H) / 6
```

### Step 4: Risk-Adjusted Total

Apply risk multipliers:
- First time with this technology: +25-50%
- Undocumented external API: +25-50%
- Complex legacy code: +50-100%
- No existing tests: +25%
- Ambiguous requirements: don't estimate — clarify first

### Step 5: Sprint Implications

Apply **sprint-planning** capacity math:
- How many engineers? What's their sprint capacity?
- Can this be done in one sprint or does it span multiple?
- What's the dependency sequence? (can it be parallelized?)

### Step 6: Confidence Level

State your confidence in the estimate:
- High confidence (similar work done before, clear requirements): PERT ± 10%
- Medium confidence (some unknowns): PERT ± 25-50%
- Low confidence (significant unknowns): needs a spike first

A low-confidence estimate should trigger a spike (1-2 day investigation before committing).

## Output

```
## Estimation: [Feature/Project]

### Summary

Task: [Description]
Low  (optimistic):   X days
Mid  (expected):     Y days  <- commit to this
High (pessimistic):  Z days
PERT estimate:       W days

### Component Breakdown

| Component | L | M | H | PERT | Risk Factors |
|-----------|---|---|---|------|-------------|
| Database migration | 1 | 2 | 4 | 2.2 | First time with zero-downtime migration |
| API changes | 2 | 4 | 6 | 4.0 | — |
| Frontend | 1 | 3 | 5 | 3.0 | Unfamiliar with new component library |
| Tests | 1 | 2 | 3 | 2.0 | — |
| **Total** | 5 | 11 | 18 | **11.2** | |

### Risk Register

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|

### Sprint Planning Implications

With [N] engineers at [X hours/day focused]:
- Fits in 1 sprint: [Yes/No]
- Spans [N] sprints
- Can parallelize: [what can run in parallel]

### Confidence Level: [High / Medium / Low]
[Reason — what would increase confidence]

### Recommendation

[ ] Commit to [Y days] with the risks above
[ ] Run a spike first to reduce uncertainty on [specific unknown]
[ ] Clarify requirements before estimating: [open questions]

### Assumptions

[List of assumptions — if any are wrong, the estimate changes significantly]
```

## Next Steps

- "Want to break this into tickets? -> `/write-tickets [epic]`"
- "Should I plan the sprint? -> ask about sprint-planning"
