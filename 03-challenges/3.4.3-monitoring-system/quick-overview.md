# Distributed Monitoring System - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A distributed monitoring system that collects, stores, and analyzes metrics from millions of endpoints (servers,
containers, applications) in real-time. The system collects metrics (CPU, memory, disk, latency, QPS, error rates) from
1M+ endpoints, stores time-series data efficiently for long-term retention (2+ years), alerts on anomalies and threshold
violations in real-time (<5 seconds), visualizes metrics through dashboards for historical analysis and troubleshooting,
and scales horizontally to handle massive write throughput (1M writes/sec).

**Similar Systems:** Prometheus, Datadog, New Relic, Grafana Cloud, AWS CloudWatch

**Key Characteristics:**

- **High write throughput** - 1M writes/sec sustained, 5M writes/sec peak
- **Low write latency** - <100ms p99 from metric collection to storage
- **Fast query performance** - <1 second for dashboard queries (last 1 hour)
- **High availability** - 99.9% uptime for alerting and query services
- **Durability** - No data loss for critical metrics
- **Scalability** - Horizontal scaling for both writes and reads

---

## Requirements & Scale

### Functional Requirements

1. **Metrics Collection**
    - Pull-based collection (scraping endpoints like Prometheus)
    - Push-based collection (agents push metrics)
    - Support for custom metrics and labels
    - Auto-discovery of new endpoints

2. **Time-Series Storage**
    - Store metrics with high resolution (1-second granularity)
    - Support retention policies (rollup and downsampling over time)
    - Efficient compression for long-term storage

3. **Real-Time Alerting**
    - Evaluate alert rules in real-time (e.g., `CPU > 80% for 5 minutes`)
    - Support complex queries (aggregations, percentiles)
    - Trigger notifications (email, Slack, PagerDuty)
    - Alert deduplication and grouping

4. **Visualization**
    - Dashboards with customizable graphs
    - Support for ad-hoc queries
    - Historical data analysis
    - Anomaly detection and forecasting

### Non-Functional Requirements

1. **High Write Throughput:** 1M writes/sec sustained, 5M writes/sec peak
2. **Low Write Latency:** <100ms p99 from metric collection to storage
3. **Fast Query Performance:** <1 second for dashboard queries (last 1 hour)
4. **High Availability:** 99.9% uptime for alerting and query services
5. **Durability:** No data loss for critical metrics (best-effort for non-critical)
6. **Scalability:** Horizontal scaling for both writes and reads

### Scale Estimation

| Metric                        | Assumption                                       | Calculation                    | Result                                                |
|-------------------------------|--------------------------------------------------|--------------------------------|-------------------------------------------------------|
| **Total Endpoints**           | 1M servers/containers                            | -                              | 1 Million endpoints                                   |
| **Metrics per Endpoint**      | 100 metrics (CPU, memory, disk, network, custom) | -                              | 100 metrics per endpoint                              |
| **Collection Frequency**      | 1 sample per second                              | -                              | 1 data point/sec per metric                           |
| **Write Throughput**          | $1\text{M}$ endpoints × $100$ metrics × $1$/sec  | -                              | **100M data points/sec** ($100\text{M}$ $\text{QPS}$) |
| **Storage (1 Day)**           | $100\text{M}$ points/sec × $86,400$ sec/day      | -                              | 8.64 trillion data points/day                         |
| **Storage (2 Years)**         | $8.64$ trillion × $365$ × $2$                    | -                              | **6.3 quadrillion data points** (petabyte scale)      |
| **Storage Size (Raw)**        | 16 bytes per point (timestamp + value + labels)  | $6.3$ quadrillion × $16$ bytes | **~100 PB** (before compression)                      |
| **Storage Size (Compressed)** | 10:1 compression ratio                           | $100$ PB / $10$                | **~10 PB** (after compression)                        |
| **Query Throughput**          | 100k dashboards × 10 queries/min                 | -                              | **~16k QPS** (read queries)                           |

**Key Insight:** The system is **write-heavy** (100M writes/sec vs 16k reads/sec). Storage must be optimized for
sequential writes and time-based compression.

---

## Key Components

| Component                 | Responsibility                          | Technology                  | Scalability                |
|---------------------------|-----------------------------------------|-----------------------------|----------------------------|
| **Collectors**            | Scrape/pull metrics from endpoints      | Prometheus                  | Horizontal (100 instances) |
| **Kafka Buffer**          | Decouple collectors from storage        | Apache Kafka                | Horizontal (24 partitions) |
| **TSDB**                  | Store time-series data with compression | M3DB, InfluxDB              | Horizontal (50+ nodes)     |
| **Alerting Engine**       | Evaluate alert rules in real-time       | Apache Flink, Kafka Streams | Horizontal (stateless)     |
| **Query Service**         | Serve dashboard and ad-hoc queries      | M3Query, PromQL             | Horizontal (stateless)     |
| **Visualization**         | Dashboards and graphs                   | Grafana                     | Horizontal (stateless)     |
| **Real-Time Aggregation** | Downsampling and rollups                | Flink, Kafka Streams        | Horizontal                 |

---

## Architecture Flow

**Time-Series Data Pipeline:**

1. **Collection Layer**
    - Pull-based: Prometheus scrapes endpoints every 1 second
    - Push-based: Agents push metrics to collectors
    - 1M endpoints × 100 metrics × 1/sec = 100M writes/sec

2. **Buffer Layer (Kafka)**
    - Decouples collectors from storage
    - 24 partitions, 3× replication
    - 24-hour retention for replay capability

3. **Storage Layer (TSDB)**
    - M3DB or InfluxDB (specialized time-series database)
    - Gorilla compression (10:1 ratio)
    - Sharded by metric name + labels (consistent hashing)

4. **Alerting Engine**
    - Stream processing (Flink/Kafka Streams)
    - Stateful windowed aggregations
    - Real-time evaluation (<5 seconds)

5. **Query Service**
    - PromQL-compatible query engine
    - Automatic rollup selection (use 1-hour rollup for long-range queries)
    - Query result caching (Redis)

6. **Visualization**
    - Grafana dashboards
    - Custom web UI
    - Historical analysis and forecasting

---

## Time-Series Database (TSDB)

**Why Specialized TSDB?**

| Factor                 | RDBMS (PostgreSQL)        | TSDB (M3DB/InfluxDB)          |
|------------------------|---------------------------|-------------------------------|
| **Write Throughput**   | ❌ 10K writes/sec          | ✅ 1M+ writes/sec per node     |
| **Compression**        | ❌ No built-in compression | ✅ Gorilla compression (10:1)  |
| **Time-Range Queries** | ❌ Slow (index scan)       | ✅ Fast (time-ordered storage) |
| **Retention Policies** | ❌ Manual cleanup          | ✅ Automatic downsampling      |
| **Storage Cost**       | ❌ High (no compression)   | ✅ Low (10× compression)       |

**M3DB Architecture:**

- **M3DB Nodes** - Distributed storage nodes (50+ nodes)
- **M3Coordinator** - Query aggregator and downsampler
- **M3Aggregator** - Real-time aggregation and rollups
- **M3Query** - PromQL-compatible query engine

**Sharding Strategy:**

- **Shard by metric name + labels** (consistent hashing)
- **Replication factor:** 3 (for durability)
- **Shard count:** 4096 (for fine-grained distribution)

**Compression:**

- **Gorilla compression** (Facebook's time-series compression algorithm)
- **Compression ratio:** 10:1 (100 PB → 10 PB)
- **Delta-of-delta encoding** for timestamps
- **XOR encoding** for float values

---

## Real-Time Aggregation (Rollups)

**Why Rollups?**

- Storing 6.3 quadrillion data points at 1-second resolution is too expensive
- Dashboard queries over long time ranges (e.g., "last 30 days") would be slow
- Downsampling reduces storage and improves query performance

**Rollup Strategy:**

**Retention Policy:**

- **Raw (1-second):** 1 day (8.64 trillion points)
- **1-minute rollup:** 7 days (1.4 trillion points)
- **5-minute rollup:** 30 days (8.6 billion points)
- **1-hour rollup:** 1 year (876 million points)
- **1-day rollup:** 2+ years (730,000 points per metric)

**Rollup Computation:**

- **Stream processor** (Flink or Kafka Streams)
- **Input:** Kafka topic `metrics.raw`
- **Processing:** Windowed aggregations (1-minute, 5-minute, 1-hour, 1-day)
- **Output:** Write to TSDB rollup tables

**Aggregation Functions:**

- `count` - Number of samples in window
- `sum` - Sum of values
- `min`, `max` - Min/max values
- `avg` - Average value
- `p50`, `p95`, `p99` - Percentiles

**Query Optimization:**

- **Automatic rollup selection:** Query optimizer uses 1-hour rollup for 30-day queries (3600× fewer points)
- **Query result caching:** Redis cache with 80% hit rate
- **Downsampling at query time:** If query asks for 1000 points over 30 days, downsample to 1000 points

---

## Alerting Engine

**Requirements:**

- Evaluate alert rules in real-time (<5 seconds)
- Support complex queries (aggregations, percentiles, windows)
- Handle stateful queries (e.g., "CPU > 80% for 5 minutes")
- Deduplicate and group alerts

**Architecture:**

**Stream Processor:** Apache Flink or Kafka Streams

- **Input:** Kafka topic `metrics.raw`
- **Processing:** Stateful windowed aggregations
- **Output:** Kafka topic `alerts.triggered`

**Alert Rule Example:**

```
Alert: HighCPUUsage
Condition: avg(cpu_usage{service="api-gateway"}) > 80 for 5 minutes
Severity: warning
Notification: slack-channel=#alerts
```

**Processing Flow:**

1. **Consume metrics** from Kafka
2. **Group by alert rule** (e.g., all `cpu_usage` metrics for `api-gateway`)
3. **Windowed aggregation** (5-minute tumbling window)
4. **Evaluate condition** (avg > 80%)
5. **Trigger alert** if condition met for entire window
6. **Deduplicate** (don't send alert if already firing)
7. **Publish** to `alerts.triggered` topic

**Stateful Processing:**

- **State store:** RocksDB (embedded key-value store)
- **State:** Current window aggregations (count, sum, min, max)
- **Checkpointing:** Periodic snapshots for fault tolerance

**Alert Deduplication:**

- Track alert state (firing, resolved)
- Don't send duplicate alerts if already firing
- Send "resolved" notification when condition clears

**Alert Grouping:**

- Group alerts by labels (e.g., all alerts for `region=us-east-1`)
- Single notification for multiple related alerts
- Reduces notification spam

---

## Bottlenecks & Solutions

### Bottleneck 1: Kafka Write Throughput

**Problem:** 100M writes/sec requires high Kafka throughput (sustained 200 MB/s per partition).

**Solution:**

- **Increase partitions:** 24 → 48 partitions (distribute load)
- **Batching:** Collectors batch 1000 metrics before sending to Kafka
- **Compression:** Use snappy compression (5× reduction)
- **Async producers:** Non-blocking writes

**Performance:**

- Before: 100M writes/sec, 200 MB/s per partition (at limit)
- After: 100M writes/sec, 100 MB/s per partition (comfortable headroom)

### Bottleneck 2: TSDB Write Throughput

**Problem:** TSDB struggles with random writes (even though time-series data is mostly sequential, labels create
randomness).

**Solution:**

- **Write buffer (M3DB aggregator):** Buffer writes in memory (1 minute), batch to disk
- **LSM-Tree storage engine:** Optimized for write-heavy workloads
- **Compression:** Gorilla compression reduces disk writes by 10×
- **Horizontal scaling:** Add more TSDB nodes (50+ nodes)

**Performance:**

- Single node: 1M writes/sec
- 100 nodes: 100M writes/sec (linear scaling)

### Bottleneck 3: Dashboard Query Latency

**Problem:** Dashboard queries over long time ranges (e.g., "last 30 days") fetch millions of data points (slow).

**Solution:**

- **Automatic rollup selection:** Query optimizer uses 1-hour rollup (3600× fewer points)
- **Query result caching:** Redis cache with 80% hit rate
- **Downsampling at query time:** If query asks for 1000 points over 30 days, downsample to 1000 points (not millions)

**Performance:**

- Before: 30-day query = 2.6 billion points, 10 seconds
- After: 30-day query (1-hour rollup) = 720 points, 100ms

### Bottleneck 4: Storage Capacity

**Problem:** 10 PB storage (after compression) is expensive and hard to manage.

**Solution:**

- **Tiered storage:** Hot (SSD), warm (HDD), cold (S3/Glacier)
    - **Hot tier (0-7 days):** SSD, fast queries
    - **Warm tier (7-90 days):** HDD, slower queries
    - **Cold tier (90+ days):** S3/Glacier, archive only
- **Aggressive rollups:** Only store 1-day rollups for data >1 year old
- **Selective retention:** Critical metrics retained longer, non-critical metrics deleted after 30 days

**Cost Analysis:**

```
Hot tier (7 days): 800 TB × $0.10/GB-month = $80k/month
Warm tier (83 days): 8 PB × $0.02/GB-month = $160k/month
Cold tier (2 years): 1 PB × $0.004/GB-month = $4k/month

Total storage cost: $244k/month
```

---

## Common Anti-Patterns to Avoid

### 1. Using RDBMS for Time-Series Data

❌ **Bad:** Using PostgreSQL/MySQL to store time-series metrics

**Why It's Bad:**

- RDBMS not optimized for high write throughput (B-Tree indexes, write amplification)
- Time-range queries slow (requires index scan)
- No built-in compression or retention policies
- Disk usage explodes (no downsampling)

✅ **Good:** Use specialized TSDB (InfluxDB, M3DB, TimescaleDB)

### 2. No Rollups/Downsampling

❌ **Bad:** Storing all metrics at full resolution (1-second) forever

**Why It's Bad:**

- Storage cost explodes (100 PB for 2 years)
- Dashboard queries over long time ranges are slow (fetching billions of points)
- Most queries don't need 1-second resolution for historical data

✅ **Good:** Implement aggressive rollups (1-minute, 1-hour, 1-day) with retention policies

### 3. Synchronous Alert Evaluation on TSDB Queries

❌ **Bad:** Alerting engine queries TSDB every 10 seconds to evaluate alert rules

**Why It's Bad:**

- High query load on TSDB (10k alert rules × 6 queries/min = 1M queries/min)
- Alert latency high (10+ seconds)
- TSDB query load competes with dashboard queries

✅ **Good:** Use stream processing (Flink/Kafka Streams) to evaluate alerts on Kafka stream in real-time

### 4. No Cardinality Control

❌ **Bad:** Allowing unlimited unique label combinations (high cardinality)

**Why It's Bad:**

- Each unique label combination creates a new time series
- Example: `request_id` label with 1 billion unique values = 1 billion time series
- TSDB memory and storage explodes
- Queries become slow (needs to scan millions of time series)

✅ **Good:** Limit label cardinality:

- Don't use high-cardinality labels (e.g., `user_id`, `request_id`)
- Enforce label whitelist (only allow approved labels)
- Monitor cardinality growth and alert

### 5. No Query Cost Control

❌ **Bad:** Allowing users to run expensive ad-hoc queries without limits

**Why It's Bad:**

- A single expensive query can overload TSDB (e.g., "Show all metrics for all services over last year")
- Impacts all users (dashboard queries slow)
- TSDB nodes crash due to memory exhaustion

✅ **Good:** Implement query cost control:

- **Query timeout:** Kill queries running >30 seconds
- **Query rate limiting:** Max 10 concurrent queries per user
- **Query cost estimation:** Warn users before running expensive queries
- **Dedicated query pool:** Separate TSDB replicas for ad-hoc queries

---

## Monitoring & Observability

### Key Metrics

**System Metrics:**

- **Write throughput:** 100M writes/sec (target)
- **Write latency:** <100ms p99 (target)
- **Query latency:** <1 second p99 (target)
- **Storage usage:** 10 PB (after compression)
- **Alert evaluation latency:** <5 seconds (target)
- **Cache hit rate:** >80% (target)

**Business Metrics:**

- **Endpoints monitored:** 1M endpoints
- **Alerts triggered:** 100k+ alerts/day
- **Dashboard queries:** 16k QPS
- **Uptime:** 99.9% (target)

### Dashboards

1. **System Health Dashboard**
    - Write throughput (writes/sec)
    - Query latency (p50, p95, p99)
    - Storage usage (by tier: hot, warm, cold)
    - TSDB node health (CPU, memory, disk)

2. **Alerting Dashboard**
    - Alerts triggered (by severity, by service)
    - Alert evaluation latency
    - Alert deduplication rate
    - Notification delivery success rate

3. **Query Performance Dashboard**
    - Query latency distribution
    - Query throughput (QPS)
    - Cache hit rate
    - Expensive queries (top 10 slowest queries)

### Alerts

**Critical Alerts (PagerDuty, 24/7 on-call):**

- TSDB write latency >500ms for 5 minutes
- Query latency >5 seconds for 5 minutes
- TSDB node down (unavailable)
- Alert evaluation engine down

**Warning Alerts (Slack, email):**

- Storage usage >90% capacity
- Cache hit rate <70%
- Query throughput >20k QPS (approaching limit)
- High cardinality detected (>1M time series per metric)

---

## Trade-offs Summary

| What We Gain                                  | What We Sacrifice                                      |
|-----------------------------------------------|--------------------------------------------------------|
| ✅ **High write throughput** (100M writes/sec) | ❌ Expensive infrastructure (100+ nodes, 10 PB storage) |
| ✅ **Real-time alerting** (<5 seconds)         | ❌ Complex stream processing (stateful Flink jobs)      |
| ✅ **Long-term retention** (2+ years)          | ❌ Aggressive rollups (lose granularity for old data)   |
| ✅ **Fast dashboard queries** (<1 second)      | ❌ Eventual consistency (rollups lag behind raw data)   |
| ✅ **Horizontal scalability** (add nodes)      | ❌ Operational complexity (manage 100+ nodes)           |
| ✅ **Multi-region deployment** (low latency)   | ❌ 3× infrastructure cost                               |
| ✅ **Automatic downsampling** (reduce storage) | ❌ Cannot reconstruct original raw data                 |

---

## Real-World Examples

### Uber's M3 Monitoring System

- **Scale:** 10M+ metrics/sec from 100k+ hosts, 100+ PB storage, 1M+ queries/day, 100k+ alerts/day
- **Architecture:** Pull-based (Prometheus-compatible), M3DB (custom TSDB), M3Query + custom alerting engine, Grafana
- **Key Insights:** Built M3DB because existing TSDB couldn't scale to 10M writes/sec, aggressive compression (Gorilla +
  downsampling) → 10:1 ratio, multi-region deployment with 5 seconds replication lag

### Datadog (SaaS Monitoring Platform)

- **Scale:** 25k+ companies, 500 billion metrics/day, multi-exabyte storage, 10M+ queries/day
- **Architecture:** Push-based (Datadog agent), custom TSDB (proprietary), real-time stream processing, custom web UI
- **Key Insights:** Multi-tenancy (data isolation per customer), tiered storage (hot SSD, warm HDD, cold S3), dynamic
  rollups based on query patterns

### Netflix's Atlas Monitoring System

- **Scale:** 2 billion metrics/minute, 100k+ instances, Cassandra (1 PB), 1M+ queries/day
- **Architecture:** Push-based (in-process collection), Cassandra + custom TSDB layer, Atlas query language (AQL),
  custom dashboards
- **Key Insights:** Built on Cassandra (not specialized TSDB) for operational simplicity, custom compression algorithm,
  in-memory aggregation (pre-aggregate before writing to Cassandra)

---

## Key Takeaways

1. **Specialized TSDB** (M3DB, InfluxDB) essential for high write throughput (100M writes/sec)
2. **Gorilla compression** reduces storage by 10× (100 PB → 10 PB)
3. **Rollups/downsampling** critical for long-term retention (1-second → 1-hour → 1-day)
4. **Stream processing** (Flink/Kafka Streams) for real-time alerting (<5 seconds)
5. **Kafka buffer** decouples collectors from storage (handles bursts)
6. **Tiered storage** (hot/warm/cold) reduces cost ($244k/month → $80k/month for hot tier only)
7. **Cardinality control** prevents storage explosion (limit high-cardinality labels)
8. **Query cost control** protects TSDB from expensive ad-hoc queries
9. **Multi-region deployment** for low-latency collection (regional + global aggregation)
10. **Automatic rollup selection** optimizes query performance (use 1-hour rollup for 30-day queries)

---

## Recommended Stack

- **Collection:** Prometheus (pull-based), Telegraf (push-based agents)
- **Buffer:** Apache Kafka (24 partitions, 3× replication)
- **TSDB:** M3DB or InfluxDB (50+ nodes, Gorilla compression)
- **Alerting:** Apache Flink or Kafka Streams (stateful windowed aggregations)
- **Query Service:** M3Query or PromQL (Prometheus Query Language)
- **Visualization:** Grafana (dashboards), Custom web UI
- **Caching:** Redis (query result cache, 80% hit rate)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, component design, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (collection, alerting, querying)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (rollups, alert evaluation, compression)

