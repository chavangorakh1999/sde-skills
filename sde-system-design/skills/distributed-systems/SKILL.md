---
name: distributed-systems
description: "CAP theorem applied to real decisions, consensus (Raft/Paxos), distributed transactions, saga pattern, idempotency. Use when designing systems that span multiple nodes or services."
---

## Distributed Systems

Building distributed systems means accepting that nodes fail, networks partition, and clocks drift. The goal is not to eliminate failure — it's to design systems that behave predictably when failure occurs.

### Context

System or problem to analyze: **$ARGUMENTS**

---

### CAP Theorem — Applied

CAP says: in a network partition, you must choose between **Consistency** and **Availability**. But the choice is per-operation, not per-system.

```
CP systems (Consistency over Availability):
  - On partition: reject writes (return error) to prevent inconsistency
  - Examples: ZooKeeper, etcd (used for leader election, distributed locks)
  - Use when: data correctness is mandatory, brief downtime is acceptable
  - Real example: bank balance — better to fail than to double-credit

AP systems (Availability over Consistency):
  - On partition: accept writes, resolve conflicts later (eventual consistency)
  - Examples: Cassandra, DynamoDB (by default), CouchDB
  - Use when: availability is paramount, staleness is acceptable
  - Real example: shopping cart — keep taking orders, reconcile inventory later

CA systems (no network partition tolerance):
  - Traditional single-node RDBMS
  - Not viable for distributed systems (partitions always happen eventually)

PACELC extension (more nuanced):
  - During Partition: choose C or A (CAP)
  - Else (no partition): choose Latency or Consistency
  - DynamoDB: PA/EL (available during partition, low latency otherwise)
  - PostgreSQL: PC/EC (consistent during partition, consistent otherwise)
```

**Practical rule:** Use strong consistency for money/inventory/auth. Use eventual consistency for social feeds, view counts, search indexes, recommendations.

---

### Distributed Transactions

Two common approaches when a transaction must span multiple services or nodes:

**Two-Phase Commit (2PC):**
```
Phase 1 (Prepare): Coordinator asks all participants: "Can you commit?"
Phase 2 (Commit):  If all say yes -> send COMMIT; if any says no -> ROLLBACK

Problems:
- Blocking protocol: if coordinator fails between phases, participants are stuck
- Not suitable for microservices (too tightly coupled, long lock hold times)
- Use only within a single database cluster (Postgres distributed transactions)
```

**Saga Pattern (for microservices):**
```
Break a distributed transaction into local transactions + compensating transactions.
Each step either succeeds and triggers the next, or fails and triggers rollbacks.

Example: E-commerce order flow
Step 1: Order Service      — create order (compensate: cancel order)
Step 2: Payment Service    — charge payment (compensate: refund)
Step 3: Inventory Service  — reserve stock (compensate: release reservation)
Step 4: Shipping Service   — schedule delivery (compensate: cancel delivery)

If Step 3 fails:
- Execute compensation for Step 2: refund payment
- Execute compensation for Step 1: cancel order
```

**Choreography vs Orchestration:**
```javascript
// Choreography: each service emits events, next service reacts
// Order placed -> PaymentService hears it -> Payment charged -> InventoryService hears it -> ...
// Pro: no central coordinator, loose coupling
// Con: flow is implicit, hard to trace, hard to handle failures

// Orchestration: a saga orchestrator commands each step
// OrderSaga -> tells PaymentService to charge -> waits -> tells InventoryService -> waits -> ...
// Pro: explicit flow, easier failure handling, traceable
// Con: orchestrator is a central point, can become a God service
// Recommendation: orchestration for critical business flows, choreography for simple chains
```

---

### Idempotency

Every distributed operation must be safe to retry. Networks fail; retries happen.

```javascript
// Server-side idempotency key pattern
// Client generates a UUID per logical operation and sends it as a header
// Server stores (idempotency_key -> result) and replays on duplicate request

// Database table for idempotency
// CREATE TABLE idempotency_keys (
//   key        VARCHAR(255) PRIMARY KEY,
//   response   JSONB,
//   status_code INT,
//   expires_at  TIMESTAMPTZ,
//   created_at  TIMESTAMPTZ DEFAULT now()
// );

async function processWithIdempotency(idempotencyKey, operation) {
  // Check if already processed
  const existing = await db.query(
    'SELECT response, status_code FROM idempotency_keys WHERE key = $1 AND expires_at > NOW()',
    [idempotencyKey]
  );
  if (existing.rows.length > 0) {
    return existing.rows[0];  // replay stored result
  }

  // Process once
  const result = await operation();

  // Store result
  await db.query(
    `INSERT INTO idempotency_keys (key, response, status_code, expires_at)
     VALUES ($1, $2, $3, NOW() + INTERVAL '24 hours')
     ON CONFLICT (key) DO NOTHING`,
    [idempotencyKey, JSON.stringify(result.body), result.statusCode]
  );

  return result;
}
```

**Idempotent operations by design:**
- PUT (replace whole resource) is naturally idempotent
- DELETE is idempotent (deleting already-deleted returns 404 or 204, both fine)
- POST is not idempotent by default — use idempotency keys
- PATCH may not be idempotent if it applies increments (`+= 1`)

---

### Leader Election and Consensus

```javascript
// When you need a single leader among distributed nodes:
// - Shard ownership
// - Scheduled job execution (run cron only once across N instances)
// - Distributed lock acquisition

// Use a proven consensus system rather than rolling your own:
// - etcd or ZooKeeper: production-grade, used by Kubernetes
// - Redis SETNX (for simple locks, not true consensus):

const LOCK_TTL = 30;  // seconds

async function acquireLock(redis, lockKey, ownerId) {
  // NX: only set if not exists, EX: set TTL
  const acquired = await redis.set(lockKey, ownerId, 'EX', LOCK_TTL, 'NX');
  return acquired === 'OK';
}

async function releaseLock(redis, lockKey, ownerId) {
  // Only release if we own the lock (Lua script for atomicity)
  const script = `
    if redis.call('get', KEYS[1]) == ARGV[1] then
      return redis.call('del', KEYS[1])
    else
      return 0
    end
  `;
  return redis.eval(script, 1, lockKey, ownerId);
}

// WARNING: Redis SETNX is not a true distributed lock (RedLock is controversial).
// For critical resources (payments, inventory): use a dedicated consensus system.
// For non-critical (cron jobs, cache warming): Redis SETNX with careful TTL is fine.
```

---

### Clock Skew and Ordering

Clocks drift. Never use wall-clock time to determine event ordering in distributed systems.

```javascript
// Problem: Server A records event at 10:00:00.001
//          Server B records event at 10:00:00.000 (clock skew)
//          B's event appears to come before A's — wrong!

// Solution 1: Logical clocks (Lamport timestamps)
// Each node increments a counter on send/receive
// Provides partial ordering: if A happened-before B, A's timestamp < B's
// Does not provide total ordering

// Solution 2: Hybrid Logical Clocks (HLC)
// Combine physical clock + logical clock
// Monotonically increasing, close to wall-clock
// Used by CockroachDB, YugabyteDB

// Solution 3: ULID / UUID v7 (for database IDs)
// ULID: 128 bits = 48-bit timestamp (ms) + 80-bit random
// Monotonically increasing within a millisecond (random component breaks ties)
// Better DB index locality than UUID v4 (random)

// For event ordering in distributed logs: use sequence numbers assigned
// by a single authoritative sequencer (Kafka partition offset, PostgreSQL sequence)
```

---

### Consistent Hashing (for data partitioning)

```javascript
// Problem: When you add/remove a server from a hash ring,
// you want to minimize the number of keys that need to be remapped.

// Traditional hash: key % N servers
// Problem: adding server N+1 changes almost every key's assignment (remaps ~N/(N+1) of keys)

// Consistent hashing: place servers on a ring, each key goes to the nearest server clockwise
// Adding a server only remaps keys in the new server's arc (~1/N of total keys)

// Virtual nodes (vnodes): each physical server gets K positions on the ring
// Provides better load distribution when servers have different capacities
// Standard: 150-200 virtual nodes per physical node (used by Cassandra)

// Use cases: cache sharding (Redis Cluster uses hash slots, a variant),
//            distributed storage (Cassandra), consistent routing
```

---

### Output Format

```
## Distributed Systems Design: [Component/Problem]

### Consistency Model
[Strong / Eventual / Causal — with justification]

### CAP Choice
[CP or AP in partition scenario — why?]

### Transaction Strategy
[Single DB transaction / Saga / 2PC — with failure scenarios]

### Idempotency
[How retries are handled safely]

### Failure Scenarios
| Failure | Behavior | Recovery |

### Tradeoffs
[Consistency vs availability, complexity vs reliability, etc.]
```
