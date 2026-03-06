---
description: Identify code smells with specific named patterns and fixes, then suggest design patterns if applicable
argument-hint: "<paste code>"
---

# /smell-check -- Code Smell Detection

Identify named code smells and prescribe targeted fixes. Chains code-smells -> design-patterns.

## Invocation

```
/smell-check [paste code]
/smell-check                    # asks for code
```

## Workflow

### Step 1: Get the Code

If not provided, ask the user to paste the code they want analyzed.

### Step 2: Smell Detection Pass

Apply **code-smells** skill. Scan for each smell category:

**Bloaters** (code that has grown too large):
- [ ] God Class / God Object
- [ ] Long Method (> 20-30 lines as a rough guide)
- [ ] Long Parameter List (4+ parameters)
- [ ] Data Clumps (same group of variables together repeatedly)
- [ ] Primitive Obsession (domain concepts as raw primitives)

**Object-Orientation Abusers:**
- [ ] Switch Statements (replace with polymorphism)
- [ ] Temporary Field (field only set for certain operations)
- [ ] Refused Bequest (subclass doesn't use parent's interface)

**Change Preventers:**
- [ ] Divergent Change (class changes for many different reasons)
- [ ] Shotgun Surgery (one change requires edits in many files)
- [ ] Parallel Inheritance Hierarchies (adding a class in one hierarchy requires adding in another)

**Dispensables:**
- [ ] Dead Code (unreachable or never-called)
- [ ] Speculative Generality (YAGNI violation — designed for hypothetical future)
- [ ] Data Class (only getters/setters, no behavior)
- [ ] Duplicate Code (same logic in multiple places)
- [ ] Lazy Class (class that does too little to justify existence)

**Couplers:**
- [ ] Feature Envy (method uses another object's data more than its own)
- [ ] Inappropriate Intimacy (class knows too much about another's internals)
- [ ] Message Chains (a.b().c().d() — Law of Demeter violation)
- [ ] Middle Man (class that just delegates everything — remove the indirection)

### Step 3: Root Cause Analysis

For smells that share a root cause, group them:
- "The God Class and Long Methods are both symptoms of SRP violation"
- "The Message Chains and Feature Envy both point to data not living in the right class"

### Step 4: Pattern Recommendation

Apply **design-patterns** skill for smells where a pattern addresses the root:

| Smell | Pattern Fix |
|-------|-------------|
| Switch on type | Strategy pattern |
| Many notification channels | Observer + Strategy |
| Complex object construction | Builder |
| Duplicate algorithms | Template Method |
| Multiple ways to do same thing | Strategy |
| Fragile dependency on implementation | Adapter |

### Step 5: Prioritize Fixes

Rank by impact on maintainability:
1. Smells that cause the most frequent bugs or test failures
2. Smells in the most-changed files (high churn = high pain)
3. Smells blocking new feature development
4. Aesthetic smells (fix when you're in the area anyway)

## Output

```
## Code Smell Report: [File/Module]

### Smells Found

| Smell | Location | Severity | Refactoring |
|-------|----------|----------|-------------|
| God Class | UserService | High | Extract Class |
| Long Parameter List | createReport():12 | Medium | Introduce Parameter Object |
| Magic Numbers | validateDiscount():45 | Low | Replace Magic Number |

### Detailed Findings

#### God Class — UserService (High)
Symptom: 450 lines, handles auth, profile, notifications, billing
Problem: Every feature change touches this file; impossible to test in isolation
Fix: Extract AuthService, UserProfileService, NotificationService
Refactoring: "Extract Class" (Fowler)

#### Feature Envy — OrderProcessor.calculateDiscount() (Medium)
Symptom: Uses order.customer.subscriptionTier, order.items.length, order.subtotal
Problem: This method belongs on Order, not OrderProcessor
Fix: Move method to Order class
Refactoring: "Move Method" (Fowler)

### Quick Wins (< 30 minutes each)
[Low-effort, high-value fixes]

### Requires Careful Planning
[Large refactors that need a test safety net and incremental approach]

### Patterns That Could Help
[Specific GoF patterns for identified smells, with brief rationale]
```

## Next Steps

- "Want a full refactoring plan? -> `/refactor [code]`"
- "Want a code review? -> `/review-code [code]`"
- "Want SOLID analysis? -> ask about solid-principles"
