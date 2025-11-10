# YouTube Top K (Trending Algorithm) - Quick Overview

## Core Concept

A trending algorithm calculates and serves the top 100 trending videos globally, by country, and by category. It uses **Lambda Architecture** (real-time speed layer + batch accuracy layer) with time-decay algorithms to favor recent viral content over old popular videos.

**Key Challenge**: Process 1M view events/sec, calculate trending scores for 10M videos every 60 seconds, and serve Top K lists with <100ms latency.

---

## Requirements

### Functional
- **Event Ingestion**: View, like, share, watch_time events
- **Real-Time Aggregation**: 1-minute windows (speed layer)
- **Trending Score Calculation**: Time-decay algorithm (Hacker News style)
- **Top K Lists**: Global, country, category (Top 100 each)
- **Fraud Detection**: Filter bots, suspicious patterns

### Non-Functional
- **Throughput**: 1M events/sec (86B events/day)
- **Latency**: <100ms for Top K queries
- **Update Frequency**: Every 60 seconds (trending score recalculation)
- **Accuracy**: Real-time ~98% (acceptable), batch 100% (source of truth)
- **Scalability**: Handle 10M active videos

---

## Components

### 1. **Event Ingestion Layer**
- **Kafka Topic**: `video_events`
- **Partitions**: 200 (partitioned by video_id hash)
- **Replication**: 3 replicas
- **Retention**: 7 days
- **Compression**: LZ4 (3:1 ratio)

### 2. **Real-Time Aggregation (Speed Layer - Flink)**
- **Pipeline**:
  1. Fraud Filter Stage: Bloom filter, rate limiting, ML model
  2. Windowed Aggregation: 1-minute tumbling windows
  3. InfluxDB Sink: Write aggregated metrics
- **State Backend**: RocksDB
- **Checkpointing**: Every 60 seconds

### 3. **Time-Series Storage (InfluxDB)**
- **Measurement**: `video_metrics`
- **Tags**: `video_id`, `country`, `category`
- **Fields**: `view_count`, `watch_time`, `unique_users`, `like_count`
- **Retention**: 7 days (speed layer only)

### 4. **Trending Score Calculation (Ranking Service)**
- **Algorithm**: Hyperbolic decay (Hacker News style)
- **Formula**: `Score = (views + likes × 10)^0.8 / (age_hours + 2)^1.5`
- **Frequency**: Every 60 seconds
- **Scope**: ~10M videos (recent activity only)

### 5. **Top K Storage (Redis Sorted Sets)**
- **Data Structure**: ZSET
- **Key Pattern**: `trending:{dimension}:top100`
- **Examples**: `trending:global:top100`, `trending:US:top100`, `trending:Gaming:top100`
- **Operations**: `ZADD` (write), `ZREVRANGE` (read Top 100)

### 6. **Batch Layer (Accuracy + Fraud Correction - Spark)**
- **Nightly Pipeline**:
  1. Read S3 Parquet files (last 24 hours)
  2. Deduplication by event_id
  3. Advanced fraud detection (XGBoost)
  4. Recalculate scores with clean data
  5. Write to Cassandra (historical trends)
  6. Refresh Redis with corrected Top K

### 7. **Query Service**
- **Real-Time**: Query Redis Sorted Sets (<1ms)
- **Historical**: Query Cassandra (last N days)
- **API**: REST API for clients

---

## Architecture Flow

### Event Ingestion Flow

```
1. Client → API Gateway: POST /events {video_id, event_type, ...}
2. API Gateway → Kafka: Write to video_events topic (partition by video_id)
3. API Gateway → Client: Return 202 Accepted (<1ms, async)
4. Flink → Kafka: Consume events
5. Flink → Fraud Detection: Filter bots, suspicious patterns
6. Flink → Windowed Aggregation: 1-minute tumbling windows
7. Flink → InfluxDB: Write aggregated metrics
```

### Trending Score Calculation Flow

```
1. Ranking Service (every 60 seconds):
   a. Query InfluxDB: Get last 6 hours of metrics for ~10M videos
   b. Calculate Score: (views + likes × 10)^0.8 / (age_hours + 2)^1.5
   c. Update Redis: ZADD trending:global:top100 score video_id
   d. Repeat for all dimensions (global, country, category)
```

### Query Flow

```
1. Client → Query Service: GET /trending?dimension=global&limit=100
2. Query Service → Redis: ZREVRANGE trending:global:top100 0 99 WITHSCORES
3. Redis → Query Service: Return Top 100 video_ids with scores
4. Query Service → PostgreSQL: Fetch video metadata (title, thumbnail, etc.)
5. Query Service → Client: Return Top 100 videos
```

---

## Key Design Decisions

### 1. **Lambda Architecture** (vs Kappa)

**Why Lambda?**
- **Real-Time Speed Layer**: Fast updates (60 seconds) for dashboards
- **Batch Accuracy Layer**: 100% accurate scores (fraud correction, deduplication)
- **Separation of Concerns**: Real-time for UX, batch for accuracy

**Trade-off**: Complex (two codebases) vs unified (Kappa)

### 2. **Hyperbolic Decay Algorithm** (Hacker News Style)

**Formula**: `Score = (views + likes × 10)^0.8 / (age_hours + 2)^1.5`

**Why This Formula?**
- **Numerator**: Weighted engagement (likes weighted 10x views), power 0.8 (diminishing returns)
- **Denominator**: Age penalty (exponential decay), constant 2 (prevents division by zero, gives new videos boost)
- **Balanced**: Not too fast (exponential) or too slow (linear)

**Trade-off**: Tuning complexity vs simple algorithms

### 3. **InfluxDB for Time-Series** (vs Redis/TimescaleDB)

**Why InfluxDB?**
- **Optimized for Time-Series**: Fast range queries (last 1 hour)
- **Built-in Downsampling**: Automatic aggregation
- **High Write Throughput**: 1M+ points/sec
- **Native Time Windows**: Tumbling/sliding windows

**Trade-off**: Limited to time-series data vs general-purpose database

### 4. **Redis Sorted Sets for Top K** (vs Database Sorting)

**Why Redis Sorted Sets?**
- **O(log N) Insertion**: Fast updates
- **O(log N + M) Retrieval**: Fast Top K queries
- **Automatic Sorting**: No manual sorting needed
- **Atomic Operations**: No race conditions
- **Sub-millisecond Latency**: <1ms for Top 100

**Trade-off**: Memory-bound vs disk-based (slower)

### 5. **Pre-computed Top K** (vs Real-Time Sorting)

**Why?**
- **Fast Reads**: <1ms vs minutes (sorting 1B videos)
- **Scalable**: Pre-compute once, serve millions of queries
- **Acceptable Staleness**: 60-second delay (users don't notice)

**Trade-off**: Slight staleness (60 sec) vs real-time (too slow)

### 6. **Async Fraud Detection** (vs Synchronous)

**Why?**
- **High Throughput**: Don't block event ingestion (100ms ML model)
- **Eventual Consistency**: Acceptable for trending (not billing)
- **Batch Correction**: Fix fraud in nightly batch

**Trade-off**: Eventual consistency vs immediate blocking

---

## Trending Algorithm Deep Dive

### Decay Factor Analysis

**Problem**: Videos that went viral yesterday dominate trending, preventing new content from surfacing.

**Solution**: Exponential time decay

**Decay Options**:

| Algorithm | Formula | Behavior | Use Case |
|-----------|---------|----------|----------|
| **Linear** | `Score = views - k × Δt` | Slow decay | Long-term trending |
| **Exponential** | `Score = views × e^(-λ × Δt)` | Fast decay | Short-term viral content |
| **Hyperbolic** | `Score = views / (t + T)^α` | Balanced | YouTube-like (balanced) |

**Chosen: Hyperbolic Decay** (Hacker News Style)

**Tuning Parameters**:
- **Engagement Weight**: `likes × 10` (likes weighted 10x views)
- **Diminishing Returns**: `^0.8` (prevents super-viral videos from dominating)
- **Age Penalty**: `^1.5` (exponential decay)
- **Boost Constant**: `+2` (gives new videos initial boost)

---

## Bottlenecks & Solutions

### 1. **Ranking Computation (CPU-Heavy)**

**Problem**: Calculating score for 10M videos every 60 seconds

**Solution**:
- **Horizontal Scaling**: Distribute 10M videos across 100 workers (100k videos each)
- **Incremental Updates**: Only recalculate scores for videos with new events (reduces computation by 90%)
- **Smart Sharding**: Shard by category (Gaming, News, Music)

### 2. **InfluxDB Write Throughput**

**Problem**: 1M events/sec → 1M writes/sec to InfluxDB (single node: ~500k writes/sec max)

**Solution**:
- **Clustering**: InfluxDB Enterprise with 10 data nodes (5M writes/sec)
- **Batching**: Flink buffers 1000 points before write (10x throughput)
- **Pre-aggregation**: Aggregate to 1-minute windows (reduces writes from 1M/sec to 17k/sec)

### 3. **Redis Memory**

**Problem**: Storing 10M "active video" scores: 10M × 100 bytes = 1 GB

**Solution**:
- **TTL-Based Eviction**: Videos without views in 6 hours → evicted (reduces to ~1M videos, 100 MB)
- **Redis Cluster**: 6 master nodes, sharded by dimension (global, regional, category)

---

## Common Anti-Patterns

### ❌ **1. Real-Time Sorting of All Videos**

**Bad**:
```sql
SELECT * FROM videos ORDER BY trending_score DESC LIMIT 100
```

**Problem**: Sorting 1 billion videos on every request (minutes)

**Good**: Pre-compute Top K every 60 seconds, store in Redis Sorted Set (<1ms query)

### ❌ **2. No Time Decay**

**Bad**: `Score = total_views` (all-time views)

**Problem**: Old viral videos dominate forever, no room for new trending content

**Good**: `Score = recent_views / (age + 2)^1.5` (time-decayed)

### ❌ **3. Synchronous Fraud Detection**

**Bad**: Block event ingestion until fraud check completes (100ms ML model)

**Problem**: 3x slower event processing, becomes scaling bottleneck

**Good**: Async fraud detection, batch correction

---

## Monitoring & Observability

### Key Metrics

- **Event Ingestion Rate**: 1M events/sec
- **Flink Processing Lag**: <5 seconds
- **Ranking Computation Time**: <60 seconds (for 10M videos)
- **Top K Query Latency**: <100ms (P99)
- **InfluxDB Write Throughput**: 1M points/sec
- **Redis Memory Usage**: <80%

### Alerts

- **Event Ingestion Rate** <800k events/sec
- **Flink Processing Lag** >10 seconds
- **Ranking Computation Time** >90 seconds
- **Top K Query Latency** >200ms (P99)
- **InfluxDB Write Throughput** <500k points/sec

---

## Trade-offs Summary

| Decision | What We Gain | What We Sacrifice |
|----------|--------------|-------------------|
| **Lambda Architecture** | ✅ Real-time + accuracy | ❌ Complex (two codebases) |
| **InfluxDB** | ✅ Fast time-series queries | ❌ Limited to time-series data |
| **Redis Sorted Sets** | ✅ O(log N) Top K | ❌ Memory-bound |
| **Hyperbolic Decay** | ✅ Balanced decay | ❌ Tuning complexity |
| **Pre-computed Top K** | ✅ Fast reads (<1ms) | ❌ Slight staleness (60 sec) |
| **Async Fraud Detection** | ✅ High throughput | ❌ Eventual consistency |

---

## Real-World Examples

### YouTube Trending Algorithm

**Known Details**:
- Uses view velocity (views per hour)
- Time decay to favor recent content
- Regional and global trending
- Manual curation for controversial content
- Updates every 15 minutes

**Technology Stack (Estimated)**:
- BigTable for historical data
- Spanner for metadata
- Bigtable + MillWheel for streaming
- Custom ML for fraud detection

### Twitter Trending Topics

**Differences**:
- Text-based (hashtags vs videos)
- Faster decay (trends change hourly)
- Geographic localization
- Real-time event detection (breaking news)

**Similar Patterns**:
- Stream processing for real-time counts
- Pre-computed Top K lists
- Fraud detection (bot trending)

---

## Key Takeaways

1. **Lambda Architecture**: Real-time speed layer (InfluxDB) + batch accuracy layer (Spark)
2. **Time-Decay Algorithm**: Hyperbolic decay (Hacker News style) favors recent viral content
3. **Pre-computed Top K**: Calculate every 60 seconds, store in Redis Sorted Sets (<1ms queries)
4. **Incremental Updates**: Only recalculate scores for videos with new events (90% reduction)
5. **Fraud Detection**: Async filtering (real-time) + batch correction (accuracy)
6. **Multi-Dimensional Ranking**: Global, country, category (271 Top K lists)
7. **Storage Tiers**: InfluxDB (real-time), Redis (Top K), Cassandra (historical), S3 (archival)
8. **Horizontal Scaling**: Distribute ranking computation across 100 workers

---

## Recommended Stack

- **Event Ingestion**: Kafka (200 partitions, 3 replicas)
- **Stream Processing**: Apache Flink (RocksDB state backend)
- **Time-Series Storage**: InfluxDB Enterprise (10 data nodes)
- **Top K Storage**: Redis Cluster (6 master nodes, Sorted Sets)
- **Batch Processing**: Apache Spark (nightly pipeline)
- **Historical Storage**: Cassandra (time-series trends)
- **Cold Storage**: S3 (Parquet format)
- **Query Service**: Go/Java REST API
- **Monitoring**: Prometheus, Grafana, ELK Stack

