# SDE Skills Marketplace

> 80+ SDE skills and 40+ chained workflows across 9 plugins — system design, code quality, architecture, debugging, execution, MERN stack, testing, DevOps, and career growth. JavaScript-first.

Designed for Claude Code and Cowork. Skills compatible with other AI assistants.

---

## Start Here

Building a new system? -> `/design`
Reviewing code or a PR? -> `/review-code`
Writing an architecture decision? -> `/write-adr`
Building a MERN feature? -> `/mern-feature`
Debugging a production issue? -> `/postmortem`
Breaking an epic into tickets? -> `/write-tickets`
Prepping for an interview? -> `/mock-design`

---

## Installation

### Claude Code (CLI)
```bash
claude plugin marketplace add yourusername/sde-skills
```

### Other AI assistants (skills only)
```bash
for plugin in sde-*/; do
  cp -r "$plugin/skills/"* ~/.gemini/skills/ 2>/dev/null
done
```

---

## Plugins

| Plugin | Skills | Commands | Domain |
|--------|--------|----------|--------|
| `sde-system-design` | 8 | 4 | HLD, estimation, data modeling, API design |
| `sde-code-quality` | 8 | 4 | PR review, refactoring, SOLID, patterns |
| `sde-architecture` | 7 | 4 | ADRs, migrations, resilience, observability |
| `sde-debugging` | 6 | 4 | RCA, postmortems, logs, incident response |
| `sde-execution` | 7 | 5 | Tickets, estimation, deployment, runbooks |
| `mern-stack` | 12 | 6 | MongoDB, Express, React, Node.js, JWT |
| `sde-testing` | 6 | 3 | Unit, integration, E2E, TDD, contracts |
| `sde-devops` | 6 | 3 | CI/CD, Docker, Kubernetes, Terraform |
| `sde-career` | 8 | 5 | Interviews, promotion, specs, feedback |

---

## Quick Reference

| I want to... | Command |
|---|---|
| Design a system end-to-end | `/design [system]` |
| Estimate capacity and scale | `/estimate [system]` |
| Design a REST API contract | `/design-api [resource]` |
| Design a data model | `/model-data [domain]` |
| Review my PR before submitting | `/review-code [diff]` |
| Refactor messy code safely | `/refactor [code]` |
| Spot and fix code smells | `/smell-check [code]` |
| Plan tests for a feature | `/test-strategy [feature]` |
| Write an architecture decision | `/write-adr [decision]` |
| Plan a safe migration | `/plan-migration [from to]` |
| Write a blameless postmortem | `/postmortem [incident]` |
| Debug a production issue | `/debug [symptom]` |
| Respond to a live incident | `/incident [description]` |
| Break an epic into tickets | `/write-tickets [epic]` |
| Write release notes | `/release-notes [commits]` |
| Create a deployment checklist | `/deploy-plan [service]` |
| Write an on-call runbook | `/write-runbook [service]` |
| Build a full MERN feature | `/mern-feature [feature]` |
| Build JWT auth (MERN) | `/mern-auth` |
| Design a MongoDB schema | `/mern-schema [domain]` |
| Build an Express REST API | `/mern-api [resource]` |
| Build a React component | `/mern-component [component]` |
| Review MERN code | `/mern-review [code]` |
| Plan a full test suite | `/test-plan [feature]` |
| Generate tests for my code | `/write-tests [code]` |
| Design a CI/CD pipeline | `/design-pipeline [service]` |
| Dockerize a service | `/dockerize [service]` |
| Write Terraform IaC | `/write-iac [infrastructure]` |
| Practice system design | `/mock-design [system]` |
| Practice behavioral interview | `/mock-behavioral` |
| Write a promotion document | `/write-promo [level]` |
| Write a technical spec | `/write-spec [feature]` |
| Review my resume | `/review-resume` |

---

## Core Engineering Rules

These rules are encoded into every skill and command output.

1. **Think Before You Code** - Reason through requirements, constraints, and tradeoffs first
2. **Production Mindset** - Error handling, input validation, security, observability by default
3. **Tradeoffs Are Mandatory** - Why this approach, what it costs, alternatives considered
4. **Quantify Everything** - O(n^2) not "slow", "hits pool limit at 1K QPS" not "won't scale"
5. **Security Is Non-Negotiable** - Flag injection, secrets, missing auth, timing attacks
6. **Explicit Over Clever** - Readable names, justified complexity, comment intent not mechanics
7. **No Hallucinated APIs** - Note "verify syntax" when uncertain, use pseudocode
8. **Fail Loudly, Recover Gracefully** - Typed errors, circuit breakers, dead letter queues
9. **Blameless by Default** - Fix systems and processes, not people
10. **Suggest the Next Step** - After every command, offer the natural follow-on

---

## License

MIT
