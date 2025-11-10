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

## 1. Problem Statement

Design a highly scalable, low-latency trending algorithm system for YouTube that can calculate and serve the "Top K" (
e.g., Top 100) most popular videos globally and regionally in real-time. The system must handle 1 million view events
per second, apply time-decay to favor recent content, filter fraudulent views, and serve the trending list with
sub-100ms latency.

**Key Challenges:**

- Processing 1M+ events/sec in real-time
- Calculating Top K from 1 billion videos efficiently
- Time-decay algorithm to favor recent trending content
- Multi-dimensional ranking (global, regional, category-based)
- Fraud detection at scale
- Serving trending lists with < 100ms P99 latency

---

## 2. Requirements and Scale Estimation

### Functional Requirements

| Requirement            | Description                                              | Priority     |
|------------------------|----------------------------------------------------------|--------------|
| **View Counting**      | Accurately count views, likes, watch time for all videos | Must Have    |
| **Real-Time Trending** | Calculate Top K (e.g., 100) videos with < 5 min lag      | Must Have    |
| **Time Decay**         | Favor newer videos over older ones                       | Must Have    |
| **Multi-Dimensional**  | Global, regional, category-based trending                | Must Have    |
| **Fraud Filtering**    | Exclude bot/fraudulent views                             | Must Have    |
| **Personalization**    | Optional: Personalized trending based on user interests  | Nice to Have |

### Non-Functional Requirements

| Requirement          | Target        | Rationale                        |
|----------------------|---------------|----------------------------------|
| **Write Throughput** | 1M events/sec | Peak load during viral events    |
| **Read Latency**     | < 100ms P99   | Fast page load                   |
| **Accuracy**         | 99%           | Genuine reflection of popularity |
| **Availability**     | 99.99%        | Critical user-facing feature     |
| **Data Freshness**   | < 5 minutes   | Near real-time trending          |

### Scale Estimation

**Data Volume:**

- Total videos: $1 \text{ Billion}$
- Daily views: $100 \text{ Billion}$
- View events per second: $\frac{100 \text{B}}{86400} \approx 1.15 \text{M events/sec}$
- Peak (2x average): $2.3 \text{M events/sec}$

**Read Load:**

- Daily Active Users (DAU): $100 \text{M}$
- Trending page views per user: $2 \text{/day}$
- Trending API reads: $\frac{200 \text{M}}{86400} \approx 2,300 \text{ QPS}$

**Storage:**

- Event size: $200 \text{ bytes}$
- Daily storage: $100 \text{B} \times 200 \text{ bytes} = 20 \text{ TB/day}$
- Annual storage (raw logs): $20 \text{ TB} \times 365 = 7.3 \text{ PB}$
- Active trending state: $\sim 10 \text{M videos} \times 1 \text{ KB} = 10 \text{ GB}$

**Compute:**

- Ranking computation frequency: Every $60 \text{ seconds}$
- Videos to score per iteration: $\sim 10 \text{M}$ (recent activity)
- Scoring rate: $\frac{10 \text{M}}{60} \approx 167 \text{k videos/sec}$

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

### Architectural Pattern: Lambda Architecture

We use **Lambda Architecture** with two paths:

1. **Speed Layer (Real-Time):** Stream processing for approximate trending (< 5 min lag)
2. **Batch Layer:** Nightly recomputation for accuracy and fraud correction

**Why Lambda Over Kappa?**

- Trending requires both real-time responsiveness AND accuracy
- Batch layer allows sophisticated fraud detection and score recalibration
- Easier to handle late-arriving events and backfill data

### Key Components

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
│  Web/Mobile Apps → CDN (CloudFront) → API Gateway                   │
└──────────────────────────────────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼────────────────────────────────────┐
│                            SPEED LAYER (Real-Time)                     │
│  View Events → Kafka → Flink → InfluxDB → Ranking Service → Redis    │
└────────────────────────────────────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼────────────────────────────────────┐
│                           BATCH LAYER (Nightly)                        │
│  Kafka → S3 → Spark → Fraud Detection → Cassandra → Redis Refresh    │
└────────────────────────────────────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼────────────────────────────────────┐
│                          SERVING LAYER                                 │
│  Redis Sorted Sets (ZSET) → API Gateway → CDN → Client                │
└────────────────────────────────────────────────────────────────────────┘
```

**Component Responsibilities:**

1. **Kafka:** Event backbone (1M events/sec ingestion)
2. **Flink:** Real-time aggregation (rolling window counts)
3. **InfluxDB:** Time-series storage (view counts per minute)
4. **Ranking Service:** Score calculation (decay algorithm)
5. **Redis:** Top K storage (sorted sets, < 1ms reads)
6. **Spark:** Batch processing (fraud detection, recalibration)
7. **Cassandra:** Video metadata and historical scores
8. **S3:** Data lake (raw event logs)

---

## 4. Detailed Component Design

### 4.1 Event Ingestion Layer

**View Event Schema:**

```json
{
  "event_id": "uuid",
  "video_id": "video_12345",
  "user_id": "user_67890",
  "timestamp": "2025-11-03T12:34:56Z",
  "event_type": "view|like|share|watch_time",
  "watch_duration_sec": 120,
  "device_type": "mobile|desktop|tv",
  "country": "US",
  "region": "California",
  "category": "Gaming",
  "ip_address": "1.2.3.4",
  "user_agent": "Mozilla/5.0 ..."
}
```

**Kafka Configuration:**

- Topic: `video_events`
- Partitions: 200 (by video_id hash)
- Replication factor: 3
- Retention: 7 days
- Compression: LZ4 (3:1 ratio)

**Partitioning Strategy:**

- Partition key: `video_id`
- Ensures all events for same video go to same partition
- Enables efficient aggregation in Flink

### 4.2 Real-Time Aggregation (Speed Layer)

**Flink Stream Processing Pipeline:**

1. **Fraud Filter Stage:**
    - Bloom filter check (known bots)
    - Rate limiting (> 100 views/min per user)
    - ML model scoring (risk 0-1)

2. **Windowed Aggregation:**
    - Tumbling window: 1 minute
    - Group by: `video_id`, `country`, `category`
    - Aggregations: `COUNT(views)`, `SUM(watch_time)`, `COUNT(DISTINCT user_id)`

3. **InfluxDB Sink:**
    - Write aggregated metrics
    - Measurement: `video_metrics`
    - Tags: `video_id`, `country`, `category`
    - Fields: `view_count`, `watch_time`, `unique_users`
    - Retention: 7 days (speed layer only)

**InfluxDB Schema:**

```
measurement: video_metrics
tags: video_id, country, category
fields: view_count, watch_time, unique_users, like_count
time: 2025-11-03T12:34:00Z
```

**Why InfluxDB?**

- Optimized for time-series aggregations
- Fast range queries (e.g., "last 1 hour")
- Built-in downsampling
- High write throughput (1M+ points/sec)

### 4.3 Trending Score Calculation

**Decay Function (Hacker News Algorithm):**

$$
\text{Score} = \frac{(\text{views} + \text{likes} \times 10)^{0.8}}{(\text{age}_{\text{hours}} + 2)^{1.5}}
$$

**Why This Formula?**

- Numerator: Weighted engagement (likes weighted 10x views)
- Power 0.8: Diminishing returns for very popular videos
- Denominator: Age penalty (exponential decay)
- Constant 2: Prevents division by zero, gives new videos boost

**Alternative: Reddit Hot Algorithm:**

$$
\text{Score} = \log_{10}(\text{engagement}) + \frac{\text{sign} \times \text{time}_{\text{epoch}}}{45000}
$$

**Implementation:**

- Ranking Service runs every 60 seconds
- Queries InfluxDB for last 6 hours of metrics
- Calculates score for ~10M videos (recent activity only)
- Updates Redis Sorted Sets

*See pseudocode.md::calculate_trending_score() for implementation*

### 4.4 Top K Storage (Redis Sorted Sets)

**Redis Data Structure:**

```
ZSET Key: trending:global:top100
Members: video_id
Scores: trending_score

ZSET Key: trending:US:top100
ZSET Key: trending:Gaming:top100
```

**Operations:**

- **Write:** `ZADD trending:global:top100 score video_id` (O(log N))
- **Read:** `ZREVRANGE trending:global:top100 0 99 WITHSCORES` (O(log N + M))
- **Update:** Atomic `ZADD` overwrites existing score

**Why Redis Sorted Sets?**

- O(log N) insertion and retrieval
- Automatic sorting by score
- Atomic operations (no race conditions)
- Sub-millisecond read latency
- Built-in range queries

**Sharding Strategy:**

- Redis Cluster with 6 master nodes
- Sharded by dimension: `global`, `country`, `category`
- Each shard stores multiple Top K lists

### 4.5 Batch Layer (Accuracy and Fraud Correction)

**Nightly Spark Pipeline:**

1. **Read S3 Parquet files** (last 24 hours)
2. **Deduplication** by `event_id`
3. **Advanced fraud detection:**
    - Behavioral analysis (click patterns)
    - IP reputation scoring
    - ML model (XGBoost)
4. **Recalculate scores** with clean data
5. **Write to Cassandra** (historical trends)
6. **Refresh Redis** with corrected Top K

**Cassandra Schema:**

```sql
CREATE TABLE video_trends (
    video_id text,
    date date,
    hour int,
    country text,
    category text,
    view_count bigint,
    watch_time_sec bigint,
    unique_users bigint,
    trending_score float,
    PRIMARY KEY ((video_id, date), hour, country)
) WITH CLUSTERING ORDER BY (hour DESC);
```

**Why Cassandra?**

- High write throughput (batch inserts)
- Time-series partitioning (by date)
- Efficient range queries (last N days)
- Multi-datacenter replication

---

## 5. Trending Algorithm Deep Dive

### 5.1 Decay Factor Analysis

**Problem:** Videos that went viral yesterday dominate trending, preventing new content from surfacing.

**Solution:** Exponential time decay

**Decay Options:**

| Algorithm       | Formula                                                         | Behavior   | Use Case                 |
|-----------------|-----------------------------------------------------------------|------------|--------------------------|
| **Linear**      | $\text{Score} = \text{views} - k \cdot \Delta t$                | Slow decay | Long-term trending       |
| **Exponential** | $\text{Score} = \text{views} \cdot e^{-\lambda \cdot \Delta t}$ | Fast decay | Short-term viral content |
| **Hyperbolic**  | $\text{Score} = \frac{\text{views}}{(t + T)^{\alpha}}$          | Balanced   | YouTube-like (balanced)  |

**Chosen: Hyperbolic Decay (Hacker News Style)**

$$
\text{Score} = \frac{\text{engagement}^{0.8}}{(\text{age}_{\text{hours}} + 2)^{1.5}}
$$

**Tuning Parameters:**

- $\alpha = 1.5$: Decay rate (higher = faster decay)
- $T = 2$: Time constant (gives new videos boost)
- $\beta = 0.8$: Engagement scaling (diminishing returns)

### 5.2 Multi-Dimensional Ranking

**Dimensions:**

1. **Global:** Top 100 videos worldwide
2. **Regional:** Top 100 per country (200 countries)
3. **Category:** Top 100 per category (20 categories)
4. **Language:** Top 100 per language (50 languages)
5. **Personalized:** Top 100 based on user watch history

**Storage:**

- Redis stores: $1 + 200 + 20 + 50 = 271$ separate ZSET lists
- Each list: $100$ videos $\times$ $50$ bytes = $5$ KB
- Total: $271 \times 5 \text{ KB} = 1.35 \text{ MB}$ (fits in memory)

**Update Strategy:**

- Global: Update every 60 seconds
- Regional: Update every 120 seconds (lower priority)
- Category: Update every 120 seconds
- Personalized: Compute on-demand (cache per user)

---

## 6. Fraud Detection and Prevention

### 6.1 Real-Time Fraud Detection

**Stage 1: Bloom Filter (Known Bots)**

- Bloom filter size: 1 billion entries
- False positive rate: 0.01%
- Memory: ~1.5 GB
- Lookup: O(k) where k=hash functions

**Stage 2: Rate Limiting**

- Per-user limit: 100 views/min
- Per-IP limit: 500 views/min
- Implementation: Redis INCR with EXPIRE

**Stage 3: ML Model Scoring**

- Model: Random Forest (100 trees)
- Features: Click velocity, IP reputation, user agent, watch time
- Inference latency: < 10ms
- Threshold: Risk > 0.7 = drop event

*See pseudocode.md::fraud_detection_pipeline() for implementation*

### 6.2 Batch Fraud Detection

**Advanced Detection (Spark Nightly Job):**

1. **Behavioral Analysis:**
    - Sequential view patterns (bot signature)
    - Impossible geographic patterns
    - Device fingerprinting inconsistencies

2. **IP Reputation:**
    - Check against known data centers
    - Tor exit nodes
    - VPN/proxy detection

3. **ML Model (XGBoost):**
    - 200+ features
    - Historical fraud patterns
    - User behavior clustering

**Fraud Removal:**

- Mark fraudulent views in Cassandra
- Recalculate scores without fraud
- Update Redis with corrected Top K

---

## 7. Availability and Fault Tolerance

### Component Failure Scenarios

| Component           | Impact if Failed       | Mitigation                     | Recovery Time            |
|---------------------|------------------------|--------------------------------|--------------------------|
| **Kafka**           | No ingestion           | 3-broker replication, cross-AZ | < 1 min (auto)           |
| **Flink**           | Real-time lag          | Checkpointing, auto-restart    | < 5 min                  |
| **InfluxDB**        | No metrics             | Clustered deployment, HA       | < 2 min                  |
| **Redis**           | Trending unavailable   | Redis Cluster, master-replica  | < 30 sec                 |
| **Ranking Service** | Stale trends           | Multiple instances, stateless  | < 1 min                  |
| **Cassandra**       | Historical unavailable | RF=3, multi-DC                 | Minimal (reads continue) |

### Disaster Recovery

**Cross-Region Replication:**

- Primary region: US-East-1
- Secondary region: EU-West-1
- Kafka MirrorMaker for event replication
- Redis cross-region sync (async)
- Cassandra multi-DC replication

**Data Backup:**

- S3 raw logs: Immutable, cross-region replication
- Redis snapshots: Every 1 hour to S3
- Cassandra snapshots: Daily to S3

---

## 8. Bottlenecks and Optimizations

### Bottleneck 1: Ranking Computation (CPU-Heavy)

**Problem:**

- Calculating score for 10M videos every 60 seconds
- Single-threaded: 10M / 60 = 167k videos/sec
- Each score calculation: ~1 µs
- Total: 10 seconds (not meeting 60-second deadline)

**Solution:**

1. **Horizontal Scaling:**
    - Distribute 10M videos across 100 workers
    - Each worker: 100k videos
    - Parallel computation: 1 second

2. **Incremental Updates:**
    - Only recalculate scores for videos with new events
    - Maintain "active video" set in Redis
    - Reduces computation by 90%

3. **Smart Sharding:**
    - Shard by category (Gaming, News, Music)
    - Each shard computes independently
    - Reduces contention

### Bottleneck 2: InfluxDB Write Throughput

**Problem:**

- 1M events/sec → 1M writes/sec to InfluxDB
- Single node: ~500k writes/sec max

**Solution:**

1. **Clustering:**
    - InfluxDB Enterprise with 10 data nodes
    - Sharded by video_id hash
    - Linear scaling: 10 nodes = 5M writes/sec

2. **Batching:**
    - Flink buffers 1000 points before write
    - Reduces HTTP overhead
    - Increases throughput by 10x

3. **Pre-aggregation in Flink:**
    - Aggregate to 1-minute windows
    - Reduces writes from 1M/sec to 17k/sec (60x reduction)

### Bottleneck 3: Redis Memory

**Problem:**

- 271 Top K lists × 5 KB = 1.35 MB (manageable)
- But: Storing 10M "active video" scores: 10M × 100 bytes = 1 GB

**Solution:**

1. **TTL-Based Eviction:**
    - Videos without views in 6 hours → evicted
    - Reduces active set to ~1M videos (100 MB)

2. **Redis Cluster:**
    - 6 master nodes
    - Sharded by dimension (global, regional, category)
    - Each node: ~200 MB

---

## 9. Common Anti-Patterns

### ❌ **Anti-Pattern 1: Real-Time Sorting of All Videos**

**Bad:**

```
SELECT * FROM videos
ORDER BY trending_score DESC
LIMIT 100
```

**Problem:**

- Sorting 1 billion videos on every request
- Query time: minutes
- Database overload

**Good:**

- Pre-compute Top K every 60 seconds
- Store in Redis Sorted Set
- Query time: < 1ms

---

### ❌ **Anti-Pattern 2: No Time Decay**

**Bad:**

```
Score = total_views  # All-time views
```

**Problem:**

- Old viral videos dominate forever
- No room for new trending content

**Good:**

```
Score = recent_views / (age + 2)^1.5  # Time-decayed
```

---

### ❌ **Anti-Pattern 3: Synchronous Fraud Detection**

**Bad:**

- Block event ingestion until fraud check completes
- Latency: 50ms per event
- Throughput: 20 events/sec (unacceptable)

**Good:**

- Async fraud detection in Flink pipeline
- Fast path: Accept all events (< 1ms)
- Slow path: Correct scores in batch layer

---

### ❌ **Anti-Pattern 4: Single Redis Instance**

**Bad:**

- Single point of failure
- Limited memory (256 GB)
- Limited throughput (100k QPS)

**Good:**

- Redis Cluster (6 masters + 6 replicas)
- Sharded storage (infinite memory)
- High availability (auto-failover)

---

## 10. Alternative Approaches

### Alternative 1: Pure Batch (Spark Every 10 Minutes)

**Pros:**

- Simple architecture
- High accuracy (full fraud detection)
- Lower infrastructure cost

**Cons:**

- High latency (10 minutes stale)
- Poor user experience
- Not "real-time trending"

**Decision:** Rejected. Users expect real-time trending.

---

### Alternative 2: Kappa Architecture (Stream-Only)

**Pros:**

- Single codebase (no batch layer)
- Lower complexity
- True real-time

**Cons:**

- Harder to implement fraud detection
- Difficult to backfill data
- Limited reprocessing

**Decision:** Rejected. Fraud detection requires batch sophistication.

---

### Alternative 3: Cassandra for Trending Storage

**Pros:**

- High write throughput
- Horizontal scalability

**Cons:**

- No native sorted set (manual Top K maintenance)
- Slower reads (10ms vs Redis 1ms)
- More complex sharding

**Decision:** Rejected. Redis Sorted Sets are ideal for Top K.

---

## 11. Monitoring and Observability

### Key Metrics

| Metric                   | Target  | Alert Threshold | Action                              |
|--------------------------|---------|-----------------|-------------------------------------|
| **Kafka Lag**            | < 1 min | > 5 min         | Scale Flink workers                 |
| **Flink Backpressure**   | 0%      | > 50%           | Add workers or optimize pipeline    |
| **InfluxDB Write Rate**  | 1M/sec  | < 800k/sec      | Add nodes                           |
| **Ranking Latency**      | < 10s   | > 30s           | Optimize algorithm or scale workers |
| **Redis Read Latency**   | < 1ms   | > 10ms          | Check cluster health                |
| **Fraud Detection Rate** | 10-15%  | > 30% or < 5%   | Investigate anomaly                 |
| **Top K Freshness**      | < 5 min | > 10 min        | Check ranking service               |

### Observability Stack

**Metrics:**

- Prometheus: Scrape metrics from all services
- Grafana: Dashboards for trending metrics
- Custom metrics: Trending algorithm performance

**Logging:**

- Kafka → Elasticsearch → Kibana
- Structured logging (JSON)
- Log retention: 30 days

**Tracing:**

- Jaeger for distributed tracing
- Trace view events end-to-end
- Performance bottleneck identification

**Alerting:**

- PagerDuty for critical alerts
- Slack for warnings
- Runbooks for common issues

---

## 12. Trade-offs Summary

| Decision                  | What We Gain               | What We Sacrifice             |
|---------------------------|----------------------------|-------------------------------|
| **Lambda Architecture**   | ✅ Real-time + accuracy     | ❌ Complex (two codebases)     |
| **InfluxDB**              | ✅ Fast time-series queries | ❌ Limited to time-series data |
| **Redis Sorted Sets**     | ✅ O(log N) Top K           | ❌ Memory-bound                |
| **Hyperbolic Decay**      | ✅ Balanced decay           | ❌ Tuning complexity           |
| **Pre-computed Top K**    | ✅ Fast reads (< 1ms)       | ❌ Slight staleness (60 sec)   |
| **Async Fraud Detection** | ✅ High throughput          | ❌ Eventual consistency        |

---

## 13. Real-World Examples

### YouTube Trending Algorithm

**Known Details:**

- Uses view velocity (views per hour)
- Time decay to favor recent content
- Regional and global trending
- Manual curation for controversial content
- Updates every 15 minutes

**Technology Stack (Estimated):**

- BigTable for historical data
- Spanner for metadata
- Bigtable + MillWheel for streaming
- Custom ML for fraud detection

### Twitter Trending Topics

**Differences:**

- Text-based (hashtags vs videos)
- Faster decay (trends change hourly)
- Geographic localization
- Real-time event detection (breaking news)

**Similar Patterns:**

- Stream processing for real-time counts
- Pre-computed Top K lists
- Fraud detection (bot trending)

---

## 14. References

### Related System Design Components

- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Event streaming backbone
- **[2.3.8 Apache Flink Deep Dive](../../02-components/2.3-messaging-streaming/2.3.8-apache-flink-deep-dive.md)** - Real-time stream processing
- **[2.1.11 Redis Deep Dive](../../02-components/2.1-databases/2.1.11-redis-deep-dive.md)** - Sorted sets for Top K
- **[2.1.9 Cassandra Deep Dive](../../02-components/2.1-databases/2.1.9-cassandra-deep-dive.md)** - Video metadata storage
- **[2.3.7 Apache Spark Deep Dive](../../02-components/2.3-messaging-streaming/2.3.7-apache-spark-deep-dive.md)** - Batch processing for fraud detection
- **[2.5.4 Bloom Filters](../../02-components/2.5-algorithms/2.5.4-bloom-filters.md)** - Fraud detection
- **[2.1.15 ClickHouse Deep Dive](../../02-components/2.1-databases/2.1.15-clickhouse-deep-dive.md)** - Time-series analytics (alternative to InfluxDB)
- **[2.1.4 Database Scaling](../../02-components/2.1.4-database-scaling.md)** - Database sharding strategies

### Related Design Challenges

- **[3.5.2 Ad Click Aggregator](../3.5.2-ad-click-aggregator/)** - Similar stream processing patterns, high write throughput
- **[3.4.2 News Feed](../3.4.2-news-feed/)** - Ranking algorithms, real-time aggregation
- **[3.4.4 Recommendation System](../3.4.4-recommendation-system/)** - ML-based ranking

### External Resources

- **YouTube Engineering:** [YouTube Architecture](https://www.youtube.com/intl/en-GB/howyoutubeworks/) - YouTube system design
- **Lambda Architecture:** [Nathan Marz Blog](https://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html) - Lambda architecture patterns
- **Time-Decay Algorithms:** [Reddit Hot Algorithm](https://github.com/reddit-archive/reddit/blob/master/r2/r2/lib/db/_sorts.pyx) - Time-decay ranking

### Books

- *Streaming Systems* by Tyler Akidau - Windowing, watermarks, late data handling
- *Designing Data-Intensive Applications* by Martin Kleppmann - Lambda architecture, stream processing

---

## 15. Deployment and Infrastructure

### Kubernetes Architecture

**Cluster Configuration:**

- 3 Kubernetes clusters (US-East, EU-West, AP-Southeast)
- Each cluster: 50-100 nodes (c5.4xlarge instances)
- Auto-scaling: HPA for Flink, Ranking Service

**Namespaces:**

- `ingestion`: API Gateway, Kafka
- `streaming`: Flink workers, InfluxDB
- `ranking`: Ranking Service, Redis
- `batch`: Spark executors
- `storage`: Cassandra, S3 proxies
- `monitoring`: Prometheus, Grafana

### Infrastructure as Code (Terraform)

**Modules:**

```
terraform/
├── vpc/              # Network configuration
├── eks/              # Kubernetes clusters
├── msk/              # Managed Kafka
├── influxdb/         # InfluxDB cluster
├── elasticache/      # Redis Cluster
├── cassandra/        # Managed Cassandra
└── s3/               # Data lake
```

**Key Resources:**

- VPC with 3 availability zones
- Private subnets for data processing
- Public subnets for API Gateway
- NAT gateways for outbound traffic
- VPC peering between regions

### Multi-Region Deployment

**Regional Strategy:**

| Region       | Role      | Components            | Latency |
|--------------|-----------|-----------------------|---------|
| US-East-1    | Primary   | Full stack + batch    | < 10ms  |
| EU-West-1    | Secondary | Ingestion + real-time | < 30ms  |
| AP-Southeast | Secondary | Ingestion + real-time | < 30ms  |

**Failover:**

- Route 53 health checks
- Auto DNS failover (< 1 min)
- Kafka MirrorMaker for cross-region replication
- Redis cross-region sync (async, eventual consistency)

### CI/CD Pipeline

**Pipeline Stages:**

1. Code push → GitHub Actions
2. Unit tests + integration tests
3. Docker build → ECR push
4. Deploy to staging (ArgoCD)
5. Smoke tests
6. Canary deployment (10% → 50% → 100%)
7. Monitor metrics for 1 hour
8. Auto-rollback on error rate > 0.5%

---

## 16. Advanced Features

### 16.1 Personalized Trending

**Approach:**

- Hybrid: Global trending + user interests
- Collaborative filtering (users with similar watch history)
- Content-based filtering (video metadata)

**Formula:**

$$
\text{PersonalizedScore} = 0.7 \times \text{TrendingScore} + 0.3 \times \text{UserAffinityScore}
$$

**Implementation:**

- User embedding vectors (TensorFlow)
- Video embedding vectors (metadata + content)
- Cosine similarity for affinity
- Cache per user (Redis, TTL=1 hour)

### 16.2 Real-Time Event Detection

**Use Case:**

- Breaking news (e.g., "Mars landing")
- Live events (e.g., "World Cup final")
- Sudden spikes in specific keywords

**Detection:**

- Monitor video title/description keywords
- Detect anomalies (Z-score > 3)
- Create dynamic "Event Trending" category
- Update every 1 minute (not 60 seconds)

### 16.3 Content Moderation Integration

**Challenge:**

- Inappropriate content shouldn't trend
- Manual review doesn't scale

**Solution:**

- ML content classification (violence, hate speech)
- Auto-demotion (reduce score by 90%)
- Manual override (human reviewers)
- Audit log for all demotion decisions

### 16.4 A/B Testing Framework

**Testing:**

- Different decay parameters ($\alpha$, $T$)
- Different time windows (1 hour vs 6 hours)
- Different engagement weights (likes vs watch time)

**Metrics:**

- User engagement (CTR on trending page)
- Session duration
- Video diversity (not dominated by single creator)

---

## 17. Performance Optimization

### Kafka Optimization

**Producer Tuning:**

```
linger.ms=5                     # Reduce latency
batch.size=32768                # 32 KB batches
compression.type=lz4            # Fast compression
acks=1                          # Async replication
buffer.memory=67108864          # 64 MB buffer
```

**Consumer Tuning (Flink):**

```
fetch.min.bytes=1024            # Minimum fetch
fetch.max.wait.ms=50            # Low latency
max.poll.records=5000           # Batch processing
```

### Flink Optimization

**Parallelism:**

```
parallelism.default=200         # Match Kafka partitions
taskmanager.numberOfTaskSlots=4
```

**State Backend:**

```
state.backend=rocksdb           # Disk-backed state
state.backend.incremental=true  # Incremental checkpoints
execution.checkpointing.interval=30s
```

**Memory:**

```
taskmanager.memory.process.size=16g
taskmanager.memory.managed.fraction=0.5
```

### InfluxDB Optimization

**Write Tuning:**

```
[data]
  cache-max-memory-size = 2147483648  # 2 GB cache
  cache-snapshot-write-cold-duration = "10m"
  wal-fsync-delay = "100ms"
```

**Shard Configuration:**

```
[shard-precreation]
  enabled = true
  check-interval = "10m"
  advance-period = "30m"
```

### Redis Optimization

**Memory:**

```
maxmemory 64gb
maxmemory-policy allkeys-lru
maxmemory-samples 10
```

**Persistence (Disabled for Speed):**

```
save ""                         # No RDB
appendonly no                   # No AOF
```

**Networking:**

```
tcp-backlog 511
timeout 0
tcp-keepalive 300
```

---

## 18. Interview Discussion Points

### Common Interview Questions

**Q1: How would you handle a video going viral (10x normal views)?**

**Answer:**

- Kafka partitions scale linearly (already 200 partitions)
- Flink auto-scales with more workers (HPA)
- InfluxDB handles burst writes (write cache)
- Redis pre-computed Top K (no query overhead)
- CDN caching prevents API overload
- **Bottleneck:** Ranking computation (mitigated by incremental updates)

---

**Q2: What if two videos have the exact same score?**

**Answer:**

- Redis ZSET allows duplicate scores
- Tie-breaking: Secondary sort by video_id (lexicographic)
- Alternatively: Add small random noise (0.001) to score
- Or: Prefer newer video (timestamp-based tie-break)

---

**Q3: How do you prevent manipulation (buying fake views)?**

**Answer:**

- Real-time: Bloom filter + rate limiting + ML scoring
- Batch: Advanced behavioral analysis + IP reputation
- User reputation: Accounts with fraud history downweighted
- Manual review: Trending videos above threshold reviewed by humans
- Trade-off: Some fraud slips through (accept 1-2% error)

---

**Q4: How would you scale to 10M events/sec (10x)?**

**Answer:**

1. **Kafka:** 200 → 500 partitions, 10 → 30 brokers
2. **Flink:** 100 → 1000 workers, optimize aggregation
3. **InfluxDB:** 10 → 50 nodes, pre-aggregate more
4. **Redis:** No change (reads don't increase)
5. **Cost:** $200k → $1M/month

**Bottleneck:** InfluxDB write capacity (may switch to Cassandra)

---

**Q5: What if InfluxDB goes down?**

**Answer:**

- Speed layer unavailable (no real-time trending)
- Fallback: Serve stale Redis data (last computed Top K)
- Batch layer unaffected (reads from S3)
- Recovery: Replay Kafka (7-day retention)
- Impact: 5-10 minutes of stale trending (acceptable)

---

### Scalability Discussions

**Scaling to 1 Billion Videos:**

- Current: 1 billion videos, 10M active
- Challenge: Ranking computation complexity
- Solution: Only rank videos with views in last 7 days
- Reduces active set to ~50M (5%)
- Pre-filter in Flink (maintain "active video" set)

**Scaling to 50 Regions:**

- Current: 3 regions (US, EU, AP)
- Challenge: 50 regional Top K lists
- Storage: 50 × 5 KB = 250 KB (trivial)
- Compute: 50x more ranking jobs
- Solution: Shard ranking service by region

---

### Design Trade-Offs

**What Would You Change at Different Scales?**

**At 10x scale (10M events/sec):**

- Switch InfluxDB → Cassandra (better write scaling)
- Pre-aggregate more aggressively in Flink
- Use Kafka Streams instead of Flink (simpler)

**At 100x scale (100M events/sec):**

- Custom C++ stream processor (Flink overhead too high)
- Distributed Redis (sharded by video_id, not dimension)
- Approximate algorithms (Count-Min Sketch for Top K)
- Sample 10% of events (statistical significance maintained)

**If latency was critical (< 10ms reads):**

- In-memory database (Redis already, but pre-load to CDN)
- Edge computing (compute Top K at CDN edge)
- Skip batch layer (accept fraud risk)

---

## 19. Lessons Learned

### What Works Well

**Pre-Computed Top K:**

- Redis Sorted Sets are perfect for this use case
- Sub-millisecond reads
- No query complexity
- Easy to scale

**Lambda Architecture:**

- Best of both worlds (speed + accuracy)
- Fraud detection requires batch sophistication
- Allows data reprocessing

**Time Decay:**

- Hyperbolic decay balances old vs new
- Tunable parameters (alpha, T)
- Prevents stale trending

### Common Pitfalls

**Pitfall 1: Ignoring Fraud**

- Problem: Bot farms can manipulate trending
- Lesson: Multi-stage fraud detection essential
- Real-time + batch required

**Pitfall 2: No Time Decay**

- Problem: Old viral videos dominate forever
- Lesson: Decay is core to trending definition
- Must favor recent content

**Pitfall 3: Real-Time Sorting**

- Problem: Sorting 1B videos on every request
- Lesson: Pre-compute Top K periodically
- Accept slight staleness (60 sec)

### Best Practices

1. **Pre-compute expensive operations** (Top K ranking)
2. **Use time-series databases** for time-based queries
3. **Implement time decay** for trending content
4. **Multi-stage fraud detection** (real-time + batch)
5. **Cache aggressively** (CDN + Redis)
6. **Monitor freshness** (alert on stale trending)
7. **A/B test decay parameters** (optimize engagement)
8. **Separate speed and batch layers** (different SLAs)