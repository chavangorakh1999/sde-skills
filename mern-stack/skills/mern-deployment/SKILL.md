---
name: mern-deployment
description: "Deploying MERN apps: Dockerfile for Node.js/React, docker-compose for local dev, CI/CD with GitHub Actions, MongoDB Atlas, environment configuration, health checks, and zero-downtime deployment. Use when containerizing or deploying a MERN app."
---

## MERN Deployment

### Context

Deployment target or problem: **$ARGUMENTS**

---

### Dockerfile — Node.js Backend

```dockerfile
# backend/Dockerfile
# Stage 1: Build dependencies
FROM node:20-alpine AS deps
WORKDIR /app

# Copy package files first for layer caching
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Stage 2: Runtime image
FROM node:20-alpine AS runtime
WORKDIR /app

# Security: run as non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodeapp && \
    chown -R nodeapp:nodejs /app

# Copy production deps from stage 1
COPY --from=deps --chown=nodeapp:nodejs /app/node_modules ./node_modules

# Copy source
COPY --chown=nodeapp:nodejs . .

USER nodeapp

# Validate required env vars at build time (optional)
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', r => r.statusCode === 200 ? process.exit(0) : process.exit(1))"

CMD ["node", "src/server.js"]
```

---

### Dockerfile — React Frontend

```dockerfile
# frontend/Dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

# Inject build-time env vars (these get baked into the JS bundle)
ARG VITE_API_URL
ENV VITE_API_URL=${VITE_API_URL}

RUN npm run build  # outputs to /app/dist

# Stage 2: Serve with nginx
FROM nginx:alpine AS runtime

# Custom nginx config for SPA routing
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# frontend/nginx.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_types text/plain application/javascript text/css application/json;
    gzip_min_length 1000;

    # Cache static assets
    location ~* \.(js|css|png|jpg|svg|ico|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA routing — all paths serve index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API calls (optional — avoids CORS in production)
    location /api {
        proxy_pass http://backend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

### Docker Compose — Local Development

```yaml
# docker-compose.yml
version: '3.9'

services:
  backend:
    build:
      context: ./backend
      target: runtime
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - MONGODB_URI=mongodb://mongo:27017/myapp
      - REDIS_URL=redis://redis:6379
    env_file:
      - ./backend/.env.local   # secrets not in compose file
    volumes:
      - ./backend/src:/app/src  # hot reload in dev
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: node --watch src/server.js  # Node 18+ built-in watch

  frontend:
    build:
      context: ./frontend
      target: builder  # use builder stage for dev server
    ports:
      - "5173:5173"
    volumes:
      - ./frontend/src:/app/src
    environment:
      - VITE_API_URL=http://localhost:3000
    command: npm run dev -- --host

  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mongo_data:
  redis_data:
```

---

### GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mongo:
        image: mongo:7
        ports: ["27017:27017"]
        options: >-
          --health-cmd mongosh --eval "db.adminCommand('ping')"
          --health-interval 10s --health-timeout 5s --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        run: npm ci
        working-directory: backend

      - name: Lint
        run: npm run lint
        working-directory: backend

      - name: Test
        run: npm test -- --coverage
        working-directory: backend
        env:
          NODE_ENV: test
          MONGODB_URI: mongodb://localhost:27017/test
          JWT_ACCESS_SECRET: ${{ secrets.TEST_JWT_ACCESS_SECRET }}
          JWT_REFRESH_SECRET: ${{ secrets.TEST_JWT_REFRESH_SECRET }}

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          directory: backend/coverage

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

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
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/backend
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest

      - uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Deploy to Railway / Render / Fly.io
        # Example: Fly.io deploy
        uses: superfly/flyctl-actions/setup-flyctl@master

      - run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

---

### Environment Configuration

```
# backend/.env.example (commit this)
NODE_ENV=production
PORT=3000
MONGODB_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/myapp
REDIS_URL=redis://:<pass>@redis-host:6379
JWT_ACCESS_SECRET=<generate: openssl rand -hex 32>
JWT_REFRESH_SECRET=<generate: openssl rand -hex 32>
ALLOWED_ORIGINS=https://app.example.com
LOG_LEVEL=info

# Generate secrets:
# openssl rand -hex 32
```

---

### Health Check Endpoint

```javascript
// routes/health.js
import mongoose from 'mongoose';
import { redisClient } from '../config/redis.js';

router.get('/health', async (req, res) => {
  const checks = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.npm_package_version,
    checks: {}
  };

  // MongoDB
  const mongoState = mongoose.connection.readyState;
  checks.checks.mongodb = mongoState === 1 ? 'healthy' : 'unhealthy';

  // Redis
  try {
    await redisClient.ping();
    checks.checks.redis = 'healthy';
  } catch {
    checks.checks.redis = 'unhealthy';
  }

  const isHealthy = Object.values(checks.checks).every(s => s === 'healthy');

  if (!isHealthy) {
    checks.status = 'degraded';
    return res.status(503).json(checks);
  }

  res.json(checks);
});

// Liveness (is app running?):
router.get('/health/live', (req, res) => res.json({ status: 'ok' }));

// Readiness (is app ready for traffic?):
router.get('/health/ready', async (req, res) => {
  const ready = mongoose.connection.readyState === 1;
  res.status(ready ? 200 : 503).json({ ready });
});
```

---

### Platform Quick Reference

```
Railway:        railway up — auto-detects Node.js, easy env vars, Postgres/Redis plugins
Render:         render.yaml — free tier, auto-deploy from git, managed Postgres
Fly.io:         flyctl deploy — edge deployment, global regions, affordable
Vercel:         frontend + serverless functions (not ideal for stateful Express)
MongoDB Atlas:  managed MongoDB — free M0 tier, M10+ for production
Redis Cloud:    managed Redis — free 30MB tier

Production minimums:
  Backend:  512MB RAM, 0.5 CPU
  MongoDB:  M10 ($57/mo) for replica set and backups
  Redis:    30MB free is enough for sessions; upgrade for queues
```
