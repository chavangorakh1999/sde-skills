---
description: Service boundary analysis with bounded context map and migration sequence
argument-hint: "<service or monolith to decompose>"
---

# /decompose -- Service Decomposition

Analyze a monolith or fat service for decomposition opportunities. Identify bounded contexts and propose extraction sequence. Chains service-decomposition -> adr.

## Invocation

```
/decompose User service that handles auth, profile, billing, and notifications
/decompose 200K LOC Rails monolith serving e-commerce platform
/decompose OrderService that calls 8 downstream services and owns 15 tables
/decompose                    # asks for service/monolith description
```

## Workflow

### Step 1: Understand the Current System

Ask for or extract:
- What does the service/monolith currently do? (list major features)
- How many engineers work on it?
- What's the deployment frequency? (and what blocks it?)
- What are the most painful points? (slow deploys, merge conflicts, bugs in unrelated areas)
- What's the team structure? (Conway's Law: architecture follows organization)

### Step 2: Make the "Should We Decompose?" Decision

Apply **service-decomposition** skill:
- List valid technical reasons to decompose (independent scaling, team autonomy, fault isolation)
- Challenge invalid reasons ("microservices are modern")
- Give an honest recommendation: decompose, clean up first, or leave it

Don't skip this. Bad decompositions are worse than monoliths.

### Step 3: Identify Bounded Contexts

For each potential bounded context:
- What's the business domain?
- What's the unique "ubiquitous language" (domain vocabulary)?
- What data does it own?
- What team would own it?

Draw the context map:
```
[Context A] --depends on--> [Context B]
[Context A] <--publishes events to-- [Context C]
```

### Step 4: Analyze the Coupling

For each boundary:
- What is the current coupling? (shared tables? direct function calls? API calls?)
- Is there a clean interface already, or must one be designed?
- What's the data overlap? (which tables are owned by which context?)

### Step 5: Propose Extraction Sequence

Apply the dependency order rule:
1. Extract services with no upstream dependencies first (leaf services)
2. Extract services with few consumers next
3. Leave the most central, coupled services for last

For each extraction:
- What seam exists to cut along?
- What's the interface (API or events)?
- How long will it take?
- What's the risk?

### Step 6: Design the Event Contracts

For any events that will cross context boundaries:
- Event type and schema
- Producer and consumers
- Delivery guarantees needed

### Step 7: Write the ADR

After presenting the analysis, offer to write an ADR:
"Want me to write an ADR capturing this decomposition decision? -> `/write-adr [decision]`"

## Output

```
## Decomposition Analysis: [System Name]

### Should We Decompose?
[Recommendation: Yes / Not yet / No — with evidence]

### Bounded Contexts Found

| Context | Domain Language | Data It Owns | Team | Extraction Complexity |

### Context Map
[Diagram showing dependencies between contexts]

### Coupling Analysis
| Boundary | Current Coupling | Interface Needed | Complexity |

### Extraction Sequence

#### 1. Extract [Service Name] (Low complexity — extract first)
Why first: [Reasoning]
Seam: [Where to cut]
Interface: [API or events]
Effort: [Estimate]
Timeline: [When]

#### 2. Extract [Service Name]
...

### Anti-Patterns to Avoid
[Shared database, chatty services, distributed monolith risk for this specific case]

### What NOT to Extract
[Parts of the system that should stay in the monolith — with reasons]

### Recommended Next Steps
1. [Immediate action]
2. [Phase 1 extraction]
3. [Phase 2 extraction]
```

## Next Steps

- "Want to plan the first extraction? -> `/plan-migration [service extraction]`"
- "Want an ADR for this decision? -> `/write-adr [decision]`"
