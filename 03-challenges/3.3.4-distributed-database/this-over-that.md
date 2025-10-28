# Distributed Database - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions for designing a distributed database with strong consistency, horizontal scalability, and high availability.

---

## Table of Contents

1. [Raft vs Paxos for Consensus](#1-raft-vs-paxos-for-consensus)
2. [Range-Based Sharding vs Hash-Based Sharding](#2-range-based-sharding-vs-hash-based-sharding)
3. [Two-Phase Commit vs Sagas for Distributed Transactions](#3-two-phase-commit-vs-sagas-for-distributed-transactions)
4. [Synchronous Replication vs Asynchronous Replication](#4-synchronous-replication-vs-asynchronous-replication)
5. [Strong Consistency vs Eventual Consistency](#5-strong-consistency-vs-eventual-consistency)
6. [LSM Tree vs B-Tree for Storage Engine](#6-lsm-tree-vs-b-tree-for-storage-engine)
7. [Multi-Region Deployment Strategies](#7-multi-region-deployment-strategies)

---

## 1. Raft vs Paxos for Consensus

### The Problem

Need a consensus algorithm to ensure that distributed nodes agree on the order and content of operations (writes to database). Without consensus, different nodes might have different views of the data, leading to inconsistency.

**Requirements:**
- Fault tolerance: System must continue operating if minority of nodes fail
- Safety: Once committed, operation never lost or changed
- Liveness: System makes progress (doesn't get stuck)

### Options Considered

| Factor | Raft | Paxos | Viewstamped Replication |
|--------|------|-------|------------------------|
| **Understandability** | ✅ Designed for clarity | ❌ Notoriously complex | ⚠️ Moderate complexity |
| **Leader Election** | ✅ Strong single leader | ⚠️ Can have multiple leaders | ✅ Single leader |
| **Log Replication** | ✅ Sequential append-only | ⚠️ Can reorder entries | ✅ Sequential |
| **Implementation** | ✅ Many open-source (etcd, CockroachDB) | ⚠️ Few correct implementations | ⚠️ Limited implementations |
| **Term/Epoch** | ✅ Clear term concept | ⚠️ Complex ballot numbers | ✅ View numbers |
| **Academic Proof** | ✅ Well-proven | ✅ Well-proven | ✅ Well-proven |

### Decision Made

**Use Raft consensus algorithm.**

**Why Raft over Paxos?**

1. **Understandability:** Raft was explicitly designed to be understandable. The paper breaks down the algorithm into clear subproblems: leader election, log replication, and safety. Paxos is notoriously difficult to understand and implement correctly.

2. **Strong Leader:** Raft has a strong leader that manages all client interactions and log replication. This simplifies the design significantly. Paxos can have multiple simultaneous leaders proposing values, requiring complex conflict resolution.

3. **Implementation Maturity:** Multiple production-grade Raft implementations exist (etcd, Consul, CockroachDB). Paxos implementations are rare and often have subtle bugs.

4. **Debugging:** Raft's clear separation of leader election (term numbers), log replication (index numbers), and commit process makes debugging much easier. Paxos's ballot numbers and promise/accept phases are harder to trace.

### Rationale

**Raft Algorithm:**

```
Leader Election:
1. Followers timeout if no heartbeat (1 second)
2. Follower → CANDIDATE, increments term, requests votes
3. Majority votes → Becomes LEADER
4. Sends heartbeats to establish authority

Log Replication:
1. Leader appends entry to local log
2. Leader sends AppendEntries RPC to followers
3. Followers append entry, ACK to leader
4. Quorum ACKs (2 out of 3) → Mark committed
5. Leader applies to state machine, returns to client
```

**Term Numbers:** Every RPC includes term number. Higher term always wins, preventing stale leaders (zombie leader problem).

**Safety Guarantee:** Leader cannot commit entries from previous terms without committing at least one entry from its own term (prevents uncommitted entries from being committed).

*See [pseudocode.md::raft_append_entries()](pseudocode.md) for detailed implementation.*

### Implementation Details

**Raft Configuration:**
- **Nodes per Group:** 3 or 5 (must be odd for quorum)
- **Election Timeout:** 1000-2000ms (randomized to prevent split votes)
- **Heartbeat Interval:** 100ms (10× faster than election timeout)
- **Log Compaction:** Snapshot every 10K entries (reduce log size)

**Performance:**
- **Write Latency:** 1-5ms (single region), 50-200ms (multi-region)
- **Throughput:** 10K writes/sec per Raft group (leader bottleneck)
- **Failover Time:** 1-2 seconds (election timeout + metadata update)

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Understandability (easier to implement correctly) | ⚠️ Slightly lower performance than Paxos in some edge cases |
| ✅ Strong leader (simplifies design) | ❌ Leader is single point of bottleneck (not failure) |
| ✅ Mature implementations (production-ready code) | - |
| ✅ Easy debugging (clear term/index numbers) | - |

**Real-World Usage:**
- **etcd:** Raft consensus for Kubernetes metadata
- **Consul:** Raft for service discovery
- **CockroachDB:** Raft for replication groups
- **TiDB:** Raft for distributed storage (TiKV)

### When to Reconsider

**Use Paxos when:**
1. **Academic Setting:** Teaching distributed systems (Paxos is foundational)
2. **Extreme Performance:** Need absolute lowest latency (Paxos can pipeline proposals)
3. **Existing Codebase:** Already have battle-tested Paxos implementation

**Use Viewstamped Replication when:**
1. **Simpler than Paxos:** Want consensus without Paxos complexity
2. **Research:** Predates both Raft and Paxos in some aspects

**Indicators for Switching:**
- Raft implementation has bugs that are hard to fix (extremely rare)
- Performance benchmarks show Paxos 10× faster (theoretical, unlikely in practice)
- Team has deep Paxos expertise (very rare)

---

## 2. Range-Based Sharding vs Hash-Based Sharding

### The Problem

Database has 100 TB of data, far too large for single node. Need to partition (shard) data across multiple nodes. Sharding strategy affects query performance, load distribution, and operational complexity.

**Requirements:**
- Efficient range queries (e.g., `WHERE timestamp BETWEEN X AND Y`)
- Even data distribution (no single node overloaded)
- Simple split/merge logic (for dynamic rebalancing)
- Minimal cross-shard transactions

### Options Considered

| Factor | Range-Based Sharding | Hash-Based Sharding | Composite (Hash + Range) |
|--------|---------------------|-------------------|------------------------|
| **Range Queries** | ✅ Single shard hit | ❌ Scatter-gather all shards | ⚠️ Depends on query pattern |
| **Load Distribution** | ⚠️ Hotspot risk (recent data) | ✅ Even distribution | ✅ Even distribution |
| **Split/Merge** | ✅ Simple (split at midpoint) | ❌ Complex (rehash all keys) | ⚠️ Moderate complexity |
| **Ordering** | ✅ Natural ordering maintained | ❌ No ordering | ⚠️ Ordering within hash bucket |
| **Predictability** | ✅ Key → Shard mapping obvious | ❌ Key → Shard unpredictable | ⚠️ Moderately predictable |

### Decision Made

**Use Range-Based Sharding with hotspot detection and automatic splitting.**

**Why Range-Based?**

1. **Range Query Performance:** Time-series data (orders, logs, events) is frequently queried by timestamp ranges. Range-based sharding allows these queries to hit 1-2 shards instead of all 100 shards (scatter-gather).

2. **Simple Split Logic:** When a range becomes too large, split at median key. No need to rehash and move data across all shards (as hash-based requires).

3. **Natural Ordering:** Related data (e.g., user's orders sorted by date) stays together on same shard, enabling efficient sequential scans.

### Rationale

**Range-Based Sharding Example:**

```
Table: orders (Primary Key: timestamp + order_id)

Shard 1: [2024-01-01...2024-01-31] → 10 GB, 5K QPS
Shard 2: [2024-02-01...2024-02-29] → 10 GB, 5K QPS
Shard 3: [2024-03-01...2024-03-31] → 10 GB, 25K QPS (HOT!)
...

Query: SELECT * FROM orders WHERE timestamp BETWEEN '2024-03-01' AND '2024-03-15'
→ Hits Shard 3 only (efficient)

Query: SELECT * FROM orders WHERE timestamp BETWEEN '2024-02-15' AND '2024-03-15'
→ Hits Shard 2 + Shard 3 (2 shards, still efficient)
```

**Hash-Based Sharding (NOT Chosen):**

```
Shard = hash(order_id) % 100

Query: SELECT * FROM orders WHERE timestamp BETWEEN '2024-03-01' AND '2024-03-15'
→ Scatter-gather to ALL 100 shards (slow!)
→ Gateway must merge results from 100 shards
→ 100× more network calls
```

**Hotspot Mitigation:**

1. **Monitoring:** Track QPS per range every 10 seconds
2. **Detection:** If range >10K QPS for 60 seconds → Flag hotspot
3. **Split:** Divide range at median key, create new Raft group
4. **Rebalance:** Move half the data to new group (background)

*See [pseudocode.md::split_range()](pseudocode.md) for implementation.*

### Implementation Details

**Metadata Schema:**

```sql
CREATE TABLE range_metadata (
    range_id BIGINT PRIMARY KEY,
    table_name VARCHAR(255),
    start_key BYTES NOT NULL,
    end_key BYTES NOT NULL,
    leader_node_id INT NOT NULL,
    replica_node_ids INT[] NOT NULL,
    size_mb INT NOT NULL,
    qps_last_min INT NOT NULL,
    last_split_time TIMESTAMP,
    INDEX idx_table_key (table_name, start_key, end_key)
);
```

**Split Thresholds:**
- **Size:** >64 MB (or 128 MB, 256 MB configurable)
- **QPS:** >10K QPS for 60 consecutive seconds
- **CPU:** >80% CPU on leader node for 5 minutes

**Split Process:**
```
1. Lock range for split (reads/writes continue)
2. Calculate median key (50th percentile)
3. Create new Raft group (allocate nodes)
4. Copy right half [median...end] to new group (background)
5. Dual-write phase: writes go to both groups (temporary)
6. Update metadata: two ranges now
7. Delete old data from original group
8. Release lock (split complete)
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Fast range queries (1-2 shards) | ❌ Hotspot risk (need monitoring + auto-split) |
| ✅ Simple split logic | ⚠️ Manual intervention if hotspot not detected |
| ✅ Natural ordering (related data together) | ❌ Uneven load if data not uniformly distributed |
| ✅ Predictable performance | - |

**Mitigation for Hotspots:**
- **Pre-splitting:** If know data will be hot (e.g., today's date), pre-split before load arrives
- **Composite Keys:** Use (shard_id, timestamp) as key to force even distribution
- **Read Replicas:** Add more followers to hot ranges (increase read capacity)

### When to Reconsider

**Use Hash-Based Sharding when:**
1. **No Range Queries:** Application only does point lookups (`WHERE id = X`)
2. **Uniform Access:** All keys accessed with equal probability (no hotspots)
3. **Simplicity:** Want predictable even distribution without monitoring

**Use Composite Sharding (Hash + Range) when:**
1. **Hybrid Workload:** Some queries are range, some are point
2. **Example:** `(hash(user_id), timestamp)` → Even distribution across users, range queries per user

**Indicators for Switching:**
- Hotspot mitigation failing (manual splits required daily)
- 80% of queries are scatter-gather (no range queries)
- Uneven load causing frequent rebalancing (operational overhead)

---

## 3. Two-Phase Commit vs Sagas for Distributed Transactions

### The Problem

Transaction spans multiple shards (e.g., transfer money from Account A on Shard 1 to Account B on Shard 2). Need atomic guarantee: either both shards commit or both abort. Cannot allow partial updates (money disappears or duplicates).

**Requirements:**
- **Atomicity:** All-or-nothing guarantee
- **Isolation:** Concurrent transactions don't interfere
- **Performance:** Low latency (<50ms)
- **Failure Handling:** Handle node crashes gracefully

### Options Considered

| Factor | Two-Phase Commit (2PC) | Sagas | No Coordination (Best Effort) |
|--------|---------------------|-------|----------------------------|
| **Atomicity** | ✅ Strong (all-or-nothing) | ⚠️ Eventual (compensations) | ❌ Weak (partial updates possible) |
| **Isolation** | ✅ Locks held during 2PC | ❌ No isolation (dirty reads) | ❌ No isolation |
| **Latency** | ⚠️ 20-50ms (2 RTTs + locks) | ✅ Low (async steps) | ✅ Very low (no coordination) |
| **Failure Handling** | ⚠️ Complex (coordinator crash) | ✅ Simple (retry compensations) | ❌ Manual intervention |
| **Throughput** | ⚠️ Limited (lock contention) | ✅ High (no locks) | ✅ Very high |

### Decision Made

**Use Two-Phase Commit (2PC) for strong consistency (ACID) with lock-based isolation.**

**Why 2PC over Sagas?**

1. **Strong Consistency Requirement:** Distributed SQL database must provide ACID guarantees. Sagas provide eventual consistency, which violates ACID.

2. **Isolation:** 2PC holds locks during transaction, preventing dirty reads. Sagas allow concurrent transactions to see intermediate states (dirty reads).

3. **User Expectation:** SQL databases are expected to provide serializable isolation. Sagas cannot provide this without application-level coordination.

4. **Simplicity for Application:** With 2PC, application doesn't need to write compensation logic. With Sagas, every transaction needs custom rollback code.

### Rationale

**Two-Phase Commit Flow:**

```
Transaction: Transfer $50 from A to B

BEGIN TRANSACTION
  UPDATE accounts SET balance = balance - 50 WHERE id = A;  -- Shard 1
  UPDATE accounts SET balance = balance + 50 WHERE id = B;  -- Shard 2
COMMIT

Phase 1 (PREPARE):
  Coordinator → Shard 1: "Can you commit? Lock row A"
  Coordinator → Shard 2: "Can you commit? Lock row B"
  
  Shard 1: Check balance >= 50 → Vote YES (lock held)
  Shard 2: Check constraints → Vote YES (lock held)
  
Phase 2 (COMMIT):
  Coordinator: All voted YES → Send COMMIT to both
  
  Shard 1: Apply change (balance -= 50), release lock
  Shard 2: Apply change (balance += 50), release lock
  
  Coordinator → Client: SUCCESS
```

**Saga Flow (NOT Chosen for ACID Systems):**

```
Step 1: Deduct from A (Shard 1) → SUCCESS
Step 2: Add to B (Shard 2) → FAILURE (crash)

Compensation: Refund to A (Shard 1)

Problem: Between Step 1 and Compensation, A's balance is $50 less (visible to other transactions)
→ Dirty read! Violates isolation!
```

*See [pseudocode.md::two_phase_commit()](pseudocode.md) for detailed implementation.*

### Implementation Details

**Coordinator State Machine:**

```
PREPARE Phase:
  - Send PREPARE to all participants
  - Wait for all votes (with timeout: 30 seconds)
  - If all YES → Proceed to COMMIT
  - If any NO or timeout → Proceed to ABORT

COMMIT Phase:
  - Send COMMIT to all participants
  - Wait for all ACKs (retry forever)
  - Log commit success to coordinator log
  - Return to client

ABORT Phase:
  - Send ABORT to all participants
  - Wait for all ACKs (retry with timeout)
  - Log abort to coordinator log
  - Return error to client
```

**Failure Scenarios:**

| Failure | 2PC Handling | Saga Handling |
|---------|-------------|--------------|
| **Participant crashes after YES** | Coordinator retries COMMIT forever (participant recovers from WAL) | Compensating transaction may fail |
| **Coordinator crashes** | Participants timeout, ABORT (conservative) | No coordinator, continue or manual fix |
| **Network partition** | Some participants commit, some timeout (use fencing tokens) | Compensations may not reach all |

**Optimization: Presumed Abort**

- Default assumption: If coordinator crashes, participants ABORT after timeout
- Reduces coordinator logging (only log COMMIT, not ABORT)
- Conservative but safe

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Strong consistency (ACID) | ❌ Higher latency (20-50ms locks held) |
| ✅ Isolation (no dirty reads) | ❌ Lower throughput (lock contention) |
| ✅ Simplicity (no compensation logic) | ⚠️ Coordinator is potential bottleneck |
| ✅ User expectation (SQL semantics) | ❌ Cannot handle external systems (payment APIs) |

**When Sagas Are Better:**

- **External Systems:** Payment APIs, email services (cannot hold locks)
- **Long-Running Transactions:** Hours or days (cannot hold locks that long)
- **High Throughput:** >100K transactions/sec (2PC lock contention)

### When to Reconsider

**Use Sagas when:**
1. **Microservices:** Transactions span independent services (can't hold locks across services)
2. **External APIs:** Payment processors, shipping providers (2PC not supported)
3. **Eventual Consistency OK:** Application tolerates temporary inconsistency
4. **High Throughput:** Need >100K transactions/sec (2PC locks limit throughput)

**Use Best-Effort (No Coordination) when:**
1. **Analytics:** Eventual consistency sufficient (dashboards, reports)
2. **Idempotent Operations:** Can safely retry (e.g., increment counter)

**Indicators for Switching:**
- Lock contention causing >5% transaction abort rate
- External system integration required (e.g., Stripe payments)
- Latency >100ms becoming user-facing issue

---

## 4. Synchronous Replication vs Asynchronous Replication

### The Problem

Data must be replicated across multiple nodes for fault tolerance. Synchronous replication waits for replicas to acknowledge (strong consistency, higher latency). Asynchronous replication doesn't wait (lower latency, risk of data loss).

**Requirements:**
- **Durability:** Data not lost if node crashes
- **Consistency:** All nodes have same data
- **Performance:** Low write latency

### Options Considered

| Factor | Synchronous Replication (Raft) | Asynchronous Replication | Semi-Synchronous |
|--------|----------------------------|------------------------|-----------------|
| **Data Loss Risk** | ✅ Zero (quorum must ACK) | ❌ High (replica lag) | ⚠️ Low (1 replica must ACK) |
| **Write Latency** | ⚠️ Higher (wait for ACKs) | ✅ Lower (no waiting) | ⚠️ Medium (wait for 1 ACK) |
| **Consistency** | ✅ Strong (immediate) | ❌ Eventual (lag) | ⚠️ Strong (after 1 ACK) |
| **Throughput** | ⚠️ Limited (quorum wait) | ✅ High (no waiting) | ⚠️ Medium |
| **Failover** | ✅ Simple (replica has all data) | ❌ Complex (replica may be behind) | ⚠️ Moderate (1 replica up-to-date) |

### Decision Made

**Use Synchronous Replication (Raft Consensus) for strong consistency and zero data loss.**

**Why Synchronous?**

1. **Data Durability:** Once write returns success to client, guaranteed durable (quorum of replicas have it). With async, write might be lost if leader crashes before replicating.

2. **Consistency Guarantee:** All committed writes are visible to subsequent reads (linearizability). Async replication allows stale reads.

3. **Simplified Failover:** Any follower with committed log can become leader. With async, must determine which replica has latest data (complex).

4. **User Expectation:** SQL database users expect strong consistency. Async would violate this.

### Rationale

**Synchronous Replication (Raft):**

```
Client → Leader: Write X
Leader: Append to log (uncommitted)
Leader → Follower 1: AppendEntries(X)
Leader → Follower 2: AppendEntries(X)

Follower 1: Append X, ACK
Follower 2: Append X, ACK

Leader: Received 2 out of 3 ACKs → COMMITTED
Leader → Client: SUCCESS (write is durable)

Latency: 1-5ms (single region), 50-200ms (multi-region)
Guarantee: X will NEVER be lost, even if leader crashes now
```

**Asynchronous Replication (NOT Chosen):**

```
Client → Leader: Write X
Leader: Append to log, apply to RocksDB
Leader → Client: SUCCESS (write is NOT yet replicated!)

Leader → Follower 1: AppendEntries(X) [non-blocking]
Leader → Follower 2: AppendEntries(X) [non-blocking]

Latency: 1ms (no waiting)
Risk: If leader crashes before followers receive X, X is LOST!
```

*See [pseudocode.md::raft_replicate_sync()](pseudocode.md) for implementation.*

### Implementation Details

**Quorum Calculation:**

```
Replication Factor (RF) = 3
Quorum (Q) = floor(RF / 2) + 1 = floor(3 / 2) + 1 = 2

For RF=3: Need 2 ACKs (leader + 1 follower)
For RF=5: Need 3 ACKs (leader + 2 followers)
```

**Performance Optimization: Pipeline Replication**

```
Don't wait for each write individually:

Bad:
  Write 1 → Wait for ACKs (2ms)
  Write 2 → Wait for ACKs (2ms)
  Write 3 → Wait for ACKs (2ms)
  Total: 6ms for 3 writes

Good (Pipelined):
  Write 1 → Send to followers
  Write 2 → Send to followers
  Write 3 → Send to followers
  Wait for all ACKs (2ms)
  Total: 2ms for 3 writes
  
Throughput: 3× improvement
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Zero data loss (quorum durability) | ❌ Higher write latency (1-5ms → 50-200ms multi-region) |
| ✅ Strong consistency (linearizable) | ❌ Lower write throughput (wait for quorum) |
| ✅ Simple failover (any follower can lead) | ⚠️ Network latency bottleneck |
| ✅ User expectation (SQL semantics) | - |

**Real-World Tradeoff:**

- **Google Spanner:** Synchronous replication globally (200ms writes) for strong consistency
- **MySQL Async Replication:** Asynchronous for low latency (1ms writes) but risk of data loss

### When to Reconsider

**Use Asynchronous Replication when:**
1. **Low Latency Critical:** Application needs <5ms writes (e.g., gaming leaderboards)
2. **Read-Heavy:** 99% reads, 1% writes (async replication for reads, eventual consistency OK)
3. **Eventual Consistency OK:** Analytics, dashboards (don't need immediate consistency)

**Use Semi-Synchronous when:**
1. **Moderate Durability:** Can tolerate losing recent writes (last 1-2 seconds)
2. **Moderate Latency:** Need <10ms writes (faster than full sync)

**Indicators for Switching:**
- Write latency >100ms causing user complaints
- Application can tolerate eventual consistency (after refactor)
- Write throughput limited by replication (need >100K writes/sec)

---

## 5. Strong Consistency vs Eventual Consistency

### The Problem

Distributed systems must choose between strong consistency (all nodes see same data immediately) and eventual consistency (nodes eventually converge, but may differ temporarily). Strong consistency provides better user experience but limits availability and performance (CAP theorem).

**CAP Theorem:** Can't have all three: Consistency, Availability, Partition Tolerance. Must choose 2.

### Options Considered

| Factor | Strong Consistency (CP) | Eventual Consistency (AP) | Causal Consistency |
|--------|----------------------|-------------------------|-------------------|
| **Read Consistency** | ✅ Always latest value | ❌ May read stale value | ⚠️ Causally related operations ordered |
| **Write Latency** | ⚠️ Higher (quorum wait) | ✅ Lower (single node) | ⚠️ Medium |
| **Availability** | ⚠️ Lower (partition stops writes) | ✅ Higher (partition continues) | ⚠️ Medium |
| **Conflict Resolution** | ✅ Not needed (single version) | ❌ Required (last-write-wins, CRDTs) | ⚠️ Some conflicts |
| **User Experience** | ✅ Predictable (read-your-writes) | ❌ Confusing (stale reads) | ⚠️ Moderate |

### Decision Made

**Use Strong Consistency (CP system) with configurable read isolation levels.**

**Why Strong Consistency?**

1. **User Expectation:** SQL databases are expected to provide strong consistency. Users expect to read their own writes immediately (read-your-writes consistency).

2. **No Conflict Resolution:** With strong consistency, only one version of data exists. No need for application to handle conflicts (last-write-wins, CRDTs, manual merges).

3. **Simplified Application Logic:** Application doesn't need to handle stale reads or implement retry logic. Reads always return latest committed value.

4. **Financial Correctness:** For financial applications (banks, payment processors), strong consistency is mandatory. Eventual consistency can lead to incorrect balances.

### Rationale

**Strong Consistency (Linearizability):**

```
Time   Client A                Client B
0ms    Write X = 1             -
5ms    Write committed         -
10ms   -                       Read X → Returns 1 (latest)
15ms   Write X = 2             -
20ms   -                       Read X → Returns 2 (latest)

Guarantee: Reads always return the latest committed value
No stale reads, no conflicts
```

**Eventual Consistency (NOT Chosen):**

```
Time   Client A (writes to Node 1)   Client B (reads from Node 2)   Node 1   Node 2
0ms    Write X = 1                    -                              X=1      X=0
5ms    -                              Read X → Returns 0 (stale!)    X=1      X=0
10ms   -                              -                              X=1      X=1 (replicated)
15ms   Write X = 2                    -                              X=2      X=1
20ms   -                              Read X → Returns 1 (stale!)    X=2      X=2 (replicated)

Problem: Client B reads stale values (confusing user experience)
```

*See [pseudocode.md::linearizable_read()](pseudocode.md) for implementation.*

### Implementation Details

**Consistency Levels (Configurable):**

1. **Linearizable (Strictest):**
   - Read from leader only
   - Guarantees: Reads reflect all writes up to this point
   - Latency: Higher (cross-region if leader far)
   - Use case: Financial transactions

2. **Snapshot Isolation:**
   - Read from any replica using timestamp
   - Guarantees: Consistent snapshot at timestamp T
   - Latency: Lower (local replica)
   - Use case: 90% of application reads

3. **Read Committed:**
   - Read only committed values (no dirty reads)
   - May see older committed values (not latest)
   - Latency: Lowest
   - Use case: Analytics, dashboards

**Implementation:**

```sql
-- Linearizable read (latest value, read from leader)
SELECT * FROM users WHERE id = 123;

-- Snapshot read (consistent snapshot, read from follower)
SELECT * FROM users WHERE id = 123 AS OF SYSTEM TIME '1 minute ago';

-- Read committed (may be stale, read from any replica)
SELECT * FROM users WHERE id = 123 WITH CONSISTENCY READ_COMMITTED;
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Predictable behavior (read-your-writes) | ❌ Lower availability during partitions |
| ✅ No conflict resolution (simpler app) | ❌ Higher read latency (cross-region to leader) |
| ✅ Financial correctness (ACID) | ⚠️ Cannot write during partition (minority side) |
| ✅ SQL semantics (user expectation) | - |

**CAP Theorem Choice: CP (Consistency + Partition Tolerance)**

- We choose **C**onsistency + **P**artition Tolerance
- We sacrifice **A**vailability during partitions (minority partition cannot accept writes)

**Example:**

```
3-node cluster partitioned:
  Partition A: 2 nodes (majority) → Can accept writes ✅
  Partition B: 1 node (minority) → Reject writes ❌ (no quorum)

This is intentional: Strong consistency requires majority quorum
```

### When to Reconsider

**Use Eventual Consistency (AP) when:**
1. **High Availability Critical:** System must accept writes even during partitions
2. **Geo-Distributed Writes:** Users in multiple regions need to write locally (<10ms)
3. **Conflict Resolution OK:** Application can handle last-write-wins or CRDTs
4. **Use Case:** Social media (likes, comments), shopping carts, collaborative editing

**Examples:**
- **DynamoDB:** Eventual consistency by default (but offers strong consistency option)
- **Cassandra:** Tunable consistency (can choose AP or CP per query)
- **Riak:** Eventually consistent with CRDTs for conflict resolution

**Use Causal Consistency when:**
1. **Middle Ground:** Need some ordering but not full linearizability
2. **Example:** "Reply must come after original post" (causal), but different threads can diverge (eventual)

**Indicators for Switching:**
- Availability during partitions more important than consistency
- Cross-region write latency (200ms) unacceptable for users
- Application can handle conflicts (e.g., CRDT for collaborative editing)

---

## 6. LSM Tree vs B-Tree for Storage Engine

### The Problem

Need storage engine for each node to persist data on disk. Storage engine affects write throughput, read latency, and space amplification.

**Requirements:**
- **Write-Heavy:** Distributed databases often write-heavy (replication overhead)
- **Range Queries:** Support efficient range scans
- **Durability:** WAL for crash recovery

### Options Considered

| Factor | LSM Tree (LevelDB, RocksDB) | B-Tree (InnoDB, PostgreSQL) | Bw-Tree (Latch-Free) |
|--------|--------------------------|---------------------------|-------------------|
| **Write Throughput** | ✅ Very high (append-only) | ⚠️ Moderate (in-place update) | ✅ High (latch-free) |
| **Read Latency** | ⚠️ Higher (check multiple levels) | ✅ Lower (single lookup) | ✅ Low (cache-optimized) |
| **Space Amplification** | ⚠️ Higher (multiple versions) | ✅ Lower (in-place update) | ✅ Low |
| **Compaction Overhead** | ❌ Significant (background I/O) | ✅ None (in-place) | ⚠️ Moderate |
| **Range Scans** | ✅ Efficient (sorted SSTables) | ✅ Efficient (B-tree order) | ✅ Efficient |
| **Crash Recovery** | ✅ WAL + SSTables | ✅ WAL + B-tree | ✅ WAL |

### Decision Made

**Use LSM Tree (RocksDB) for write-optimized performance.**

**Why LSM Tree?**

1. **Write Throughput:** Distributed database has high write load due to replication (3× write amplification). LSM tree's append-only structure provides 5-10× higher write throughput than B-tree.

2. **Replication Overhead:** Every write replicated to 3 nodes. LSM tree handles this better (sequential writes faster than random writes).

3. **Compaction:** Modern LSM implementations (RocksDB) have efficient compaction strategies (leveled, tiered) that minimize impact on foreground queries.

4. **Industry Standard:** CockroachDB, TiDB, YugabyteDB all use RocksDB (proven at scale).

### Rationale

**LSM Tree Write Path:**

```
1. Write to WAL (append-only file) → Durability
2. Write to Memtable (in-memory sorted tree)
3. When Memtable full (64 MB) → Flush to SSTable (sorted file on disk)
4. Background: Compact SSTables (merge and sort)

Write Latency: 1-2ms (memory write + WAL)
Write Throughput: 100K writes/sec per node
```

**B-Tree Write Path (NOT Chosen for Write-Heavy):**

```
1. Write to WAL (append-only file) → Durability
2. Find page in B-tree (disk seek if not in cache)
3. Update page in-place (disk write)
4. If page full → Split page (expensive)

Write Latency: 5-10ms (disk seek + write)
Write Throughput: 10K writes/sec per node
```

*See [pseudocode.md::lsm_tree_write()](pseudocode.md) for implementation.*

### Implementation Details

**RocksDB Configuration:**

```
Memtable size: 64 MB
Number of memtables: 4 (allows buffering during flush)
SSTable size: 64 MB (Level 0), 256 MB (Level 1+)
Compaction style: Leveled (lower read amplification)
Block cache: 10 GB (cache hot SSTables in memory)
Bloom filters: Yes (reduce read amplification)
```

**Compaction Strategies:**

1. **Leveled Compaction:**
   - Level 0: 4 SSTables (overlapping keys OK)
   - Level 1: 256 MB (no overlapping keys)
   - Level 2: 2.56 GB (no overlapping keys)
   - Each level 10× larger than previous
   - **Read Amplification:** Log(N) levels (typically 4-5)
   - **Write Amplification:** 10-30× (data rewritten multiple times)

2. **Tiered Compaction:**
   - Each level has multiple SSTables with overlapping keys
   - **Read Amplification:** Higher (check multiple SSTables per level)
   - **Write Amplification:** Lower (less rewriting)
   - **Use case:** Write-heavy workloads

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ High write throughput (100K writes/sec) | ❌ Higher read latency (check multiple levels) |
| ✅ Sequential disk I/O (fast) | ❌ Compaction overhead (background CPU/disk) |
| ✅ Efficient range scans (sorted SSTables) | ⚠️ Space amplification (1.5-2× due to multiple versions) |
| ✅ Proven at scale (RocksDB in production) | - |

**Mitigation for Read Latency:**
- **Block Cache:** Cache hot SSTables in memory (10 GB cache → 90% cache hit rate)
- **Bloom Filters:** Probabilistic data structure to skip SSTables that don't contain key (reduces read amplification from 10× to 2×)

### When to Reconsider

**Use B-Tree when:**
1. **Read-Heavy:** 99% reads, 1% writes (B-tree has lower read latency)
2. **In-Place Updates:** Frequently update same keys (B-tree avoids multiple versions)
3. **Space-Constrained:** Cannot afford 1.5-2× space amplification

**Use Bw-Tree when:**
1. **Multi-Core:** Want to utilize many CPU cores (latch-free design)
2. **In-Memory:** Database fits entirely in memory (no disk I/O bottleneck)

**Indicators for Switching:**
- Read latency >50ms causing user complaints (despite caching)
- Compaction consuming >30% of disk I/O (starving foreground queries)
- Space amplification >3× (running out of disk)

---

## 7. Multi-Region Deployment Strategies

### The Problem

Need to deploy distributed database across multiple geographic regions (US, EU, APAC) for:
- **Low Latency:** Users read from nearest region (<50ms)
- **High Availability:** Survive regional failures (datacenter down)
- **Data Residency:** Comply with GDPR, data sovereignty laws

**Challenge:** Balance between consistency (cross-region consensus slow) and latency (users want <50ms).

### Options Considered

| Factor | Regional Raft Groups | Global Raft Groups | Multi-Master (Async) |
|--------|-------------------|------------------|-------------------|
| **Write Latency** | ✅ Low (in-region: 10ms) | ❌ High (cross-region: 200ms) | ✅ Low (local: 10ms) |
| **Consistency** | ✅ Strong (per region) | ✅ Strong (global) | ❌ Eventual (conflicts) |
| **Cross-Region Txn** | ⚠️ Slow (2PC: 200ms) | ✅ Native (200ms) | ❌ Complex (conflict resolution) |
| **Data Residency** | ✅ Easy (data stays in region) | ⚠️ Hard (data replicated globally) | ✅ Easy |
| **Availability** | ✅ High (region fails, others continue) | ⚠️ Lower (majority of regions needed) | ✅ Very high |

### Decision Made

**Use Regional Raft Groups with cross-region replication for critical data.**

**Why Regional Raft Groups?**

1. **Low Latency for Regional Data:** 80% of data is region-specific (e.g., US users' profiles). These writes are 10ms (in-region consensus), not 200ms (global consensus).

2. **Data Residency Compliance:** GDPR requires EU user data stay in EU. Regional Raft groups naturally enforce this (EU data on EU nodes).

3. **High Availability:** If US region fails, EU and APAC regions continue operating independently (no global quorum required).

4. **Incremental Complexity:** Start with regional groups (simple), add cross-region replication only for critical data (gradual).

### Rationale

**Regional Raft Groups Architecture:**

```
US Region (Virginia):
  - User data: US users
  - Raft Group: 3 nodes in US
  - Write latency: 10ms (in-region consensus)
  
EU Region (Frankfurt):
  - User data: EU users
  - Raft Group: 3 nodes in EU
  - Write latency: 10ms (in-region consensus)
  
APAC Region (Singapore):
  - User data: APAC users
  - Raft Group: 3 nodes in APAC
  - Write latency: 10ms (in-region consensus)

Global Data (Financial transactions, audit logs):
  - Raft Group: 1 node per region (3 total)
  - Write latency: 200ms (cross-region consensus)
  - Use case: <20% of writes
```

**Data Placement Strategy:**

```sql
-- Regional table (data stays in user's region)
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    name VARCHAR(255),
    region VARCHAR(10)
) PARTITION BY LIST (region);

-- Global table (replicated across all regions)
CREATE TABLE audit_log (
    log_id UUID PRIMARY KEY,
    user_id UUID,
    action VARCHAR(255),
    timestamp TIMESTAMP
) WITH (replication = 'global');
```

*See [pseudocode.md::route_to_region()](pseudocode.md) for implementation.*

### Implementation Details

**Cross-Region Read Strategy:**

```
User in EU reads US user's profile:

Option 1 (Strong Consistency):
  - EU Gateway → US Raft Leader (100ms cross-region)
  - Return latest value
  - Latency: 100ms

Option 2 (Eventual Consistency):
  - EU Gateway → EU Raft Follower (async replica of US data)
  - Return cached value (may be 1-5 seconds stale)
  - Latency: 10ms
  - Use case: Social media profiles, dashboards

Default: Strong consistency for reads, configurable
```

**Cross-Region Transaction (2PC):**

```
Transaction spans US and EU:

BEGIN TRANSACTION
  UPDATE accounts SET balance = balance - 50 WHERE user_id = US_USER;  -- US region
  UPDATE accounts SET balance = balance + 50 WHERE user_id = EU_USER;  -- EU region
COMMIT

Phase 1 (PREPARE):
  Coordinator → US Raft Leader: PREPARE (100ms)
  Coordinator → EU Raft Leader: PREPARE (100ms)
  Total: 100ms (parallel)

Phase 2 (COMMIT):
  Coordinator → US Raft Leader: COMMIT (100ms)
  Coordinator → EU Raft Leader: COMMIT (100ms)
  Total: 100ms (parallel)

Total Latency: 200ms (2 RTTs cross-region)
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ Low latency for regional data (10ms: 80% of writes) | ❌ Higher latency for cross-region txn (200ms: 20% of writes) |
| ✅ Data residency compliance (GDPR) | ⚠️ Complex routing logic (which region owns data?) |
| ✅ High availability (region failure isolated) | ❌ Cross-region conflict resolution (if regions partition) |
| ✅ Incremental rollout (start with 1 region) | - |

**Real-World Examples:**

- **Google Spanner:** Global Raft groups (prioritizes strong consistency, accepts 200ms writes)
- **CockroachDB:** Regional Raft groups (configurable per table)
- **YugabyteDB:** Regional primary, async replicas in other regions

### When to Reconsider

**Use Global Raft Groups when:**
1. **Strong Consistency Critical:** Financial transactions, audit logs (cannot tolerate eventual consistency)
2. **Cross-Region Txn Common:** >50% of transactions span regions
3. **Latency Acceptable:** Users OK with 200ms writes (B2B applications)

**Use Multi-Master (Async) when:**
1. **Offline Support:** Mobile apps need to work offline
2. **Eventual Consistency OK:** Social media, collaborative editing
3. **Conflict Resolution Acceptable:** CRDTs or last-write-wins

**Indicators for Switching:**
- >50% of writes are cross-region (regional groups lose benefit)
- Data residency laws change (can replicate globally)
- Users complaining about 200ms cross-region transaction latency

---

## Summary Comparison

| Decision | What We Chose | Why | Trade-off Accepted |
|----------|--------------|-----|-------------------|
| **Consensus** | Raft | Understandable, strong leader, mature implementations | Slightly lower perf than Paxos (rare) |
| **Sharding** | Range-Based | Efficient range queries, simple split logic | Hotspot risk (need auto-split) |
| **Distributed Txn** | Two-Phase Commit | Strong consistency (ACID), no compensation logic | Higher latency (20-50ms locks) |
| **Replication** | Synchronous (Raft) | Zero data loss, strong consistency | Higher write latency (quorum wait) |
| **Consistency Model** | Strong (CP) | Predictable, no conflicts, SQL semantics | Lower availability during partitions |
| **Storage Engine** | LSM Tree (RocksDB) | High write throughput, proven at scale | Higher read latency (compaction) |
| **Multi-Region** | Regional Raft Groups | Low latency (80% of writes), data residency | Cross-region txn slow (200ms) |

**Key Insight:** Distributed SQL database prioritizes **consistency** and **correctness** over absolute **performance**. Trade-offs favor strong guarantees (ACID, durability) at the cost of higher latency and lower availability during failures.

