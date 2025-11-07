# Distributed ID Generator (Snowflake) - High-Level Design

## Table of Contents

1. [System Architecture Diagram](#system-architecture-diagram)
2. [Snowflake ID Structure (64-bit)](#snowflake-id-structure-64-bit)
3. [ID Generation Flow](#id-generation-flow)
4. [Worker ID Assignment Strategy](#worker-id-assignment-strategy)
5. [ID Capacity Analysis](#id-capacity-analysis)
6. [Clock Drift Handling](#clock-drift-handling)
7. [Multi-Region Deployment](#multi-region-deployment)
8. [Comparison with Alternative ID Generation Strategies](#comparison-with-alternative-id-generation-strategies)
9. [Monitoring & Observability Dashboard](#monitoring--observability-dashboard)
10. [Scaling Strategy](#scaling-strategy)
11. [Failure Scenarios & Recovery](#failure-scenarios--recovery)
12. [Deployment Architecture](#deployment-architecture)
13. [Performance Characteristics](#performance-characteristics)

---

## System Architecture Diagram

**Flow Explanation:**
Distributed architecture for generating globally unique 64-bit IDs at scale.

**Components:**

1. **ID Generator Nodes:** Multiple stateless nodes (each with unique Worker ID) generate IDs independently using
   Snowflake algorithm
2. **Coordination Service (etcd/ZooKeeper):** Assigns unique Worker IDs (0-1023), health checking, prevents ID conflicts
3. **Load Balancer:** Routes requests evenly across nodes
4. **Monitoring:** Tracks ID generation rate, clock drift, duplicates

**Capacity:** Each node generates 4,096 IDs/ms = ~4.1M IDs/sec. 1024 nodes = 4.2 billion IDs/sec theoretical max.

```mermaid
graph TB
    subgraph "Application Layer"
        AppA[Service A]
        AppB[Service B]
        AppC[Service C]
        AppD[Service D]
    end

    subgraph "Load Balancer"
        LB[Load Balancer<br/>Round Robin]
    end

    AppA --> LB
    AppB --> LB
    AppC --> LB
    AppD --> LB

    subgraph "ID Generator Nodes"
        Node1[ID Generator Node 1<br/>Worker ID: 1<br/>Snowflake Algorithm<br/>Sequence: 0<br/>Last TS: ...]
        Node2[ID Generator Node 2<br/>Worker ID: 2<br/>Snowflake Algorithm<br/>Sequence: 0<br/>Last TS: ...]
        NodeN[ID Generator Node N<br/>Worker ID: N<br/>Snowflake Algorithm<br/>Sequence: 0<br/>Last TS: ...]
    end

    LB --> Node1
    LB --> Node2
    LB --> NodeN

    subgraph "Coordination Service"
        Coord[ZooKeeper/etcd<br/>- Assigns Worker IDs<br/>- Health checks<br/>- Service discovery]
    end

    Node1 -.->|Register & Heartbeat| Coord
    Node2 -.->|Register & Heartbeat| Coord
    NodeN -.->|Register & Heartbeat| Coord

    subgraph "Monitoring"
        Monitor[Prometheus/Grafana<br/>- Track ID generation rate<br/>- Clock drift monitoring<br/>- Duplicate detection]
    end

    Node1 -.->|Metrics| Monitor
    Node2 -.->|Metrics| Monitor
    NodeN -.->|Metrics| Monitor
    style Node1 fill: #e1ffe1
    style Node2 fill: #e1ffe1
    style NodeN fill: #e1ffe1
    style Coord fill: #fffce1
    style Monitor fill: #e1f5ff
```

## Snowflake ID Structure (64-bit)

**Flow Explanation:**
Anatomy of a 64-bit Snowflake ID with embedded metadata.

**Structure:** [1 unused bit | 41-bit timestamp | 10-bit worker ID | 12-bit sequence]

- **Timestamp (41 bits):** Milliseconds since custom epoch → 69 years range
- **Worker ID (10 bits):** Unique node identifier → 1024 max nodes
- **Sequence (12 bits):** Per-millisecond counter → 4096 IDs per ms per node

**Benefits:** Sortable by time, decentralized generation, no coordination needed per ID, embeds generation metadata.

```mermaid
graph LR
subgraph "64-bit ID Breakdown"
Sign[1 bit<br/>Sign<br/>Always 0]
Timestamp[41 bits<br/>Timestamp<br/>Milliseconds since epoch<br/>~69 years]
WorkerID[10 bits<br/>Worker ID<br/>0-1023 nodes<br/>Unique per node]
Sequence[12 bits<br/>Sequence<br/>0-4095<br/>Counter per ms]
end

Sign --> Timestamp
Timestamp --> WorkerID
WorkerID --> Sequence

style Sign fill: #ffe1e1
style Timestamp fill: #e1f5ff
style WorkerID fill: #e1ffe1
style Sequence fill:#fffce1
```

### ID Composition Example

```
Binary Layout:
┌─┬──────────────────────────────────────────┬──────────┬────────────┐
│0│ 00000010101010101010101010101010101010101│ 0000000001│ 000000000001│
└─┴──────────────────────────────────────────┴──────────┴────────────┘
 ↑              ↑                                  ↑            ↑
 │              │                                  │            │
Sign          Timestamp                        Worker ID    Sequence
(0)    (Milliseconds since epoch)              (1)          (1)

Result: 123456789012345678 (decimal)
```

## ID Generation Flow

**Flow Explanation:**
Step-by-step process for generating a single ID.

**Steps:**

1. Get current timestamp (milliseconds)
2. If same as last timestamp: increment sequence (0-4095)
3. If sequence overflow (>4095): wait for next millisecond
4. If new timestamp: reset sequence to 0
5. Check clock backwards (critical error if detected)
6. Compose: (timestamp << 22) | (worker_id << 12) | sequence
7. Return 64-bit ID

**Performance:** Sub-microsecond generation latency, lock-free.

```mermaid
flowchart TD
    Start([Request ID]) --> GetTS[Get Current<br/>Timestamp ms]
    GetTS --> CheckBackward{Timestamp <<br/>Last Timestamp?}
    CheckBackward -->|Yes| CheckTolerance{Drift <<br/>5ms?}
    CheckTolerance -->|Yes| Wait[Wait for<br/>clock to catch up]
    Wait --> GetTS
    CheckTolerance -->|No| Error[Error: Clock<br/>moved backwards]
    Error --> End1([End])
    CheckBackward -->|No| CheckSame{Timestamp ==<br/>Last Timestamp?}
    CheckSame -->|Yes| IncrSeq[Increment<br/>Sequence]
    IncrSeq --> CheckOverflow{Sequence<br/>overflow?}
    CheckOverflow -->|Yes| WaitNext[Wait for<br/>next millisecond]
    WaitNext --> GetTS
    CheckOverflow -->|No| Construct
    CheckSame -->|No| ResetSeq[Reset Sequence<br/>to 0]
    ResetSeq --> Construct
    Construct[Construct ID:<br/>timestamp left-shift 22 OR<br/>worker_id left-shift 12 OR<br/>sequence]
    Construct --> Update[Update<br/>Last Timestamp]
    Update --> Return[Return ID]
    Return --> End2([End])
    style Error fill: #ffe1e1
    style Construct fill: #e1ffe1
    style Return fill: #e1f5ff
```

## Worker ID Assignment Strategy

**Flow Explanation:**
How nodes acquire unique Worker IDs on startup using coordination service.

**Strategy:** Node starts → Registers with etcd → etcd assigns available ID (0-1023) → Node keeps lease alive via
heartbeat → If node dies, ID freed for reuse

**Protection:** etcd ensures no duplicate IDs. TTL-based leases prevent orphaned IDs.

```mermaid
graph TB
    subgraph "Startup Process"
        Start([Node Starts]) --> ConnectCoord[Connect to<br/>ZooKeeper/etcd]
        ConnectCoord --> CheckExisting{Existing<br/>Worker ID?}
        CheckExisting -->|Yes| Reuse[Reuse existing<br/>Worker ID]
        Reuse --> StartService
        CheckExisting -->|No| FindAvailable[Find available<br/>Worker ID 0-1023]
        FindAvailable --> CreateLease[Create lease<br/>with TTL 30s]
        CreateLease --> Assign[Assign Worker ID]
        Assign --> StartHeartbeat[Start heartbeat<br/>thread every 10s]
        StartHeartbeat --> StartService[Start ID<br/>Generation Service]
    end

    subgraph "Heartbeat Loop"
        StartService --> Heartbeat[Send Heartbeat<br/>to ZooKeeper]
        Heartbeat --> RefreshLease[Refresh Lease]
        RefreshLease --> Sleep[Sleep 10s]
        Sleep --> Heartbeat
    end

    subgraph "Failure Handling"
        NodeDown[Node Goes Down] --> LeaseExpire[Lease expires<br/>after 30s]
        LeaseExpire --> ReleaseID[Worker ID<br/>released]
        ReleaseID --> Available[Available for<br/>other nodes]
    end

    style Assign fill: #e1ffe1
    style StartService fill: #e1f5ff
    style NodeDown fill: #ffe1e1
```

## ID Capacity Analysis

**Flow Explanation:**
Capacity breakdown showing theoretical vs practical throughput limits.

**Per Node:** 4,096 IDs/ms = 4.1M IDs/sec (sequence bits limit)
**Cluster (1024 nodes):** 4.2 billion IDs/sec theoretical → ~1-2 billion IDs/sec practical (network/CPU overhead)

**69-year timeline:** 41-bit timestamp provides 69 years from custom epoch. After that, either reset epoch or migrate to
new ID system.

```mermaid
graph TB
    subgraph "Timestamp Capacity (41 bits)"
        TS[2^41 milliseconds<br/>= 2,199,023,255,551 ms<br/>= 69.7 years]
    end

    subgraph "Worker ID Capacity (10 bits)"
        WID[2^10 = 1,024 nodes<br/>Can support up to<br/>1,024 concurrent generators]
    end

    subgraph "Sequence Capacity (12 bits)"
        SEQ[2^12 = 4,096 IDs<br/>per millisecond<br/>per node]
    end

    subgraph "Total Capacity"
        Total[Per Node:<br/>4,096,000 IDs/second<br/><br/>Cluster 1024 nodes:<br/>4,194,304,000 IDs/second<br/>= 4.2 billion IDs/sec]
    end

    TS --> Total
    WID --> Total
    SEQ --> Total
    style Total fill: #e1ffe1
```

## Clock Drift Handling

**Flow Explanation:**
Critical safety mechanism to prevent duplicate IDs from clock issues.

**Detection:** Compare current timestamp vs last timestamp. If current < last → Clock moved backwards!
**Strategies:**

- **Small drift (<5ms):** Wait for clock to catch up (tolerable)
- **Large drift (>5ms):** Throw error, stop generating (safe but service interruption)

**Prevention:** NTP synchronization (keep drift <10ms), monotonic clocks, monitoring alerts.

```mermaid
flowchart TD
    Start([Generate ID]) --> CheckTime[Read System Time]
    CheckTime --> Compare{Compare with<br/>Last Timestamp}
    Compare -->|Time moved forward| Normal[Normal flow:<br/>Generate ID]
    Normal --> Success([Return ID])
    Compare -->|Same time| SameTime[Increment sequence<br/>within same ms]
    SameTime --> CheckSeq{Sequence<br/>< 4096?}
    CheckSeq -->|Yes| Success
    CheckSeq -->|No| WaitNext[Spin-wait for<br/>next millisecond]
    WaitNext --> CheckTime
    Compare -->|Time moved backward| CheckDrift{Drift amount}
    CheckDrift -->|< 5ms| Tolerate[Wait and tolerate<br/>small drift]
    Tolerate --> WaitCatch[Sleep drift_ms]
    WaitCatch --> CheckTime
    CheckDrift -->|> = 5ms| Refuse[Refuse to generate:<br/>Clock moved backwards]
    Refuse --> LogError[Log critical error]
    LogError --> AlertOps[Alert operations team]
    AlertOps --> Fail([Throw Exception])
    style Success fill: #e1ffe1
    style Fail fill: #ffe1e1
    style Tolerate fill: #fffce1
```

## Multi-Region Deployment

**Flow Explanation:**
Global deployment with region-specific Worker ID ranges to avoid conflicts.

**ID Allocation:**

- US Region: Worker IDs 0-341
- EU Region: Worker IDs 342-683
- APAC Region: Worker IDs 684-1023

**Benefits:** Each region operates independently, low latency, no cross-region coordination needed. IDs globally unique
by design.

```mermaid
graph TB
    subgraph "Region: US-East"
        USNodes[ID Generators<br/>Worker IDs: 0-99]
        USCoord[ZooKeeper<br/>US Cluster]
        USNodes -.-> USCoord
    end

    subgraph "Region: EU-West"
        EUNodes[ID Generators<br/>Worker IDs: 100-199]
        EUCoord[ZooKeeper<br/>EU Cluster]
        EUNodes -.-> EUCoord
    end

    subgraph "Region: AP-South"
        APNodes[ID Generators<br/>Worker IDs: 200-299]
        APCoord[ZooKeeper<br/>AP Cluster]
        APNodes -.-> APCoord
    end

    subgraph "Global Applications"
        USApp[US Applications] --> USNodes
        EUApp[EU Applications] --> EUNodes
        APApp[AP Applications] --> APNodes
    end

    Note[Each region operates<br/>independently<br/>No cross-region<br/>coordination needed]
    style USNodes fill: #e1ffe1
    style EUNodes fill: #e1f5ff
    style APNodes fill: #fffce1
    style Note fill: #ffe1e1
```

## Comparison with Alternative ID Generation Strategies

**Flow Explanation:**
Trade-off matrix comparing Snowflake to other ID generation approaches.

**Snowflake:** Sortable, decentralized, high throughput, reveals timestamp
**UUID:** Truly random, no coordination, not sortable, 128-bit (larger)
**Database Auto-Increment:** Simple, sequential, requires DB coordination (bottleneck), single point of failure

**When to use Snowflake:** Distributed systems needing high throughput, time-ordered IDs, with multiple generators.

```mermaid
graph TB
subgraph "Snowflake"
SF[✅ 64-bit compact<br/>✅ Time-sorted<br/>✅ High performance<br/>❌ Requires coordination]
end

subgraph "UUID v4"
UUID4[✅ No coordination<br/>✅ Globally unique<br/>❌ 128-bit large<br/>❌ Not sortable]
end

subgraph "UUID v7"
UUID7[✅ Time-sorted<br/>✅ No coordination<br/>❌ 128-bit large<br/>❌ Less efficient storage]
end

subgraph "Database Auto-Increment"
AutoInc[✅ Strictly monotonic<br/>✅ Simple<br/>❌ Single point of failure<br/>❌ Doesn't scale]
end

UseCase{Use Case Requirements}

UseCase -->|High scale + sortable|SF
UseCase -->|Simple + unique|UUID4
UseCase -->|Sortable UUID|UUID7
UseCase -->|Small scale|AutoInc

style SF fill: #e1ffe1
style UUID4 fill: #e1f5ff
style UUID7 fill: #fffce1
style AutoInc fill: #ffe1e1
```

## Monitoring & Observability Dashboard

**Flow Explanation:**
Key metrics for operational health and early problem detection.

**Critical Metrics:**

1. **Generation Rate:** IDs/sec per node (watch for drops)
2. **Clock Drift:** Monitor NTP drift (<10ms healthy)
3. **Duplicate Rate:** Should be 0 (alert immediately if >0)
4. **Sequence Exhaustion:** How often waiting for next ms
5. **Worker ID Conflicts:** etcd assignment failures

**Alerts:** Clock drift >50ms, duplicates detected, generation rate drop >50%, etcd unavailable.

```mermaid
graph TB
subgraph "Key Metrics"
M1[ID Generation Rate<br/>Baseline ±20%<br/>Alert: > 2x baseline]
M2[Latency p99<br/>Target: < 1ms<br/>Alert: > 10ms]
M3[Clock Drift<br/>Target: < 10ms<br/>Alert: > 50ms]
M4[Sequence Resets<br/>Per new ms<br/>Alert: > 100/sec]
M5[Clock Backwards<br/>Target: 0<br/>Alert: > 0]
M6[Worker ID Conflicts<br/>Target: 0<br/>Alert: > 0]
end

subgraph "Data Sources"
Prometheus[Prometheus]
IDNodes[ID Generator Nodes]
Coord[Coordination Service]
end

IDNodes -->|Scrape metrics| Prometheus
Coord -->|Worker ID registry|Prometheus

Prometheus --> M1
Prometheus --> M2
Prometheus --> M3
Prometheus --> M4
Prometheus --> M5
Prometheus --> M6

subgraph "Visualization & Alerts"
Grafana[Grafana Dashboard]
AlertMgr[Alert Manager]
PagerDuty[PagerDuty/Slack]
end

Prometheus --> Grafana
Prometheus --> AlertMgr
AlertMgr --> PagerDuty

style M5 fill: #ffe1e1
style M6 fill: #ffe1e1
style AlertMgr fill: #fffce1
```

## Scaling Strategy

**Flow Explanation:**
Progressive scaling from single node to globally distributed cluster.

**Phase 1:** Single node (Worker ID 0) → 4.1M IDs/sec → Good for small apps
**Phase 2:** 10 nodes (Worker IDs 0-9) → 41M IDs/sec → Regional deployment
**Phase 3:** 100 nodes across regions → 410M IDs/sec → Multi-region
**Phase 4:** 1000 nodes globally → 4.1B IDs/sec → Massive scale

**When to scale:** Monitor generation rate. Add nodes when approaching 80% capacity.

```mermaid
graph LR
    subgraph "Phase 1: Low Load (< 10K IDs/sec)"
        P1[2-3 Nodes<br/>Redundancy for HA]
    end

    subgraph "Phase 2: Medium Load (10-100K IDs/sec)"
        P2[10-20 Nodes<br/>Distribute load]
    end

    subgraph "Phase 3: High Load (100K-1M IDs/sec)"
        P3[50-100 Nodes<br/>High throughput]
    end

    subgraph "Phase 4: Extreme Load (> 1M IDs/sec)"
        P4[100+ Nodes<br/>Maximum scale]
    end

    P1 -->|Load increases| P2
    P2 -->|Load increases| P3
    P3 -->|Load increases| P4
    Note[Linear scaling:<br/>Each node generates<br/>~10K IDs/sec independently]

style P1 fill:#e1ffe1
style P2 fill: #e1f5ff
style P3 fill: #fffce1
style P4 fill: #ffe1e1
```

## Failure Scenarios & Recovery

**Flow Explanation:**
How system handles various failure modes gracefully.

**Node Failure:** Other nodes continue generating IDs → No impact (stateless)
**etcd Failure:** Nodes keep running with existing Worker IDs → New nodes can't join → Deploy redundant etcd cluster
**Clock Drift:** Node detects drift → Stops generating → Alerts ops → Fix NTP sync
**Network Partition:** Nodes in each partition continue independently → No duplicates (unique Worker IDs)

**Recovery:** Failed nodes restart → Get new Worker ID from etcd → Resume generation.

```mermaid
graph TB
    subgraph "Scenario 1: Single Node Failure"
        S1[Node Crashes] --> S1A[Lease expires 30s]
        S1A --> S1B[Worker ID released]
        S1B --> S1C[Load redistributed<br/>to other nodes]
        S1C --> S1D[Node restarts,<br/>gets new Worker ID]
    end

    subgraph "Scenario 2: ZooKeeper Partition"
        S2[ZK becomes unreachable] --> S2A[Existing nodes<br/>continue with<br/>current Worker IDs]
        S2A --> S2B[New nodes<br/>cannot start]
        S2B --> S2C[Fix network partition]
        S2C --> S2D[Resume normal<br/>operations]
    end

    subgraph "Scenario 3: Clock Drift"
        S3[NTP correction] --> S3A{Backward<br/>or forward?}
        S3A -->|Forward| S3B[Larger gaps in IDs<br/>but no duplicates]
        S3A -->|Backward > 5ms| S3C[Reject ID generation<br/>log critical error]
        S3C --> S3D[Alert ops team<br/>fix clock sync]
    end

    style S1D fill: #e1ffe1
    style S2D fill: #e1f5ff
    style S3C fill: #ffe1e1
```

## Deployment Architecture

**Flow Explanation:**
Production deployment with redundancy and monitoring.

**Components:**

- **3+ ID Generator nodes:** Behind load balancer for high availability
- **3-node etcd cluster:** Quorum-based for fault tolerance
- **Prometheus + Grafana:** Real-time monitoring and alerting
- **NTP servers:** Time synchronization
- **Load Balancer:** Health checks and routing

**HA Design:** No single point of failure. etcd quorum tolerates 1 node failure. ID generators are stateless.

```mermaid
graph TB
    subgraph "Production Environment"
        subgraph "Availability Zone 1"
            AZ1_Gen[ID Generator 1-3]
            AZ1_ZK[ZooKeeper 1]
        end

        subgraph "Availability Zone 2"
            AZ2_Gen[ID Generator 4-6]
            AZ2_ZK[ZooKeeper 2]
        end

        subgraph "Availability Zone 3"
            AZ3_Gen[ID Generator 7-9]
            AZ3_ZK[ZooKeeper 3]
        end
    end

    AZ1_Gen -.->|Register| AZ1_ZK
    AZ2_Gen -.->|Register| AZ2_ZK
    AZ3_Gen -.->|Register| AZ3_ZK
    AZ1_ZK <-->|Quorum| AZ2_ZK
    AZ2_ZK <-->|Quorum| AZ3_ZK
    AZ3_ZK <-->|Quorum| AZ1_ZK

    subgraph "Clients"
        Apps[Applications]
    end

    Apps -->|Load Balanced| AZ1_Gen
    Apps -->|Load Balanced| AZ2_Gen
    Apps -->|Load Balanced| AZ3_Gen
    Note[Benefits:<br/>- AZ failure tolerated<br/>- No single point of failure<br/>- Automatic failover]
    style AZ1_Gen fill: #e1ffe1
    style AZ2_Gen fill: #e1ffe1
    style AZ3_Gen fill: #e1ffe1
```

## Performance Characteristics

**Flow Explanation:**
Performance profile across key metrics.

**Throughput:** 4.1M IDs/sec per node (CPU-bound, not network-bound)
**Latency:** P50: 0.1ms, P99: 0.5ms, P999: 2ms (mostly in-memory operations)
**CPU:** ~5% per node at 1M IDs/sec (lightweight algorithm)
**Memory:** <100MB per node (stateless, minimal state)
**Network:** <1 Mbps per node (tiny payloads, 8 bytes per ID)

**Bottleneck:** Sequence exhaustion at 4096 IDs/ms. Solution: Add more nodes.

```mermaid
graph LR
    subgraph "Latency Profile"
        L1[P50: < 0.5ms<br/>Typical case]
        L2[P95: < 1ms<br/>Most requests]
        L3[P99: < 5ms<br/>Rare cases]
        L4[P99.9: < 10ms<br/>Sequence exhaustion]
    end

subgraph "Throughput Profile"
T1[Per Node:<br/>~10K IDs/sec<br/>typical]
T2[Per Node:<br/>~100K IDs/sec<br/>burst capacity]
T3[Cluster:<br/>Linear scaling<br/>with nodes]
end

subgraph "Reliability"
R1[Availability:<br/>99.999%<br/>with multi-AZ]
R2[Uniqueness:<br/>100%<br/>guaranteed]
R3[Ordering:<br/>Rough time-based<br/>k-sortable]
end

style L1 fill: #e1ffe1
style T1 fill: #e1f5ff
style R1 fill: #fffce1
```

