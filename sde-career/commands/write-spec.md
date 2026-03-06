---
description: "Write a technical specification (design doc or RFC) for a software project"
argument-hint: "[project or feature to spec, with context about the problem and constraints]"
---

## Write Technical Specification

Generate a complete, high-quality technical specification.

### Phase 1 — Problem Clarification

Before writing, clarify the problem space:

**Scope questions:**
- What exactly is broken or missing today?
- Who is affected and how?
- What's the business impact of not solving this?
- What does "done" look like?

**Constraint questions:**
- What's the performance SLA?
- What's the scale (users, QPS, data volume)?
- What systems does this interact with?
- Are there deadlines, budget constraints, or team capacity limits?

**Alternative questions:**
- What approaches have already been considered?
- What would a simpler solution look like?
- What would a more comprehensive solution look like?

### Phase 2 — Design Options

Generate 2-3 distinct approaches:

For each option:
- Describe the approach in 2-3 sentences
- List concrete pros (performance, simplicity, cost, team familiarity)
- List concrete cons (complexity, risk, operational burden)
- Estimate implementation effort (rough sprint estimate)

Make a clear recommendation with explicit reasoning:
"We recommend Option [X] because [specific reasoning addressing the key trade-offs]."

### Phase 3 — Write the Specification

Generate the complete spec using the standard template:

**Required sections:**
1. Problem Statement — clear, concise, motivating
2. Goals and Non-Goals — scope boundaries
3. Requirements — functional (must/should/may) and non-functional
4. Design — architecture diagram (text-based), data model, API design
5. Alternatives Considered — with honest trade-offs
6. Implementation Plan — phased with effort estimates
7. Testing Plan — how correctness and performance will be verified
8. Observability — metrics, logs, alerts to add
9. Security Considerations — auth, data sensitivity, input validation
10. Open Questions — with owner and due date

**Calibrate length to risk:**
- Small feature: 1-2 pages
- Medium feature: 3-5 pages
- Platform/API change: 5-8 pages

### Phase 4 — Review Preparation

Anticipate and address reviewer objections:
- What's the most likely pushback on the design choice?
- What will the SRE/infrastructure team flag?
- What will security review raise?
- What will the PM ask about scope or timeline?

List the open questions that need resolution before the spec can be approved.

### Output

Provide:
1. **Complete technical specification** — ready to circulate for review
2. **Design decision summary** — 3-5 bullets for a brief sync or async summary
3. **Review guidance** — "I'd especially like feedback on sections X and Y"
4. **Stakeholder list** — who should review and what they'll care about
5. **Decision log starter** — first entry documenting the key design choice
