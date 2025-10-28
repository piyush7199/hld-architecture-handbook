# Distributed Database - High-Level Design Diagrams

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Range-Based Sharding](#2-range-based-sharding)
3. [Raft Consensus Group](#3-raft-consensus-group)
4. [Write Path with Consensus](#4-write-path-with-consensus)
5. [Read Path (Leader vs Follower)](#5-read-path-leader-vs-follower)
6. [Dynamic Range Splitting](#6-dynamic-range-splitting)
7. [Two-Phase Commit (2PC) Flow](#7-two-phase-commit-2pc-flow)
8. [Timestamp Oracle Architecture](#8-timestamp-oracle-architecture)
9. [Metadata Store (etcd) Architecture](#9-metadata-store-etcd-architecture)
10. [Leader Election Flow](#10-leader-election-flow)
11. [Split Brain Prevention](#11-split-brain-prevention)
12. [Multi-Region Deployment](#12-multi-region-deployment)
13. [Hotspot Detection and Mitigation](#13-hotspot-detection-and-mitigation)
14. [Online Schema Migration](#14-online-schema-migration)

---

## 1. System Architecture Overview

**Flow Explanation:**

This diagram shows the complete architecture of a distributed database. Clients send SQL queries to stateless gateways, which consult the metadata store (etcd) to determine which Raft consensus group owns the data. The gateway then routes the request to the appropriate leader node, which coordinates with its followers using Raft consensus before committing the write.

**Components:**

1. **Client Layer:** Applications using SQL drivers
2. **Gateway/Router:** Stateless SQL parser and router
3. **Metadata Store:** etcd cluster storing range assignments
4. **Raft Groups:** Collections of 3 nodes forming quorums
5. **Storage Layer:** RocksDB (LSM tree) with WAL

**Key Characteristics:**

- **Stateless Gateways:** Can scale horizontally (add more gateways)
- **Metadata Caching:** Gateways cache range mappings (99% hit rate)
- **Automatic Sharding:** Data distributed across Raft groups by key range
- **Fault Tolerance:** Each Raft group survives 1 node failure (quorum=2/3)

**Performance:**

- Gateway routing: <1ms (cache hit)
- Cross-region metadata lookup: 50ms (cache miss)
- Raft consensus latency: 1-5ms (single region), 50-200ms (multi-region)

```mermaid
graph TB
    subgraph Clients
        App1[Application 1<br/>SQL Driver]
        App2[Application 2<br/>SQL Driver]
        App3[Application N<br/>SQL Driver]
    end
    
    subgraph Gateway Layer
        GW1[Gateway 1<br/>Stateless SQL Parser]
        GW2[Gateway 2<br/>Stateless SQL Parser]
        GW3[Gateway N<br/>Stateless SQL Parser]
        
        Cache[Metadata Cache<br/>range_id to leader_node<br/>TTL: 60 seconds]
    end
    
    subgraph Metadata Store
        etcd1[etcd Node 1<br/>Leader]
        etcd2[etcd Node 2<br/>Follower]
        etcd3[etcd Node 3<br/>Follower]
    end
    
    subgraph Raft Group A - Range A-F
        LA[Leader Node 1<br/>RocksDB Storage]
        FA1[Follower Node 2]
        FA2[Follower Node 3]
    end
    
    subgraph Raft Group B - Range G-M
        LB[Leader Node 4<br/>RocksDB Storage]
        FB1[Follower Node 5]
        FB2[Follower Node 6]
    end
    
    subgraph Raft Group C - Range N-Z
        LC[Leader Node 7<br/>RocksDB Storage]
        FC1[Follower Node 8]
        FC2[Follower Node 9]
    end
    
    App1 --> GW1
    App2 --> GW2
    App3 --> GW3
    
    GW1 --> Cache
    GW2 --> Cache
    GW3 --> Cache
    
    Cache -.->|Cache miss| etcd1
    
    GW1 -->|Route by key range| LA
    GW1 -->|Route by key range| LB
    GW1 -->|Route by key range| LC
    
    LA -->|Raft AppendEntries| FA1
    LA -->|Raft AppendEntries| FA2
    
    LB -->|Raft AppendEntries| FB1
    LB -->|Raft AppendEntries| FB2
    
    LC -->|Raft AppendEntries| FC1
    LC -->|Raft AppendEntries| FC2
    
    etcd1 -->|Replication| etcd2
    etcd1 -->|Replication| etcd3
    
    style LA fill:#95e1d3
    style LB fill:#95e1d3
    style LC fill:#95e1d3
    style etcd1 fill:#ffd93d
```

---

## 2. Range-Based Sharding

**Flow Explanation:**

This diagram illustrates how data is partitioned by key ranges rather than hash. A table with primary key `user_id` (UUID) is split into 3 ranges based on the UUID value. Each range is assigned to a different Raft consensus group. Range-based sharding enables efficient range queries but can suffer from hotspots if recent data is accessed more frequently.

**Sharding Logic:**

1. Extract primary key from query: `user_id = '7a3f2c1b...'`
2. Find range containing key: `[55555556...AAAAAAAA]` → Range 2
3. Route query to Raft Group B (Nodes 4, 5, 6)

**Benefits:**

- **Efficient Range Queries:** `WHERE user_id BETWEEN X AND Y` hits single range (not all shards)
- **Simple Split Logic:** Split at midpoint of key range
- **Ordered Scans:** Can iterate over keys in sorted order

**Trade-offs:**

- **Hotspot Risk:** Recent data (e.g., today's orders) may concentrate in one range
- **Uneven Distribution:** If keys not uniformly distributed, some ranges larger than others

**Mitigation:**

- Monitor QPS per range, auto-split hot ranges
- Use composite keys (e.g., `shard_id + user_id`) for better distribution

```mermaid
graph TB
    subgraph Table users - 1 Billion Rows
        PK[Primary Key: user_id UUID<br/>Example values:<br/>00000000-aaaa-bbbb-cccc-dddddddddddd<br/>7a3f2c1b-xxxx-yyyy-zzzz-111111111111<br/>ffffffff-9999-8888-7777-666666666666]
    end
    
    subgraph Range Partitioning
        Range1[Range 1<br/>start_key: 00000000...<br/>end_key: 55555555...<br/>333M rows<br/>100 GB]
        
        Range2[Range 2<br/>start_key: 55555556...<br/>end_key: AAAAAAAA...<br/>333M rows<br/>100 GB]
        
        Range3[Range 3<br/>start_key: AAAAAAAB...<br/>end_key: FFFFFFFF...<br/>334M rows<br/>100 GB]
    end
    
    subgraph Physical Nodes
        RG1[Raft Group A<br/>Nodes 1 2 3<br/>Leader: Node 1]
        RG2[Raft Group B<br/>Nodes 4 5 6<br/>Leader: Node 4]
        RG3[Raft Group C<br/>Nodes 7 8 9<br/>Leader: Node 7]
    end
    
    subgraph Query Examples
        Q1[Query: SELECT * FROM users<br/>WHERE user_id = '7a3f2c1b...'<br/>Routes to: Range 2]
        
        Q2[Query: SELECT * FROM users<br/>WHERE user_id BETWEEN '00000000...' AND '60000000...'<br/>Routes to: Range 1 + Range 2 scatter-gather]
    end
    
    PK --> Range1
    PK --> Range2
    PK --> Range3
    
    Range1 --> RG1
    Range2 --> RG2
    Range3 --> RG3
    
    Q1 -.->|Single range hit| RG2
    Q2 -.->|Two range hit| RG1
    Q2 -.->|Two range hit| RG2
    
    style Range1 fill:#a8dadc
    style Range2 fill:#a8dadc
    style Range3 fill:#a8dadc
    style RG1 fill:#95e1d3
    style RG2 fill:#95e1d3
    style RG3 fill:#95e1d3
```

---

## 3. Raft Consensus Group

**Flow Explanation:**

A Raft consensus group consists of 3 nodes: 1 leader and 2 followers. The leader accepts writes, appends them to its log, and replicates to followers. Once a quorum (2 out of 3) acknowledges the write, it's committed. This ensures that even if 1 node fails, the data is safe. Followers can serve reads with eventual consistency or snapshot reads.

**Raft Roles:**

1. **Leader:** Accepts writes, coordinates replication
2. **Follower:** Replicates leader's log, can serve reads
3. **Candidate:** Temporary state during election

**Replication Flow:**

1. Client writes to Leader
2. Leader appends to log (uncommitted)
3. Leader sends AppendEntries RPC to Followers
4. Followers append to log, ACK to Leader
5. Leader receives quorum ACKs (2/3) → Mark committed
6. Leader applies to state machine (RocksDB)
7. Leader notifies Followers to apply

**Performance:**

- **Single-region latency:** 1-5ms (1 RTT: Leader → Follower → Leader)
- **Multi-region latency:** 50-200ms (cross-region network)
- **Throughput:** 10K writes/sec per Raft group (leader bottleneck)

**Fault Tolerance:**

- Tolerates 1 node failure (2/3 quorum still forms)
- Cannot tolerate 2 node failures (no quorum)
- Leader election: ~1-2 seconds downtime

```mermaid
graph TB
    subgraph Raft Consensus Group - Range A-F
        Leader[LEADER Node 1<br/>Term: 5<br/>Log Index: 1000<br/>Committed Index: 999<br/>Role: Accepts writes coordinates replication]
        
        Follower1[FOLLOWER Node 2<br/>Term: 5<br/>Log Index: 1000<br/>Committed Index: 999<br/>Role: Replicates leader log]
        
        Follower2[FOLLOWER Node 3<br/>Term: 5<br/>Log Index: 999<br/>Committed Index: 998<br/>Role: Replicates leader log]
    end
    
    subgraph Log Replication
        LeaderLog[Leader Log:<br/>Index 997: SET x=1 Committed<br/>Index 998: SET y=2 Committed<br/>Index 999: SET z=3 Committed<br/>Index 1000: SET w=4 Uncommitted]
        
        Follower1Log[Follower 1 Log:<br/>Index 997: SET x=1 Committed<br/>Index 998: SET y=2 Committed<br/>Index 999: SET z=3 Committed<br/>Index 1000: SET w=4 Uncommitted]
        
        Follower2Log[Follower 2 Log:<br/>Index 997: SET x=1 Committed<br/>Index 998: SET y=2 Committed<br/>Index 999: SET z=3 Committed<br/>Index 1000: Missing lagging]
    end
    
    subgraph Quorum
        Q[Quorum Requirement:<br/>2 out of 3 nodes must ACK<br/>Current: Leader + Follower1 = 2<br/>Status: Quorum formed can commit]
    end
    
    Client[Client Write Request<br/>SET w=4]
    
    Client -->|1. Write| Leader
    Leader -->|2. AppendEntries RPC| Follower1
    Leader -->|2. AppendEntries RPC| Follower2
    Follower1 -->|3. ACK Success| Leader
    Follower2 -.->|3. ACK Delayed network| Leader
    
    Leader --> LeaderLog
    Follower1 --> Follower1Log
    Follower2 --> Follower2Log
    
    Leader --> Q
    
    style Leader fill:#95e1d3
    style Follower1 fill:#a8dadc
    style Follower2 fill:#ffd93d
    style Q fill:#95e1d3
```

---

## 4. Write Path with Consensus

**Flow Explanation:**

This diagram shows the complete write path from client to storage. The client sends a write to the gateway, which routes it to the appropriate Raft leader. The leader proposes the write to followers via AppendEntries RPC. Once a quorum (2/3) acknowledges, the leader commits the write to RocksDB and returns success. This provides strong consistency: once committed, the write is durable even if nodes fail.

**Steps:**

1. **Client → Gateway** (1ms): SQL statement parsed
2. **Gateway → Metadata** (1ms): Lookup range owner (cache hit 99%)
3. **Gateway → Leader** (1ms): Route to Raft leader
4. **Leader → Followers** (2ms): AppendEntries RPC to 2 followers
5. **Followers → Leader** (2ms): ACKs from followers
6. **Leader commits** (1ms): Apply to RocksDB, mark committed
7. **Leader → Gateway → Client** (2ms): Return success

**Total Latency:** 10ms (single region), 150ms (multi-region)

**Benefits:**

- **Strong Consistency:** Quorum ensures durability
- **Fault Tolerance:** Write survives 1 node failure
- **No Single Point of Failure:** Leader can failover

**Trade-offs:**

- **Higher Latency:** Consensus adds network RTTs
- **Write Amplification:** 3× network traffic (1 leader + 2 followers)
- **Leader Bottleneck:** All writes through single leader

```mermaid
sequenceDiagram
    participant Client
    participant Gateway
    participant Metadata
    participant Leader as Leader Node 1
    participant Follower1 as Follower Node 2
    participant Follower2 as Follower Node 3
    participant RocksDB as RocksDB Storage
    
    Client->>Gateway: 1. INSERT INTO users VALUES abc
    Gateway->>Metadata: 2. Lookup: Which range owns key abc?
    Metadata-->>Gateway: Range 1 Leader Node 1
    Gateway->>Leader: 3. Write request key=abc value=xyz
    
    Leader->>Leader: 4. Append to log uncommitted
    
    par Parallel Replication
        Leader->>Follower1: 5a. AppendEntries key=abc value=xyz term=5 index=1000
        Leader->>Follower2: 5b. AppendEntries key=abc value=xyz term=5 index=1000
    end
    
    par Follower ACKs
        Follower1->>Follower1: 6a. Append to log
        Follower1-->>Leader: 7a. ACK Success term=5 index=1000
        
        Follower2->>Follower2: 6b. Append to log
        Follower2-->>Leader: 7b. ACK Success term=5 index=1000
    end
    
    Leader->>Leader: 8. Quorum 2 out of 3 mark committed
    Leader->>RocksDB: 9. Apply to state machine key=abc value=xyz
    RocksDB-->>Leader: Write confirmed
    
    Leader-->>Gateway: 10. Success committed
    Gateway-->>Client: 11. INSERT confirmed
    
    Note over Leader,Follower2: Total Latency: 10ms single region 150ms multi-region
```

---

## 5. Read Path (Leader vs Follower)

**Flow Explanation:**

This diagram compares three read strategies: reading from leader (strong consistency), reading from follower (eventual consistency), and reading from follower with timestamp (snapshot isolation). Reading from leader provides linearizability but higher latency (cross-region). Reading from follower provides lower latency but may return stale data. Reading with timestamp allows strong consistency from followers by reading a historical snapshot.

**Strategies:**

1. **Read from Leader:**
   - Consistency: Strong (linearizable)
   - Latency: Higher (may be cross-region)
   - Use case: Financial transactions

2. **Read from Follower (Stale):**
   - Consistency: Eventual (may lag 100ms-1s)
   - Latency: Lower (nearest replica)
   - Use case: Analytics dashboards

3. **Read from Follower with Timestamp:**
   - Consistency: Strong (snapshot isolation at T-1min)
   - Latency: Lower (nearest replica)
   - Use case: Most application reads (90%)

**Performance Comparison:**

| Strategy | Latency | Consistency | QPS Capacity |
|----------|---------|-------------|--------------|
| Leader | 100ms | Strong | 30K reads/sec (1 node) |
| Follower (stale) | 10ms | Eventual | 90K reads/sec (3 nodes) |
| Follower (timestamp) | 15ms | Strong | 90K reads/sec (3 nodes) |

**Recommendation:** Use follower reads with timestamp for 90% of reads.

```mermaid
graph TB
    subgraph Query Type 1 - Read from Leader
        Q1[SELECT * FROM users WHERE id=X]
        G1[Gateway]
        L1[Leader Node 1<br/>US Region]
        
        Q1 -->|Route to leader| G1
        G1 -->|100ms cross-region| L1
        L1 -->|Return latest value| G1
        
        Result1[Result: x=5<br/>Consistency: Strong linearizable<br/>Latency: 100ms cross-region]
        G1 --> Result1
    end
    
    subgraph Query Type 2 - Read from Follower Stale
        Q2[SELECT * FROM users WHERE id=X]
        G2[Gateway]
        F2[Follower Node 2<br/>EU Region nearest]
        
        Q2 -->|Route to nearest| G2
        G2 -->|10ms local| F2
        F2 -->|Return stale value| G2
        
        Result2[Result: x=4 1 sec old<br/>Consistency: Eventual<br/>Latency: 10ms local]
        G2 --> Result2
    end
    
    subgraph Query Type 3 - Read from Follower with Timestamp
        Q3[SELECT * FROM users WHERE id=X<br/>AS OF SYSTEM TIME 1 min ago]
        G3[Gateway]
        F3[Follower Node 3<br/>EU Region nearest]
        
        Q3 -->|Route to nearest timestamp| G3
        G3 -->|15ms local| F3
        F3 -->|Return snapshot at T-1min| G3
        
        Result3[Result: x=4 at T-1min<br/>Consistency: Strong snapshot isolation<br/>Latency: 15ms local]
        G3 --> Result3
    end
    
    subgraph Comparison
        Recommendation[Recommendation:<br/>Use follower reads with timestamp<br/>90 percent of reads<br/>10x capacity lower latency strong consistency]
    end
    
    style L1 fill:#95e1d3
    style F2 fill:#a8dadc
    style F3 fill:#95e1d3
    style Recommendation fill:#ffd93d
```

---

## 6. Dynamic Range Splitting

**Flow Explanation:**

This diagram shows how a range is automatically split when it becomes too large or too hot. The system monitors each range's size and QPS. When a range exceeds thresholds (>64 MB or >10K QPS for 60 seconds), it's split at the median key. A new Raft group is created, and data is migrated. This prevents hotspots and distributes load evenly.

**Split Triggers:**

1. **Size-Based:** Range >64 MB (or configurable threshold)
2. **Load-Based:** Range >10K QPS for 60 consecutive seconds
3. **Manual:** Operator-initiated split

**Split Process:**

1. **Detect:** Monitor thread detects hotspot (QPS >10K)
2. **Decide:** Calculate median key to split at
3. **Create:** Create new Raft group for right half
4. **Migrate:** Copy right half data to new group (background)
5. **Update Metadata:** Update etcd with new range mapping
6. **Complete:** Delete old data from original group, serve from both

**Downtime:** Zero (reads/writes continue during split)

**Duration:** 30 seconds - 5 minutes (depends on range size)

```mermaid
graph TB
    subgraph Before Split - Overloaded Range
        Range_Old[Range 1 Original<br/>Key Range: A-Z<br/>Size: 100 MB exceeds 64 MB<br/>QPS: 50K exceeds 10K<br/>Status: OVERLOADED<br/>Raft Group A Nodes 1 2 3]
        
        Monitor[Monitoring Thread:<br/>Detects: size greater than 64 MB<br/>Detects: QPS greater than 10K for 60 sec<br/>Decision: Split range at median key M]
    end
    
    subgraph Split Operation
        Split[Split Coordinator:<br/>1. Calculate split point median key M<br/>2. Create new Raft Group B<br/>3. Migrate data N-Z to Group B<br/>4. Update metadata etcd<br/>5. Delete N-Z from Group A]
        
        NewGroup[Create New Raft Group B:<br/>Assign Nodes 4 5 6<br/>Initialize empty storage]
    end
    
    subgraph After Split - Balanced Ranges
        Range_Left[Range 1a Left<br/>Key Range: A-M<br/>Size: 50 MB under limit<br/>QPS: 25K manageable<br/>Status: HEALTHY<br/>Raft Group A Nodes 1 2 3]
        
        Range_Right[Range 1b Right<br/>Key Range: N-Z<br/>Size: 50 MB under limit<br/>QPS: 25K manageable<br/>Status: HEALTHY<br/>Raft Group B Nodes 4 5 6]
    end
    
    subgraph Metadata Update
        etcd_before[etcd Before:<br/>range_id=1 start=A end=Z leader=Node1]
        etcd_after[etcd After:<br/>range_id=1 start=A end=M leader=Node1<br/>range_id=2 start=N end=Z leader=Node4]
    end
    
    Range_Old --> Monitor
    Monitor --> Split
    Split --> NewGroup
    Split --> Range_Left
    NewGroup --> Range_Right
    
    etcd_before -.->|Update| etcd_after
    
    style Range_Old fill:#ff6b6b
    style Range_Left fill:#95e1d3
    style Range_Right fill:#95e1d3
    style Monitor fill:#ffd93d
```

---

## 7. Two-Phase Commit (2PC) Flow

**Flow Explanation:**

This diagram illustrates distributed transaction management using Two-Phase Commit (2PC). When a transaction spans multiple shards (e.g., transferring money between accounts on different shards), a coordinator ensures atomicity. In Phase 1 (PREPARE), all participants lock resources and vote YES/NO. In Phase 2 (COMMIT/ABORT), the coordinator tells all participants to commit (if all voted YES) or abort (if any voted NO).

**Phases:**

**Phase 1 (PREPARE):**
1. Coordinator sends "PREPARE" to all participant shards
2. Each shard locks affected rows, checks constraints
3. Each shard votes "YES" (can commit) or "NO" (must abort)
4. Coordinator waits for all votes

**Phase 2 (COMMIT/ABORT):**
1. If all votes YES → Coordinator sends "COMMIT" to all
2. If any vote NO → Coordinator sends "ABORT" to all
3. Participants execute command, release locks
4. Participants ACK to coordinator
5. Coordinator returns result to client

**Failure Scenarios:**

| Failure | Handling |
|---------|----------|
| Participant crashes after YES | Coordinator retries COMMIT until participant recovers (locks held) |
| Participant crashes before vote | Timeout → Coordinator ABORTs |
| Coordinator crashes | Participants timeout, ABORT (conservative) |

**Performance:**

- **Latency:** 20-50ms (2 network RTTs)
- **Throughput:** 5K transactions/sec (coordinator bottleneck)
- **Lock Duration:** 20-50ms (during 2PC)

```mermaid
sequenceDiagram
    participant Client
    participant Coordinator
    participant Shard1 as Shard 1 Account A
    participant Shard2 as Shard 2 Account B
    
    Note over Client,Shard2: Transaction: Transfer $50 from A to B
    
    Client->>Coordinator: BEGIN TRANSACTION
    Client->>Coordinator: UPDATE accounts SET balance=balance-50 WHERE id=A
    Client->>Coordinator: UPDATE accounts SET balance=balance+50 WHERE id=B
    Client->>Coordinator: COMMIT
    
    Note over Coordinator,Shard2: PHASE 1 PREPARE All or Nothing
    
    Coordinator->>Shard1: PREPARE Can you commit? Lock row A
    Coordinator->>Shard2: PREPARE Can you commit? Lock row B
    
    Shard1->>Shard1: Lock row A check balance>= 50
    Shard1-->>Coordinator: Vote YES can commit
    
    Shard2->>Shard2: Lock row B check constraints
    Shard2-->>Coordinator: Vote YES can commit
    
    Note over Coordinator: All voted YES proceed to commit
    
    Note over Coordinator,Shard2: PHASE 2 COMMIT Execute Changes
    
    Coordinator->>Shard1: COMMIT Apply change balance=balance-50
    Coordinator->>Shard2: COMMIT Apply change balance=balance+50
    
    Shard1->>Shard1: Apply change release lock
    Shard1-->>Coordinator: ACK Committed
    
    Shard2->>Shard2: Apply change release lock
    Shard2-->>Coordinator: ACK Committed
    
    Coordinator-->>Client: Transaction SUCCESS
    
    Note over Shard1,Shard2: Total latency: 20-50ms 2 network RTTs
```

---

## 8. Timestamp Oracle Architecture

**Flow Explanation:**

This diagram shows the Timestamp Oracle service, which provides globally unique, monotonically increasing timestamps for transactions. The oracle is a highly available Raft cluster that issues timestamps on demand. Each transaction receives a unique timestamp, which is used for MVCC (Multi-Version Concurrency Control) and determining transaction order across shards.

**How it Works:**

1. **Transaction Start:** Client requests timestamp from oracle
2. **Oracle:** Increments counter, returns unique timestamp (e.g., 1640000000001)
3. **Transaction Execution:** All writes tagged with this timestamp
4. **Transaction Commit:** Timestamp used to determine visibility in MVCC

**Timestamp Structure:**

```
64-bit timestamp: [Physical Time: 48 bits][Logical Counter: 16 bits]

Physical Time: Unix milliseconds (1640000000000)
Logical Counter: Monotonic counter for same millisecond (0-65535)
```

**High Availability:**

- **Raft Cluster:** 3 oracle nodes (1 leader, 2 followers)
- **Failover:** <2 seconds (Raft election)
- **Throughput:** 1M timestamps/sec (leader bottleneck)

**Google Spanner's TrueTime:**
- Uses atomic clocks + GPS for global sync
- Provides uncertainty bound: [earliest, latest] (<7ms)
- Commits wait out uncertainty ("commit wait")

```mermaid
graph TB
    subgraph Timestamp Oracle Cluster - HA
        Oracle_Leader[Timestamp Oracle Leader<br/>Node 1<br/>Current timestamp: 1640000000001<br/>Increment rate: 1M per sec<br/>Role: Issue unique timestamps]
        
        Oracle_F1[Timestamp Oracle Follower<br/>Node 2<br/>Replicate leader state<br/>Standby for failover]
        
        Oracle_F2[Timestamp Oracle Follower<br/>Node 3<br/>Replicate leader state<br/>Standby for failover]
    end
    
    subgraph Timestamp Generation
        Counter[Monotonic Counter:<br/>Physical Time: 1640000000<br/>Logical Counter: 0000 to 9999<br/>Format: 1640000000.0001<br/>Guarantee: Never go backwards]
        
        Batch[Batch Allocation:<br/>Pre-allocate 1000 timestamps<br/>Persist to Raft log<br/>Serve from memory<br/>Refill when depleted]
    end
    
    subgraph Clients
        TxnA[Transaction A<br/>Timestamp: 1640000000001]
        TxnB[Transaction B<br/>Timestamp: 1640000000002]
        TxnC[Transaction C<br/>Timestamp: 1640000000003]
    end
    
    subgraph Usage in MVCC
        MVCC[Multi-Version Concurrency Control:<br/>Row 1:<br/>  version 1640000000001: value=A<br/>  version 1640000000002: value=B<br/>  version 1640000000003: value=C<br/>Read at timestamp 1640000000002 returns value=B]
    end
    
    TxnA -->|Request timestamp| Oracle_Leader
    TxnB -->|Request timestamp| Oracle_Leader
    TxnC -->|Request timestamp| Oracle_Leader
    
    Oracle_Leader --> Counter
    Oracle_Leader --> Batch
    
    Oracle_Leader -->|Raft replication| Oracle_F1
    Oracle_Leader -->|Raft replication| Oracle_F2
    
    Counter --> MVCC
    
    style Oracle_Leader fill:#95e1d3
    style MVCC fill:#ffd93d
```

---

## 9. Metadata Store (etcd) Architecture

**Flow Explanation:**

This diagram shows the metadata store (etcd) that manages cluster state. Etcd stores range assignments (which nodes own which key ranges), node health status, schema definitions, and cluster configuration. Gateways watch etcd for changes and cache metadata locally. When a range splits or a leader changes, etcd is updated, and gateways are notified.

**Metadata Stored:**

1. **Range Assignments:** range_id → (start_key, end_key, leader_node, replicas)
2. **Node Status:** node_id → (health, cpu, disk, last_heartbeat)
3. **Schema:** table definitions, indexes, constraints
4. **Configuration:** replication factor, quorum size, timeouts

**Watch Mechanism:**

- Gateways "watch" etcd for changes
- When metadata changes (e.g., leader election), etcd sends notification
- Gateway invalidates cache, fetches new metadata

**Performance:**

- **Read Latency:** <1ms (gateway cache hit 99%)
- **Write Latency:** 10-50ms (etcd Raft consensus)
- **Watch Notification:** <100ms (push-based)

**High Availability:**

- 3-node etcd cluster (Raft)
- Tolerates 1 node failure
- Automatic leader election

```mermaid
graph TB
    subgraph etcd Cluster - Metadata Store
        etcd_L[etcd Leader Node 1<br/>Stores cluster metadata<br/>Handles writes]
        etcd_F1[etcd Follower Node 2<br/>Replicates metadata<br/>Handles reads]
        etcd_F2[etcd Follower Node 3<br/>Replicates metadata<br/>Handles reads]
    end
    
    subgraph Metadata Schema
        Ranges[Range Assignments:<br/>range_id start_key end_key leader_node replicas<br/>1 A F Node1 1 2 3<br/>2 G M Node4 4 5 6<br/>3 N Z Node7 7 8 9]
        
        Nodes[Node Status:<br/>node_id health cpu disk last_heartbeat<br/>1 healthy 50 percent 80 percent 2s ago<br/>2 healthy 30 percent 60 percent 1s ago]
        
        Schema[Schema:<br/>table_name columns indexes<br/>users user_id name email<br/>orders order_id user_id amount]
    end
    
    subgraph Gateways with Cache
        GW1[Gateway 1<br/>Cache: range mappings<br/>Watch: etcd changes<br/>TTL: 60 seconds]
        
        GW2[Gateway 2<br/>Cache: range mappings<br/>Watch: etcd changes<br/>TTL: 60 seconds]
    end
    
    subgraph Events
        Event1[Event: Range 1 split<br/>etcd updates:<br/>range 1: A-M<br/>range 4: N-Z new<br/>Notify watchers]
        
        Event2[Event: Leader failover<br/>etcd updates:<br/>range 2 leader: Node 5 was Node 4<br/>Notify watchers]
    end
    
    etcd_L -->|Raft replication| etcd_F1
    etcd_L -->|Raft replication| etcd_F2
    
    etcd_L --> Ranges
    etcd_L --> Nodes
    etcd_L --> Schema
    
    GW1 -.->|Watch etcd| etcd_L
    GW2 -.->|Watch etcd| etcd_L
    
    Event1 -->|Notification 100ms| GW1
    Event1 -->|Notification 100ms| GW2
    
    Event2 -->|Notification 100ms| GW1
    Event2 -->|Notification 100ms| GW2
    
    style etcd_L fill:#ffd93d
    style Event1 fill:#ff6b6b
    style Event2 fill:#ff6b6b
```

---

## 10. Leader Election Flow

**Flow Explanation:**

This diagram shows the Raft leader election process when a leader fails. Followers detect leader failure via missing heartbeats (timeout after 1 second). A follower transitions to CANDIDATE, increments the term (epoch), and requests votes from peers. If it receives a majority of votes, it becomes the new leader and starts sending heartbeats. This process typically completes in 1-2 seconds.

**Election Process:**

1. **Normal Operation:** Leader sends heartbeats every 100ms
2. **Failure:** Followers don't receive heartbeat for 1 second (election timeout)
3. **Candidate:** Follower transitions to CANDIDATE, increments term
4. **Vote Request:** CANDIDATE sends RequestVote RPC to all peers
5. **Voting:** Each node votes for at most one candidate per term
6. **Win Election:** CANDIDATE receives majority votes (2 out of 3)
7. **New Leader:** CANDIDATE transitions to LEADER, sends heartbeats
8. **Update Metadata:** New leader updates etcd with its node ID

**Split Vote:** If two candidates split votes (each gets 1 vote), election times out, and they retry with higher term.

**Downtime:** ~1-2 seconds (election timeout + vote + metadata update)

```mermaid
stateDiagram-v2
    [*] --> Normal : Cluster operating normally
    
    Normal --> LeaderFailed : Leader Node 1 crashes or network partition
    
    state LeaderFailed {
        state "Leader Node 1" as L1
        state "Follower Node 2" as F2
        state "Follower Node 3" as F3
        
        L1 --> Crashed : Network partition or process crash
        F2 --> NoHeartbeat : No heartbeat for 1 second election timeout
        F3 --> NoHeartbeat : No heartbeat for 1 second election timeout
    }
    
    NoHeartbeat --> Candidate : Node 2 transitions to CANDIDATE term 5 to 6
    
    state Candidate {
        state "Candidate Node 2" as C2
        state "Request Votes" as RV
        state "Voting" as V
        
        C2 --> RV : Send RequestVote to Node 2 Node 3
        RV --> V : Nodes respond with votes
    }
    
    V --> NewLeader : Node 2 receives majority 2 out of 3 votes
    
    state NewLeader {
        state "Leader Node 2" as L2
        state "Update Metadata" as UM
        state "Send Heartbeats" as SH
        
        L2 --> UM : Write to etcd range 1 leader Node 2
        UM --> SH : Notify followers Establish authority
    }
    
    NewLeader --> [*] : Cluster operating normally New leader elected
    
    note right of LeaderFailed
        Detection: 1 second election timeout
    end note
    
    note right of Candidate
        Vote Request: 100-300ms send vote receive vote
    end note
    
    note right of NewLeader
        Metadata Update: 50-100ms etcd write
        Total Downtime: 1-2 seconds
    end note
```

---

## 11. Split Brain Prevention

**Flow Explanation:**

This diagram demonstrates how Raft prevents split brain during network partitions. When a cluster is partitioned, only the partition with a majority of nodes (>N/2) can elect a leader and accept writes. The minority partition cannot form a quorum and rejects all writes. This ensures that conflicting writes never occur in different partitions.

**Scenario:**

- 5-node cluster: Nodes 1, 2, 3, 4, 5
- Network partition splits into:
  - Partition A: Nodes 1, 2, 3 (3 nodes = majority)
  - Partition B: Nodes 4, 5 (2 nodes = minority)

**Behavior:**

**Partition A (Majority):**
- 3 out of 5 nodes = >50% ✅
- Can elect leader (Node 1)
- Can accept writes (quorum = 2 out of 3)
- System operational

**Partition B (Minority):**
- 2 out of 5 nodes = <50% ❌
- Cannot elect leader (need 3 votes)
- Reject all writes (no quorum)
- System read-only or unavailable

**After Partition Heals:**

- Partition B rejoins
- Nodes 4, 5 discover higher term in Partition A
- Nodes 4, 5 sync from Partition A leader
- Cluster unified with single leader

**Key Insight:** Raft requires **strictly more than half** (N/2 + 1) to form quorum, preventing split brain.

```mermaid
graph TB
    subgraph Before Partition - Normal Operation
        N1[Node 1 Leader<br/>Term: 5]
        N2[Node 2 Follower<br/>Term: 5]
        N3[Node 3 Follower<br/>Term: 5]
        N4[Node 4 Follower<br/>Term: 5]
        N5[Node 5 Follower<br/>Term: 5]
        
        N1 --- N2
        N1 --- N3
        N1 --- N4
        N1 --- N5
    end
    
    Partition[NETWORK PARTITION occurs<br/>Cluster splits into 2 partitions]
    
    subgraph After Partition - Split Cluster
        subgraph Partition A - Majority 3 out of 5
            PA_N1[Node 1 LEADER<br/>Term: 6<br/>Votes: Node1 Node2 Node3 = 3<br/>Quorum: YES can elect]
            PA_N2[Node 2 Follower<br/>Term: 6]
            PA_N3[Node 3 Follower<br/>Term: 6]
            
            PA_N1 --- PA_N2
            PA_N1 --- PA_N3
            
            PA_Status[Status: OPERATIONAL<br/>Leader elected: Node 1<br/>Accepts writes: YES<br/>Quorum: 2 out of 3]
        end
        
        subgraph Partition B - Minority 2 out of 5
            PB_N4[Node 4 CANDIDATE<br/>Term: 6<br/>Votes: Node4 Node5 = 2<br/>Quorum: NO need 3<br/>Cannot elect leader]
            PB_N5[Node 5 CANDIDATE<br/>Term: 6]
            
            PB_N4 --- PB_N5
            
            PB_Status[Status: UNAVAILABLE<br/>Leader: NONE<br/>Accepts writes: NO<br/>Reason: No quorum]
        end
    end
    
    Heal[Partition heals Network restored]
    
    subgraph After Heal - Unified Cluster
        H_N1[Node 1 LEADER<br/>Term: 6]
        H_N2[Node 2 Follower<br/>Term: 6]
        H_N3[Node 3 Follower<br/>Term: 6]
        H_N4[Node 4 Follower<br/>Term: 6 synced from Node 1]
        H_N5[Node 5 Follower<br/>Term: 6 synced from Node 1]
        
        H_N1 --- H_N2
        H_N1 --- H_N3
        H_N1 --- H_N4
        H_N1 --- H_N5
        
        H_Status[Status: OPERATIONAL<br/>Single leader: Node 1<br/>All nodes synchronized]
    end
    
    N1 -.-> Partition
    Partition -.-> PA_N1
    Partition -.-> PB_N4
    
    PA_N1 -.-> Heal
    PB_N4 -.-> Heal
    Heal -.-> H_N1
    
    style PA_N1 fill:#95e1d3
    style PA_Status fill:#95e1d3
    style PB_Status fill:#ff6b6b
    style H_Status fill:#95e1d3
```

---

## 12. Multi-Region Deployment

**Flow Explanation:**

This diagram shows a multi-region distributed database deployment. Data is replicated across 3 geographic regions (US, EU, APAC) for high availability and disaster recovery. Regional Raft groups keep replicas close together (low latency), while cross-region replication provides global durability. Clients read from the nearest region for low latency. Cross-region writes are slower (100-200ms) due to network latency.

**Architecture:**

**Per-Region Raft Groups:**
- US Region: Range US-data → US Raft Group (3 nodes in US)
- EU Region: Range EU-data → EU Raft Group (3 nodes in EU)
- APAC Region: Range APAC-data → APAC Raft Group (3 nodes in APAC)

**Global Raft Groups:**
- Critical data: Global Raft Group (1 replica per region = 3 total)
- Consensus across regions (200ms latency)

**Latency:**

| Operation | Single Region | Multi-Region |
|-----------|--------------|--------------|
| **Read (local)** | 10ms | 10ms (nearest replica) |
| **Read (cross-region)** | 100ms | 100ms |
| **Write (regional)** | 10ms | 10ms (in-region consensus) |
| **Write (global)** | 200ms | 200ms (cross-region consensus) |

**Data Placement Strategy:**

- **Regional Data (80%):** User profiles, content (in-region writes: 10ms)
- **Global Data (20%):** Financial transactions, audit logs (cross-region writes: 200ms)

```mermaid
graph TB
    subgraph US Region - Virginia
        US_Client[US Clients]
        US_GW[US Gateway]
        US_RG[US Raft Group<br/>Range: US-data<br/>Nodes: US-1 US-2 US-3<br/>Leader: US-1]
        US_Latency[Regional Write: 10ms<br/>Regional Read: 10ms]
    end
    
    subgraph EU Region - Frankfurt
        EU_Client[EU Clients]
        EU_GW[EU Gateway]
        EU_RG[EU Raft Group<br/>Range: EU-data<br/>Nodes: EU-1 EU-2 EU-3<br/>Leader: EU-1]
        EU_Latency[Regional Write: 10ms<br/>Regional Read: 10ms]
    end
    
    subgraph APAC Region - Singapore
        APAC_Client[APAC Clients]
        APAC_GW[APAC Gateway]
        APAC_RG[APAC Raft Group<br/>Range: APAC-data<br/>Nodes: APAC-1 APAC-2 APAC-3<br/>Leader: APAC-1]
        APAC_Latency[Regional Write: 10ms<br/>Regional Read: 10ms]
    end
    
    subgraph Global Raft Group - Critical Data
        Global[Global Raft Group<br/>Range: global-critical<br/>Replicas:<br/>US-Global Leader<br/>EU-Global Follower<br/>APAC-Global Follower<br/>Write Latency: 200ms<br/>Quorum: 2 out of 3 regions]
    end
    
    subgraph Cross-Region Replication
        Repl[Replication:<br/>US to EU: 100ms<br/>US to APAC: 200ms<br/>EU to APAC: 150ms]
    end
    
    US_Client --> US_GW
    US_GW --> US_RG
    US_GW -.->|Cross-region read 100ms| EU_RG
    
    EU_Client --> EU_GW
    EU_GW --> EU_RG
    EU_GW -.->|Cross-region read 100ms| US_RG
    
    APAC_Client --> APAC_GW
    APAC_GW --> APAC_RG
    APAC_GW -.->|Cross-region read 200ms| US_RG
    
    US_RG -.->|Async replication| Global
    EU_RG -.->|Async replication| Global
    APAC_RG -.->|Async replication| Global
    
    Global --> Repl
    
    style US_RG fill:#95e1d3
    style EU_RG fill:#95e1d3
    style APAC_RG fill:#95e1d3
    style Global fill:#ffd93d
```

---

## 13. Hotspot Detection and Mitigation

**Flow Explanation:**

This diagram shows the hotspot detection and mitigation system. A monitoring thread continuously tracks QPS, storage size, and CPU usage for each range. When a range exceeds thresholds (>10K QPS for 60 seconds or >64 MB size), it's flagged as a hotspot. The split coordinator automatically splits the range at the median key, creates a new Raft group, and migrates data. This prevents node overload and distributes traffic evenly.

**Detection Metrics:**

1. **QPS (Queries Per Second):** >10K QPS for 60 seconds
2. **Storage Size:** >64 MB (or 128 MB, 256 MB configurable)
3. **CPU Usage:** >80% CPU for 5 minutes
4. **Network Bandwidth:** >1 GB/sec for 5 minutes

**Mitigation Actions:**

1. **Split Range:** Divide hot range into two
2. **Load Shedding:** Rate limit requests temporarily
3. **Add Replicas:** Increase replication factor (3 → 5)
4. **Rebalance:** Move range to less loaded nodes

**Split Strategy:**

- **Median Split:** Split at median key (balanced sizes)
- **Hot Key Split:** Isolate hot key into separate range
- **Time-Based Split:** Split by timestamp (recent vs old data)

**Performance:**

- **Detection Latency:** 60 seconds (metrics aggregation)
- **Split Duration:** 30 seconds - 5 minutes (copy data)
- **Downtime:** Zero (reads/writes continue)

```mermaid
graph TB
    subgraph Monitoring System
        Monitor[Monitoring Thread<br/>Every 10 seconds:<br/>Query metrics for all ranges<br/>Aggregate QPS storage CPU]
        
        Metrics[Metrics Store:<br/>range_id QPS storage_mb cpu_percent<br/>1 50K 100 80<br/>2 5K 40 30<br/>3 8K 60 50]
    end
    
    subgraph Detection Logic
        Detect[Hotspot Detector:<br/>Threshold checks:<br/>QPS greater than 10K for 60 sec<br/>Storage greater than 64 MB<br/>CPU greater than 80 percent for 5 min]
        
        Alert[Alert: Range 1 HOTSPOT detected<br/>QPS: 50K exceeds 10K<br/>Storage: 100 MB exceeds 64 MB<br/>Action: Split required]
    end
    
    subgraph Mitigation Actions
        Action1[Action 1: Split Range<br/>Median key split:<br/>Range 1: A-M 50 MB 25K QPS<br/>Range 4: N-Z 50 MB 25K QPS]
        
        Action2[Action 2: Load Shedding<br/>Rate limit:<br/>Reject 50 percent of requests<br/>Return 429 Too Many Requests]
        
        Action3[Action 3: Add Replicas<br/>Increase replication:<br/>3 replicas to 5 replicas<br/>Distribute read load]
    end
    
    subgraph After Mitigation
        Result[Result:<br/>Range 1: 25K QPS 50 MB HEALTHY<br/>Range 4: 25K QPS 50 MB HEALTHY<br/>Total capacity: 2x increased]
    end
    
    Monitor --> Metrics
    Metrics --> Detect
    Detect --> Alert
    
    Alert --> Action1
    Alert -.->|Temporary| Action2
    Alert -.->|If split not enough| Action3
    
    Action1 --> Result
    
    style Alert fill:#ff6b6b
    style Result fill:#95e1d3
    style Monitor fill:#ffd93d
```

---

## 14. Online Schema Migration

**Flow Explanation:**

This diagram illustrates the online schema migration process that allows adding/removing columns without downtime. The migration happens in 4 phases: (1) Write Both old and new schemas, (2) Backfill old rows with new schema in background, (3) Switch reads to new schema, (4) Drop old schema. Different rows can temporarily have different schema versions, tracked via MVCC. This enables zero-downtime schema changes on tables with billions of rows.

**Phases:**

**Phase 1 (Write Both - Week 1):**
- New writes: Include new column (email)
- Old writes (in-flight): Old schema (no email)
- Reads: Return old schema only (backwards compatible)

**Phase 2 (Backfill - Week 2-4):**
- Background worker: Update all old rows to include email
- Progress: 0% → 100% over 2-3 weeks (for billion-row table)
- Reads: Still return old schema (users don't see partial data)

**Phase 3 (Read New - Week 5):**
- Reads: Switch to new schema (include email)
- Writes: Continue writing new schema
- All rows now have new schema

**Phase 4 (Cleanup - Week 6):**
- Drop old schema version
- Delete old column metadata
- Reclaim storage space

**Duration:** 4-6 weeks for billion-row table (depends on size)

**Downtime:** Zero (all operations continue during migration)

```mermaid
stateDiagram-v2
    [*] --> Phase1 : Schema change initiated
    
    state Phase1 {
        state "Phase 1 Write Both" as P1
        P1 : Writes - user_id name email new
        P1 : Writes - user_id name old still exist
        P1 : Reads - user_id name old schema
        P1 : Duration - 1 week
    }
    
    Phase1 --> Phase2 : Backfill starts
    
    state Phase2 {
        state "Phase 2 Backfill" as P2
        P2 : Background worker
        P2 : UPDATE users SET email NULL WHERE missing
        P2 : Progress - 0 percent to 100 percent
        P2 : Reads - user_id name old schema
        P2 : Duration - 2 to 3 weeks
        
        state "Backfill Progress" as BP
        BP : Day 1 at 10 percent
        BP : Day 7 at 50 percent
        BP : Day 14 at 90 percent
        BP : Day 21 at 100 percent
    }
    
    Phase2 --> Phase3 : Backfill complete
    
    state Phase3 {
        state "Phase 3 Read New" as P3
        P3 : Writes - user_id name email
        P3 : Reads - user_id name email new schema
        P3 : All rows have new schema
        P3 : Duration - 1 week validation
    }
    
    Phase3 --> Phase4 : Validation complete
    
    state Phase4 {
        state "Phase 4 Cleanup" as P4
        P4 : Drop old schema metadata
        P4 : Reclaim storage space
        P4 : Update documentation
        P4 : Duration - 1 day
    }
    
    Phase4 --> [*] : Migration complete
    
    note right of Phase1
        Multi-version schema Row 1 v1 old Row 2 v2 new MVCC tracks version per row
    end note
    
    note right of Phase2
        Backfill rate 10K rows per sec equals 28 hours for 1 billion rows
    end note
    
    note right of Phase3
        Gradual rollout 10 percent Monitor for errors then 100 percent
    end note

