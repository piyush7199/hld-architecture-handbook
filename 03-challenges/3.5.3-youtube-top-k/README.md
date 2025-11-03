# 3.5.3 Design YouTube Top K (Trending Algorithm)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## Problem Statement

Design a highly scalable, low-latency trending algorithm system for YouTube that can calculate and serve the "Top K" most popular videos globally and regionally in real-time. The system must handle 1 million view events per second, apply time-decay to favor recent content, filter fraudulent views, and serve trending lists with sub-100ms latency.

---

## Requirements and Scale Estimation

### Functional Requirements

| Requirement | Description | Priority |
|-------------|-------------|----------|
| **View Counting** | Accurately count views, likes, watch time | Must Have |
| **Real-Time Trending** | Calculate Top K with < 5 min lag | Must Have |
| **Time Decay** | Favor newer videos over older ones | Must Have |
| **Multi-Dimensional** | Global, regional, category-based | Must Have |
| **Fraud Filtering** | Exclude bot/fraudulent views | Must Have |

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Write Throughput** | 1M events/sec | Peak viral load |
| **Read Latency** | < 100ms P99 | Fast page load |
| **Accuracy** | 99% | Genuine popularity |
| **Availability** | 99.99% | Critical feature |
| **Data Freshness** | < 5 minutes | Near real-time |

### Scale Estimation

**Write Load:**

- Total videos: 1 Billion
- Daily views: 100 Billion
- View events/sec: 100B / 86400 ≈ 1.15M events/sec
- Peak (2x): 2.3M events/sec

**Read Load:**

- Daily Active Users: 100M
- Trending page views: 2/user/day
- Trending API QPS: 200M / 86400 ≈ 2,300 QPS

**Storage:**

- Event size: 200 bytes
- Daily storage: 100B × 200 bytes = 20 TB/day
- Annual storage: 7.3 PB
- Active trending state: ~10 GB

---

## High-Level Architecture

### Architectural Pattern: Lambda Architecture

Two parallel paths:

1. **Speed Layer:** Real-time stream processing (< 5 min lag)
2. **Batch Layer:** Nightly fraud detection and accuracy correction

**Why Lambda Over Kappa?**

- Trending needs both speed AND accuracy
- Batch enables sophisticated fraud detection
- Easier to handle late events and backfill

### System Architecture

```
Client → CDN → API Gateway
    ↓
┌───────────────────────────────────────────────────────────┐
│                    SPEED LAYER                            │
│  Kafka → Flink → InfluxDB → Ranking Service → Redis      │
└───────────────────────────────────────────────────────────┘
    ↓
┌───────────────────────────────────────────────────────────┐
│                    BATCH LAYER                            │
│  S3 → Spark → Fraud Detection → Cassandra → Redis        │
└───────────────────────────────────────────────────────────┘
    ↓
Serving: Redis Sorted Sets → API → CDN → Client
```

### Key Components

1. **Kafka:** Event ingestion (1M events/sec)
2. **Flink:** Real-time aggregation (windowed counts)
3. **InfluxDB:** Time-series storage (view metrics)
4. **Ranking Service:** Score calculation (decay algorithm)
5. **Redis:** Top K storage (sorted sets, < 1ms)
6. **Spark:** Batch fraud detection
7. **Cassandra:** Historical trends
8. **S3:** Raw event logs (data lake)

---

## Detailed Component Design

### Event Ingestion

**View Event Schema:**

```json
{
  "event_id": "uuid",
  "video_id": "video_12345",
  "user_id": "user_67890",
  "timestamp": "2025-11-03T12:34:56Z",
  "event_type": "view|like|share",
  "watch_duration_sec": 120,
  "device_type": "mobile",
  "country": "US",
  "region": "California",
  "category": "Gaming",
  "ip_address": "1.2.3.4"
}
```

**Kafka Configuration:**

- Topic: `video_events`
- Partitions: 200 (by video_id hash)
- Replication: 3
- Retention: 7 days
- Compression: LZ4 (3:1)

**Partitioning:**

- Key: `video_id`
- Ensures all events for same video go to same partition
- Enables efficient aggregation

### Real-Time Aggregation (Flink)

**Pipeline Stages:**

1. **Fraud Filter:**
   - Bloom filter (known bots)
   - Rate limiting (> 100 views/min)
   - ML model scoring (risk 0-1)

2. **Windowed Aggregation:**
   - Tumbling window: 1 minute
   - Group by: video_id, country, category
   - Metrics: view_count, watch_time, unique_users

3. **InfluxDB Sink:**
   - Write aggregated metrics
   - Retention: 7 days
   - Tags: video_id, country, category

**InfluxDB Schema:**

```
measurement: video_metrics
tags: video_id, country, category
fields: view_count, watch_time, unique_users
time: 2025-11-03T12:34:00Z
```

**Why InfluxDB?**

- Optimized for time-series data
- Fast range queries (last 1 hour)
- Built-in downsampling
- High write throughput (1M+ points/sec)

### Trending Score Calculation

**Decay Function (Hacker News Algorithm):**

$$
\text{Score} = \frac{(\text{views} + \text{likes} \times 10)^{0.8}}{(\text{age}_{\text{hours}} + 2)^{1.5}}
$$

**Components:**

- **Numerator:** Weighted engagement (likes = 10× views)
- **Power 0.8:** Diminishing returns for viral videos
- **Denominator:** Age penalty (exponential decay)
- **Constant 2:** Prevents division by zero, boosts new videos

**Ranking Service:**

- Runs every 60 seconds
- Queries InfluxDB (last 6 hours)
- Calculates scores for ~10M active videos
- Updates Redis Sorted Sets

*See pseudocode.md::calculate_trending_score() for details*

### Top K Storage (Redis Sorted Sets)

**Data Structure:**

```
ZSET Key: trending:global:top100
Members: video_id
Scores: trending_score

ZSET Key: trending:US:top100
ZSET Key: trending:Gaming:top100
```

**Operations:**

- Write: `ZADD trending:global:top100 score video_id` (O(log N))
- Read: `ZREVRANGE trending:global:top100 0 99` (O(log N + M))
- Atomic updates (no race conditions)

**Why Redis Sorted Sets?**

- O(log N) insertion/retrieval
- Automatic sorting
- Sub-millisecond latency
- Built-in range queries

**Sharding:**

- Redis Cluster: 6 master nodes
- Sharded by dimension (global, country, category)
- Each shard: multiple Top K lists

### Batch Layer (Nightly Processing)

**Spark Pipeline:**

1. Read S3 Parquet files (last 24 hours)
2. Deduplication by event_id
3. Advanced fraud detection:
   - Behavioral analysis
   - IP reputation
   - ML model (XGBoost)
4. Recalculate scores with clean data
5. Write to Cassandra (historical trends)
6. Refresh Redis with corrected Top K

**Cassandra Schema:**

```sql
CREATE TABLE video_trends (
    video_id text,
    date date,
    hour int,
    country text,
    view_count bigint,
    trending_score float,
    PRIMARY KEY ((video_id, date), hour)
);
```

**Why Cassandra?**

- High write throughput
- Time-series partitioning
- Efficient range queries
- Multi-DC replication

---

## Trending Algorithm Deep Dive

### Decay Factor Analysis

**Problem:** Old viral videos dominate, blocking new content

**Solution:** Exponential time decay

**Decay Comparison:**

| Algorithm | Formula | Behavior | Use Case |
|-----------|---------|----------|----------|
| **Linear** | views - k·Δt | Slow decay | Long-term |
| **Exponential** | views·e^(-λ·Δt) | Fast decay | Viral content |
| **Hyperbolic** | views/(t+T)^α | Balanced | YouTube |

**Chosen: Hyperbolic Decay**

$$
\text{Score} = \frac{\text{engagement}^{0.8}}{(\text{age}_{\text{hours}} + 2)^{1.5}}
$$

**Parameters:**

- α = 1.5: Decay rate
- T = 2: Time constant (new video boost)
- β = 0.8: Engagement scaling

### Multi-Dimensional Ranking

**Dimensions:**

1. Global: Top 100 worldwide
2. Regional: Top 100 per country (200 countries)
3. Category: Top 100 per category (20 categories)
4. Language: Top 100 per language (50 languages)

**Storage:**

- Total lists: 1 + 200 + 20 + 50 = 271 ZSETs
- Size per list: 100 videos × 50 bytes = 5 KB
- Total memory: 271 × 5 KB = 1.35 MB

**Update Frequency:**

- Global: Every 60 seconds
- Regional: Every 120 seconds
- Category: Every 120 seconds

---

## Fraud Detection and Prevention

### Real-Time Detection

**Stage 1: Bloom Filter**

- Size: 1 billion entries
- False positive: 0.01%
- Memory: ~1.5 GB
- Lookup: O(k) constant time

**Stage 2: Rate Limiting**

- Per-user: 100 views/min
- Per-IP: 500 views/min
- Implementation: Redis INCR + EXPIRE

**Stage 3: ML Model**

- Model: Random Forest (100 trees)
- Features: Click velocity, IP reputation, watch time
- Latency: < 10ms
- Threshold: Risk > 0.7 = drop

*See pseudocode.md::fraud_detection_pipeline()*

### Batch Fraud Detection

**Advanced Analysis (Spark):**

1. Behavioral patterns (bot signatures)
2. Impossible geo patterns
3. Device fingerprinting
4. IP reputation (data centers, Tor, VPN)
5. ML model (XGBoost, 200+ features)

**Fraud Removal:**

- Mark fraudulent in Cassandra
- Recalculate scores
- Update Redis Top K

---

## Availability and Fault Tolerance

### Component Failure Impact

| Component | Impact | Mitigation | Recovery |
|-----------|--------|------------|----------|
| **Kafka** | No ingestion | 3-broker replication | < 1 min |
| **Flink** | Real-time lag | Checkpointing | < 5 min |
| **InfluxDB** | No metrics | Clustered HA | < 2 min |
| **Redis** | Trending down | Cluster + replicas | < 30 sec |
| **Ranking** | Stale trends | Multiple instances | < 1 min |

### Disaster Recovery

**Cross-Region:**

- Primary: US-East-1
- Secondary: EU-West-1
- Kafka MirrorMaker replication
- Redis cross-region sync
- Cassandra multi-DC

**Backups:**

- S3 logs: Cross-region replication
- Redis snapshots: Hourly to S3
- Cassandra snapshots: Daily to S3

---

## Bottlenecks and Optimizations

### Bottleneck 1: Ranking Computation

**Problem:**

- 10M videos every 60 seconds
- Single-threaded: 10 seconds (too slow)

**Solution:**

1. **Horizontal Scaling:**
   - 100 workers
   - Each: 100k videos
   - Parallel: 1 second

2. **Incremental Updates:**
   - Only videos with new events
   - Reduces computation 90%

3. **Category Sharding:**
   - Gaming, News, Music separate
   - Parallel computation

### Bottleneck 2: InfluxDB Writes

**Problem:**

- 1M writes/sec
- Single node: 500k max

**Solution:**

1. **Clustering:**
   - 10 data nodes
   - Sharded by video_id
   - 5M writes/sec capacity

2. **Batching:**
   - Flink buffers 1000 points
   - Reduces HTTP overhead 10x

3. **Pre-aggregation:**
   - 1-minute windows in Flink
   - 60x write reduction

### Bottleneck 3: Redis Memory

**Problem:**

- 10M active videos × 100 bytes = 1 GB

**Solution:**

1. **TTL Eviction:**
   - No views in 6 hours = evict
   - Reduces to 1M videos (100 MB)

2. **Redis Cluster:**
   - 6 master nodes
   - Sharded by dimension
   - ~200 MB per node

---

## Common Anti-Patterns

### ❌ Real-Time Sorting of All Videos

**Bad:**

```sql
SELECT * FROM videos
ORDER BY trending_score DESC
LIMIT 100
```

**Problem:** Sorting 1B videos on every request (minutes)

**Good:** Pre-compute Top K every 60 seconds in Redis

---

### ❌ No Time Decay

**Bad:** `Score = total_views` (all-time)

**Problem:** Old viral videos dominate forever

**Good:** `Score = recent_views / (age + 2)^1.5`

---

### ❌ Synchronous Fraud Detection

**Bad:** Block ingestion for fraud check (50ms)

**Problem:** Throughput: 20 events/sec

**Good:** Async fraud detection in Flink

---

### ❌ Single Redis Instance

**Bad:** Single point of failure, limited memory

**Good:** Redis Cluster (6 masters + 6 replicas)

---

## Alternative Approaches

### Alt 1: Pure Batch (Spark Every 10 Min)

**Pros:**

- Simple architecture
- High accuracy
- Lower cost

**Cons:**

- 10-minute lag (not real-time)
- Poor UX

**Decision:** Rejected (users expect real-time)

---

### Alt 2: Kappa Architecture

**Pros:**

- Single codebase
- Lower complexity
- True real-time

**Cons:**

- Harder fraud detection
- Limited reprocessing

**Decision:** Rejected (fraud requires batch)

---

### Alt 3: Cassandra for Top K Storage

**Pros:**

- High write throughput
- Horizontal scaling

**Cons:**

- No native sorted set
- Slower reads (10ms vs 1ms)

**Decision:** Rejected (Redis Sorted Sets ideal)

---

## Monitoring and Observability

### Key Metrics

| Metric | Target | Alert | Action |
|--------|--------|-------|--------|
| **Kafka Lag** | < 1 min | > 5 min | Scale Flink |
| **Flink Backpressure** | 0% | > 50% | Add workers |
| **InfluxDB Writes** | 1M/sec | < 800k | Add nodes |
| **Ranking Latency** | < 10s | > 30s | Optimize |
| **Redis Latency** | < 1ms | > 10ms | Check cluster |
| **Fraud Rate** | 10-15% | > 30% | Investigate |
| **Freshness** | < 5 min | > 10 min | Check service |

### Observability Stack

**Metrics:** Prometheus + Grafana

**Logging:** Kafka → Elasticsearch → Kibana

**Tracing:** Jaeger (distributed tracing)

**Alerting:** PagerDuty + Slack

---

## Trade-offs Summary

| Decision | Gain | Sacrifice |
|----------|------|-----------|
| **Lambda Architecture** | ✅ Speed + accuracy | ❌ Complex (2 codebases) |
| **InfluxDB** | ✅ Fast time queries | ❌ Limited to time-series |
| **Redis Sorted Sets** | ✅ O(log N) Top K | ❌ Memory-bound |
| **Hyperbolic Decay** | ✅ Balanced decay | ❌ Tuning complexity |
| **Pre-computed Top K** | ✅ Fast reads | ❌ 60-sec staleness |
| **Async Fraud** | ✅ High throughput | ❌ Eventual consistency |

---

## Real-World Examples

### YouTube Trending

**Known Details:**

- View velocity (views/hour)
- Time decay (recent content)
- Regional + global
- Manual curation
- Updates every 15 minutes

**Tech Stack (Estimated):**

- BigTable (historical)
- Spanner (metadata)
- MillWheel (streaming)
- Custom ML (fraud)

### Twitter Trending

**Differences:**

- Text-based (hashtags)
- Faster decay (hourly)
- Real-time events (breaking news)

**Similar:**

- Stream processing
- Pre-computed Top K
- Fraud detection

---

## Deployment and Infrastructure

### Kubernetes Architecture

**Clusters:**

- 3 regions (US-East, EU-West, AP-Southeast)
- 50-100 nodes each (c5.4xlarge)
- HPA for Flink, Ranking Service

**Namespaces:**

- `ingestion`: API Gateway, Kafka
- `streaming`: Flink, InfluxDB
- `ranking`: Ranking Service, Redis
- `batch`: Spark
- `storage`: Cassandra, S3
- `monitoring`: Prometheus, Grafana

### Infrastructure as Code (Terraform)

**Modules:**

```
terraform/
├── vpc/         # Network
├── eks/         # Kubernetes
├── msk/         # Kafka
├── influxdb/    # Time-series DB
├── elasticache/ # Redis
├── cassandra/   # Historical
└── s3/          # Data lake
```

### Multi-Region Strategy

| Region | Role | Components | Latency |
|--------|------|------------|---------|
| US-East-1 | Primary | Full stack + batch | < 10ms |
| EU-West-1 | Secondary | Ingestion + real-time | < 30ms |
| AP-Southeast | Secondary | Ingestion + real-time | < 30ms |

**Failover:**

- Route 53 health checks
- Auto DNS failover (< 1 min)
- Kafka MirrorMaker
- Redis cross-region sync

### CI/CD Pipeline

**Stages:**

1. GitHub Actions (code push)
2. Unit + integration tests
3. Docker build → ECR
4. Deploy to staging (ArgoCD)
5. Smoke tests
6. Canary (10% → 50% → 100%)
7. Monitor 1 hour
8. Auto-rollback on error > 0.5%

---

## Advanced Features

### Personalized Trending

**Approach:**

- Hybrid: Global + user interests
- Collaborative filtering
- Content-based filtering

**Formula:**

$$
\text{PersonalizedScore} = 0.7 \times \text{TrendingScore} + 0.3 \times \text{UserAffinity}
$$

**Implementation:**

- User embeddings (TensorFlow)
- Video embeddings
- Cosine similarity
- Cache per user (Redis, 1 hour TTL)

### Real-Time Event Detection

**Use Case:**

- Breaking news (Mars landing)
- Live events (World Cup)
- Keyword spikes

**Detection:**

- Monitor video titles/descriptions
- Anomaly detection (Z-score > 3)
- Dynamic "Event Trending" category
- Update every 1 minute

### Content Moderation

**Challenge:**

- Inappropriate content shouldn't trend
- Manual review doesn't scale

**Solution:**

- ML classification (violence, hate)
- Auto-demotion (90% score reduction)
- Manual override
- Audit log

### A/B Testing Framework

**Testing:**

- Different decay parameters
- Different time windows
- Different engagement weights

**Metrics:**

- CTR on trending page
- Session duration
- Video diversity

---

## Performance Optimization

### Kafka Optimization

**Producer:**

```
linger.ms=5
batch.size=32768
compression.type=lz4
acks=1
buffer.memory=67108864
```

**Consumer:**

```
fetch.min.bytes=1024
fetch.max.wait.ms=50
max.poll.records=5000
```

### Flink Optimization

**Parallelism:**

```
parallelism.default=200
taskmanager.numberOfTaskSlots=4
```

**State:**

```
state.backend=rocksdb
state.backend.incremental=true
execution.checkpointing.interval=30s
```

### InfluxDB Optimization

**Write:**

```
cache-max-memory-size=2147483648
cache-snapshot-write-cold-duration=10m
wal-fsync-delay=100ms
```

### Redis Optimization

**Memory:**

```
maxmemory 64gb
maxmemory-policy allkeys-lru
```

**Persistence (Disabled):**

```
save ""
appendonly no
```

---

## Interview Discussion Points

### Q1: How to handle viral video (10x views)?

**Answer:**

- Kafka scales linearly (200 partitions)
- Flink auto-scales (HPA)
- InfluxDB handles bursts (write cache)
- Redis pre-computed (no query overhead)
- CDN prevents API overload
- Bottleneck: Ranking (mitigated by incremental)

---

### Q2: Two videos with same score?

**Answer:**

- Redis allows duplicate scores
- Tie-break: Secondary sort by video_id
- Alternative: Add random noise (0.001)
- Or: Prefer newer (timestamp tie-break)

---

### Q3: Prevent fake view manipulation?

**Answer:**

- Real-time: Bloom + rate limit + ML
- Batch: Behavioral + IP reputation
- User reputation: Fraud history downweight
- Manual review: High-visibility videos
- Accept 1-2% error

---

### Q4: Scale to 10M events/sec (10x)?

**Answer:**

1. Kafka: 200 → 500 partitions, 10 → 30 brokers
2. Flink: 100 → 1000 workers
3. InfluxDB: 10 → 50 nodes
4. Redis: No change
5. Cost: $200k → $1M/month

Bottleneck: InfluxDB (may switch to Cassandra)

---

### Q5: InfluxDB goes down?

**Answer:**

- Speed layer unavailable
- Fallback: Serve stale Redis
- Batch layer unaffected
- Recovery: Replay Kafka (7 days)
- Impact: 5-10 min stale (acceptable)

---

### Scalability Discussions

**1 Billion Videos:**

- Current: 1B videos, 10M active
- Solution: Only rank videos with recent views (7 days)
- Reduces to 50M (5%)
- Pre-filter in Flink

**50 Regions:**

- Current: 3 regions
- Challenge: 50 regional Top K
- Storage: 50 × 5 KB = 250 KB (trivial)
- Compute: 50x more ranking jobs
- Solution: Shard by region

---

### Design Trade-Offs

**At 10x (10M events/sec):**

- InfluxDB → Cassandra (better writes)
- Pre-aggregate more in Flink
- Kafka Streams instead of Flink

**At 100x (100M events/sec):**

- Custom C++ processor
- Distributed Redis (shard by video_id)
- Approximate algorithms (Count-Min Sketch)
- Sample 10% (statistical significance)

**If latency critical (< 10ms):**

- Pre-load to CDN edge
- Edge computing (Top K at edge)
- Skip batch (accept fraud risk)

---

## Lessons Learned

### What Works Well

**Pre-Computed Top K:**

- Redis Sorted Sets perfect
- Sub-ms reads
- Easy scaling

**Lambda Architecture:**

- Speed + accuracy
- Fraud needs batch
- Data reprocessing

**Time Decay:**

- Hyperbolic balances old vs new
- Tunable parameters
- Prevents staleness

### Common Pitfalls

**Ignoring Fraud:**

- Bot farms manipulate
- Multi-stage essential

**No Time Decay:**

- Old videos dominate
- Must favor recent

**Real-Time Sorting:**

- Sorting 1B on request
- Pre-compute instead

### Best Practices

1. Pre-compute expensive operations
2. Use time-series databases
3. Implement time decay
4. Multi-stage fraud detection
5. Cache aggressively
6. Monitor freshness
7. A/B test parameters
8. Separate speed/batch layers