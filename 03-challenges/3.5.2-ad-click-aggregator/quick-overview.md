# Ad Click Aggregator - Quick Overview

## Core Concept

An ad click aggregator ingests billions of click events per day, filters fraud, aggregates metrics in real-time (clicks, unique users, cost), and provides dashboards for advertisers. It uses **Kappa Architecture** (unified stream processing) instead of Lambda (separate batch/stream).

**Key Challenge**: Process 500k events/sec with <5s real-time latency, 100% accurate billing (batch), and fraud filtering.

---

## Requirements

### Functional
- **Event Ingestion**: Accept click events from publishers
- **Real-Time Aggregation**: 5-minute, 1-hour, 1-day windows
- **Fraud Detection**: Filter bots, duplicate clicks, suspicious patterns
- **Dashboard Queries**: Real-time metrics (clicks, unique users, cost)
- **Billing**: 100% accurate invoices (batch processing)

### Non-Functional
- **Throughput**: 500k events/sec (43B events/day)
- **Latency**: <5s for real-time metrics
- **Accuracy**: 100% for billing (batch), ~98% for real-time (acceptable)
- **Durability**: Zero data loss (S3 archival)
- **Scalability**: Handle 10x traffic spikes (Black Friday)

---

## Components

### 1. **Ingestion Layer**
- **NGINX (L4 Load Balancer)**: 500k connections/sec
- **API Gateway**: Validates schema, rate limiting (1000 events/sec per IP)
- **Validation Service**: Schema validation, duplicate detection (Bloom filter)

### 2. **Streaming Backbone (Kafka)**
- **Topic**: `raw_clicks`
- **Partitions**: 100 (partitioned by campaign_id hash)
- **Replication**: 3 replicas
- **Retention**: 7 days
- **Compression**: LZ4 (3:1 ratio)

### 3. **Stream Processing (Flink)**
- **Cluster**: 50 task managers (4 vCPU, 16 GB RAM each)
- **Parallelism**: 100 (matches Kafka partitions)
- **State Backend**: RocksDB (disk-backed)
- **Checkpointing**: Every 60 seconds (exactly-once semantics)

### 4. **Real-Time Storage (Redis)**
- **Cluster**: 6 nodes (3 masters + 3 replicas)
- **Data Structure**: Redis Hash
- **Key Pattern**: `counter:{campaign_id}:{time_window}:{dimension}`
- **TTL**: 24 hours

### 5. **Cold Storage (S3 Data Lake)**
- **Format**: Parquet (columnar, 10x compression vs JSON)
- **Compression**: Snappy
- **Partitioning**: year/month/day/hour
- **Lifecycle**: Hot (30 days) → Warm (180 days) → Cold (Glacier) → Delete (3 years)

### 6. **Batch Processing (Spark)**
- **Nightly Pipeline**: Re-process last 24 hours
- **Deduplication**: By event_id
- **Advanced Fraud Detection**: ML model (XGBoost)
- **Billing Calculation**: 100% accurate invoices

### 7. **Query Service**
- **Real-Time**: Query Redis (5-minute, 1-hour windows)
- **Historical**: Query S3 via Spark SQL (batch)
- **Dashboard API**: REST API for advertisers

---

## Architecture Flow

### Ingestion Flow

```
1. Publisher → NGINX: POST /click {event}
2. NGINX → API Gateway: Route request
3. API Gateway → Validation Service: Validate schema, check duplicates (Bloom filter)
4. API Gateway → Kafka: Write to raw_clicks topic (partition by campaign_id)
5. API Gateway → Publisher: Return 202 Accepted (<1ms, async)
6. Flink → Kafka: Consume events
7. Flink → Fraud Detection: Filter bots, suspicious patterns
8. Flink → Windowed Aggregation: 5-minute tumbling windows
9. Flink → Redis: Write aggregated counters (pipeline writes, 100 commands/batch)
10. Flink → S3: Archive raw events (Parquet format)
```

### Query Flow

```
1. Advertiser → Dashboard API: GET /campaigns/{id}/metrics?window=5min
2. Dashboard API → Redis: GET counter:campaign_123:5min:US:mobile
3. Redis → Dashboard API: {clicks: 15234, unique_users: 8901, cost: 7617.00}
4. Dashboard API → Advertiser: Return metrics
```

### Batch Processing Flow (Nightly)

```
1. Spark → S3: Read Parquet files (last 24 hours)
2. Spark → Deduplication: Remove duplicate event_ids
3. Spark → Fraud Detection: ML model (XGBoost), behavioral analysis
4. Spark → Re-aggregation: Recalculate metrics with clean data
5. Spark → Billing Service: Generate invoices (100% accurate)
6. Spark → Cassandra: Store historical metrics (optional)
```

---

## Key Design Decisions

### 1. **Kappa Architecture** (vs Lambda)

**Why Kappa?**
- **Unified Codebase**: Same code for real-time and batch (reprocess from Kafka)
- **No Code Duplication**: Lambda requires separate batch/stream codebases
- **Consistency**: Same logic for real-time and batch (no discrepancies)

**Trade-off**: Harder batch reprocessing (need to replay Kafka) vs code duplication

### 2. **Kafka Partitioning by Campaign ID**

**Why?**
- All events for one campaign go to same partition
- Enables sequential processing per campaign
- Simplifies state management in Flink
- Maintains event ordering per campaign

**Trade-off**: Uneven partition sizes (popular campaigns) vs random partitioning

### 3. **Redis for Real-Time Storage** (vs InfluxDB/TimescaleDB)

**Why Redis?**
- **Sub-ms Query Latency**: <1ms for counter lookups
- **High Write Throughput**: 100k writes/sec per node
- **Simple Data Model**: Hash structure (campaign_id → metrics)

**Trade-off**: Eventual consistency, evictions vs strong consistency

### 4. **S3 for Cold Storage** (vs HDFS/Cassandra)

**Why S3?**
- **Cheap**: $0.023/GB/month (vs $0.10/GB for HDFS)
- **Durable**: 99.999999999% (11 nines)
- **Scalable**: Unlimited storage
- **Parquet Format**: 10x compression vs JSON

**Trade-off**: Slow query (seconds) vs fast query (milliseconds)

### 5. **Bloom Filter for Duplicate Detection**

**Why?**
- **Fast Lookup**: O(1) time complexity
- **Memory Efficient**: 1 bit per event (vs storing full event_id)
- **False Positives**: 0.1% (acceptable, can verify in batch)

**Trade-off**: 0.1% false positives vs 100% accuracy (requires storing all event_ids)

### 6. **Async API Response** (202 Accepted)

**Why?**
- **High Throughput**: Don't wait for Kafka write (50ms)
- **Non-Blocking**: API returns immediately (<1ms)
- **Durability**: Kafka ensures delivery (acks=all)

**Trade-off**: No immediate confirmation vs high throughput

---

## Fraud Detection

### Multi-Stage Pipeline

1. **Bloom Filter Check**: Known bot IPs/user-agents (5ms)
2. **Click Pattern Analysis**: Same IP > 100 clicks/min = suspicious (10ms)
3. **Honeypot Detection**: Clicks on hidden ads (5ms)
4. **ML Model**: Real-time scoring (risk 0-1) (50ms)

**Total Latency**: ~70ms (acceptable for async processing)

### Batch Fraud Correction

- **Advanced ML Model**: XGBoost (behavioral analysis)
- **IP Reputation Scoring**: Historical patterns
- **Re-aggregation**: Recalculate metrics with clean data
- **Billing Correction**: Adjust invoices if fraud detected

---

## Bottlenecks & Solutions

### 1. **Ingestion Bottleneck**

**Problem**: NGINX cannot handle 500k connections/sec

**Solution**:
- Use L4 load balancer (AWS NLB) instead of L7
- Connection pooling (reuse TCP connections)
- HTTP/2 multiplexing (multiple requests per connection)

### 2. **Kafka Write Throughput**

**Problem**: Kafka brokers saturate at 300k writes/sec

**Solution**:
- Add more brokers (horizontal scaling)
- Increase partitions: 100 → 200
- Use faster disks (NVMe SSDs)
- Batch producer writes (16 KB batches)

### 3. **Flink State Size**

**Problem**: 10M counters × 100 bytes = 1 GB state per worker

**Solution**:
- Use RocksDB (disk-backed state)
- Incremental checkpoints (only changed state)
- State TTL: Remove inactive campaigns after 7 days
- Compress state with Snappy

### 4. **Query Performance**

**Problem**: Dashboard queries slow (>1 second)

**Solution**:
- Pre-aggregate in Flink (don't query raw Redis keys)
- Use Redis pipelining (batch 100 GET commands)
- Cache popular queries in CDN (1-minute TTL)
- Use materialized views for common breakdowns

---

## Common Anti-Patterns

### ❌ **1. Synchronous API Response**

**Problem**: API waits for Kafka write confirmation before responding

**Bad**:
```
POST /click → Write to Kafka (50ms) → Return 200 OK
Total latency: 50ms+ per request
```

**Good**:
```
POST /click → Buffer in memory → Return 202 Accepted (<1ms)
Async: Flush buffer to Kafka every 10ms
```

### ❌ **2. Real-Time Billing**

**Problem**: Using real-time counts for billing invoices

**Why It's Wrong**:
- Real-time has ~2% error rate (duplicates, fraud)
- Late arrivals not included
- Redis evictions cause missing data

**Solution**: Use batch processing for billing (source of truth)

### ❌ **3. No Fraud Filtering**

**Problem**: Count all clicks blindly

**Impact**:
- 10-30% of clicks are fraud
- Advertisers overpay
- Legal liability

**Solution**: Multi-stage fraud detection (Bloom filter + ML + batch analysis)

### ❌ **4. Lambda Architecture**

**Problem**: Separate batch and stream codebases

**Why It's Wrong**:
- Code duplication
- Hard to maintain consistency
- Bugs in one path not in other

**Solution**: Kappa architecture (unified stream processing)

---

## Monitoring & Observability

### Key Metrics

- **Ingestion Rate**: 500k events/sec
- **Flink Processing Lag**: <5 seconds
- **Redis Memory Usage**: <80%
- **Kafka Consumer Lag**: <1000 events
- **Fraud Detection Rate**: ~10% (suspicious events)
- **Query Latency P99**: <100ms (Redis queries)

### Alerts

- **Ingestion Rate** <400k events/sec (5-minute window)
- **Flink Processing Lag** >10 seconds
- **Redis Memory Usage** >90%
- **Kafka Consumer Lag** >10k events
- **Fraud Detection Rate** >20% (potential attack)

---

## Trade-offs Summary

| Decision | What We Gain | What We Sacrifice |
|----------|--------------|-------------------|
| **Kappa Architecture** | ✅ Unified codebase | ❌ Harder batch reprocessing |
| **Kafka Partitioning by Campaign** | ✅ Event ordering per campaign | ❌ Uneven partition sizes |
| **Redis for Real-Time** | ✅ Sub-ms query latency | ❌ Eventual consistency, evictions |
| **S3 for Cold Storage** | ✅ Cheap, durable | ❌ Slow query (seconds) |
| **Bloom Filter Fraud** | ✅ Fast lookup (O(1)) | ❌ 0.1% false positives |
| **Async API Response** | ✅ High throughput | ❌ No immediate confirmation |
| **Batch for Billing** | ✅ 100% accuracy | ❌ 24-hour lag |
| **Flink Checkpointing** | ✅ Exactly-once semantics | ❌ 60-second recovery window |

---

## Real-World Examples

### Google Ads
- **Architecture**: Dremel (Presto-like) for ad-hoc queries, Flume for ingestion, Bigtable for real-time counters, MapReduce for batch billing
- **Scale**: 100+ billion clicks/day, 10 PB/day ingestion

### Facebook Ads
- **Architecture**: Scribe for event ingestion, Puma for real-time aggregation, Hive for batch processing, Scuba for exploratory analytics
- **Innovation**: Custom ML models for fraud detection, 3-tier storage (hot/warm/cold)

### Amazon Advertising
- **Architecture**: Kinesis for event streaming, Redshift for analytics, S3 for archival
- **Scale**: 1B+ clicks/day

---

## Key Takeaways

1. **Kappa Architecture**: Unified stream processing (reprocess from Kafka for batch)
2. **Partitioning Strategy**: Partition by campaign_id (maintains ordering, simplifies state)
3. **Real-Time vs Batch**: Real-time for dashboards (~98% accuracy), batch for billing (100% accuracy)
4. **Fraud Detection**: Multi-stage pipeline (Bloom filter + ML + batch correction)
5. **Storage Tiers**: Redis (real-time), S3 (cold storage), Spark (batch processing)
6. **Async Ingestion**: 202 Accepted response (don't wait for Kafka write)
7. **State Management**: RocksDB for large state (10M+ counters)
8. **Query Optimization**: Pre-aggregation, pipelining, CDN caching

---

## Recommended Stack

- **Ingestion**: NGINX (L4), Go/Rust API Gateway
- **Streaming**: Kafka (100 partitions, 3 replicas)
- **Stream Processing**: Apache Flink (RocksDB state backend)
- **Real-Time Storage**: Redis Cluster (6 nodes)
- **Cold Storage**: S3 (Parquet format, Snappy compression)
- **Batch Processing**: Apache Spark (nightly pipeline)
- **Query Service**: Go/Java REST API
- **Monitoring**: Prometheus, Grafana, ELK Stack

