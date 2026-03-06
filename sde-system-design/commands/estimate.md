---
description: Capacity estimation worksheet — QPS, storage, bandwidth, cache sizing with worked numbers and scale implications
argument-hint: "<system name and scale inputs>"
---

# /estimate -- Capacity Estimation

Generate a complete capacity estimation worksheet with worked calculations. Forces you to quantify before you design.

## Invocation

```
/estimate ride-sharing app — 5M daily rides, global
/estimate photo sharing — 50M DAU, 100M photo uploads/day
/estimate chat system — 500M users, 100M messages/day
/estimate                    # prompts for inputs
```

## Workflow

### Step 1: Collect Inputs

Ask for (or extract from arguments):
- DAU (Daily Active Users)
- Key actions per user per day (reads and writes, separately)
- Average object size for the primary data type
- Expected peak multiplier (usually 3-5x average)
- Data retention period (usually 3-5 years)
- Any known constraints (budget, latency targets)

### Step 2: Run the Calculations

Apply **capacity-estimation** skill in full:

**Traffic (QPS):**
```
Avg Read QPS  = DAU × reads per user / 86,400
Avg Write QPS = DAU × writes per user / 86,400
Peak Read QPS = Avg × peak multiplier
Peak Write QPS = Avg × peak multiplier
```

**Storage:**
```
Daily data volume = Write QPS × avg object size × 86,400
3-year storage    = Daily volume × 365 × 3
With replication  = Storage × replication factor (usually 3)
```

**Bandwidth:**
```
Inbound  = Write QPS × avg object size
Outbound = Read QPS × avg response size
Peak outbound = include CDN miss rate
```

**Cache:**
```
Working set = total objects × 0.20 (Pareto estimate)
Cache size  = working set × avg object size
Redis nodes = cache size / per-node capacity
```

**Database connections:**
```
API server count  = ceil(peak QPS / per-server capacity)
Total DB connections = API servers × pool size
Compare vs DB max_connections
```

### Step 3: Identify Bottleneck

State explicitly: what is the primary scaling challenge for this system?
- Write-heavy? -> Focus on DB write throughput, sharding trigger
- Read-heavy? -> Focus on caching strategy, read replica count
- Storage-heavy? -> Focus on object storage, tiering strategy
- Compute-heavy? -> Focus on worker scaling, CDN offload

### Step 4: Infrastructure Sizing

Translate numbers into concrete infrastructure:

```
Component          Count    Sizing           Cost driver
API Servers        12       c5.2xlarge       Peak QPS / server
Read Replicas      3        r5.4xlarge       Read QPS + headroom
Primary DB         1        r5.8xlarge       Write QPS + storage IOPS
Redis Cache        2        r6g.2xlarge      Working set memory
Object Storage     -        S3 Standard      3-year data volume
CDN                -        CloudFront       Outbound bandwidth
```

### Step 5: Scale Triggers

Define when each component needs to grow:

```
Scale trigger: Add read replica when sustained read QPS > X
Scale trigger: Add Redis node when memory > 80%
Scale trigger: Shard DB when data > 1TB or write QPS > Y
Scale trigger: Add API servers when CPU > 70% sustained
```

### Step 6: Offer Next Steps

- "Want a full system design? -> `/design [system name]`"
- "Should I design the database schema? -> `/model-data [domain]`"
- "Want the API design? -> `/design-api [resource]`"

## Output

```
## Capacity Estimation: [System Name]

### Inputs
| Parameter | Value | Assumption |

### Traffic
| Metric | Calculation | Result |

### Storage
| Metric | Calculation | Result |

### Bandwidth
| Metric | Calculation | Result |

### Cache
| Metric | Calculation | Result |

### Infrastructure
| Component | Count | Sizing | Rationale |

### Scale Triggers
| Component | Metric | Threshold | Action |

### Key Insight
[One paragraph: the dominant scaling challenge and the most important decision it drives]
```
