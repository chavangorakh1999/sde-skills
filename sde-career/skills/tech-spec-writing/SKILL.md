---
name: tech-spec-writing
description: "Write a technical specification (design doc, RFC): problem statement, requirements, design options, trade-offs, and decision. Use when starting a significant technical project."
---

## Technical Spec Writing

### Context

Project or system to spec: **$ARGUMENTS**

---

### When to Write a Tech Spec

```
WRITE a spec when:
  - More than one engineer will work on it
  - It will take > 1 sprint
  - It involves API/data contract changes that affect other teams
  - There are meaningful trade-offs between approaches
  - You're changing something architectural

SKIP the spec when:
  - It's a well-understood bug fix
  - The approach is obvious and uncontroversial
  - You're the only implementer and the change is contained
  - A short Slack message is sufficient
```

---

### Tech Spec Template

```markdown
# [Feature/System Name] Technical Specification

**Author:** [Name]
**Status:** Draft | In Review | Approved | Implemented
**Created:** [Date]
**Last Updated:** [Date]
**Stakeholders:** [PM: Name, EM: Name, SRE: Name]
**Review Deadline:** [Date]

---

## Problem Statement

What is broken, missing, or needs to change? Why does it matter now?
Include: customer impact, business motivation, technical debt being addressed.

Keep this to 2-3 paragraphs. If you can't explain the problem clearly, the design won't be clear either.

## Goals and Non-Goals

### Goals
- [ ] [Specific, measurable outcome the spec will achieve]
- [ ] [Another goal]

### Non-Goals (explicitly out of scope)
- [Thing that might seem related but isn't in scope]
- [Future improvement deliberately deferred]

Non-goals are as important as goals — they prevent scope creep.

## Requirements

### Functional
- The system MUST [required behavior]
- The system SHOULD [preferred behavior]
- The system MAY [optional behavior]

### Non-Functional
- Latency: [p99 read/write SLA]
- Throughput: [expected QPS at launch and 1 year]
- Availability: [99.9%, 99.99%?]
- Data retention: [how long is data kept?]

## Design

### Overview
2-4 sentence summary of the approach.

### Architecture Diagram
[ASCII or embedded image — show components and data flow]

### Data Model
Tables/collections/schemas with field types and indexes.
Explain any non-obvious design choices.

### API Design
Endpoints, request/response schemas, error codes.
Follow existing API conventions.

### Detailed Design

#### [Component 1]
How it works. Key algorithms or state machines.

#### [Component 2]
Dependencies, failure modes.

## Alternatives Considered

### Option A: [Current proposal]
**Pros:** [Benefits]
**Cons:** [Drawbacks]

### Option B: [Alternative approach]
**Pros:** [Benefits]
**Cons:** [Drawbacks, why not chosen]

### Option C: [Another alternative]
**Pros:**
**Cons:**

**Decision:** Chose Option A because [the specific reasons that outweigh the cons].

## Implementation Plan

### Phase 1: [Name] — [Sprint/Week estimate]
- [ ] Task 1
- [ ] Task 2

### Phase 2: [Name]
- [ ] Task 3

### Rollout Plan
How will this be deployed?
- Feature flagged? What's the rollout percentage ladder?
- Dark launch period?
- Canary before full rollout?
- Rollback plan?

## Testing Plan
- Unit tests for [components]
- Integration tests for [interactions]
- Load testing if performance-sensitive
- How will you validate in production? (metrics to watch)

## Observability
- New metrics to add
- New log fields
- Alerts to configure
- Dashboard link (or plan to create one)

## Security Considerations
- Auth requirements
- Data classification (PII? encrypted at rest/transit?)
- Input validation
- Rate limiting

## Open Questions
- [ ] **[Question]** — Owner: [Name] — Due: [Date]
- [ ] **[Unresolved trade-off]** — need decision from [team/person]

## Decision Log
| Date | Decision | Rationale | Decided By |
|------|----------|-----------|------------|
| [date] | [what was decided] | [why] | [who] |
```

---

### Writing Tips

```
Problem first, solution second:
  Readers who don't understand the problem will challenge your solution.
  Spend 20% of effort on the problem statement — it frames everything.

Alternatives show rigor:
  A spec with no alternatives looks like you didn't think deeply.
  2-3 options with honest trade-offs builds reviewer confidence.

Make decisions explicit:
  Don't end alternatives with "both have trade-offs." Make a recommendation.
  "We will use X because Y." Open questions are fine, but own the call.

Scope your open questions:
  Each open question needs an owner and a due date.
  Otherwise they become permanent uncertainty.

Review asynchronously, decide synchronously:
  Circulate for comments 3-5 days before a review meeting.
  Use the meeting to resolve open questions, not read the doc together.

Calibrate length to risk:
  Simple feature: 1-2 pages
  Complex feature: 3-5 pages
  Platform/API change: 5-10 pages
  If it's longer than 10 pages, you probably need an executive summary.
```
