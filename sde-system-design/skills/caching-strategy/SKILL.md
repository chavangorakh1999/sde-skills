---
name: caching-strategy
description: "Cache topology, eviction policies (LRU/LFU/TTL), and cache-aside vs write-through vs write-behind patterns. Use when adding caching to a system or debugging cache performance issues."
---

## Caching Strategy

Caching is the most impactful performance optimization available. Used incorrectly, it causes stale data bugs, thundering herd, and cache stampedes. Used correctly, it can reduce database load by 90%+.

### Context

System/service to design caching for: **$ARGUMENTS**

---

### Step 1: Decide What to Cache

Cache data that is: **expensive to compute, read frequently, rarely changed.**

```
Cache candidates:
High value (cache these):
  - Database query results (user profile, product catalog)
  - Computed aggregates (unread count, follower count)
  - External API responses (third-party data, exchange rates)
  - Session data (authenticated user state)
  - Pre-rendered HTML fragments

Low value (don't cache these):
  - Highly personalized data (different for every user/request)
  - Rapidly changing data (stock prices at sub-second intervals)
  - Data that's fast to compute (simple math, in-memory lookups)
  - One-time data (password reset tokens — use Redis TTL directly)
```

**The cache question:** Is the DB query < 5ms? Then caching saves little latency. Is it 200ms? Caching is critical.

---

### Step 2: Cache Topology

**Cache-Aside (Lazy Loading)** — most common pattern

```javascript
// Application reads from cache first; on miss, reads from DB and populates cache
async function getUser(userId) {
  const cacheKey = `user:${userId}`;

  // 1. Try cache
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // 2. Cache miss — read from DB
  const user = await User.findById(userId).lean();
  if (!user) return null;

  // 3. Populate cache (TTL: 1 hour)
  await redis.setex(cacheKey, 3600, JSON.stringify(user));

  return user;
}

// On write: invalidate the cache entry
async function updateUser(userId, updates) {
  const user = await User.findByIdAndUpdate(userId, updates, { new: true });
  await redis.del(`user:${userId}`);  // invalidate, don't update (simpler, consistent)
  return user;
}
```

**Pros:** Simple, only caches what's actually read, no wasted memory on cold data
**Cons:** Cache miss penalty (read from DB + write to cache), potential thundering herd on cold start

---

**Write-Through** — write to cache and DB simultaneously

```javascript
async function updateUser(userId, updates) {
  const user = await User.findByIdAndUpdate(userId, updates, { new: true });

  // Update cache synchronously with the write
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(user));

  return user;
}
```

**Pros:** Cache always warm, no cold-start miss penalty
**Cons:** Write latency includes Redis round-trip; populates cache with data that may never be read

---

**Write-Behind (Write-Back)** — write to cache immediately, persist to DB asynchronously

```javascript
// Write to Redis, queue a background job to persist to DB
async function updateUser(userId, updates) {
  const updatedUser = { ...currentUser, ...updates, updatedAt: new Date() };
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(updatedUser));
  await queue.add('persist-user-update', { userId, updates });
  return updatedUser;
}
```

**Pros:** Lowest write latency (no DB round-trip in hot path)
**Cons:** Data loss risk if Redis fails before job executes; complex failure handling; requires durable queue

**Use write-behind only for:** analytics events, metrics, non-critical counters (view counts, like counts), data that can tolerate ~seconds of persistence delay.

---

### Step 3: Eviction Policies

```javascript
// Redis maxmemory-policy options:

// allkeys-lru — evict least recently used keys across all keys
// Use when: pure cache (all keys are cache entries)
// This is the safest default for a dedicated cache Redis

// volatile-lru — evict LRU keys only from keys with TTL set
// Use when: Redis stores both cache and persistent data (sessions + cache)

// allkeys-lfu — evict least frequently used (Redis 4.0+)
// Use when: access pattern is power-law (few items very hot, many cold)
// Better than LRU for Zipf distribution (typical for user data)

// volatile-ttl — evict keys with shortest remaining TTL first
// Use when: you want strict TTL-based expiration

// noeviction — return error when memory is full (don't evict)
// Use when: Redis is the primary store, not a cache

// Recommendation: allkeys-lru for pure cache, volatile-lru for mixed use
```

**TTL guidelines:**
```
Session data:       30 min - 24h (based on session timeout policy)
User profile:       1h - 4h (changes infrequently)
Product catalog:    15min - 1h (business wants changes to reflect quickly)
Aggregate counts:   5min - 30min (stale is acceptable, freshness matters)
Rate limit windows: Exactly the window duration (1min, 1h)
Email verification: 15min - 1h (security sensitive, short TTL)
Feed results:       5min - 15min (staleness acceptable in social feeds)
```

---

### Step 4: Thundering Herd Prevention

When a high-traffic cache key expires, all concurrent requests miss and hammer the DB simultaneously.

```javascript
// Solution 1: Probabilistic Early Expiration (simpler, no locking)
const BETA = 1.0;  // tuning parameter, higher = more aggressive early refresh

async function getWithEarlyExpiry(key, fetchFn, ttl) {
  const { value, expiresAt, computeTime } = await redis.get(key) ?? {};

  const shouldRefreshEarly =
    value &&
    (Date.now() / 1000) - BETA * computeTime * Math.log(Math.random()) >= expiresAt;

  if (!value || shouldRefreshEarly) {
    const start = Date.now();
    const freshValue = await fetchFn();
    const elapsed = (Date.now() - start) / 1000;

    await redis.setex(key, ttl, JSON.stringify({
      value: freshValue,
      expiresAt: Date.now() / 1000 + ttl,
      computeTime: elapsed
    }));

    return freshValue;
  }

  return value;
}

// Solution 2: Mutex Lock (stronger guarantee, more complex)
async function getWithLock(key, fetchFn, ttl) {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  const lockAcquired = await redis.set(lockKey, '1', 'EX', 10, 'NX');

  if (!lockAcquired) {
    // Another process is refreshing — wait briefly and retry
    await new Promise(resolve => setTimeout(resolve, 100));
    return getWithLock(key, fetchFn, ttl);  // retry
  }

  try {
    const value = await fetchFn();
    await redis.setex(key, ttl, JSON.stringify(value));
    return value;
  } finally {
    await redis.del(lockKey);
  }
}
```

---

### Step 5: Cache Invalidation Patterns

"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

```javascript
// 1. Key-based invalidation (simplest — delete the key on write)
await redis.del(`user:${userId}`);

// 2. Tag-based invalidation (invalidate groups of related keys)
// Tag a set of cache keys with a shared tag
// When any entity changes, invalidate by tag
const cacheKey = `post:${postId}:comments`;
await redis.sadd(`tag:post:${postId}`, cacheKey);

// On post update, invalidate all tagged keys
const taggedKeys = await redis.smembers(`tag:post:${postId}`);
if (taggedKeys.length) await redis.del(...taggedKeys, `tag:post:${postId}`);

// 3. TTL-based (accept some staleness, simplest operationally)
// Just set a reasonable TTL and don't worry about invalidation
// Works when: stale data is acceptable for the TTL window

// 4. Version-based (append version to key, old versions naturally expire)
const version = await redis.get(`user:${userId}:version`) ?? 1;
const cacheKey = `user:${userId}:v${version}`;

// On update: increment version (old key expires naturally by TTL)
await redis.incr(`user:${userId}:version`);
```

---

### Step 6: Redis Cluster vs Sentinel vs Standalone

```
Standalone Redis:
  - Development, low-traffic production (< ~10K QPS)
  - Single point of failure — not for critical cache

Redis Sentinel:
  - Automatic failover, monitoring, no data sharding
  - 1 primary + N replicas (usually 2 replicas minimum)
  - Good for: caches under ~50 GB, single-region

Redis Cluster:
  - Horizontal sharding across 6+ nodes (3 primary, 3 replica minimum)
  - Automatic sharding by key hash slot (16,384 slots)
  - Good for: > 50 GB cache, > 100K QPS, multi-region
  - Multi-key operations (MGET, pipeline) require keys on same node (use hash tags)

// Hash tag example: {user:123}:profile and {user:123}:posts go to same shard
const profileKey = `{user:${userId}}:profile`;
const postsKey   = `{user:${userId}}:posts`;
// Now these can be used in the same MULTI/EXEC transaction
```

---

### Output Format

```
## Caching Strategy: [System]

### What to Cache
[List of cache candidates with justification]

### Cache Topology
[Cache-aside / write-through / write-behind — with rationale]

### Key Design
| Cache Key Pattern | TTL | Eviction Policy | Invalidation Strategy |

### Infrastructure
[Redis standalone / Sentinel / Cluster — sizing and justification]

### Thundering Herd Protection
[How cold start and expiry storms are handled]

### Tradeoffs
[Consistency vs performance, complexity vs reliability]
```
