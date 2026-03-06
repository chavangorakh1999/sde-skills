---
description: "Run a mock system design interview with feedback on structure, depth, and communication"
argument-hint: "[system to design, e.g. 'design a URL shortener' or 'design Twitter's feed']"
---

## Mock System Design Interview

Run a full 45-minute mock system design interview with structured feedback.

### Interview Format

Act as the interviewer. Present the problem and guide the candidate through the session:

```
Problem: Design [ARGUMENTS]

Rules:
- I'll ask questions and probe like a real interviewer
- You'll work through the design in real-time
- I'll give hints if you get stuck (note when I do)
- At the end, I'll give detailed feedback
```

### Phase 1 — Problem Presentation (0-2 min)

Present the problem with minimal context (realistic conditions):
"Design [system]. You have 45 minutes."

Wait for the candidate to start clarifying requirements.

### Phase 2 — Guided Interview (2-43 min)

Guide through all phases, probing at each:

**Requirements (0-8 min):**
- Do they clarify functional requirements?
- Do they ask about scale (DAU, QPS)?
- Do they identify non-functional requirements (latency, availability)?
- Probe: "How many users? What's the read/write ratio?"

**Estimation (8-12 min):**
- Do they estimate QPS and storage?
- Is the math reasonable?
- Probe: "How much storage will you need for 1 year?"

**High-level design (12-22 min):**
- Do they draw a complete system (client → LB → servers → cache → DB)?
- Do they identify the core data flow?
- Probe: "What happens when a user tries to create a short URL that already exists?"

**Deep dive (22-38 min):**
- Do they go deep on 1-2 components?
- Do they discuss trade-offs, not just solutions?
- Do they handle failure modes?
- Probe: "What happens if the cache goes down?" / "How does this scale to 10x?"

**Scale and bottlenecks (38-43 min):**
- Do they identify the bottleneck?
- Can they propose a reasonable scaling strategy?

### Phase 3 — Feedback (43-45 min)

Provide detailed feedback in this format:

```
## System Design Interview Feedback: [System]

### Overall Score
Requirements:         X/10 — [comment]
Estimation:           X/10 — [comment]
High-Level Design:    X/10 — [comment]
Deep Dive:            X/10 — [comment]
Trade-Off Analysis:   X/10 — [comment]
Communication:        X/10 — [comment]
Overall:              X/10

### Strengths
- [What was done well with specifics]

### Critical Gaps
- [Missing component or concept — explain why it matters]

### Trade-Offs to Know
[The key trade-offs for this specific system that were missed]

### What Would Pass vs. Fail
Pass: [specific criteria]
Fail: [what would cause rejection]

### Study Recommendations
- [Specific concept to review]
- [Comparable system to study]
```
