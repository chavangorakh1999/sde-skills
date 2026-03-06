---
name: nodejs-patterns
description: "Node.js patterns for production: async/await error handling, streams, worker threads, clustering, event emitter patterns, graceful shutdown. Use when building or optimizing Node.js backend services."
---

## Node.js Production Patterns

### Context

Node.js pattern or problem: **$ARGUMENTS**

---

### Async Error Handling

```javascript
// Anti-pattern: unhandled promise rejections crash the process
app.get('/users', async (req, res) => {
  const users = await User.find();  // throws? — unhandled!
  res.json(users);
});

// Pattern 1: asyncHandler wrapper
export function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Usage — no try/catch needed in route handlers
router.get('/users', asyncHandler(async (req, res) => {
  const users = await userService.list(req.query);
  res.json({ data: users });
}));

// Pattern 2: express-async-errors (patches Express globally)
import 'express-async-errors';  // must be first import after express
// Now async errors automatically reach error middleware

// Always handle process-level errors
process.on('unhandledRejection', (reason, promise) => {
  logger.fatal({ reason, promise }, 'unhandled rejection — shutting down');
  process.exit(1);  // let process manager restart
});

process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'uncaught exception — shutting down');
  process.exit(1);
});
```

---

### Graceful Shutdown

```javascript
// server.js — proper shutdown prevents in-flight request drops
import http from 'http';
import { mongoose } from './config/db.js';

const server = http.createServer(app);
let isShuttingDown = false;

// Health check respects shutdown state
app.get('/health', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).json({ status: 'shutting_down' });
  }
  res.json({ status: 'ok' });
});

async function shutdown(signal) {
  if (isShuttingDown) return;
  isShuttingDown = true;

  logger.info({ signal }, 'shutdown initiated');

  // 1. Stop accepting new connections
  server.close(async () => {
    logger.info('HTTP server closed');

    try {
      // 2. Close database connections
      await mongoose.connection.close();
      logger.info('MongoDB disconnected');

      // 3. Close Redis, queues, etc.
      await redisClient.quit();
      logger.info('Redis disconnected');

      logger.info('graceful shutdown complete');
      process.exit(0);
    } catch (err) {
      logger.error({ err }, 'error during shutdown');
      process.exit(1);
    }
  });

  // Force exit if graceful shutdown takes too long
  setTimeout(() => {
    logger.error('forced shutdown after timeout');
    process.exit(1);
  }, 30_000);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

server.listen(process.env.PORT ?? 3000, () => {
  logger.info({ port: server.address().port }, 'server started');
});
```

---

### Streams for Large Data

```javascript
// Never load entire files/datasets into memory
import { createReadStream, createWriteStream } from 'fs';
import { pipeline } from 'stream/promises';
import { createGzip } from 'zlib';
import { Transform } from 'stream';

// Stream a large CSV response
app.get('/exports/users', authenticate, asyncHandler(async (req, res) => {
  const cursor = User.find({ active: true }).cursor();  // Mongoose cursor

  res.setHeader('Content-Type', 'text/csv');
  res.setHeader('Content-Disposition', 'attachment; filename=users.csv');

  // Write CSV header
  res.write('id,email,createdAt\n');

  for await (const user of cursor) {
    res.write(`${user._id},${user.email},${user.createdAt.toISOString()}\n`);
  }

  res.end();
}));

// Transform stream example — process large file line by line
import { createInterface } from 'readline';

async function processLargeFile(filePath) {
  const rl = createInterface({
    input: createReadStream(filePath),
    crlfDelay: Infinity
  });

  const BATCH_SIZE = 500;
  let batch = [];

  for await (const line of rl) {
    batch.push(JSON.parse(line));

    if (batch.length >= BATCH_SIZE) {
      await processBatch(batch);
      batch = [];
    }
  }

  if (batch.length > 0) await processBatch(batch);
}

// Piping with proper error handling
async function compressFile(inputPath, outputPath) {
  await pipeline(
    createReadStream(inputPath),
    createGzip(),
    createWriteStream(outputPath)
  );
  // pipeline automatically handles errors and cleanup
}
```

---

### Worker Threads for CPU Work

```javascript
// CPU-bound work blocks the event loop — offload to worker threads
// workers/imageProcessor.js
import { workerData, parentPort } from 'worker_threads';
import sharp from 'sharp';

const { inputBuffer, width, height } = workerData;

sharp(inputBuffer)
  .resize(width, height)
  .webp({ quality: 80 })
  .toBuffer()
  .then(output => parentPort.postMessage({ output }))
  .catch(err => parentPort.postMessage({ error: err.message }));

// Main thread — worker pool
import { Worker } from 'worker_threads';

function runWorker(workerPath, data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(workerPath, { workerData: data });
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', code => {
      if (code !== 0) reject(new Error(`Worker exited with code ${code}`));
    });
  });
}

// Service using worker:
export async function processImage(buffer, width, height) {
  const result = await runWorker('./workers/imageProcessor.js', {
    inputBuffer: buffer, width, height
  });
  if (result.error) throw new Error(result.error);
  return result.output;
}

// For production: use a thread pool library like piscina
import Piscina from 'piscina';

const pool = new Piscina({
  filename: new URL('./workers/imageProcessor.js', import.meta.url).href,
  maxThreads: 4
});

export async function processImagePooled(buffer, width, height) {
  return pool.run({ inputBuffer: buffer, width, height });
}
```

---

### Event Emitter Pattern

```javascript
// Use EventEmitter for decoupled internal pub/sub
import { EventEmitter } from 'events';

class DomainEventBus extends EventEmitter {
  constructor() {
    super();
    this.setMaxListeners(20);  // avoid memory leak warning for large apps
  }

  publish(event, data) {
    this.emit(event, { ...data, timestamp: new Date() });
  }
}

export const eventBus = new DomainEventBus();

// In user service:
async function registerUser(dto) {
  const user = await userRepo.create(dto);
  eventBus.publish('user.registered', { userId: user.id, email: user.email });
  return user;
}

// In email service (separate module — no coupling):
eventBus.on('user.registered', async ({ userId, email }) => {
  await emailService.sendWelcome(email);
});

// In analytics service:
eventBus.on('user.registered', async ({ userId }) => {
  await analyticsService.trackSignup(userId);
});

// Error in async listener won't propagate — handle explicitly:
function safeListener(fn) {
  return async (...args) => {
    try {
      await fn(...args);
    } catch (err) {
      logger.error({ err }, 'event listener error');
    }
  };
}

eventBus.on('user.registered', safeListener(async ({ email }) => {
  await emailService.sendWelcome(email);
}));
```

---

### Clustering

```javascript
// cluster.js — use all CPU cores
import cluster from 'cluster';
import { cpus } from 'os';

const NUM_WORKERS = process.env.WEB_CONCURRENCY ?? cpus().length;

if (cluster.isPrimary) {
  logger.info({ workers: NUM_WORKERS }, 'primary process starting');

  for (let i = 0; i < NUM_WORKERS; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    logger.warn({ workerId: worker.id, code, signal }, 'worker died — restarting');
    cluster.fork();  // auto-restart dead workers
  });
} else {
  // Each worker runs the full Express app
  import('./server.js');
  logger.info({ workerId: cluster.worker.id, pid: process.pid }, 'worker started');
}

// Note: prefer PM2 or container orchestration over manual clustering
// PM2: pm2 start server.js -i max
```

---

### Config Management

```javascript
// config/index.js — validated env vars at startup, fail fast
import Joi from 'joi';

const schema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
  PORT: Joi.number().default(3000),
  MONGODB_URI: Joi.string().uri().required(),
  JWT_ACCESS_SECRET: Joi.string().min(32).required(),
  JWT_REFRESH_SECRET: Joi.string().min(32).required(),
  REDIS_URL: Joi.string().uri().required(),
}).unknown(true);  // allow other env vars

const { error, value } = schema.validate(process.env);
if (error) {
  console.error('Invalid environment config:', error.message);
  process.exit(1);  // crash early with helpful message
}

export const config = {
  env: value.NODE_ENV,
  port: value.PORT,
  mongoUri: value.MONGODB_URI,
  jwt: {
    accessSecret: value.JWT_ACCESS_SECRET,
    refreshSecret: value.JWT_REFRESH_SECRET,
  },
  redis: { url: value.REDIS_URL },
};
```

---

### Output Format

```
## Node.js Pattern: [Pattern/Problem]

### Root Cause / Context
[Why this pattern is needed]

### Implementation
[Code with comments]

### Production Considerations
- Memory: [impact]
- Error handling: [approach]
- Observability: [what to log/monitor]
```
