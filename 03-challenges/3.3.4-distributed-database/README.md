# 3.3.4 Design a Distributed Database (CockroachDB/Spanner)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions. 
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## 1. Problem Statement

Design a globally distributed database that provides:
- **Strong Consistency** (ACID transactions across multiple nodes)
- **High Availability** (survives datacenter failures)
- **Horizontal Scalability** (add nodes to increase capacity)
- **Automatic Sharding** (distribute data without manual intervention)

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

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

1. **Automatic Sharding:** Data automatically distributed across nodes based on key ranges
2. **Multi-Region Replication:** Data replicated across geographic regions (US, EU, APAC)
3. **Strong Consistency:** ACID transactions with serializable isolation
4. **Fault Tolerance:** System remains operational if up to $N/2 - 1$ nodes fail
5. **Schema Changes:** Support DDL operations (ALTER TABLE) without downtime
6. **SQL Interface:** Standard SQL for compatibility with existing applications

### Non-Functional Requirements (NFRs)

| Requirement | Target | Rationale |
|------------|--------|-----------|
| **Availability** | 99.99% ($\sim 52$ min downtime/year) | Tolerate node/datacenter failures |
| **Write Latency** | <50ms (single region), <200ms (multi-region) | Consensus quorum latency |
| **Read Latency** | <10ms (local), <100ms (cross-region) | Read from nearest replica |
| **Throughput** | 1M QPS (100K writes/sec, 900K reads/sec) | Handle large-scale applications |
| **Data Durability** | 99.999999999% (11 nines) | No data loss even with failures |
| **Scale** | 1000+ nodes, 100+ TB per node | Petabyte-scale deployments |

### Scale Estimation

| Metric | Assumption | Calculation | Result |
|--------|-----------|-------------|--------|
| **Total Data** | 100 TB across all regions | - | 100 TB |
| **Replication Factor** | 3 replicas (quorum) | $100 \text{ TB} \times 3$ | 300 TB total storage |
| **Node Size** | 10 TB per node | $300 \text{ TB} / 10 \text{ TB}$ | 30 nodes minimum |
| **Read QPS** | 900,000 reads/sec | Assume 30 nodes | 30K reads/sec per node |
| **Write QPS** | 100,000 writes/sec | Consensus overhead: $3\times$ writes | 10K writes/sec per node |
| **Network Bandwidth** | 10 Gbps per node | Replication traffic | 1.25 GB/sec per node |
| **Consensus Groups** | 1000 shards (ranges) | 30 nodes / 3 replicas | ~10 shards per node |

**Key Insight:** With replication factor of 3, every write requires 3× network traffic (leader → 2 followers). This is the fundamental trade-off of strong consistency.

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

### Component Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  App 1   │  │  App 2   │  │  App 3   │  │  App N   │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
│       │             │             │             │                │
│       └─────────────┴─────────────┴─────────────┘                │
│                         │                                         │
└─────────────────────────┼─────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                     GATEWAY / ROUTER LAYER                       │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  SQL Gateway (Stateless)                                    │ │
│  │  - Parse SQL query                                          │ │
│  │  - Consult metadata: Which shard owns key range?           │ │
│  │  - Route to Raft Group Leader                              │ │
│  └────────────────────────────────────────────────────────────┘ │
│                          │                                       │
│                  ┌───────┴───────┐                              │
│                  │   Metadata    │                              │
│                  │   Store       │                              │
│                  │ (etcd/Raft)   │                              │
│                  └───────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
            ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      RAFT CONSENSUS GROUPS                       │
│                                                                  │
│  Range 1: [A-F]         Range 2: [G-M]       Range 3: [N-Z]    │
│  ┌──────────────┐      ┌──────────────┐     ┌──────────────┐   │
│  │    Leader    │      │    Leader    │     │    Leader    │   │
│  │   Node 1     │      │   Node 4     │     │   Node 7     │   │
│  └──────┬───────┘      └──────┬───────┘     └──────┬───────┘   │
│         │                     │                    │            │
│    ┌────┴────┐           ┌────┴────┐          ┌────┴────┐      │
│    │         │           │         │          │         │       │
│    ▼         ▼           ▼         ▼          ▼         ▼       │
│  ┌────┐   ┌────┐      ┌────┐   ┌────┐     ┌────┐   ┌────┐     │
│  │ N2 │   │ N3 │      │ N5 │   │ N6 │     │ N8 │   │ N9 │     │
│  └────┘   └────┘      └────┘   └────┘     └────┘   └────┘     │
│  Follower Follower    Follower Follower   Follower Follower    │
│                                                                  │
│  Each group: 3 replicas (1 leader, 2 followers)                 │
│  Quorum: 2 out of 3 nodes must agree for write to commit        │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE LAYER (LSM Tree)                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  RocksDB / LevelDB (Log-Structured Merge Tree)             │ │
│  │  - Memtable (in-memory buffer)                             │ │
│  │  - WAL (Write-Ahead Log for durability)                    │ │
│  │  - SSTable files (on-disk sorted files)                    │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Flow Summary:**

1. **Client** sends SQL query to **Gateway**
2. **Gateway** parses query, extracts key, consults **Metadata Store** to find which Raft Group owns that key range
3. **Gateway** routes request to **Raft Group Leader**
4. **Leader** proposes write to **Followers** (Raft consensus)
5. **Quorum** (2 out of 3) acknowledges → Write committed
6. **Leader** returns success to **Gateway** → **Client**

---

## 4. Data Model and Sharding Strategy

### 4.1 Range-Based Sharding

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

**Metadata Schema:**

```sql
CREATE TABLE range_metadata (
    range_id BIGINT PRIMARY KEY,
    start_key BYTES NOT NULL,
    end_key BYTES NOT NULL,
    leader_node_id INT NOT NULL,
    replica_node_ids INT[] NOT NULL,  -- [node1, node2, node3]
    version INT NOT NULL,  -- Incremented on split/merge
    INDEX idx_key_lookup (start_key, end_key)
);
```

### 4.2 Dynamic Range Splitting

When a range becomes too large (>64 MB) or too hot (>10K QPS), split it:

```
Before:
  Range 1: [A-Z] (100 MB, 50K QPS) → Overloaded

After:
  Range 1a: [A-M] (50 MB, 25K QPS) → Raft Group A
  Range 1b: [N-Z] (50 MB, 25K QPS) → Raft Group B (new)
```

*See [pseudocode.md::split_range()](pseudocode.md) for implementation.*

---

## 5. Consensus and Replication (Raft)

### 5.1 Why Raft Over Paxos?

| Feature | Raft | Paxos |
|---------|------|-------|
| **Understandability** | ✅ Designed to be understandable | ❌ Notoriously complex |
| **Leader Election** | ✅ Strong leader (simplifies design) | ⚠️ Multi-leader possible |
| **Implementation** | ✅ Many open-source implementations | ⚠️ Few correct implementations |
| **Log Replication** | ✅ Sequential log (simple) | ⚠️ Can reorder (complex) |

**Raft Guarantees:**

1. **Leader Election:** If leader fails, new leader elected within $\sim 1$ second (election timeout)
2. **Log Replication:** All writes appended to leader's log, replicated to followers
3. **Safety:** Once committed (quorum acknowledges), never lost
4. **Linearizability:** Reads/writes appear instantaneous and sequential

### 5.2 Raft Write Flow

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

*See [pseudocode.md::raft_append_entries()](pseudocode.md) for implementation.*

---

## 6. Transaction Management (Distributed ACID)

### 6.1 The Problem

Transaction spans multiple shards (e.g., transfer money from Account A in Shard 1 to Account B in Shard 2).

**Without Coordination:** Race condition can violate consistency:
```
Time   Shard 1 (Account A)      Shard 2 (Account B)      Balance
0ms    A: $100                  B: $100                  $200
1ms    Deduct $50 → A: $50      -                        $150 ❌ (B not updated yet)
2ms    -                        Add $50 → B: $150        $200 ✅
```

**Attack Vector:** Client reads B at 1ms, sees $100 (old value) + $50 (A's deduction) = $150 total. Money disappeared!

### 6.2 Solution: Two-Phase Commit (2PC) with Distributed Locks

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
|----------|----------|
| **Shard 1 replies NO** | Coordinator sends ABORT to all → Transaction rolled back |
| **Shard 2 crashes after YES** | Coordinator retries COMMIT until Shard 2 recovers (locks held) |
| **Coordinator crashes** | Participant shards timeout, release locks, abort (conservative) |

*See [pseudocode.md::two_phase_commit()](pseudocode.md) for implementation.*

### 6.3 Timestamp Oracle (Global Ordering)

**Problem:** Without global clock, can't determine transaction order across shards.

**Solution:** Centralized **Timestamp Oracle** (HA cluster using Raft):

```
Transaction Start:
  Client → Timestamp Oracle: "Give me a timestamp"
  Oracle: Return unique timestamp (e.g., 1640000000000001)
  
Transaction Commit:
  All writes tagged with this timestamp
  Timestamp acts as global version for MVCC (Multi-Version Concurrency Control)
```

**Google Spanner's TrueTime:**
- Uses atomic clocks + GPS to bound clock uncertainty
- Guarantees: Time is between [earliest, latest] with <7ms uncertainty
- Waits out uncertainty before committing (commit wait)

*See [pseudocode.md::get_timestamp()](pseudocode.md) for implementation.*

---

## 7. Read Path Optimization

### 7.1 Read Strategies

| Strategy | Consistency | Latency | Use Case |
|----------|------------|---------|----------|
| **Read from Leader** | ✅ Strong (linearizable) | ⚠️ Higher (leader may be far) | Financial transactions |
| **Read from Follower** | ⚠️ Stale (eventual consistency) | ✅ Lower (nearest replica) | Analytics, dashboards |
| **Read with Timestamp** | ✅ Strong (snapshot isolation) | ✅ Lower (any replica) | Most reads |

### 7.2 Follower Reads with Timestamp

**Key Idea:** Read from nearest replica using historical timestamp (MVCC).

```
Client wants: SELECT * FROM users WHERE id = X AS OF SYSTEM TIME '1 min ago'

1. Gateway routes to nearest replica (Follower)
2. Follower checks: Do I have data for timestamp T-1min?
3. If YES: Return data (O(1) lookup in LSM tree)
4. If NO: Wait for replication to catch up, then return
```

**Benefit:** 90% of reads can be served from local replicas (10ms latency) instead of cross-region leader (100ms latency).

---

## 8. Metadata Management and Failure Recovery

### 8.1 Metadata Store (etcd/ZooKeeper)

**What's Stored:**
- Range → Raft Group mapping
- Node health status
- Schema (table definitions)
- Cluster configuration

**Schema:**

```sql
CREATE TABLE range_assignments (
    range_id BIGINT PRIMARY KEY,
    table_name VARCHAR(255),
    start_key BYTES,
    end_key BYTES,
    leader_node INT,
    replica_nodes INT[],
    last_updated TIMESTAMP
);
```

### 8.2 Failure Detection and Leader Election

**Heartbeat Mechanism:**

```
Every 100ms:
  Leader → Followers: "I'm alive"
  
If Follower doesn't receive heartbeat for 1 second (election timeout):
  1. Follower becomes CANDIDATE
  2. Increments term (epoch)
  3. Sends RequestVote RPC to all peers
  4. If receives majority votes → Becomes LEADER
  5. Sends heartbeat to establish authority
```

**Metadata Update:**

```
New Leader:
  1. Write to etcd: range_id X → leader = Node 4
  2. Increment version
  
Gateway:
  1. Watch etcd for changes
  2. Update local cache: range_id X → Node 4
  3. Route future requests to Node 4
```

**Latency:**

- **Detection:** 1 second (election timeout)
- **Election:** 100-500ms (RequestVote + majority ACK)
- **Total Downtime:** ~1.5 seconds per failover

*See [pseudocode.md::raft_request_vote()](pseudocode.md) for implementation.*

---

## 9. Schema Changes (Online DDL)

### 9.1 The Problem

```sql
ALTER TABLE users ADD COLUMN email VARCHAR(255);
```

**Challenge:** Table has 1 billion rows across 100 shards. Can't lock entire table (hours of downtime).

### 9.2 Solution: Multi-Version Schema

**Phases:**

```
Phase 1 (Write Both):
  Old writes: (user_id, name)
  New writes: (user_id, name, email)
  Reads: Return (user_id, name) [old schema]

Phase 2 (Backfill):
  Background worker: UPDATE all rows to add email column
  Progress: 0% → 100% over hours/days

Phase 3 (Read New):
  Reads: Return (user_id, name, email) [new schema]
  
Phase 4 (Write New Only):
  All nodes switch to new schema
  Old schema deleted
```

**Key Insight:** Schema version stored per-row in MVCC. Different rows can temporarily have different schemas.

*See [pseudocode.md::online_schema_change()](pseudocode.md) for implementation.*

---

## 10. Bottlenecks and Optimizations

### 10.1 Hotspot Problem

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

*See [pseudocode.md::detect_hotspot()](pseudocode.md) for implementation.*

### 10.2 Cross-Region Write Latency

**Problem:** Multi-region consensus requires cross-region network (100-200ms).

**Solution: Regional Raft Groups**

```
US Region:     [A-M] → US Raft Group (3 nodes in US)
EU Region:     [N-Z] → EU Raft Group (3 nodes in EU)

Writes to US data: 10ms latency (in-region consensus)
Writes to EU data: 10ms latency (in-region consensus)

Cross-region transaction: Use 2PC across regional groups (200ms)
```

### 10.3 Metadata Bottleneck

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

## 11. Deep Dive: Handling Edge Cases

### 11.1 Split Brain Prevention

**Scenario:** Network partition splits cluster into two halves.

```
Partition A: Node 1, Node 2 (2 nodes, CAN form quorum)
Partition B: Node 3 (1 node, CANNOT form quorum)
```

**Raft Protection:**

```
Partition A:
  - 2 out of 3 nodes = Majority ✅
  - Can elect leader
  - Continue accepting writes

Partition B:
  - 1 out of 3 nodes = No majority ❌
  - Cannot elect leader
  - Reject writes (return error to client)
```

**Key Insight:** Raft requires **strictly more than half** (N/2 + 1) to form quorum. This prevents split brain.

### 11.2 Clock Skew

**Problem:** Node 1's clock is 5 seconds ahead of Node 2.

```
Node 1 timestamp: 1640000005
Node 2 timestamp: 1640000000

Transaction order ambiguous!
```

**Solution 1: NTP Synchronization**
- Sync all nodes with NTP (Network Time Protocol)
- Acceptable skew: <100ms
- Monitor skew, alert if >500ms

**Solution 2: Hybrid Logical Clock (HLC)**
```
HLC = (physical_time, logical_counter)

Node 1 sends: (1640000005, 0)
Node 2 receives, has local time 1640000000
Node 2 adjusts: max(1640000000, 1640000005) = 1640000005
Node 2 increments logical counter: (1640000005, 1)
```

*See [pseudocode.md::hybrid_logical_clock()](pseudocode.md) for implementation.*

### 11.3 Zombie Leader

**Scenario:** Old leader partitioned, doesn't know it's not leader anymore.

```
Time   Leader (Node 1)      Follower (Node 2)      Follower (Node 3)
0ms    I'm leader           -                      -
1s     [Network partition]  No heartbeat           No heartbeat
2s     Still leader? Yes!   Start election         Start election
3s     -                    Node 2 elected leader  Vote for Node 2
4s     Accept write? Yes!   I'm leader now         Follower of Node 2
```

**Problem:** Node 1 accepts write at 4s, but it's not the leader!

**Raft Protection:**

```
Node 1 tries to replicate write:
  Node 1 → Node 2: AppendEntries(term=1, entry=X)
  Node 2: Rejects (my term=2 > your term=1)
  
Node 1 realizes: I'm not leader anymore
  - Step down to follower
  - Reject client write
```

**Key Insight:** Every Raft RPC includes **term number**. Stale leader detected by term mismatch.

---

## 12. Common Anti-Patterns

### ❌ Anti-Pattern 1: Using Hash-Based Sharding for Range Queries

**Problem:**

```sql
SELECT * FROM orders WHERE timestamp BETWEEN '2024-01-01' AND '2024-01-31';

With Hash Sharding:
  Hash('2024-01-01') → Shard 5
  Hash('2024-01-02') → Shard 12
  Hash('2024-01-03') → Shard 3
  ...
  
Must query ALL shards (scatter-gather) = 100 network calls
```

**✅ Best Practice:** Use **range-based sharding** for time-series data.

```
Shard 1: [2024-01-01...2024-01-10]
Shard 2: [2024-01-11...2024-01-20]
Shard 3: [2024-01-21...2024-01-31]

Query hits 3 shards instead of 100 (33× improvement)
```

---

### ❌ Anti-Pattern 2: Reading from Leader Only

**Problem:**

```
All reads → Leader
Leader overloaded (90% CPU)
Followers idle (10% CPU)
```

**✅ Best Practice:** Enable **follower reads** with timestamp bounds.

```
Strong consistency required?
  → Read from Leader

Stale data acceptable (< 1 min old)?
  → Read from nearest Follower
```

**Result:** 10× read capacity (utilize all replicas).

---

### ❌ Anti-Pattern 3: Large Transactions Spanning Many Shards

**Problem:**

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 1 WHERE user_id IN (1, 2, 3, ..., 1000);
COMMIT;

Affects 1000 rows across 50 shards
2PC coordinator must lock 50 shards for entire transaction (seconds)
High contention → Deadlocks
```

**✅ Best Practice:** **Batch small transactions** or **co-locate related data**.

```
Option 1: 10 transactions of 100 rows each
  - Each transaction locks 5 shards (manageable)
  - Lower contention

Option 2: Shard by account_id
  - All of user's accounts on same shard
  - Single-shard transaction (no 2PC)
```

---

### ❌ Anti-Pattern 4: Ignoring Hotspot Monitoring

**Problem:**

```
Deploy system, assume even distribution
Week 1: Works fine
Month 1: Range [A-B] becomes hot (celebrity user with ID starting with 'A')
Month 2: Node crashes (overload)
```

**✅ Best Practice:** **Automated load-based splitting**.

```
Monitor every range:
  - QPS (queries per second)
  - Storage size (MB)
  - CPU usage
  
If QPS > 10K for 5 minutes:
  - Auto-split range
  - Create new Raft Group
  - Migrate half the data
```

*See [pseudocode.md::auto_split_range()](pseudocode.md) for implementation.*

---

## 13. Alternative Approaches

### Alternative 1: Master-Slave Replication (Traditional)

**Architecture:**
- Single Master (writes)
- Multiple Slaves (reads)
- Async replication

**Why NOT Chosen:**

| Factor | Master-Slave | Distributed DB (Our Choice) |
|--------|--------------|---------------------------|
| **Write Scalability** | ❌ Single master bottleneck | ✅ Horizontal (add shards) |
| **Fault Tolerance** | ❌ Manual failover (minutes) | ✅ Auto failover (seconds) |
| **Consistency** | ⚠️ Replication lag (eventual) | ✅ Strong (Raft quorum) |
| **Complexity** | ✅ Simple (1 master) | ⚠️ Complex (consensus, 2PC) |

**When to Use Master-Slave:**
- Read-heavy workload (100:1 read/write ratio)
- Stale reads acceptable (analytics, dashboards)
- Small dataset (<1 TB)

---

### Alternative 2: Multi-Master Replication

**Architecture:**
- Multiple masters accept writes
- Conflict resolution required

**Why NOT Chosen:**

**Conflict Example:**

```
Time   Master A                Master B
0ms    User 1: balance = $100  User 1: balance = $100
1ms    Deduct $50 → $50        Deduct $60 → $40
2ms    Replicate to B →        Replicate to A →

Result: Last-write-wins? User 1: $40 or $50? Conflict!
```

**Conflict Resolution Strategies:**

1. **Last-Write-Wins (LWW):** Master B's write wins (timestamp-based). Problem: Lost update (User 1 should have $-10, not $40).

2. **Application-Level Merge:** Requires custom logic per table. Complex.

3. **CRDT (Conflict-Free Replicated Data Types):** Only works for specific data types (counters, sets). Not general-purpose SQL.

**When to Use Multi-Master:**
- Offline-first applications (mobile apps)
- CRDTs for counters (e.g., social media likes)
- Casual consistency acceptable (no financial data)

---

### Alternative 3: DynamoDB-Style Eventual Consistency

**Architecture:**
- Consistent hashing for sharding
- Quorum reads/writes (tunable: R + W > N)
- Conflict resolution via vector clocks

**Why NOT Chosen:**

| Factor | DynamoDB | Distributed SQL (Our Choice) |
|--------|----------|---------------------------|
| **Consistency** | ⚠️ Eventual (can read stale) | ✅ Strong (ACID) |
| **SQL Support** | ❌ No (NoSQL only) | ✅ Yes (standard SQL) |
| **Transactions** | ⚠️ Limited (single partition) | ✅ Multi-shard 2PC |
| **Latency** | ✅ Lower (no consensus) | ⚠️ Higher (Raft + 2PC) |

**When to Use DynamoDB:**
- Key-value workloads (no joins)
- Eventual consistency acceptable
- Single-partition transactions sufficient
- Ultra-low latency required (<5ms)

---

## 14. Monitoring and Observability

### 14.1 Key Metrics

| Metric | Target | Alert Threshold | Rationale |
|--------|--------|----------------|-----------|
| **Write Latency (p99)** | <50ms | >100ms | Indicates consensus slow |
| **Read Latency (p99)** | <10ms | >50ms | Indicates follower lag |
| **Raft Election Frequency** | <1/day | >5/hour | Indicates instability |
| **Range Split Frequency** | 10-100/day | >1000/day | Indicates hotspot issues |
| **Transaction Abort Rate** | <1% | >5% | Indicates contention |
| **Follower Lag** | <1 second | >10 seconds | Indicates replication slow |

### 14.2 Distributed Tracing

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

### 14.3 Chaos Engineering

**Test Failure Scenarios:**

1. **Kill Leader:** Verify failover <2 seconds
2. **Partition Network:** Verify no split brain
3. **Disk Failure:** Verify data not lost (WAL recovery)
4. **Clock Skew:** Inject 10-second skew, verify HLC correction
5. **Overload:** Send 10× traffic, verify graceful degradation

*See [pseudocode.md::chaos_kill_leader()](pseudocode.md) for implementation.*

---

## 15. Real-World Examples

### 16.1 Google Spanner

**Usage:** Google Ads, Gmail, Google Cloud Platform

**Scale:**
- Millions of QPS
- Petabytes of data
- Deployed in 20+ regions globally

**Unique Features:**
- **TrueTime API:** Atomic clocks + GPS for global clock synchronization
- **Commit Wait:** Waits out clock uncertainty before committing (7ms overhead)
- **External Consistency:** Transactions appear to execute in real-time order

**Architecture:**

```
Directory (Shard) → Paxos Group (5 replicas)
Colossus (GFS successor) → Persistent storage
Chubby (Distributed lock) → Metadata coordination
```

### 16.2 CockroachDB

**Usage:** Digital Ocean, Comcast, Netflix

**Scale:**
- 100K QPS per cluster
- 100 TB typical deployment
- Multi-region (5 regions common)

**Unique Features:**
- **PostgreSQL Wire Protocol:** Drop-in replacement for Postgres
- **Horizontal Scaling:** Add nodes, data auto-rebalances
- **Survivability Modes:** Zone, Region, or Global (different latency/consistency)

**Differences from Spanner:**

| Feature | Spanner | CockroachDB |
|---------|---------|-------------|
| **Clock** | TrueTime (atomic) | HLC (hybrid logical) |
| **Consensus** | Paxos | Raft |
| **Storage** | Colossus (custom) | RocksDB (open-source) |
| **Deployment** | Google Cloud only | Multi-cloud (AWS, GCP, Azure) |

### 16.3 YugabyteDB

**Usage:** Kroger, Xignite

**Scale:**
- 50K QPS per cluster
- 10 TB typical deployment

**Unique Features:**
- **PostgreSQL + Cassandra APIs:** Dual interface
- **DocDB Engine:** Custom storage engine (combines benefits of both)

---

## 16. Interview Discussion Points

### Q1: How do you handle a network partition?

**Answer:**

Raft prevents split brain via **quorum**:
- Partition with majority (>N/2) nodes continues
- Partition with minority rejects writes
- After partition heals, minority nodes sync from majority

**Follow-up:** What if partition splits evenly (2 vs 2 in 4-node cluster)?
- **Answer:** No partition can form quorum (need 3 out of 4). System becomes read-only until partition heals or operator removes a node.

### Q2: Why not use PostgreSQL with read replicas?

**Answer:**

**PostgreSQL Read Replicas (Not Chosen):**
- ❌ Write bottleneck (single master)
- ❌ Manual sharding required (application-level)
- ❌ No automatic failover (pg_auto_failover adds some automation)
- ⚠️ Replication lag (eventual consistency)

**Distributed DB (Our Choice):**
- ✅ Horizontal write scaling (add shards)
- ✅ Automatic sharding (transparent to app)
- ✅ Automatic failover (Raft election <2s)
- ✅ Strong consistency (Raft quorum)

**When Postgres is Sufficient:**
- <100K writes/sec
- Data fits on single node (< 2 TB)
- Read-heavy (can scale with replicas)

### Q3: What happens during schema migration on live traffic?

**Answer:**

**Multi-version schema:**

1. **Phase 1:** Writes go to both old and new schema
2. **Phase 2:** Background backfill (can take hours/days)
3. **Phase 3:** Reads switch to new schema
4. **Phase 4:** Old schema dropped

**Zero Downtime:** Different rows can have different schema versions during migration (MVCC tracks version per row).

**Risk:** Long-running queries might see mixed schema. Mitigation: Use schema version filtering in queries.

---

## 17. Trade-offs Summary

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Strong Consistency** (ACID across regions) | ❌ **Higher Latency** (50-200ms writes due to consensus + 2PC) |
| ✅ **Horizontal Write Scalability** (add shards) | ❌ **Complexity** (Raft, 2PC, sharding, metadata management) |
| ✅ **Automatic Failover** (<2s downtime) | ❌ **Write Amplification** (3× network traffic due to replication) |
| ✅ **SQL Compatibility** (standard interface) | ❌ **Cost** ($50K/month for 1M QPS cluster) |
| ✅ **Multi-Region Support** (global deployment) | ❌ **Operational Overhead** (monitor consensus health, hotspots) |

**Best For:**
- Financial systems (banks, payment processors)
- Multi-region SaaS platforms
- Applications requiring strong consistency + horizontal scaling

**NOT For:**
- Ultra-low-latency workloads (<5ms writes)
- Simple CRUD applications (overkill)
- Offline-first mobile apps (conflicts unavoidable)

---

## 18. Conclusion

Designing a distributed database is one of the hardest problems in systems engineering. The key insights:

1. **Consensus is Expensive:** Every write requires network round-trips to majority of replicas (Raft quorum).

2. **CAP Theorem is Real:** You must choose between Consistency and Availability during partitions. We chose Consistency (CP system).

3. **Sharding is Complex:** Automatic sharding requires sophisticated metadata management, hotspot detection, and range splitting.

4. **Global Scale is Slow:** Speed of light limits cross-region latency (100-200ms unavoidable).

5. **Operational Excellence:** Monitoring, chaos engineering, and automated recovery are not optional—they're the difference between 99.9% and 99.99% availability.

**When to Build vs Buy:**

| Scenario | Recommendation |
|----------|---------------|
| **Need strong consistency + multi-region** | ✅ Use CockroachDB / YugabyteDB |
| **Google Cloud customer** | ✅ Use Google Spanner |
| **Eventual consistency OK** | ✅ Use DynamoDB / Cassandra |
| **Single-region, <1TB** | ✅ Use PostgreSQL with replicas |

**Building from scratch:** Only justified if:
- Unique requirements not met by existing systems
- Engineering team >50 people
- Multi-year commitment (distributed DB takes years to mature)

---

## 19. References

### Related System Design Components

- **[2.1.4 Database Scaling](../../02-components/2.1-databases/2.1.4-database-scaling.md)** - Sharding, Replication
- **[2.5.2 Consensus Algorithms](../../02-components/2.5-algorithms/2.5.2-consensus-algorithms.md)** - Raft, Paxos
- **[2.5.3 Distributed Locking](../../02-components/2.5-algorithms/2.5.3-distributed-locking.md)** - Lock management
- **[2.3.4 Distributed Transactions & Idempotency](../../02-components/2.3-messaging-streaming/2.3.4-distributed-transactions-and-idempotency.md)** - 2PC, Sagas
- **[1.1.1 CAP Theorem](../../01-principles/1.1.1-cap-theorem.md)** - Consistency vs Availability
- **[2.1.7 PostgreSQL Deep Dive](../../02-components/2.1-databases/2.1.7-postgresql-deep-dive.md)** - ACID transactions, WAL

### Related Design Challenges

- **[3.1.2 Distributed Cache](../3.1.2-distributed-cache/)** - Consistent hashing, replication
- **[3.2.1 Twitter Timeline](../3.2.1-twitter-timeline/)** - Sharding strategies

### External Resources

- **Raft Consensus Algorithm:** [Raft Paper](https://raft.github.io/raft.pdf) - Diego Ongaro, Stanford, 2014
- **Google Spanner:** [Spanner Paper](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf) - OSDI 2012
- **Calvin:** [Fast Distributed Transactions](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) - SIGMOD 2012
- **CockroachDB Design:** [CockroachDB Paper](https://dl.acm.org/doi/10.1145/3318464.3386134) - SIGMOD 2020
- **CockroachDB:** [GitHub Repository](https://github.com/cockroachdb/cockroach) - Distributed SQL (Go)
- **TiDB:** [GitHub Repository](https://github.com/pingcap/tidb) - MySQL-compatible distributed DB (Go)
- **YugabyteDB:** [GitHub Repository](https://github.com/yugabyte/yugabyte-db) - PostgreSQL-compatible (C++)
- **etcd:** [GitHub Repository](https://github.com/etcd-io/etcd) - Distributed key-value store (Go)

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Chapter 9: Consistency and Consensus
- *Database Internals* by Alex Petrov - Part II: Distributed Systems

