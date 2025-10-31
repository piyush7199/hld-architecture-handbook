# Ad Click Aggregator - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made when designing an ad click aggregator
system capable of handling 500k events/sec with real-time analytics and accurate billing.

---

## Table of Contents

1. [Kappa vs Lambda Architecture](#1-kappa-vs-lambda-architecture)
2. [Kafka vs Kinesis vs Pulsar](#2-kafka-vs-kinesis-vs-pulsar)
3. [Flink vs Spark Streaming vs Storm](#3-flink-vs-spark-streaming-vs-storm)
4. [Redis vs ClickHouse for Real-Time Storage](#4-redis-vs-clickhouse-for-real-time-storage)
5. [Parquet vs Avro vs ORC for Cold Storage](#5-parquet-vs-avro-vs-orc-for-cold-storage)
6. [Bloom Filter vs Hash Table for Fraud Detection](#6-bloom-filter-vs-hash-table-for-fraud-detection)
7. [Partition by Campaign ID vs Random Partitioning](#7-partition-by-campaign-id-vs-random-partitioning)
8. [Two-Speed Pipeline vs Single Pipeline](#8-two-speed-pipeline-vs-single-pipeline)
9. [Exactly-Once vs At-Least-Once Semantics](#9-exactly-once-vs-at-least-once-semantics)
10. [Synchronous vs Asynchronous API Response](#10-synchronous-vs-asynchronous-api-response)

---

## 1. Kappa vs Lambda Architecture

### The Problem

How should we architect the data processing pipeline to support both real-time analytics and accurate batch billing?

### Options Considered

| Aspect               | Kappa Architecture                    | Lambda Architecture                   |
|----------------------|---------------------------------------|---------------------------------------|
| **Pipeline**         | Single stream processing pipeline     | Separate batch and stream pipelines   |
| **Code Maintenance** | One codebase                          | Two codebases (batch + stream)        |
| **Consistency**      | Easier to maintain consistency        | Risk of divergence between paths      |
| **Reprocessing**     | Replay from Kafka (limited retention) | Easy full reprocessing from data lake |
| **Complexity**       | Lower operational complexity          | Higher complexity (two systems)       |
| **Flexibility**      | Stream-first, batch for corrections   | Batch-first, stream for speed         |

### Decision Made

**Kappa Architecture** with two-speed processing.

### Rationale

1. **Unified Codebase:** Single stream processing logic in Flink eliminates code duplication
2. **Consistency:** Same transformation logic for real-time and batch = no divergence
3. **Operational Simplicity:** One pipeline to monitor, debug, and optimize
4. **Modern Tooling:** Flink supports both stream and batch processing natively
5. **Cost Efficiency:** No need to maintain separate batch infrastructure

**Implementation:**

- Fast path: Flink stream processing → Redis (real-time dashboard)
- Slow path: Spark batch from S3 → PostgreSQL (billing accuracy)
- Kafka serves as the unified log for both paths

### Trade-offs Accepted

| What We Gain         | What We Sacrifice                                       |
|----------------------|---------------------------------------------------------|
| ✅ Single codebase    | ❌ Harder full reprocessing (limited by Kafka retention) |
| ✅ Easier consistency | ❌ Kafka storage cost (7-day retention)                  |
| ✅ Lower complexity   | ❌ Less flexibility for different batch logic            |

### When to Reconsider

- If batch processing requirements diverge significantly from real-time
- If Kafka retention becomes too expensive at scale
- If full historical reprocessing is frequently needed

---

## 2. Kafka vs Kinesis vs Pulsar

### The Problem

Which event streaming platform should we use for handling 500k events/sec with 7-day retention?

### Options Considered

| Feature         | Kafka                     | Kinesis                 | Pulsar                     |
|-----------------|---------------------------|-------------------------|----------------------------|
| **Throughput**  | 1M+ msg/sec per broker    | 1 MB/sec per shard      | 1M+ msg/sec                |
| **Retention**   | Unlimited (disk-based)    | 7 days max              | Unlimited (tiered storage) |
| **Cost**        | $0.01/GB (self-managed)   | $0.015/GB + shard hours | $0.02/GB                   |
| **Scaling**     | Manual partition add      | Auto-scaling shards     | Auto-scaling               |
| **Ordering**    | Per-partition             | Per-shard               | Per-key                    |
| **Operational** | Self-managed (complex)    | Fully managed           | Self or managed            |
| **Ecosystem**   | Rich (Flink, Spark, etc.) | AWS-only                | Growing                    |
| **Latency**     | < 10ms                    | ~100ms                  | < 10ms                     |

### Decision Made

**Apache Kafka** (self-managed on AWS MSK).

### Rationale

1. **Cost:** 10x cheaper than Kinesis at our scale ($30k/month vs $300k/month)
2. **Throughput:** Can handle 500k events/sec easily with 10 brokers
3. **Retention:** 7-day retention with no extra cost (vs Kinesis 7-day limit)
4. **Ecosystem:** Best integration with Flink, Spark, and data lake tools
5. **Proven at Scale:** Used by LinkedIn, Uber, Netflix at petabyte scale

**Configuration:**

- AWS MSK (managed Kafka)
- 10 brokers (kafka.m5.4xlarge)
- 100 partitions for parallelism
- Replication factor: 3
- Compression: LZ4

### Trade-offs Accepted

| What We Gain        | What We Sacrifice                          |
|---------------------|--------------------------------------------|
| ✅ 10x lower cost    | ❌ More operational complexity (vs Kinesis) |
| ✅ Higher throughput | ❌ No auto-scaling (manual partition add)   |
| ✅ Longer retention  | ❌ Self-managed upgrades                    |
| ✅ Rich ecosystem    | ❌ Vendor lock-in (less than AWS)           |

### When to Reconsider

- If operational overhead becomes unsustainable
- If auto-scaling is critical requirement
- If already heavily invested in AWS ecosystem
- If throughput requirements exceed 10M events/sec (consider Pulsar)

---

## 3. Flink vs Spark Streaming vs Storm

### The Problem

Which stream processing framework should we use for real-time aggregation and fraud detection?

### Options Considered

| Feature              | Flink                        | Spark Streaming             | Storm          |
|----------------------|------------------------------|-----------------------------|----------------|
| **Processing Model** | True streaming               | Micro-batching              | True streaming |
| **Latency**          | < 100ms                      | ~1 second                   | < 100ms        |
| **Throughput**       | Very high                    | Very high                   | Medium         |
| **State Management** | RocksDB (disk-backed)        | In-memory                   | Manual         |
| **Fault Tolerance**  | Checkpointing (exactly-once) | RDD lineage                 | At-least-once  |
| **Windowing**        | Event-time, processing-time  | Micro-batches               | Manual         |
| **Ecosystem**        | Rich (Kafka, S3, Redis)      | Excellent (Spark ecosystem) | Basic          |
| **Maturity**         | Mature                       | Very mature                 | Legacy         |
| **Learning Curve**   | Moderate                     | Easy (if know Spark)        | Steep          |

### Decision Made

**Apache Flink** for stream processing.

### Rationale

1. **True Streaming:** Sub-second latency for real-time dashboard updates
2. **State Management:** Built-in RocksDB support for 10M+ counters
3. **Exactly-Once Semantics:** Critical for accurate counting (no duplicates)
4. **Event-Time Processing:** Handles late arrivals with watermarks
5. **Checkpointing:** Automatic fault tolerance with S3 snapshots

**Implementation:**

- 50 Flink TaskManagers (4 vCPU, 16 GB RAM each)
- RocksDB state backend for large state
- Checkpointing every 60 seconds to S3
- Parallelism: 100 (matches Kafka partitions)

### Trade-offs Accepted

| What We Gain                 | What We Sacrifice              |
|------------------------------|--------------------------------|
| ✅ Sub-second latency         | ❌ Higher complexity (vs Spark) |
| ✅ Exactly-once semantics     | ❌ Steeper learning curve       |
| ✅ Efficient state management | ❌ Smaller community (vs Spark) |
| ✅ Event-time processing      | ❌ More JVM tuning required     |

### When to Reconsider

- If team has strong Spark expertise
- If latency requirements relax to > 5 seconds
- If state size remains small (< 1 GB total)
- If micro-batching semantics are acceptable

---

## 4. Redis vs ClickHouse for Real-Time Storage

### The Problem

What storage system should we use for serving real-time aggregated counters with sub-100ms query latency?

### Options Considered

| Feature               | Redis              | ClickHouse      | DynamoDB         |
|-----------------------|--------------------|-----------------|------------------|
| **Query Latency**     | < 1ms              | 10-100ms        | 5-10ms           |
| **Data Model**        | Key-value, Hash    | Columnar SQL    | Key-value        |
| **Aggregation**       | Application-side   | SQL (fast)      | Application-side |
| **Write Throughput**  | 100k writes/sec    | 50k inserts/sec | 10k writes/sec   |
| **Memory Efficiency** | In-memory only     | Disk + cache    | Disk-based       |
| **Cost**              | $5k/month (384 GB) | $8k/month       | $15k/month       |
| **Complexity**        | Low                | Medium          | Low              |
| **Querying**          | Simple GET/MGET    | SQL (powerful)  | Key lookups only |

### Decision Made

**Redis Cluster** for real-time storage.

### Rationale

1. **Sub-Millisecond Latency:** Critical for responsive dashboard UX
2. **Simple Data Model:** Hash structure perfect for counters
3. **High Write Throughput:** Handles 100k writes/sec from Flink
4. **Proven at Scale:** Used by Twitter, Instagram for similar use cases
5. **Automatic TTL:** Old windows expire automatically (24-hour TTL)

**Implementation:**

- Redis Cluster: 6 nodes (3 masters + 3 replicas)
- cache.r6g.2xlarge instances (64 GB memory each)
- Sharded by campaign_id hash
- Redis Sentinel for automatic failover

### Trade-offs Accepted

| What We Gain            | What We Sacrifice                     |
|-------------------------|---------------------------------------|
| ✅ < 1ms query latency   | ❌ In-memory only (expensive at scale) |
| ✅ Simple operations     | ❌ No SQL queries (less flexible)      |
| ✅ High write throughput | ❌ LRU evictions under memory pressure |
| ✅ Mature ecosystem      | ❌ Manual aggregation logic            |

### When to Reconsider

- If query latency relaxes to > 100ms (consider ClickHouse)
- If SQL analytics become primary use case
- If memory cost becomes prohibitive
- If data size exceeds 1 TB

---

## 5. Parquet vs Avro vs ORC for Cold Storage

### The Problem

Which file format should we use for storing 15 PB of raw click events in S3 for long-term analysis?

### Options Considered

| Feature               | Parquet               | Avro                 | ORC            |
|-----------------------|-----------------------|----------------------|----------------|
| **Format**            | Columnar              | Row-based            | Columnar       |
| **Compression**       | 10:1 (excellent)      | 3:1 (good)           | 12:1 (best)    |
| **Read Performance**  | Fast (column pruning) | Slow (full row)      | Fastest        |
| **Write Performance** | Medium                | Fast                 | Slow           |
| **Splittable**        | Yes                   | Yes                  | Yes            |
| **Schema Evolution**  | Good                  | Excellent            | Good           |
| **Ecosystem**         | Spark, Athena, Presto | Kafka, Flink         | Hive, Spark    |
| **Use Case**          | Analytics (OLAP)      | Streaming, messaging | Hive workloads |

### Decision Made

**Parquet** with Snappy compression.

### Rationale

1. **Compression:** 10:1 ratio reduces storage cost ($100k/month vs $1M/month)
2. **Analytics:** Columnar format perfect for Spark batch jobs (column pruning)
3. **Ecosystem:** Best support in AWS (Athena, Redshift Spectrum)
4. **Splittable:** Parallel reads for Spark processing
5. **Standard:** Industry standard for data lakes

**Implementation:**

- File size: 128 MB (optimal for Spark)
- Compression: Snappy (fast read/write)
- Partitioning: year/month/day/hour
- S3 lifecycle: Standard → IA → Glacier

### Trade-offs Accepted

| What We Gain          | What We Sacrifice         |
|-----------------------|---------------------------|
| ✅ 10x compression     | ❌ Slower writes (vs Avro) |
| ✅ Fast column reads   | ❌ Slower full-row scans   |
| ✅ Lower storage cost  | ❌ More complex format     |
| ✅ Analytics-optimized | ❌ Not ideal for streaming |

### When to Reconsider

- If write performance becomes critical
- If full-row scans are primary use case
- If switching to Hive ecosystem (consider ORC)
- If real-time analytics from S3 needed (consider Avro)

---

## 6. Bloom Filter vs Hash Table for Fraud Detection

### The Problem

How should we check if an IP or user-agent is a known bad actor (100 million entries)?

### Options Considered

| Feature             | Bloom Filter     | Hash Table      | Database Lookup |
|---------------------|------------------|-----------------|-----------------|
| **Memory**          | 150 MB           | 10 GB           | Disk-based      |
| **Lookup Time**     | O(1) - 10ns      | O(1) - 50ns     | O(1) - 10ms     |
| **False Positives** | 0.1%             | 0%              | 0%              |
| **False Negatives** | 0%               | 0%              | 0%              |
| **Updates**         | Rebuild required | In-place update | Easy update     |
| **Scalability**     | Excellent        | Poor (memory)   | Good (disk)     |

### Decision Made

**Bloom Filter** for known bad actor checks.

### Rationale

1. **Memory Efficiency:** 150 MB vs 10 GB for 100M entries
2. **Speed:** O(1) lookup in ~10 nanoseconds
3. **False Positive Rate:** 0.1% acceptable (manually review)
4. **Scalability:** Can fit in memory on every Flink worker
5. **No Network Calls:** Local check (no DB roundtrip)

**Implementation:**

- Size: 100 million elements
- Hash functions: 7 (optimal for 0.1% FPR)
- Memory: ~150 MB per Flink worker
- Update frequency: Hourly (rebuild from fraud DB)

### Trade-offs Accepted

| What We Gain             | What We Sacrifice                    |
|--------------------------|--------------------------------------|
| ✅ 100x memory savings    | ❌ 0.1% false positives               |
| ✅ Sub-microsecond lookup | ❌ Cannot delete elements             |
| ✅ No network calls       | ❌ Rebuild required for updates       |
| ✅ Local to each worker   | ❌ Not suitable for real-time updates |

### When to Reconsider

- If false positives are unacceptable
- If real-time fraud list updates are critical
- If memory is not a constraint
- If bad actor list is small (< 1 million entries)

---

## 7. Partition by Campaign ID vs Random Partitioning

### The Problem

How should we partition Kafka events to ensure efficient processing and ordering?

### Options Considered

| Strategy           | Campaign ID Hash           | Random / Round-Robin  | User ID Hash          |
|--------------------|----------------------------|-----------------------|-----------------------|
| **Event Ordering** | Per-campaign ordering      | No ordering           | Per-user ordering     |
| **Load Balancing** | Uneven (popular campaigns) | Perfect balance       | Uneven                |
| **State Locality** | Excellent                  | Poor                  | Poor                  |
| **Hot Partitions** | Risk with viral campaigns  | No risk               | Risk with power users |
| **Aggregation**    | Efficient (local state)    | Inefficient (shuffle) | Wrong dimension       |

### Decision Made

**Partition by campaign_id hash**.

### Rationale

1. **Event Ordering:** All events for a campaign go to same partition (sequential)
2. **State Locality:** Flink can aggregate locally without shuffle
3. **Correctness:** Ensures accurate counts (no race conditions)
4. **Simplicity:** Flink state management simpler with co-located events
5. **Kafka Semantics:** Leverages Kafka's partition ordering guarantee

**Implementation:**

```
partition_id = hash(campaign_id) % 100
```

### Trade-offs Accepted

| What We Gain                     | What We Sacrifice                        |
|----------------------------------|------------------------------------------|
| ✅ Event ordering per campaign    | ❌ Uneven partition sizes                 |
| ✅ Local aggregation (no shuffle) | ❌ Risk of hot partitions                 |
| ✅ Simpler state management       | ❌ Manual rebalancing for viral campaigns |
| ✅ Accurate counting              | ❌ Load imbalance                         |

### When to Reconsider

- If viral campaigns cause severe hot partitions
- If load balance is more critical than ordering
- If state size per campaign exceeds memory
- If aggregation dimension changes (e.g., per-user)

---

## 8. Two-Speed Pipeline vs Single Pipeline

### The Problem

Should we use one pipeline for both real-time and billing, or separate pipelines optimized for each?

### Options Considered

| Aspect                | Two-Speed Pipeline         | Single Pipeline            |
|-----------------------|----------------------------|----------------------------|
| **Real-Time Latency** | < 5 seconds (optimized)    | ~30 seconds (compromised)  |
| **Billing Accuracy**  | 100% (batch deduplication) | 98% (stream deduplication) |
| **Complexity**        | Higher (two paths)         | Lower (one path)           |
| **Cost**              | Higher (Redis + Spark)     | Lower (stream only)        |
| **Trade-offs**        | Separate optimization      | Compromises on both        |

### Decision Made

**Two-Speed Pipeline** (fast + slow).

### Rationale

1. **Optimized for Purpose:** Real-time optimized for latency, batch for accuracy
2. **Billing Integrity:** 100% accuracy critical for revenue
3. **User Experience:** Sub-5-second dashboard updates
4. **Deduplication:** Batch can fully deduplicate (7-day window)
5. **Late Arrivals:** Batch catches all late events

**Implementation:**

- Fast path: Flink → Redis (< 5 sec lag, 98% accuracy)
- Slow path: Spark → PostgreSQL (24-hour lag, 100% accuracy)
- Reconciliation: Daily comparison to detect issues

### Trade-offs Accepted

| What We Gain                     | What We Sacrifice                        |
|----------------------------------|------------------------------------------|
| ✅ Optimized latency AND accuracy | ❌ Higher infrastructure cost             |
| ✅ Billing integrity              | ❌ Operational complexity (two pipelines) |
| ✅ Better UX (fast dashboard)     | ❌ Eventual consistency                   |
| ✅ Fault isolation                | ❌ Reconciliation overhead                |

### When to Reconsider

- If budget is severely constrained
- If 98% accuracy is acceptable for billing
- If operational simplicity is priority
- If real-time latency can relax to > 30 seconds

---

## 9. Exactly-Once vs At-Least-Once Semantics

### The Problem

What delivery semantics should we guarantee for event processing?

### Options Considered

| Semantics       | Exactly-Once         | At-Least-Once       | At-Most-Once    |
|-----------------|----------------------|---------------------|-----------------|
| **Duplicates**  | No duplicates        | Possible duplicates | No duplicates   |
| **Data Loss**   | No loss              | No loss             | Possible loss   |
| **Complexity**  | High                 | Medium              | Low             |
| **Performance** | Slower (checkpoints) | Faster              | Fastest         |
| **Use Case**    | Financial accuracy   | Logs, metrics       | Fire-and-forget |

### Decision Made

**Exactly-Once** semantics for billing accuracy.

### Rationale

1. **Billing Integrity:** Cannot charge advertisers twice for same click
2. **Financial Impact:** Duplicates = revenue loss or legal issues
3. **Flink Support:** Built-in exactly-once with checkpointing
4. **Idempotent Sinks:** Redis HINCRBY is idempotent (safe retry)
5. **Industry Standard:** Expected for financial systems

**Implementation:**

- Flink checkpointing: Every 60 seconds
- Kafka consumer offsets: Stored in checkpoint
- Two-phase commit: Kafka offsets + Flink state atomic
- Idempotent producers: Kafka enable.idempotence=true

### Trade-offs Accepted

| What We Gain            | What We Sacrifice                            |
|-------------------------|----------------------------------------------|
| ✅ No duplicate counting | ❌ Slower processing (checkpointing overhead) |
| ✅ Billing accuracy      | ❌ Higher complexity                          |
| ✅ No revenue loss       | ❌ Recovery takes ~60 seconds                 |
| ✅ Compliance-friendly   | ❌ Higher resource usage                      |

### When to Reconsider

- If billing accuracy can tolerate ~1% error
- If performance is critical (choose at-least-once)
- If cost of duplicates is low
- If using non-idempotent sinks

---

## 10. Synchronous vs Asynchronous API Response

### The Problem

Should the API wait for Kafka confirmation before responding to the client?

### Options Considered

| Approach           | Synchronous            | Asynchronous                  |
|--------------------|------------------------|-------------------------------|
| **Client Latency** | 50-100ms               | < 5ms                         |
| **Confirmation**   | Guaranteed in Kafka    | Buffered (risk of loss)       |
| **Throughput**     | 10k req/sec per server | 100k req/sec per server       |
| **Complexity**     | Simple                 | Requires buffering            |
| **Data Loss Risk** | None                   | Small (if crash before flush) |

### Decision Made

**Asynchronous** (return 202 Accepted immediately).

### Rationale

1. **Throughput:** 10x higher throughput at 500k events/sec
2. **Latency:** Sub-5ms response critical for ad networks
3. **Buffering:** Kafka producer buffers (16 KB batches)
4. **Flush Frequency:** 10ms linger = 10ms max delay
5. **Risk Mitigation:** Producer retries + Kafka replication

**Implementation:**

- Return 202 Accepted immediately
- Kafka producer buffers events (16 KB batches)
- Auto-flush every 10ms (linger.ms=10)
- Producer retries: 3 attempts with backoff
- Monitoring: Track producer buffer size

### Trade-offs Accepted

| What We Gain            | What We Sacrifice                    |
|-------------------------|--------------------------------------|
| ✅ 10x higher throughput | ❌ No immediate Kafka confirmation    |
| ✅ < 5ms API latency     | ❌ Small risk of data loss (if crash) |
| ✅ Better UX for clients | ❌ Client doesn't know if persisted   |
| ✅ Resource efficiency   | ❌ Monitoring buffer critical         |

### When to Reconsider

- If immediate confirmation is required
- If data loss risk is unacceptable
- If client needs to know persistence status
- If throughput requirements are lower (< 50k/sec)

---

## Summary Table: All Design Decisions

| Decision              | Choice Made      | Key Trade-off                            |
|-----------------------|------------------|------------------------------------------|
| **Architecture**      | Kappa            | Unified code vs Reprocessing flexibility |
| **Event Streaming**   | Kafka            | Cost & throughput vs Auto-scaling        |
| **Stream Processing** | Flink            | Latency & exactly-once vs Simplicity     |
| **Real-Time Storage** | Redis            | Sub-ms latency vs Memory cost            |
| **Cold Storage**      | Parquet          | Compression & analytics vs Write speed   |
| **Fraud Detection**   | Bloom Filter     | Memory efficiency vs False positives     |
| **Partitioning**      | Campaign ID hash | Ordering & locality vs Load balance      |
| **Pipeline**          | Two-Speed        | Optimized for purpose vs Complexity      |
| **Semantics**         | Exactly-Once     | Accuracy vs Performance                  |
| **API Response**      | Asynchronous     | Throughput vs Confirmation               |
