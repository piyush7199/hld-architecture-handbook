# Ad Click Aggregator - High-Level Design Diagrams

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Ingestion Layer Architecture](#2-ingestion-layer-architecture)
3. [Kafka Cluster Design](#3-kafka-cluster-design)
4. [Flink Stream Processing Pipeline](#4-flink-stream-processing-pipeline)
5. [Fraud Detection Flow](#5-fraud-detection-flow)
6. [Real-Time Storage Architecture](#6-real-time-storage-architecture)
7. [Two-Speed Processing Pipeline](#7-two-speed-processing-pipeline)
8. [Batch Processing Architecture](#8-batch-processing-architecture)
9. [Data Flow (End-to-End)](#9-data-flow-end-to-end)
10. [State Management in Flink](#10-state-management-in-flink)
11. [Multi-Region Deployment](#11-multi-region-deployment)
12. [Monitoring Dashboard](#12-monitoring-dashboard)

---

## 1. System Architecture Overview

**Flow Explanation:**

This diagram shows the complete end-to-end architecture of the ad click aggregator system, from event ingestion to query
serving.

**Key Components:**

1. Client layer sends HTTP POST requests with click events
2. NGINX load balancer distributes traffic to API Gateway instances
3. Kafka serves as the durable event backbone
4. Flink processes events in real-time for dashboard updates
5. Spark batch processes for accurate billing
6. S3 Data Lake stores raw events for long-term analysis

**Performance Characteristics:**

- Ingestion: 500k events/sec peak throughput
- Real-time lag: < 5 seconds from click to dashboard
- Batch latency: 24 hours for billing accuracy

```mermaid
graph TB
    Client[Ad Networks / Publishers<br/>500k clicks/sec] -->|HTTP POST| NGINX[NGINX L4 Proxy]
    NGINX -->|Round Robin| APIGateway[API Gateway Cluster<br/>20-100 instances]
    APIGateway -->|Async Produce| Kafka[Kafka Cluster<br/>100 partitions]
    Kafka -->|Stream| Flink[Flink Processing<br/>50 workers]
    Kafka -->|Hourly Dump| S3[S3 Data Lake<br/>15 PB storage]
    Flink -->|Write Counters| Redis[Redis Cluster<br/>Real-time storage]
    S3 -->|Batch Read| Spark[Spark Batch<br/>Nightly processing]
    Spark -->|Write Results| PostgreSQL[PostgreSQL<br/>Billing DB]
    Redis -->|Query| Dashboard[Real-Time Dashboard]
    PostgreSQL -->|Query| Reporting[Billing Reports]
    style Kafka fill: #f9f, stroke: #333, stroke-width: 4px
    style Flink fill: #bbf, stroke: #333, stroke-width: 2px
    style Redis fill: #fbb, stroke: #333, stroke-width: 2px
```

---

## 2. Ingestion Layer Architecture

**Flow Explanation:**

Shows how click events are accepted, validated, and buffered before being written to Kafka.

**Steps:**

1. Client sends click event via HTTP POST
2. NGINX terminates TLS and load balances
3. API Gateway validates schema and enriches data
4. Validation service checks for duplicates
5. Events buffered and batched to Kafka

**Benefits:**

- Async response (202 Accepted) = high throughput
- Validation catches malformed events early
- Batching reduces Kafka overhead

```mermaid
graph LR
    Client[Client] -->|POST /click| NGINX
    NGINX -->|Load Balance| API1[API Gateway 1]
    NGINX -->|Load Balance| API2[API Gateway 2]
    NGINX -->|Load Balance| API3[API Gateway N]
    API1 -->|Validate| Validator1[Validation Service]
    API2 -->|Validate| Validator2[Validation Service]
    API3 -->|Validate| Validator3[Validation Service]
    Validator1 -->|Batch Write| KafkaProducer[Kafka Producer<br/>Buffer 16KB]
    Validator2 -->|Batch Write| KafkaProducer
    Validator3 -->|Batch Write| KafkaProducer
    KafkaProducer -->|Flush every 10ms| Kafka[Kafka Cluster]
```

---

## 3. Kafka Cluster Design

**Flow Explanation:**

Kafka cluster configuration optimized for high-throughput write workload.

**Configuration:**

- 10 brokers across 3 availability zones
- 100 partitions (partitioned by campaign_id hash)
- Replication factor: 3
- Retention: 7 days
- Compression: LZ4 (3:1 ratio)

**Trade-offs:**

- More partitions = higher parallelism but more overhead
- 7-day retention = supports replay for late arrivals
- LZ4 compression = fast with good ratio

```mermaid
graph TB
    Producer[Kafka Producers<br/>API Gateway instances] -->|Write| Partition

    subgraph Kafka Cluster
        subgraph Broker 1
            P1[Partition 1<br/>Leader]
            P2[Partition 2<br/>Follower]
        end

        subgraph Broker 2
            P3[Partition 1<br/>Follower]
            P4[Partition 2<br/>Leader]
        end

        subgraph Broker 3
            P5[Partition 1<br/>Follower]
            P6[Partition 2<br/>Follower]
        end
    end

    P1 -->|Replicate| P3
    P1 -->|Replicate| P5
    P4 -->|Replicate| P2
    P4 -->|Replicate| P6
    P1 -->|Consume| FlinkWorker1[Flink Worker 1]
    P4 -->|Consume| FlinkWorker2[Flink Worker 2]
```

---

## 4. Flink Stream Processing Pipeline

**Flow Explanation:**

Flink pipeline that processes click events in real-time, filters fraud, aggregates counts, and writes to Redis.

**Processing Steps:**

1. Consume from Kafka (100 parallel consumers)
2. Fraud detection filter (Bloom filter + ML)
3. Windowed aggregation (5-minute tumbling windows)
4. Write to Redis (real-time counters)
5. Write to S3 (archival)

**Performance:**

- Throughput: 500k events/sec
- State size: 10M counters (1 GB per worker)
- Checkpointing: Every 60 seconds

```mermaid
graph LR
    Kafka[Kafka<br/>raw_clicks topic] -->|Consume| Source[Kafka Source<br/>Parallelism: 100]
    Source --> FraudFilter[Fraud Detection Filter<br/>Bloom Filter + ML]
    FraudFilter -->|Clean Events| Window[Windowed Aggregation<br/>5-min tumbling]
    FraudFilter -->|Fraudulent| FraudSink[Fraud Events Sink<br/>S3]
    Window -->|Aggregate| StateBackend[(RocksDB State<br/>10M counters)]
    StateBackend --> RedisSink[Redis Sink<br/>Write counters]
    StateBackend --> S3Sink[S3 Sink<br/>Archive events]
    RedisSink --> Redis[Redis Cluster]
    S3Sink --> S3[S3 Data Lake]
```

---

## 5. Fraud Detection Flow

**Flow Explanation:**

Multi-stage fraud detection pipeline that filters fraudulent clicks before counting.

**Stages:**

1. Bloom filter check (known bad IPs/user-agents)
2. Click pattern analysis (rate limiting)
3. ML model scoring (risk 0-1)
4. Decision: Clean, Suspicious, or Fraud

**Thresholds:**

- Clean (0.0-0.3): Count normally
- Suspicious (0.3-0.7): Count but flag
- Fraud (0.7-1.0): Drop immediately

```mermaid
graph TB
    Event[Click Event] --> BloomCheck{Bloom Filter<br/>Known Bad Actors?}
    BloomCheck -->|Yes| Drop[Drop Event<br/>Log to fraud DB]
    BloomCheck -->|No| PatternAnalysis[Click Pattern Analysis]
    PatternAnalysis --> RateCheck{High Frequency?<br/>> 100/min}
    RateCheck -->|Yes| Suspicious[Flag as Suspicious]
    RateCheck -->|No| MLScoring[ML Model Scoring]
    MLScoring --> RiskScore{Risk Score}
    RiskScore -->|0 . 0 - 0 . 3 Clean| CountNormal[Count Normally]
    RiskScore -->|0 . 3 - 0 . 7 Suspicious| CountFlag[Count + Flag for Review]
    RiskScore -->|0 . 7 - 1 . 0 Fraud| Drop
    CountNormal --> Redis[Write to Redis]
    CountFlag --> Redis
    Suspicious --> Redis
```

---

## 6. Real-Time Storage Architecture

**Flow Explanation:**

Redis cluster design for storing real-time counters with sub-millisecond query latency.

**Data Structures:**

- Key pattern: counter:{campaign_id}:{window}:{dimension}
- Value: Hash with clicks, unique_users, cost
- TTL: 24 hours (automatic cleanup)

**Sharding:**

- 3 master nodes + 3 replica nodes
- Sharded by campaign_id hash
- Redis Sentinel for automatic failover

```mermaid
graph TB
    Flink[Flink Workers] -->|Write| RedisProxy[Redis Cluster Proxy]

    subgraph Redis Cluster
        Master1[(Master 1<br/>campaigns 1-33k)]
        Master2[(Master 2<br/>campaigns 34k-66k)]
        Master3[(Master 3<br/>campaigns 67k-100k)]
        Replica1[(Replica 1)]
        Replica2[(Replica 2)]
        Replica3[(Replica 3)]
        Master1 -.->|Replicate| Replica1
        Master2 -.->|Replicate| Replica2
        Master3 -.->|Replicate| Replica3
    end

    RedisProxy -->|Hash Routing| Master1
    RedisProxy -->|Hash Routing| Master2
    RedisProxy -->|Hash Routing| Master3
    Dashboard[Dashboard API] -->|Query| RedisProxy
    RedisProxy -->|Read| Replica1
    RedisProxy -->|Read| Replica2
    RedisProxy -->|Read| Replica3
```

---

## 7. Two-Speed Processing Pipeline

**Flow Explanation:**

Dual-pipeline architecture: fast path for real-time dashboards, slow path for accurate billing.

**Fast Path (Real-Time):**

- Kafka → Flink → Redis
- Latency: < 5 seconds
- Accuracy: ~98% (acceptable for dashboards)

**Slow Path (Batch):**

- Kafka → S3 → Spark → PostgreSQL
- Latency: 24 hours
- Accuracy: 100% (source of truth for billing)

**Reconciliation:**

- Compare real-time vs batch daily
- Alert if discrepancy > 5%

```mermaid
graph TB
    Kafka[Kafka Event Stream] --> FastPath[FAST PATH]
    Kafka --> SlowPath[SLOW PATH]

    subgraph Fast Path Real-Time
        FastPath -->|Stream| Flink[Flink Processing<br/>< 5 sec lag]
        Flink -->|Write| Redis[Redis<br/>Real-time counters]
        Redis -->|Query| Dashboard[Live Dashboard<br/>98% accuracy]
    end

    subgraph Slow Path Batch
        SlowPath -->|Dump| S3[S3 Data Lake<br/>Hourly dumps]
        S3 -->|Read| Spark[Spark Batch<br/>Nightly processing]
        Spark -->|Dedupe + Join| PostgreSQL[PostgreSQL<br/>Billing DB]
        PostgreSQL -->|Query| Billing[Billing Reports<br/>100% accuracy]
    end

    Dashboard -.->|Compare| Reconciliation[Daily Reconciliation]
    Billing -.->|Compare| Reconciliation
```

---

## 8. Batch Processing Architecture

**Flow Explanation:**

Spark batch job that runs nightly to generate accurate billing data with full deduplication and fraud removal.

**Processing Steps:**

1. Read Parquet files from S3 (previous 24 hours)
2. Deduplication by event_id
3. Fraud removal (ML model inference)
4. Join with campaign metadata
5. Aggregate by campaign/day/dimension
6. Write to PostgreSQL billing DB

**Performance:**

- Data processed: 43.2 TB/day
- Executors: 100 (EMR Spot instances)
- Duration: ~2 hours

```mermaid
graph LR
    S3[(S3 Data Lake<br/>Parquet files)] -->|Read| SparkRead[Spark Read<br/>Distributed]
    SparkRead --> Dedupe[Deduplication<br/>Group by event_id]
    Dedupe --> FraudRemoval[Fraud Removal<br/>ML inference]
    FraudRemoval --> Join[Join Campaign Metadata<br/>campaign_id]
    Join --> Aggregate[Aggregate<br/>GROUP BY campaign, date]
    Aggregate --> Write[Write to PostgreSQL]
    Write --> PostgreSQL[(PostgreSQL<br/>Billing DB<br/>Partitioned by date)]
```

---

## 9. Data Flow (End-to-End)

**Flow Explanation:**

Complete data flow from click event to billing report, showing both real-time and batch paths.

**Timeline:**

- T+0: Click event ingested
- T+5 sec: Real-time counter updated in Redis
- T+1 hour: Event dumped to S3
- T+24 hours: Batch processing complete, billing DB updated

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant Kafka
    participant Flink
    participant Redis
    participant S3
    participant Spark
    participant PostgreSQL
    Client ->> API: POST /click (event)
    API ->> Kafka: Produce event
    API ->> Client: 202 Accepted
    Kafka ->> Flink: Consume event
    Flink ->> Flink: Fraud detection
    Flink ->> Flink: Windowed aggregation
    Flink ->> Redis: Update counter (T+5 sec)
    Kafka ->> S3: Hourly dump (T+1 hour)
    S3 ->> Spark: Batch read (T+24 hours)
    Spark ->> Spark: Dedupe + Fraud removal
    Spark ->> PostgreSQL: Write billing data
```

---

## 10. State Management in Flink

**Flow Explanation:**

How Flink manages stateful processing with RocksDB and checkpointing for fault tolerance.

**State Backend:**

- RocksDB: Disk-backed embedded database
- State size: ~1 GB per worker (10M counters)
- Checkpoints: S3 every 60 seconds

**Recovery:**

- On failure: Restart from last checkpoint
- Recovery time: < 5 minutes
- Exactly-once semantics preserved

```mermaid
graph TB
    Events[Kafka Events] -->|Process| FlinkTask[Flink Task]
    FlinkTask -->|Read/Write| LocalState[(RocksDB<br/>Local State<br/>1 GB)]
    LocalState -->|Checkpoint| S3Checkpoint[S3 Checkpoints<br/>Incremental snapshots]
    FlinkTask -->|Failure| Recovery[Recovery Process]
    Recovery -->|Restore| S3Checkpoint
    S3Checkpoint -->|Load State| LocalState
    LocalState -->|Final Result| Sink[Redis/S3 Sink]
```

---

## 11. Multi-Region Deployment

**Flow Explanation:**

Multi-region architecture for global low-latency ingestion with centralized processing.

**Strategy:**

- Regional ingestion (US-East, EU-West, AP-Southeast)
- Kafka MirrorMaker for cross-region replication
- Batch processing in primary region only
- S3 cross-region replication for DR

```mermaid
graph TB
    subgraph US-East Primary
        US_API[API Gateway US] -->|Write| US_Kafka[Kafka US<br/>Primary]
        US_Kafka -->|Stream| US_Flink[Flink US]
        US_Flink --> US_Redis[Redis US]
        US_Kafka -->|Dump| US_S3[S3 US]
        US_S3 -->|Batch| Spark[Spark Batch<br/>Primary Region Only]
    end

    subgraph EU-West Secondary
        EU_API[API Gateway EU] -->|Write| EU_Kafka[Kafka EU<br/>Secondary]
        EU_Kafka -->|Stream| EU_Flink[Flink EU]
        EU_Flink --> EU_Redis[Redis EU]
    end

    subgraph AP-Southeast Secondary
        AP_API[API Gateway AP] -->|Write| AP_Kafka[Kafka AP<br/>Secondary]
        AP_Kafka -->|Stream| AP_Flink[Flink AP]
        AP_Flink --> AP_Redis[Redis AP]
    end

    EU_Kafka -.->|MirrorMaker| US_Kafka
    AP_Kafka -.->|MirrorMaker| US_Kafka
    US_S3 -.->|Cross - Region Replication| EU_S3[S3 EU Backup]
```

---

## 12. Monitoring Dashboard

**Flow Explanation:**

Key metrics monitored for system health and performance.

**Critical Metrics:**

1. Ingestion rate (target: 500k/sec)
2. Kafka lag (target: < 1 min)
3. Flink backpressure (target: 0%)
4. Fraud rate (target: 10-20%)
5. Real-time accuracy (target: 98%+ vs batch)

```mermaid
graph TB
    subgraph Metrics Collection
        APIMetrics[API Gateway Metrics<br/>Requests/sec, Latency]
        KafkaMetrics[Kafka Metrics<br/>Throughput, Lag]
        FlinkMetrics[Flink Metrics<br/>Backpressure, State size]
        RedisMetrics[Redis Metrics<br/>Memory, Query latency]
    end

    APIMetrics -->|Push| Prometheus[Prometheus<br/>Time-series DB]
    KafkaMetrics -->|Push| Prometheus
    FlinkMetrics -->|Push| Prometheus
    RedisMetrics -->|Push| Prometheus
    Prometheus -->|Query| Grafana[Grafana Dashboards]
    Prometheus -->|Alert Rules| Alertmanager[Alertmanager]
    Alertmanager -->|Notify| PagerDuty[PagerDuty / Slack]
```

