# YouTube Top K (Trending Algorithm) - High-Level Design

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Lambda Architecture Pattern](#2-lambda-architecture-pattern)
3. [Event Ingestion Pipeline](#3-event-ingestion-pipeline)
4. [Kafka Cluster Design](#4-kafka-cluster-design)
5. [Flink Stream Processing](#5-flink-stream-processing)
6. [InfluxDB Time-Series Storage](#6-influxdb-time-series-storage)
7. [Ranking Service Architecture](#7-ranking-service-architecture)
8. [Redis Top K Storage](#8-redis-top-k-storage)
9. [Batch Processing Pipeline](#9-batch-processing-pipeline)
10. [Fraud Detection Flow](#10-fraud-detection-flow)
11. [Multi-Dimensional Trending](#11-multi-dimensional-trending)
12. [Multi-Region Deployment](#12-multi-region-deployment)

---

## 1. System Architecture Overview

**Flow Explanation:**

This diagram shows the end-to-end architecture for YouTube trending algorithm with Lambda pattern (speed + batch layers).

**Key Components:**

1. Client requests go through CDN and API Gateway
2. View events are ingested via Kafka (1M events/sec)
3. Speed layer processes events in real-time (Flink → InfluxDB → Redis)
4. Batch layer corrects fraud nightly (Spark → Cassandra → Redis)
5. Serving layer uses Redis Sorted Sets for sub-ms reads

**Benefits:**

- Real-time trending (< 5 min lag)
- High accuracy (batch fraud correction)
- Low latency reads (< 1ms from Redis)

**Trade-offs:**

- Complex architecture (two processing paths)
- Slight staleness in speed layer (acceptable)

```mermaid
graph TB
    Client[Web/Mobile Clients<br/>100M DAU]
    CDN[CloudFront CDN<br/>Edge caching]
    API[API Gateway<br/>Rate limiting]
    
    Client --> CDN
    CDN --> API
    
    subgraph Speed Layer Real-Time
        Kafka[Kafka Cluster<br/>200 partitions<br/>1M events/sec]
        Flink[Flink Stream Processing<br/>50 workers<br/>Fraud filter + Aggregation]
        InfluxDB[InfluxDB Time-Series<br/>10 nodes<br/>View metrics]
        Ranking[Ranking Service<br/>Score calculation<br/>Every 60 sec]
        
        API --> Kafka
        Kafka --> Flink
        Flink --> InfluxDB
        InfluxDB --> Ranking
    end
    
    subgraph Batch Layer Nightly
        S3[S3 Data Lake<br/>7.3 PB annually<br/>Raw events]
        Spark[Spark Batch<br/>100 executors<br/>Fraud detection]
        Cassandra[Cassandra<br/>Historical trends<br/>RF=3]
        
        Kafka --> S3
        S3 --> Spark
        Spark --> Cassandra
    end
    
    subgraph Serving Layer
        Redis[Redis Cluster<br/>6 masters + 6 replicas<br/>Top K sorted sets]
        
        Ranking --> Redis
        Spark --> Redis
        Redis --> API
    end
    
    style Kafka fill:#f9f,stroke:#333,stroke-width:2px
    style Redis fill:#fbb,stroke:#333,stroke-width:2px
    style Flink fill:#bbf,stroke:#333,stroke-width:2px
```

---

## 2. Lambda Architecture Pattern

**Flow Explanation:**

Shows the Lambda architecture with two parallel processing paths: speed (real-time) and batch (accuracy).

**Steps:**

1. Events flow into both speed and batch layers simultaneously
2. Speed layer provides low-latency approximate results
3. Batch layer provides high-accuracy corrected results
4. Serving layer merges both views

**Performance:**

- Speed layer: < 5 seconds lag
- Batch layer: 24 hours lag
- Accuracy: 98% (speed), 100% (batch)

```mermaid
graph LR
    Events[View Events<br/>1M/sec]
    
    subgraph Speed Layer
        StreamKafka[Kafka Stream]
        StreamProcess[Flink Processing]
        StreamStore[InfluxDB]
        StreamRanking[Ranking Service]
    end
    
    subgraph Batch Layer
        BatchS3[S3 Archive]
        BatchProcess[Spark Processing]
        BatchStore[Cassandra]
        BatchRanking[Batch Ranking]
    end
    
    subgraph Serving Layer
        Redis[Redis Sorted Sets]
        API[API Gateway]
    end
    
    Events --> StreamKafka
    Events --> BatchS3
    
    StreamKafka --> StreamProcess
    StreamProcess --> StreamStore
    StreamStore --> StreamRanking
    StreamRanking --> Redis
    
    BatchS3 --> BatchProcess
    BatchProcess --> BatchStore
    BatchStore --> BatchRanking
    BatchRanking --> Redis
    
    Redis --> API
```

---

## 3. Event Ingestion Pipeline

**Flow Explanation:**

View events from clients are validated, enriched, and batched before being sent to Kafka.

**Steps:**

1. Client sends view event (play button clicked)
2. API Gateway validates schema and rate limits
3. Enrichment service adds geo, device info
4. Kafka producer batches and compresses
5. Events written to Kafka topic

**Performance:**

- Ingestion latency: < 10ms P99
- Throughput: 1M events/sec sustained
- Compression ratio: 3:1 (LZ4)

```mermaid
graph LR
    Client[Client<br/>Play video] -->|HTTP POST| LB[Load Balancer<br/>NGINX]
    LB --> API1[API Gateway 1]
    LB --> API2[API Gateway 2]
    LB --> API3[API Gateway N]
    
    API1 --> Validate[Schema Validator]
    API2 --> Validate
    API3 --> Validate
    
    Validate --> Enrich[Enrichment Service<br/>Add geo, device]
    
    Enrich --> Producer[Kafka Producer<br/>Batch + compress]
    
    Producer --> Kafka[Kafka Cluster<br/>Topic: video_events]
```

---

## 4. Kafka Cluster Design

**Configuration:**

- 10 brokers across 3 availability zones
- 200 partitions (sharded by video_id hash)
- Replication factor: 3
- Retention: 7 days

**Trade-offs:**

- More partitions = higher parallelism but more overhead
- 7-day retention supports replay and late arrivals

```mermaid
graph TB
    Producer[Kafka Producers] -->|video_id hash| Router[Partition Router]
    
    subgraph Kafka Cluster
        subgraph AZ-1
            B1[Broker 1<br/>P1-P66 leaders]
            B2[Broker 2<br/>P67-P133 leaders]
        end
        
        subgraph AZ-2
            B3[Broker 3<br/>P134-P200 leaders]
            B4[Broker 4<br/>P1-P66 replicas]
        end
        
        subgraph AZ-3
            B5[Broker 5<br/>P67-P133 replicas]
            B6[Broker 6<br/>P134-P200 replicas]
        end
    end
    
    Router --> B1
    Router --> B2
    Router --> B3
    
    B1 -.->|Replicate| B4
    B2 -.->|Replicate| B5
    B3 -.->|Replicate| B6
    
    B1 --> Consumer1[Flink Worker 1]
    B2 --> Consumer2[Flink Worker 2]
    B3 --> Consumer3[Flink Worker 3]
```

---

## 5. Flink Stream Processing

**Flow Explanation:**

Shows the complete Flink stream processing pipeline with fraud detection, windowed aggregation, and multiple sinks.

**Steps:**

1. Consume events from Kafka (200 parallel consumers)
2. Parse JSON and validate schema
3. Fraud detection filter (Bloom filter + ML)
4. Windowed aggregation (1-minute tumbling windows)
5. Write to InfluxDB (metrics) and S3 (archival)

**Performance:**

- Throughput: 1M events/sec
- Latency: < 2 seconds
- State size: 1 GB per worker

```mermaid
graph LR
    Kafka[Kafka<br/>200 partitions] -->|Consume| Source[Kafka Source<br/>Parallelism: 200]
    
    Source --> Parse[JSON Parser]
    Parse --> FraudFilter[Fraud Detection<br/>Bloom + ML]
    FraudFilter -->|Clean| Window[Windowed Aggregation<br/>1-min tumbling]
    FraudFilter -->|Fraudulent| FraudSink[Fraud Log Sink]
    
    Window --> Aggregate[Aggregate Function<br/>COUNT, SUM, DISTINCT]
    Aggregate --> StateBackend[(RocksDB State<br/>10M counters)]
    
    StateBackend --> InfluxSink[InfluxDB Sink<br/>Batched writes]
    StateBackend --> S3Sink[S3 Sink<br/>Parquet format]
    
    InfluxSink --> InfluxDB[InfluxDB Cluster]
    S3Sink --> S3[S3 Data Lake]
```

---

## 6. InfluxDB Time-Series Storage

**Flow Explanation:**

InfluxDB cluster architecture with sharding and replication for high write throughput.

**Configuration:**

- 10 data nodes (sharded by video_id)
- Write throughput: 5M points/sec
- Retention: 7 days (speed layer)
- Downsampling: 1min → 5min → 1hour

**Query Patterns:**

- Range queries: "Views in last 1 hour"
- Aggregations: SUM, AVG, COUNT
- Fast: 10ms P99

```mermaid
graph TB
    Flink[Flink Workers] -->|Batch Write| Router[Write Router<br/>Hash by video_id]
    
    subgraph InfluxDB Cluster
        subgraph Shard 1
            Node1[(InfluxDB Node 1<br/>video_id 0-10%)]
        end
        
        subgraph Shard 2
            Node2[(InfluxDB Node 2<br/>video_id 10-20%)]
        end
        
        subgraph Shard 3
            Node3[(InfluxDB Node 3<br/>video_id 20-30%)]
        end
        
        subgraph Shard N
            NodeN[(InfluxDB Node 10<br/>video_id 90-100%)]
        end
    end
    
    Router --> Node1
    Router --> Node2
    Router --> Node3
    Router --> NodeN
    
    Node1 --> Ranking[Ranking Service]
    Node2 --> Ranking
    Node3 --> Ranking
    NodeN --> Ranking
```

---

## 7. Ranking Service Architecture

**Flow Explanation:**

Ranking Service queries InfluxDB, calculates trending scores, and updates Redis every 60 seconds.

**Algorithm:**

- Decay function: Score = engagement^0.8 / (age + 2)^1.5
- Processes 10M active videos
- Parallelized across 100 workers

**Performance:**

- Computation time: < 10 seconds
- Update frequency: Every 60 seconds
- Score precision: float64

```mermaid
graph TB
    Scheduler[Scheduler<br/>Cron: Every 60s] --> Coordinator[Ranking Coordinator]
    
    Coordinator --> Query[Query InfluxDB<br/>Last 6 hours metrics]
    Query --> InfluxDB[(InfluxDB<br/>Time-series data)]
    
    InfluxDB --> Coordinator
    
    Coordinator --> Shard1[Worker Shard 1<br/>Gaming videos]
    Coordinator --> Shard2[Worker Shard 2<br/>News videos]
    Coordinator --> Shard3[Worker Shard 3<br/>Music videos]
    Coordinator --> ShardN[Worker Shard N<br/>Other categories]
    
    Shard1 --> Calc1[Score Calculator<br/>Hyperbolic decay]
    Shard2 --> Calc2[Score Calculator<br/>Hyperbolic decay]
    Shard3 --> Calc3[Score Calculator<br/>Hyperbolic decay]
    ShardN --> CalcN[Score Calculator<br/>Hyperbolic decay]
    
    Calc1 --> Redis[Redis Cluster<br/>Update Top K]
    Calc2 --> Redis
    Calc3 --> Redis
    CalcN --> Redis
```

---

## 8. Redis Top K Storage

**Flow Explanation:**

Redis Cluster stores Top K lists using Sorted Sets (ZSET) sharded by dimension.

**Data Structure:**

- Key: trending:global:top100
- Members: video_id
- Scores: trending_score

**Sharding:**

- 6 master nodes + 6 replicas
- Sharded by dimension (global, regional, category)
- Read throughput: 100k QPS per node

```mermaid
graph TB
    Ranking[Ranking Service] -->|ZADD| Proxy[Redis Cluster Proxy]
    
    subgraph Redis Cluster
        subgraph Master Nodes
            M1[(Master 1<br/>Global trending)]
            M2[(Master 2<br/>US regional)]
            M3[(Master 3<br/>EU regional)]
            M4[(Master 4<br/>Gaming category)]
            M5[(Master 5<br/>News category)]
            M6[(Master 6<br/>Music category)]
        end
        
        subgraph Replica Nodes
            R1[(Replica 1)]
            R2[(Replica 2)]
            R3[(Replica 3)]
            R4[(Replica 4)]
            R5[(Replica 5)]
            R6[(Replica 6)]
        end
        
        M1 -.->|Replicate| R1
        M2 -.->|Replicate| R2
        M3 -.->|Replicate| R3
        M4 -.->|Replicate| R4
        M5 -.->|Replicate| R5
        M6 -.->|Replicate| R6
    end
    
    Proxy --> M1
    Proxy --> M2
    Proxy --> M3
    Proxy --> M4
    Proxy --> M5
    Proxy --> M6
    
    M1 --> API[API Gateway<br/>Read requests]
    M2 --> API
    M3 --> API
```

---

## 9. Batch Processing Pipeline

**Flow Explanation:**

Nightly Spark batch job reads S3, performs advanced fraud detection, and updates Redis with corrected scores.

**Steps:**

1. Read S3 Parquet files (last 24 hours)
2. Deduplication by event_id
3. Advanced fraud detection (ML model)
4. Recalculate scores with clean data
5. Write to Cassandra (historical)
6. Refresh Redis (corrected Top K)

**Performance:**

- Data processed: 20 TB/day
- Processing time: 2 hours
- Executors: 100 (Spark)

```mermaid
graph LR
    S3[(S3 Data Lake<br/>Parquet files<br/>20 TB/day)] -->|Distributed Read| Spark[Spark Cluster<br/>100 executors]
    
    Spark --> Dedupe[Deduplication<br/>GROUP BY event_id]
    Dedupe --> Fraud[Fraud Detection<br/>XGBoost ML model]
    Fraud --> Join[Join Metadata<br/>Video info]
    Join --> Aggregate[Aggregate<br/>By video, date, country]
    Aggregate --> Score[Score Calculation<br/>Hyperbolic decay]
    
    Score --> Cassandra[(Cassandra<br/>Historical trends<br/>RF=3)]
    Score --> RedisRefresh[Redis Refresh<br/>Update Top K]
    
    RedisRefresh --> Redis[Redis Cluster]
```

---

## 10. Fraud Detection Flow

**Flow Explanation:**

Multi-stage fraud detection pipeline with Bloom filter, rate limiting, and ML scoring.

**Stages:**

1. Bloom filter: Check known bots (O(k) lookup)
2. Rate limiting: Check per-user/IP rates (Redis)
3. ML model: Random Forest scoring (< 10ms)
4. Decision: Accept, flag, or reject event

**Thresholds:**

- Clean (0.0-0.3): Count normally
- Suspicious (0.3-0.7): Count + flag
- Fraud (0.7-1.0): Reject

```mermaid
graph TD
    Event[View Event] --> BloomCheck{Bloom Filter<br/>Known bot?}
    
    BloomCheck -->|Yes| Reject[Reject Event<br/>Log to fraud DB]
    BloomCheck -->|No| RateCheck[Rate Limiter<br/>Redis INCR]
    
    RateCheck --> CheckRate{Rate > 100/min?}
    CheckRate -->|Yes| Flag[Flag as Suspicious]
    CheckRate -->|No| MLModel[ML Model Scoring<br/>Random Forest]
    
    MLModel --> RiskScore{Risk Score}
    
    RiskScore -->|0.0-0.3<br/>Clean| Accept[Accept Event<br/>Count normally]
    RiskScore -->|0.3-0.7<br/>Suspicious| FlagReview[Accept + Flag<br/>For review]
    RiskScore -->|0.7-1.0<br/>Fraud| Reject
    
    Accept --> Kafka[Kafka Stream<br/>Continue processing]
    FlagReview --> Kafka
    Flag --> Kafka
```

---

## 11. Multi-Dimensional Trending

**Flow Explanation:**

Shows how trending is calculated across multiple dimensions simultaneously.

**Dimensions:**

- Global: Top 100 worldwide
- Regional: Top 100 per country (200 countries)
- Category: Top 100 per category (20 categories)
- Language: Top 100 per language (50 languages)

**Total Lists:** 1 + 200 + 20 + 50 = 271 Top K lists

```mermaid
graph TB
    Ranking[Ranking Service] --> Global[Global Ranking<br/>Top 100 worldwide]
    Ranking --> Regional[Regional Ranking<br/>200 countries]
    Ranking --> Category[Category Ranking<br/>20 categories]
    Ranking --> Language[Language Ranking<br/>50 languages]
    
    Global --> Redis1[(Redis ZSET<br/>trending:global:top100)]
    
    Regional --> Redis2[(Redis ZSET<br/>trending:US:top100)]
    Regional --> Redis3[(Redis ZSET<br/>trending:UK:top100)]
    Regional --> Redis4[(Redis ZSET<br/>trending:JP:top100)]
    
    Category --> Redis5[(Redis ZSET<br/>trending:Gaming:top100)]
    Category --> Redis6[(Redis ZSET<br/>trending:News:top100)]
    Category --> Redis7[(Redis ZSET<br/>trending:Music:top100)]
    
    Language --> Redis8[(Redis ZSET<br/>trending:en:top100)]
    Language --> Redis9[(Redis ZSET<br/>trending:es:top100)]
    Language --> Redis10[(Redis ZSET<br/>trending:fr:top100)]
    
    Redis1 --> API[API Gateway]
    Redis2 --> API
    Redis5 --> API
    Redis8 --> API
```

---

## 12. Multi-Region Deployment

**Flow Explanation:**

Global deployment with regional ingestion and centralized batch processing.

**Strategy:**

- Regional ingestion (US, EU, AP) for low latency
- Kafka MirrorMaker for cross-region replication
- Centralized batch in primary region only

**Latency:**

- Regional: < 30ms (ingestion)
- Cross-region: < 5 seconds (replication)

```mermaid
graph TB
    subgraph US-East-1 Primary
        US_Client[US Clients] --> US_API[API Gateway US]
        US_API --> US_Kafka[Kafka US<br/>Primary]
        US_Kafka --> US_Flink[Flink US]
        US_Flink --> US_InfluxDB[InfluxDB US]
        US_InfluxDB --> US_Ranking[Ranking US]
        US_Ranking --> US_Redis[Redis US]
        
        US_Kafka --> US_S3[S3 US<br/>Data Lake]
        US_S3 --> Spark[Spark Batch<br/>PRIMARY ONLY]
        Spark --> Cassandra[Cassandra US]
        Spark --> US_Redis
    end
    
    subgraph EU-West-1 Secondary
        EU_Client[EU Clients] --> EU_API[API Gateway EU]
        EU_API --> EU_Kafka[Kafka EU<br/>Secondary]
        EU_Kafka --> EU_Flink[Flink EU]
        EU_Flink --> EU_InfluxDB[InfluxDB EU]
        EU_InfluxDB --> EU_Ranking[Ranking EU]
        EU_Ranking --> EU_Redis[Redis EU]
    end
    
    subgraph AP-Southeast Secondary
        AP_Client[AP Clients] --> AP_API[API Gateway AP]
        AP_API --> AP_Kafka[Kafka AP<br/>Secondary]
        AP_Kafka --> AP_Flink[Flink AP]
        AP_Flink --> AP_InfluxDB[InfluxDB AP]
        AP_InfluxDB --> AP_Ranking[Ranking AP]
        AP_Ranking --> AP_Redis[Redis AP]
    end
    
    EU_Kafka -.->|MirrorMaker| US_Kafka
    AP_Kafka -.->|MirrorMaker| US_Kafka
    
    Cassandra -.->|Multi-DC Replication| EU_Cassandra[Cassandra EU]
    Cassandra -.->|Multi-DC Replication| AP_Cassandra[Cassandra AP]
```