# 3.5.2 Design Ad Click Aggregator (Low-Latency Analytics)

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

Design a highly scalable, low-latency ad click aggregation system that can ingest billions of click events per day,
provide real-time analytics for campaign dashboards, filter fraudulent clicks, and generate accurate billing reports.
The system must handle massive write throughput (500k events/sec), maintain sub-second dashboard latency, ensure no data
loss for billing, and operate cost-efficiently at petabyte scale.

---

## 1. Requirements and Scale Estimation

### Functional Requirements

| Requirement                  | Description                                     | Priority  |
|------------------------------|-------------------------------------------------|-----------|
| **Click Ingestion**          | Ingest billions of ad click events per day      | Must Have |
| **Real-Time Counting**       | Provide near real-time (seconds lag) counts     | Must Have |
| **Fraud Filtering**          | Filter bot traffic and fraudulent clicks        | Must Have |
| **Final Reporting**          | Accurate counts for billing and analysis        | Must Have |
| **Time-Window Aggregations** | Support queries like "clicks in last 5 minutes" | Must Have |

### Non-Functional Requirements

| Requirement               | Target                | Rationale                      |
|---------------------------|-----------------------|--------------------------------|
| **High Write Throughput** | 500k events/sec peak  | Extremely write-heavy workload |
| **Durability**            | 99.999%               | Critical for billing           |
| **Cost Efficiency**       | < $0.01 per 1M events | Massive volume                 |
| **Low Latency**           | < 5 seconds lag       | Real-time dashboards           |

### Scale Estimation

- **Peak Writes:** 500,000 events/sec
- **Daily Events:** 43.2 billion events/day
- **Annual Storage:** ~15 PB of raw logs
- **Active Campaigns:** ~100,000 concurrent campaigns
- **Aggregation State:** 10 million counters in memory

---

## 2. High-Level Architecture

**Kappa Architecture** (stream-only processing) with **two-speed pipeline**:

- **Fast Path:** Real-time dashboard (< 5 sec lag) via Redis
- **Slow Path:** Accurate billing (24-hour lag) via Spark batch

### Key Components

1. **Ingestion Layer:** NGINX + API Gateway (stateless)
2. **Streaming Backbone:** Kafka (100 partitions, 7-day retention)
3. **Stream Processing:** Apache Flink (50 workers)
4. **Real-Time Storage:** Redis Cluster (sub-millisecond queries)
5. **Cold Storage:** S3 Data Lake (Parquet, compressed)
6. **Batch Processing:** Apache Spark (nightly reconciliation)
7. **Billing Database:** PostgreSQL (ACID-compliant)

---

## 3. Detailed Component Design

### Ingestion Layer

**API Gateway:**

- Lightweight HTTP server (Go/Rust)
- Returns 202 Accepted immediately (async)
- Rate limiting: 1000 events/sec per IP
- Schema validation and enrichment

**Event Schema:**

```json
{
  "event_id": "uuid",
  "timestamp": "ISO 8601",
  "campaign_id": "campaign_123",
  "ad_id": "ad_456",
  "user_id": "user_hash",
  "ip_address": "1.2.3.4",
  "device_type": "mobile",
  "country": "US",
  "cost_per_click": 0.50
}
```

### Streaming Backbone (Kafka)

**Configuration:**

- Topic: `raw_clicks`
- Partitions: 100 (by campaign_id hash)
- Replication: 3
- Retention: 7 days
- Compression: LZ4 (3:1 ratio)

**Why Partition by Campaign ID:**

- Maintains event ordering per campaign
- Simplifies state management
- Enables efficient aggregation

### Stream Processing (Flink)

**Processing Pipeline:**

1. Fraud Detection Filter
2. Windowed Aggregation (5-min tumbling windows)
3. Redis Sink (real-time counters)
4. S3 Sink (archival)

**Fraud Detection:**

- Bloom filter for known bad actors
- Click pattern analysis (> 100 clicks/min = suspicious)
- ML model scoring (risk 0-1)

**State Management:**

- RocksDB for large state (10M counters)
- Checkpointing every 60 seconds
- State TTL: 7 days

### Real-Time Storage (Redis)

**Data Structures:**

```
Key: counter:{campaign_id}:{window}:{dimension}
Value: {"clicks": 15234, "unique_users": 8901, "cost": 7617.00}
TTL: 24 hours
```

**Performance:**

- 100k writes/sec per node
- < 1ms per write
- Pipeline writes: 100 commands per batch

### Cold Storage (S3)

**Structure:**

```
s3://ad-clicks/raw/year=2025/month=10/day=31/hour=12/
```

**Format:**

- Parquet columnar (10x compression)
- File size: 128 MB (optimal for Spark)
- Lifecycle: Standard → IA → Glacier

### Batch Processing (Spark)

**Nightly Pipeline:**

1. Read S3 Parquet files
2. Deduplication (by event_id)
3. Fraud removal (ML model)
4. Join with campaign metadata
5. Aggregate by campaign/day
6. Write to PostgreSQL

---

## 4. Fraud Detection and Prevention

### Real-Time Detection

**Bloom Filter:**

- 100 million entries
- 0.1% false positive rate
- Memory: ~150 MB

**Click Patterns:**

- High frequency: > 100 clicks/min
- Sequential clicks: > 10 clicks/sec
- Impossible geo: IP mismatch

**ML Model:**

- Random Forest classifier
- Inference: < 5ms
- Risk score: 0.0-1.0

---

## 5. Availability and Fault Tolerance

| Component | Impact if Failed | Mitigation                        |
|-----------|------------------|-----------------------------------|
| **Kafka** | No ingestion     | 3-broker replication, cross-AZ    |
| **Flink** | Real-time lag    | Checkpointing, auto-restart       |
| **Redis** | Dashboard down   | Master-replica, Sentinel failover |

**Disaster Recovery:**

- Kafka: Mirror Maker 2 for cross-region replication
- Flink: Checkpoints to S3 every 60 seconds
- Redis: RDB snapshots + AOF every second

---

## 6. Bottlenecks and Optimizations

**Ingestion Bottleneck:**

- Problem: 500k connections/sec
- Solution: L4 load balancer, HTTP/2 multiplexing

**Kafka Throughput:**

- Problem: Broker saturation
- Solution: Add brokers, increase partitions, NVMe SSDs

**Flink State Size:**

- Problem: 1 GB state per worker
- Solution: RocksDB disk-backed state, incremental checkpoints

---

## 7. Common Anti-Patterns

### ❌ Synchronous API Response

**Bad:** Wait for Kafka confirmation (50ms latency)
**Good:** Return 202 Accepted immediately, async flush

### ❌ Real-Time Billing

**Bad:** Use real-time counts for invoices
**Good:** Use batch processing for billing accuracy

### ❌ No Fraud Filtering

**Bad:** Count all clicks blindly
**Good:** Multi-stage fraud detection

---

## 8. Alternative Approaches

**Lambda Architecture:**

- Separate batch and stream layers
- Pros: Easy reprocessing
- Cons: Code duplication
- Decision: Kappa chosen for unified codebase

**Kinesis vs Kafka:**

- Kinesis: Managed, auto-scaling
- Kafka: Lower cost, higher throughput
- Decision: Kafka chosen for cost and scale

---

## 9. Monitoring and Observability

| Metric                 | Target     | Alert Threshold |
|------------------------|------------|-----------------|
| **Ingestion Rate**     | 500k/sec   | < 400k/sec      |
| **Kafka Lag**          | < 1 minute | > 5 minutes     |
| **Flink Backpressure** | 0%         | > 50%           |
| **Fraud Rate**         | 10-20%     | > 30% or < 5%   |

---

## 10. Trade-offs Summary

| Decision               | What We Gain       | What We Sacrifice           |
|------------------------|--------------------|-----------------------------|
| **Kappa Architecture** | ✅ Unified codebase | ❌ Harder batch reprocessing |
| **Redis Real-Time**    | ✅ Sub-ms latency   | ❌ Eventual consistency      |
| **Batch for Billing**  | ✅ 100% accuracy    | ❌ 24-hour lag               |
| **Bloom Filter**       | ✅ Fast lookup      | ❌ 0.1% false positives      |

---

## 11. Real-World Examples

**Google Ads:**

- 100+ billion clicks/day
- Dremel for queries, Bigtable for counters

**Facebook Ads:**

- Scribe for ingestion, Puma for aggregation
- Custom ML for fraud detection

---

## 12. References

- **[2.3.2 Apache Kafka](../../02-components/2.3-messaging-streaming/2.3.2-apache-kafka-deep-dive.md)**
- **[2.3.8 Apache Flink](../../02-components/2.3-messaging-streaming/2.3.8-apache-flink-deep-dive.md)**
- **[2.2.1 Redis](../../02-components/2.2-caching/2.2.1-redis-deep-dive.md)**
- **[2.3.7 Apache Spark](../../02-components/2.3-messaging-streaming/2.3.7-apache-spark-deep-dive.md)**
- **[2.5.4 Bloom Filters](../../02-components/2.5-algorithms/2.5.4-bloom-filters.md)**

---

## 13. Deployment and Infrastructure

### Kubernetes Architecture

**Cluster Configuration:**

- 3 Kubernetes clusters (US-East, EU-West, AP-Southeast)
- Each cluster: 100-200 nodes (c5.4xlarge instances)
- Auto-scaling: HPA for API Gateway, Flink, Spark

**Namespaces:**

- `ingestion`: API Gateway, NGINX
- `streaming`: Kafka, Flink workers
- `storage`: Redis, PostgreSQL
- `batch`: Spark executors
- `monitoring`: Prometheus, Grafana

### Infrastructure as Code

**Terraform Modules:**

```
terraform/
├── vpc/              # Network configuration
├── eks/              # Kubernetes clusters
├── msk/              # Managed Kafka
├── rds/              # PostgreSQL RDS
├── elasticache/      # Redis clusters
└── s3/               # Data lake buckets
```

**Key Resources:**

- VPC with 3 AZs per region
- Private subnets for data processing
- Public subnets for API Gateway
- NAT gateways for outbound traffic

### Multi-Region Strategy

**Regional Deployment:**

| Region       | Role      | Components                 | Latency |
|--------------|-----------|----------------------------|---------|
| US-East-1    | Primary   | Full stack + batch         | < 20ms  |
| EU-West-1    | Secondary | Ingestion + real-time only | < 50ms  |
| AP-Southeast | Secondary | Ingestion + real-time only | < 50ms  |

**Failover Process:**

1. Health check detects primary region failure
2. Route 53 switches DNS to secondary region
3. Kafka MirrorMaker continues replication
4. Batch processing paused until primary recovered

### CI/CD Pipeline

**Build Pipeline:**

```
Code Push → GitHub Actions → Docker Build → ECR Push → ArgoCD Deploy
```

**Deployment Strategy:**

- Canary releases (10% → 50% → 100%)
- Blue-green deployment for critical services
- Automated rollback on error rate > 1%

---

## 14. Advanced Features

### Anomaly Detection

**Real-Time Anomaly Detection:**

- Detects sudden spikes in click volume
- Z-score analysis (> 3 standard deviations)
- Auto-alerts on campaign anomalies
- ML-based seasonality adjustment

**Implementation:**

- Flink CEP (Complex Event Processing)
- Rolling window statistics (last 24 hours)
- Per-campaign baselines
- Alert thresholds: 5x normal volume

### Cost Attribution

**Multi-Touch Attribution:**

- First-click attribution
- Last-click attribution
- Linear attribution
- Time-decay attribution
- Data-driven attribution (ML)

**Implementation:**

- User journey tracking (cookie-based)
- Attribution window: 30 days
- Stored in separate Cassandra cluster
- Batch processing for attribution modeling

### Click Fraud Prediction

**Advanced ML Models:**

- XGBoost for click fraud prediction
- Feature engineering (200+ features)
- Model retraining: Weekly
- A/B testing for model improvements

**Features:**

- Click velocity patterns
- IP reputation scores
- Device fingerprinting
- Behavioral biometrics
- Historical fraud rate per advertiser

### Real-Time Bidding Integration

**RTB Integration:**

- Expose real-time click data to DSPs
- Sub-100ms API response for bid decisions
- Rate limiting per DSP (10k QPS)
- Caching of popular campaign data

---

## 15. Performance Optimization

### Kafka Optimization

**Producer Tuning:**

```
linger.ms=10                    # Batch writes
batch.size=16384                # 16 KB batches
compression.type=lz4            # Fast compression
acks=1                          # Async replication
buffer.memory=134217728         # 128 MB buffer
max.in.flight.requests=5        # Pipeline requests
```

**Broker Tuning:**

```
num.network.threads=8           # Network I/O threads
num.io.threads=16               # Disk I/O threads
socket.send.buffer.bytes=102400 # 100 KB send buffer
socket.receive.buffer.bytes=102400
log.segment.bytes=134217728     # 128 MB segments
```

**Consumer Tuning:**

```
fetch.min.bytes=1024            # Minimum fetch size
fetch.max.wait.ms=100           # Max wait time
max.poll.records=1000           # Batch size
```

### Flink Optimization

**Memory Configuration:**

```
taskmanager.memory.process.size=16g
taskmanager.memory.flink.size=12g
taskmanager.memory.managed.size=8g
taskmanager.memory.network.min=1g
```

**Checkpointing Tuning:**

```
execution.checkpointing.interval=60s
execution.checkpointing.mode=EXACTLY_ONCE
state.backend=rocksdb
state.backend.incremental=true
state.backend.async=true
```

**Parallelism:**

```
parallelism.default=100
taskmanager.numberOfTaskSlots=4
```

### Redis Optimization

**Memory Configuration:**

```
maxmemory 64gb
maxmemory-policy allkeys-lru
maxmemory-samples 5
```

**Persistence:**

```
save ""                         # Disable RDB (fast writes)
appendonly yes                  # Enable AOF
appendfsync everysec           # AOF sync frequency
```

**Networking:**

```
tcp-backlog 511
tcp-keepalive 300
timeout 0
```

### S3 Optimization

**Upload Strategy:**

- Multipart uploads (10 MB parts)
- S3 Transfer Acceleration (cross-region)
- Parallel uploads (10 concurrent)

**Lifecycle Policies:**

```
0-30 days:   S3 Standard
31-90 days:  S3 Intelligent-Tiering
91-365 days: S3 Glacier
366+ days:   S3 Deep Archive
```

### Spark Optimization

**Executor Configuration:**

```
spark.executor.instances=100
spark.executor.cores=4
spark.executor.memory=16g
spark.executor.memoryOverhead=2g
spark.driver.memory=8g
```

**Shuffle Optimization:**

```
spark.sql.shuffle.partitions=400
spark.shuffle.service.enabled=true
spark.shuffle.compress=true
spark.io.compression.codec=lz4
```

**Caching:**

```
spark.sql.autoBroadcastJoinThreshold=10485760  # 10 MB
spark.sql.adaptive.enabled=true
spark.sql.adaptive.coalescePartitions.enabled=true
```

---

## 16. Data Quality and Validation

### Input Validation

**Schema Validation:**

- JSON schema validation at API Gateway
- Required fields: event_id, timestamp, campaign_id
- Data type validation (integers, floats, strings)
- Range validation (timestamps within ±5 minutes)

**Duplicate Detection:**

- Event ID uniqueness check
- Bloom filter for recent events (last 1 hour)
- Exact deduplication in batch processing

### Data Quality Metrics

**Quality Dimensions:**

| Dimension        | Metric                      | Target | Alert Threshold |
|------------------|-----------------------------|--------|-----------------|
| **Completeness** | % events with all fields    | 99.9%  | < 99%           |
| **Accuracy**     | % events passing validation | 99.5%  | < 98%           |
| **Timeliness**   | % events within 5 min       | 99%    | < 95%           |
| **Uniqueness**   | Duplicate rate              | < 0.1% | > 1%            |

### Data Lineage

**Tracking:**

- Event ingestion timestamp
- Processing timestamps (Flink, Spark)
- Data transformation steps
- Final storage location

**Auditing:**

- Full event history in S3
- Replay capability from Kafka
- Immutable append-only logs

---

## 17. Security and Compliance

### Encryption

**Data at Rest:**

- S3: AES-256 encryption (SSE-S3)
- Kafka: Volume encryption (AWS EBS)
- PostgreSQL: Transparent Data Encryption (TDE)
- Redis: Volume encryption

**Data in Transit:**

- TLS 1.3 for all API endpoints
- Kafka inter-broker TLS
- Redis TLS connections
- PostgreSQL SSL connections

### Access Control

**Authentication:**

- API Gateway: JWT tokens
- Kafka: SASL/SCRAM
- PostgreSQL: IAM authentication
- Redis: AUTH password

**Authorization:**

- Role-Based Access Control (RBAC)
- Least privilege principle
- Service accounts per component
- Regular credential rotation

### Privacy Compliance

**GDPR Compliance:**

- User consent tracking
- Right to deletion (data purging)
- Data retention policies (3 years)
- Data minimization

**CCPA Compliance:**

- Consumer data access
- Opt-out mechanisms
- Data sale disclosure

### Audit Logging

**Events Logged:**

- All API requests
- Data access patterns
- Configuration changes
- Security events

**Retention:**

- Security logs: 7 years
- Access logs: 1 year
- Application logs: 90 days

---

## 18. Capacity Planning

### Growth Projections

**Year-over-Year Growth:**

| Metric            | Current | Year 1 | Year 2  | Year 3  |
|-------------------|---------|--------|---------|---------|
| **Events/day**    | 43.2 B  | 86.4 B | 172.8 B | 345.6 B |
| **Storage**       | 15 PB   | 30 PB  | 60 PB   | 120 PB  |
| **Kafka brokers** | 10      | 20     | 40      | 80      |
| **Flink workers** | 50      | 100    | 200     | 400     |
| **Cost/month**    | $100k   | $200k  | $400k   | $800k   |

### Resource Allocation

**Cost Breakdown:**

- Storage (S3): 40% ($40k/month)
- Compute (EC2): 35% ($35k/month)
- Kafka (MSK): 15% ($15k/month)
- Database (RDS): 5% ($5k/month)
- Networking: 5% ($5k/month)

**Scaling Triggers:**

- Kafka lag > 5 minutes → Add brokers
- Flink backpressure > 50% → Add workers
- Redis memory > 80% → Add nodes
- API latency > 100ms → Add gateways

---

## 19. Interview Discussion Points

### Common Interview Questions

**Q1: Why Kappa over Lambda architecture?**

**Answer:**

- Unified codebase reduces complexity
- Same transformation logic for real-time and batch
- Modern tools (Flink) handle both well
- Easier to maintain consistency
- Trade-off: Limited reprocessing window (Kafka retention)

**Q2: How do you handle exactly-once semantics?**

**Answer:**

- Flink checkpointing with Kafka offsets
- Idempotent Kafka producers
- Two-phase commit for state + offsets
- Trade-off: 60-second recovery time
- Critical for billing accuracy

**Q3: How do you prevent hot partitions?**

**Answer:**

- Partition by campaign_id hash (uniform distribution)
- Monitor partition sizes
- Manual repartitioning for viral campaigns
- Consider consistent hashing for better balance

**Q4: What happens if Redis fails?**

**Answer:**

- Dashboard shows stale data or errors
- Real-time analytics unavailable
- Billing unaffected (batch path)
- Sentinel auto-failover (< 30 sec)
- Fallback to PostgreSQL (slower queries)

**Q5: How do you handle late arrivals?**

**Answer:**

- Flink watermarks (5-minute grace period)
- Batch reprocessing catches all late events
- Two-speed pipeline design
- Real-time: 98% accuracy
- Batch: 100% accuracy (source of truth)

### Scalability Discussions

**Scaling to 10M events/sec:**

1. **Kafka:** 100 → 200 brokers, 200 → 400 partitions
2. **Flink:** 50 → 500 workers, distributed state store (Cassandra)
3. **Redis:** 6 → 60 nodes, Redis Cluster sharding
4. **S3:** No changes (scales infinitely)
5. **Cost:** $100k → $1M/month

**Bottlenecks at Scale:**

- Flink state size (10M → 100M counters)
- Redis memory (384 GB → 3.8 TB)
- Kafka replication lag (cross-region)
- Spark job duration (2 hours → 20 hours)

### Design Trade-Offs

**What Would You Change?**

**At 10x scale (5M events/sec):**

- Switch to ClickHouse for real-time analytics (columnar, SQL)
- Use Cassandra for distributed counters (better write scaling)
- Implement custom partitioning (consistent hashing)
- Pre-aggregate in Flink (reduce Redis writes)

**At 100x scale (50M events/sec):**

- Abandon Redis (too expensive)
- ClickHouse + Materialized Views
- Custom C++ stream processor (Flink overhead)
- Multi-tier storage (hot/warm/cold)

**If cost was no object:**

- AWS Kinesis Data Analytics (fully managed)
- Amazon Timestream (time-series database)
- Real-time ML inference (GPU instances)
- Global active-active multi-master

---

## 20. Lessons Learned

### What Works Well

**Stream Processing:**

- Flink's exactly-once semantics are reliable
- RocksDB state backend scales well
- Checkpointing to S3 is fast and durable

**Two-Speed Pipeline:**

- Best of both worlds (latency + accuracy)
- Allows real-time UX without compromising billing
- Reconciliation catches discrepancies early

**Kafka:**

- Extremely reliable at scale
- Low-latency replication
- Easy to operate (MSK)

### Common Pitfalls

**State Size Explosion:**

- Problem: Flink state grew to 10 GB per worker
- Solution: Aggressive TTL, state compaction
- Lesson: Monitor state size closely

**Redis Evictions:**

- Problem: LRU evictions caused missing data
- Solution: Increased memory, shorter TTL
- Lesson: Size Redis for peak load + 20%

**Kafka Rebalancing:**

- Problem: Consumer rebalances caused lag spikes
- Solution: Static membership, longer session timeout
- Lesson: Avoid frequent worker restarts

### Best Practices

1. **Always use exactly-once semantics** for financial data
2. **Monitor Kafka lag religiously** (alert on > 1 min)
3. **Test failover regularly** (chaos engineering)
4. **Optimize for 99th percentile**, not average
5. **Cache aggressively**, but invalidate correctly
6. **Use incremental checkpoints** (10x faster recovery)
7. **Partition by business key** (campaign_id, not random)
8. **Separate real-time and batch** (different SLAs)
9. **Design for eventual consistency** in real-time path
10. **Keep batch as source of truth** for billing

### Production Readiness Checklist

**Before Going Live:**

- [ ] Load testing completed (2x expected peak)
- [ ] Chaos engineering tests passed
- [ ] Disaster recovery procedures documented
- [ ] Runbooks created for common issues
- [ ] On-call rotation established
- [ ] Monitoring dashboards configured
- [ ] Alert thresholds validated
- [ ] Security audit completed
- [ ] Data retention policies implemented
- [ ] Cost projections validated

**Operational Metrics to Track:**

- Ingestion rate (events/sec)
- End-to-end latency (P50, P95, P99)
- Kafka consumer lag
- Flink backpressure
- Redis memory usage
- Fraud detection rate
- Real-time vs batch discrepancy
- Error rates per component
