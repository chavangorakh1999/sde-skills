---
description: "Containerize an application with Docker: write optimized Dockerfile, .dockerignore, and docker-compose for local dev"
argument-hint: "[application type and requirements, e.g. 'Node.js Express API with MongoDB and Redis']"
---

## Dockerize Application

Create production-quality Docker configuration for the application.

### Phase 1 — Application Analysis

Identify:
- **Language/runtime**: Node.js 20, Python 3.11, Go 1.21, etc.
- **Build step needed?**: TypeScript compile, Vite build, etc.
- **External dependencies**: databases, caches, queues
- **Required system packages**: ImageMagick, ffmpeg, etc.
- **Secrets/config**: how does the app receive config?

### Phase 2 — Dockerfile Strategy

Choose the image base:
- `node:20-alpine` — Node.js apps (small: ~180MB)
- `node:20-slim` — when alpine is incompatible with native modules
- `nginx:alpine` — for serving static files (React build)
- Distroless — for maximum security (no shell, minimal surface)

Design the multi-stage build:
```
Stage 1: deps     — install production dependencies
Stage 2: builder  — install all deps, run build step (if needed)
Stage 3: runtime  — minimal image, copy only what's needed to run
```

Apply security hardening:
- Non-root user
- Read-only root filesystem where possible
- Drop all Linux capabilities
- Pin base image to specific version or digest

### Phase 3 — Layer Caching Optimization

Order Dockerfile instructions for maximum cache reuse:
1. Base image (never changes)
2. System packages (rarely changes)
3. package.json + lock file + npm ci (changes with deps)
4. Source code COPY (changes frequently)
5. Build step

Verify with: `docker history <image> --human`

### Phase 4 — .dockerignore

Write .dockerignore to exclude:
- node_modules (will be reinstalled in container)
- .git history
- test files and coverage reports
- .env files (secrets must never be in image)
- Documentation

### Phase 5 — Docker Compose for Local Dev

Write docker-compose.yml with:
- Application service with volume mounts for hot reload
- All dependency services (MongoDB, Redis, Postgres) with health checks
- Environment configuration (env_file for secrets, environment for non-sensitive)
- Networks and named volumes
- Resource limits
- Optional tool containers (mongo-express, redis-commander) with profiles

### Phase 6 — Health Check

Add HEALTHCHECK to Dockerfile:
- Endpoint: `/health/live` for liveness
- Tool: prefer `wget` over `curl` on alpine (already installed)
- Timing: start-period for slow-starting apps, reasonable interval

### Phase 7 — Production Considerations

Address:
- **Image size**: target < 500MB, verify with `docker images`
- **Security scanning**: `trivy image myapp:latest` or `docker scout cves`
- **Signal handling**: use tini as PID 1 (or `docker run --init`)
- **Logging**: write to stdout/stderr (Docker captures this)
- **Secrets**: never in Dockerfile ENV, always runtime via --env-file or secrets manager

### Output

Provide:
1. **Dockerfile** with multi-stage build, comments on each decision
2. **.dockerignore** with everything that should be excluded
3. **docker-compose.yml** with all services and health checks
4. **Build and run commands** to verify it works
5. **Image size breakdown** and optimization notes
6. **Security checklist** for the specific image
