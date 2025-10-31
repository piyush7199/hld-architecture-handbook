# Ad Click Aggregator - Sequence Diagrams

## Table of Contents

1. [Click Event Ingestion (Happy Path)](#1-click-event-ingestion-happy-path)
2. [Click Event Ingestion (Failure Scenarios)](#2-click-event-ingestion-failure-scenarios)
3. [Real-Time Aggregation Flow](#3-real-time-aggregation-flow)
4. [Fraud Detection Flow](#4-fraud-detection-flow)
5. [Batch Processing Flow](#5-batch-processing-flow)
6. [Dashboard Query Flow](#6-dashboard-query-flow)
7. [Kafka Partition Rebalancing](#7-kafka-partition-rebalancing)
8. [Flink Checkpoint and Recovery](#8-flink-checkpoint-and-recovery)
9. [Redis Failover](#9-redis-failover)
10. [Late Arrival Handling](#10-late-arrival-handling)
11. [Reconciliation Process](#11-reconciliation-process)
12. [Multi-Region Replication](#12-multi-region-replication)

---

## 1. Click Event Ingestion (Happy Path)

**Flow:** Complete flow of a click event from client to Kafka.

**Steps:**

1. Client sends POST request with click event
2. NGINX load balances to API Gateway
3. API Gateway validates schema and enriches data
4. Event buffered in Kafka producer
5. Batch flushed to Kafka every 10ms
6. Client receives 202 Accepted (async response)

**Performance:**

- Total latency: < 50ms (client to Kafka)
- Throughput: 500k events/sec peak

```mermaid
sequenceDiagram
    participant Client
    participant NGINX
    participant API as API Gateway
    participant Validator
    participant Producer as Kafka Producer
    participant Kafka
    Client ->> NGINX: POST /click (event)
    Note over Client: T=0ms
    NGINX ->> API: Forward request
    Note over NGINX: Load balance (Round-robin)
    API ->> Validator: Validate schema
    Validator ->> Validator: Check required fields
    Validator ->> Validator: Enrich (geolocation)
    Validator ->> Producer: Buffer event
    Note over Producer: Batch buffer (16 KB)<br/>Linger: 10ms
    Producer -->> API: Buffered (async)
    API -->> Client: 202 Accepted
    Note over Client: T=5ms (response)
    Producer ->> Kafka: Flush batch (10ms timer)
    Note over Producer: Batch size: 16 KB<br/>Compression: LZ4
    Kafka ->> Kafka: Write to partition
    Kafka ->> Kafka: Replicate (factor 3)
    Kafka -->> Producer: Ack (min.insync.replicas=2)
    Note over Kafka: T=50ms (durable)
```

---

## 2. Click Event Ingestion (Failure Scenarios)

**Flow:** How the system handles various failure scenarios during ingestion.

**Failure Cases:**

1. Invalid schema: Return 400 Bad Request
2. Rate limit exceeded: Return 429 Too Many Requests
3. Kafka unavailable: Buffer in memory, retry with backoff
4. Kafka full: Circuit breaker opens, reject writes

**Recovery:**

- Producer retries: 3 attempts with exponential backoff
- Circuit breaker: Opens after 10 consecutive failures
- Fallback: Drop events if buffer full (log to dead letter queue)

```mermaid
sequenceDiagram
    participant Client
    participant API as API Gateway
    participant Validator
    participant Producer as Kafka Producer
    participant Kafka
    participant DLQ as Dead Letter Queue
    Client ->> API: POST /click (invalid event)
    API ->> Validator: Validate schema
    Validator ->> Validator: Check fails (missing field)
    Validator -->> Client: 400 Bad Request
    Note over Client: Immediate failure response
    Client ->> API: POST /click (valid event)
    API ->> Validator: Validate schema
    Validator ->> Producer: Buffer event
    Producer ->> Kafka: Write attempt 1
    Kafka -->> Producer: Timeout / Error
    Producer ->> Producer: Retry 1 (delay 100ms)
    Producer ->> Kafka: Write attempt 2
    Kafka -->> Producer: Error
    Producer ->> Producer: Retry 2 (delay 200ms)
    Producer ->> Kafka: Write attempt 3
    Kafka -->> Producer: Error
    Producer ->> DLQ: Write to Dead Letter Queue
    Note over DLQ: Event saved for<br/>manual recovery
    Producer -->> API: Write failed (after retries)
    API -->> Client: 503 Service Unavailable
```

---

## 3. Real-Time Aggregation Flow

**Flow:** How Flink processes events and updates Redis counters in real-time.

**Steps:**

1. Flink consumes events from Kafka
2. Fraud detection filter applied
3. Events grouped by campaign_id + window
4. Aggregation state updated (RocksDB)
5. Window closes, results written to Redis
6. Dashboard queries Redis for real-time counts

**Latency:**

- Kafka to Flink: < 1 second
- Flink to Redis: < 2 seconds
- Total end-to-end: < 5 seconds

```mermaid
sequenceDiagram
    participant Kafka
    participant Flink as Flink Worker
    participant State as RocksDB State
    participant Redis
    participant Dashboard
    Kafka ->> Flink: Consume event batch (1000 events)
    Note over Flink: T=0sec
    Flink ->> Flink: Fraud detection filter
    Note over Flink: Bloom filter check<br/>ML scoring
    Flink ->> Flink: Group by (campaign_id, window)
    Note over Flink: 5-minute tumbling window
    Flink ->> State: Read current count
    State -->> Flink: Current: 1000 clicks
    Flink ->> Flink: Increment: 1000 + 100 = 1100
    Flink ->> State: Write updated count
    Note over Flink: Window closes (5 min elapsed)
    Flink ->> Redis: HINCRBY counter:campaign_123:5min 1100
    Note over Redis: Atomic increment<br/>T=5sec
    Redis -->> Flink: OK
    Dashboard ->> Redis: GET counter:campaign_123:5min
    Redis -->> Dashboard: {"clicks": 1100, "cost": 550.00}
```

---

## 4. Fraud Detection Flow

**Flow:** Multi-stage fraud detection process applied to each click event.

**Detection Stages:**

1. Bloom filter (known bad actors)
2. Click pattern analysis (rate limiting)
3. ML model scoring
4. Decision: Clean, Suspicious, or Fraud

**Actions:**

- Clean (0.0-0.3): Count normally
- Suspicious (0.3-0.7): Count but flag for review
- Fraud (0.7-1.0): Drop immediately, log to fraud DB

```mermaid
sequenceDiagram
    participant Event as Click Event
    participant Bloom as Bloom Filter
    participant Pattern as Pattern Analyzer
    participant ML as ML Model
    participant FraudDB as Fraud DB
    participant Counter as Counter Update
    Event ->> Bloom: Check IP + User-Agent
    Bloom ->> Bloom: Hash lookup (O(1))

    alt Known Bad Actor
        Bloom ->> FraudDB: Log fraud event
        Bloom -->> Event: DROP (fraud detected)
    else Not in Bloom Filter
        Bloom ->> Pattern: Analyze click pattern
        Pattern ->> Pattern: Check rate (clicks/min)
        Pattern ->> Pattern: Check sequential pattern

        alt High Frequency (> 100/min)
            Pattern ->> FraudDB: Flag as suspicious
            Pattern ->> ML: Score event
        else Normal Pattern
            Pattern ->> ML: Score event
        end

        ML ->> ML: Feature extraction
        ML ->> ML: Random Forest inference
        ML -->> Pattern: Risk score (0.0-1.0)

        alt Risk < 0.3 (Clean)
            Pattern ->> Counter: Count normally
        else Risk 0.3-0.7 (Suspicious)
            Pattern ->> Counter: Count + flag
            Pattern ->> FraudDB: Log for review
        else Risk > 0.7 (Fraud)
            Pattern ->> FraudDB: Log fraud
            Pattern -->> Event: DROP
        end
    end
```

---

## 5. Batch Processing Flow

**Flow:** Nightly Spark batch job for accurate billing with deduplication and fraud removal.

**Steps:**

1. Read S3 Parquet files (last 24 hours)
2. Deduplication by event_id
3. Fraud removal (ML model batch inference)
4. Join with campaign metadata
5. Aggregate by campaign/day
6. Write to PostgreSQL billing DB

**Duration:** ~2 hours for 43.2 TB data

```mermaid
sequenceDiagram
    participant S3
    participant Spark as Spark Executor
    participant ML as ML Model
    participant CampaignDB as Campaign Metadata
    participant PostgreSQL
    Note over Spark: Nightly job starts (T=00:00)
    Spark ->> S3: Read Parquet files (24 hours)
    Note over S3: 43.2 TB data<br/>Partitioned by hour
    S3 -->> Spark: Stream data (distributed read)
    Spark ->> Spark: Deduplication (GROUP BY event_id)
    Note over Spark: Remove duplicates<br/>Keep first occurrence
    Spark ->> ML: Batch fraud detection
    Note over ML: 100 events/sec inference<br/>on 100 executors
    ML -->> Spark: Fraud flags
    Spark ->> Spark: Filter out fraudulent events
    Spark ->> CampaignDB: Fetch campaign metadata
    CampaignDB -->> Spark: Campaign info (name, advertiser_id)
    Spark ->> Spark: Join events with metadata
    Spark ->> Spark: Aggregate (GROUP BY campaign_id, date)
    Note over Spark: SUM(clicks), COUNT(DISTINCT user_id)
    Spark ->> PostgreSQL: Write aggregated results
    Note over PostgreSQL: Partitioned table<br/>by date
    PostgreSQL -->> Spark: Write complete
    Note over Spark: Job complete (T=02:00)
```

---

## 6. Dashboard Query Flow

**Flow:** How dashboard queries Redis for real-time campaign statistics.

**Query Types:**

1. Single campaign: GET counter:campaign_123:5min
2. Multiple dimensions: MGET with pattern
3. Aggregation: Pipeline multiple commands

**Performance:**

- Single key: < 1ms
- Multi-key (100 keys): < 10ms
- Pipeline (1000 keys): < 50ms

```mermaid
sequenceDiagram
    participant User
    participant Dashboard as Dashboard UI
    participant API as Dashboard API
    participant Redis as Redis Cluster
    participant Fallback as PostgreSQL (Fallback)
    User ->> Dashboard: View campaign stats
    Dashboard ->> API: GET /campaigns/123/stats?window=5min
    API ->> Redis: GET counter:campaign_123:5min:total
    Redis -->> API: {"clicks": 15234, "cost": 7617.00}
    API ->> Redis: MGET counter:campaign_123:5min:US, counter:campaign_123:5min:UK
    Note over Redis: Pipeline request<br/>(batch 100 keys)
    Redis -->> API: [{"clicks": 8000}, {"clicks": 3000}]

    alt Redis Hit
        API -->> Dashboard: Return real-time stats
        Dashboard -->> User: Display (< 100ms)
    else Redis Miss (Key Expired)
        API ->> Fallback: Query PostgreSQL
        Fallback -->> API: Historical stats
        API -->> Dashboard: Return fallback data
        Dashboard -->> User: Display (stale data indicator)
    end
```

---

## 7. Kafka Partition Rebalancing

**Flow:** What happens when Kafka partitions rebalance due to consumer changes.

**Trigger Events:**

- Flink worker added/removed
- Flink worker crashes
- Kafka partition added

**Impact:**

- Brief processing pause during rebalance
- State reassignment
- Resume from last committed offset

```mermaid
sequenceDiagram
    participant ZK as Zookeeper
    participant Kafka
    participant Worker1 as Flink Worker 1
    participant Worker2 as Flink Worker 2
    participant Worker3 as Flink Worker 3 (NEW)
    Note over Worker1, Worker2: Normal processing<br/>Worker1: P1-P50<br/>Worker2: P51-P100
    Worker3 ->> ZK: Register as consumer
    Note over Worker3: New worker joins
    ZK ->> Kafka: Trigger rebalance
    Kafka ->> Worker1: Stop consuming
    Kafka ->> Worker2: Stop consuming
    Note over Worker1, Worker2: Checkpoint in-flight state
    Worker1 ->> Kafka: Commit offsets P1-P50
    Worker2 ->> Kafka: Commit offsets P51-P100
    Kafka ->> ZK: Request partition assignment
    ZK ->> ZK: Calculate new assignment
    Note over ZK: Rebalance algorithm:<br/>Worker1: P1-P33<br/>Worker2: P34-P66<br/>Worker3: P67-P100
    ZK ->> Worker1: Assign P1-P33
    ZK ->> Worker2: Assign P34-P66
    ZK ->> Worker3: Assign P67-P100
    Worker1 ->> Kafka: Resume from P1 offset
    Worker2 ->> Kafka: Resume from P34 offset
    Worker3 ->> Kafka: Resume from P67 offset
    
    Note over Worker1, Worker3: Processing resumed<br/>(Rebalance took ~10 seconds)
```

---

## 8. Flink Checkpoint and Recovery

**Flow:** How Flink checkpoints state for fault tolerance and recovers from failures.

**Checkpoint Process:**

1. Flink JobManager triggers checkpoint (every 60 seconds)
2. Workers snapshot state to RocksDB
3. Incremental snapshot uploaded to S3
4. Kafka offsets committed
5. Checkpoint complete

**Recovery:**

- Worker crashes
- Restart from last successful checkpoint
- Restore state from S3
- Resume from committed Kafka offset

```mermaid
sequenceDiagram
    participant JM as JobManager
    participant TM as TaskManager
    participant State as RocksDB State
    participant S3
    participant Kafka
    Note over JM: Checkpoint interval: 60s
    JM ->> TM: Trigger checkpoint#42
    TM ->> State: Snapshot current state
    Note over State: Incremental snapshot<br/>(only changed keys)
    State -->> TM: Snapshot ready (1.2 GB)
    TM ->> S3: Upload checkpoint#42
    Note over S3: s3://checkpoints/<br/>job-id/chk-42/
    TM ->> Kafka: Commit offsets
    Note over Kafka: Offset: 1,234,567,890
    TM -->> JM: Checkpoint#42 complete
    Note over TM: CRASH! (hardware failure)
    JM ->> JM: Detect failure (heartbeat timeout)
    JM ->> TM: Restart TaskManager
    TM ->> JM: Request last checkpoint
    JM -->> TM: Checkpoint#42
    TM ->> S3: Download checkpoint#42
    S3 -->> TM: State snapshot
    TM ->> State: Restore state
    TM ->> Kafka: Seek to offset 1,234,567,890
    TM ->> Kafka: Resume consuming
    Note over TM: Recovery complete<br/>(downtime: ~2 minutes)
```

---

## 9. Redis Failover

**Flow:** Redis Sentinel-managed automatic failover when master fails.

**Failover Process:**

1. Sentinel detects master down (heartbeat timeout)
2. Sentinel elects new master from replicas
3. Sentinel reconfigures other replicas
4. Clients reconnect to new master
5. Old master rejoins as replica (when recovered)

**Downtime:** < 30 seconds

```mermaid
sequenceDiagram
    participant Sentinel as Redis Sentinel
    participant Master as Redis Master
    participant Replica1 as Redis Replica 1
    participant Replica2 as Redis Replica 2
    participant Client as Flink Worker
    Note over Master: Normal operation
    Client ->> Master: HINCRBY counter:campaign_123
    Master -->> Client: OK
    Master ->> Replica1: Replicate
    Master ->> Replica2: Replicate
    Note over Master: CRASH! (hardware failure)
    Sentinel ->> Master: Heartbeat check
    Note over Sentinel: No response (timeout 5s)
    Sentinel ->> Sentinel: Quorum check (3 sentinels agree)
    Sentinel ->> Sentinel: Elect new master
    Note over Sentinel: Replica1 chosen<br/>(lowest replication lag)
    Sentinel ->> Replica1: SLAVEOF NO ONE
    Note over Replica1: Promoted to master
    Sentinel ->> Replica2: SLAVEOF new-master-ip
    Replica2 ->> Replica1: Start replicating
    Sentinel ->> Client: Publish master-switch event
    Client ->> Client: Reconnect to new master
    Client ->> Replica1: HINCRBY counter:campaign_123
    Replica1 -->> Client: OK
    Note over Master: Recovers
    Sentinel ->> Master: Reconfigure as replica
    Master ->> Replica1: SLAVEOF new-master-ip
    Master ->> Replica1: Start replicating
    
    Note over Replica1, Master: Failover complete<br/>(downtime: 15 seconds)
```

---

## 10. Late Arrival Handling

**Flow:** How the system handles events that arrive late (delayed network, clock skew).

**Strategy:**

- Real-time: Use watermarks (5-minute grace period)
- Batch: Reprocess last 7 days nightly to catch late arrivals
- Trade-off: Real-time slightly inaccurate, batch is source of truth

```mermaid
sequenceDiagram
    participant Client
    participant Kafka
    participant Flink
    participant Redis
    participant S3
    participant Spark
    Note over Client: Event timestamp: 12:00:00
    Note over Client: Network delay: 10 minutes
    Client ->> Kafka: Event arrives at 12:10:00
    Note over Kafka: Event timestamp (event-time): 12:00:00<br/>Ingestion timestamp (processing-time): 12:10:00
    Kafka ->> Flink: Consume event (12:10:00)
    Flink ->> Flink: Check watermark
    Note over Flink: Current watermark: 12:05:00<br/>Event time: 12:00:00<br/>Within grace period!
    Flink ->> Flink: Assign to correct window (12:00-12:05)
    Note over Flink: Window already closed<br/>but within grace period
    Flink ->> Redis: Update counter (retroactive)
    Redis -->> Flink: OK
    Note over Flink: Late event counted in real-time
    Kafka ->> S3: Event archived to S3
    Note over Spark: Nightly batch job
    S3 ->> Spark: Read events (last 7 days)
    Note over Spark: Includes all late arrivals<br/>up to 7 days
    Spark ->> Spark: Reprocess with correct timestamps
    Note over Spark: 100% accurate billing
```

---

## 11. Reconciliation Process

**Flow:** Daily reconciliation between real-time (Redis) and batch (PostgreSQL) to detect discrepancies.

**Process:**

1. Compare real-time vs batch counts
2. Calculate discrepancy percentage
3. Alert if discrepancy > 5%
4. Investigate root cause

**Common Causes:**

- Late arrivals not yet in batch
- Redis evictions
- Flink lag during high load

```mermaid
sequenceDiagram
    participant Scheduler
    participant Reconciliation as Reconciliation Service
    participant Redis
    participant PostgreSQL
    participant Alert as Alerting
    Note over Scheduler: Daily at 02:00 (after batch job)
    Scheduler ->> Reconciliation: Trigger reconciliation
    Reconciliation ->> Redis: GET counter:campaign_123:daily:2025-10-30
    Redis -->> Reconciliation: Real-time: 1,000,000 clicks
    Reconciliation ->> PostgreSQL: SELECT clicks FROM stats WHERE campaign_id=123 AND date='2025-10-30'
    PostgreSQL -->> Reconciliation: Batch: 980,000 clicks
    Reconciliation ->> Reconciliation: Calculate discrepancy
    Note over Reconciliation: |1M - 980k| / 980k = 2.04%

    alt Discrepancy < 5%
        Reconciliation ->> Reconciliation: Log: OK (within threshold)
    else Discrepancy >= 5%
        Reconciliation ->> Alert: Send alert (high discrepancy)
        Note over Alert: PagerDuty + Slack notification
        Reconciliation ->> Reconciliation: Investigate causes
        Note over Reconciliation: Check: Flink lag, Redis evictions,<br/>late arrivals, fraud rate
    end

    Reconciliation -->> Scheduler: Reconciliation complete
```

---

## 12. Multi-Region Replication

**Flow:** How events are replicated across regions for global low-latency ingestion.

**Strategy:**

- Regional ingestion (US, EU, AP)
- Kafka MirrorMaker 2 for cross-region replication
- Centralized batch processing in primary region

**Latency:**

- Regional ingestion: < 50ms
- Cross-region replication: < 5 seconds
- Trade-off: Slight delay in cross-region visibility

```mermaid
sequenceDiagram
    participant US_Client as US Client
    participant US_API as US API Gateway
    participant US_Kafka as US Kafka
    participant EU_Kafka as EU Kafka
    participant MirrorMaker as Kafka MirrorMaker 2
    participant US_Flink as US Flink
    participant EU_Flink as EU Flink
    participant US_S3 as US S3 (Primary)
    US_Client ->> US_API: POST /click (US region)
    US_API ->> US_Kafka: Write event
    Note over US_Kafka: T=0ms (local write)
    US_Kafka ->> US_Flink: Process (local)
    US_Kafka ->> MirrorMaker: Replicate to EU
    MirrorMaker ->> EU_Kafka: Write replicated event
    Note over EU_Kafka: T=50ms (cross-region)
    EU_Kafka ->> EU_Flink: Process (EU dashboard)
    US_Kafka ->> US_S3: Archive to S3
    Note over US_S3: Batch processing<br/>primary region only
```