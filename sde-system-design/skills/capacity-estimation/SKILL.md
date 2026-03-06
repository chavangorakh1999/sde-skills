---
name: capacity-estimation
description: "Back-of-envelope capacity estimation: QPS, storage, bandwidth, cache sizing with worked numbers and scale implications. Use when sizing infrastructure or answering 'how big does this need to be?'"
---

## Capacity Estimation

Turn vague scale requirements into concrete numbers that drive infrastructure decisions. Every estimate should answer: what hardware, how much of it, and when do we need to scale?

### Context

System to estimate: **$ARGUMENTS**

---

### Step 1: Nail Down the Inputs

Before calculating anything, extract:

```
DAU (Daily Active Users):     _______
Read-to-write ratio:          ___:1
Peak multiplier (spike):      ___x average
Data retention period:        ___ years
Average object/record size:   ___ KB
```

**Useful reference points:**
- Twitter: ~300M DAU, ~500M tweets/day
- YouTube: ~80M DAU uploads, ~5B views/day
- Average user session: 10-30 min
- Pareto principle: 20% of users generate 80% of traffic

---

### Step 2: Traffic (QPS)

```javascript
// Example: Photo sharing app
// DAU = 50M, avg 10 reads + 2 writes per user per day

const DAU = 50_000_000;
const readsPerUser = 10;
const writesPerUser = 2;
const secondsPerDay = 86_400;

const avgReadQPS  = (DAU * readsPerUser)  / secondsPerDay;  // ~5,787 QPS
const avgWriteQPS = (DAU * writesPerUser) / secondsPerDay;  // ~1,157 QPS

// Peak (3-5x average for most consumer apps)
const peakReadQPS  = avgReadQPS  * 3;  // ~17,361 QPS
const peakWriteQPS = avgWriteQPS * 3;  // ~3,472 QPS
```

**Rule of thumb:**
- < 1K QPS: single server fine
- 1K-10K QPS: caching layer critical, consider read replicas
- 10K-100K QPS: horizontal scaling + CDN required
- > 100K QPS: sharding, regional deployment, serious infra work

---

### Step 3: Storage

```javascript
// Photo metadata (rows in DB)
const photoMetaSize = 300;        // bytes per row
const dailyPhotoUploads = avgWriteQPS * secondsPerDay;  // ~100M/day
const metaStoragePerYear = dailyPhotoUploads * photoMetaSize * 365;
// = 100M * 300B * 365 = ~10.95 TB/year (metadata only)

// Photo files (object storage)
const avgPhotoSize = 500_000;     // 500 KB compressed
const photoStoragePerYear = dailyPhotoUploads * avgPhotoSize * 365;
// = 100M * 500KB * 365 = ~18.25 PB/year

// 3-year total
const totalStorage = photoStoragePerYear * 3;  // ~54.75 PB
```

**Storage tiers:**
- Hot (< 30 days): SSD, fast retrieval, expensive — keep minimal
- Warm (30d - 1y): HDD or S3 Standard
- Cold (> 1y): S3 Glacier, cheap but slow retrieval

---

### Step 4: Bandwidth

```javascript
// Inbound (uploads)
const inboundBandwidth = avgWriteQPS * avgPhotoSize;
// 1,157 QPS * 500 KB = ~578 MB/s = ~4.6 Gbps

// Outbound (reads — often much higher due to CDN miss)
const avgResponseSize = 50_000;  // 50 KB (thumbnail)
const outboundBandwidth = avgReadQPS * avgResponseSize;
// 5,787 QPS * 50 KB = ~289 MB/s = ~2.3 Gbps

// CDN absorbs ~90% of reads -> origin gets ~289 MB/s
```

**Bandwidth → server count:**
- Standard 1Gbps NIC: ~100 MB/s effective
- Standard 10Gbps NIC: ~1 GB/s effective
- Divide total bandwidth by per-server capacity to get minimum server count

---

### Step 5: Cache Sizing

```javascript
// Working set principle: 20% of data handles 80% of reads
const totalObjects = dailyPhotoUploads * 365 * 3;   // 3-year corpus
const hotData      = totalObjects * 0.20;            // 20% hot
const cacheSize    = hotData * avgPhotoSize;

// For metadata cache (small, fast):
const metadataCacheSize = hotData * photoMetaSize;
// Much smaller — fits in Redis comfortably
```

**Cache hit rate targets:**
- < 80%: revisit data model or cache strategy
- 80-90%: acceptable for most apps
- > 95%: good for read-heavy systems
- > 99%: exceptional (static content, pre-computed results)

**Redis memory sizing:**
- Overhead: ~50-100 bytes per key beyond value size
- Eviction: LRU for general cache, LFU for frequency-biased access
- `maxmemory-policy`: `allkeys-lru` for pure cache, `volatile-lru` if TTLs mixed

---

### Step 6: Database Connections

```javascript
// Connection pool math
const apiServers    = Math.ceil(peakReadQPS / 500);  // assume 500 RPS per server
const connectionsPerServer = 10;   // typical pool size
const totalConnections = apiServers * connectionsPerServer;

// PostgreSQL default: max_connections = 100
// At 10 API servers * 10 conn = 100 connections — maxed out!
// Solution: PgBouncer connection pooler, or increase max_connections
```

**Database server sizing:**
- Single Postgres: handles ~5K-10K simple queries/sec on decent hardware
- Read replica: offload reads, add 1 replica per 3K read QPS above primary capacity
- Sharding trigger: > 1TB data and query performance degrading, or > 10K writes/sec

---

### Step 7: Implications Summary

After calculating, always state:
- How many API servers do you need?
- Do you need DB sharding now or can you defer?
- What's the biggest cost driver?
- When does the current architecture hit its limits?

```
## Capacity Summary: [System Name]

| Metric         | Value      | Implication                          |
|----------------|-----------|--------------------------------------|
| Read QPS (avg) | 5,787      | Single DB replica handles this       |
| Write QPS (avg)| 1,157      | Well within single primary capacity  |
| Peak Read QPS  | 17,361     | Need cache layer, 2-3 read replicas  |
| Storage (3yr)  | 54.75 PB   | Object storage (S3), not block        |
| Bandwidth out  | 2.3 Gbps   | CDN mandatory for cost               |
| Cache size     | ~50 GB     | Fits in single Redis node            |
| API servers    | ~35        | At 500 RPS per server, peak          |

### Scale Triggers
- Add read replica when: read QPS > 8K sustained
- Shard DB when: data > 1TB or write QPS > 5K
- Add Redis cluster when: cache > 80% memory threshold
```
