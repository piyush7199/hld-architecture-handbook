# Distributed Database - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A globally distributed database that provides strong consistency (ACID transactions across multiple nodes), high availability (survives datacenter failures), horizontal scalability (add nodes to increase capacity), and automatic sharding (distribute data without manual intervention).

**Real-World Examples:**
- **Google Spanner:** Powers Google Ads, Gmail, Google Cloud Platform
- **CockroachDB:** Multi-region distributed SQL database
- **YugabyteDB:** PostgreSQL-compatible distributed database
- **TiDB:** MySQL-compatible distributed database

**The Core Challenge:**

Traditional databases are either:
1. **Strongly Consistent but NOT Scalable** (single PostgreSQL/MySQL instance)
2. **Scalable but NOT Strongly Consistent** (Cassandra - eventual consistency)

We need **BOTH**: Strong consistency (ACID) **AND** horizontal scalability.

---

## Requirements & Scale

### Functional Requirements

1. **Automatic Sharding:** Data automatically distributed across nodes based on key ranges
2. **Multi-Region Replication:** Data replicated across geographic regions (US, EU, APAC)
3. **Strong Consistency:** ACID transactions with serializable isolation
4. **Fault Tolerance:** System remains operational if up to N/2 - 1 nodes fail
5. **Schema Changes:** Support DDL operations (ALTER TABLE) without downtime
6. **SQL Interface:** Standard SQL for compatibility with existing applications

### Non-Functional Requirements

| Requirement | Target | Rationale |
|------------|--------|-----------|
| **Availability** | 99.99% (~52 min downtime/year) | Tolerate node/datacenter failures |
| **Write Latency** | <50ms (single region), <200ms (multi-region) | Consensus quorum latency |
| **Read Latency** | <10ms (local), <100ms (cross-region) | Read from nearest replica |
| **Throughput** | 1M QPS (100K writes/sec, 900K reads/sec) | Handle large-scale applications |
| **Data Durability** | 99.999999999% (11 nines) | No data loss even with failures |
| **Scale** | 1000+ nodes, 100+ TB per node | Petabyte-scale deployments |

### Scale Estimation

| Metric | Assumption | Calculation | Result |
|--------|-----------|-------------|--------|
| **Total Data** | 100 TB across all regions | - | 100 TB |
| **Replication Factor** | 3 replicas (quorum) | 100 TB × 3 | 300 TB total storage |
| **Node Size** | 10 TB per node | 300 TB / 10 TB | 30 nodes minimum |
| **Read QPS** | 900,000 reads/sec | Assume 30 nodes | 30K reads/sec per node |
| **Write QPS** | 100,000 writes/sec | Consensus overhead: 3× writes | 10K writes/sec per node |
| **Network Bandwidth** | 10 Gbps per node | Replication traffic | 1.25 GB/sec per node |
| **Consensus Groups** | 1000 shards (ranges) | 30 nodes / 3 replicas | ~10 shards per node |

**Key Insight:** With replication factor of 3, every write requires 3× network traffic (leader → 2 followers). This is the fundamental trade-off of strong consistency.

---

## Key Components

| Component | Responsibility | Technology | Scalability |
|-----------|---------------|------------|-------------|
| **SQL Gateway** | Parse SQL, route to shards | Stateless (Go, Java) | Horizontal |
| **Metadata Store** | Track shard locations, leaders | etcd (Raft) | Replicated |
| **Raft Groups** | Consensus and replication | Raft algorithm | Per-shard |
| **Storage Layer** | Persistent data storage | RocksDB (LSM Tree) | Per-node |
| **Transaction Coordinator** | 2PC for cross-shard transactions | Stateless | Horizontal |

---

## Architecture Overview

**Lambda Architecture Components:**

1. **Client Layer:** Applications send SQL queries
2. **Gateway/Router Layer:** SQL Gateway parses queries, consults metadata, routes to Raft Group Leaders
3. **Raft Consensus Groups:** Each shard (key range) has a Raft group (1 leader + 2 followers)
4. **Storage Layer:** RocksDB (LSM Tree) for persistent storage

**Flow Summary:**

1. **Client** sends SQL query to **Gateway**
2. **Gateway** parses query, extracts key, consults **Metadata Store** to find which Raft Group owns that key range
3. **Gateway** routes request to **Raft Group Leader**
4. **Leader** replicates write to **Followers** via Raft consensus
5. **Leader** commits after **quorum** (2 out of 3 nodes acknowledge)
6. **Leader** applies to **Storage Layer** (RocksDB)
7. **Leader** returns success to **Gateway** → **Client**

---

## Data Model and Sharding Strategy

### Range-Based Sharding

**Why Range-Based?**
- Efficient range scans (`WHERE timestamp BETWEEN X AND Y`)
- Natural fit for time-series data
- Simple split/merge logic

**Shard Structure:**
```
Table: users
Primary Key: user_id (UUID)

Shard 1 (Range: [00000000...5555555])  → Raft Group A (Nodes 1, 2, 3)
Shard 2 (Range: [55555556...AAAAAAAA])  → Raft Group B (Nodes 4, 5, 6)
Shard 3 (Range: [AAAAAAAB...FFFFFFFF])  → Raft Group C (Nodes 7, 8, 9)
```

### Dynamic Range Splitting

When a range becomes too large (>64 MB) or too hot (>10K QPS), split it:

```
Before:
  Range 1: [A-Z] (100 MB, 50K QPS) → Overloaded

After:
  Range 1a: [A-M] (50 MB, 25K QPS) → Raft Group A
  Range 1b: [N-Z] (50 MB, 25K QPS) → Raft Group B (new)
```

---

## Consensus and Replication (Raft)

### Why Raft Over Paxos?

| Feature | Raft | Paxos |
|---------|------|-------|
| **Understandability** | ✅ Designed to be understandable | ❌ Notoriously complex |
| **Leader Election** | ✅ Strong leader (simplifies design) | ⚠️ Multi-leader possible |
| **Implementation** | ✅ Many open-source implementations | ⚠️ Few correct implementations |
| **Log Replication** | ✅ Sequential log (simple) | ⚠️ Can reorder (complex) |

**Raft Guarantees:**

1. **Leader Election:** If leader fails, new leader elected within ~1 second (election timeout)
2. **Log Replication:** All writes appended to leader's log, replicated to followers
3. **Safety:** Once committed (quorum acknowledges), never lost
4. **Linearizability:** Reads/writes appear instantaneous and sequential

### Raft Write Flow

```
Client → Leader:    Write(key=X, value=Y)

Leader:
  1. Append to local log (uncommitted)
  2. Send AppendEntries RPC to Followers
  
Followers:
  1. Receive AppendEntries
  2. Append to local log
  3. ACK to Leader
  
Leader:
  1. Wait for quorum (2 out of 3 ACKs)
  2. Mark entry as "committed"
  3. Apply to state machine (RocksDB)
  4. Return success to Client
  
Followers:
  1. Receive "committed" notification
  2. Apply to state machine
```

**Latency Analysis:**
- **Single Region:** 1 RTT (Leader → Follower → Leader) = 1-5ms
- **Multi-Region:** 1 RTT across regions = 50-200ms (speed of light limit)

---

## Transaction Management (Distributed ACID)

### The Problem

Transaction spans multiple shards (e.g., transfer money from Account A in Shard 1 to Account B in Shard 2).

**Without Coordination:** Race condition can violate consistency:
```
Time   Shard 1 (Account A)      Shard 2 (Account B)      Balance
0ms    A: $100                  B: $100                  $200
1ms    Deduct $50 → A: $50      -                        $150 ❌ (B not updated yet)
2ms    -                        Add $50 → B: $150        $200 ✅
```

**Attack Vector:** Client reads B at 1ms, sees $100 (old value) + $50 (A's deduction) = $150 total. Money disappeared!

### Solution: Two-Phase Commit (2PC) with Distributed Locks

**Why 2PC?**
- Ensures atomicity: All shards commit OR all abort
- Prevents partial updates visible to other transactions

**2PC Flow:**

```
Transaction: UPDATE accounts SET balance = balance - 50 WHERE id = A;
             UPDATE accounts SET balance = balance + 50 WHERE id = B;

Phase 1 (PREPARE):
  Coordinator → Shard 1: "Can you commit?"
  Coordinator → Shard 2: "Can you commit?"
  
  Shard 1: Lock row A, check constraints → Reply "YES"
  Shard 2: Lock row B, check constraints → Reply "YES"

Phase 2 (COMMIT):
  Coordinator: All replied YES → Send "COMMIT" to both
  
  Shard 1: Apply change, release lock → ACK
  Shard 2: Apply change, release lock → ACK
  
  Coordinator: All ACKs received → Return SUCCESS to client
```

**Failure Scenarios:**

| Scenario | Handling |
|---------|---------|
| **Shard replies NO** | Coordinator sends ABORT to all shards, release locks |
| **Coordinator fails before COMMIT** | Shards wait (timeout after 30s), then abort |
| **Shard fails after PREPARE** | Coordinator waits for recovery, then commits or aborts |

---

## Read Path Optimization

### Follower Reads

**Problem:** All reads hit leader → Leader overloaded (90% CPU), followers idle (10% CPU).

**Solution:** Enable follower reads with timestamp bounds.

```
Strong consistency required?
  → Read from Leader

Stale data acceptable (< 1 min old)?
  → Read from nearest Follower
```

**Result:** 10× read capacity (utilize all replicas).

---

## Bottlenecks & Solutions

### Bottleneck 1: Hotspot Problem

**Scenario:** Key range `[2024-01-01...2024-01-02]` receives 100K writes/sec (current day's data).

**Problem:** Single Raft Group can't handle load (leader overloaded).

**Solution 1: Split Range**
```
Before: [2024-01-01...2024-12-31] → 1 Raft Group (overloaded)

After:
  [2024-01-01...2024-01-01] → Raft Group A (100K QPS)
  [2024-01-02...2024-12-31] → Raft Group B (1K QPS)
```

**Solution 2: Load-Based Splitting**
Monitor QPS per range:
```
If range QPS > 10K for 60 seconds:
  Split range at median key
  Create new Raft Group
  Redistribute data
```

### Bottleneck 2: Cross-Region Write Latency

**Problem:** Multi-region consensus requires cross-region network (100-200ms).

**Solution: Regional Raft Groups**
```
US Region:     [A-M] → US Raft Group (3 nodes in US)
EU Region:     [N-Z] → EU Raft Group (3 nodes in EU)

Writes to US data: 10ms latency (in-region consensus)
Writes to EU data: 10ms latency (in-region consensus)

Cross-region transaction: Use 2PC across regional groups (200ms)
```

### Bottleneck 3: Metadata Bottleneck

**Problem:** Every query must consult metadata store (etcd).

**Solution: Client-Side Caching**
```
Gateway maintains local cache:
  range_id → leader_node
  TTL: 60 seconds
  
On cache miss:
  Query etcd → Update cache
  
On leader change:
  etcd sends notification → Invalidate cache
```

**Result:** 99% cache hit rate, 1% queries hit etcd.

---

## Common Anti-Patterns to Avoid

### 1. Using Hash-Based Sharding for Range Queries

❌ **Bad:** Hash sharding for time-series data
```sql
SELECT * FROM orders WHERE timestamp BETWEEN '2024-01-01' AND '2024-01-31';

With Hash Sharding:
  Hash('2024-01-01') → Shard 5
  Hash('2024-01-02') → Shard 12
  Hash('2024-01-03') → Shard 3
  ...
  
Must query ALL shards (scatter-gather) = 100 network calls
```

✅ **Good:** Range-based sharding for time-series data
```
Shard 1: [2024-01-01...2024-01-10]
Shard 2: [2024-01-11...2024-01-20]
Shard 3: [2024-01-21...2024-01-31]

Query hits 3 shards instead of 100 (33× improvement)
```

### 2. Reading from Leader Only

❌ **Bad:** All reads → Leader (Leader overloaded, followers idle)

✅ **Good:** Enable follower reads with timestamp bounds
```
Strong consistency required?
  → Read from Leader

Stale data acceptable (< 1 min old)?
  → Read from nearest Follower
```

**Result:** 10× read capacity (utilize all replicas).

### 3. Large Transactions Spanning Many Shards

❌ **Bad:**
```sql
BEGIN;
  UPDATE accounts SET balance = balance - 1 WHERE user_id IN (1, 2, 3, ..., 1000);
COMMIT;

Affects 1000 rows across 50 shards
2PC coordinator must lock 50 shards for entire transaction (seconds)
High contention → Deadlocks
```

✅ **Good:** Batch small transactions or co-locate related data
```
Option 1: 10 transactions of 100 rows each
  - Each transaction locks 5 shards (manageable)
  - Lower contention

Option 2: Shard by account_id
  - All of user's accounts on same shard
  - Single-shard transaction (no 2PC)
```

### 4. Ignoring Hotspot Monitoring

❌ **Bad:** Deploy system, assume even distribution → Week 1: Works fine → Month 1: Range [A-B] becomes hot → Month 2: Node crashes (overload)

✅ **Good:** Automated load-based splitting
```
Monitor every range:
  - QPS (queries per second)
  - Storage size (MB)
  - CPU usage

If QPS > 10K for 60 seconds:
  Split range automatically
```

---

## Monitoring & Observability

### Key Metrics

| Metric | Target | Alert Threshold | Rationale |
|--------|--------|----------------|-----------|
| **Write Latency (p99)** | <50ms | >100ms | Indicates consensus slow |
| **Read Latency (p99)** | <10ms | >50ms | Indicates follower lag |
| **Raft Election Frequency** | <1/day | >5/hour | Indicates instability |
| **Range Split Frequency** | 10-100/day | >1000/day | Indicates hotspot issues |
| **Transaction Abort Rate** | <1% | >5% | Indicates contention |
| **Follower Lag** | <1 second | >10 seconds | Indicates replication slow |

### Distributed Tracing

**Trace Example:**
```
Trace ID: abc-123-def
Span 1: Client → Gateway (5ms)
Span 2: Gateway → Metadata (2ms)
Span 3: Gateway → Leader (3ms)
Span 4: Leader → Follower 1 (20ms) [SLOW!]
Span 5: Leader → Follower 2 (15ms)
Span 6: Leader → Gateway (3ms)
Span 7: Gateway → Client (2ms)

Total: 50ms
Bottleneck: Follower 1 replication (20ms)
```

**Tools:**
- **Jaeger:** Open-source distributed tracing
- **Prometheus:** Metrics collection
- **Grafana:** Dashboards

---

## Trade-offs Summary

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Strong consistency (ACID) | ❌ Higher latency (consensus overhead) |
| ✅ Horizontal scalability | ❌ Complex architecture (Raft, 2PC) |
| ✅ Automatic sharding | ❌ Network overhead (3× writes for replication) |
| ✅ High availability (99.99%) | ❌ Higher cost (3× storage, network) |
| ✅ SQL compatibility | ❌ Operational complexity (monitoring, tuning) |

---

## Real-World Examples

### Google Spanner
- **Usage:** Google Ads, Gmail, Google Cloud Platform
- **Scale:** Millions of QPS, Petabytes of data, 20+ regions globally
- **Unique Features:** TrueTime API (atomic clocks + GPS), Commit Wait (7ms overhead), External Consistency

### CockroachDB
- **Usage:** Digital Ocean, Comcast, Netflix
- **Scale:** 100K QPS per cluster, 100 TB typical deployment, Multi-region (5 regions common)
- **Unique Features:** PostgreSQL Wire Protocol, Horizontal Scaling, Survivability Modes

### YugabyteDB
- **Usage:** Kroger, Xignite
- **Scale:** 50K QPS per cluster, 10 TB typical deployment
- **Unique Features:** PostgreSQL + Cassandra APIs, DocDB Engine

---

## Key Takeaways

1. **Raft consensus** enables strong consistency with automatic failover (<2s)
2. **Range-based sharding** efficient for time-series and range queries
3. **2PC transactions** ensure atomicity across multiple shards
4. **Follower reads** increase read capacity by 10× (utilize all replicas)
5. **Dynamic range splitting** handles hotspots automatically
6. **Metadata caching** reduces etcd load by 99% (client-side cache)
7. **Regional Raft groups** reduce cross-region latency (10ms vs 200ms)
8. **LSM Tree (RocksDB)** provides high write throughput (10K+ writes/sec)
9. **Quorum-based replication** prevents split brain (N/2 + 1 nodes)
10. **Automatic rebalancing** distributes data evenly when nodes added/removed

---

## Recommended Stack

- **SQL Gateway:** Go or Java (stateless, horizontal scaling)
- **Metadata Store:** etcd (Raft consensus)
- **Consensus:** Raft algorithm (per-shard groups)
- **Storage:** RocksDB (LSM Tree, high write throughput)
- **Transaction Coordinator:** Stateless service (2PC orchestration)
- **Monitoring:** Prometheus + Grafana (metrics), Jaeger (tracing)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, component design, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (Raft consensus, 2PC transactions, failure scenarios)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (Raft, 2PC, range splitting)

