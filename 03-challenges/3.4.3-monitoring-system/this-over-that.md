# Distributed Monitoring System - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made in the distributed monitoring system
design.

---

## Table of Contents

1. [M3DB Over InfluxDB and Cassandra for Time-Series Storage](#1-m3db-over-influxdb-and-cassandra-for-time-series-storage)
2. [Kafka Over Direct TSDB Writes for Buffering](#2-kafka-over-direct-tsdb-writes-for-buffering)
3. [Stream Processing (Flink) Over TSDB Queries for Alerting](#3-stream-processing-flink-over-tsdb-queries-for-alerting)
4. [Gorilla Compression Over Standard Compression](#4-gorilla-compression-over-standard-compression)
5. [Hybrid Collection (Pull + Push) Over Pull-Only](#5-hybrid-collection-pull--push-over-pull-only)
6. [Tiered Storage Over Single-Tier Storage](#6-tiered-storage-over-single-tier-storage)
7. [Redis Cache Over No Cache for Query Results](#7-redis-cache-over-no-cache-for-query-results)

---

## 1. M3DB Over InfluxDB and Cassandra for Time-Series Storage

### The Problem

The monitoring system needs to store 100M data points/sec with:

- High write throughput (100M writes/sec sustained, 500M peak)
- Fast time-range queries (<1 second for dashboard queries)
- Long-term retention (2+ years with 10 PB compressed storage)
- Efficient compression (10:1 ratio)
- Horizontal scalability (add nodes linearly)

### Options Considered

| Aspect                    | M3DB (Uber)                          | InfluxDB (Specialized TSDB)          | Cassandra + Custom TSDB Layer     |
|---------------------------|--------------------------------------|--------------------------------------|-----------------------------------|
| **Write Throughput**      | Billions writes/sec (proven at Uber) | 1M writes/sec per node (limited)     | 100M writes/sec (with tuning)     |
| **Horizontal Scaling**    | Excellent (consistent hashing)       | Limited (Enterprise only)            | Excellent (proven at Netflix)     |
| **Compression**           | Gorilla (10:1 ratio)                 | TSM engine (7:1 ratio)               | Custom (requires implementation)  |
| **Query Performance**     | Fast (optimized for time-range)      | Fast (purpose-built)                 | Medium (requires custom indexing) |
| **Prometheus Compatible** | Yes (PromQL support)                 | No (InfluxQL/Flux)                   | No (custom query language)        |
| **Ops Complexity**        | Medium (complex setup)               | Low (simple deployment)              | High (custom code + Cassandra)    |
| **Cost**                  | Medium (open source)                 | High (InfluxDB Enterprise expensive) | Medium (Cassandra + dev cost)     |
| **Community**             | Growing (Uber, others)               | Large (mature product)               | Large (Cassandra ecosystem)       |

### Decision Made

**Use M3DB as the primary time-series database for the distributed monitoring system.**

### Rationale

1. **Extreme Write Throughput**
    - M3DB is designed for scale (Uber ingests 10M+ metrics/sec)
    - Our requirement: 100M writes/sec sustained, 500M peak
    - InfluxDB: Limited to ~1M writes/sec per node (would need 100+ nodes, expensive)
    - Cassandra: Could handle it but requires custom TSDB layer (high dev cost)

2. **Horizontal Scalability**
    - M3DB uses consistent hashing (4096 virtual shards)
    - Add nodes → automatic resharding → linear scaling
    - InfluxDB: Horizontal scaling only in Enterprise (expensive)
    - Cassandra: Excellent scaling but needs custom TSDB logic

3. **Gorilla Compression (10:1 Ratio)**
    - M3DB uses Facebook's Gorilla algorithm (10:1 compression)
    - Our storage: 100 PB raw → 10 PB compressed
    - InfluxDB TSM: ~7:1 ratio (would need 14 PB, 40% more expensive)
    - Cassandra: Requires custom compression implementation

4. **Prometheus Compatibility**
    - M3DB supports PromQL natively (easy migration from Prometheus)
    - Existing Prometheus users can switch easily
    - InfluxDB: Requires rewriting queries (Influx

QL/Flux)

- Cassandra: Custom query language (steep learning curve)

5. **Proven at Scale**
    - Uber built M3DB for their monitoring (10M+ metrics/sec)
    - Battle-tested in production at extreme scale
    - InfluxDB: Not proven at 100M writes/sec
    - Cassandra: Proven but requires custom TSDB layer

### Implementation Details

**M3DB Cluster Configuration:**

- **Nodes:** 100 M3DB nodes (1 TB SSD + 100 GB memory each)
- **Shards:** 4096 virtual shards (consistent hashing)
- **Replication Factor:** 3x (durability, survives 2 node failures)
- **Write throughput:** 1M writes/sec per node × 100 nodes = 100M writes/sec
- **Storage:** 10 PB compressed (100 PB raw with 10:1 Gorilla compression)

**M3DB Components:**

- **M3DB Nodes** - Distributed storage with embedded Gorilla compression
- **M3Coordinator** - Query aggregator and shard router
- **M3Aggregator** - Real-time aggregation for rollups
- **M3Query** - PromQL-compatible query engine

**Example Configuration:**

```yaml
m3db:
  replication_factor: 3
  num_shards: 4096
  compression: gorilla
  retention:
    raw: 24h
    rollup_1min: 168h   # 7 days
    rollup_1hr: 8760h   # 1 year
    rollup_1day: 17520h # 2 years
```

### Trade-offs Accepted

| What We Gain                             | What We Sacrifice                          |
|------------------------------------------|--------------------------------------------|
| ✅ Extreme write throughput (100M/sec)    | ❌ Complex setup (multiple components)      |
| ✅ Horizontal scalability (linear)        | ❌ Smaller community (vs InfluxDB)          |
| ✅ Best-in-class compression (10:1)       | ❌ Relatively new (less mature than Influx) |
| ✅ Prometheus compatible (easy migration) | ❌ Higher ops complexity (vs InfluxDB)      |
| ✅ Proven at scale (Uber production)      | ❌ Requires custom tuning for optimal perf  |

### When to Reconsider

- **If scale is <10M writes/sec** → Use InfluxDB (simpler, lower ops overhead)
- **If budget is very limited** → Use Cassandra + custom TSDB (open source, no licensing)
- **If team has no Go expertise** → Use InfluxDB (easier to customize, written in Go vs M3DB also Go but more complex)
- **If Prometheus compatibility not needed** → Consider InfluxDB or VictoriaMetrics

---

## 2. Kafka Over Direct TSDB Writes for Buffering

### The Problem

Collectors produce 100M metrics/sec, and TSDB (M3DB) needs to persist them. During traffic spikes (e.g., 500M
metrics/sec during Black Friday), TSDB could be overwhelmed, causing:

- Write failures (429 Too Many Requests)
- Increased write latency (>1 second)
- Potential data loss
- Cascading failures (TSDB crashes, collectors retry, amplifying the problem)

### Options Considered

| Aspect                 | Kafka Buffer                      | Direct TSDB Writes            | RabbitMQ Buffer           | Redis Streams Buffer       |
|------------------------|-----------------------------------|-------------------------------|---------------------------|----------------------------|
| **Decoupling**         | Excellent (collectors ↔ TSDB)     | None (tightly coupled)        | Good                      | Good                       |
| **Durability**         | Excellent (disk + 3x replication) | N/A (write fails = data loss) | Good (persistent queues)  | Medium (Redis persistence) |
| **Replay Capability**  | Yes (retain 24h, backfill)        | No (can't reprocess)          | Limited (max queue size)  | Limited (capped stream)    |
| **Throughput**         | Excellent (100M+ msgs/sec)        | Depends on TSDB (limited)     | Medium (1M msgs/sec)      | High (10M+ msgs/sec)       |
| **Latency**            | <10ms (p99)                       | <50ms (best case)             | <20ms                     | <5ms                       |
| **Multiple Consumers** | Yes (TSDB, alerting, rollups)     | No (single path)              | Yes (fan-out)             | Yes (consumer groups)      |
| **Ops Complexity**     | Medium (Kafka cluster)            | Low (no buffer)               | Medium (RabbitMQ cluster) | Low (Redis cluster)        |
| **Cost**               | Medium (Kafka brokers)            | Low (no extra infra)          | Medium (RabbitMQ nodes)   | Low (Redis memory)         |

### Decision Made

**Use Kafka as the buffering layer between collectors and TSDB.**

### Rationale

1. **Decouples Collectors from TSDB**
    - If TSDB is slow (e.g., high query load), Kafka absorbs writes
    - Collectors don't block waiting for TSDB
    - TSDB processes at its own pace (not overwhelmed)
    - Prevents cascading failures

2. **Durability (No Data Loss)**
    - Kafka persists messages to disk before ACK
    - 3x replication (survives 2 broker failures)
    - Direct writes: If TSDB rejects, data is lost
    - Retry logic: Collectors can retry safely

3. **Replay Capability (Backfill)**
    - Kafka retains 24 hours of data
    - Can replay for rollup recomputation (e.g., fix bug in aggregation logic)
    - Can backfill if TSDB was down for hours
    - Direct writes: Can't reprocess historical data

4. **Multiple Consumers**
    - **TSDB writers** - Write to M3DB
    - **Alerting engine** - Real-time alert evaluation
    - **Rollup aggregators** - Downsampling for long-term storage
    - Direct writes: Only one consumer (TSDB)

5. **Proven Throughput**
    - Kafka handles 100M+ msgs/sec (24 partitions × 4.2M msgs/sec each)
    - Well within Kafka's capabilities
    - RabbitMQ: ~1M msgs/sec (would need 100 nodes, expensive)
    - Redis Streams: Good but limited memory (expensive for 24h retention)

### Implementation Details

**Kafka Configuration:**

```yaml
topic: metrics.raw
partitions: 24  # For 100M msgs/sec
replication_factor: 3  # Durability
retention.ms: 86400000  # 24 hours
compression.type: snappy  # 5x compression
min.insync.replicas: 2  # Require 2 ACKs before success
```

**Partitioning Strategy:**

- Partition by `hash(metric_name + labels)`
- Ensures all data points for a time series go to same partition (ordering preserved)
- Example: `cpu_usage{host="server-1"}` always goes to partition 7

**Consumer Groups:**

1. **tsdb-writers** (24 consumers) - Write to M3DB
2. **alerting-engine** (12 consumers) - Real-time alerts
3. **rollup-aggregators** (12 consumers) - Downsampling

**Backpressure Handling:**

- If TSDB slow → Kafka consumer lag increases
- Monitor lag: Alert if lag > 1M messages
- Auto-scale TSDB writers: Add more consumers if lag high

**Performance:**

```
Without Kafka:
- Collector → TSDB direct: 50ms (p99) in normal traffic
- Collector → TSDB direct: 1000ms+ (p99) in spike traffic
- Risk: Data loss during spikes

With Kafka:
- Collector → Kafka: 10ms (p99) always (non-blocking)
- Kafka → TSDB: 50ms (p99) (TSDB processes at own pace)
- Risk: No data loss (Kafka absorbs spikes)
```

### Trade-offs Accepted

| What We Gain                          | What We Sacrifice                    |
|---------------------------------------|--------------------------------------|
| ✅ Decoupling (collectors ↔ TSDB)      | ❌ Additional infrastructure (Kafka)  |
| ✅ No data loss (durability)           | ❌ Additional latency (~10ms)         |
| ✅ Replay capability (backfill)        | ❌ Operational complexity (Kafka ops) |
| ✅ Multiple consumers (alerting, etc.) | ❌ Storage cost (24h retention)       |
| ✅ Handles 5x traffic spikes           | ❌ Kafka becomes critical dependency  |

### When to Reconsider

- **If scale is <1M writes/sec** → Direct TSDB writes (simpler, no Kafka overhead)
- **If budget is very limited** → Redis Streams buffer (cheaper than Kafka)
- **If team has no Kafka expertise** → Use RabbitMQ (simpler to operate)
- **If replay not needed** → Use Redis Streams (no long retention needed)

---

## 3. Stream Processing (Flink) Over TSDB Queries for Alerting

### The Problem

The alerting system needs to evaluate 10,000 alert rules in real-time (<5 seconds) on 100M metrics/sec. Requirements:

- Real-time evaluation (not batch)
- Stateful queries (e.g., "CPU > 80% for 5 minutes")
- Complex aggregations (avg, p95, p99)
- No duplicate alerts (deduplication)

### Options Considered

| Aspect                  | Stream Processing (Flink)     | TSDB Queries (Poll-Based)        | Rule Engine (Prometheus Alertmanager) |
|-------------------------|-------------------------------|----------------------------------|---------------------------------------|
| **Latency**             | <5 seconds (real-time)        | 10-60 seconds (polling interval) | <10 seconds                           |
| **TSDB Load**           | Zero (consumes Kafka)         | High (10k queries every 10s)     | Medium (PromQL queries)               |
| **Stateful Processing** | Yes (RocksDB state store)     | No (stateless polling)           | Yes (Alertmanager state)              |
| **Complex Queries**     | Yes (windowed aggregations)   | Limited (PromQL complexity)      | Yes (PromQL)                          |
| **Deduplication**       | Built-in (state management)   | Manual (external store)          | Built-in                              |
| **Scalability**         | Excellent (Flink parallelism) | Limited (TSDB query capacity)    | Medium (Alertmanager clustering)      |
| **Ops Complexity**      | High (Flink cluster)          | Low (cron + scripts)             | Low (Prometheus + Alertmanager)       |
| **Cost**                | Medium (Flink cluster)        | High (TSDB query load)           | Low (lightweight)                     |

### Decision Made

**Use Apache Flink (or Kafka Streams) for real-time alert evaluation on Kafka stream.**

### Rationale

1. **Real-Time (<5 Seconds)**
    - Flink consumes Kafka `metrics.raw` stream in real-time
    - No need to wait for TSDB write + query cycle
    - Alerting latency: <5 seconds (vs 10-60 seconds with polling)

2. **Zero TSDB Load**
    - TSDB queries: 10k alert rules × 6 queries/min = 1M queries/min
    - This would overload TSDB (competes with dashboard queries)
    - Flink: Consumes Kafka directly (no TSDB queries)

3. **Stateful Processing (Complex Queries)**
    - Flink maintains state in RocksDB (e.g., "CPU > 80% for 5 minutes")
    - Can evaluate windowed aggregations (avg, p95, p99)
    - Polling: Requires external state store (complex)

4. **Deduplication Built-In**
    - Flink state store tracks alert status (firing, resolved)
    - Don't send duplicate alerts if already firing
    - Polling: Requires manual deduplication logic

5. **Horizontal Scalability**
    - Flink parallelism: Scale to 100+ task managers
    - Can handle 10k+ alert rules concurrently
    - Polling: Limited by TSDB query capacity (bottleneck)

### Implementation Details

**Flink Job:**

```java
// Flink alerting job
DataStream<Metric> metrics = env.addSource(new KafkaSource("metrics.raw"));

// Group by alert rule
KeyedStream<Metric, String> byRule = metrics
        .keyBy(metric -> metric.getAlertRule());

// 5-minute tumbling window
WindowedStream<Metric, String, TimeWindow> windowed = byRule
        .window(TumblingEventTimeWindows.of(Time.minutes(5)));

// Evaluate condition
DataStream<Alert> alerts = windowed
        .aggregate(new AvgAggregator())
        .filter(agg -> agg.getValue() > agg.getThreshold())
        .map(agg -> new Alert(agg.getRuleName(), agg.getValue()));

// Deduplicate (stateful)
alerts
        .

keyBy(Alert::getRuleName)
    .

process(new DeduplicateFunction())  // Check if already firing
        .

addSink(new KafkaSink("alerts.triggered"));
```

**State Management:**

- **State store:** RocksDB (embedded key-value store)
- **State:** Current alert status (firing, resolved), window aggregations
- **Checkpointing:** Every 60 seconds (fault tolerance)
- **Exactly-once semantics:** No duplicate alert notifications

**Alert Rule Example:**

```yaml
name: HighCPUUsage
condition: avg(cpu_usage{service="api-gateway"}) > 80 for 5 minutes
severity: warning
notification: slack-channel=#alerts
```

**Performance:**

```
With Flink:
- Alert latency: <5 seconds (real-time)
- TSDB query load: 0 (no queries)
- Scalability: 10k+ rules (horizontal scaling)
- State size: <10 GB per task manager

With TSDB Polling:
- Alert latency: 10-60 seconds (polling interval)
- TSDB query load: 1M queries/min (overload)
- Scalability: Limited (TSDB capacity)
- Requires external state store
```

### Trade-offs Accepted

| What We Gain                         | What We Sacrifice                    |
|--------------------------------------|--------------------------------------|
| ✅ Real-time alerts (<5 seconds)      | ❌ Complex setup (Flink cluster)      |
| ✅ Zero TSDB load (no queries)        | ❌ Operational complexity (Flink ops) |
| ✅ Stateful processing (windows)      | ❌ Higher infrastructure cost (Flink) |
| ✅ Built-in deduplication             | ❌ Requires Flink expertise           |
| ✅ Horizontal scalability (10k rules) | ❌ State management overhead          |

### When to Reconsider

- **If scale is <100 alert rules** → Use TSDB polling (simpler, Prometheus Alertmanager)
- **If team has no Flink expertise** → Use Prometheus Alertmanager (simpler)
- **If budget is limited** → Use Prometheus Alertmanager (lightweight)
- **If latency requirement is >1 minute** → Use TSDB polling (acceptable for non-critical)

---

## 4. Gorilla Compression Over Standard Compression

### The Problem

Storing 100 PB of raw time-series data for 2 years is prohibitively expensive:

- Storage cost: 100 PB × $0.10/GB-month (SSD) = $10M/month
- Need 10:1 compression to reduce to 10 PB = $1M/month
- Standard compression (gzip, snappy) only achieves 2-3x for time-series data

### Options Considered

| Aspect                  | Gorilla Compression (Facebook) | Standard Compression (gzip/snappy)    | No Compression             |
|-------------------------|--------------------------------|---------------------------------------|----------------------------|
| **Compression Ratio**   | 10:1 (proven at Facebook)      | 2-3:1 (not optimized for time-series) | 1:1 (raw storage)          |
| **Storage Cost**        | $1M/month (10 PB)              | $3.3M/month (33 PB)                   | $10M/month (100 PB)        |
| **Decompression Speed** | Fast (<1ms per block)          | Slow (CPU-intensive)                  | Instant (no decompression) |
| **CPU Overhead**        | Low (<5% CPU)                  | High (10-20% CPU)                     | None                       |
| **Write Amplification** | Low (compress in batches)      | High (compress every write)           | None                       |
| **Query Performance**   | Fast (decompress on-demand)    | Slow (decompress entire block)        | Fastest (raw access)       |
| **Implementation**      | M3DB built-in                  | Requires custom integration           | None                       |

### Decision Made

**Use Gorilla compression (Facebook's algorithm) as implemented in M3DB.**

### Rationale

1. **10:1 Compression Ratio**
    - Gorilla achieves 10:1 compression for time-series data (proven at Facebook, Uber)
    - Our storage: 100 PB raw → 10 PB compressed
    - Standard gzip: Only 2-3:1 (would need 33-50 PB, 3-5x more expensive)

2. **Cost Savings (90% Reduction)**
    - Without compression: $10M/month
    - With Gorilla: $1M/month
    - Savings: $9M/month ($108M/year)
    - ROI: Massive (compression is essentially free in comparison)

3. **Fast Decompression**
    - Gorilla decompression: <1ms per block
    - No noticeable impact on query latency
    - Standard gzip: 10-100ms decompression (would slow queries)

4. **Low CPU Overhead (<5%)**
    - Gorilla compression: <5% CPU overhead
    - Acceptable trade-off for 10x storage savings
    - Standard gzip: 10-20% CPU (significant overhead)

5. **Optimized for Time-Series**
    - **Timestamp compression:** Delta-of-delta encoding (1-2 bits per timestamp)
    - **Value compression:** XOR encoding for floats (1.37 bits per value average)
    - Standard compression: Not specialized for time-series patterns

### Implementation Details

**Gorilla Algorithm:**

**1. Timestamp Compression (Delta-of-Delta Encoding)**

```
Raw timestamps: [1000, 2000, 3000, 4000] (256 bits total)
Deltas: [-, 1000, 1000, 1000]
Delta-of-deltas: [-, -, 0, 0]
Compressed: 64 + 12 + 1 + 1 = 78 bits (3.3x compression)
```

**2. Value Compression (XOR Encoding)**

```
Raw values: [65.2, 67.5, 70.1, 72.3] (256 bits total as doubles)
XOR with previous:
- 67.5 XOR 65.2 = 8 significant bits
- 70.1 XOR 67.5 = 12 significant bits
- 72.3 XOR 70.1 = 10 significant bits
Compressed: 64 + 8 + 12 + 10 = 94 bits (2.7x compression)

Total: 78 + 94 = 172 bits (vs 512 bits raw = 3x compression for 4 points)
```

**Compression Improves with More Data:**

- 4 points: 3x compression
- 1-hour block (3600 points): 10x compression
- 1-day block (86,400 points): 12x compression

**Performance:**

```
Write path:
- Metrics buffered in memory (1-minute window)
- Batch compressed (1-minute block = 60 data points)
- Written to disk (compressed block)
- CPU overhead: <5%

Query path:
- Read compressed block from disk
- Decompress block (<1ms)
- Return data points
- Query latency: No noticeable impact (<1ms decompression)
```

### Trade-offs Accepted

| What We Gain                         | What We Sacrifice                       |
|--------------------------------------|-----------------------------------------|
| ✅ 10:1 compression (vs 1:1 raw)      | ❌ 5% CPU overhead (compression)         |
| ✅ $9M/month savings (90% reduction)  | ❌ Cannot reconstruct exact raw data     |
| ✅ Fast decompression (<1ms)          | ❌ Write latency +1ms (batching)         |
| ✅ Query performance (minimal impact) | ❌ Lossy for some data patterns          |
| ✅ Proven at scale (Facebook, Uber)   | ❌ Requires M3DB (not all TSDBs support) |

### When to Reconsider

- **If storage is not a concern** → No compression (fastest writes/queries)
- **If data patterns are not time-series** → Use standard compression (gzip)
- **If CPU is a constraint** → Use lighter compression (snappy, 2-3x ratio)
- **If using TSDB without Gorilla** → Use TSDB's native compression (e.g., InfluxDB TSM)

---

## 5. Hybrid Collection (Pull + Push) Over Pull-Only

### The Problem

The monitoring system needs to collect metrics from diverse endpoints:

- Internal services (Kubernetes pods, VMs) - accessible via HTTP
- Edge devices (IoT devices, mobile apps) - behind NAT/firewalls
- Cloud services (AWS Lambda, serverless) - ephemeral, no persistent endpoints
- External services (partner APIs, SaaS) - no HTTP access

Pull-only (Prometheus model) can't reach all endpoints.

### Options Considered

| Aspect                | Hybrid (Pull + Push)                      | Pull-Only (Prometheus)       | Push-Only (StatsD)           |
|-----------------------|-------------------------------------------|------------------------------|------------------------------|
| **Firewall/NAT**      | Push works behind NAT                     | Pull requires inbound access | Push works behind NAT        |
| **Service Discovery** | Centralized (pull) + self-register (push) | Centralized only             | Self-register only           |
| **Failure Detection** | Pull detects down endpoints               | Pull detects down            | Push requires heartbeat      |
| **Load Control**      | Pull: collector controls rate             | Collector controls rate      | Push: endpoints can overload |
| **Latency**           | Pull: 1 second minimum                    | 1 second minimum             | Push: <100ms (instant)       |
| **Scalability**       | Medium (pull bottleneck)                  | Medium (pull bottleneck)     | High (endpoints independent) |
| **Ops Complexity**    | Medium (two collection paths)             | Low (single path)            | Low (single path)            |

### Decision Made

**Use hybrid collection: Pull-based for internal services, Push-based for edge devices and ephemeral endpoints.**

### Rationale

1. **Firewall/NAT Support (Push)**
    - Edge devices (IoT, mobile) are behind NAT → can't be scraped
    - Push allows devices to initiate connection (outbound only)
    - Pull-only: Would require exposing inbound ports (security risk)

2. **Ephemeral Endpoints (Push)**
    - Serverless functions (AWS Lambda) are ephemeral (short-lived)
    - No persistent endpoint to scrape
    - Push allows function to emit metrics before termination

3. **Service Discovery (Pull)**
    - Internal services (Kubernetes pods) use service discovery
    - Collector automatically discovers new pods (no manual registration)
    - Push-only: Requires manual registration (error-prone)

4. **Failure Detection (Pull)**
    - Pull detects endpoint down (failed scrape)
    - Automatic alerting (endpoint unreachable)
    - Push-only: Requires heartbeat mechanism (complex)

5. **Best of Both Worlds**
    - Pull: Good for internal, well-known services (auto-discovery, failure detection)
    - Push: Good for edge, ephemeral, external services (NAT-friendly, instant)

### Implementation Details

**Pull-Based Collection (Internal Services):**

```yaml
# Prometheus collector config
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [ __meta_kubernetes_pod_annotation_prometheus_io_scrape ]
        regex: "true"
        action: keep
    scrape_interval: 1s
    scrape_timeout: 500ms
```

**Push-Based Collection (Edge Devices):**

```python
# Agent on edge device
import requests

def push_metrics(metrics):
    payload = {
        "metrics": metrics,
        "labels": {"device_id": "iot-device-123", "location": "warehouse-5"}
    }
    requests.post("https://collector.monitoring.com/metrics", json=payload)

# Emit metrics every 1 second
while True:
    metrics = collect_metrics()  # CPU, memory, temperature
    push_metrics(metrics)
    time.sleep(1)
```

**Collection Statistics:**

```
Pull-based (70% of endpoints):
- Internal services: 700k Kubernetes pods
- VMs: 300k virtual machines
- Total: 1M endpoints (70M metrics/sec)
- Benefit: Auto-discovery, failure detection

Push-based (30% of endpoints):
- Edge devices: 200k IoT devices
- Serverless: 100k Lambda functions (ephemeral)
- External: 50k partner APIs
- Total: 350k endpoints (30M metrics/sec)
- Benefit: NAT-friendly, instant metrics
```

### Trade-offs Accepted

| What We Gain                   | What We Sacrifice                   |
|--------------------------------|-------------------------------------|
| ✅ Supports all endpoint types  | ❌ Two collection paths (complexity) |
| ✅ NAT/firewall friendly (push) | ❌ No auto-discovery for push        |
| ✅ Ephemeral endpoints (push)   | ❌ More client-side code (agents)    |
| ✅ Auto-discovery (pull)        | ❌ Can't scrape behind NAT (pull)    |
| ✅ Failure detection (pull)     | ❌ Requires heartbeat for push       |

### When to Reconsider

- **If all endpoints are internal (no NAT/firewall)** → Use pull-only (simpler)
- **If all endpoints are ephemeral/edge** → Use push-only (simpler)
- **If team has no Prometheus expertise** → Use push-only (easier for developers)
- **If service discovery is not needed** → Use push-only (self-registration)

---

## 6. Tiered Storage Over Single-Tier Storage

### The Problem

Storing 10 PB of compressed data for 2+ years is expensive:

- Single-tier (SSD): 10 PB × $0.10/GB-month = $1M/month
- Query patterns: 80% of queries access last 7 days (hot data), 20% access older data
- Most queries don't need SSD speed for old data

### Options Considered

| Aspect                     | Tiered Storage (Hot/Warm/Cold) | Single-Tier (SSD)            | Single-Tier (HDD)                   |
|----------------------------|--------------------------------|------------------------------|-------------------------------------|
| **Cost**                   | $244k/month (mixed tiers)      | $1M/month                    | $200k/month                         |
| **Query Latency (Recent)** | <10ms (hot SSD)                | <10ms                        | <50ms                               |
| **Query Latency (Old)**    | 1-5 seconds (cold S3)          | <10ms                        | <50ms                               |
| **Storage Capacity**       | Unlimited (S3)                 | Limited (expensive to scale) | Limited (slower for large datasets) |
| **Ops Complexity**         | High (manage 3 tiers)          | Low (single tier)            | Low (single tier)                   |
| **Data Lifecycle**         | Automatic (age-based)          | Manual (no auto-transition)  | Manual                              |

### Decision Made

**Use tiered storage: Hot (SSD) → Warm (HDD) → Cold (S3/Glacier) with automatic lifecycle transitions.**

### Rationale

1. **Cost Savings (75% Reduction)**
    - Single-tier SSD: $1M/month
    - Tiered storage: $244k/month
    - Savings: $756k/month ($9M/year)
    - ROI: Massive (4x cheaper)

2. **Query Performance (Hot Data)**
    - 80% of queries access last 7 days (hot tier)
    - Hot tier: SSD, <10ms latency
    - No impact on most queries (fast SSD for recent data)

3. **Unlimited Retention (Cold Tier)**
    - S3/Glacier: Unlimited storage (cheap: $0.004/GB-month)
    - Can keep data for 5+ years (compliance, historical analysis)
    - Single-tier SSD: Prohibitively expensive for long retention

4. **Automatic Lifecycle Management**
    - Age-based transitions: 7 days (hot → warm), 90 days (warm → cold)
    - No manual intervention (automated)
    - Single-tier: Manual archival (error-prone)

5. **Optimal for Query Patterns**
    - Hot tier (7 days): 80% of queries, <10ms latency
    - Warm tier (7-90 days): 15% of queries, <50ms latency
    - Cold tier (90+ days): 5% of queries, 1-5 seconds latency
    - Match storage tier to query frequency (cost-effective)

### Implementation Details

**Storage Tiers:**

**1. Hot Tier (0-7 days)**

- **Storage:** SSD (NVMe)
- **Speed:** Very fast (<10ms read latency)
- **Cost:** $0.10/GB-month
- **Size:** 800 TB (7 days × 114 TB/day)
- **Cost:** $80k/month

**2. Warm Tier (7-90 days)**

- **Storage:** HDD (SATA)
- **Speed:** Fast (<50ms read latency)
- **Cost:** $0.02/GB-month
- **Size:** 8 PB (83 days × 114 TB/day)
- **Cost:** $160k/month

**3. Cold Tier (90+ days)**

- **Storage:** S3/Glacier
- **Speed:** Slow (1-5 seconds retrieval)
- **Cost:** $0.004/GB-month
- **Size:** 1 PB (730 days - 90 days = 640 days × 1.5 TB/day after aggressive rollups)
- **Cost:** $4k/month

**Total Cost:** $80k + $160k + $4k = $244k/month (vs $1M/month single-tier SSD)

**Lifecycle Policy:**

```yaml
lifecycle:
  - name: transition_to_warm
    age: 7 days
    action: move_to_tier
    target: warm (HDD)

  - name: transition_to_cold
    age: 90 days
    action: move_to_tier
    target: cold (S3)

  - name: delete_old_data
    age: 730 days
    action: delete
```

**Query Routing (Automatic):**

```python
def query_metrics(metric_name, time_range):
    if time_range.end <= now() - 7 days:
        # Recent data: hot tier (SSD)
        return query_hot_tier(metric_name, time_range)
    elif time_range.end <= now() - 90 days:
        # Medium-old data: warm tier (HDD)
        return query_warm_tier(metric_name, time_range)
    else:
        # Old data: cold tier (S3)
        return query_cold_tier(metric_name, time_range)
```

### Trade-offs Accepted

| What We Gain                        | What We Sacrifice                       |
|-------------------------------------|-----------------------------------------|
| ✅ 75% cost savings ($756k/month)    | ❌ Complex setup (3 storage tiers)       |
| ✅ Fast recent queries (<10ms)       | ❌ Slow old queries (1-5 seconds)        |
| ✅ Unlimited retention (S3)          | ❌ Lifecycle management overhead         |
| ✅ Automatic transitions (age-based) | ❌ Can't easily reconstruct cold data    |
| ✅ Optimal for query patterns        | ❌ Query latency varies (tier-dependent) |

### When to Reconsider

- **If all queries need <10ms latency** → Use single-tier SSD (expensive)
- **If retention is <30 days** → Use single-tier SSD (simpler)
- **If budget is not a concern** → Use single-tier SSD (fastest, simplest)
- **If ops team is small** → Use single-tier HDD (simpler, cheaper than tiered)

---

## 7. Redis Cache Over No Cache for Query Results

### The Problem

Dashboard queries can be expensive:

- Dashboard with 10 panels × 10 queries = 100 queries
- Each query: 50-500ms (depending on time range)
- Total dashboard load: 5-50 seconds (unacceptable)
- 100k dashboards × 10 queries/min = 16k QPS (high TSDB load)

Caching can reduce latency and TSDB load.

### Options Considered

| Aspect                  | Redis Cache               | No Cache                | In-Memory Cache (App-Level)        |
|-------------------------|---------------------------|-------------------------|------------------------------------|
| **Query Latency (Hit)** | <10ms (Redis)             | 50-500ms (TSDB query)   | <1ms (in-memory)                   |
| **Hit Rate**            | 80% (popular dashboards)  | 0% (no cache)           | 60% (per-instance cache)           |
| **TSDB Load**           | 20% (only cache misses)   | 100% (all queries)      | 40% (partial cache)                |
| **Consistency**         | Eventual (TTL-based)      | Strong (always fresh)   | Eventual (TTL-based)               |
| **Scalability**         | Excellent (Redis cluster) | Limited (TSDB capacity) | Poor (no sharing across instances) |
| **Ops Complexity**      | Medium (Redis cluster)    | Low (no cache)          | Low (built-in)                     |
| **Cost**                | Medium (Redis memory)     | High (more TSDB nodes)  | Low (app memory)                   |

### Decision Made

**Use Redis cluster as a distributed cache for query results.**

### Rationale

1. **80% Cache Hit Rate (Popular Dashboards)**
    - Popular dashboards (e.g., "Global QPS", "Error Rates") accessed frequently
    - 80% of queries hit cache (no TSDB query needed)
    - 20% of queries miss cache (fetch from TSDB)

2. **10x Faster (Cache Hit)**
    - Redis cache hit: <10ms
    - TSDB query (cache miss): 50-500ms
    - Dashboard load time: 60ms (cached) vs 5-50 seconds (no cache)

3. **80% TSDB Load Reduction**
    - Without cache: 16k QPS → TSDB
    - With cache: 3.2k QPS → TSDB (16k × 20% miss rate)
    - Can handle 5x more users with same TSDB capacity

4. **Distributed Cache (Shared Across Instances)**
    - Redis cluster: All query service instances share cache
    - In-memory cache: Each instance has separate cache (no sharing, lower hit rate)

5. **TTL-Based Invalidation (Acceptable)**
    - Recent data: TTL = 1 minute (acceptable staleness)
    - Old data: TTL = 1 hour (acceptable for historical queries)
    - Trade-off: Eventual consistency (acceptable for dashboards)

### Implementation Details

**Redis Cache Configuration:**

```yaml
redis:
  cluster_mode: true
  nodes: 3
  memory_per_node: 100 GB
  total_memory: 300 GB
  eviction_policy: allkeys-lru  # Evict least recently used
  maxmemory_policy: allkeys-lru
```

**Cache Key Format:**

```python
def generate_cache_key(query, time_range):
    # Hash query + time range
    key = f"query:{hash(query)}:range:{time_range.start}:{time_range.end}"
    return key

# Example: "query:a3f5b2c8:range:1609459200:1609545600"
```

**Cache TTL Strategy:**

```python
def get_ttl(time_range):
    age = now() - time_range.end
    if age < 1 hour:
        return 60 seconds  # Recent data changes frequently
    elif age < 1 day:
        return 300 seconds  # 5 minutes
    else:
        return 3600 seconds  # 1 hour (old data rarely changes)
```

**Query Flow with Cache:**

```python
def query_metrics(query, time_range):
    # Generate cache key
    cache_key = generate_cache_key(query, time_range)
    
    # Check cache
    cached_result = redis.get(cache_key)
    if cached_result:
        # Cache HIT
        return cached_result  # <10ms
    
    # Cache MISS
    result = tsdb.query(query, time_range)  # 50-500ms
    
    # Store in cache
    ttl = get_ttl(time_range)
    redis.setex(cache_key, ttl, result)
    
    return result
```

**Performance:**

```
Without Cache:
- Dashboard load: 5-50 seconds (100 queries × 50-500ms each)
- TSDB load: 16k QPS (all queries)

With Redis Cache (80% hit rate):
- Dashboard load: 60ms (cache hits: 80 × 10ms + cache misses: 20 × 500ms)
- TSDB load: 3.2k QPS (only 20% miss rate)
- 10x faster, 5x lower TSDB load
```

### Trade-offs Accepted

| What We Gain                 | What We Sacrifice                    |
|------------------------------|--------------------------------------|
| ✅ 10x faster dashboard load  | ❌ Eventual consistency (TTL-based)   |
| ✅ 80% TSDB load reduction    | ❌ Additional infrastructure (Redis)  |
| ✅ 80% cache hit rate         | ❌ Cache invalidation complexity      |
| ✅ Distributed cache (shared) | ❌ Memory cost (300 GB Redis cluster) |
| ✅ Can handle 5x more users   | ❌ Requires cache warming strategy    |

### When to Reconsider

- **If strong consistency is required** → No cache (always query TSDB)
- **If query patterns are very diverse (low hit rate)** → No cache (not effective)
- **If budget is limited** → Use in-memory cache (cheaper, but lower hit rate)
- **If TSDB capacity is not a concern** → No cache (simpler)

---

## Summary Table: All Design Decisions

| Decision               | Chosen Approach           | Alternative               | Key Reason                                   |
|------------------------|---------------------------|---------------------------|----------------------------------------------|
| **1. TSDB**            | M3DB                      | InfluxDB                  | Extreme write throughput (100M/sec)          |
| **2. Buffering**       | Kafka                     | Direct TSDB Writes        | Decoupling + durability + replay             |
| **3. Alerting**        | Stream Processing (Flink) | TSDB Queries (Poll-Based) | Real-time (<5s) + zero TSDB load             |
| **4. Compression**     | Gorilla                   | Standard gzip             | 10:1 ratio (vs 2-3:1) + $9M/month savings    |
| **5. Collection**      | Hybrid (Pull + Push)      | Pull-Only                 | Supports all endpoint types (NAT, ephemeral) |
| **6. Storage Tiering** | Hot/Warm/Cold             | Single-Tier SSD           | 75% cost savings ($756k/month)               |
| **7. Query Cache**     | Redis Cache               | No Cache                  | 10x faster + 80% TSDB load reduction         |
