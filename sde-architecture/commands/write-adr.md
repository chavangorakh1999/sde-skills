---
description: Full Architecture Decision Record with alternatives, consequences, and review trigger
argument-hint: "<decision to document>"
---

# /write-adr -- Architecture Decision Record

Write a complete ADR that the team can understand and revisit years later.

## Invocation

```
/write-adr Use PostgreSQL instead of MongoDB for the orders service
/write-adr Adopt event sourcing for the payments domain
/write-adr Migrate from REST to GraphQL for the public API
/write-adr                    # asks what decision to document
```

## Workflow

### Step 1: Gather Context

Ask for (or extract from arguments):
- What decision is being made?
- What problem or situation led to this decision?
- What were the constraints? (timeline, team expertise, cost, compliance)
- Who was involved in making the decision?
- Has this decision already been made, or is it still being evaluated?

### Step 2: Explore Alternatives

Before writing the ADR, explore what alternatives were considered:
- What are the 2-4 realistic alternatives?
- What are the tradeoffs of each?
- Why was each alternative rejected?

If the user hasn't thought through alternatives, prompt them:
- "What else did you consider?"
- "Why not [obvious alternative]?"

### Step 3: Identify Consequences

For the chosen decision:
**Positive consequences:** What does this enable? What problems does it solve?
**Negative consequences:** What does this make harder? What technical debt does it introduce? What dependencies does it add?
**Risks:** What could go wrong? What's the mitigation?

### Step 4: Define Review Trigger

Every decision should have a condition that triggers re-evaluation:
- A specific metric threshold ("revisit if data volume > 1TB")
- A time checkpoint ("revisit in 6 months")
- An architectural trigger ("revisit if team grows past 15 engineers")
- A technology trigger ("revisit when the library reaches end of life")

### Step 5: Write the ADR

Apply **adr** skill:
- Assign next sequential number (ask user for the next ADR number, or default to ADR-001)
- Use active voice for the decision ("We will use..." not "X was chosen")
- Keep context factual — explain the situation, not the decision
- Be specific about tradeoffs — not "more complex" but "requires saga pattern for cross-service transactions, adding ~1 week of implementation complexity per use case"

### Step 6: Offer Next Steps

- "Should I update the ADR index? -> Share the index file and I'll add this entry"
- "Want to plan the migration? -> `/plan-migration [from -> to]`"
- "Should I design the implementation details? -> `/design [system]`"

## Output

```markdown
# ADR-[NUMBER]: [Title]

**Date:** [today's date]
**Status:** Proposed
**Deciders:** [from user input]
**Tags:** [relevant categories]

## Context

[Situation and forces at play]

## Decision

We will [decision in active voice].

[Brief rationale — 2-3 sentences]

## Consequences

### Positive
- ...

### Negative
- ...

### Risks
- ...

## Alternatives Considered

### [Alternative 1]
[Description, why considered, why rejected]

### [Alternative 2]
...

## Implementation Notes

[Constraints, caveats, or specifics for engineers]

## Review Trigger

[Condition under which this decision should be reconsidered]
```

Then:
```
## Questions for Team Review
1. [Question for reviewers to consider]
2. ...
```
