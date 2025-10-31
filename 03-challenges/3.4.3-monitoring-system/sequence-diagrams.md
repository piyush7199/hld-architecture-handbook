# Distributed Monitoring System - Sequence Diagrams

## Table of Contents

1. [Metrics Collection Flow (Pull-Based Scrape)](#1-metrics-collection-flow-pull-based-scrape)
2. [Metrics Collection Flow (Push-Based Agent)](#2-metrics-collection-flow-push-based-agent)
3. [Kafka Buffering and TSDB Write](#3-kafka-buffering-and-tsdb-write)
4. [Real-Time Alert Evaluation](#4-real-time-alert-evaluation)
5. [Rollup Aggregation Processing](#5-rollup-aggregation-processing)
6. [Dashboard Query Execution (Cache Hit)](#6-dashboard-query-execution-cache-hit)
7. [Dashboard Query Execution (Cache Miss)](#7-dashboard-query-execution-cache-miss)
8. [Alert Notification Flow](#8-alert-notification-flow)
9. [Cardinality Limit Rejection](#9-cardinality-limit-rejection)
10. [Multi-Region Metric Replication](#10-multi-region-metric-replication)
11. [Tiered Storage Transition](#11-tiered-storage-transition)
12. [Failure Recovery - Collector Restart](#12-failure-recovery---collector-restart)

---

## 1. Metrics Collection Flow (Pull-Based Scrape)

**Flow:**

Shows the complete pull-based metrics collection flow from endpoint scraping to Kafka write.

**Steps:**

1. **Service Discovery (0ms)** - Collector queries Kubernetes API for pod list
2. **Schedule Scrape (1000ms)** - Collector schedules scrape for endpoint every 1 second
3. **HTTP GET (10ms)** - Collector sends GET to `/metrics` endpoint
4. **Endpoint Response (5ms)** - Endpoint returns Prometheus text format
5. **Parse Metrics (3ms)** - Parse metric name, labels, value, timestamp
6. **Local Buffer (1ms)** - Write to local buffer (batch 100 metrics)
7. **Kafka Write (8ms)** - Async write to Kafka (non-blocking)
8. **Total latency:** ~27ms from scrape to Kafka

**Performance:**

- Scrape frequency: 1 second per endpoint
- Batch size: 100 metrics per Kafka write
- Parallelism: 1000 scraper threads per collector
- Throughput: 1M metrics/sec per collector

```mermaid
sequenceDiagram
    participant SD as Service Discovery<br/>(Kubernetes API)
    participant Collector as Prometheus Collector
    participant Endpoint as Monitored Endpoint<br/>:9090/metrics
    participant Buffer as Local Buffer<br/>(100 metrics batch)
    participant Kafka as Kafka Topic<br/>metrics.raw
    Collector ->> SD: List endpoints (0ms)
    SD -->> Collector: Pod list [pod-1, pod-2, ...]
    Note over Collector: Schedule scrape every 1 second

    loop Every 1 second per endpoint
        Collector ->> Endpoint: HTTP GET /metrics (10ms)
        Note over Endpoint: Collect metrics:<br/>cpu_usage, memory_usage, etc.
        Endpoint -->> Collector: Response 200 OK<br/>Prometheus format<br/>metric_name{labels} value timestamp (5ms)
        Collector ->> Collector: Parse metrics (3ms)
        Note over Collector: Extract:<br/>- metric_name<br/>- labels<br/>- value<br/>- timestamp
        Collector ->> Buffer: Write parsed metric (1ms)

        alt Buffer Full (100 metrics)
            Buffer ->> Kafka: Async write batch (8ms)<br/>Partition: hash(metric+labels)
            Kafka -->> Buffer: Ack (offset: 12345)
            Buffer ->> Collector: Batch written (27ms total)
        end
    end

    Note over Collector, Kafka: Throughput: 1M metrics/sec per collector<br/>Latency: 27ms p99
```

---

## 2. Metrics Collection Flow (Push-Based Agent)

**Flow:**

Shows push-based metrics collection where agents on endpoints push metrics to collectors.

**Steps:**

1. **Agent Collection (0ms)** - Agent collects system metrics (CPU, memory, disk)
2. **Aggregation (1000ms)** - Agent aggregates over 1-second window
3. **Batching (1000ms)** - Agent batches 100 metrics before sending
4. **HTTP POST (15ms)** - Agent POSTs batch to collector
5. **Collector Validation (3ms)** - Collector validates format and rate limits
6. **Kafka Write (8ms)** - Collector writes to Kafka
7. **Total latency:** ~26ms from collection to Kafka

**Performance:**

- Agent collection frequency: 1 second
- Batch size: 100 metrics per POST
- Retry logic: Exponential backoff (1s, 2s, 4s, 8s)
- Agent resource usage: 50 MB memory, 2% CPU

```mermaid
sequenceDiagram
    participant App as Application<br/>(Custom metrics)
    participant Agent as Telegraf Agent
    participant AgentBuffer as Agent Buffer<br/>(1-second window)
    participant Collector as Collector<br/>HTTP Receiver
    participant Kafka as Kafka Topic<br/>metrics.raw
    App ->> Agent: Emit custom metric (0ms)<br/>http_requests_total: 1
    Note over Agent: Collect system metrics every 1 second
    Agent ->> Agent: Collect CPU usage (0ms)
    Agent ->> Agent: Collect memory usage
    Agent ->> Agent: Collect disk I/O
    Agent ->> Agent: Collect network bytes
    Agent ->> AgentBuffer: Write metrics (1ms)<br/>100 metrics collected
    Note over AgentBuffer: Wait for batch (100 metrics or 1 second)
    AgentBuffer ->> Collector: HTTP POST /metrics (15ms)<br/>JSON payload: [{metric, labels, value, timestamp}, ...]
    Collector ->> Collector: Validate request (3ms)
    Note over Collector: Check:<br/>- Valid format<br/>- Rate limit (1000 req/min per agent)<br/>- Label whitelist

    alt Validation Success
        Collector ->> Kafka: Async write batch (8ms)<br/>Partition: hash(metric+labels)
        Kafka -->> Collector: Ack (offset: 67890)
        Collector -->> AgentBuffer: 200 OK (26ms total)
        AgentBuffer -->> Agent: Batch sent successfully
    else Validation Failure
        Collector -->> AgentBuffer: 400 Bad Request<br/>Invalid label: user_id
        AgentBuffer ->> Agent: Retry with exponential backoff
        Note over Agent: Retry after 1s, 2s, 4s, 8s...
    end

    Note over Agent, Kafka: Throughput: 100 metrics/sec per agent<br/>1M agents = 100M metrics/sec
```

---

## 3. Kafka Buffering and TSDB Write

**Flow:**

Shows how Kafka buffers metrics and TSDB writers consume and write to M3DB.

**Steps:**

1. **Produce to Kafka (0ms)** - Collector writes metrics to Kafka
2. **Kafka Persistence (5ms)** - Kafka persists to disk (3x replication)
3. **Consumer Poll (10ms)** - TSDB writer polls Kafka (batch size: 1000)
4. **Transform (5ms)** - Transform to M3DB format
5. **M3DB Write (20ms)** - Write to M3DB (batch write to 3 replicas)
6. **Kafka Commit (2ms)** - Commit offset back to Kafka
7. **Total latency:** ~42ms from Kafka produce to M3DB write

**Performance:**

- Kafka throughput: 100M msgs/sec (24 partitions)
- Consumer lag: <1000 messages (healthy)
- TSDB write throughput: 100M writes/sec (100 nodes)
- Durability: 3x replication (no data loss)

```mermaid
sequenceDiagram
    participant Collector
    participant Kafka
    participant KafkaLog as Kafka Log<br/>(Disk + 3x replication)
    participant TSDBWriter as TSDB Writer Consumer
    participant M3Coordinator
    participant M3DB1 as M3DB Node 1<br/>(Shard 0)
    participant M3DB2 as M3DB Node 2<br/>(Shard 0 replica)
    participant M3DB3 as M3DB Node 3<br/>(Shard 0 replica)
    Collector ->> Kafka: Produce metric (0ms)<br/>Async, non-blocking
    Note over Collector: Producer continues<br/>without waiting
    Kafka ->> KafkaLog: Write to log (5ms)<br/>Persist to disk<br/>Replicate to 2 other brokers
    KafkaLog -->> Kafka: Persisted (3x replication)
    Kafka -->> Collector: Ack (offset: 12345)
    Note over TSDBWriter: Poll every 100ms<br/>Batch size: 1000 messages
    TSDBWriter ->> Kafka: Poll messages (10ms)<br/>Consumer group: tsdb-writers
    Kafka -->> TSDBWriter: 1000 messages<br/>(from partition 0)
    TSDBWriter ->> TSDBWriter: Transform to M3DB format (5ms)
    Note over TSDBWriter: Convert Prometheus format<br/>to M3DB data points
    TSDBWriter ->> M3Coordinator: Write batch (20ms)<br/>1000 data points
    M3Coordinator ->> M3Coordinator: Shard placement<br/>Determine target shards

    par Write to 3 replicas (RF=3)
        M3Coordinator ->> M3DB1: Write shard 0 (primary)
        M3Coordinator ->> M3DB2: Write shard 0 (replica 1)
        M3Coordinator ->> M3DB3: Write shard 0 (replica 2)
        M3DB1 -->> M3Coordinator: Write complete
        M3DB2 -->> M3Coordinator: Write complete
        M3DB3 -->> M3Coordinator: Write complete
    end

    M3Coordinator -->> TSDBWriter: Batch write complete (20ms)
    TSDBWriter ->> Kafka: Commit offset (2ms)<br/>Offset: 13345
    Kafka -->> TSDBWriter: Committed
    Note over Collector, M3DB3: Total latency: ~42ms<br/>Write throughput: 100M/sec<br/>Durability: 3x replication
```

---

## 4. Real-Time Alert Evaluation

**Flow:**

Shows the stateful stream processing flow for real-time alert evaluation using Apache Flink.

**Steps:**

1. **Consume Metrics (0ms)** - Flink consumes from Kafka
2. **Group by Rule (5ms)** - Group metrics by alert rule
3. **Windowed Aggregation (5000ms)** - 5-minute tumbling window
4. **Evaluate Condition (10ms)** - Check if avg > 80%
5. **State Check (5ms)** - Check if alert already firing
6. **Trigger Alert (20ms)** - Publish to alerts.triggered topic
7. **Total latency:** ~4 seconds from metric to alert notification

**Performance:**

- Alert evaluation latency: <5 seconds (p99)
- Concurrent alert rules: 10,000 rules
- State size: <10 GB per Flink task
- Exactly-once semantics: No duplicate alerts

```mermaid
sequenceDiagram
    participant Kafka as Kafka metrics.raw
    participant Flink as Flink Alerting Engine
    participant StateStore as State Store<br/>(RocksDB)
    participant AlertKafka as Kafka alerts.triggered
    participant NotifySvc as Notification Service

    loop Continuous stream processing
        Kafka ->> Flink: Consume metrics (0ms)<br/>cpu_usage{service="api-gateway"}
        Flink ->> Flink: Group by alert rule (5ms)
        Note over Flink: Rule: HighCPUUsage<br/>Condition: avg(cpu_usage) > 80<br/>for 5 minutes
        Note over Flink: 5-minute tumbling window<br/>Aggregate metrics
        Flink ->> Flink: Window complete (5000ms)
        Note over Flink: Window data:<br/>- Count: 300 samples<br/>- Sum: 24,500<br/>- Avg: 81.67%
        Flink ->> Flink: Evaluate condition (10ms)
        Note over Flink: avg (81.67%) > threshold (80%)<br/>Condition MET
        Flink ->> StateStore: Check alert state (5ms)<br/>Key: HighCPUUsage:api-gateway
        StateStore -->> Flink: State: NOT_FIRING

        alt Condition Met AND Not Already Firing
            Flink ->> StateStore: Update state (5ms)<br/>State: FIRING
            StateStore -->> Flink: Updated
            Flink ->> AlertKafka: Publish alert (20ms)
            Note over Flink: Alert: {<br/> name: "HighCPUUsage",<br/> service: "api-gateway",<br/> value: 81.67%,<br/> threshold: 80%,<br/> severity: "warning"<br/>}
            AlertKafka -->> Flink: Published (offset: 456)
            AlertKafka ->> NotifySvc: Alert event
            NotifySvc ->> NotifySvc: Send to Slack
            Note over NotifySvc: POST to Slack webhook<br/>#alerts channel
            NotifySvc -->> AlertKafka: Notification sent
        else Already Firing
            Flink ->> Flink: Skip (deduplicate)
            Note over Flink: Don't send duplicate alert
        end

        Note over Flink: Checkpoint state (every 60s)<br/>For fault tolerance
        Flink ->> StateStore: Checkpoint (60000ms)
    end

    Note over Kafka, NotifySvc: Alert latency: <5 seconds<br/>No duplicate alerts (deduplication)<br/>Fault-tolerant (checkpointing)
```

---

## 5. Rollup Aggregation Processing

**Flow:**

Shows the real-time aggregation flow for downsampling raw metrics to 1-minute rollups.

**Steps:**

1. **Consume Raw Metrics (0ms)** - Kafka Streams consumes raw metrics
2. **Group by Metric (5ms)** - Group by metric_name + labels
3. **1-Minute Window (60000ms)** - Tumbling window aggregation
4. **Compute Aggregates (10ms)** - Count, sum, min, max, avg, p50, p95, p99
5. **Write Rollup (15ms)** - Write to M3DB rollup table
6. **Emit Rollup (60025ms total)** - Emit aggregated data point

**Performance:**

- Window size: 1 minute (tumbling)
- Aggregation latency: <30 seconds after window close
- Storage reduction: 60x (1-minute vs 1-second)
- Rollup throughput: 1.7M rollups/sec (100M raw / 60)

```mermaid
sequenceDiagram
    participant Kafka as Kafka metrics.raw<br/>100M msgs/sec
    participant KStreams as Kafka Streams Aggregator
    participant Window as Windowed State<br/>(RocksDB)
    participant M3DB as M3DB Rollup Table<br/>metrics_rollup_1min

    loop Every second (60 seconds)
        Kafka ->> KStreams: Consume metric (0ms)<br/>cpu_usage{host="server-1"} 65.2 @ 12:00:00
        KStreams ->> KStreams: Group by metric+labels (5ms)
        Note over KStreams: Key: cpu_usage:host=server-1<br/>Window: 12:00:00 - 12:00:59
        KStreams ->> Window: Update window state (2ms)
        Note over Window: State:<br/>- Count: 1 → 2<br/>- Sum: 65.2 → 132.7<br/>- Min: 65.2<br/>- Max: 67.5<br/>- Values: [65.2, 67.5]
        Window -->> KStreams: State updated
    end

    Note over KStreams: Window closes at 12:01:00<br/>(60 seconds elapsed)
    KStreams ->> Window: Finalize window (10ms)
    Window -->> KStreams: Final state:<br/>- Count: 60<br/>- Sum: 4080.5<br/>- Min: 65.2<br/>- Max: 75.1<br/>- Values: [sorted for percentiles]
    KStreams ->> KStreams: Compute aggregates (10ms)
    Note over KStreams: Calculate:<br/>- avg: sum / count = 68.0<br/>- p50: values[30] = 67.8<br/>- p95: values[57] = 74.2<br/>- p99: values[59] = 75.0
    KStreams ->> M3DB: Write rollup (15ms)
    Note over KStreams: Rollup: {<br/> metric: "cpu_usage_1min",<br/> labels: {host: "server-1"},<br/> timestamp: 12:00:00,<br/> count: 60,<br/> sum: 4080.5,<br/> min: 65.2,<br/> max: 75.1,<br/> avg: 68.0,<br/> p50: 67.8,<br/> p95: 74.2,<br/> p99: 75.0<br/>}
    M3DB -->> KStreams: Write complete
    Note over Kafka, M3DB: Storage reduction: 60x<br/>(60 raw points → 1 rollup)<br/>Query speedup: 60x<br/>(fetch 1 point vs 60)
```

---

## 6. Dashboard Query Execution (Cache Hit)

**Flow:**

Shows the fast dashboard query execution flow when query result is cached in Redis.

**Steps:**

1. **Grafana Query (0ms)** - User opens dashboard
2. **Query API (5ms)** - API receives PromQL query
3. **Query Optimizer (3ms)** - Optimize query (select rollup)
4. **Redis Lookup (2ms)** - Check cache for query result
5. **Cache HIT (2ms)** - Result found in cache
6. **Return Response (10ms total)** - Return cached result to Grafana

**Performance:**

- Cache hit latency: <10ms (p50)
- Cache hit rate: 80% (popular dashboards)
- Redis cluster: 3 nodes, 100 GB memory
- TTL: 1 minute (recent data), 1 hour (old data)

```mermaid
sequenceDiagram
    participant Grafana as Grafana Dashboard
    participant QueryAPI as Query API
    participant Optimizer as Query Optimizer
    participant Redis as Redis Cache
    participant User as User
    User ->> Grafana: Open dashboard (0ms)<br/>"CPU Usage Last 1 Hour"
    Grafana ->> QueryAPI: PromQL query (5ms)<br/>avg(cpu_usage{service="api-gateway"})[1h]
    QueryAPI ->> Optimizer: Optimize query (3ms)
    Note over Optimizer: Analyze query:<br/>- Time range: 1 hour<br/>- Select rollup: use raw (1-second)<br/>- Expected points: 3600<br/>- Predicate pushdown: service="api-gateway"
    Optimizer ->> Optimizer: Generate cache key
    Note over Optimizer: Key: hash(<br/> query="avg(cpu_usage{service='api-gateway'})",<br/> time_range="now-1h to now"<br/>)<br/>= "a3f5b2c8..."
    Optimizer ->> Redis: GET a3f5b2c8... (2ms)

    alt Cache HIT (80% probability)
        Redis -->> Optimizer: Cache hit (2ms)<br/>Result: [timestamps, values]
        Note over Redis: Cached result:<br/>[<br/> [12:00:00, 65.2],<br/> [12:00:01, 67.5],<br/> ...<br/> [12:59:59, 72.3]<br/>]
        Optimizer -->> QueryAPI: Cached result (7ms)
        QueryAPI -->> Grafana: JSON response (10ms total)
        Note over QueryAPI: Response:<br/>{<br/> status: "success",<br/> data: {<br/> resultType: "matrix",<br/> result: [{<br/> metric: {...},<br/> values: [...]<br/> }]<br/> }<br/>}
        Grafana ->> Grafana: Render graph (50ms)
        Grafana -->> User: Dashboard displayed (60ms total)
    end

    Note over Grafana, User: Cache hit latency: <10ms<br/>Cache hit rate: 80%<br/>Total dashboard load: <60ms
```

---

## 7. Dashboard Query Execution (Cache Miss)

**Flow:**

Shows the query execution flow when cache miss occurs, requiring query to TSDB.

**Steps:**

1. **Grafana Query (0ms)** - User opens dashboard
2. **Query API (5ms)** - API receives query
3. **Query Optimizer (3ms)** - Optimize query
4. **Redis Lookup (2ms)** - Check cache (MISS)
5. **M3DB Query (500ms)** - Execute on M3DB (parallel across shards)
6. **Store in Cache (10ms)** - Cache result for future requests
7. **Return Response (520ms total)** - Return result to Grafana

**Performance:**

- Cache miss latency: <1 second (p99)
- M3DB query: 500ms (depends on time range and cardinality)
- Parallel execution: Query 4096 shards in parallel
- Result merge: <50ms

```mermaid
sequenceDiagram
    participant Grafana
    participant QueryAPI
    participant Optimizer
    participant Redis
    participant M3Coordinator
    participant M3Shard1 as M3DB Shard 1
    participant M3Shard2 as M3DB Shard 2
    participant M3ShardN as M3DB Shard N
    Grafana ->> QueryAPI: PromQL query (5ms)<br/>avg(cpu_usage{service="api-gateway"})[1h]
    QueryAPI ->> Optimizer: Optimize query (3ms)
    Optimizer ->> Redis: GET cache_key (2ms)

    alt Cache MISS (20% probability)
        Redis -->> Optimizer: Cache miss (2ms)
        Optimizer ->> M3Coordinator: Execute query (10ms)
        Note over Optimizer: Optimized query:<br/>- Time range: now-1h to now<br/>- Metric: cpu_usage<br/>- Filter: service="api-gateway"<br/>- Resolution: raw (1-second)
        M3Coordinator ->> M3Coordinator: Determine target shards
        Note over M3Coordinator: Shard selection:<br/>- Hash(cpu_usage:service=api-gateway)<br/>- Target shards: [42, 789, 1234, ...]<br/>- 120 shards contain this time series

        par Parallel query to shards
            M3Coordinator ->> M3Shard1: Query shard 42 (500ms)
            M3Shard1 -->> M3Coordinator: 1200 data points
            M3Coordinator ->> M3Shard2: Query shard 789 (500ms)
            M3Shard2 -->> M3Coordinator: 1200 data points
            M3Coordinator ->> M3ShardN: Query shard 1234 (500ms)
            M3ShardN -->> M3Coordinator: 1200 data points
        end

        M3Coordinator ->> M3Coordinator: Merge results (50ms)
        Note over M3Coordinator: Merge 120 shard results<br/>Total points: 3600<br/>Apply avg() aggregation
        M3Coordinator -->> Optimizer: Query result (560ms)
        Note over M3Coordinator: Result:<br/>3600 data points<br/>(1 per second for 1 hour)
        Optimizer ->> Redis: SET cache_key (10ms)<br/>TTL: 60 seconds
        Redis -->> Optimizer: Cached
        Optimizer -->> QueryAPI: Result (570ms)
        QueryAPI -->> Grafana: JSON response (575ms total)
        Grafana ->> Grafana: Render graph (50ms)
        Grafana -->> Grafana: Dashboard displayed (625ms total)
    end

    Note over Grafana, M3ShardN: Cache miss latency: <1 second<br/>Parallel execution: 120 shards<br/>Result cached for next request
```

---

## 8. Alert Notification Flow

**Flow:**

Shows the end-to-end alert notification flow from alert trigger to external notification systems.

**Steps:**

1. **Alert Triggered (0ms)** - Flink publishes to alerts.triggered topic
2. **Notification Service Consumes (5ms)** - Consume alert event
3. **Deduplicate (2ms)** - Check if alert already sent recently
4. **Route Notification (3ms)** - Determine notification channels (email, Slack, PagerDuty)
5. **Send Notifications (200ms)** - Parallel send to all channels
6. **Update State (5ms)** - Mark alert as notified
7. **Total latency:** ~215ms from alert trigger to notification sent

**Performance:**

- Notification latency: <500ms (p99)
- Throughput: 1000 alerts/sec
- Retry logic: 3 retries with exponential backoff
- Notification channels: Email, Slack, PagerDuty, Webhook

```mermaid
sequenceDiagram
    participant Flink as Flink Alerting Engine
    participant Kafka as Kafka alerts.triggered
    participant NotifySvc as Notification Service
    participant DedupStore as Dedup Store<br/>(Redis)
    participant Email
    participant Slack
    participant PagerDuty
    Flink ->> Kafka: Publish alert (0ms)
    Note over Flink: Alert: {<br/> name: "HighCPUUsage",<br/> service: "api-gateway",<br/> severity: "warning",<br/> value: 85%,<br/> threshold: 80%,<br/> timestamp: 1234567890<br/>}
    Kafka ->> NotifySvc: Consume alert (5ms)
    NotifySvc ->> DedupStore: Check if already sent (2ms)
    Note over NotifySvc: Key: alert:HighCPUUsage:api-gateway<br/>TTL: 5 minutes

    alt Not Sent Recently
        DedupStore -->> NotifySvc: Not found (first time)
        NotifySvc ->> NotifySvc: Route notification (3ms)
        Note over NotifySvc: Routing rules:<br/>- severity=warning → Slack<br/>- severity=critical → Slack + PagerDuty + Email<br/><br/>This alert (warning) → Slack only

        par Send to channels (parallel)
            NotifySvc ->> Slack: POST webhook (200ms)
            Note over NotifySvc, Slack: Payload:<br/>{<br/> channel: "#alerts",<br/>  text: "⚠️ HighCPUUsage",<br/>  attachments: [{<br/>    title: "api-gateway CPU high",<br/>    text: "Current: 85%, Threshold: 80%",<br/>    color: "warning"<br/>  }]<br/>}
            Slack -->> NotifySvc: 200 OK (200ms)
            Note over Email, PagerDuty: (Not sent for warning severity)
        end

        NotifySvc ->> DedupStore: Mark as sent (5ms)<br/>TTL: 5 minutes
        DedupStore -->> NotifySvc: Stored
        NotifySvc ->> Kafka: Commit offset (2ms)
        Kafka -->> NotifySvc: Committed
        Note over NotifySvc: Log notification sent<br/>Metric: notifications_sent_total++

    else Already Sent Recently (within 5 min)
        DedupStore -->> NotifySvc: Found (sent 2 min ago)
        NotifySvc ->> NotifySvc: Skip (deduplicate)
        Note over NotifySvc: Don't spam notifications<br/>Wait 5 minutes before resending
        NotifySvc ->> Kafka: Commit offset (2ms)
    end

    Note over Flink, PagerDuty: Notification latency: <500ms<br/>Deduplication window: 5 minutes<br/>Parallel delivery: All channels
```

---

## 9. Cardinality Limit Rejection

**Flow:**

Shows the cardinality control flow where metrics with too many unique label combinations are rejected.

**Steps:**

1. **Collector Receives Metric (0ms)** - Metric with high-cardinality label
2. **Label Validation (2ms)** - Check against whitelist/blacklist
3. **Cardinality Check (5ms)** - Query current cardinality for this metric
4. **Limit Check (2ms)** - Check if adding this would exceed limit (10k unique combos)
5. **Rejection (9ms total)** - Return 400 error
6. **Alert on High Cardinality (10ms)** - Alert SRE team

**Performance:**

- Validation latency: <10ms
- Cardinality limit: 10k unique label combinations per metric
- Label limit: Max 20 labels per metric
- Blacklist: `user_id`, `request_id`, `session_id`, `trace_id`

```mermaid
sequenceDiagram
    participant Endpoint
    participant Collector
    participant LabelValidator as Label Validator
    participant CardTracker as Cardinality Tracker
    participant CardDB as Cardinality DB<br/>(PostgreSQL)
    participant AlertSvc as Alert Service
    Endpoint ->> Collector: Emit metric (0ms)
    Note over Endpoint: ❌ BAD metric:<br/>http_requests{<br/> service="api",<br/> user_id="user_12345", ← High cardinality!<br/> request_id="req_abc" ← High cardinality!<br/>} 1
    Collector ->> LabelValidator: Validate labels (2ms)
    LabelValidator ->> LabelValidator: Check whitelist
    Note over LabelValidator: Whitelist:<br/>✅ service<br/>✅ method<br/>✅ endpoint<br/>✅ status_code<br/>✅ region
    LabelValidator ->> LabelValidator: Check blacklist
    Note over LabelValidator: Blacklist:<br/>❌ user_id (HIGH CARDINALITY)<br/>❌ request_id (HIGH CARDINALITY)<br/>❌ session_id (HIGH CARDINALITY)<br/>❌ trace_id (HIGH CARDINALITY)

    alt Invalid Labels (Blacklisted)
        LabelValidator -->> Collector: Validation FAILED (2ms)<br/>Reason: Blacklisted labels detected
        Collector ->> AlertSvc: Alert high-cardinality attempt (10ms)
        Note over Collector, AlertSvc: Alert: {<br/> title: "High-Cardinality Label Detected",<br/> endpoint: "10.0.1.5:9090",<br/> metric: "http_requests",<br/> invalid_labels: ["user_id", "request_id"],<br/> action: "Rejected"<br/>}
        Collector -->> Endpoint: 400 Bad Request (9ms)<br/>Error: Invalid labels detected<br/>Blacklisted: user_id, request_id<br/>Use approved labels only
        Note over Endpoint: Fix code to remove<br/>high-cardinality labels

    else Valid Labels
        LabelValidator -->> Collector: Validation OK
        Collector ->> CardTracker: Check cardinality (5ms)<br/>Metric: http_requests
        CardTracker ->> CardDB: Query current cardinality
        Note over CardTracker, CardDB: SELECT COUNT(DISTINCT labels)<br/>FROM metrics<br/>WHERE metric_name = 'http_requests'
        CardDB -->> CardTracker: Current: 9,500 unique combos
        CardTracker ->> CardTracker: Check limit (2ms)
        Note over CardTracker: Limit: 10,000 unique combos<br/>Current: 9,500<br/>New combo? Check if exists

        alt Exceeds Limit
            CardTracker -->> Collector: REJECTED (7ms)<br/>Reason: Cardinality limit exceeded
            Collector -->> Endpoint: 429 Too Many Requests<br/>Cardinality limit reached:<br/>10,000 unique label combinations

        else Within Limit
            CardTracker -->> Collector: ACCEPTED (7ms)
            Note over CardTracker: New combo: 9,501 / 10,000
            Collector ->> Collector: Write to Kafka
        end
    end

    Note over Endpoint, AlertSvc: Cardinality limit: 10k combos<br/>Label limit: 20 labels<br/>Prevents TSDB memory explosion
```

---

## 10. Multi-Region Metric Replication

**Flow:**

Shows cross-region metric replication from regional Kafka to global Kafka using MirrorMaker 2.

**Steps:**

1. **Regional Collection (0ms)** - Metrics collected in EU-West region
2. **Regional Kafka (5ms)** - Write to EU-West Kafka
3. **MirrorMaker 2 Consume (1000ms)** - MirrorMaker polls regional Kafka
4. **Transform (10ms)** - Transform and enrich with region metadata
5. **Global Kafka Write (20ms)** - Write to global Kafka in US-East
6. **Global Processing (1030ms total)** - Process in global TSDB
7. **Replication lag:** 1-5 seconds (acceptable)

**Performance:**

- Replication lag: 1-5 seconds (p99)
- Throughput: 30M metrics/sec (EU region)
- Cross-region bandwidth: 50 Gbps
- Failover time: <30 seconds (if regional Kafka fails)

```mermaid
sequenceDiagram
    participant EU_Endpoint as EU Endpoint<br/>(server-eu-123)
    participant EU_Collector as EU Collector
    participant EU_Kafka as EU Kafka Cluster<br/>(eu-west-1)
    participant MirrorMaker as MirrorMaker 2<br/>(Replication)
    participant Global_Kafka as Global Kafka Cluster<br/>(us-east-1)
    participant Global_TSDB as Global M3DB
    EU_Endpoint ->> EU_Collector: Emit metric (0ms)<br/>cpu_usage{host="server-eu-123"} 65.2
    EU_Collector ->> EU_Kafka: Write to regional Kafka (5ms)<br/>Topic: metrics.raw
    Note over EU_Collector, EU_Kafka: Regional processing:<br/>- Low latency (5ms)<br/>- Regional dashboards<br/>- Regional alerts
    EU_Kafka -->> EU_Collector: Ack (offset: 123456)
    Note over MirrorMaker: Poll regional Kafka every 1 second<br/>Batch size: 10,000 messages
    MirrorMaker ->> EU_Kafka: Poll messages (1000ms)<br/>Consumer group: mm2-replication
    EU_Kafka -->> MirrorMaker: 10,000 messages
    MirrorMaker ->> MirrorMaker: Transform (10ms)
    Note over MirrorMaker: Enrich with metadata:<br/>- Add label: source_region="eu-west"<br/>- Add label: replicated="true"<br/>- Preserve original timestamp
    MirrorMaker ->> Global_Kafka: Write to global Kafka (20ms)<br/>Topic: metrics.global<br/>Cross-region network
    Note over MirrorMaker, Global_Kafka: Cross-region:<br/>- 50 Gbps bandwidth<br/>- ~20ms latency (transatlantic)<br/>- Compressed (snappy)
    Global_Kafka -->> MirrorMaker: Ack (offset: 789012)
    Global_Kafka ->> Global_TSDB: Consume and write (500ms)
    Note over Global_Kafka, Global_TSDB: Global aggregation:<br/>- All regions combined<br/>- Global dashboards<br/>- Cross-region analysis
    Note over EU_Endpoint, Global_TSDB: Replication lag: 1-5 seconds<br/>Regional query: <50ms<br/>Global query: <1 second
```

---

## 11. Tiered Storage Transition

**Flow:**

Shows the automated data lifecycle management flow for transitioning data from hot (SSD) to warm (HDD) to cold (S3)
tiers.

**Steps:**

1. **Day 0-7: Hot Tier (0ms)** - Data written to SSD (fast access)
2. **Day 7: Lifecycle Manager (0ms)** - Detects data >7 days old
3. **Transition to Warm (3600000ms)** - Copy to HDD, verify, delete from SSD
4. **Day 90: Lifecycle Manager (0ms)** - Detects data >90 days old
5. **Transition to Cold (7200000ms)** - Copy to S3, verify, delete from HDD
6. **Day 730 (optional): Delete (0ms)** - Delete from S3 based on retention policy

**Performance:**

- Hot tier latency: <10ms (SSD)
- Warm tier latency: <50ms (HDD)
- Cold tier latency: 1-5 seconds (S3)
- Transition time: 1-2 hours (background process)

```mermaid
sequenceDiagram
    participant Writer as TSDB Writer
    participant Hot as Hot Tier<br/>SSD<br/>0-7 days
    participant LM as Lifecycle Manager<br/>(Cron job)
    participant Warm as Warm Tier<br/>HDD<br/>7-90 days
    participant Cold as Cold Tier<br/>S3/Glacier<br/>90+ days
    Note over Writer, Hot: Day 0: Data written
    Writer ->> Hot: Write metric data (0ms)<br/>cpu_usage @ 2025-10-01
    Hot -->> Writer: Write complete<br/>Stored on SSD
    Note over Hot: Days 1-7: Data remains in hot tier<br/>Fast access (<10ms)
    Note over LM: Day 7: Lifecycle check (cron: daily 2 AM)
    LM ->> Hot: Query data older than 7 days
    Hot -->> LM: Data: cpu_usage @ 2025-10-01<br/>Age: 7 days
    LM ->> LM: Trigger transition to warm (0ms)
    Note over LM: Decision: Age > 7 days<br/>Action: Move to HDD
    LM ->> Warm: Copy data (3600000ms = 1 hour)<br/>Background process<br/>800 TB of data
    Note over LM, Warm: Parallel copy:<br/>- 10 Gbps network<br/>- Compressed transfer<br/>- Checksummed
    Warm -->> LM: Copy complete<br/>Verified
    LM ->> Hot: Delete data (10000ms)
    Hot -->> LM: Deleted from SSD<br/>800 TB freed
    Note over Warm: Days 7-90: Data in warm tier<br/>Medium access (<50ms)
    Note over LM: Day 90: Lifecycle check
    LM ->> Warm: Query data older than 90 days
    Warm -->> LM: Data: cpu_usage @ 2025-10-01<br/>Age: 90 days
    LM ->> LM: Trigger transition to cold (0ms)
    Note over LM: Decision: Age > 90 days<br/>Action: Move to S3
    LM ->> Cold: Upload to S3 (7200000ms = 2 hours)<br/>Background upload<br/>8 PB of data
    Note over LM, Cold: S3 upload:<br/>- Multi-part upload<br/>- Compressed (gzip)<br/>- Glacier storage class
    Cold -->> LM: Upload complete<br/>Verified
    LM ->> Warm: Delete data (10000ms)
    Warm -->> LM: Deleted from HDD<br/>8 PB freed
    Note over Cold: Days 90+: Data in cold tier<br/>Slow access (1-5 seconds)
    Note over LM: Day 730 (optional): Delete old data
    LM ->> Cold: Query data older than 2 years
    Cold -->> LM: Data: cpu_usage @ 2025-10-01<br/>Age: 730 days
    LM ->> Cold: Delete data (1000ms)
    Cold -->> LM: Deleted from S3
    Note over Writer, Cold: Cost savings: 4x cheaper<br/>(vs SSD-only for all data)<br/>Hot: $80k/mo, Warm: $160k/mo, Cold: $4k/mo
```

---

## 12. Failure Recovery - Collector Restart

**Flow:**

Shows the failure recovery flow when a Prometheus collector crashes and restarts.

**Steps:**

1. **Collector Crash (0ms)** - Collector process dies (OOM, crash, etc.)
2. **Kubernetes Detects (10000ms)** - Liveness probe fails
3. **Pod Restart (30000ms)** - Kubernetes restarts pod
4. **Service Discovery (5000ms)** - New collector fetches endpoint list
5. **Resume Scraping (36000ms)** - Resume scraping from last checkpoint
6. **Gap Detection (40000ms)** - Detect missing data for 36-second window
7. **Alerting (45000ms)** - Alert on collector downtime

**Performance:**

- Detection time: 10 seconds (liveness probe interval)
- Recovery time: 36 seconds (total downtime)
- Data loss: 36 seconds of metrics (acceptable for non-critical)
- Alert threshold: >60 seconds downtime

```mermaid
sequenceDiagram
    participant Endpoints as Monitored Endpoints<br/>(1000 endpoints)
    participant Collector as Prometheus Collector<br/>(Pod: collector-1)
    participant K8s as Kubernetes<br/>(Control Plane)
    participant NewCollector as New Collector<br/>(Pod: collector-1 restarted)
    participant ServiceDiscovery as Service Discovery<br/>(Kubernetes API)
    participant Kafka
    participant Monitor as Monitoring System<br/>(Self-monitoring)
    Note over Collector: Normal operation<br/>Scraping 1000 endpoints<br/>1M metrics/sec
    Collector ->> Collector: CRASH (0ms)<br/>Reason: Out of Memory
    Note over Collector: Process dies<br/>Pod status: Running → CrashLoopBackOff
    Note over K8s: Liveness probe checks every 10 seconds
    K8s ->> Collector: Liveness probe (10000ms)<br/>HTTP GET /healthz
    Collector --x K8s: No response (timeout)
    K8s ->> K8s: Detect unhealthy (10000ms)
    Note over K8s: Pod status: Unhealthy<br/>Restart policy: Always<br/>Action: Restart pod
    K8s ->> Collector: Kill pod (12000ms)
    K8s ->> NewCollector: Start new pod (30000ms)
    Note over NewCollector: Initialize:<br/>- Load config<br/>- Connect to Kafka<br/>- Start HTTP server
    NewCollector ->> ServiceDiscovery: Fetch endpoint list (5000ms)
    Note over NewCollector: Query:<br/>GET /api/v1/pods?label=app=backend
    ServiceDiscovery -->> NewCollector: Endpoint list (35000ms)<br/>1000 endpoints
    NewCollector ->> NewCollector: Schedule scrapes (1000ms)
    Note over NewCollector: Schedule:<br/>- 1000 endpoints<br/>- 1 per second each<br/>- Staggered start times
    NewCollector ->> Endpoints: Resume scraping (36000ms)
    Endpoints -->> NewCollector: Metrics
    NewCollector ->> Kafka: Write metrics (36000ms)
    Kafka -->> NewCollector: Ack
    Note over Collector, NewCollector: Gap: 36 seconds<br/>(10s detection + 26s restart)
    Monitor ->> Monitor: Detect gap (40000ms)
    Note over Monitor: Alert: CollectorDowntime<br/>Condition: no metrics from collector-1<br/>for >60 seconds<br/>Status: Not triggered (gap = 36s < 60s threshold)
    Note over Endpoints, Monitor: Recovery complete (36 seconds)<br/>Data loss: 36 seconds of metrics<br/>Acceptable for non-critical metrics
```