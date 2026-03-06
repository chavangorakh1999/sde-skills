---
name: ci-cd-pipeline
description: "CI/CD pipeline design with GitHub Actions: test stages, build, Docker image, deployment strategies, branch protection, and rollback. Use when designing or debugging a CI/CD pipeline."
---

## CI/CD Pipeline Design

### Context

Pipeline requirement or problem: **$ARGUMENTS**

---

### Pipeline Principles

```
Fast feedback:   PRs get test results in < 5 minutes
Deterministic:   same code always produces same result
Progressive:     fast checks run first, slow checks last
Safe:            production deployments require all checks + manual approval (or auto with canary)
Traceable:       every deployment links to a commit and deployment log
```

---

### GitHub Actions — Full Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # cancel stale PR runs when new commit pushed

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  NODE_VERSION: '20'

jobs:
  # ── Stage 1: Code Quality (fast, ~2min) ──────────────────────────────
  quality:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci

      - run: npm run lint
      - run: npm run typecheck  # tsc --noEmit if using TypeScript
      - run: npm run format:check  # prettier --check

  # ── Stage 2: Unit Tests (fast, ~3min) ────────────────────────────────
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npm run test:unit -- --coverage

      - uses: codecov/codecov-action@v4
        with:
          flags: unit
          token: ${{ secrets.CODECOV_TOKEN }}

  # ── Stage 3: Integration Tests (~5min) ───────────────────────────────
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      mongodb:
        image: mongo:7
        ports: ['27017:27017']
        options: >-
          --health-cmd "mongosh --eval 'db.adminCommand(\"ping\")'"
          --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s --health-timeout 5s --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci

      - name: Run integration tests
        run: npm run test:integration
        env:
          MONGODB_URI: mongodb://localhost:27017/test
          REDIS_URL: redis://localhost:6379
          JWT_ACCESS_SECRET: ${{ secrets.TEST_JWT_ACCESS_SECRET }}
          JWT_REFRESH_SECRET: ${{ secrets.TEST_JWT_REFRESH_SECRET }}

  # ── Stage 4: Build & Push Docker Image ───────────────────────────────
  build:
    name: Build Docker Image
    needs: [quality, unit-tests, integration-tests]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    permissions:
      contents: read
      packages: write

    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=raw,value=latest,enable={{is_default_branch}}

      - uses: docker/build-push-action@v5
        id: build
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            GIT_SHA=${{ github.sha }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}

  # ── Stage 5: Deploy to Staging ───────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            docker pull ${{ needs.build.outputs.image-tag }}
            docker stop app || true
            docker run -d --rm --name app \
              -p 3000:3000 \
              --env-file /etc/app/staging.env \
              ${{ needs.build.outputs.image-tag }}

  # ── Stage 6: E2E Tests on Staging (~10min) ───────────────────────────
  e2e-tests:
    name: E2E Tests
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npx playwright install --with-deps chromium

      - name: Run E2E tests against staging
        run: npx playwright test
        env:
          BASE_URL: https://staging.example.com
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}

      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/

  # ── Stage 7: Deploy to Production (manual approval) ──────────────────
  deploy-production:
    name: Deploy to Production
    needs: e2e-tests
    runs-on: ubuntu-latest
    environment:
      name: production    # requires approval in GitHub Environments settings
      url: https://app.example.com

    steps:
      - name: Deploy to production
        run: |
          # Canary: deploy to 10% traffic first
          # Then promote to 100% after health check
          echo "Deploying ${{ needs.build.outputs.image-tag }}"
```

---

### Branch Protection Rules

```
main branch:
  - Require PR with at least 1 review
  - Require all status checks to pass (quality, unit-tests, integration-tests)
  - Require branches to be up to date before merging
  - Restrict who can push directly

develop branch:
  - Require PR with at least 1 review
  - Require quality + unit-tests to pass
```

---

### Pipeline Optimization

```yaml
# Cache node_modules across jobs — don't reinstall every time
- uses: actions/setup-node@v4
  with:
    cache: 'npm'  # caches ~/.npm, restores automatically

# Run independent jobs in parallel
jobs:
  quality:     # parallel
  unit-tests:  # parallel
  integration-tests:  # parallel
  build:
    needs: [quality, unit-tests, integration-tests]  # waits for all

# Skip CI for docs-only changes
on:
  push:
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

---

### Rollback Strategy

```bash
# Quick rollback: re-deploy previous image tag
docker pull ghcr.io/org/app:sha-abc123  # previous SHA
docker stop app
docker run -d --name app --env-file /etc/app/prod.env ghcr.io/org/app:sha-abc123

# Or via GitHub Actions: re-run the deploy job with previous SHA
# gh run rerun <run-id> --job deploy-production
```
