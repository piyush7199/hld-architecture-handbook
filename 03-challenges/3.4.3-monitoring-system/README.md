# Design a Distributed Monitoring System

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

Design a distributed monitoring system that collects, stores, and analyzes metrics from millions of endpoints (servers,
containers, applications) in real-time. The system must handle **100M writes/sec**, store **10 PB** of compressed data,
and provide **real-time alerting** (<5 seconds) with **fast dashboard queries** (<1 second).

**Similar Systems:** Prometheus, Datadog, New Relic, Grafana Cloud, AWS CloudWatch, Uber's M3

---

## Requirements and Scale Estimation

### Functional Requirements

1. **Metrics Collection** - Pull and push-based collection from 1M endpoints
2. **Time-Series Storage** - Store metrics with 1-second granularity for 2+ years
3. **Real-Time Alerting** - Evaluate alert rules in <5 seconds with complex queries
4. **Visualization** - Dashboards with historical analysis and ad-hoc queries
5. **Auto-Discovery** - Automatically discover new endpoints (Kubernetes pods, cloud instances)

### Non-Functional Requirements

1. **High Write Throughput:** 100M writes/sec sustained, 500M writes/sec peak
2. **Low Write Latency:** <100ms p99 from collection to storage
3. **Fast Query Performance:** <1 second for dashboard queries (last 1 hour)
4. **High Availability:** 99.9% uptime for alerting and query services
5. **Horizontal Scalability:** Add nodes to scale writes and reads independently

### Scale Estimation

| Metric                   | Calculation                                      | Result                                 |
|--------------------------|--------------------------------------------------|----------------------------------------|
| **Endpoints**            | 1M servers/containers                            | 1 Million endpoints                    |
| **Metrics per Endpoint** | 100 metrics (CPU, memory, disk, network, custom) | 100 metrics per endpoint               |
| **Write Throughput**     | 1M endpoints × 100 metrics × 1 sample/sec        | **100M data points/sec**               |
| **Storage (1 Day)**      | 100M points/sec × 86,400 sec/day                 | 8.64 trillion points/day               |
| **Storage (2 Years)**    | 8.64 trillion × 730 days                         | **6.3 quadrillion points**             |
| **Raw Storage Size**     | 6.3 quadrillion × 16 bytes                       | ~100 PB (before compression)           |
| **Compressed Storage**   | 100 PB / 10 (compression ratio)                  | **~10 PB** (after Gorilla compression) |
| **Query Throughput**     | 100k dashboards × 10 queries/min / 60            | **~16k QPS** (read queries)            |

**Key Insight:** System is **write-heavy** (100M writes/sec vs 16k reads/sec). Must optimize for sequential writes and
time-based compression.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    MONITORED INFRASTRUCTURE                          │
│  [ Server 1 ] [ Server 2 ] [ Container ] [ App ] ... [ Server 1M ]  │
│       │            │            │           │              │         │
└───────┼────────────┼────────────┼───────────┼──────────────┼─────────┘
        │ scrape     │ scrape     │ push      │ push         │ scrape
        ▼            ▼            ▼           ▼              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      COLLECTION LAYER                                │
│  [ Collector 1 ] [ Collector 2 ] ... [ Collector N ]                │
│  (Prometheus/Agents - 100 instances, 1M metrics/sec each)            │
└───────┬──────────────┬────────────────────────┬──────────────────────┘
        │ write        │ write                  │ write
        ▼              ▼                        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       BUFFER LAYER                                   │
│                   [ Kafka Cluster ]                                  │
│                   100M msgs/sec, 24 partitions                       │
└───────┬──────────────┬────────────────────────┬──────────────────────┘
        │              │                        │
        ▼              ▼                        ▼
  ┌──────────┐  ┌────────────┐  ┌──────────────────────┐
  │ ALERTING │  │  TSDB      │  │ REAL-TIME AGGREGATION│
  │ ENGINE   │  │ (M3DB)     │  │ (Stream Processor)   │
  │ (Flink)  │  │            │  │                      │
  │          │  │ 10 PB      │  │ - Downsampling       │
  │ Rules:   │  │ Sharded by │  │ - Rollups            │
  │ CPU>80%  │  │ - Time     │  └──────────┬───────────┘
  │ for 5min │  │ - Metric   │             │ write rollups
  └────┬─────┘  └─────┬──────┘             │
       │ notify       │ read queries       ▼
       ▼              ▼           ┌────────────────┐
  ┌────────────┐  ┌──────────────────────┐        │
  │NOTIFICATION│  │   QUERY SERVICE       │◄───────┘
  │ SERVICE    │  │   (API + Cache)       │
  │ - Email    │  │   - Query optimizer   │
  │ - Slack    │  │   - Redis cache       │
  │ - PagerDuty│  │   - Result cache      │
  └────────────┘  └────────┬──────────────┘
                           │ HTTP API
                           ▼
                  ┌────────────────┐
                  │  GRAFANA       │
                  │  (Dashboards)  │
                  └────────────────┘
```

**Components:**

1. **Collectors (Prometheus/Agents)** - Scrape or receive metrics (pull/push)
2. **Kafka Buffer** - Decouples collectors from storage (handles bursts)
3. **TSDB (M3DB)** - Optimized time-series storage (10 PB compressed)
4. **Alerting Engine (Flink)** - Real-time rule evaluation on stream
5. **Real-Time Aggregation** - Downsampling and rollups (1min, 1hr, 1day)
6. **Query Service** - API layer with caching for fast dashboard queries
7. **Notification Service** - Sends alerts (email, Slack, PagerDuty)
8. **Grafana** - Visualization and dashboard UI

---

## Data Model

### Time-Series Data Structure

```
Metric: http_request_latency_ms
Timestamp: 1698765432000 (Unix epoch, milliseconds)
Value: 234.5 (numeric)
Labels: {
  service: "api-gateway",
  method: "POST",
  endpoint: "/users",
  region: "us-east-1",
  status_code: "200"
}
```

**Storage Format (Prometheus-style):**

```
http_request_latency_ms{service="api-gateway",method="POST",endpoint="/users",region="us-east-1",status_code="200"} 234.5 1698765432000
```

### TSDB Schema (Cassandra Example)

**Table 1: Raw Metrics (1-Second Resolution)**

```sql
CREATE TABLE metrics_raw (
    metric_name text,
    labels map<text, text>,
    timestamp timestamp,
    value double,
    PRIMARY KEY ((metric_name, labels), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

**Partitioning:** By `(metric_name, labels)` to group all data points for a specific time series. Cluster by
`timestamp DESC` for efficient time-range queries.

**Table 2: Rollup Metrics (Downsampled)**

```sql
CREATE TABLE metrics_rollup_1min (
    metric_name text,
    labels map<text, text>,
    timestamp_bucket timestamp,
    count bigint,
    sum double,
    min double,
    max double,
    avg double,
    p50 double,
    p95 double,
    p99 double,
    PRIMARY KEY ((metric_name, labels), timestamp_bucket)
);
```

**Rollup Hierarchy:**

- **Raw (1-second)** → Retained for 1 day
- **1-minute rollup** → Retained for 7 days
- **5-minute rollup** → Retained for 30 days
- **1-hour rollup** → Retained for 1 year
- **1-day rollup** → Retained for 2+ years

**Benefits:** Reduces storage from 100 PB to 10 PB. Speeds up long-range queries by 3600x (use 1-hour rollup instead of
raw).

---

## Detailed Component Design

### Metrics Collection

**Pull-Based (Prometheus Model):**

1. **Service Discovery** - Auto-discover endpoints (Kubernetes API, AWS API)
2. **Scraping** - HTTP GET to `/metrics` endpoint every 1 second
3. **Parsing** - Parse Prometheus text format
4. **Buffering** - Write to local buffer → Kafka

**Push-Based (Agent Model):**

1. **Agent Installation** - Deploy on each endpoint (Telegraf, Datadog agent)
2. **Collection** - Collect system metrics (CPU, memory, disk, network)
3. **Push** - HTTP POST to collector endpoint (batched)

**Comparison:**

| Aspect        | Pull-Based                            | Push-Based                    |
|---------------|---------------------------------------|-------------------------------|
| **Discovery** | Centralized (scraper knows endpoints) | Distributed (agents register) |
| **Firewall**  | Requires inbound access               | Requires outbound only        |
| **Failure**   | Scraper detects down                  | Endpoint retries              |
| **Scale**     | Horizontal scaling                    | Self-managing                 |

**Decision:** **Hybrid** - pull for internal services, push for edge devices.

### Kafka Buffer Layer

**Why Kafka?**

- **Decouples** collectors from storage (TSDB can lag, Kafka absorbs)
- **Replay** capability (reprocess for backfill)
- **Durability** (persisted to disk before ack)
- **Partitioning** (distributes load)

**Configuration:**

```
Topic: metrics.raw
Partitions: 24 (for 100M msgs/sec)
Replication: 3 (durability)
Retention: 24 hours (replay window)
Compression: snappy (5x reduction)
```

**Partitioning:** `hash(metric_name + labels)` to ensure ordering for each time series.

**Consumer Groups:**

1. **TSDB Writers** - Write to time-series database
2. **Alerting Engine** - Real-time alert evaluation
3. **Real-Time Aggregation** - Downsampling and rollups

### Time-Series Database (TSDB)

**TSDB Options:**

| Database      | Pros                                     | Cons                          |
|---------------|------------------------------------------|-------------------------------|
| **InfluxDB**  | Purpose-built, excellent compression     | Limited horizontal scaling    |
| **M3DB**      | Designed for scale (billions writes/sec) | Complex to operate            |
| **Cassandra** | Proven scalability                       | Not optimized for time-series |

**Decision:** **M3DB** for write-heavy workloads with Prometheus compatibility. *(
See [this-over-that.md](./this-over-that.md) for analysis.)*

**M3DB Architecture:**

- **M3DB Nodes** - Distributed storage (50+ nodes)
- **M3Coordinator** - Query aggregator
- **M3Aggregator** - Real-time aggregation
- **M3Query** - PromQL-compatible query engine

**Sharding:** Consistent hashing by `metric_name + labels` (4096 shards, 3x replication).

**Compression:** Gorilla compression (Facebook's algorithm) → 10:1 ratio (100 PB → 10 PB).

### Alerting Engine

**Requirements:**

- Evaluate rules in real-time (<5 seconds)
- Support complex queries (aggregations, percentiles, windows)
- Stateful queries ("CPU > 80% for 5 minutes")
- Deduplicate and group alerts

**Architecture:** Apache Flink or Kafka Streams

- **Input:** Kafka `metrics.raw`
- **Processing:** Stateful windowed aggregations
- **Output:** Kafka `alerts.triggered`

**Alert Rule Example:**

```
Alert: HighCPUUsage
Condition: avg(cpu_usage{service="api-gateway"}) > 80 for 5 minutes
Severity: warning
Notification: slack-channel=#alerts
```

**Processing Flow:**

1. Consume metrics from Kafka
2. Group by alert rule
3. Windowed aggregation (5-minute tumbling window)
4. Evaluate condition (avg > 80%)
5. Trigger alert if condition met for entire window
6. Deduplicate (don't resend if already firing)
7. Publish to `alerts.triggered`

**State Management:**

- **State store:** RocksDB (embedded key-value store)
- **State:** Current window aggregations
- **Checkpointing:** Periodic snapshots for fault tolerance

### Real-Time Aggregation (Rollups)

**Why Rollups?**

- Storing 6.3 quadrillion points at 1-second resolution is too expensive
- Dashboard queries over long ranges (e.g., "last 30 days") would be slow
- Downsampling reduces storage (10x) and improves query performance (3600x)

**Retention Policy:**

| Resolution      | Retention | Storage (per metric) |
|-----------------|-----------|----------------------|
| 1-second (raw)  | 1 day     | 8.64 trillion points |
| 1-minute rollup | 7 days    | 1.4 trillion points  |
| 5-minute rollup | 30 days   | 8.6 billion points   |
| 1-hour rollup   | 1 year    | 876 million points   |
| 1-day rollup    | 2+ years  | 730,000 points       |

**Rollup Computation:**

- **Stream processor:** Flink or Kafka Streams
- **Input:** Kafka `metrics.raw`
- **Processing:** Windowed aggregations (1-min, 5-min, 1-hour, 1-day)
- **Output:** Write to TSDB rollup tables

**Aggregation Functions:**

- `count`, `sum`, `min`, `max`, `avg`, `p50`, `p95`, `p99`

**Example:**

```
Raw (1-second):
cpu_usage{host="server-1"} 65.2 @ 12:00:00
cpu_usage{host="server-1"} 67.5 @ 12:00:01
...
cpu_usage{host="server-1"} 72.3 @ 12:00:59

1-minute rollup:
cpu_usage_1min{host="server-1"} {
  count: 60,
  avg: 68.0,
  min: 65.2,
  max: 75.1,
  p50: 67.8,
  p95: 74.2
} @ 12:00:00
```

### Query Service

**Requirements:**

- Fast queries (<1 second for dashboards)
- Complex queries (aggregations, joins, percentiles)
- Caching for frequently accessed data
- Query optimization

**Query Optimization:**

**1. Automatic Rollup Selection**

- Query: "Show CPU for last 30 days"
- Optimizer: Use 1-hour rollup (not raw)
- Benefit: 3600x fewer data points

**2. Predicate Pushdown**

- Push filters to TSDB (not app layer)
- Example: Filter `region=us-east-1` at storage

**3. Query Result Caching**

- Cache key: `hash(query + time_range)`
- TTL: 1 minute (recent), 1 hour (old data)
- Hit rate: 80% (popular dashboards)

**4. Query Parallelization**

- Split query into sub-queries (per shard)
- Execute in parallel
- Merge results

**Caching Strategy:**

| Cache Layer        | Technology | TTL               | Hit Rate |
|--------------------|------------|-------------------|----------|
| Query Result Cache | Redis      | 1 minute (recent) | 80%      |
| TSDB Block Cache   | In-Memory  | LRU eviction      | 60%      |

---

## Bottlenecks and Scaling

### Write Path Bottlenecks

**Bottleneck 1: Kafka Write Throughput**

- **Problem:** 100M writes/sec requires high throughput (200 MB/s per partition)
- **Solution:**
    - Increase partitions: 24 → 48
    - Batching: Collectors batch 1000 metrics
    - Compression: Snappy (5x reduction)
    - Async producers (non-blocking)
- **Result:** 100M writes/sec, 100 MB/s per partition (comfortable headroom)

**Bottleneck 2: TSDB Write Throughput**

- **Problem:** Random writes due to label combinations
- **Solution:**
    - Write buffer (M3DB aggregator): 1-minute memory buffer
    - LSM-Tree storage (write-optimized)
    - Gorilla compression (10x reduction)
    - Horizontal scaling: 100 nodes
- **Result:** 1M writes/sec per node × 100 nodes = 100M writes/sec

### Read Path Bottlenecks

**Bottleneck 3: Dashboard Query Latency**

- **Problem:** 30-day query fetches millions of points (slow)
- **Solution:**
    - Automatic rollup selection: 1-hour rollup (3600x fewer points)
    - Query result caching: Redis (80% hit rate)
    - Downsampling at query time
- **Result:** 30-day query: 720 points (1-hour rollup), 100ms (vs 2.6B points, 10 seconds)

**Bottleneck 4: Ad-Hoc Query Overload**

- **Problem:** Expensive queries overwhelm TSDB
- **Solution:**
    - Query rate limiting: Max 10 concurrent/user
    - Query timeout: Kill queries >30 seconds
    - Query cost estimation: Warn before expensive queries
    - Dedicated query pool: Separate TSDB replicas
- **Result:** Dashboard queries unaffected by ad-hoc queries

### Storage Bottlenecks

**Bottleneck 5: Storage Capacity**

- **Problem:** 10 PB storage is expensive
- **Solution:**
    - Tiered storage:
        - **Hot (0-7 days):** SSD, fast queries
        - **Warm (7-90 days):** HDD, slower queries
        - **Cold (90+ days):** S3/Glacier, archive
    - Aggressive rollups: 1-day rollups only for >1 year data
    - Selective retention: Critical metrics longer, non-critical deleted
- **Cost:** $244k/month ($80k hot + $160k warm + $4k cold)

---

## Multi-Region Deployment

**Requirements:**

- Low-latency collection (collect in region where endpoint is)
- Global view of metrics (aggregate across regions)
- Region failover

**Architecture:**

**Regional Deployment:**

- Each region: Full stack (Collectors → Kafka → TSDB → Query)
- Metrics collected locally (low latency)
- Regional dashboards for local troubleshooting

**Global Aggregation:**

- **Cross-region replication:** Kafka MirrorMaker → central region
- **Global TSDB:** Aggregates metrics from all regions
- **Global dashboards:** View across all regions

**Trade-offs:**

- **Eventual consistency:** Global metrics lag 5-30 seconds
- **Higher cost:** 3x infrastructure (3 regions)
- **Complexity:** Cross-region networking

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Using RDBMS for Time-Series Data

**Problem:** PostgreSQL/MySQL for metrics
**Why Bad:** Not optimized for high writes, slow time-range queries, no compression
**Solution:** ✅ Use TSDB (InfluxDB, M3DB, TimescaleDB)

### ❌ Anti-Pattern 2: No Rollups/Downsampling

**Problem:** Store all metrics at 1-second resolution forever
**Why Bad:** 100 PB storage, slow long-range queries
**Solution:** ✅ Implement rollups (1-min, 1-hour, 1-day) with retention policies

### ❌ Anti-Pattern 3: Synchronous Alert Evaluation on TSDB

**Problem:** Query TSDB every 10 seconds to evaluate alerts
**Why Bad:** High query load, 10+ second latency, competes with dashboards
**Solution:** ✅ Stream processing (Flink/Kafka Streams) on Kafka stream

### ❌ Anti-Pattern 4: No Cardinality Control

**Problem:** Unlimited label combinations (high cardinality)
**Why Bad:** Each unique label combo = new time series, memory/storage explodes
**Solution:** ✅ Limit cardinality:

- Don't use high-cardinality labels (`user_id`, `request_id`)
- Enforce label whitelist
- Monitor cardinality growth

### ❌ Anti-Pattern 5: No Query Cost Control

**Problem:** Allow expensive ad-hoc queries without limits
**Why Bad:** Single query overloads TSDB, impacts all users
**Solution:** ✅ Query cost control:

- Timeout (30 seconds)
- Rate limiting (10 concurrent/user)
- Cost estimation (warn users)
- Dedicated query pool

---

## Alternative Approaches

### Approach 1: Serverless (AWS CloudWatch)

**Pros:** ✅ No ops overhead, auto-scaling, cloud-integrated
**Cons:** ❌ Expensive ($500k/month for 100M metrics/sec), vendor lock-in, limited customization
**Use Case:** Small-medium scale (<1M metrics/sec), cloud-native apps

### Approach 2: Prometheus + Thanos

**Pros:** ✅ Cost-effective (S3 storage), unlimited retention, multi-cluster
**Cons:** ❌ Slower queries (S3), complex architecture, eventual consistency
**Use Case:** Medium scale (1-10M metrics/sec), long-term retention, cost-sensitive

### Approach 3: VictoriaMetrics

**Pros:** ✅ Cost-effective (10x cheaper), fast queries, simple (single binary)
**Cons:** ❌ Smaller community, single-vendor, limited horizontal scaling
**Use Case:** Cost-sensitive, high compression, simple architecture

---

## Monitoring the Monitoring System

**Challenge:** How to monitor the monitoring system itself?

**Solution: Dual-Layered Monitoring**

**Layer 1: Self-Monitoring**

- Collectors emit metrics about themselves
- Kafka emits metrics (consumer lag, rebalancing)
- TSDB emits metrics (write throughput, query latency)
- Alerting engine emits metrics (evaluation time)

**Layer 2: External Monitoring**

- Deploy separate lightweight system (Prometheus on separate infra)
- Monitor critical metrics of primary system
- Alert if primary system down

**Key Metrics:**

| Component  | Metric                         | Alert Threshold |
|------------|--------------------------------|-----------------|
| Collectors | `scrape_duration_seconds`      | >5 seconds      |
| Kafka      | `consumer_lag`                 | >1M messages    |
| TSDB       | `write_throughput_qps`         | <50M QPS        |
| TSDB       | `query_latency_p99_ms`         | >5000ms         |
| Alerting   | `alert_evaluation_duration_ms` | >10 seconds     |

---

## Trade-offs Summary

| What We Gain                         | What We Sacrifice                              |
|--------------------------------------|------------------------------------------------|
| ✅ High write throughput (100M/sec)   | ❌ Expensive infrastructure (100+ nodes, 10 PB) |
| ✅ Real-time alerting (<5 seconds)    | ❌ Complex stream processing (stateful Flink)   |
| ✅ Long-term retention (2+ years)     | ❌ Aggressive rollups (lose granularity)        |
| ✅ Fast dashboard queries (<1 second) | ❌ Eventual consistency (rollups lag)           |
| ✅ Horizontal scalability             | ❌ Operational complexity (100+ nodes)          |
| ✅ Multi-region (low latency)         | ❌ 3x infrastructure cost                       |

---

## Real-World Examples

### Uber's M3

- **Scale:** 10M+ metrics/sec, 100+ PB, 100k+ hosts
- **Storage:** M3DB (custom TSDB)
- **Key:** Built M3DB for 10M writes/sec, 10:1 compression

### Datadog

- **Scale:** 500B metrics/day, multi-exabyte storage
- **Architecture:** Push-based, custom TSDB
- **Key:** Multi-tenancy, tiered storage (SSD/HDD/S3)

### Netflix's Atlas

- **Scale:** 2B metrics/minute, 100k+ instances, 1 PB
- **Storage:** Cassandra + custom TSDB layer
- **Key:** In-memory aggregation, custom compression

---

## References

### Official Documentation

- **M3DB Documentation:** https://m3db.io/docs/
- **Prometheus Documentation:** https://prometheus.io/docs/
- **InfluxDB Documentation:** https://docs.influxdata.com/
- **OpenTelemetry Collector:** https://opentelemetry.io/docs/collector/
- **Thanos Documentation:** https://thanos.io/tip/thanos/getting-started.md/

### Technical Papers

- **Gorilla: A Fast, Scalable, In-Memory Time Series Database** (Facebook, 2015)
    - Delta-of-delta encoding and XOR compression
    - https://www.vldb.org/pvldb/vol8/p1816-teller.pdf

- **M3: Uber's Open Source, Large-scale Metrics Platform** (Uber, 2018)
    - Custom TSDB for 10M writes/sec
    - https://eng.uber.com/m3/

- **Monarch: Google's Planet-Scale In-Memory Time Series Database** (Google, 2020)
    - Distributed in-memory TSDB with 10B metrics/sec
    - https://research.google/pubs/pub49897/

- **Atlas: Netflix's Telemetry Platform** (Netflix, 2018)
    - In-memory aggregation and custom compression
    - https://netflixtechblog.com/introducing-atlas-netflixs-primary-telemetry-platform-bd31f4d8ed9a

### Related Chapters

From this handbook:

**01-principles/** (Foundational Concepts)

- [1.1.2 Latency, Throughput, and Scale](../../01-principles/1.1.2-latency-throughput-scale.md) - Understanding QPS and
  throughput calculations
- [1.1.4 Data Consistency Models](../../01-principles/1.1.4-data-consistency-models.md) - Eventual consistency in
  multi-region
- [1.1.5 Back-of-Envelope Calculations](../../01-principles/1.1.5-back-of-envelope-calculations.md) - Storage and
  bandwidth estimation

**02-components/** (Deep Dives)

- [2.1.2 NoSQL Deep Dive](../../02-components/2.1.2-no-sql-deep-dive.md) - Cassandra for time-series data
- [2.2.1 Caching Deep Dive](../../02-components/2.2.1-caching-deep-dive.md) - Redis for query result caching
- [2.2.2 Consistent Hashing](../../02-components/2.2.2-consistent-hashing.md) - Sharding strategy for M3DB
- [2.3.2 Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md) - Kafka as write buffer
- [2.3.5 Batch vs Stream Processing](../../02-components/2.3.5-batch-vs-stream-processing.md) - Stream processing for
  alerting
- [2.4.2 Observability](../../02-components/2.4.2-observability.md) - Monitoring the monitoring system

**03-challenges/** (Related Design Problems)

- [3.1.2 Distributed Cache](../3.1.2-distributed-cache/) - Caching strategies
- [3.3.1 Live Chat System](../3.3.1-live-chat-system/) - Real-time data flow
- [3.3.4 Distributed Database](../3.3.4-distributed-database/) - Sharding and replication

### Key Takeaways

**1. Time-Series Databases Are Essential**

- Standard RDBMS cannot handle 100M writes/sec
- TSDB optimizations: Delta-of-delta encoding, columnar storage, aggressive compression
- Gorilla compression achieves 20:1 ratio for typical metrics

**2. Rollups Are Mandatory for Long-Term Storage**

- Raw 1-second data: 6.3 quadrillion points (infeasible to store)
- With rollups: 10 PB (feasible with compression)
- Automatic rollup selection based on query time range

**3. Stream Processing for Real-Time Alerting**

- Cannot wait for data to be fully written to TSDB (adds latency)
- Kafka Streams or Flink process alert rules on stream
- Sub-5-second alerting from metric emission to notification

**4. Cardinality Is the Silent Killer**

- High-cardinality labels (user_id, request_id) explode time series count
- 1M unique label combinations = 1M time series
- Use label value limits, drop high-cardinality labels, or aggregate

**5. Multi-Region for Global Scale**

- Regional collection (low latency): <50ms in-region
- Global aggregation (eventual consistency): 5-30s lag acceptable
- 3x infrastructure cost but necessary for global deployments

**6. Monitoring the Monitoring System Is Critical**

- Self-monitoring: System monitors itself (Kafka lag, TSDB throughput)
- External monitoring: Separate lightweight system for failover detection
- Meta-metrics: How healthy is the alerting system itself?

**7. Cost Optimization Through Tiering**

- Hot tier (last 7 days): SSD, high IOPS, expensive
- Warm tier (7-90 days): SSD, lower IOPS, medium cost
- Cold tier (>90 days): HDD or S3, slow queries, cheap
- Tiering reduces cost by 5-10x for long-term retention

### Performance Characteristics Summary

| Aspect                         | Target            | Achieved                                          |
|--------------------------------|-------------------|---------------------------------------------------|
| **Write Throughput**           | 100M points/sec   | 100M points/sec with Kafka buffer + M3DB sharding |
| **Write Latency (p99)**        | <100ms            | ~50ms (collection → Kafka → TSDB write)           |
| **Query Latency (dashboard)**  | <1 second         | <500ms with Redis cache (80% hit rate)            |
| **Query Latency (long-range)** | <2 seconds        | <2 seconds with 1-hour rollups (30 days query)    |
| **Alert Latency**              | <5 seconds        | <3 seconds (stream processing on Kafka)           |
| **Storage Efficiency**         | 10 PB for 2 years | 10 PB (20:1 compression + rollups)                |
| **Query Cache Hit Rate**       | >70%              | 80% (Redis query result cache)                    |
| **Availability**               | 99.9%             | 99.95% (multi-region, 3 replicas)                 |

### Cost Estimation (AWS, 2-Year Retention)

**Infrastructure Costs (Monthly):**

| Component           | Configuration                          | Monthly Cost        |
|---------------------|----------------------------------------|---------------------|
| **Kafka Cluster**   | 12 nodes (i3.2xlarge)                  | $10,000             |
| **M3DB Cluster**    | 100 nodes (r5.4xlarge, 500GB SSD each) | $80,000             |
| **Query Service**   | 20 nodes (c5.2xlarge)                  | $6,000              |
| **Alerting Engine** | 10 nodes (c5.xlarge)                   | $2,000              |
| **Redis Cache**     | 5 nodes (r5.xlarge)                    | $2,000              |
| **S3 (cold tier)**  | 5 PB (archival)                        | $2,500              |
| **Data Transfer**   | Cross-AZ, multi-region                 | $5,000              |
| **Total**           |                                        | **~$107,500/month** |

**Cost per Million Metrics per Second:** ~$1,075/month

**Cost Breakdown:**

- 75% Storage (TSDB nodes + S3)
- 15% Compute (Query, Alerting)
- 10% Networking (Multi-region replication)

**Cost Optimization Strategies:**

1. Use Spot Instances for non-critical nodes (30-70% savings)
2. S3 Glacier for >1 year data (90% savings on cold tier)
3. Aggressive rollups (reduce storage by 100x)
4. Regional deployments only (eliminate cross-region costs)
5. VictoriaMetrics or Thanos (10x cheaper than M3DB)
