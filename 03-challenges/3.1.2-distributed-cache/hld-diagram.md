# Distributed Cache - High-Level Design

## System Architecture Diagram

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
    
    Master1 -->|Async Replication<br/>1-5ms lag| Replica1A
    Master1 -->|Async Replication<br/>1-5ms lag| Replica1B
    
    Master2 -->|Async Replication<br/>1-5ms lag| Replica2A
    Master2 -->|Async Replication<br/>1-5ms lag| Replica2B
    
    Master3 -->|Async Replication<br/>1-5ms lag| Replica3A
    Master3 -->|Async Replication<br/>1-5ms lag| Replica3B
    
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
    
    style Master1 fill:#e1ffe1
    style Master2 fill:#e1ffe1
    style Master3 fill:#e1ffe1
    style Client fill:#e1f5ff
    style Sentinel1 fill:#fffce1
    style Sentinel2 fill:#fffce1
    style Sentinel3 fill:#fffce1
```

## Consistent Hashing Ring

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
    
    style Ring fill:#e1f5ff
    style Node1 fill:#e1ffe1
    style Node2 fill:#e1ffe1
    style Node3 fill:#e1ffe1
```

## Replication Architecture

```mermaid
graph TB
    subgraph "Write Path"
        WriteReq[Write Request]
    end
    
    WriteReq --> Master[Master Node<br/>Primary]
    
    Master -->|Async Replication<br/>Non-blocking| Replica1[Replica 1<br/>Read-only<br/>Lags 1-5ms]
    Master -->|Async Replication<br/>Non-blocking| Replica2[Replica 2<br/>Read-only<br/>Lags 1-5ms]
    
    subgraph "Read Path (Load Balanced)"
        ReadReq1[Read Request 1]
        ReadReq2[Read Request 2]
        ReadReq3[Read Request 3]
    end
    
    ReadReq1 --> Master
    ReadReq2 --> Replica1
    ReadReq3 --> Replica2
    
    style Master fill:#e1ffe1
    style Replica1 fill:#f0fff0
    style Replica2 fill:#f0fff0
    style WriteReq fill:#ffe1e1
```

## Cache Eviction Policies

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
    
    LRU_Head -->|move down| LRU_Mid
    LRU_Mid -->|move down| LRU_Tail
    
    LFU_Hot -->|infrequent access| LFU_Warm
    LFU_Warm -->|infrequent access| LFU_Cold
    
    TTL_New -->|time passes| TTL_Mid
    TTL_Mid -->|time passes| TTL_Exp
    
    style LRU_Tail fill:#ffe1e1
    style LFU_Cold fill:#ffe1e1
    style TTL_Exp fill:#ffe1e1
    style RandomEvict fill:#ffe1e1
```

## Data Flow Patterns

### Cache-Aside (Lazy Loading)

```mermaid
flowchart TD
    Start([Read Request]) --> CheckCache{Check<br/>Cache}
    
    CheckCache -->|HIT| ReturnCache[Return from Cache<br/>~1ms]
    ReturnCache --> End1([End])
    
    CheckCache -->|MISS| QueryDB[Query Database<br/>~50ms]
    QueryDB --> UpdateCache[Update Cache<br/>with TTL]
    UpdateCache --> ReturnDB[Return Data<br/>Total: ~50ms]
    ReturnDB --> End2([End])
    
    style ReturnCache fill:#e1ffe1
    style QueryDB fill:#ffe1e1
    style UpdateCache fill:#e1f5ff
```

### Write-Through Pattern

```mermaid
flowchart TD
    Start([Write Request]) --> WriteCache[Write to Cache]
    WriteCache --> WriteDB[Write to Database]
    WriteDB --> Confirm[Confirm Success]
    Confirm --> End([End])
    
    style WriteCache fill:#e1f5ff
    style WriteDB fill:#e1ffe1
```

### Write-Behind (Write-Back) Pattern

```mermaid
flowchart TD
    Start([Write Request]) --> WriteCache[Write to Cache]
    WriteCache --> AckClient[Acknowledge Client<br/>Fast Response]
    AckClient --> End([Client Done])
    
    WriteCache -.->|Async| QueueWrite[Queue for DB Write]
    QueueWrite -.->|Batch| WriteDB[Write to Database<br/>in background]
    
    style WriteCache fill:#e1f5ff
    style AckClient fill:#e1ffe1
    style WriteDB fill:#f0fff0
```

## Failover Mechanism

```mermaid
sequenceDiagram
    participant App as Application
    participant Sentinel as Sentinel Cluster
    participant Master as Master Node
    participant Replica1 as Replica 1
    participant Replica2 as Replica 2

    Note over Master: Master is Healthy
    
    App->>Master: Normal Operations
    Sentinel->>Master: Health Check (Every 1s)
    Master-->>Sentinel: OK
    
    Note over Master: Master FAILS!
    
    Sentinel->>Master: Health Check
    Note over Sentinel: No response (5s timeout)
    
    Sentinel->>Sentinel: Quorum Vote<br/>(2 out of 3 agree)
    Note over Sentinel: Master declared down
    
    Sentinel->>Replica1: Promote to Master
    activate Replica1
    Replica1-->>Sentinel: Promotion Successful
    
    Note over Replica1: Now Master
    
    Sentinel->>App: Update Configuration<br/>New Master: Replica1
    
    App->>Replica1: Resume Operations<br/>(~10-15s total downtime)
    
    Note over Replica2: Reconfigures to<br/>replicate from new Master
    
    Replica1->>Replica2: Start Replication
    
    deactivate Replica1
```

## Cluster Sharding (Redis Cluster)

```mermaid
graph TB
    subgraph "Client Layer"
        RedisClient[Redis Client]
    end
    
    subgraph "Hash Slot Distribution (16384 slots)"
        Router[CRC16 key mod 16384]
    end
    
    RedisClient --> Router
    
    Router -->|Slots 0-5460| Shard1[Shard 1<br/>Master + Replicas]
    Router -->|Slots 5461-10922| Shard2[Shard 2<br/>Master + Replicas]
    Router -->|Slots 10923-16383| Shard3[Shard 3<br/>Master + Replicas]
    
    subgraph "Shard 1 Detail"
        S1M[Master]
        S1R1[Replica 1]
        S1R2[Replica 2]
        S1M --> S1R1
        S1M --> S1R2
    end
    
    Shard1 --> S1M
    
    style RedisClient fill:#e1f5ff
    style Router fill:#fffce1
    style Shard1 fill:#e1ffe1
    style Shard2 fill:#e1ffe1
    style Shard3 fill:#e1ffe1
```

## Performance Optimization Techniques

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
    
    App3 -->|Large Data| Compress
    Compress -->|Compressed| Cache3
    Cache3 -->|Compressed| Compress
    Compress -->|Decompressed| App3
    
    style Pool fill:#e1f5ff
    style Pipeline fill:#e1ffe1
    style Compress fill:#fffce1
```

## Scaling Strategy

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
    
    P1 -->|Add Replication| P2M
    P2M -->|Add Sharding| P3S1
    P3S1 -->|Geographic Distribution| P4R1
    
    style P1 fill:#e1ffe1
    style P2M fill:#e1f5ff
    style P3S1 fill:#fffce1
    style P4R1 fill:#ffe1e1
```

## Technology Comparison

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
    
    UseCase -->|Complex data structures| R
    UseCase -->|Pure speed & simplicity| M
    UseCase -->|High performance Redis| K
    
    style R fill:#e1ffe1
    style M fill:#e1f5ff
    style K fill:#fffce1
```

## Monitoring Dashboard

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
    
    style A1 fill:#ffe1e1
    style A2 fill:#ffe1e1
    style A3 fill:#ffe1e1
    style A4 fill:#ffe1e1
```

