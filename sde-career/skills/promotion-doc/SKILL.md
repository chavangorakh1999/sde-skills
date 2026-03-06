---
name: promotion-doc
description: "Write a promotion document or self-assessment for a software engineer: impact evidence, scope demonstration, leadership examples, and framing for the target level. Use when writing or reviewing a promo doc."
---

## Promotion Document

### Context

Current level, target level, and key projects to highlight: **$ARGUMENTS**

---

### Promotion Philosophy

```
Promotions are backward-looking evidence, not future promises.
You are promoted for work you have ALREADY done at the next level.

The doc proves: "I have been consistently operating at [target level] for X months."

For most companies:
  L3 → L4: execute well on defined tasks
  L4 → L5: own a feature/project end-to-end, influence team decisions
  L5 → L6: org-wide impact, multiply others, drive ambiguous problems
  L6+ → Staff: company-level impact, define technical direction
```

---

### Document Structure

```markdown
# Promotion Document: [Name] — [Current Level] → [Target Level]
### Submitted: [Date] | Manager: [Name]

## Summary (3-5 sentences)
High-level: who you are, your role, and the case for promotion in one paragraph.
"Over the past 18 months, I have consistently operated at the [target] level by
[summary of scope]. My most significant contributions include [2-3 highlights]."

## Impact at [Target Level]

### Project 1: [Project Name]
**Context:** What was the problem and why did it matter?
**My role:** Was I the lead? A key contributor? What did I own specifically?
**Actions:** What did I do? (technical decisions, cross-team work, leadership)
**Outcome:** Measurable result. (latency, revenue, reliability, team velocity)
**Evidence of [target level] scope:** [explicit mapping to level criteria]

### Project 2: [Project Name]
[same structure]

### Project 3: [Project Name]
[same structure]

## Scope and Influence
Demonstrate breadth beyond individual execution:
- Technical decisions that affected the team/org
- Cross-functional work (PM, design, data science, other teams)
- Other engineers helped, mentored, unblocked

## Collaboration and Leadership
Evidence of working beyond your immediate team:
- RFCs written and how they were received
- Design reviews led
- Oncall improvements that helped the whole team
- Recruiting, interviewing, onboarding contributions

## Growth and Learning
Optional — show how you've grown into the next level:
- Skills developed
- Feedback acted on
- Areas you're still working on (honesty builds trust)
```

---

### Evidence Mining

```
For each major project from the last 12-18 months, extract:

Technical scope:
  - Did you design the architecture or follow one given to you?
  - What were the hardest technical decisions? What were the trade-offs?
  - What would have broken without your involvement?

Business impact:
  - Revenue: did it unlock revenue, prevent loss, enable a product?
  - Reliability: latency improvement? Reduced incidents?
  - Scale: users served, QPS handled?
  - Developer productivity: faster deploys, better tooling?

Cross-team impact:
  - How many teams depended on your work?
  - Did other teams adopt your API, library, or pattern?
  - Did you remove blockers for others?

Leadership:
  - Did junior engineers come to you for help?
  - Did you run design reviews or RFC processes?
  - Did you define the approach, or implement someone else's?
```

---

### Level Differentiation

```
L4 → L5 (senior) evidence:
  - Owned a project from design through launch (not just implementation)
  - Made significant technical decisions without being told what to do
  - Collaborated effectively across team boundaries
  - Code/designs improved quality bar of the team
  - Can be counted on for the hardest tickets

L5 → L6 (staff) evidence:
  - Defined the technical direction for a team-level initiative
  - Work had org-wide adoption or impact (not just your team)
  - Unblocked multiple teams, not just your own
  - Identified and addressed problems before they were defined as problems
  - Mentored multiple engineers to growth
  - Drove ambiguous problems to clarity and execution

Common mistakes:
  - Impact stated without context ("reduced latency by 40%" — 40% of what? 1s → 600ms or 100ms → 60ms?)
  - "We" not "I" — committee language makes your contribution invisible
  - Technical jargon without business impact
  - Only one big project — breadth matters
  - Missing the senior-scoped dimension (anyone can execute; what did you define/lead?)
```

---

### Writing Tips

```
Use numbers everywhere possible:
  "improved latency" → "reduced p99 latency from 1.8s to 340ms"
  "helped the team" → "led 3 design reviews, mentored 2 engineers through their first production launches"
  "cross-team project" → "coordinated with 4 teams across 2 orgs"

Write to the level criteria:
  Pull out the actual leveling rubric and map each project to it explicitly.
  "This demonstrates [Senior criteria: 'sets technical direction for the team'] because..."

Get peer quotes:
  Ask 2-3 colleagues for 2-3 sentence written quotes about your impact.
  These add credibility and different perspectives.
  "Would you be willing to write a 2-sentence quote about the work we did on X?"
```
