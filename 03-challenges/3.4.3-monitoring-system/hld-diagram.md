# Distributed Monitoring System - High-Level Design

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Metrics Collection Flow (Pull-Based)](#2-metrics-collection-flow-pull-based)
3. [Metrics Collection Flow (Push-Based)](#3-metrics-collection-flow-push-based)
4. [Kafka Buffer Architecture](#4-kafka-buffer-architecture)
5. [TSDB Storage Architecture (M3DB)](#5-tsdb-storage-architecture-m3db)
6. [Data Compression Pipeline](#6-data-compression-pipeline)
7. [Rollup and Aggregation Pipeline](#7-rollup-and-aggregation-pipeline)
8. [Alerting Engine Architecture](#8-alerting-engine-architecture)
9. [Query Service Architecture](#9-query-service-architecture)
10. [Multi-Region Deployment](#10-multi-region-deployment)
11. [Tiered Storage Strategy](#11-tiered-storage-strategy)
12. [Cardinality Control System](#12-cardinality-control-system)
13. [Meta-Monitoring Architecture](#13-meta-monitoring-architecture)
14. [Scaling Strategy](#14-scaling-strategy)

---

## 1. System Architecture Overview

**Flow Explanation:**

This diagram shows the complete end-to-end architecture of the distributed monitoring system.

**Key Components:**
1. **Monitored Infrastructure** - 1M endpoints (servers, containers, applications)
2. **Collection Layer** - 100 Prometheus collectors (pull) + agents (push)
3. **Kafka Buffer** - 24 partitions, 100M msgs/sec throughput
4. **TSDB (M3DB)** - 10 PB compressed storage, 100 M3DB nodes
5. **Alerting Engine** - Apache Flink with stateful processing
6. **Real-Time Aggregation** - Stream processor for rollups
7. **Query Service** - API layer with Redis caching
8. **Grafana** - Visualization dashboards

**Performance:**
- Write throughput: 100M data points/sec
- Write latency: <100ms p99
- Query latency: <1 second (dashboard queries)
- Storage: 10 PB (after 10:1 compression)

**Benefits:**
- Decoupled architecture (Kafka buffer prevents TSDB overload)
- Real-time alerting (< 5 seconds)
- Horizontal scalability (add nodes to scale writes/reads)
- Cost-effective storage (tiered storage + compression)

```mermaid
graph TB
    subgraph Monitored Infrastructure
        E1[Server 1<br/>100 metrics/sec]
        E2[Server 2<br/>100 metrics/sec]
        E3[Container<br/>100 metrics/sec]
        E_N[Server 1M<br/>100 metrics/sec]
    end
    
    subgraph Collection Layer
        C1[Collector 1<br/>Prometheus<br/>1M metrics/sec]
        C2[Collector 2<br/>Prometheus<br/>1M metrics/sec]
        C_N[Collector 100<br/>1M metrics/sec]
    end
    
    subgraph Buffer Layer
        Kafka[Kafka Cluster<br/>24 partitions<br/>100M msgs/sec<br/>Retention: 24h]
    end
    
    subgraph Storage Layer
        TSDB[M3DB Cluster<br/>100 nodes<br/>10 PB compressed<br/>Gorilla compression]
    end
    
    subgraph Processing Layer
        Alert[Alerting Engine<br/>Apache Flink<br/>Stateful processing<br/>5-second latency]
        Rollup[Real-Time Aggregation<br/>Kafka Streams<br/>1min, 1hr, 1day rollups]
    end
    
    subgraph Query Layer
        QS[Query Service<br/>API + Cache<br/>Redis 80% hit rate]
    end
    
    subgraph Notification
        Notify[Notification Service<br/>Email, Slack, PagerDuty]
    end
    
    subgraph Visualization
        Grafana[Grafana Dashboards<br/>16k QPS read queries]
    end
    
    E1 --> C1
    E2 --> C1
    E3 --> C2
    E_N --> C_N
    
    C1 --> Kafka
    C2 --> Kafka
    C_N --> Kafka
    
    Kafka --> TSDB
    Kafka --> Alert
    Kafka --> Rollup
    
    Rollup --> TSDB
    
    Alert --> Notify
    
    TSDB --> QS
    QS --> Grafana
    
    style Kafka fill:#ff9
    style TSDB fill:#9f9
    style Alert fill:#f99
    style QS fill:#99f
```

---

## 2. Metrics Collection Flow (Pull-Based)

**Flow Explanation:**

This diagram shows the pull-based collection model (Prometheus style) where collectors scrape metrics from endpoints.

**Steps:**
1. **Service Discovery (0s)** - Consul/Kubernetes API returns list of endpoints
2. **Scrape Scheduling (1s)** - Collector schedules scrape every 1 second per endpoint
3. **HTTP GET (10ms)** - Collector sends GET request to `/metrics` endpoint
4. **Parse Response (5ms)** - Parse Prometheus text format
5. **Local Buffer (1ms)** - Write to local buffer (100 metrics batched)
6. **Kafka Write (10ms)** - Batch write to Kafka (async)
7. **Total latency:** ~26ms from scrape to Kafka

**Benefits:**
- Centralized control (collector knows all endpoints)
- Endpoint discovery automatic (Kubernetes integration)
- Dead endpoint detection (failed scrapes)

**Trade-offs:**
- Requires inbound network access to endpoints
- Collector becomes bottleneck (horizontal scaling needed)
- Pull interval fixed (1 second minimum)

```mermaid
graph TB
    subgraph Service Discovery
        SD[Service Discovery<br/>Consul/Kubernetes<br/>Returns endpoint list]
    end
    
    subgraph Endpoints
        E1[Endpoint 1<br/>:9090/metrics<br/>CPU, memory, disk]
        E2[Endpoint 2<br/>:9090/metrics]
        E3[Endpoint N<br/>:9090/metrics]
    end
    
    subgraph Collector
        Sched[Scrape Scheduler<br/>Every 1 second<br/>Per endpoint]
        Scraper[Scraper Thread Pool<br/>1000 threads]
        Parser[Metrics Parser<br/>Prometheus format]
        Buffer[Local Buffer<br/>Batch 100 metrics]
    end
    
    subgraph Kafka
        Topic[Kafka Topic<br/>metrics.raw<br/>Partition by hash]
    end
    
    SD -->|List endpoints| Sched
    Sched -->|Schedule scrape| Scraper
    
    Scraper -->|HTTP GET /metrics| E1
    Scraper -->|HTTP GET /metrics| E2
    Scraper -->|HTTP GET /metrics| E3
    
    E1 -->|Response 200 OK<br/>metric_name value timestamp| Scraper
    E2 -->|Response| Scraper
    E3 -->|Response| Scraper
    
    Scraper --> Parser
    Parser --> Buffer
    Buffer -->|Batch write<br/>100 metrics| Topic
    
    style Sched fill:#ff9
    style Parser fill:#9f9
    style Buffer fill:#99f
```

---

## 3. Metrics Collection Flow (Push-Based)

**Flow Explanation:**

This diagram shows the push-based collection model where agents installed on endpoints push metrics to collectors.

**Steps:**
1. **Agent Collection (0s)** - Agent collects system metrics (CPU, memory, disk, network)
2. **Local Aggregation (1s)** - Agent aggregates metrics over 1-second window
3. **Batching (1s)** - Agent batches 100 metrics before sending
4. **HTTP POST (10ms)** - Agent sends POST request to collector
5. **Collector Processing (5ms)** - Collector validates and formats metrics
6. **Kafka Write (10ms)** - Write to Kafka (async)
7. **Total latency:** ~26ms from collection to Kafka

**Benefits:**
- No inbound network access required (agent initiates connection)
- Edge devices supported (firewalls, NAT)
- Agent retries on failure (reliability)

**Trade-offs:**
- Agent installation overhead (deploy on every endpoint)
- Agent resource usage (CPU, memory)
- Endpoint-initiated (more network connections)

```mermaid
graph TB
    subgraph Endpoint with Agent
        App[Application<br/>Custom metrics]
        System[System Metrics<br/>CPU, memory, disk]
        Agent[Agent<br/>Telegraf/Datadog<br/>Collects metrics]
        AgentBuffer[Agent Buffer<br/>Batch 100 metrics<br/>1-second window]
    end
    
    subgraph Collector
        Receiver[HTTP Receiver<br/>POST /metrics]
        Validator[Validator<br/>Check format<br/>Rate limiting]
        Formatter[Formatter<br/>Convert to standard format]
    end
    
    subgraph Kafka
        Topic[Kafka Topic<br/>metrics.raw]
    end
    
    App -->|Emit metrics| Agent
    System -->|Collect metrics| Agent
    
    Agent --> AgentBuffer
    AgentBuffer -->|HTTP POST<br/>JSON payload<br/>100 metrics| Receiver
    
    Receiver --> Validator
    Validator --> Formatter
    Formatter -->|Write| Topic
    
    style Agent fill:#ff9
    style AgentBuffer fill:#9f9
    style Validator fill:#99f
```

---

## 4. Kafka Buffer Architecture

**Flow Explanation:**

This diagram shows the Kafka cluster architecture for buffering 100M metrics/sec with partitioning and replication strategy.

**Kafka Configuration:**
- **Partitions:** 24 (distributes load across TSDB writers)
- **Replication:** 3x (durability, no data loss)
- **Retention:** 24 hours (replay window for backfill)
- **Compression:** snappy (5x reduction in storage)

**Partitioning Strategy:**
- Partition by `hash(metric_name + labels)` to ensure all data points for a specific time series go to the same partition (maintains ordering)

**Consumer Groups:**
1. **tsdb-writers** (24 consumers) - Write to M3DB
2. **alerting-engine** (12 consumers) - Real-time alert evaluation
3. **rollup-aggregators** (12 consumers) - Downsampling and rollups

**Performance:**
- Write throughput: 100M msgs/sec sustained
- Per-partition throughput: 4.2M msgs/sec (comfortable for Kafka)
- Latency: p99 <10ms (producer to consumer)
- Durability: 3x replication (no data loss on broker failure)

**Benefits:**
- Decouples collectors from TSDB (if TSDB slow, Kafka absorbs)
- Replay capability (reprocess metrics for backfill)
- Multiple consumers (TSDB, alerting, rollups) read same stream

```mermaid
graph TB
    subgraph Collectors
        C1[Collector 1]
        C2[Collector 2]
        C3[Collector 100]
    end
    
    subgraph Kafka Cluster
        subgraph Broker 1
            P1[Partition 0-7<br/>Replication: 3x]
        end
        subgraph Broker 2
            P2[Partition 8-15<br/>Replication: 3x]
        end
        subgraph Broker 3
            P3[Partition 16-23<br/>Replication: 3x]
        end
    end
    
    subgraph Consumer Groups
        subgraph tsdb-writers 24 consumers
            TW1[Writer 1<br/>Partitions: 0]
            TW2[Writer 2<br/>Partitions: 1]
            TW24[Writer 24<br/>Partitions: 23]
        end
        subgraph alerting-engine 12 consumers
            AE1[Alert 1<br/>Partitions: 0-1]
            AE12[Alert 12<br/>Partitions: 22-23]
        end
        subgraph rollup-aggregators 12 consumers
            RA1[Rollup 1<br/>Partitions: 0-1]
            RA12[Rollup 12<br/>Partitions: 22-23]
        end
    end
    
    C1 -->|Write<br/>Async| P1
    C2 -->|Write<br/>Async| P2
    C3 -->|Write<br/>Async| P3
    
    P1 --> TW1
    P1 --> AE1
    P1 --> RA1
    
    P2 --> TW2
    P2 --> AE1
    P2 --> RA1
    
    P3 --> TW24
    P3 --> AE12
    P3 --> RA12
    
    style P1 fill:#ff9
    style TW1 fill:#9f9
    style AE1 fill:#f99
    style RA1 fill:#99f
```

---

## 5. TSDB Storage Architecture (M3DB)

**Flow Explanation:**

This diagram shows the M3DB cluster architecture with sharding, replication, and query coordination.

**M3DB Components:**
1. **M3DB Nodes (100 nodes)** - Distributed storage, 100 GB memory + 1 TB SSD each
2. **M3Coordinator (3 nodes)** - Query aggregator and router
3. **M3Aggregator (10 nodes)** - Real-time aggregation for rollups
4. **M3Query (5 nodes)** - PromQL-compatible query engine

**Sharding Strategy:**
- **Shard count:** 4096 virtual shards
- **Shard distribution:** Consistent hashing by `hash(metric_name + labels)`
- **Replication factor:** 3x (each shard stored on 3 nodes)
- **Shard placement:** RF=3 means each time series stored on 3 different nodes

**Data Flow:**
1. **Write:** M3Coordinator → Shard placement → 3 M3DB nodes (parallel writes)
2. **Read:** M3Query → M3Coordinator → Query all relevant shards → Merge results

**Performance:**
- Write throughput: 1M writes/sec per node × 100 nodes = 100M writes/sec
- Storage: 100 TB raw per node × 100 nodes = 10 PB (after compression)
- Query latency: p50=50ms, p99=500ms (depends on time range)

**Benefits:**
- Horizontal scalability (add nodes linearly)
- High availability (3x replication, survives 2 node failures)
- Consistent hashing (no resharding on node add/remove)

```mermaid
graph TB
    subgraph M3 Cluster
        subgraph M3Coordinator
            Coord[M3Coordinator<br/>Query router<br/>Shard placement]
        end
        
        subgraph M3DB Nodes 100 nodes
            Node1[M3DB Node 1<br/>Shards: 0-40<br/>100 GB mem<br/>1 TB SSD]
            Node2[M3DB Node 2<br/>Shards: 41-81]
            Node3[M3DB Node 100<br/>Shards: 4056-4095]
        end
        
        subgraph M3Aggregator
            Agg[M3Aggregator<br/>10 nodes<br/>Real-time rollups]
        end
        
        subgraph M3Query
            Query[M3Query<br/>5 nodes<br/>PromQL engine]
        end
    end
    
    subgraph Kafka
        K[Kafka metrics.raw]
    end
    
    subgraph Clients
        Grafana[Grafana Dashboard]
    end
    
    K -->|Consume| Coord
    Coord -->|Shard placement<br/>RF=3| Node1
    Coord -->|Shard placement<br/>RF=3| Node2
    Coord -->|Shard placement<br/>RF=3| Node3
    
    K -->|Consume| Agg
    Agg -->|Write rollups| Coord
    
    Grafana -->|PromQL query| Query
    Query --> Coord
    Coord -->|Query shards| Node1
    Coord -->|Query shards| Node2
    Coord -->|Query shards| Node3
    Node1 -->|Results| Coord
    Node2 -->|Results| Coord
    Node3 -->|Results| Coord
    Coord -->|Merged results| Query
    Query -->|Response| Grafana
    
    style Coord fill:#ff9
    style Node1 fill:#9f9
    style Agg fill:#f99
    style Query fill:#99f
```

---

## 6. Data Compression Pipeline

**Flow Explanation:**

This diagram shows the Gorilla compression algorithm used by M3DB to achieve 10:1 compression ratio.

**Gorilla Compression (Facebook's Algorithm):**

**1. Timestamp Compression (Delta-of-Delta Encoding)**
- Store first timestamp as-is (64 bits)
- Store delta from previous timestamp (e.g., 1 second = 1000ms)
- Store delta-of-delta (if interval constant, delta-of-delta = 0)
- Result: 1-2 bits per timestamp (vs 64 bits raw)

**2. Value Compression (XOR Encoding)**
- XOR current value with previous value
- If XOR result has leading/trailing zeros, store only significant bits
- Result: Average 1.37 bits per value (vs 64 bits for double)

**Example:**
```
Raw timestamps: [1000, 2000, 3000, 4000] (256 bits total)
Deltas: [-, 1000, 1000, 1000]
Delta-of-deltas: [-, -, 0, 0]
Compressed: 64 + 12 + 1 + 1 = 78 bits (3.3x compression)

Raw values: [65.2, 67.5, 70.1, 72.3] (256 bits total)
XOR compression: 64 + 8 + 12 + 10 = 94 bits (2.7x compression)

Total: 78 + 94 = 172 bits (vs 512 bits raw = 3x compression for 4 points)
```

**Overall Compression:**
- 100 PB raw data → 10 PB compressed (10:1 ratio)
- Compression CPU cost: <5% (acceptable overhead)

**Benefits:**
- Massive storage savings (10x reduction)
- Faster queries (less data to scan)
- Lower network transfer (less data to send)

```mermaid
graph TB
    subgraph Raw Data
        Raw[Raw Time-Series<br/>1000: 65.2<br/>2000: 67.5<br/>3000: 70.1<br/>4000: 72.3<br/>256 bits per metric]
    end
    
    subgraph Timestamp Compression
        TS1[First Timestamp<br/>1000<br/>64 bits]
        TS2[Delta<br/>2000-1000=1000<br/>12 bits]
        TS3[Delta-of-Delta<br/>1000-1000=0<br/>1 bit]
        TS4[Delta-of-Delta<br/>1000-1000=0<br/>1 bit]
        TSResult[Total: 78 bits<br/>vs 256 bits<br/>3.3x compression]
    end
    
    subgraph Value Compression XOR
        V1[First Value<br/>65.2<br/>64 bits]
        V2[XOR with prev<br/>67.5 XOR 65.2<br/>8 bits significant]
        V3[XOR with prev<br/>70.1 XOR 67.5<br/>12 bits significant]
        V4[XOR with prev<br/>72.3 XOR 70.1<br/>10 bits significant]
        VResult[Total: 94 bits<br/>vs 256 bits<br/>2.7x compression]
    end
    
    subgraph Compressed Data
        Compressed[Compressed Block<br/>172 bits<br/>vs 512 bits raw<br/>3x compression]
    end
    
    subgraph Storage
        Disk[M3DB Storage<br/>10 PB<br/>vs 100 PB raw<br/>10:1 overall]
    end
    
    Raw --> TS1
    Raw --> V1
    
    TS1 --> TS2 --> TS3 --> TS4 --> TSResult
    V1 --> V2 --> V3 --> V4 --> VResult
    
    TSResult --> Compressed
    VResult --> Compressed
    
    Compressed --> Disk
    
    style TSResult fill:#ff9
    style VResult fill:#9f9
    style Compressed fill:#99f
```

---

## 7. Rollup and Aggregation Pipeline

**Flow Explanation:**

This diagram shows the real-time aggregation pipeline for downsampling metrics from 1-second resolution to 1-minute, 5-minute, 1-hour, and 1-day rollups.

**Rollup Strategy:**
- **Raw (1-second):** Retained for 1 day (8.64 trillion points)
- **1-minute rollup:** Retained for 7 days (1.4 trillion points)
- **5-minute rollup:** Retained for 30 days (8.6 billion points)
- **1-hour rollup:** Retained for 1 year (876 million points)
- **1-day rollup:** Retained for 2+ years (730,000 points per metric)

**Aggregation Functions:**
- `count` - Number of samples in window
- `sum` - Sum of values
- `min`, `max` - Min/max values
- `avg` - Average value
- `p50`, `p95`, `p99` - Percentiles

**Processing:**
1. **Kafka Streams** consumes `metrics.raw` topic
2. **Windowed aggregation** (tumbling windows: 1min, 5min, 1hr, 1day)
3. **Compute aggregates** (count, sum, min, max, avg, percentiles)
4. **Write to TSDB** rollup tables
5. **Auto-expire** old raw data (1 day retention)

**Benefits:**
- Storage reduction: 100 PB raw → 10 PB (with rollups)
- Query performance: 3600x faster (1-hour rollup vs raw for 30-day query)
- Long-term retention: Store 2+ years of data cost-effectively

```mermaid
graph TB
    subgraph Input Stream
        Kafka[Kafka metrics.raw<br/>100M msgs/sec<br/>1-second resolution]
    end
    
    subgraph Kafka Streams Aggregators
        Agg1Min[1-Minute Aggregator<br/>Tumbling window: 60s<br/>Compute: count, sum, min, max, avg, p50, p95, p99]
        Agg5Min[5-Minute Aggregator<br/>Tumbling window: 300s]
        Agg1Hr[1-Hour Aggregator<br/>Tumbling window: 3600s]
        Agg1Day[1-Day Aggregator<br/>Tumbling window: 86400s]
    end
    
    subgraph TSDB Rollup Tables
        T1[metrics_raw<br/>Retention: 1 day<br/>8.64T points]
        T2[metrics_rollup_1min<br/>Retention: 7 days<br/>1.4T points]
        T3[metrics_rollup_5min<br/>Retention: 30 days<br/>8.6B points]
        T4[metrics_rollup_1hr<br/>Retention: 1 year<br/>876M points]
        T5[metrics_rollup_1day<br/>Retention: 2+ years<br/>730K points]
    end
    
    subgraph Auto-Expiration
        Expire[TTL Cleanup<br/>Delete data older than retention<br/>Runs daily]
    end
    
    Kafka --> T1
    Kafka --> Agg1Min
    Kafka --> Agg5Min
    Kafka --> Agg1Hr
    Kafka --> Agg1Day
    
    Agg1Min -->|Write aggregates| T2
    Agg5Min -->|Write aggregates| T3
    Agg1Hr -->|Write aggregates| T4
    Agg1Day -->|Write aggregates| T5
    
    Expire -->|Delete old data| T1
    Expire -->|Delete old data| T2
    Expire -->|Delete old data| T3
    Expire -->|Delete old data| T4
    
    style Agg1Min fill:#ff9
    style T2 fill:#9f9
    style Expire fill:#f99
```

---

## 8. Alerting Engine Architecture

**Flow Explanation:**

This diagram shows the stateful stream processing architecture for real-time alert evaluation using Apache Flink.

**Alert Processing Flow:**
1. **Consume metrics** from Kafka `metrics.raw`
2. **Group by alert rule** (e.g., all `cpu_usage` for `service=api-gateway`)
3. **Windowed aggregation** (e.g., 5-minute tumbling window)
4. **Evaluate condition** (e.g., `avg > 80%`)
5. **State management** (track if alert already firing)
6. **Deduplicate** (don't send duplicate alerts)
7. **Trigger notification** (email, Slack, PagerDuty)

**Stateful Processing:**
- **State store:** RocksDB (embedded key-value store)
- **State:** Current alert status (firing, resolved), window aggregations
- **Checkpointing:** Periodic snapshots every 60 seconds for fault tolerance
- **Exactly-once semantics:** No duplicate alert notifications

**Alert Rule Example:**
```
name: HighCPUUsage
condition: avg(cpu_usage{service="api-gateway"}) > 80 for 5 minutes
severity: warning
notification: slack-channel=#alerts
```

**Performance:**
- Alert evaluation latency: <5 seconds (from metric arrival to notification)
- Throughput: 10k alert rules evaluated concurrently
- State size: <10 GB per Flink task (for 10k rules)

**Benefits:**
- Real-time (<5 seconds latency)
- Stateful (remembers alert state)
- Scalable (horizontal scaling with Flink parallelism)
- Fault-tolerant (checkpointing and recovery)

```mermaid
graph TB
    subgraph Input
        Kafka[Kafka metrics.raw<br/>100M msgs/sec]
    end
    
    subgraph Flink Alerting Engine
        subgraph Task 1
            Filter1[Filter by Rule 1<br/>cpu_usage service=api-gateway]
            Window1[5-Minute Window<br/>Tumbling]
            Agg1[Aggregate<br/>avg, min, max]
            Eval1[Evaluate Condition<br/>avg > 80%]
            State1[State Store<br/>RocksDB<br/>Alert status: firing/resolved]
        end
        
        subgraph Task 2
            Filter2[Filter by Rule 2<br/>memory_usage]
            Window2[5-Minute Window]
            Agg2[Aggregate]
            Eval2[Evaluate Condition]
            State2[State Store]
        end
        
        Dedup[Deduplicator<br/>Don't resend if already firing]
        Checkpoint[Checkpointing<br/>Every 60 seconds<br/>Fault tolerance]
    end
    
    subgraph Output
        AlertsTopic[Kafka alerts.triggered]
        NotifySvc[Notification Service<br/>Email, Slack, PagerDuty]
    end
    
    Kafka --> Filter1
    Kafka --> Filter2
    
    Filter1 --> Window1 --> Agg1 --> Eval1
    Eval1 <-->|Read/Write state| State1
    
    Filter2 --> Window2 --> Agg2 --> Eval2
    Eval2 <-->|Read/Write state| State2
    
    Eval1 --> Dedup
    Eval2 --> Dedup
    
    Dedup --> AlertsTopic
    AlertsTopic --> NotifySvc
    
    Checkpoint -.Snapshot.-> State1
    Checkpoint -.Snapshot.-> State2
    
    style Eval1 fill:#ff9
    style State1 fill:#9f9
    style Dedup fill:#f99
    style NotifySvc fill:#99f
```

---

## 9. Query Service Architecture

**Flow Explanation:**

This diagram shows the query service architecture with caching, query optimization, and parallel execution.

**Query Processing Flow:**
1. **Receive query** from Grafana (PromQL or HTTP API)
2. **Query optimization** (automatic rollup selection, predicate pushdown)
3. **Check cache** (Redis query result cache, 80% hit rate)
4. **If cache miss:** Execute query on TSDB
5. **Parallel execution** (query multiple shards in parallel)
6. **Merge results** from all shards
7. **Store in cache** (TTL: 1 minute for recent data, 1 hour for old data)
8. **Return to client**

**Query Optimization:**

**1. Automatic Rollup Selection**
- Query: "Show CPU for last 30 days"
- Optimizer: Use 1-hour rollup (not raw)
- Benefit: 3600x fewer data points (720 vs 2.6M)

**2. Predicate Pushdown**
- Push filters to TSDB storage layer
- Example: `region=us-east-1` filtered at shard level

**3. Query Parallelization**
- Split query across shards
- Execute in parallel
- Merge results

**Caching:**
- **L1 Cache (Redis):** Query result cache, 80% hit rate
- **L2 Cache (M3DB):** Block cache (1-hour chunks), 60% hit rate

**Performance:**
- Cache hit: <10ms (Redis)
- Cache miss: <1 second (for last 1 hour query)
- Long-range query (30 days): <2 seconds (with 1-hour rollup)

**Benefits:**
- Fast dashboard queries (<1 second)
- Reduced TSDB load (80% cache hit rate)
- Automatic optimization (rollup selection)

```mermaid
graph TB
    subgraph Clients
        Grafana[Grafana Dashboard<br/>PromQL query<br/>avg cpu_usage 30 days]
    end
    
    subgraph Query Service
        API[Query API<br/>REST + PromQL]
        Optimizer[Query Optimizer<br/>1. Select rollup<br/>2. Predicate pushdown<br/>3. Parallelization]
        Cache[Redis Cache<br/>Query result cache<br/>Hit rate: 80%<br/>TTL: 1 min recent]
        Exec[Query Executor<br/>Parallel execution]
    end
    
    subgraph TSDB M3DB
        Shard1[Shard 1<br/>Query sharded data]
        Shard2[Shard 2]
        ShardN[Shard N<br/>4096 shards total]
    end
    
    subgraph Result Processing
        Merge[Result Merger<br/>Merge from shards]
        Format[Result Formatter<br/>JSON response]
    end
    
    Grafana -->|1. Query| API
    API -->|2. Optimize| Optimizer
    Optimizer -->|3. Check cache| Cache
    
    Cache -->|Cache HIT<br/>80%| API
    Cache -->|Cache MISS<br/>20%| Exec
    
    Exec -->|4. Parallel queries| Shard1
    Exec -->|4. Parallel queries| Shard2
    Exec -->|4. Parallel queries| ShardN
    
    Shard1 -->|Results| Merge
    Shard2 -->|Results| Merge
    ShardN -->|Results| Merge
    
    Merge --> Format
    Format -->|5. Store in cache| Cache
    Format -->|6. Response| API
    API -->|7. JSON| Grafana
    
    style Optimizer fill:#ff9
    style Cache fill:#9f9
    style Exec fill:#f99
    style Merge fill:#99f
```

---

## 10. Multi-Region Deployment

**Flow Explanation:**

This diagram shows the multi-region deployment architecture for low-latency global metric collection.

**Regional Deployment:**
- **3 Regions:** US-East, EU-West, AP-Southeast
- **Per-region stack:** Collectors → Kafka → TSDB → Query Service
- **Local collection:** Metrics collected in the region where endpoints are located
- **Regional dashboards:** Fast queries (local TSDB)

**Global Aggregation:**
- **Cross-region replication:** Kafka MirrorMaker 2 replicates metrics to central region
- **Global TSDB:** Aggregates metrics from all regions
- **Global dashboards:** View metrics across all regions

**Replication Lag:**
- Regional to global: 5-30 seconds (acceptable for most use cases)
- Eventually consistent (global metrics lag behind regional)

**Benefits:**
- Low-latency collection (<50ms in-region)
- Regional dashboards for local troubleshooting
- Global view for cross-region analysis
- Region failover (if one region fails, others continue)

**Trade-offs:**
- Higher cost (3x infrastructure)
- Eventual consistency (global lag)
- Cross-region networking complexity

```mermaid
graph TB
    subgraph US-East Primary
        US_Endpoints[US Endpoints<br/>500k servers]
        US_Collectors[Collectors<br/>50 instances]
        US_Kafka[Kafka Cluster<br/>12 partitions]
        US_TSDB[M3DB Cluster<br/>40 nodes]
        US_Query[Query Service]
        US_Dash[Regional Dashboard<br/>US metrics only]
    end
    
    subgraph EU-West
        EU_Endpoints[EU Endpoints<br/>300k servers]
        EU_Collectors[Collectors<br/>30 instances]
        EU_Kafka[Kafka Cluster<br/>8 partitions]
        EU_TSDB[M3DB Cluster<br/>25 nodes]
        EU_Query[Query Service]
        EU_Dash[Regional Dashboard<br/>EU metrics only]
    end
    
    subgraph AP-Southeast
        AP_Endpoints[AP Endpoints<br/>200k servers]
        AP_Collectors[Collectors<br/>20 instances]
        AP_Kafka[Kafka Cluster<br/>4 partitions]
        AP_TSDB[M3DB Cluster<br/>20 nodes]
        AP_Query[Query Service]
        AP_Dash[Regional Dashboard<br/>AP metrics only]
    end
    
    subgraph Global Aggregation US-East
        Global_Kafka[Global Kafka<br/>Aggregated metrics<br/>All regions]
        Global_TSDB[Global M3DB<br/>35 nodes<br/>All regions]
        Global_Query[Global Query Service]
        Global_Dash[Global Dashboard<br/>All regions]
    end
    
    US_Endpoints --> US_Collectors --> US_Kafka --> US_TSDB --> US_Query --> US_Dash
    EU_Endpoints --> EU_Collectors --> EU_Kafka --> EU_TSDB --> EU_Query --> EU_Dash
    AP_Endpoints --> AP_Collectors --> AP_Kafka --> AP_TSDB --> AP_Query --> AP_Dash
    
    US_Kafka -.MirrorMaker 2<br/>5-30s lag.-> Global_Kafka
    EU_Kafka -.MirrorMaker 2<br/>5-30s lag.-> Global_Kafka
    AP_Kafka -.MirrorMaker 2<br/>5-30s lag.-> Global_Kafka
    
    Global_Kafka --> Global_TSDB --> Global_Query --> Global_Dash
    
    style US_TSDB fill:#ff9
    style EU_TSDB fill:#9f9
    style AP_TSDB fill:#f99
    style Global_TSDB fill:#99f
```

---

## 11. Tiered Storage Strategy

**Flow Explanation:**

This diagram shows the tiered storage strategy (hot/warm/cold) for cost-effective long-term retention.

**Storage Tiers:**

**1. Hot Tier (0-7 days)**
- **Storage:** SSD (NVMe)
- **Speed:** Very fast (<10ms read latency)
- **Cost:** $0.10/GB-month
- **Size:** 800 TB
- **Use Case:** Recent metrics, real-time dashboards

**2. Warm Tier (7-90 days)**
- **Storage:** HDD (SATA)
- **Speed:** Fast (<50ms read latency)
- **Cost:** $0.02/GB-month
- **Size:** 8 PB
- **Use Case:** Historical analysis, troubleshooting

**3. Cold Tier (90+ days)**
- **Storage:** S3/Glacier
- **Speed:** Slow (seconds to minutes)
- **Cost:** $0.004/GB-month
- **Size:** 1 PB
- **Use Case:** Compliance, long-term archival

**Data Lifecycle:**
1. **Day 0-7:** Store in hot tier (SSD)
2. **Day 7:** Transition to warm tier (HDD)
3. **Day 90:** Transition to cold tier (S3)
4. **Day 730:** Delete (optional, based on retention policy)

**Cost Analysis:**
```
Hot tier (7 days): 800 TB × $0.10 = $80k/month
Warm tier (83 days): 8 PB × $0.02 = $160k/month
Cold tier (2 years): 1 PB × $0.004 = $4k/month

Total storage cost: $244k/month
vs Single-tier (SSD): 10 PB × $0.10 = $1M/month (4x more expensive)
```

**Benefits:**
- Cost-effective (4x cheaper than SSD-only)
- Fast recent data access (hot tier)
- Long-term retention (cold tier)

**Trade-offs:**
- Complexity (manage 3 storage tiers)
- Query latency varies (hot: 10ms, cold: seconds)

```mermaid
graph TB
    subgraph Metrics Flow
        Kafka[Kafka metrics.raw<br/>100M msgs/sec]
    end
    
    subgraph Hot Tier 0-7 days
        Hot[SSD Storage<br/>NVMe<br/>800 TB<br/>$0.10/GB-month<br/><10ms read latency]
    end
    
    subgraph Warm Tier 7-90 days
        Warm[HDD Storage<br/>SATA<br/>8 PB<br/>$0.02/GB-month<br/><50ms read latency]
    end
    
    subgraph Cold Tier 90+ days
        Cold[S3/Glacier<br/>1 PB<br/>$0.004/GB-month<br/>Seconds read latency]
    end
    
    subgraph Lifecycle Manager
        LM[Data Lifecycle Manager<br/>Automated transitions<br/>Based on age]
    end
    
    subgraph Query Service
        QS[Query Service<br/>Auto-select tier<br/>Based on time range]
    end
    
    Kafka -->|Write| Hot
    
    Hot -.Day 7 transition.-> LM
    LM -->|Move data| Warm
    
    Warm -.Day 90 transition.-> LM
    LM -->|Move data| Cold
    
    Cold -.Day 730 optional.-> LM
    LM -->|Delete| Cold
    
    QS -->|Query last 7 days<br/>Fast| Hot
    QS -->|Query 7-90 days<br/>Medium| Warm
    QS -->|Query 90+ days<br/>Slow| Cold
    
    style Hot fill:#ff9
    style Warm fill:#9f9
    style Cold fill:#99f
    style LM fill:#f99
```

---

## 12. Cardinality Control System

**Flow Explanation:**

This diagram shows the cardinality control system to prevent label explosion (high-cardinality problem).

**Cardinality Problem:**
- Each unique label combination = new time series
- Example: `request_id` label with 1B unique values = 1B time series
- High cardinality → Memory explosion, slow queries, TSDB crashes

**Cardinality Control:**

**1. Label Validation (Ingestion Time)**
- Whitelist approved labels (only allow specific label keys)
- Blacklist high-cardinality labels (`user_id`, `request_id`, `session_id`)
- Reject metrics with invalid labels

**2. Cardinality Monitoring**
- Track unique label combinations per metric
- Alert if cardinality growth >10% per day
- Dashboard shows top high-cardinality metrics

**3. Cardinality Limits**
- Max 20 labels per metric
- Max 10k unique label combinations per metric
- Reject metrics exceeding limits

**Example:**
```
❌ BAD (high cardinality):
http_requests{request_id="abc123", user_id="user456"}
→ 1B unique request_ids × 100M users = 100 quadrillion time series

✅ GOOD (low cardinality):
http_requests{service="api-gateway", endpoint="/users", method="POST", status_code="200"}
→ 10 services × 100 endpoints × 5 methods × 10 status_codes = 50k time series
```

**Benefits:**
- Prevents memory explosion (keeps TSDB stable)
- Faster queries (fewer time series to scan)
- Lower cost (less storage)

**Trade-offs:**
- Less flexibility (can't use any label)
- Requires label design discipline

```mermaid
graph TB
    subgraph Collectors
        C[Collector<br/>Receives metrics<br/>with labels]
    end
    
    subgraph Label Validation
        Whitelist[Label Whitelist<br/>Approved: service, method, endpoint, region]
        Blacklist[Label Blacklist<br/>Rejected: user_id, request_id, session_id]
        Validator[Label Validator<br/>Check against lists]
    end
    
    subgraph Cardinality Monitoring
        CardTracker[Cardinality Tracker<br/>Count unique label combos<br/>Per metric]
        CardDB[(Cardinality Database<br/>PostgreSQL<br/>Track growth)]
        CardAlert[Cardinality Alerts<br/>Growth >10%/day<br/>Total >10k combos]
    end
    
    subgraph Cardinality Limits
        LimitCheck[Limit Checker<br/>Max 20 labels<br/>Max 10k combos]
        Reject[Reject Metric<br/>Return 400 error]
    end
    
    subgraph Output
        Kafka[Kafka metrics.raw<br/>Only valid metrics]
    end
    
    C --> Whitelist
    C --> Blacklist
    Whitelist --> Validator
    Blacklist --> Validator
    
    Validator -->|Valid| CardTracker
    Validator -->|Invalid| Reject
    
    CardTracker --> CardDB
    CardDB --> CardAlert
    
    CardTracker --> LimitCheck
    LimitCheck -->|Within limits| Kafka
    LimitCheck -->|Exceeds limits| Reject
    
    style Validator fill:#ff9
    style CardTracker fill:#9f9
    style LimitCheck fill:#f99
    style Reject fill:#f99
```

---

## 13. Meta-Monitoring Architecture

**Flow Explanation:**

This diagram shows the dual-layered monitoring architecture for monitoring the monitoring system itself.

**Challenge:** How do you monitor the monitoring system? If the monitoring system is down, how do you know?

**Solution: Dual-Layered Monitoring**

**Layer 1: Self-Monitoring**
- Primary monitoring system monitors itself
- Collectors emit metrics about themselves (scrape duration, errors)
- Kafka emits metrics (consumer lag, broker health)
- TSDB emits metrics (write throughput, query latency, disk usage)
- Alerts configured for self-monitoring metrics

**Layer 2: External Monitoring**
- Separate lightweight monitoring system (e.g., standalone Prometheus)
- Deployed on separate infrastructure (different cluster, different region)
- Monitors critical metrics of primary monitoring system
- Alerts if primary system is down or degraded

**Key Metrics Monitored:**

| Component | Metric | Alert Threshold |
|-----------|--------|-----------------|
| Collectors | `scrape_duration_seconds` | >5 seconds |
| Collectors | `scrape_errors_total` | >10 errors/min |
| Kafka | `consumer_lag` | >1M messages |
| TSDB | `write_throughput_qps` | <50M QPS (expected 100M) |
| TSDB | `query_latency_p99_ms` | >5000ms |
| Alerting | `alert_evaluation_duration_ms` | >10 seconds |

**Benefits:**
- Catches primary monitoring system failures
- Independent external monitoring (different failure domain)
- Lightweight external system (simple, reliable)

**Trade-offs:**
- Additional infrastructure cost (external monitoring)
- Two monitoring systems to maintain

```mermaid
graph TB
    subgraph Primary Monitoring System
        subgraph Collectors
            C[Collectors<br/>Self-monitoring<br/>Emit metrics]
        end
        subgraph Kafka
            K[Kafka<br/>Self-monitoring<br/>Emit metrics]
        end
        subgraph TSDB
            T[M3DB<br/>Self-monitoring<br/>Emit metrics]
        end
        subgraph Alerting
            A[Alerting Engine<br/>Self-monitoring<br/>Emit metrics]
        end
        
        Self_TSDB[Primary TSDB<br/>Stores self-monitoring metrics]
        Self_Alerts[Primary Alerts<br/>Alert on self-monitoring metrics]
    end
    
    subgraph External Monitoring System Separate Infra
        Ext_Prom[Standalone Prometheus<br/>Lightweight<br/>Separate cluster]
        Ext_Alerts[External Alerts<br/>PagerDuty<br/>Email]
    end
    
    subgraph External Dependencies
        PagerDuty[PagerDuty<br/>Alert notifications]
        Email[Email<br/>Alert notifications]
    end
    
    C -->|Self metrics| Self_TSDB
    K -->|Self metrics| Self_TSDB
    T -->|Self metrics| Self_TSDB
    A -->|Self metrics| Self_TSDB
    
    Self_TSDB --> Self_Alerts
    Self_Alerts --> PagerDuty
    
    C -.Monitored by external.-> Ext_Prom
    K -.Monitored by external.-> Ext_Prom
    T -.Monitored by external.-> Ext_Prom
    A -.Monitored by external.-> Ext_Prom
    
    Ext_Prom --> Ext_Alerts
    Ext_Alerts --> PagerDuty
    Ext_Alerts --> Email
    
    style Self_TSDB fill:#ff9
    style Self_Alerts fill:#9f9
    style Ext_Prom fill:#f99
    style Ext_Alerts fill:#99f
```

---

## 14. Scaling Strategy

**Flow Explanation:**

This diagram shows the horizontal scaling strategy for each component to handle growth from 100M writes/sec to 1B writes/sec.

**Scaling Approach:**

**1. Collectors (Pull-Based)**
- **Current:** 100 collectors, 1M metrics/sec each
- **Scale to 1B:** 1000 collectors, 1M metrics/sec each
- **Strategy:** Deploy more collector instances, service discovery auto-assigns endpoints

**2. Kafka**
- **Current:** 24 partitions, 4.2M msgs/sec each
- **Scale to 1B:** 240 partitions, 4.2M msgs/sec each
- **Strategy:** Add more partitions (requires rebalancing), add more brokers

**3. M3DB**
- **Current:** 100 nodes, 1M writes/sec each
- **Scale to 1B:** 1000 nodes, 1M writes/sec each
- **Strategy:** Add more nodes, consistent hashing automatically redistributes shards

**4. Alerting Engine (Flink)**
- **Current:** 12 Flink tasks
- **Scale to 1B:** 120 Flink tasks
- **Strategy:** Increase Flink parallelism, add more task managers

**5. Query Service**
- **Current:** 5 query service instances
- **Scale to 1B:** 50 query service instances
- **Strategy:** Add more instances behind load balancer, cache layer scales with Redis cluster

**Scaling Formula:**
```
To scale from X to 10X writes/sec:
- Collectors: 10x more instances
- Kafka: 10x more partitions (or increase per-partition throughput)
- M3DB: 10x more nodes (linear scaling)
- Alerting: 10x more Flink parallelism
- Query Service: Minimal scaling (reads don't scale linearly with writes)
```

**Performance:**
- 100M writes/sec → $244k/month infrastructure cost
- 1B writes/sec → $2.4M/month infrastructure cost (linear scaling)

**Benefits:**
- Horizontal scalability (add nodes, not bigger nodes)
- Linear cost scaling (10x writes = 10x cost)
- No major architectural changes needed

```mermaid
graph TB
    subgraph Current Scale 100M writes/sec
        C_Curr[Collectors: 100<br/>1M writes/sec each]
        K_Curr[Kafka: 24 partitions<br/>4.2M msgs/sec each]
        T_Curr[M3DB: 100 nodes<br/>1M writes/sec each]
        A_Curr[Flink: 12 tasks<br/>Parallelism: 12]
        Q_Curr[Query Service: 5 instances<br/>16k QPS reads]
    end
    
    subgraph Target Scale 1B writes/sec 10x growth
        C_Target[Collectors: 1000<br/>1M writes/sec each]
        K_Target[Kafka: 240 partitions<br/>4.2M msgs/sec each]
        T_Target[M3DB: 1000 nodes<br/>1M writes/sec each]
        A_Target[Flink: 120 tasks<br/>Parallelism: 120]
        Q_Target[Query Service: 50 instances<br/>160k QPS reads]
    end
    
    subgraph Scaling Strategy
        Horizontal[Horizontal Scaling<br/>Add more instances<br/>No bigger nodes]
        Linear[Linear Cost Scaling<br/>10x writes = 10x nodes<br/>= 10x cost]
        Auto[Auto-Scaling<br/>Kubernetes HPA<br/>Based on metrics]
    end
    
    C_Curr -.10x scale.-> C_Target
    K_Curr -.10x scale.-> K_Target
    T_Curr -.10x scale.-> T_Target
    A_Curr -.10x scale.-> A_Target
    Q_Curr -.10x scale.-> Q_Target
    
    C_Target --> Horizontal
    K_Target --> Horizontal
    T_Target --> Horizontal
    
    Horizontal --> Linear
    Linear --> Auto
    
    style C_Target fill:#ff9
    style K_Target fill:#9f9
    style T_Target fill:#f99
    style Horizontal fill:#99f
```

---

**Total:** This comprehensive diagram document covers all aspects of the distributed monitoring system architecture with 14 detailed Mermaid diagrams and flow explanations.
