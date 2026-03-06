---
description: "Design a CI/CD pipeline for a project: stages, tooling, deployment strategy, and GitHub Actions configuration"
argument-hint: "[project type and requirements, e.g. 'Node.js API with Docker deployment to AWS ECS']"
---

## CI/CD Pipeline Design

Design a complete, production-ready CI/CD pipeline.

### Phase 1 — Requirements Analysis

Clarify:
- **Tech stack**: languages, frameworks, build tools
- **Deployment target**: AWS ECS, Kubernetes, Heroku, Railway, etc.
- **Branching strategy**: GitFlow, trunk-based, main-only
- **Release frequency**: multiple times/day vs. weekly
- **Compliance needs**: manual approval gates, audit logs

### Phase 2 — Pipeline Stages

Design the stage progression (fast → slow):

```
Stage 1: Code Quality    (~2 min) — lint, format check, type check
Stage 2: Unit Tests      (~3 min) — Jest unit tests with coverage
Stage 3: Integration     (~5 min) — API tests with real DB (in-memory)
Stage 4: Build           (~5 min) — Docker build + push to registry
Stage 5: Deploy Staging  (~2 min) — deploy to staging environment
Stage 6: E2E Tests      (~10 min) — Playwright against staging
Stage 7: Deploy Prod     (~5 min) — deploy to production (with approval gate)
```

Define:
- Which stages run on every push vs. only on main
- Which stages can run in parallel
- Required status checks before merge

### Phase 3 — GitHub Actions Implementation

Write the complete workflow file(s):
- Concurrency control (cancel stale runs)
- Caching strategy (node_modules, Docker layers)
- Service containers for test dependencies (MongoDB, Redis, Postgres)
- Secrets usage pattern (from GitHub Environments)
- Reusable workflows if multiple services share pipeline logic
- Branch protection rules to configure

### Phase 4 — Docker Registry

Configure image management:
- Registry choice: GitHub Container Registry (free), ECR, Docker Hub
- Tagging strategy: `sha-{git_sha}`, `latest`, branch name
- Image scanning in CI (Trivy, Docker Scout)
- Multi-arch builds if needed (linux/amd64 + linux/arm64)

### Phase 5 — Deployment Strategy

Choose and implement:
- **Rolling**: default for most Kubernetes/ECS deployments
- **Blue-Green**: two environments, instant traffic switch
- **Canary**: 10% → 50% → 100% with automated promotion/rollback
- Rollback procedure (re-deploy previous image tag)
- Health check validation after deploy before marking success

### Phase 6 — Notifications

Configure feedback channels:
- Slack: notify on deploy success/failure to #deployments channel
- PagerDuty: alert on-call if production deploy fails
- GitHub deployment status: visible in PR/commit view
- Dashboard: link to Grafana during deployment for monitoring

### Output

Provide:
1. **Architecture diagram** (text-based) of all pipeline stages
2. **Complete GitHub Actions workflow** (`.github/workflows/ci-cd.yml`)
3. **Branch protection rules** to configure in GitHub settings
4. **Rollback runbook** — steps to revert a bad deployment
5. **Required secrets list** — what to add to GitHub Environments
