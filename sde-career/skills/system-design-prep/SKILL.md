---
name: system-design-prep
description: "Prepare for system design interviews: the 7-phase framework, common patterns, capacity estimation, and how to talk through trade-offs. Use when preparing for or practicing system design interviews."
---

## System Design Interview Prep

### Context

System to design or prep area: **$ARGUMENTS**

---

### The 7-Phase Framework (Use This Every Time)

```
Phase 1: Clarify Requirements         (3-5 min)
Phase 2: Estimate Scale               (2-3 min)
Phase 3: Define API / Data Model      (5 min)
Phase 4: High-Level Design            (10 min)
Phase 5: Deep Dive into Components    (15 min)
Phase 6: Identify Bottlenecks & Scale (5 min)
Phase 7: Wrap Up                      (2 min)

Total: 45 min. Watch the clock — most candidates spend too long on Phase 1.
```

---

### Phase 1: Requirements Clarification

```
Functional requirements — what does the system DO?
  "Let me confirm the scope before we dive in..."
  - Who are the users?
  - What are the core features? (list them, then prioritize 2-3 for depth)
  - What's out of scope?

Non-functional requirements — how should it PERFORM?
  - Scale: DAU, QPS target
  - Latency: read p99? write p99?
  - Availability: 99.9% (43min/month), 99.99% (4.3min/month)?
  - Consistency: strong, eventual?
  - Durability: can we lose data? (usually: no)
  - Global vs. single region
```

---

### Phase 2: Capacity Estimation

```
Quick math — know these numbers:
  QPS from DAU:  DAU × actions/day / 86,400 = QPS
  Storage:       count × avg_object_size
  Bandwidth:     QPS × avg_response_size
  Read:write ratio: usually 10:1 or 100:1 for social/content apps

Example — URL shortener:
  100M users, 1 URL/day → 1,160 QPS write
  Read:write = 100:1 → 116,000 QPS read
  URL = 500 bytes → storage: 100M × 500B = 50GB/day

Signal to interviewer: "I'll assume X. Is that the right ballpark?"
```

---

### Phase 4: High-Level Design (Draw First, Explain Second)

```
Standard components to draw:
  Client → [CDN] → [Load Balancer] → [API Servers] → [Cache] → [DB]
                                                    ↘ [Message Queue] → [Workers]

Draw on whiteboard/shared doc before explaining:
  - Start with the happy path for the core use case
  - Add components as you explain the need for them
  - Speak as you draw: "The client hits the load balancer, which routes to..."
```

---

### Common System Patterns

```
Read-heavy system:
  → Cache with Redis (cache-aside, write-through)
  → CDN for static assets
  → Read replicas for DB

Write-heavy system:
  → Message queue (Kafka, SQS) to decouple and buffer
  → Write to queue, workers persist to DB
  → Async processing for non-critical writes

Low-latency reads:
  → Keep data in Redis (< 1ms)
  → Denormalize for read efficiency (precompute results)
  → Consider read-your-writes consistency

Fan-out (Twitter, Instagram):
  → Push model: write to followers' feeds on post
  → Pull model: compute feed on read
  → Hybrid: push for normal users, pull for celebrities (millions of followers)

Distributed ID generation:
  → Twitter Snowflake: 64-bit (timestamp + datacenter ID + sequence)
  → UUID v7: sortable, random, no coordination needed

Rate limiting:
  → Token bucket or sliding window in Redis
  → Key: user_id or ip
  → Distributed: Redis atomic INCR + TTL
```

---

### Talking Through Trade-offs

```
Never say just "I'll use Kafka" — always explain WHY and the trade-off:

"I'll use Kafka here rather than SQS because:
  - We need replay ability — Kafka retains messages for 7 days
  - We have multiple consumers (analytics, notifications) that need the same events
  - Trade-off: more operational complexity, need to manage offsets
  - SQS would work if we only had one consumer and didn't need replay"

Trade-off language:
  "The benefit is X, the trade-off is Y"
  "We could alternatively use Z, but that would require..."
  "I'll start with the simpler approach (A) and note that at higher scale we'd move to (B)"
```

---

### Common Topics to Prepare

```
Must-know:
  - URL shortener (encoding, hash collision, redirection)
  - Twitter feed / newsfeed (fan-out, pagination)
  - Typeahead / autocomplete (trie vs. Elasticsearch)
  - Rate limiter (token bucket, sliding window in Redis)
  - Distributed cache (eviction, consistency, thundering herd)
  - Notification system (push, pull, WebSockets, SSE)
  - File upload service (chunked upload, S3 presigned URLs)

Senior/Staff level:
  - Search system (inverted index, Elasticsearch)
  - Payment processing (idempotency, saga pattern, reconciliation)
  - Distributed job scheduler (sharding, exactly-once execution)
  - Recommendation engine (collaborative filtering, feature store)
  - Real-time collaborative editing (OT, CRDTs)
```

---

### Interviewer Signals (What They're Looking For)

```
GREEN signals:                         RED signals:
  Clarify before designing               Jumps to solution immediately
  Estimates back their constraints       No capacity estimation
  Explains trade-offs                    Says "just use Kafka/Redis" without why
  Handles failure modes                  Ignores single points of failure
  Moves between detail levels smoothly   Gets stuck in one component too long
  Says "I'd validate this with..."       Defensive when interviewer probes
```
