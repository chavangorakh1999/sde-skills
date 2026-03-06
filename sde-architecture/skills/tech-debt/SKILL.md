---
name: tech-debt
description: "Classify debt using Fowler matrix (reckless/deliberate/inadvertent/prudent), payoff analysis, priority scoring, tracking strategy. Use when deciding what tech debt to pay off and in what order."
---

## Tech Debt Management

Tech debt is not inherently bad — some debt is a rational business decision. The problem is untracked, un-quantified debt that silently slows teams down.

### Context

Tech debt to analyze or classify: **$ARGUMENTS**

---

### Fowler's Tech Debt Matrix

Not all debt is equal. Classify first.

```
               RECKLESS              |  PRUDENT
               "We don't have time   |  "We must ship now and deal
               for design"           |  with it later"
    ─────────────────────────────────────────────────
    DELIBERATE  High risk: you        |  Intentional tradeoff with
                chose speed but       |  a plan to pay it back
                knew the cost         |  (document it as an ADR)
    ─────────────────────────────────────────────────
    INADVERTENT "What's layering?"    |  "Now we know how we
                (incompetence)        |  should have done it"
                Low risk that this    |  (normal — knowledge
                will ever be paid off |  gained in development)
```

**Reckless + Deliberate:** "We don't have time, let's just copy/paste it." — Highest risk. No plan. Will compound.

**Reckless + Inadvertent:** Team didn't know better practices. Not malicious, but needs training + refactoring.

**Prudent + Deliberate:** "We know the right way but we're shipping fast. We'll refactor in Q2." — Acceptable IF you actually schedule Q2.

**Prudent + Inadvertent:** You built the best way you knew at the time. As you learned more, you realized a better approach. Normal part of software development.

---

### Debt Inventory and Scoring

```javascript
// Score each debt item on three dimensions (1-5 each):

const debtItem = {
  title: 'UserService is 1,200 lines handling 15 concerns',
  category: 'design',  // design | performance | security | infrastructure | test-coverage
  type: 'prudent-inadvertent',

  // Impact: how much does this slow us down or create bugs?
  impact: 4,  // 1=trivial, 3=moderate slowdown, 5=blocks feature development

  // Probability: how often do engineers interact with this code?
  churnRate: 5,  // 1=rarely touched, 5=touched every sprint

  // Cost to fix: estimate in days of engineer time
  fixCost: 10,  // 10 engineer-days to extract to 3 services

  // Risk of NOT fixing: what happens if we leave it?
  riskIfIgnored: 'Every feature involving users takes 2x longer; new engineers ramp 4x slower',

  // Interest rate: is this getting worse over time?
  interest: 'High — every new feature adds more code to UserService',
};

// Priority score = (impact × churnRate) / fixCost
// Higher = fix sooner (high pain, frequently touched, cheap to fix)
const priorityScore = (debtItem.impact * debtItem.churnRate) / debtItem.fixCost;
// = (4 × 5) / 10 = 2.0

// Debt quadrant:
// High impact + Low fix cost = Fix NOW (quick wins)
// High impact + High fix cost = Plan for major refactor sprint
// Low impact + Low fix cost = Fix opportunistically (when nearby)
// Low impact + High fix cost = Accept / defer / delete (never fix)
```

---

### Debt Payoff Analysis

Before committing to pay off debt, calculate the return on investment:

```
Debt: Missing indexes on orders table
Current cost:
  - 5 engineers run the orders list page in testing every day
  - Page takes 3 seconds without indexes (400ms with)
  - Time wasted per day: 5 engineers × 5 page loads × 2.6 extra seconds = 65 seconds/day
  - Annualized: 65s × 250 workdays = ~4.5 hours/year (negligible for engineers)
  - But: customer-facing queries also slow -> P99 = 4s, dropping conversion rate ~0.5%
  - Revenue impact: $50K ARR × 0.005 = $250/month

Fix cost:
  - Write migration to add 3 indexes: 2 hours engineer time
  - Deploy and verify: 1 hour

ROI: $250/month benefit, 3 hours cost -> payoff in < 1 day
Decision: Fix immediately.

---

Debt: Upgrade from Node.js 16 to 22
Current cost:
  - Node 16 end of life, security patches stop
  - Risk of unpatched CVE increases over time
  - Upgrade itself is medium-risk (potential breaking changes in dependencies)

Fix cost:
  - Upgrade node version: 1 day
  - Fix breaking changes in dependencies: 2-5 days (unknown until done)
  - Test: 2 days
  Total: 5-8 engineer-days

Decision: Schedule in next sprint. Security risk justifies the cost.
```

---

### Tech Debt Tracking

```markdown
// Example debt register (GitHub Project, Notion, Jira — pick one place)
// Key: visibility is more important than the tool

| ID | Title | Type | Priority Score | Fix Cost | Owner | Target Quarter |
|----|-------|------|---------------|----------|-------|----------------|
| TD-001 | UserService god class | design | 2.0 | 10d | @alice | Q3 2024 |
| TD-002 | Missing pg indexes | performance | 12.0 | 0.5d | @bob | Now |
| TD-003 | Auth uses MD5 hashing | security | 15.0 | 1d | @alice | Now |
| TD-004 | No error monitoring | observability | 8.0 | 3d | @carol | Q2 2024 |

// Rules for a healthy debt register:
// 1. Every item has an owner (not "the team" — a person)
// 2. Every item has a target quarter (or "Accepted — Won't Fix" with reason)
// 3. Review quarterly: close completed items, add new ones, reprioritize
// 4. Debt budget: allocate 20% of sprint capacity to debt payoff (prevents it from growing)
```

---

### The 20% Rule

Many high-performing teams allocate 20% of each sprint to tech debt:

```
Sprint capacity: 100 points
Product features: 80 points
Tech debt: 20 points (pick from the top of the priority-scored list)

Why this works:
- Debt doesn't compound infinitely (you're paying interest continuously)
- Product still moves fast (80% on features)
- Engineers stay motivated (not perpetually drowning in legacy code)
- Sustainable: no "big refactor quarter" that never ships product

When teams DON'T do this:
- Debt accumulates -> velocity slows -> pressure to skip tests, hardcode, cut corners
- Eventually: no new features possible without a 6-month rewrite
```

---

### Output Format

```
## Tech Debt Analysis: [System/Component]

### Debt Inventory

| Item | Type | Impact | Churn | Fix Cost | Priority Score | Recommendation |

### Detailed Findings

#### [Item Name] — Priority: X
Type: [Fowler classification]
Description: [What the debt is]
Current cost: [How it slows the team, bugs it causes, security risk]
Fix: [What the refactoring involves]
Fix cost: [Estimate in engineer-days]
ROI: [Break-even analysis]

### Prioritized Payoff Plan
1. Fix now (< 1 day, high impact)
2. Fix next sprint
3. Plan for major refactor

### Debt Budget Recommendation
[What % of sprint capacity to allocate to debt payoff]

### Won't Fix (Accepted Debt)
[Items that aren't worth fixing — with rationale]
```
