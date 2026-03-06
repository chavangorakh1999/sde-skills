---
name: resume-review
description: "Review and improve a software engineer resume: impact metrics, bullet point strength, skills section, ATS optimization, and targeting a specific role. Use when reviewing or writing your own resume."
---

## Resume Review

### Context

Resume content or target role: **$ARGUMENTS**

---

### The SWE Resume Framework

```
Format: 1 page (< 5 years), 2 pages max (10+ years)
Order:  Contact → Summary (optional) → Experience → Skills → Education → Projects
Goal:   Pass the 30-second skim AND the detailed read

Recruiter scan order:
1. Current/recent title and company
2. Bullet impact numbers
3. Skills/tech stack
4. Education
```

---

### Impact-Driven Bullet Points

```
Formula: [Strong verb] + [what you did] + [measurable result]

WEAK bullets (no impact):
- "Worked on the recommendation engine"
- "Fixed bugs in the payment service"
- "Responsible for backend development"

STRONG bullets (verb + result):
- "Redesigned recommendation engine query pattern, reducing P99 latency from 2.3s → 340ms"
- "Identified and fixed a race condition in payment processing that caused $50K/month in duplicate charges"
- "Led migration of monolith to 5 microservices, enabling independent deployments and 40% faster CI runs"
- "Implemented Redis caching layer reducing MongoDB load by 60% and cutting infrastructure cost $3K/month"

Strong verbs by category:
  Built/shipped:    Architected, Built, Designed, Implemented, Shipped, Launched, Developed
  Improved:         Optimized, Reduced, Improved, Accelerated, Streamlined, Automated
  Led:              Led, Mentored, Managed, Coordinated, Drove, Championed
  Analyzed:         Investigated, Debugged, Diagnosed, Identified, Discovered
  Saved:            Reduced, Eliminated, Saved, Cut, Prevented
```

---

### Quantification Guide

```
If you don't have exact numbers, estimate with confidence:
  Latency: "reduced average latency from ~800ms to ~200ms"
  Scale:   "service handling 50K+ requests/day"
  Cost:    "reduced infrastructure cost by ~$2K/month"
  Team:    "collaborated with 6-person team"
  Coverage:"increased test coverage from 40% to 85%"
  Time:    "reduced deploy time from 45 minutes to 8 minutes"

Types of metrics to include:
  - Performance: latency, throughput, error rate
  - Scale: users, requests/sec, data volume
  - Business: revenue, cost, conversion rate
  - Developer experience: build time, deploy frequency, time-to-merge
  - Reliability: uptime improvement, incidents reduced
```

---

### Skills Section

```
Group by category, list most relevant to target role first:

Languages:    JavaScript/TypeScript, Python, Go, Rust
Backend:      Node.js, Express.js, NestJS, GraphQL
Frontend:     React, Next.js, TailwindCSS
Databases:    PostgreSQL, MongoDB, Redis, Elasticsearch
Cloud/DevOps: AWS (ECS, RDS, Lambda), Docker, Kubernetes, Terraform, GitHub Actions
Testing:      Jest, Playwright, Pytest

What NOT to include:
- Microsoft Office, HTML (too basic)
- Outdated tech: jQuery, PHP 5 (if you're targeting modern roles)
- Skills you couldn't be interviewed on (Kubernetes if you've only touched it once)
```

---

### ATS Optimization

```
Applicant Tracking Systems scan for keyword matches.

1. Mirror language from the job description:
   JD says "Node.js" → use "Node.js" (not "NodeJS" or "node")
   JD says "CI/CD pipelines" → use that exact phrase

2. Use a clean, parseable format:
   - Single column preferred
   - No tables, text boxes, or graphics (ATS can't parse these)
   - Standard section headers: "Experience", "Skills", "Education"
   - PDF format (preserves formatting) or plain text

3. Include all relevant technologies in bullets AND skills section

4. Check ATS score: jobscan.co or similar tools
```

---

### Review Rubric

```
## Resume Review: [Name / Role]

### Impact (1-5)
Bullet strength, quantification, specificity

### Relevance (1-5)
Match between experience and target role

### Clarity (1-5)
Readability, conciseness, grammar

### Technical Depth (1-5)
Shows engineering skills, architecture decisions, scale

### ATS Readiness (1-5)
Format, keywords, standard headers

---

### Rewrites
For each weak bullet:
BEFORE: [original]
AFTER:  [improved version with impact]

### Top 3 Priorities
1. [Most impactful change]
2. [Second priority]
3. [Third priority]
```
