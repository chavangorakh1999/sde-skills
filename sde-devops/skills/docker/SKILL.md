---
name: docker
description: "Docker for Node.js applications: multi-stage Dockerfiles, layer caching, security hardening, docker-compose, and image optimization. Use when containerizing an application or debugging Docker issues."
---

## Docker for Node.js

### Context

Docker problem or containerization task: **$ARGUMENTS**

---

### Optimized Multi-Stage Dockerfile

```dockerfile
# ── Stage 1: Install dependencies ─────────────────────────
FROM node:20-alpine AS deps

WORKDIR /app

# Copy package files FIRST — leverages layer cache
# These layers only rebuild when package.json/lock changes
COPY package.json package-lock.json ./

# ci = clean install (faster, reproducible, fails on lock mismatch)
# --only=production = skip devDependencies for runtime image
RUN npm ci --only=production

# ── Stage 2: Build (if you have a build step) ─────────────
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci  # needs devDeps to build

COPY . .
RUN npm run build  # TypeScript compile, Vite build, etc.

# ── Stage 3: Runtime image ────────────────────────────────
FROM node:20-alpine AS runtime

WORKDIR /app

# Security: non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser  --system --uid 1001 --ingroup nodejs nodeapp

# Copy only what's needed to run
COPY --from=deps    --chown=nodeapp:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodeapp:nodejs /app/dist        ./dist
COPY --chown=nodeapp:nodejs package.json ./

USER nodeapp

# Document the port (doesn't publish it — just metadata)
EXPOSE 3000

# Tini as init process — handles PID 1 signal forwarding correctly
# Without tini, SIGTERM may not reach Node.js
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]

# Health check — Docker marks container unhealthy after failures
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

---

### .dockerignore

```
# .dockerignore — critical for build performance and security
node_modules     # copy from deps stage, not host
.git
.github
*.md
docs/
coverage/
.env
.env.*
!.env.example
*.test.js
*.spec.js
__tests__/
.nyc_output
dist/            # rebuilt in builder stage
```

---

### Layer Caching Strategy

```dockerfile
# ORDER MATTERS — most stable layers first

# Layer 1: Base image (changes rarely)
FROM node:20-alpine

# Layer 2: System deps (changes rarely)
RUN apk add --no-cache tini curl

# Layer 3: Package files (changes with dependency updates)
COPY package.json package-lock.json ./
RUN npm ci

# Layer 4: Source code (changes frequently — cache miss here is cheap)
COPY . .
RUN npm run build

# If you put COPY . . before npm ci:
# → every code change rebuilds npm ci = slow!
```

---

### Docker Compose for Development

```yaml
# docker-compose.yml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime           # use runtime stage
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
    env_file:
      - .env.local              # secrets not in compose file
    volumes:
      - ./src:/app/src          # hot reload — bind mount source only
      - /app/node_modules       # anonymous volume — prevent host overwrite
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'

  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
      - ./scripts/mongo-init.js:/docker-entrypoint-initdb.d/init.js:ro
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Optional: Mongo Express for DB browser at http://localhost:8081
  mongo-express:
    image: mongo-express
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_SERVER: mongo
      ME_CONFIG_BASICAUTH: false
    profiles:
      - tools  # only start with: docker compose --profile tools up

volumes:
  mongo_data:
  redis_data:
```

---

### Image Size Optimization

```bash
# Check image size and layers
docker images my-app
docker history my-app --human

# Use alpine base — ~5MB vs ~900MB for node:20
# node:20-alpine = ~180MB vs node:20 = ~1GB

# Audit what's in your image
docker run --rm -it my-app sh
# Then: find / -size +10M 2>/dev/null

# Remove build tools after use
RUN apk add --no-cache --virtual .build-deps \
      python3 make g++ && \
    npm rebuild bcrypt --build-from-source && \
    apk del .build-deps  # removes build tools, keeps bcrypt

# Typical final sizes:
# Node.js API (alpine, production deps only): ~200-400MB
# React build served by nginx: ~30-50MB
```

---

### Security Hardening

```dockerfile
# 1. Non-root user (shown above)
USER nodeapp

# 2. Read-only filesystem where possible
# docker run --read-only --tmpfs /tmp my-app

# 3. No secrets in environment variables in Dockerfile
# BAD:
ENV API_KEY=abc123  # baked into image layer!
# GOOD: pass at runtime via --env-file or secret management

# 4. Scan for vulnerabilities
# docker scout cves my-app
# trivy image my-app

# 5. Pin base image to digest
FROM node:20.11.0-alpine3.19@sha256:abc123...  # immutable, not "latest"

# 6. Minimal attack surface
# Prefer distroless or scratch for compiled languages
# For Node.js: alpine is a good balance
```

---

### Useful Commands

```bash
# Build with specific target stage
docker build --target runtime -t my-app:latest .

# Build with build args
docker build --build-arg NODE_ENV=production -t my-app:prod .

# Inspect layer sizes
docker inspect my-app --format='{{json .RootFS.Layers}}'

# Run with memory limit
docker run --memory=512m --memory-swap=512m my-app

# Copy file out of running container (debugging)
docker cp my-container:/app/logs/error.log ./

# Exec into running container
docker exec -it my-container sh

# View real-time stats
docker stats my-container

# Prune unused images/containers/volumes
docker system prune -a --volumes
```
