# Distributed Cache - High-Level Design

## Table of Contents

1. [System Architecture Diagram](#system-architecture-diagram)
2. [Consistent Hashing Ring](#consistent-hashing-ring)
3. [Replication Architecture](#replication-architecture)
4. [Cache Eviction Policies](#cache-eviction-policies)
5. [Data Flow Patterns](#data-flow-patterns)
6. [Failover Mechanism](#failover-mechanism)
7. [Cluster Sharding (Redis Cluster)](#cluster-sharding-redis-cluster)
8. [Performance Optimization Techniques](#performance-optimization-techniques)
9. [Scaling Strategy](#scaling-strategy)
10. [Technology Comparison](#technology-comparison)
11. [Monitoring Dashboard](#monitoring-dashboard)

---

## System Architecture Diagram

**Flow Explanation:**
Complete architecture of a distributed cache cluster with sharding, replication, and high availability.

**Key Components:**

1. **Application Layer:** Multiple app servers connect via cache client library
2. **Cache Client:** Implements consistent hashing to route requests, connection pooling for efficiency, retry logic for
   resilience
3. **Primary Nodes (3 Masters):** Each handles 1/3 of keyspace (16,384 slots total) using hash slots, 64GB RAM each,
   100K QPS capacity
4. **Replica Layer:** Each master has 2 replicas with async replication (1-5ms lag), provides read scaling and failover
   capability
5. **Sentinel Monitoring:** Detects failures, triggers automatic promotion of replica to master

**Data Flow:** Client hashes key → Determines slot → Routes to correct master → Async replication to replicas → Read
from master or replica

```mermaid
graph TB
    subgraph "Application Layer"
        App1[Application Server 1]
        App2[Application Server 2]
        App3[Application Server N]
    end

    subgraph "Cache Client Library"
        Client[Cache Client<br/>- Consistent Hashing<br/>- Connection Pooling<br/>- Retry Logic]
    end

    App1 --> Client
    App2 --> Client
    App3 --> Client

    subgraph "Primary Nodes"
        Master1[Node 1 Master<br/>64 GB RAM<br/>100K QPS<br/>Slots 0-5460]
        Master2[Node 2 Master<br/>64 GB RAM<br/>100K QPS<br/>Slots 5461-10922]
        Master3[Node 3 Master<br/>64 GB RAM<br/>100K QPS<br/>Slots 10923-16383]
    end

    Client --> Master1
    Client --> Master2
    Client --> Master3

    subgraph "Replica Layer"
        Replica1A[Replica 1A<br/>64 GB]
        Replica1B[Replica 1B<br/>64 GB]
        Replica2A[Replica 2A<br/>64 GB]
        Replica2B[Replica 2B<br/>64 GB]
        Replica3A[Replica 3A<br/>64 GB]
        Replica3B[Replica 3B<br/>64 GB]
    end

    Master1 -->|Async Replication<br/>1 - 5ms lag| Replica1A
    Master1 -->|Async Replication<br/>1 - 5ms lag| Replica1B
    Master2 -->|Async Replication<br/>1 - 5ms lag| Replica2A
    Master2 -->|Async Replication<br/>1 - 5ms lag| Replica2B
    Master3 -->|Async Replication<br/>1 - 5ms lag| Replica3A
    Master3 -->|Async Replication<br/>1 - 5ms lag| Replica3B

    subgraph "Sentinel Cluster (Monitoring)"
        Sentinel1[Sentinel 1<br/>Quorum=2]
        Sentinel2[Sentinel 2<br/>Quorum=2]
        Sentinel3[Sentinel 3<br/>Quorum=2]
    end

    Sentinel1 -.->|Monitor| Master1
    Sentinel1 -.->|Monitor| Master2
    Sentinel1 -.->|Monitor| Master3
    Sentinel2 -.->|Monitor| Master1
    Sentinel2 -.->|Monitor| Master2
    Sentinel2 -.->|Monitor| Master3
    Sentinel3 -.->|Monitor| Master1
    Sentinel3 -.->|Monitor| Master2
    Sentinel3 -.->|Monitor| Master3
    style Master1 fill: #e1ffe1
    style Master2 fill: #e1ffe1
    style Master3 fill: #e1ffe1
    style Client fill: #e1f5ff
    style Sentinel1 fill: #fffce1
    style Sentinel2 fill: #fffce1
    style Sentinel3 fill: #fffce1
```

## Consistent Hashing Ring

**Flow Explanation:**
Shows how consistent hashing distributes keys evenly across nodes using virtual nodes.

**Mechanism:**

1. Hash ring spans 0 to 2^32-1
2. Each physical node creates 150 virtual nodes (VNodes) on the ring
3. Key is hashed (e.g., hash("user:123"))
4. Search clockwise on ring to find next VNode
5. VNode maps back to physical node

**Benefits:** When adding/removing nodes, only ~1/N keys need remapping (vs. ~100% with modulo hashing). Virtual nodes
ensure even distribution even with unequal node count.

```mermaid
graph LR
    subgraph "Hash Ring (0 to 2^32-1)"
        direction TB
        Ring[Consistent Hash Ring]
    end

    subgraph "Virtual Nodes (VNodes)"
        Node1_V1[Node1:0]
        Node1_V2[Node1:1]
        Node1_V150[Node1:149]
        Node2_V1[Node2:0]
        Node2_V2[Node2:1]
        Node2_V150[Node2:149]
        Node3_V1[Node3:0]
        Node3_V2[Node3:1]
        Node3_V150[Node3:149]
    end

    subgraph "Key Distribution"
        Key1[user:123]
        Key2[product:456]
        Key3[session:789]
    end

    Key1 -->|hash key| Ring
    Key2 -->|hash key| Ring
    Key3 -->|hash key| Ring
    Ring -->|clockwise search| Node1_V1
    Ring -->|clockwise search| Node2_V1
    Ring -->|clockwise search| Node3_V1
    Node1_V1 -->|maps to| Node1[Physical Node 1]
    Node2_V1 -->|maps to| Node2[Physical Node 2]
    Node3_V1 -->|maps to| Node3[Physical Node 3]
    style Ring fill: #e1f5ff
    style Node1 fill: #e1ffe1
    style Node2 fill: #e1ffe1
    style Node3 fill: #e1ffe1
```

## Replication Architecture

**Flow Explanation:**
Master-replica setup for high availability and read scaling.

**Write Path:** All writes go to master → Master performs async replication to 2 replicas (non-blocking, 1-5ms lag)
**Read Path:** Reads load-balanced across master + 2 replicas (3x read capacity)

**Trade-off:** Async replication means eventual consistency - replicas may lag behind master by 1-5ms. Acceptable for
cache use case (vs. database where consistency is critical).

```mermaid
graph TB
    subgraph "Write Path"
        WriteReq[Write Request]
    end

    WriteReq --> Master[Master Node<br/>Primary]
    Master -->|Async Replication<br/>Non - blocking| Replica1[Replica 1<br/>Read-only<br/>Lags 1-5ms]
    Master -->|Async Replication<br/>Non - blocking| Replica2[Replica 2<br/>Read-only<br/>Lags 1-5ms]

    subgraph "Read Path (Load Balanced)"
        ReadReq1[Read Request 1]
        ReadReq2[Read Request 2]
        ReadReq3[Read Request 3]
    end

    ReadReq1 --> Master
    ReadReq2 --> Replica1
    ReadReq3 --> Replica2
    style Master fill: #e1ffe1
    style Replica1 fill: #f0fff0
    style Replica2 fill: #f0fff0
    style WriteReq fill: #ffe1e1
```

## Cache Eviction Policies

**Flow Explanation:**
Comparison of algorithms for deciding which data to evict when cache is full.

**LRU (Least Recently Used):** Doubly-linked list tracks access order → Evict tail (least recent)
**LFU (Least Frequently Used):** Counter tracks access frequency → Evict lowest frequency
**TTL (Time To Live):** Each key has expiry timestamp → Evict expired keys first
**Random:** Random selection → Simple but unpredictable

**Recommendation:** LRU for most use cases (80-90% hit rate). LFU for access patterns with clear hot/cold data
distinction.

```mermaid
graph LR
subgraph "LRU (Least Recently Used)"
LRU_Head[Most Recent]
LRU_Mid[...]
LRU_Tail[Least Recent<br/>← EVICT]
end

subgraph "LFU (Least Frequently Used)"
LFU_Hot[Hot Data<br/>High Frequency]
LFU_Warm[Warm Data<br/>Medium Frequency]
LFU_Cold[Cold Data<br/>Low Frequency<br/>← EVICT]
end

subgraph "TTL (Time To Live)"
TTL_New[Newest<br/>Just Added]
TTL_Mid[Mid-life]
TTL_Exp[Expired<br/>← EVICT]
end

subgraph "Random"
RandomEvict[Random Selection<br/>← EVICT]
end

Access[Data Access] --> LRU_Head
Access --> LFU_Hot
Access --> TTL_New

LRU_Head -->|move down|LRU_Mid
LRU_Mid -->|move down| LRU_Tail

LFU_Hot -->|infrequent access|LFU_Warm
LFU_Warm -->|infrequent access|LFU_Cold

TTL_New -->|time passes|TTL_Mid
TTL_Mid -->|time passes|TTL_Exp

style LRU_Tail fill: #ffe1e1
style LFU_Cold fill: #ffe1e1
style TTL_Exp fill: #ffe1e1
style RandomEvict fill: #ffe1e1
```

## Data Flow Patterns

### Cache-Aside (Lazy Loading)

**Flow Explanation:**
Most common caching pattern where application manages cache population.

**Read:** App checks cache → If miss: Query DB → Write to cache → Return data
**Write:** App writes to DB → Invalidate cache (optional)

**Pros:** Simple, cache only stores accessed data. **Cons:** Cache misses add latency, risk of stale data.

```mermaid
flowchart TD
    Start([Read Request]) --> CheckCache{Check<br/>Cache}
    CheckCache -->|HIT| ReturnCache[Return from Cache<br/>~1ms]
ReturnCache --> End1([End])

CheckCache -->|MISS|QueryDB[Query Database<br/>~50ms]
QueryDB --> UpdateCache[Update Cache<br/>with TTL]
UpdateCache --> ReturnDB[Return Data<br/>Total: ~50ms]
ReturnDB --> End2([End])

style ReturnCache fill: #e1ffe1
style QueryDB fill: #ffe1e1
style UpdateCache fill: #e1f5ff
```

### Write-Through Pattern

**Flow Explanation:**
Cache is always in sync with database - writes go through cache.

**Write:** App → Cache → DB (synchronous) → Return. **Read:** App → Cache (always has latest data).

**Pros:** Cache never stale, simple consistency. **Cons:** Higher write latency (must wait for DB), unused data cached.

```mermaid
flowchart TD
    Start([Write Request]) --> WriteCache[Write to Cache]
    WriteCache --> WriteDB[Write to Database]
    WriteDB --> Confirm[Confirm Success]
    Confirm --> End([End])
    style WriteCache fill: #e1f5ff
    style WriteDB fill: #e1ffe1
```

### Write-Behind (Write-Back) Pattern

**Flow Explanation:**
Writes batched and persisted asynchronously for maximum write performance.

**Write:** App → Cache (immediate return) → Async batch write to DB. **Read:** App → Cache.

**Pros:** Ultra-fast writes, reduced DB load. **Cons:** Risk of data loss if cache fails before DB write, eventual
consistency.

```mermaid
flowchart TD
    Start([Write Request]) --> WriteCache[Write to Cache]
    WriteCache --> AckClient[Acknowledge Client<br/>Fast Response]
    AckClient --> End([Client Done])
    WriteCache -.->|Async| QueueWrite[Queue for DB Write]
    QueueWrite -.->|Batch| WriteDB[Write to Database<br/>in background]
    style WriteCache fill: #e1f5ff
    style AckClient fill: #e1ffe1
    style WriteDB fill: #f0fff0
```

## Failover Mechanism

**Flow Explanation:**
Automatic recovery when master node fails using Redis Sentinel.

**Steps:**

1. Master node crashes
2. Sentinel detects failure (heartbeat timeout)
3. Sentinels form quorum (majority vote required)
4. Promote Replica 1 to new master
5. Update clients with new master address
6. Old master becomes replica when it recovers

**Downtime:** ~5-30 seconds depending on Sentinel configuration. Minimal data loss (only data not yet replicated).

```mermaid
sequenceDiagram
    participant App as Application
    participant Sentinel as Sentinel Cluster
    participant Master as Master Node
    participant Replica1 as Replica 1
    participant Replica2 as Replica 2
    Note over Master: Master is Healthy
    App ->> Master: Normal Operations
    Sentinel ->> Master: Health Check (Every 1s)
    Master -->> Sentinel: OK
    Note over Master: Master FAILS!
    Sentinel ->> Master: Health Check
    Note over Sentinel: No response (5s timeout)
    Sentinel ->> Sentinel: Quorum Vote<br/>(2 out of 3 agree)
    Note over Sentinel: Master declared down
    Sentinel ->> Replica1: Promote to Master
    activate Replica1
    Replica1 -->> Sentinel: Promotion Successful
    Note over Replica1: Now Master
    Sentinel ->> App: Update Configuration<br/>New Master: Replica1
    App ->> Replica1: Resume Operations<br/>(~10-15s total downtime)
    Note over Replica2: Reconfigures to<br/>replicate from new Master
    Replica1 ->> Replica2: Start Replication
    deactivate Replica1
```

## Cluster Sharding (Redis Cluster)

**Flow Explanation:**
Horizontal scaling by partitioning data across multiple masters using hash slots.

**Sharding Logic:** 16,384 total hash slots divided evenly across N masters. Key hashed to slot: `CRC16(key) mod 16384`.

**Example (3 masters):**

- Master 1: Slots 0-5460
- Master 2: Slots 5461-10922
- Master 3: Slots 10923-16383

**Benefits:** Linear scaling (add masters for more capacity), automatic slot migration, built-in client-side routing.

```mermaid
graph TB
    subgraph "Client Layer"
        RedisClient[Redis Client]
    end

    subgraph "Hash Slot Distribution (16384 slots)"
        Router[CRC16 key mod 16384]
    end

    RedisClient --> Router
    Router -->|Slots 0 - 5460| Shard1[Shard 1<br/>Master + Replicas]
    Router -->|Slots 5461 - 10922| Shard2[Shard 2<br/>Master + Replicas]
    Router -->|Slots 10923 - 16383| Shard3[Shard 3<br/>Master + Replicas]

    subgraph "Shard 1 Detail"
        S1M[Master]
        S1R1[Replica 1]
        S1R2[Replica 2]
        S1M --> S1R1
        S1M --> S1R2
    end

    Shard1 --> S1M
    style RedisClient fill: #e1f5ff
    style Router fill: #fffce1
    style Shard1 fill: #e1ffe1
    style Shard2 fill: #e1ffe1
    style Shard3 fill: #e1ffe1
```

## Performance Optimization Techniques

**Flow Explanation:**
Key optimizations to maximize throughput and minimize latency.

**Techniques:**

1. **Connection Pooling:** Reuse TCP connections → Avoid 3ms handshake overhead
2. **Pipelining:** Batch multiple commands in single round-trip → 10x throughput improvement
3. **Compression:** gzip/snappy for large values (>1KB) → 10x memory savings
4. **TTL Management:** Automatic expiration → No manual cleanup needed
5. **Read from Replicas:** Load balance reads across 3 nodes → 3x read capacity

**Impact:** Sub-millisecond P99 latency, 300K+ QPS per node.

```mermaid
graph TB
    subgraph "Connection Pooling"
        App[Application]
        Pool[Connection Pool<br/>50 connections<br/>Reuse connections]
        Cache1[Cache Node]
    end

    App --> Pool
    Pool <-->|Reuse| Cache1

    subgraph "Pipelining"
        App2[Application]
        Pipeline[Pipeline<br/>Batch 1000 commands]
        Cache2[Cache Node]
    end

    App2 -->|1 request| Pipeline
    Pipeline -->|1 network RTT| Cache2
    Cache2 -->|1 response| Pipeline
    Pipeline -->|1000 results| App2

subgraph "Compression"
App3[Application]
Compress[Compress<br/>10 KB → 1 KB]
Cache3[Cache Node]
end

App3 -->|Large Data|Compress
Compress -->|Compressed|Cache3
Cache3 -->|Compressed|Compress
Compress -->|Decompressed| App3

style Pool fill: #e1f5ff
style Pipeline fill: #e1ffe1
style Compress fill: #fffce1
```

## Scaling Strategy

**Flow Explanation:**
Progressive scaling path from single instance to distributed cluster.

**Phase 1 (Small):** Single Redis instance (64GB RAM, 100K QPS) → Good for MVP
**Phase 2 (Medium):** Master + 2 replicas → 3x read capacity, high availability
**Phase 3 (Large):** 3 masters with sharding + replicas → 300K QPS, 180GB total capacity
**Phase 4 (Massive):** Multi-region clusters → Global scale, low latency worldwide

**When to scale:** Monitor memory usage (>70%), QPS (>80% capacity), or latency (P99 > 10ms).

```mermaid
graph LR
subgraph "Phase 1: Single Instance"
P1[Single Redis<br/>64 GB RAM<br/>~100K QPS]
end

subgraph "Phase 2: Master-Replica"
P2M[Master<br/>Writes]
P2R1[Replica 1<br/>Reads]
P2R2[Replica 2<br/>Reads]
P2M --> P2R1
P2M --> P2R2
end

subgraph "Phase 3: Sharded Cluster"
P3S1[Shard 1<br/>+ Replicas]
P3S2[Shard 2<br/>+ Replicas]
P3S3[Shard N<br/>+ Replicas]
end

subgraph "Phase 4: Multi-Region"
P4R1[Region 1<br/>Full Cluster]
P4R2[Region 2<br/>Full Cluster]
P4R3[Region 3<br/>Full Cluster]
end

P1 -->|Add Replication|P2M
P2M -->|Add Sharding|P3S1
P3S1 -->|Geographic Distribution| P4R1

style P1 fill: #e1ffe1
style P2M fill: #e1f5ff
style P3S1 fill: #fffce1
style P4R1 fill: #ffe1e1
```

## Technology Comparison

**Flow Explanation:**
Comparison matrix of popular caching solutions to guide technology selection.

**Redis:** Best all-around choice - rich data structures, persistence, Lua scripting, pub/sub. Use for: most
applications.
**Memcached:** Simpler, pure key-value, slightly faster for basic ops. Use for: simple caching needs.
**Hazelcast:** Java-native, distributed data structures, compute grid. Use for: Java microservices.
**Apache Ignite:** SQL support, compute capabilities, ML integration. Use for: data-intensive applications.

**Decision factors:** Language ecosystem, data structure needs, persistence requirements, operational complexity.

```mermaid
graph TB
subgraph "Redis"
R[Redis<br/>✅ Rich data types<br/>✅ Persistence<br/>✅ Single-threaded<br/>✅ Built-in replication]
end

subgraph "Memcached"
M[Memcached<br/>✅ Simple key-value<br/>❌ No persistence<br/>✅ Multi-threaded<br/>❌ No replication]
end

subgraph "KeyDB"
K[KeyDB<br/>✅ Redis compatible<br/>✅ Multi-threaded<br/>✅ Better performance<br/>✅ Active-active]
end

UseCase{Use Case}

UseCase -->|Complex data structures|R
UseCase -->|Pure speed & simplicity|M
UseCase -->|High performance Redis|K

style R fill: #e1ffe1
style M fill: #e1f5ff
style K fill: #fffce1
```

## Monitoring Dashboard

**Flow Explanation:**
Key metrics to monitor for cache health and performance.

**Critical Metrics:**

1. **Hit Rate:** % of cache hits vs misses (target: >80%)
2. **Memory Usage:** Current/max memory (alert at >70%)
3. **QPS:** Queries per second (watch for capacity limits)
4. **Latency:** P50/P99/P999 response times (target: P99 <5ms)
5. **Eviction Rate:** Keys evicted per second (high = undersized cache)
6. **Network I/O:** Bandwidth utilization per node

**Alerts:** Hit rate <70%, memory >80%, P99 latency >10ms, replication lag >100ms, node failures.

```mermaid
graph TB
    subgraph KeyMetrics["Key Metrics"]
        M1[Cache Hit Rate<br/>Target: greater than 80 percent]
        M2[Latency p99<br/>Target: less than 1ms]
        M3[Memory Usage<br/>Target: less than 80 percent]
        M4[Evicted Keys<br/>Target: Minimal]
        M5[Commands per sec<br/>Monitor Baseline]
        M6[Network I/O<br/>Target: less than 70 percent]
    end

    subgraph AlertsGroup["Alerts"]
        A1[Alert: Hit Rate less than 70 percent]
        A2[Alert: Latency greater than 5ms]
        A3[Alert: Memory greater than 90 percent]
        A4[Alert: Evictions greater than 1000 per sec]
    end

    M1 -->|Trigger| A1
    M2 -->|Trigger| A2
    M3 -->|Trigger| A3
    M4 -->|Trigger| A4

    subgraph ActionsGroup["Actions"]
        Act1[Investigate Cache Strategy]
        Act2[Check Slow Queries]
        Act3[Scale Up Memory]
        Act4[Increase Cache Size]
    end

    A1 --> Act1
    A2 --> Act2
    A3 --> Act3
    A4 --> Act4
    style A1 fill: #ffe1e1
    style A2 fill: #ffe1e1
    style A3 fill: #ffe1e1
    style A4 fill: #ffe1e1
```

