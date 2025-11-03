# YouTube Top K (Trending Algorithm) - Sequence Diagrams

## Table of Contents

1. [View Event Ingestion (Happy Path)](#1-view-event-ingestion-happy-path)
2. [View Event Ingestion (Failure Scenarios)](#2-view-event-ingestion-failure-scenarios)
3. [Real-Time Aggregation Flow](#3-real-time-aggregation-flow)
4. [Fraud Detection Flow](#4-fraud-detection-flow)
5. [Ranking Score Calculation](#5-ranking-score-calculation)
6. [Top K Update in Redis](#6-top-k-update-in-redis)
7. [Trending Page Query Flow](#7-trending-page-query-flow)
8. [Batch Processing Flow](#8-batch-processing-flow)
9. [Flink Checkpoint and Recovery](#9-flink-checkpoint-and-recovery)
10. [Redis Failover](#10-redis-failover)
11. [Multi-Dimensional Ranking Update](#11-multi-dimensional-ranking-update)
12. [Viral Video Spike Handling](#12-viral-video-spike-handling)

---

## 1. View Event Ingestion (Happy Path)

**Flow:**

Shows the complete flow of a user playing a video from client request to Kafka storage.

**Steps:**

1. User clicks play button (T=0ms)
2. Client sends HTTP POST to API Gateway (T=2ms)
3. Schema validation passes (T=3ms)
4. Event enriched with geo/device data (T=4ms)
5. Kafka producer batches and compresses (T=5ms)
6. Client receives 202 Accepted (T=5ms)
7. Event written to Kafka partition (T=15ms, durable)

**Performance:**

- Client latency: 5ms
- Kafka durability: 15ms
- Throughput: 1M events/sec sustained

```mermaid
sequenceDiagram
    participant Client
    participant API as API Gateway
    participant Validator
    participant Enrichment
    participant Producer as Kafka Producer
    participant Kafka
    
    Note over Client: User clicks play
    Client ->> API: POST /events/view<br/>video_id, user_id, timestamp
    Note over API: T=0ms
    
    API ->> Validator: Validate schema
    Validator ->> Validator: Check required fields<br/>Check data types
    Validator -->> API: Valid
    
    API ->> Enrichment: Enrich event
    Enrichment ->> Enrichment: GeoIP lookup (country)<br/>Parse user agent (device)<br/>Add timestamp
    Enrichment -->> API: Enriched event
    Note over API: T=4ms
    
    API ->> Producer: Buffer event
    Producer -->> API: Buffered (async)
    API -->> Client: 202 Accepted
    Note over Client: T=5ms (response)
    
    Producer ->> Producer: Batch 1000 events<br/>Compress with LZ4
    Producer ->> Kafka: Flush batch
    Note over Producer: T=10ms
    
    Kafka ->> Kafka: Write to partition<br/>(hash by video_id)
    Kafka ->> Kafka: Replicate (RF=3)
    Kafka -->> Producer: Ack (min.insync=2)
    Note over Kafka: T=15ms (durable)
```

---

## 2. View Event Ingestion (Failure Scenarios)

**Flow:**

Handles various failure cases during event ingestion with retries and dead letter queue.

**Failure Cases:**

1. Invalid schema: Return 400 Bad Request
2. Rate limit exceeded: Return 429 Too Many Requests
3. Kafka timeout: Retry with exponential backoff (3 attempts)
4. Final failure: Write to Dead Letter Queue

**Recovery:**

- DLQ events replayed manually
- Monitoring alerts on DLQ growth

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Kafka
    participant DLQ as Dead Letter Queue
    
    Note over Client: Scenario 1: Invalid Schema
    Client ->> API: POST /events/view<br/>(missing required fields)
    API ->> API: Validate schema
    API -->> Client: 400 Bad Request<br/>{"error": "Missing video_id"}
    
    Note over Client: Scenario 2: Rate Limit
    Client ->> API: POST /events/view<br/>(101st request in 1 min)
    API ->> API: Check rate limit (Redis)
    API -->> Client: 429 Too Many Requests<br/>Retry-After: 60
    
    Note over Client: Scenario 3: Kafka Timeout + Retry
    Client ->> API: POST /events/view (valid)
    API ->> Kafka: Write attempt 1
    Kafka -->> API: Timeout (broker down)
    
    API ->> API: Retry 1 (delay 100ms)
    API ->> Kafka: Write attempt 2
    Kafka -->> API: Timeout
    
    API ->> API: Retry 2 (delay 200ms)
    API ->> Kafka: Write attempt 3
    Kafka -->> API: Timeout
    
    API ->> DLQ: Write to DLQ<br/>(S3 + alert)
    API -->> Client: 503 Service Unavailable<br/>{"error": "Temporary failure"}
```

---

## 3. Real-Time Aggregation Flow

**Flow:**

Flink consumes events from Kafka, filters fraud, aggregates in windows, and writes to InfluxDB.

**Steps:**

1. Flink consumes batch of 1000 events (T=0s)
2. Fraud detection filters out 10-15% (T=0.5s)
3. Window aggregation (1-minute tumbling) (T=1s)
4. State update in RocksDB (T=1.5s)
5. Write to InfluxDB when window closes (T=2s)

**Performance:**

- End-to-end latency: < 2 seconds
- Throughput: 1M events/sec
- State size: 1 GB per worker

```mermaid
sequenceDiagram
    participant Kafka
    participant Flink as Flink Worker
    participant State as RocksDB State
    participant InfluxDB
    
    Note over Kafka: 1000 events in partition
    Kafka ->> Flink: Consume event batch
    Note over Flink: T=0s
    
    Flink ->> Flink: Parse JSON
    Flink ->> Flink: Fraud detection filter<br/>Bloom + ML (10% rejected)
    Note over Flink: 900 clean events
    
    Flink ->> Flink: Key by (video_id, country)<br/>Window: 1-minute tumbling
    
    Flink ->> State: Read current count
    State -->> Flink: Current: {views: 5000}
    
    Flink ->> Flink: Increment count<br/>5000 + 900 = 5900
    Flink ->> State: Write updated count
    Note over State: Persisted to disk
    
    Note over Flink: Window closes (60 sec elapsed)
    Flink ->> InfluxDB: Write metrics<br/>video_id=v123,country=US<br/>view_count=5900
    Note over InfluxDB: T=2s
    InfluxDB -->> Flink: OK
```

---

## 4. Fraud Detection Flow

**Flow:**

Multi-stage fraud detection with Bloom filter, rate limiting, and ML scoring.

**Steps:**

1. Check Bloom filter for known bots (< 1ms)
2. Check rate limit in Redis (< 1ms)
3. ML model inference (< 10ms)
4. Decision based on risk score

**Thresholds:**

- Risk 0.0-0.3: Accept (clean)
- Risk 0.3-0.7: Flag for review
- Risk 0.7-1.0: Reject (fraud)

```mermaid
sequenceDiagram
    participant Event
    participant Bloom as Bloom Filter
    participant Redis as Rate Limiter
    participant ML as ML Model
    participant Kafka
    participant FraudDB
    
    Event ->> Bloom: Check (user_id + IP)
    Bloom ->> Bloom: Hash lookup (O(k))
    
    alt Known Bot
        Bloom ->> FraudDB: Log fraud event
        Bloom -->> Event: REJECT (fraud)
    else Not in Bloom
        Bloom ->> Redis: INCR rate:user:123
        Redis -->> Bloom: Count: 45/min
        
        alt Rate > 100/min
            Bloom ->> FraudDB: Flag high frequency
            Bloom ->> ML: Score event (flagged)
        else Normal Rate
            Bloom ->> ML: Score event
        end
        
        ML ->> ML: Extract features<br/>Random Forest inference
        ML -->> Bloom: Risk score: 0.45
        
        alt Risk < 0.3 (Clean)
            Bloom ->> Kafka: ACCEPT (count normally)
        else if Risk 0.3-0.7 (Suspicious)
            Bloom ->> Kafka: ACCEPT + FLAG
            Bloom ->> FraudDB: Log for review
        else Risk > 0.7 (Fraud)
            Bloom ->> FraudDB: Log fraud
            Bloom -->> Event: REJECT
        end
    end
```

---

## 5. Ranking Score Calculation

**Flow:**

Ranking Service queries InfluxDB, calculates trending scores using hyperbolic decay, and returns Top K.

**Steps:**

1. Scheduler triggers ranking job (every 60 sec)
2. Query InfluxDB for last 6 hours of metrics
3. Calculate score for each video (hyperbolic decay)
4. Sort and return Top 100

**Formula:**

Score = (engagement^0.8) / (age + 2)^1.5

**Performance:**

- Videos processed: 10M
- Computation time: < 10 seconds
- Workers: 100 (parallel)

```mermaid
sequenceDiagram
    participant Scheduler
    participant Coordinator as Ranking Coordinator
    participant InfluxDB
    participant Worker as Worker Pool
    participant Redis
    
    Note over Scheduler: Every 60 seconds
    Scheduler ->> Coordinator: Trigger ranking job
    
    Coordinator ->> InfluxDB: Query last 6 hours<br/>SELECT video_id, SUM(views), SUM(likes)
    InfluxDB -->> Coordinator: 10M videos with metrics
    
    Coordinator ->> Coordinator: Shard videos<br/>by category
    
    Coordinator ->> Worker: Shard 1 (Gaming)<br/>100k videos
    Coordinator ->> Worker: Shard 2 (News)<br/>100k videos
    Coordinator ->> Worker: Shard 3 (Music)<br/>100k videos
    Note over Worker: 100 workers total
    
    Worker ->> Worker: For each video:<br/>engagement = views + likes*10<br/>age = now - upload_time<br/>score = engagement^0.8 / (age+2)^1.5
    
    Worker -->> Coordinator: Top 100 from shard
    
    Coordinator ->> Coordinator: Merge all shards<br/>Sort by score DESC<br/>Take top 100
    
    Coordinator ->> Redis: Update ZSET<br/>ZADD trending:global:top100
    Note over Redis: Atomic update
    
    Redis -->> Coordinator: OK
    Note over Coordinator: Job complete (10 seconds)
```

---

## 6. Top K Update in Redis

**Flow:**

Ranking Service updates multiple Redis Sorted Sets (global, regional, category) atomically.

**Steps:**

1. Ranking Service computes scores
2. Open Redis pipeline (batch mode)
3. ZADD for each dimension (global, US, Gaming, etc.)
4. ZREMRANGEBYRANK (keep only top 100)
5. Execute pipeline atomically

**Performance:**

- Dimensions updated: 271 (global + regional + category)
- Update time: < 1 second
- Operations: Pipelined (atomic)

```mermaid
sequenceDiagram
    participant Ranking as Ranking Service
    participant Pipeline as Redis Pipeline
    participant Redis as Redis Cluster
    
    Ranking ->> Pipeline: pipeline.start()
    
    Note over Ranking: Update global trending
    Ranking ->> Pipeline: ZADD trending:global:top100<br/>score1 video1<br/>score2 video2<br/>...<br/>score100 video100
    
    Note over Ranking: Update US regional
    Ranking ->> Pipeline: ZADD trending:US:top100<br/>scores for US videos
    
    Note over Ranking: Update Gaming category
    Ranking ->> Pipeline: ZADD trending:Gaming:top100<br/>scores for Gaming videos
    
    Note over Ranking: Remove lower-ranked (keep top 100)
    Ranking ->> Pipeline: ZREMRANGEBYRANK<br/>trending:global:top100 0 -101
    Ranking ->> Pipeline: ZREMRANGEBYRANK<br/>trending:US:top100 0 -101
    Ranking ->> Pipeline: ZREMRANGEBYRANK<br/>trending:Gaming:top100 0 -101
    
    Note over Ranking: Set expiration (24 hours)
    Ranking ->> Pipeline: EXPIRE trending:global:top100 86400
    
    Ranking ->> Pipeline: pipeline.execute()
    Pipeline ->> Redis: Execute all commands atomically
    Note over Redis: 271 ZSETs updated
    
    Redis -->> Ranking: All OK
    Note over Ranking: Update complete (< 1 sec)
```

---

## 7. Trending Page Query Flow

**Flow:**

User requests trending page, API Gateway queries Redis, returns Top 100 with video metadata.

**Steps:**

1. User opens trending page
2. CDN cache miss (or expired)
3. API Gateway queries Redis
4. Fetch video metadata from cache
5. Return response to client

**Performance:**

- Redis query: < 1ms
- Metadata fetch: < 5ms
- Total: < 10ms (CDN hit: < 1ms)

```mermaid
sequenceDiagram
    participant User
    participant CDN
    participant API as API Gateway
    participant Redis
    participant MetadataCache as Metadata Cache
    
    User ->> CDN: GET /trending?region=US
    CDN ->> CDN: Check cache
    
    alt Cache Hit
        CDN -->> User: Cached response (< 1ms)
    else Cache Miss
        CDN ->> API: Forward request
        
        API ->> Redis: ZREVRANGE trending:US:top100<br/>0 99 WITHSCORES
        Note over Redis: O(log N + 100)
        Redis -->> API: [video1, score1, video2, score2, ...]
        
        API ->> API: Extract video IDs
        API ->> MetadataCache: MGET video:1, video:2, ..., video:100
        MetadataCache -->> API: [title, thumbnail, creator, ...]
        
        API ->> API: Merge scores + metadata<br/>Format response JSON
        
        API -->> CDN: Response + Cache-Control: 60s
        CDN ->> CDN: Cache response
        CDN -->> User: Trending list (< 10ms)
    end
```

---

## 8. Batch Processing Flow

**Flow:**

Nightly Spark job reads S3, performs advanced fraud detection, recalculates scores, and updates Redis.

**Steps:**

1. Spark reads S3 Parquet files (last 24 hours) - T=0
2. Deduplication by event_id - T=15 min
3. Advanced fraud detection (XGBoost ML) - T=45 min
4. Recalculate scores - T=1 hour
5. Write to Cassandra (historical) - T=1.5 hours
6. Refresh Redis Top K - T=2 hours

**Performance:**

- Data processed: 20 TB
- Total time: 2 hours
- Executors: 100

```mermaid
sequenceDiagram
    participant Scheduler
    participant Spark
    participant S3
    participant MLModel as ML Model (XGBoost)
    participant Cassandra
    participant Redis
    
    Note over Scheduler: Nightly at 00:00
    Scheduler ->> Spark: Trigger batch job
    
    Spark ->> S3: Read Parquet files<br/>last 24 hours (20 TB)
    S3 -->> Spark: Stream data (distributed)
    Note over Spark: T=15 min
    
    Spark ->> Spark: Deduplication<br/>GROUP BY event_id<br/>FIRST(timestamp)
    Note over Spark: Removed duplicates: 0.1%
    
    Spark ->> MLModel: Fraud detection<br/>200+ features per event
    MLModel ->> MLModel: XGBoost inference<br/>100 events/sec per executor
    MLModel -->> Spark: Risk scores
    Note over Spark: T=45 min
    
    Spark ->> Spark: Filter fraud (risk > 0.7)<br/>Removed: 12% of events
    
    Spark ->> Spark: Aggregate by video, date<br/>Calculate trending scores
    Note over Spark: T=1 hour
    
    Spark ->> Cassandra: Write historical trends<br/>Batch inserts
    Note over Cassandra: Partitioned by date
    Cassandra -->> Spark: OK
    Note over Spark: T=1.5 hours
    
    Spark ->> Redis: Refresh Top K<br/>with corrected scores
    Redis -->> Spark: OK
    Note over Spark: T=2 hours (job complete)
```

---

## 9. Flink Checkpoint and Recovery

**Flow:**

Flink checkpoints state to S3 every 60 seconds. On failure, recovers from last checkpoint.

**Steps:**

1. JobManager triggers checkpoint
2. TaskManager snapshots RocksDB state to S3
3. Commit Kafka offsets
4. On failure, restart from last checkpoint

**Recovery:**

- Checkpoint interval: 60 seconds
- Recovery time: < 5 minutes
- State size: 1 GB per worker

```mermaid
sequenceDiagram
    participant JM as JobManager
    participant TM as TaskManager
    participant State as RocksDB State
    participant S3
    participant Kafka
    
    Note over JM: Every 60 seconds
    JM ->> TM: Trigger checkpoint#42
    
    TM ->> State: Snapshot current state<br/>(incremental)
    State -->> TM: Snapshot ready (1.2 GB)
    
    TM ->> S3: Upload checkpoint#42
    Note over S3: s3://checkpoints/job/chk-42/
    S3 -->> TM: Upload complete
    
    TM ->> Kafka: Commit offsets<br/>partition 1: offset 1234567890
    Kafka -->> TM: Committed
    
    TM -->> JM: Checkpoint#42 complete
    
    Note over TM: CRASH! (hardware failure)
    
    JM ->> JM: Detect failure (heartbeat timeout)
    JM ->> TM: Restart TaskManager
    
    TM ->> JM: Request last checkpoint
    JM -->> TM: Checkpoint#42
    
    TM ->> S3: Download checkpoint#42
    S3 -->> TM: State snapshot
    
    TM ->> State: Restore state
    TM ->> Kafka: Seek to offset 1234567890
    TM ->> Kafka: Resume consuming
    Note over TM: Recovery complete (< 5 min)
```

---

## 10. Redis Failover

**Flow:**

Redis Sentinel detects master failure, promotes replica to master, reconfigures clients.

**Steps:**

1. Sentinel detects master down (heartbeat timeout)
2. Sentinel quorum elects new master
3. Promote replica to master
4. Notify clients of new master
5. Old master becomes replica when recovered

**Downtime:**

- Detection: 5 seconds
- Promotion: 10 seconds
- Total: < 30 seconds

```mermaid
sequenceDiagram
    participant Sentinel as Redis Sentinel
    participant Master as Redis Master
    participant Replica1
    participant Replica2
    participant Client as Ranking Service
    
    Note over Master: Normal operation
    Client ->> Master: ZADD trending:global:top100
    Master -->> Client: OK
    Master ->> Replica1: Replicate
    Master ->> Replica2: Replicate
    
    Note over Master: CRASH! (hardware failure)
    
    Sentinel ->> Master: Heartbeat check
    Note over Sentinel: No response (timeout 5s)
    
    Sentinel ->> Sentinel: Quorum check<br/>(3 sentinels agree)
    Sentinel ->> Sentinel: Elect new master<br/>(Replica1: lowest lag)
    
    Sentinel ->> Replica1: SLAVEOF NO ONE
    Note over Replica1: Promoted to master
    
    Sentinel ->> Replica2: SLAVEOF new-master-ip
    Replica2 ->> Replica1: Start replicating
    
    Sentinel ->> Client: Publish master-switch event
    Client ->> Client: Reconnect to new master
    
    Client ->> Replica1: ZADD trending:global:top100
    Replica1 -->> Client: OK
    
    Note over Master: Recovers
    Sentinel ->> Master: Reconfigure as replica
    Master ->> Replica1: SLAVEOF new-master
    Master ->> Replica1: Start replicating
    
    Note over Replica1, Master: Failover complete (< 30 sec)
```

---

## 11. Multi-Dimensional Ranking Update

**Flow:**

Ranking Service updates multiple dimensions (global, regional, category) in parallel.

**Steps:**

1. Ranking Coordinator distributes work to workers
2. Each worker computes scores for its dimension
3. Workers update Redis in parallel
4. Coordinator waits for all to complete

**Dimensions:**

- 1 global
- 200 regional (by country)
- 20 category
- 50 language

**Total:** 271 Top K lists updated

```mermaid
sequenceDiagram
    participant Coordinator
    participant Global as Global Worker
    participant Regional as Regional Workers
    participant Category as Category Workers
    participant Redis
    
    Coordinator ->> Global: Compute global Top 100<br/>(10M videos)
    Coordinator ->> Regional: Compute US Top 100<br/>(1M US videos)
    Coordinator ->> Regional: Compute UK Top 100<br/>(500k UK videos)
    Coordinator ->> Category: Compute Gaming Top 100<br/>(2M Gaming videos)
    
    Note over Global, Category: Parallel execution
    
    Global ->> Global: Calculate scores<br/>Sort by score DESC<br/>Take top 100
    Regional ->> Regional: Calculate scores (US)<br/>Filter by country=US<br/>Take top 100
    Category ->> Category: Calculate scores (Gaming)<br/>Filter by category=Gaming<br/>Take top 100
    
    Global ->> Redis: ZADD trending:global:top100<br/>(100 videos)
    Regional ->> Redis: ZADD trending:US:top100<br/>(100 videos)
    Category ->> Redis: ZADD trending:Gaming:top100<br/>(100 videos)
    
    Redis -->> Global: OK
    Redis -->> Regional: OK (US)
    Redis -->> Category: OK (Gaming)
    
    Global -->> Coordinator: Complete
    Regional -->> Coordinator: Complete (200 countries)
    Category -->> Coordinator: Complete (20 categories)
    
    Note over Coordinator: All 271 lists updated (< 10 sec)
```

---

## 12. Viral Video Spike Handling

**Flow:**

Shows system behavior when a video goes viral (10x normal views).

**Steps:**

1. Video receives 10x views in 5 minutes
2. Kafka handles burst (partitions scale linearly)
3. Flink auto-scales workers (HPA)
4. InfluxDB buffers writes
5. Ranking Service detects spike and updates Redis

**Auto-Scaling:**

- Flink: 50 → 100 workers (< 2 min)
- Kafka: No scaling needed (over-provisioned)
- InfluxDB: Write cache absorbs burst

```mermaid
sequenceDiagram
    participant Video as Viral Video
    participant Kafka
    participant Flink
    participant HPA as Horizontal Pod Autoscaler
    participant InfluxDB
    participant Ranking
    participant Redis
    
    Note over Video: Video goes viral<br/>10x views (10k → 100k/min)
    
    Video ->> Kafka: 100k events/min<br/>(vs normal 10k)
    Note over Kafka: Partition absorbs burst<br/>Linear scaling
    
    Kafka ->> Flink: High throughput detected
    Flink ->> Flink: Backpressure increases<br/>(40% → 60%)
    
    Flink ->> HPA: Metrics: backpressure=60%
    HPA ->> HPA: Trigger scale-up<br/>(threshold: 50%)
    HPA ->> Flink: Scale 50 → 100 workers
    Note over Flink: Rebalance partitions<br/>(< 2 min)
    
    Flink ->> InfluxDB: Burst writes<br/>(10x normal)
    Note over InfluxDB: Write cache absorbs<br/>Batch writes to disk
    InfluxDB -->> Flink: OK (no drops)
    
    Note over Ranking: Next ranking cycle (60 sec)
    Ranking ->> InfluxDB: Query last 6 hours
    InfluxDB -->> Ranking: Video score: 9850<br/>(vs previous 850)
    
    Ranking ->> Ranking: Video jumps to#1<br/>in trending
    Ranking ->> Redis: ZADD trending:global:top100<br/>9850 video_viral123
    
    Redis -->> Ranking: OK
    Note over Redis: Video now#1 trending<br/>(within 60 sec of spike)
```