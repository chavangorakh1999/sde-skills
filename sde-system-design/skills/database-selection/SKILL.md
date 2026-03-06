---
name: database-selection
description: "RDBMS vs document vs wide-column vs graph vs time-series — tradeoffs matched to access patterns and team context. Use when choosing a database for a new service or evaluating whether to migrate."
---

## Database Selection

The right database makes queries natural; the wrong one makes every query a workaround. Choose based on access patterns, consistency requirements, and team expertise — not hype.

### Context

Use case to evaluate: **$ARGUMENTS**

---

### Decision Framework

Ask these questions before opening a database comparison article:

1. What are my dominant access patterns? (by ID, by range, full-text, graph traversal?)
2. What's the consistency requirement? (strong / eventual / per-operation?)
3. How does my data structure change over time? (stable schema vs rapidly evolving?)
4. What's my team's existing expertise?
5. What's the data volume and query rate? (GB vs TB vs PB; 100 QPS vs 100K QPS?)

---

### PostgreSQL — the "start here" default

**Use when:** Relational data with clear entities and relationships; need ACID transactions; team knows SQL; unsure what you need.

```javascript
// Strengths:
// - ACID transactions (even across multiple tables)
// - JSONB for semi-structured data (best of both worlds)
// - Full-text search via pg_trgm and tsvector
// - Partitioning, row-level security, point-in-time recovery
// - Enormous ecosystem (Prisma, Drizzle, Sequelize, Knex)

// Scale ceiling: ~5-10TB on a single node, ~5K-10K writes/sec
// Horizontal: read replicas (PgBouncer for connection pooling)
// Sharding: Citus extension or application-level sharding

// When to move away:
// - > 1TB and hitting performance walls on writes
// - Need sub-millisecond latency at massive scale
// - Schema changes are happening 10x/day (consider MongoDB)
// - Graph traversal is the primary access pattern (consider Neo4j)
```

**Managed options:** AWS RDS PostgreSQL, Supabase, Neon, Railway, Render

---

### MongoDB — flexible documents

**Use when:** Schema evolves rapidly; hierarchical/nested data; team thinks in JSON objects; early-stage product where data model is uncertain.

```javascript
// Strengths:
// - Flexible schema (great during rapid iteration)
// - Natural JSON/JavaScript fit for Node.js teams
// - Horizontal sharding built-in (Atlas handles it)
// - Great for content: articles, products, user profiles
// - Aggregation pipeline for analytics

// Limitations:
// - Multi-document transactions only from v4.0+ (and slower than Postgres)
// - No foreign key enforcement at DB level (application must handle)
// - JOINS require $lookup (expensive vs SQL JOIN)
// - 16MB document size limit
// - Without discipline, schema becomes inconsistent mess

// When MongoDB excels:
// - CMS content, product catalogs, user profiles
// - Event logs, activity feeds (append-mostly, hierarchical)
// - Early startup with unknown schema
// - Team is JS-native, no SQL expertise

// When to avoid:
// - Heavy relational data with many JOINs needed
// - Strong consistency across multiple collections required
// - Strict schema compliance is important (use Mongoose + validation)
```

**Managed options:** MongoDB Atlas (best), AWS DocumentDB (MongoDB-compatible, some missing features)

---

### Redis — in-memory, speed-first

**Use when:** Caching, session storage, rate limiting, leaderboards, pub/sub, message queues (simple).

```javascript
// Strengths:
// - Microsecond latency (in-memory)
// - Rich data structures: String, Hash, List, Set, Sorted Set, Stream, HyperLogLog
// - TTL-native (expiration built-in)
// - Pub/Sub for simple messaging
// - Lua scripting for atomic operations

// Use cases by data structure:
// String:      cache values, counters, flags, distributed locks
// Hash:        user sessions, entity cache (field-level updates)
// Sorted Set:  leaderboards, rate limiting (sliding window), priority queues
// List:        task queues, recent items (capped)
// Set:         unique visitors, tags, "user follows user" (small sets)
// HyperLogLog: approximate unique count with 0.81% error (huge memory saving)
// Streams:     durable message queue with consumer groups (alternative to Kafka for lower volume)

// Limitations:
// - Memory-bound (everything must fit in RAM)
// - Not a primary store for critical data (unless with RDB+AOF persistence)
// - Cluster complicates multi-key operations

// Persistence options:
// RDB (snapshots): fast restart, can lose minutes of data
// AOF (append-only file): durable, up to 1 second loss, larger disk usage
// RDB+AOF: best durability, use for any data that matters
```

---

### Elasticsearch / OpenSearch — search and analytics

**Use when:** Full-text search, log analytics (ELK stack), faceted filtering across many fields.

```javascript
// Strengths:
// - Inverted index — blazing fast text search (BM25 ranking)
// - Aggregations for analytics dashboards (counts, histograms, percentiles)
// - Fuzzy matching, autocomplete, multi-language support
// - Horizontal scaling with automatic shard rebalancing

// Usage pattern (Elasticsearch is usually secondary to a primary DB):
// Primary write: PostgreSQL/MongoDB -> Change Data Capture -> Elasticsearch
// Primary read for search: Elasticsearch -> detail view from primary DB

// Limitations:
// - Not a primary transactional store (no ACID)
// - Operationally complex (shard sizing, index lifecycle management)
// - Eventual consistency within cluster
// - Cost: significant memory requirement (~heap = 50% of RAM, RAM = 2x index size)

// Alternatives:
// - PostgreSQL pg_trgm / tsvector: good enough for < 10M rows, saves infra
// - Algolia: managed, excellent DX, expensive at scale
// - Typesense/Meilisearch: self-hosted, simpler, less powerful
```

---

### DynamoDB — AWS-native, infinite scale

**Use when:** Massive scale (millions of QPS), AWS-native app, predictable access patterns, key-value or simple document reads.

```javascript
// Strengths:
// - Predictable single-digit millisecond latency at any scale
// - Fully managed (no DB tuning, automatic scaling)
// - Pay-per-request or provisioned capacity
// - Global Tables for multi-region active-active
// - Streams for change data capture

// CRITICAL: Data modeling is fundamentally different
// Design your partition key and sort key BEFORE anything else
// All your access patterns must be servable by your key design

// Good fit: shopping carts, gaming leaderboards, IoT sensor data,
//            session stores, time-series with known access patterns

// Poor fit:
// - Ad-hoc queries (no SQL, must know access patterns upfront)
// - Complex reporting (no aggregations, use Athena)
// - Small teams / early stage (operational overhead, key design mistakes are costly)
// - Relational data (JOINs don't exist)

// Key design example: multi-tenant SaaS
// PK: tenantId, SK: userId -> get user by tenant
// GSI PK: email -> login lookup
```

---

### ClickHouse / TimescaleDB — analytics / time-series

**Use when:** High-volume time-series (metrics, events, logs), analytical queries over large datasets.

```javascript
// TimescaleDB (PostgreSQL extension):
// - Add time-series superpowers to Postgres
// - Automatic time-based partitioning (chunks)
// - Continuous aggregates (pre-computed rollups)
// - Works with existing PostgreSQL tooling
// - Good for: IoT data, application metrics, financial data

// ClickHouse:
// - Columnar storage — aggregations 10-100x faster than row stores
// - Best-in-class compression (5-10x vs PostgreSQL)
// - Blazing fast for GROUP BY, COUNT, SUM across billions of rows
// - Write-only-ish: optimize for write throughput, not individual updates
// - Good for: product analytics, log analytics, business intelligence
// - Poor for: OLTP workloads, frequent single-row updates

// Decision:
// < 100M rows/day, team knows Postgres -> TimescaleDB
// > 100M rows/day, analytics-first -> ClickHouse
// Already on AWS, want managed -> Timestream or Redshift
```

---

### Polyglot Persistence (using multiple databases)

Most production systems use 2-3 databases for different jobs:

```
PostgreSQL  — source of truth for transactional entities (users, orders, payments)
Redis       — session cache, rate limiting, real-time leaderboards
Elasticsearch — search and log analytics
S3          — object storage (files, images, videos)
ClickHouse  — product analytics events
```

**Warning:** Every additional database adds operational complexity, backup/restore coordination, consistency coordination across stores, and on-call burden. Start with one. Add others only when the primary store genuinely can't handle the workload.

---

### Output Format

```
## Database Selection: [Use Case]

### Recommended Database(s)
[Primary recommendation with 3-5 sentence justification]

### Access Pattern Match
| Access Pattern | How Database Handles It | Performance |

### Alternatives Considered
| Option | Why Rejected |

### Scale Plan
[When to add replicas, when to shard, migration path]

### Operational Considerations
[Backups, monitoring, connection pooling, managed vs self-hosted]

### Tradeoffs
[3-5 explicit tradeoffs in this decision]
```
