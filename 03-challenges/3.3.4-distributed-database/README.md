# 3.3.4 Design a Distributed Database

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions. 
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, Raft consensus, sharding strategies
- **[Sequence Diagrams](./sequence-diagrams.md)** - Write paths, read paths, failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - Why Raft vs Paxos, 2PC vs Sagas, range vs hash sharding
- **[Pseudocode Implementations](./pseudocode.md)** - Raft consensus, 2PC coordinator, range splitting algorithms

---

## Problem Statement

Design a **globally distributed SQL database** that provides:

- **Strong Consistency:** ACID transactions across multiple datacenters
- **Horizontal Scalability:** Add nodes to increase capacity linearly
- **High Availability:** Survive datacenter failures with automatic failover
- **SQL Compatibility:** Standard SQL interface for application compatibility

**Real-World Systems:** Google Spanner, CockroachDB, YugabyteDB, TiDB

**The Core Trade-off:** Traditional databases choose between consistency OR scalability. We need BOTH.

---

## Requirements

### Functional Requirements

1. **Automatic Sharding:** Distribute data across nodes without manual partitioning
2. **Multi-Region Replication:** Data replicated across geographic regions (US, EU, APAC)
3. **ACID Transactions:** Support multi-row, multi-shard transactions
4. **Schema Evolution:** Add/remove columns without downtime
5. **Fault Tolerance:** Automatic leader election when nodes fail

### Non-Functional Requirements

| Requirement | Target | Why |
|------------|--------|-----|
| **Availability** | 99.99% | Tolerate node/datacenter failures |
| **Write Latency** | <50ms (single region), <200ms (multi-region) | Consensus quorum overhead |
| **Read Latency** | <10ms (local), <100ms (cross-region) | Read from nearest replica |
| **Throughput** | 1M QPS (100K writes, 900K reads) | Large-scale applications |
| **Scale** | 1000+ nodes, 100+ TB per node | Petabyte-scale |

### Scale Estimation

| Metric | Calculation | Result |
|--------|-------------|--------|
| **Total Data** | 100 TB original | 100 TB |
| **Replicated Data** | 100 TB × 3 replicas | 300 TB storage |
| **Nodes Required** | 300 TB / 10 TB per node | 30 nodes minimum |
| **Read QPS per Node** | 900K reads / 30 nodes | 30K reads/sec |
| **Write QPS per Node** | 100K writes / 30 nodes | 10K writes/sec |

---

## High-Level Architecture

### Component Overview

The system consists of four main layers:

1. **Client Layer:** Applications connect via standard SQL drivers
2. **Gateway/Router Layer:** Routes queries to correct shard based on key
3. **Consensus Groups (Raft):** Collections of nodes that form quorums
4. **Storage Layer:** LSM tree (RocksDB) with WAL for durability

```
┌──────────────────────────────────────────┐
│       CLIENT APPLICATIONS                 │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│       SQL GATEWAY (Stateless)             │
│  - Parse SQL                              │
│  - Consult metadata for routing           │
│  - Direct to correct Raft Group           │
└────────────────┬─────────────────────────┘
                 │
      ┌──────────┼──────────┐
      │          │          │
      ▼          ▼          ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Range 1 │ │ Range 2 │ │ Range 3 │
│ [A-F]   │ │ [G-M]   │ │ [N-Z]   │
├─────────┤ ├─────────┤ ├─────────┤
│ Leader  │ │ Leader  │ │ Leader  │
│ Node 1  │ │ Node 4  │ │ Node 7  │
├─────────┤ ├─────────┤ ├─────────┤
│Follow N2│ │Follow N5│ │Follow N8│
│Follow N3│ │Follow N6│ │Follow N9│
└─────────┘ └─────────┘ └─────────┘
   3 Replicas  3 Replicas  3 Replicas
   (Quorum=2)  (Quorum=2)  (Quorum=2)
```

**Key Insight:** Each range (shard) has its own Raft consensus group. Writes require quorum (2 out of 3 nodes).

---

## Data Model and Sharding

### Range-Based Sharding

Data partitioned by **key ranges**, not hash:

```
Table: users (primary key: user_id UUID)

Shard 1: [00000000...55555555] → Raft Group A (Nodes 1,2,3)
Shard 2: [55555556...AAAAAAAA] → Raft Group B (Nodes 4,5,6)
Shard 3: [AAAAAAAB...FFFFFFFF] → Raft Group C (Nodes 7,8,9)
```

**Why Range-Based Over Hash?**

| Factor | Range-Based | Hash-Based |
|--------|-------------|------------|
| **Range Queries** | ✅ Efficient (single shard) | ❌ Scatter-gather (all shards) |
| **Hotspots** | ⚠️ Possible (recent data) | ✅ Even distribution |
| **Split/Merge** | ✅ Simple (split at midpoint) | ❌ Complex (rehash all keys) |

**Use Range-Based for:**
- Time-series data (recent data frequently accessed)
- Applications with range queries (`WHERE timestamp BETWEEN X AND Y`)

**Metadata Schema:**

```sql
CREATE TABLE range_metadata (
    range_id BIGINT PRIMARY KEY,
    start_key BYTES NOT NULL,
    end_key BYTES NOT NULL,
    leader_node_id INT NOT NULL,
    replica_node_ids INT[] NOT NULL,
    version INT NOT NULL
);
```

### Dynamic Range Splitting

When range becomes too large (>64 MB) or too hot (>10K QPS):

```
Before: Range [A-Z] (100 MB, 50K QPS) → Overloaded

After:
  Range [A-M] (50 MB, 25K QPS) → Raft Group A (Nodes 1,2,3)
  Range [N-Z] (50 MB, 25K QPS) → Raft Group B (Nodes 4,5,6)
```

*See [pseudocode.md::split_range()](pseudocode.md) for implementation.*

---

## Consensus and Replication (Raft)

### Why Raft?

**Raft vs Paxos:**

| Feature | Raft | Paxos |
|---------|------|-------|
| **Understandability** | ✅ Easier to learn/implement | ❌ Notoriously complex |
| **Leader** | ✅ Strong leader (simplifies) | ⚠️ Can have multiple leaders |
| **Implementations** | ✅ Many (etcd, CockroachDB) | ⚠️ Few correct implementations |

### Raft Guarantees

1. **Leader Election:** New leader elected within ~1 second if leader fails
2. **Log Replication:** All committed writes durably replicated
3. **Safety:** Once committed (quorum ACKs), never lost
4. **Linearizability:** Reads/writes appear atomic

### Write Flow

```
Client → Leader: Write(key=X, value=Y)

Leader:
  1. Append to local log (uncommitted)
  2. Send AppendEntries to Followers
  
Followers:
  1. Append to local log
  2. ACK to Leader
  
Leader (receives 2 out of 3 ACKs):
  1. Mark as "committed"
  2. Apply to RocksDB
  3. Return success to Client
```

**Latency:**
- **Single Region:** 1 RTT = 1-5ms (Leader → Follower → Leader)
- **Multi-Region:** 1 RTT = 50-200ms (cross-region network)

*See [sequence-diagrams.md](./sequence-diagrams.md) for detailed flows.*

---

## Transaction Management (Distributed ACID)

### The Problem

Transaction spans multiple shards:

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 50 WHERE id = A;  -- Shard 1
  UPDATE accounts SET balance = balance + 50 WHERE id = B;  -- Shard 2
COMMIT;
```

**Without coordination:** Partial updates visible (user sees $50 disappeared).

### Solution: Two-Phase Commit (2PC)

**Phase 1 (PREPARE):**

```
Coordinator → Shard 1: "Can you commit?"
Coordinator → Shard 2: "Can you commit?"

Shard 1: Lock row A, check constraints → "YES"
Shard 2: Lock row B, check constraints → "YES"
```

**Phase 2 (COMMIT):**

```
Coordinator: All said YES → Send "COMMIT" to both

Shard 1: Apply change, release lock → ACK
Shard 2: Apply change, release lock → ACK

Coordinator: Return SUCCESS to client
```

**Failure Handling:**

| Scenario | Action |
|----------|--------|
| **Any shard replies NO** | Coordinator sends ABORT to all |
| **Shard crashes after YES** | Coordinator retries COMMIT (idempotent) |
| **Coordinator crashes** | Participant shards timeout, abort (conservative) |

### Timestamp Oracle

**Problem:** Need global transaction ordering.

**Solution:** Centralized timestamp service (HA cluster using Raft):

```
Transaction Start:
  Client → Timestamp Oracle: "Give me a timestamp"
  Oracle: Return unique timestamp (e.g., 1640000000001)
  
All transaction writes tagged with this timestamp
Used for MVCC (Multi-Version Concurrency Control)
```

**Google Spanner's TrueTime:**
- Atomic clocks + GPS for global clock sync
- Uncertainty bound: <7ms
- Waits out uncertainty before committing ("commit wait")

*See [pseudocode.md::two_phase_commit()](pseudocode.md) for implementation.*

---

## Read Path Optimization

### Read Strategies

| Strategy | Consistency | Latency | Use Case |
|----------|------------|---------|----------|
| **Read from Leader** | ✅ Strong | ⚠️ Higher | Financial transactions |
| **Read from Follower** | ⚠️ Stale | ✅ Lower | Analytics |
| **Read with Timestamp** | ✅ Snapshot isolation | ✅ Lower | Most reads |

### Follower Reads with Timestamp

```sql
SELECT * FROM users WHERE id = X AS OF SYSTEM TIME '1 min ago';
```

**Benefit:** Read from nearest replica (10ms) instead of leader (100ms cross-region).

**How it Works:**

1. Client specifies timestamp (T-1min)
2. Gateway routes to nearest replica
3. Replica checks: "Do I have data for T-1min?"
4. If YES: Return data immediately
5. If NO: Wait for replication to catch up

**Result:** 90% of reads served locally, 10× latency improvement.

---

## Metadata Management and Failure Recovery

### Metadata Store (etcd)

**What's Stored:**
- Range → Raft Group mapping
- Node health status
- Schema definitions
- Cluster configuration

### Leader Election (Raft)

**Heartbeat Mechanism:**

```
Every 100ms:
  Leader → Followers: "I'm alive"

If no heartbeat for 1 second:
  Follower becomes CANDIDATE
  Requests votes from peers
  Majority votes → Becomes LEADER
```

**Metadata Update:**

```
New Leader:
  1. Update etcd: range_id X → leader = Node 4
  2. Increment version

Gateway:
  1. Watch etcd for changes
  2. Update local cache
  3. Route future requests to new leader
```

**Failover Latency:**
- Detection: 1 second (election timeout)
- Election: 100-500ms (vote + majority ACK)
- **Total:** ~1.5 seconds downtime

---

## Schema Changes (Online DDL)

### The Problem

```sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);
```

**Challenge:** Table has 1 billion rows across 100 shards. Can't lock all.

### Multi-Version Schema

**Phases:**

```
Phase 1 (Write Both):
  Writes: (user_id, name, email)
  Reads: (user_id, name) [old schema]

Phase 2 (Backfill):
  Background worker updates all rows
  Progress: 0% → 100% over hours/days

Phase 3 (Read New):
  Reads: (user_id, name, email) [new schema]

Phase 4 (Cleanup):
  Drop old schema version
```

**Zero Downtime:** Different rows temporarily have different schema versions (tracked via MVCC).

*See [pseudocode.md::online_schema_change()](pseudocode.md) for implementation.*

---

## Bottlenecks and Solutions

### Bottleneck 1: Hotspot

**Problem:** Range `[2024-01-01...2024-01-02]` receives 100K writes/sec (today's data).

**Solution: Load-Based Splitting**

```
Monitor QPS per range:
  If QPS > 10K for 60 seconds:
    Split range at median key
    Create new Raft Group
    Redistribute data

Before: [2024-01-01...2024-12-31] → 1 group (100K QPS)
After:
  [2024-01-01...2024-01-01] → Group A (100K QPS)
  [2024-01-02...2024-12-31] → Group B (1K QPS)
```

*See [pseudocode.md::detect_hotspot()](pseudocode.md) for implementation.*

### Bottleneck 2: Cross-Region Write Latency

**Problem:** Multi-region consensus requires 100-200ms (speed of light).

**Solution: Regional Raft Groups**

```
US data: [A-M] → US Raft Group (3 nodes in US) = 10ms writes
EU data: [N-Z] → EU Raft Group (3 nodes in EU) = 10ms writes

Cross-region transaction: Use 2PC (200ms)
```

**Result:** 80% of writes are in-region (10ms), 20% are cross-region (200ms).

### Bottleneck 3: Metadata Lookup Overhead

**Problem:** Every query must consult metadata store (etcd).

**Solution: Client-Side Caching**

```
Gateway cache:
  range_id → leader_node
  TTL: 60 seconds

On cache miss: Query etcd → Update cache
On leader change: etcd notification → Invalidate cache
```

**Result:** 99% cache hit rate, 1% queries hit etcd.

---

## Edge Cases

### Split Brain Prevention

**Scenario:** Network partition splits cluster.

```
Partition A: Node 1, Node 2 (2 nodes)
Partition B: Node 3 (1 node)
```

**Raft Protection:**

- Partition A: 2 out of 3 = Majority ✅ → Can elect leader, accept writes
- Partition B: 1 out of 3 = No majority ❌ → Reject writes

**Key:** Raft requires strictly >N/2 to form quorum (prevents split brain).

### Clock Skew

**Problem:** Node 1 clock is 5 seconds ahead of Node 2 (transaction order ambiguous).

**Solution: Hybrid Logical Clock (HLC)**

```
HLC = (physical_time, logical_counter)

Node 1: (1640000005, 0)
Node 2 receives, has local time 1640000000
Node 2 adjusts: max(1640000000, 1640000005) = 1640000005
Node 2 increments logical: (1640000005, 1)
```

**Result:** Total ordering maintained even with clock skew.

*See [pseudocode.md::hybrid_logical_clock()](pseudocode.md) for implementation.*

### Zombie Leader

**Scenario:** Old leader partitioned, doesn't know it's not leader.

```
Node 1 (old leader): Still thinks it's leader
Node 2 (new leader): Elected after Node 1 partitioned

Node 1 tries to replicate:
  Node 1 → Node 2: AppendEntries(term=1, entry=X)
  Node 2: Reject (my term=2 > your term=1)

Node 1: Realizes stale, steps down to follower
```

**Raft Protection:** Every RPC includes term number. Stale leader detected by term mismatch.

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Using Hash Sharding for Range Queries

**Problem:**

```sql
SELECT * FROM orders WHERE timestamp BETWEEN '2024-01-01' AND '2024-01-31';
```

With hash sharding, must query ALL 100 shards (scatter-gather).

**✅ Best Practice:** Use range-based sharding for time-series data. Query hits 3 shards instead of 100.

---

### ❌ Anti-Pattern 2: Reading Only from Leader

**Problem:** Leader overloaded, followers idle.

**✅ Best Practice:** Enable follower reads with timestamp bounds. Result: 10× read capacity.

---

### ❌ Anti-Pattern 3: Large Multi-Shard Transactions

**Problem:**

```sql
UPDATE accounts SET balance = balance - 1 WHERE user_id IN (1...1000);
```

Affects 50 shards, 2PC locks all for seconds → High contention.

**✅ Best Practice:** Batch small transactions OR co-locate related data on same shard.

---

### ❌ Anti-Pattern 4: Ignoring Hotspot Monitoring

**Problem:** Range becomes hot, node crashes due to overload.

**✅ Best Practice:** Automated load-based splitting (monitor QPS, auto-split hot ranges).

---

## Alternative Approaches

### Alternative 1: Master-Slave Replication

**Why NOT Chosen:**
- ❌ Single master bottleneck (no write scalability)
- ❌ Manual failover (minutes of downtime)
- ⚠️ Replication lag (eventual consistency)

**When to Use:** Read-heavy workload, small dataset (<1 TB).

### Alternative 2: Multi-Master Replication

**Why NOT Chosen:**
- ❌ Conflict resolution required (complex)
- ❌ Lost updates possible (last-write-wins)

**When to Use:** Offline-first apps, CRDTs for counters.

### Alternative 3: DynamoDB (Eventual Consistency)

**Why NOT Chosen:**
- ⚠️ Eventual consistency (can read stale)
- ❌ No SQL support (NoSQL only)
- ⚠️ Limited transactions (single partition)

**When to Use:** Key-value workloads, ultra-low latency required (<5ms).

---

## Monitoring

### Key Metrics

| Metric | Target | Alert Threshold |
|--------|--------|----------------|
| **Write Latency (p99)** | <50ms | >100ms |
| **Read Latency (p99)** | <10ms | >50ms |
| **Raft Election Frequency** | <1/day | >5/hour |
| **Transaction Abort Rate** | <1% | >5% |
| **Follower Lag** | <1s | >10s |

### Distributed Tracing

**Example Trace:**

```
Trace ID: abc-123
Span 1: Client → Gateway (5ms)
Span 2: Gateway → Metadata (2ms)
Span 3: Gateway → Leader (3ms)
Span 4: Leader → Follower 1 (20ms) [SLOW!]
Span 5: Leader → Follower 2 (15ms)

Total: 50ms
Bottleneck: Follower 1 replication
```

**Tools:** Jaeger, Prometheus, Grafana

---

## Cost Analysis

**Example: 30-Node Cluster (100 TB Data, 1M QPS)**

| Component | Cost |
|-----------|------|
| **Compute** (30 × r5.4xlarge) | $726/day |
| **Storage** (300 TB × $0.10/GB/month) | $30,000/month |
| **Network** (10 TB/day cross-region) | $900/day |
| **Total** | **$52,000/month** |

**Cost Optimization:**

1. **Regional Isolation:** 80% data in single region → Save $12K/month (network)
2. **Cold Storage:** Archive old data to S3 Glacier → Save $10K/month (storage)
3. **Follower Reads:** Offload 90% reads → Save $5K/month (compute)

**Optimized Total:** $25,000/month (52% reduction)

---

## Real-World Examples

### Google Spanner

**Scale:** Millions of QPS, petabytes of data, 20+ regions

**Unique Features:**
- **TrueTime API:** Atomic clocks + GPS for global sync
- **Commit Wait:** Waits out clock uncertainty (7ms overhead)
- **External Consistency:** Transactions appear in real-time order

### CockroachDB

**Scale:** 100K QPS, 100 TB, multi-region

**Unique Features:**
- **PostgreSQL Wire Protocol:** Drop-in replacement
- **Horizontal Scaling:** Add nodes, data auto-rebalances
- **Survivability Modes:** Zone, Region, or Global

**Differences from Spanner:**

| Feature | Spanner | CockroachDB |
|---------|---------|-------------|
| **Clock** | TrueTime (atomic) | HLC (hybrid logical) |
| **Consensus** | Paxos | Raft |
| **Deployment** | Google Cloud only | Multi-cloud |

---

## Interview Discussion Points

### Q1: How do you handle network partition?

**Answer:** Raft prevents split brain via quorum. Partition with majority (>N/2) continues, minority rejects writes.

### Q2: Why not PostgreSQL with read replicas?

**Answer:** PostgreSQL has:
- ❌ Write bottleneck (single master)
- ❌ Manual sharding required
- ⚠️ Replication lag

Distributed DB provides:
- ✅ Horizontal write scaling
- ✅ Automatic sharding
- ✅ Strong consistency

### Q3: What happens during schema migration?

**Answer:** Multi-version schema:
1. Write both old and new schema
2. Background backfill
3. Switch reads to new schema
4. Drop old schema

Zero downtime (MVCC tracks version per row).

---

## Trade-offs Summary

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Strong Consistency (ACID) | ❌ Higher Latency (50-200ms) |
| ✅ Horizontal Write Scalability | ❌ Complexity (Raft, 2PC, sharding) |
| ✅ Automatic Failover (<2s) | ❌ Write Amplification (3× network) |
| ✅ SQL Compatibility | ❌ Cost ($50K/month for 1M QPS) |
| ✅ Multi-Region Support | ❌ Operational Overhead |

**Best For:**
- Financial systems (banks, payment processors)
- Multi-region SaaS platforms
- Applications requiring strong consistency + horizontal scaling

**NOT For:**
- Ultra-low-latency workloads (<5ms writes)
- Simple CRUD applications (overkill)
- Offline-first mobile apps

---

## Conclusion

Key insights:

1. **Consensus is Expensive:** Every write requires network round-trips (Raft quorum)
2. **CAP Theorem is Real:** We chose Consistency over Availability (CP system)
3. **Sharding is Complex:** Requires metadata management, hotspot detection, range splitting
4. **Global Scale is Slow:** Speed of light limits cross-region latency (100-200ms)
5. **Operational Excellence:** Monitoring and chaos engineering are critical

**When to Build vs Buy:**

| Scenario | Recommendation |
|----------|---------------|
| **Need strong consistency + multi-region** | ✅ CockroachDB / YugabyteDB |
| **Google Cloud customer** | ✅ Google Spanner |
| **Eventual consistency OK** | ✅ DynamoDB / Cassandra |
| **Single-region, <1TB** | ✅ PostgreSQL with replicas |

---

## References

### Academic Papers
- **[Raft Consensus Algorithm](https://raft.github.io/raft.pdf)** - Diego Ongaro, 2014
- **[Google Spanner](https://research.google.com/archive/spanner-osdi2012.pdf)** - OSDI 2012
- **[Calvin: Fast Distributed Transactions](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf)** - SIGMOD 2012

### Related Chapters
- **[2.1.4 Database Scaling](../../02-components/2.1-databases/2.1.4-database-scaling.md)** - Sharding, Replication
- **[2.5.2 Consensus Algorithms](../../02-components/2.5-algorithms/2.5.2-consensus-algorithms.md)** - Raft, Paxos
- **[2.5.3 Distributed Locking](../../02-components/2.5-algorithms/2.5.3-distributed-locking.md)** - Lock management
- **[1.1.1 CAP Theorem](../../01-principles/1.1.1-cap-theorem.md)** - Consistency vs Availability

### Open Source Projects
- **[CockroachDB](https://github.com/cockroachdb/cockroach)** - Distributed SQL (Go)
- **[TiDB](https://github.com/pingcap/tidb)** - MySQL-compatible (Go)
- **[YugabyteDB](https://github.com/yugabyte/yugabyte-db)** - PostgreSQL-compatible (C++)
- **[etcd](https://github.com/etcd-io/etcd)** - Distributed key-value store (Go)

