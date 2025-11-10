# Stock Exchange Matching Engine - High-Level Design

## Table of Contents

1. [Complete System Architecture](#1-complete-system-architecture)
2. [Order Book Data Structure](#2-order-book-data-structure)
3. [Single-Threaded Matching Engine](#3-single-threaded-matching-engine)
4. [Ring Buffer (Lock-Free Queue)](#4-ring-buffer-lock-free-queue)
5. [DPDK Kernel Bypass Networking](#5-dpdk-kernel-bypass-networking)
6. [Audit Log (Write-Ahead Log)](#6-audit-log-write-ahead-log)
7. [Market Data Fanout Architecture](#7-market-data-fanout-architecture)
8. [Crash Recovery Flow](#8-crash-recovery-flow)
9. [Sharding by Symbol](#9-sharding-by-symbol)
10. [CPU Affinity and NUMA](#10-cpu-affinity-and-numa)
11. [Object Pool Memory Management](#11-object-pool-memory-management)
12. [Multi-Region Deployment](#12-multi-region-deployment)
13. [Circuit Breaker System](#13-circuit-breaker-system)
14. [Monitoring and Observability](#14-monitoring-and-observability)

---

## 1. Complete System Architecture

**Flow Explanation:**

This diagram shows the end-to-end architecture of the stock exchange matching engine, optimized for sub-100μs latency.

**Key Components:**

1. **Client Layer:** Traders, HFT algorithms, market makers connect via FIX protocol
2. **Low-Latency Gateway:** Validates orders, uses DPDK kernel bypass for <5μs network latency
3. **Ring Buffer:** Lock-free shared memory queue (<100ns latency) for inter-process communication
4. **Matching Engine:** Single-threaded core processes orders from in-memory order book
5. **Audit Log:** Asynchronous writes to NVMe SSD for durability (272 MB/sec throughput)
6. **Market Data Feed:** Kafka + WebSocket broadcasts to 1M+ subscribers
7. **Ledger Service:** ACID database for user balances (async updates, eventual consistency)

**Performance Characteristics:**

- **End-to-end latency:** <100μs (p99)
- **Throughput:** 1M orders/sec per matching engine
- **Scalability:** Shard by symbol (10K engines = 10B orders/sec)

**Trade-offs:**

- ✅ **Ultra-low latency:** Single-threaded design avoids lock overhead
- ❌ **Single-core limit:** Must shard by symbol to scale beyond 1-2M orders/sec
- ⚠️ **Eventual consistency:** Ledger updates async (user balances lag by 1-10ms)

```mermaid
graph TB
    subgraph Clients
        C1[HFT Firm]
        C2[Retail Trader]
        C3[Market Maker]
        C4[Algo Bot]
    end

    subgraph Gateway Cluster
        G1[Gateway 1<br/>DPDK Kernel Bypass]
        G2[Gateway 2<br/>DPDK Kernel Bypass]
        G3[Gateway N<br/>DPDK Kernel Bypass]
    end

    subgraph Matching Engine Core
        RB[Ring Buffer<br/>Lock-Free Queue<br/>1M slots<br/>Latency under 100ns]
        ME[Matching Engine<br/>Single-Threaded<br/>In-Memory Order Book]
        OB[Order Book<br/>Red-Black Tree<br/>Price-Time Priority]
    end

    subgraph Storage and Persistence
        AL[Audit Log WAL<br/>NVMe SSD<br/>128 MB per sec<br/>Async Writes]
        LS[Ledger Service<br/>PostgreSQL<br/>ACID Guarantees<br/>Async Updates]
    end

    subgraph Market Data
        K[Kafka Cluster<br/>100 partitions<br/>100K msg per sec]
        WS1[WebSocket Server 1<br/>333K clients]
        WS2[WebSocket Server 2<br/>333K clients]
        WS3[WebSocket Server N<br/>333K clients]
    end

    C1 -->|FIX Protocol<br/>TCP| G1
    C2 -->|FIX Protocol<br/>TCP| G2
    C3 -->|FIX Protocol<br/>TCP| G3
    C4 -->|FIX Protocol<br/>TCP| G1
    G1 -->|Shared Memory| RB
    G2 -->|Shared Memory| RB
    G3 -->|Shared Memory| RB
    RB -->|Single Consumer| ME
    ME -->|Process| OB
    ME -->|Async Write<br/>Batched| AL
    ME -->|Events| K
    K -->|Consume| LS
    K -->|Broadcast| WS1
    K -->|Broadcast| WS2
    K -->|Broadcast| WS3
    WS1 -->|WebSocket| C1
    WS2 -->|WebSocket| C2
    WS3 -->|WebSocket| C3
    style ME fill: #ff9999
    style OB fill: #ffcccc
    style RB fill: #99ccff
    style AL fill: #99ff99
    style K fill: #ffcc99
```

---

## 2. Order Book Data Structure

**Flow Explanation:**

This diagram illustrates the in-memory order book data structure using Red-Black Tree + Linked Lists for optimal
matching performance.

**Structure:**

1. **Bid Side (Buy Orders):** Red-Black Tree sorted in descending order (highest price first)
2. **Ask Side (Sell Orders):** Red-Black Tree sorted in ascending order (lowest price first)
3. **Price Level:** Each tree node contains a linked list of orders at that price (FIFO time priority)
4. **Order Lookup:** HashMap for O(1) cancellation by order_id

**Algorithmic Complexity:**

- **Insert Order:** O(log N) tree insert + O(1) linked list append = O(log N)
- **Best Bid/Ask:** O(1) cached at tree root
- **Match Order:** O(log N) tree traversal + O(K) matches (K = orders matched)
- **Cancel Order:** O(1) hash lookup + O(log N) tree update

**Memory Layout:**

- **Order:** 64 bytes (cache-line aligned)
- **Tree Node:** 40 bytes (parent, left, right, color, data)
- **10K orders:** 640 KB (fits in L2 cache: 1-2 MB)

**Trade-offs:**

- ✅ **Fast matching:** O(log N) for best price, O(1) for execution
- ✅ **Cache-friendly:** Sequential access within price level (linked list)
- ❌ **Memory overhead:** Tree nodes + linked list pointers (~40% overhead)

```mermaid
graph TB
    subgraph Order Book Structure
        direction LR
        OB[OrderBook<br/>Symbol AAPL]

        subgraph Bid Side Buy Orders
            BT[Red-Black Tree<br/>Sorted DESC]
            B1[Price 100.50<br/>Total Qty 100]
            B2[Price 100.49<br/>Total Qty 700]
            B3[Price 100.48<br/>Total Qty 200]
            BL1[Linked List<br/>Order1 100 shares]
            BL2[Linked List<br/>Order2 200 shares<br/>Order3 500 shares]
            BL3[Linked List<br/>Order4 200 shares]
        end

        subgraph Ask Side Sell Orders
            AT[Red-Black Tree<br/>Sorted ASC]
            A1[Price 100.52<br/>Total Qty 200]
            A2[Price 100.53<br/>Total Qty 700]
            A3[Price 100.54<br/>Total Qty 100]
            AL1[Linked List<br/>Order5 200 shares]
            AL2[Linked List<br/>Order6 300 shares<br/>Order7 400 shares]
            AL3[Linked List<br/>Order8 100 shares]
        end

        HM[HashMap<br/>OrderID to Order Pointer<br/>For O 1 Cancellation]
    end

    OB --> BT
    OB --> AT
    OB --> HM
    BT --> B1
    BT --> B2
    BT --> B3
    B1 --> BL1
    B2 --> BL2
    B3 --> BL3
    AT --> A1
    AT --> A2
    AT --> A3
    A1 --> AL1
    A2 --> AL2
    A3 --> AL3
    HM -.->|Lookup| BL1
    HM -.->|Lookup| BL2
    HM -.->|Lookup| AL1
    style B1 fill: #99ff99
    style A1 fill: #ff9999
    style HM fill: #ffcc99
```

---

## 3. Single-Threaded Matching Engine

**Flow Explanation:**

This diagram shows why single-threaded design is faster than multi-threaded for ultra-low-latency matching.

**Single-Threaded Benefits:**

1. **No Locks:** No mutex acquire/release overhead (50-100 cycles saved)
2. **No Cache Coherency:** All data in L1 cache (no ping-pong between cores)
3. **No Context Switches:** No OS scheduler interruption (1000-10000 cycles saved)
4. **Predictable Latency:** Sequential execution, no contention

**Multi-Threaded Problems:**

1. **Lock Contention:** Multiple threads competing for order book lock
2. **False Sharing:** Cache line invalidation when threads access adjacent memory
3. **Priority Inversion:** Low-priority thread holds lock, blocking high-priority thread

**Performance Comparison:**

| Design                 | Latency (p50) | Latency (p99) | Throughput       |
|------------------------|---------------|---------------|------------------|
| Single-Threaded        | 50μs          | 100μs         | 1-2M orders/sec  |
| Multi-Threaded + Locks | 500μs         | 5ms           | 5-10M orders/sec |

**When to Use Multi-Threaded:**

- Latency requirement >1ms (locks acceptable)
- Throughput priority (>10M orders/sec needed)
- Complex business logic (not pure matching)

**Trade-offs:**

- ✅ **Minimal latency:** <100μs consistently
- ❌ **Throughput limit:** Single core maxes at 1-2M orders/sec
- ⚠️ **Scaling:** Must shard by symbol (deploy multiple engines)

```mermaid
graph TB
    subgraph Single-Threaded Design Chosen
        RB1[Ring Buffer<br/>Lock-Free Queue]
        ME1[Matching Engine<br/>Single Thread<br/>CPU Core 0]
        OB1[Order Book<br/>In-Memory<br/>No Locks Needed]
        PERF1[Performance<br/>Latency 50 to 100 microseconds p99<br/>Throughput 1 to 2M orders per sec<br/>No Lock Overhead]
    end

    subgraph Multi-Threaded Alternative Not Chosen
        Q2[Queue<br/>With Locks]
        ME2A[Matching Thread 1<br/>Competing for Lock]
        ME2B[Matching Thread 2<br/>Waiting for Lock]
        ME2C[Matching Thread 3<br/>Waiting for Lock]
        OB2[Order Book<br/>Mutex Protected<br/>Lock Overhead]
        PERF2[Performance<br/>Latency 500 microseconds to 5ms p99<br/>Throughput 5 to 10M orders per sec<br/>Lock Contention]
    end

    RB1 --> ME1
    ME1 --> OB1
    OB1 --> PERF1
    Q2 --> ME2A
    Q2 --> ME2B
    Q2 --> ME2C
    ME2A -->|Lock Wait| OB2
    ME2B -->|Lock Wait| OB2
    ME2C -->|Lock Wait| OB2
    OB2 --> PERF2
    style ME1 fill: #99ff99
    style PERF1 fill: #99ff99
    style OB2 fill: #ff9999
    style PERF2 fill: #ff9999
```

---

## 4. Ring Buffer (Lock-Free Queue)

**Flow Explanation:**

This diagram illustrates the LMAX Disruptor ring buffer pattern for lock-free communication between gateway and matching
engine.

**How It Works:**

1. **Pre-Allocated Array:** 1M slots allocated at startup (no dynamic allocation)
2. **Write Index:** Producer (gateway) atomically increments write pointer
3. **Read Index:** Consumer (matching engine) atomically increments read pointer
4. **Cache Line Alignment:** Write index and read index on separate cache lines (no false sharing)

**Performance:**

- **Enqueue:** Atomic CAS (compare-and-swap) + memory copy = 50-100 cycles = 33ns at 3 GHz
- **Dequeue:** Atomic CAS + memory copy = 50-100 cycles = 33ns
- **Total:** <100ns round-trip (10× faster than mutex lock)

**Key Optimizations:**

1. **Single Producer, Single Consumer:** No contention (no CAS retry loop)
2. **Power-of-2 Size:** Modulo becomes bitwise AND (fast: `index & (SIZE-1)`)
3. **Huge Pages:** Reduce TLB misses (2 MB pages instead of 4 KB)

**Trade-offs:**

- ✅ **Ultra-low latency:** <100ns per operation
- ✅ **Lock-free:** No blocking, no priority inversion
- ❌ **Fixed size:** Must pre-allocate (cannot grow dynamically)
- ⚠️ **Backpressure:** If full, producer must block or drop orders

```mermaid
graph TB
    subgraph Ring Buffer Lock-Free Queue
        direction LR
        P[Producer Gateway<br/>Write Index 5]
        RB[Ring Buffer<br/>Pre-Allocated 1M Slots<br/>Cache Line Aligned]

        subgraph Buffer Slots
            S0[Slot 0<br/>Order Data]
            S1[Slot 1<br/>Order Data]
            S2[Slot 2<br/>Order Data]
            S3[Slot 3<br/>Empty]
            S4[Slot 4<br/>Empty]
            S5[Slot 5<br/>Write Here]
        end

        C[Consumer Matching Engine<br/>Read Index 2]
    end

    subgraph Cache Line Layout
        CL0[Cache Line 0<br/>Write Index<br/>64 bytes aligned]
        CL1[Cache Line 1<br/>Read Index<br/>64 bytes aligned]
        CL2[Cache Lines 2 to N<br/>Data Slots<br/>64 bytes each]
    end

    P -->|Atomic CAS<br/>Increment Write Index| RB
    RB -->|Write Order| S5
    RB -->|Read Order| S2
    C -->|Atomic CAS<br/>Increment Read Index| RB
    P -.->|No False Sharing| CL0
    C -.->|No False Sharing| CL1
    S0 -.-> CL2
    style S5 fill: #99ff99
    style S2 fill: #ffcc99
    style CL0 fill: #99ccff
    style CL1 fill: #ff99cc
```

---

## 5. DPDK Kernel Bypass Networking

**Flow Explanation:**

This diagram compares traditional Linux networking stack with DPDK kernel bypass for low-latency packet processing.

**Traditional Stack Problems:**

1. **Interrupt Overhead:** NIC generates interrupt, CPU context switches (1000+ cycles)
2. **Kernel Copy:** Packet copied from NIC buffer to kernel buffer to application buffer (3× memory copy)
3. **Network Stack:** TCP/IP processing in kernel (checksums, fragmentation, routing)
4. **Latency:** 10-50μs total (too slow for HFT)

**DPDK (Data Plane Development Kit):**

1. **Poll Mode:** Application polls NIC directly (no interrupts, no context switches)
2. **Zero-Copy:** Packet data accessed directly in NIC memory (DMA)
3. **Userspace Stack:** TCP/IP processing in application (optimized for latency)
4. **Latency:** 1-5μs (10× faster!)

**Requirements:**

- **Dedicated CPU Cores:** Polling loop runs at 100% CPU utilization
- **Huge Pages:** Reduce TLB misses (2 MB pages)
- **NIC Support:** Intel X710, Mellanox ConnectX-6 (DPDK drivers)

**Trade-offs:**

- ✅ **10× latency reduction:** 50μs → 5μs
- ❌ **CPU overhead:** 100% utilization for polling (1-2 cores dedicated)
- ❌ **Complexity:** Custom userspace TCP/IP stack required

```mermaid
graph TB
    subgraph Traditional Networking Latency 10 to 50 microseconds
        NIC1[NIC Network Interface Card]
        INT1[Hardware Interrupt<br/>Context Switch<br/>1000 plus cycles]
        KERN1[Linux Kernel<br/>Network Stack<br/>TCP IP Processing]
        COPY1[Copy to User Buffer<br/>Kernel to User Space]
        APP1[Application Gateway<br/>Receives Order]
        NIC1 -->|Interrupt| INT1
        INT1 --> KERN1
        KERN1 -->|Copy| COPY1
        COPY1 --> APP1
    end

    subgraph DPDK Kernel Bypass Latency 1 to 5 microseconds
        NIC2[NIC Network Interface Card<br/>DMA Direct Memory Access]
        POLL[Poll Mode Driver<br/>No Interrupts<br/>100 percent CPU]
        ZERO[Zero-Copy<br/>Direct Access to NIC Memory]
        APP2[Application Gateway<br/>Userspace TCP IP Stack]
        NIC2 -->|DMA| ZERO
        APP2 -->|Poll| POLL
        POLL -->|Read| ZERO
    end

    style APP1 fill: #ff9999
    style APP2 fill: #99ff99
    style POLL fill: #99ccff
    style ZERO fill: #99ff99
```

---

## 6. Audit Log (Write-Ahead Log)

**Flow Explanation:**

This diagram shows the async write strategy for the audit log to achieve durability without blocking the matching
engine.

**Synchronous Write Problem:**

```
Order → Match → fsync() disk write → Return to client
                   ↑
               10-50μs latency (blocks matching engine!)
```

**Async Write Solution:**

```
Order → Match → Copy to ring buffer → Return to client
                     ↓
                Background thread → Batch write → fsync()
```

**Write Strategy:**

1. **Matching Engine:** Copies log entry to ring buffer (100ns, non-blocking)
2. **Background Thread:** Consumes ring buffer, batches 1000 entries
3. **Disk Write:** Single fsync() for 1000 entries (amortizes disk latency)
4. **Throughput:** 1M entries/sec × 272 bytes = 272 MB/sec (well within NVMe 3 GB/sec)

**Crash Recovery:**

1. **Read Log:** Sequential read (3 GB/sec) = 90ms for 272 MB
2. **Replay:** 1M orders × 1μs = 1 second
3. **Total Downtime:** ~1 second

**Battery-Backed Cache:**

- NVMe SSD with capacitor backup
- Guarantees persistence on power loss
- Trade-off: +$500 per drive

**Performance:**

- ✅ **Non-blocking writes:** Matching engine latency unaffected
- ✅ **High throughput:** 272 MB/sec sustainable
- ⚠️ **Crash window:** Last 1-10ms of data at risk (mitigated by battery backup)

```mermaid
graph TB
    subgraph Matching Engine Process
        ME[Matching Engine<br/>Process Order]
        RB[Ring Buffer<br/>Audit Log Queue<br/>Lock-Free]
        ME -->|Copy Entry<br/>100ns Non - Blocking| RB
    end

    subgraph Async Write Thread
        BG[Background Thread<br/>Consume Queue]
        BATCH[Batch Buffer<br/>1000 Entries<br/>272 KB]
        RB -->|Dequeue| BG
        BG -->|Accumulate| BATCH
    end

    subgraph Storage Layer
        NVME[NVMe SSD<br/>Battery-Backed Cache<br/>3 GB per sec Write]
        WAL[Write-Ahead Log<br/>Append-Only<br/>Sequential Writes]
        BATCH -->|fsync Every 1ms<br/>Batched Write| NVME
        NVME --> WAL
    end

    subgraph Crash Recovery
        CRASH[System Crash<br/>Matching Engine Dies]
        READ[Read Audit Log<br/>Sequential Read<br/>90ms for 272 MB]
        REPLAY[Replay Events<br/>Rebuild Order Book<br/>1 Second]
        RESUME[Resume Trading<br/>Total Downtime 1 Second]
        CRASH --> READ
        READ --> REPLAY
        REPLAY --> RESUME
    end

    WAL -.->|On Crash| READ
    style ME fill: #99ccff
    style RB fill: #ffcc99
    style NVME fill: #99ff99
    style REPLAY fill: #ff99cc
```

---

## 7. Market Data Fanout Architecture

**Flow Explanation:**

This diagram shows the hierarchical fanout architecture for broadcasting market data to 1M+ subscribers.

**Challenge:**

- **100K updates/sec** × **1M subscribers** = 100 billion messages/sec (impossible from single server)

**Solution: Hierarchical Fanout**

1. **Matching Engine:** Publishes to Kafka (100K msg/sec)
2. **Kafka Cluster:** 100 partitions (shard by symbol)
3. **WebSocket Servers:** 100 servers, each handles 10K clients
4. **Per-Client Rate:** 100K msg/sec ÷ 1M clients = 0.1 msg/sec (manageable)

**Optimizations:**

**Conflation:**

```
Raw updates (within 10ms):
  $100.50 → $100.51 → $100.52 → $100.50

Conflated (send only last):
  $100.50

Reduction: 10× fewer messages
```

**Protocol Buffers (Not JSON):**

- JSON: 48 bytes, 5μs encoding
- ProtoBuf: 16 bytes, 0.5μs encoding
- **Result:** 3× smaller, 10× faster

**Latency Breakdown:**

```
Matching Engine → Kafka:    100μs
Kafka → WS Server:           500μs
WS Server → Client:          500μs
Total:                       1.1ms (acceptable)
```

**HFT Optimization:**

- Direct feed from matching engine (bypass Kafka)
- Co-location: HFT servers in same datacenter
- Latency: <10μs

```mermaid
graph TB
    subgraph Matching Engine
        ME[Matching Engine<br/>100K updates per sec<br/>Executions plus Top-of-Book]
    end

    subgraph Kafka Cluster
        K[Kafka<br/>100 Partitions<br/>Shard by Symbol]
        P1[Partition 1<br/>AAPL]
        P2[Partition 2<br/>GOOG]
        P3[Partition N<br/>MSFT]
    end

    subgraph Market Data Service Layer
        MDS1[Market Data Service 1<br/>Filter by Subscription]
        MDS2[Market Data Service 2<br/>Filter by Subscription]
        MDS3[Market Data Service N<br/>Filter by Subscription]
    end

    subgraph WebSocket Server Cluster
        WS1[WS Server 1<br/>10K Clients<br/>Protocol Buffers]
        WS2[WS Server 2<br/>10K Clients<br/>Protocol Buffers]
        WS3[WS Server N<br/>10K Clients<br/>Protocol Buffers]
    end

    subgraph Clients
        C1[HFT Firm<br/>Subscribes AAPL]
        C2[Retail Trader<br/>Subscribes GOOG]
        C3[Market Maker<br/>Subscribes All]
    end

    ME -->|Publish| K
    K --> P1
    K --> P2
    K --> P3
    P1 --> MDS1
    P2 --> MDS2
    P3 --> MDS3
    MDS1 --> WS1
    MDS2 --> WS2
    MDS3 --> WS3
    WS1 -->|WebSocket<br/>Conflated<br/>10x Reduction| C1
    WS2 -->|WebSocket<br/>Conflated| C2
    WS3 -->|WebSocket<br/>Conflated| C3
    ME -.->|Direct Feed<br/>Co - Location<br/>under 10 microseconds| C1
    style ME fill: #ff9999
    style K fill: #ffcc99
    style WS1 fill: #99ccff
    style C1 fill: #99ff99
```

---

## 8. Crash Recovery Flow

**Flow Explanation:**

This diagram illustrates the crash recovery process using audit log replay to rebuild the order book.

**Recovery Steps:**

1. **Detect Crash:** Heartbeat monitor (1-second timeout)
2. **Read Audit Log:** Sequential read from NVMe SSD (90ms for 272 MB)
3. **Replay Events:** Process each log entry to rebuild order book state
4. **Validate State:** Verify best bid/ask, total volumes
5. **Resume Trading:** Accept new orders (<1 second total downtime)

**Hot Standby Failover:**

- **Primary Engine:** Processes orders, writes audit log
- **Standby Engine:** Receives audit log (async replication), maintains shadow order book
- **On Crash:** Promote standby to primary (<1 second failover)

**Event Replay:**

```
Log Entry 1: ORDER_RECEIVED → Insert into order book
Log Entry 2: ORDER_MATCHED → Remove from book, generate execution
Log Entry 3: ORDER_CANCELLED → Remove from book
...
Log Entry N: Rebuild complete → Resume trading
```

**Performance:**

- **Downtime:** ~1 second (hot standby) or 10-60 seconds (cold start)
- **Data Loss:** 0 orders (with battery-backed cache)
- **Recovery Rate:** 1M orders/sec × 1μs = 1 second replay time

```mermaid
graph TB
    subgraph Normal Operation
        ME[Matching Engine<br/>Primary]
        AL[Audit Log<br/>NVMe SSD<br/>Write-Ahead Log]
        ST[Hot Standby<br/>Shadow Order Book<br/>Async Replication]
        ME -->|Write Every Order<br/>Every Trade| AL
        AL -.->|Replicate<br/>Async| ST
    end

    subgraph Crash Detected
        CRASH[Primary Crashes<br/>Heartbeat Timeout<br/>1 Second]
        ALERT[Alert System<br/>Notify Operators<br/>Auto-Failover]
        CRASH --> ALERT
    end

    subgraph Recovery Process
        PROMOTE[Promote Standby<br/>to Primary<br/>under 1 Second]
        READ[Read Audit Log<br/>Sequential Read<br/>90ms for 272 MB]
        REPLAY[Replay Events<br/>Rebuild Order Book<br/>1M orders in 1 Second]
        VALIDATE[Validate State<br/>Check Best Bid Ask<br/>Verify Volumes]
        RESUME[Resume Trading<br/>Accept New Orders<br/>Total Downtime 1 Second]
        ALERT -->|Hot Standby| PROMOTE
        ALERT -->|Cold Start| READ
        PROMOTE --> RESUME
        READ --> REPLAY
        REPLAY --> VALIDATE
        VALIDATE --> RESUME
    end

    ST -.->|Failover| PROMOTE
    AL -.->|Read| READ
    style ME fill: #99ff99
    style CRASH fill: #ff9999
    style RESUME fill: #99ff99
    style ST fill: #ffcc99
```

---

## 9. Sharding by Symbol

**Flow Explanation:**

This diagram shows how to scale beyond single-core throughput limits by deploying separate matching engines per stock
symbol.

**Scaling Challenge:**

- **Single Engine:** 1-2M orders/sec (single CPU core limit)
- **Target:** 10B orders/sec (10,000 symbols × 1M orders/sec)

**Sharding Strategy:**

1. **Symbol-Based Partitioning:** Each stock symbol assigned to dedicated matching engine
2. **Independent Engines:** Separate CPU core, memory, order book per symbol
3. **Gateway Routing:** Gateway routes order to correct engine based on symbol
4. **No Cross-Symbol Coordination:** Each engine operates independently

**Routing Table:**

```
Symbol AAPL → Engine 1 (CPU Core 0, Port 10001)
Symbol GOOG → Engine 2 (CPU Core 1, Port 10002)
Symbol MSFT → Engine 3 (CPU Core 2, Port 10003)
...
Symbol TSLA → Engine 10000 (CPU Core 27, Port 20000)
```

**Cross-Symbol Orders:**

- Rare: <1% of orders (e.g., pairs trading, spreads)
- Handle via slow path: Two-phase commit across engines
- Latency: 1-10ms (acceptable for rare case)

**Benefits:**

- ✅ **Linear Scalability:** 10K symbols × 1M orders/sec = 10B orders/sec
- ✅ **Fault Isolation:** One engine crash doesn't affect others
- ❌ **Resource Overhead:** Each engine needs dedicated core + memory

```mermaid
graph TB
    subgraph Gateway Layer
        G[Gateway<br/>Route by Symbol<br/>Routing Table Hash Map]
    end

    subgraph Matching Engine Cluster
        ME1[Matching Engine 1<br/>CPU Core 0<br/>Symbol AAPL<br/>1M orders per sec]
        ME2[Matching Engine 2<br/>CPU Core 1<br/>Symbol GOOG<br/>1M orders per sec]
        ME3[Matching Engine 3<br/>CPU Core 2<br/>Symbol MSFT<br/>1M orders per sec]
        ME4[Matching Engine N<br/>CPU Core 27<br/>Symbol TSLA<br/>1M orders per sec]
    end

    subgraph Order Books In-Memory
        OB1[Order Book AAPL<br/>Bid Ask Levels<br/>Independent State]
        OB2[Order Book GOOG<br/>Bid Ask Levels<br/>Independent State]
        OB3[Order Book MSFT<br/>Bid Ask Levels<br/>Independent State]
        OB4[Order Book TSLA<br/>Bid Ask Levels<br/>Independent State]
    end

    G -->|Route Symbol AAPL| ME1
    G -->|Route Symbol GOOG| ME2
    G -->|Route Symbol MSFT| ME3
    G -->|Route Symbol TSLA| ME4
    ME1 --> OB1
    ME2 --> OB2
    ME3 --> OB3
    ME4 --> OB4

    subgraph Total Capacity
        CAP[Total Throughput<br/>10K symbols times 1M orders per sec<br/>equals 10 Billion orders per sec]
    end

    ME1 -.-> CAP
    ME2 -.-> CAP
    ME3 -.-> CAP
    ME4 -.-> CAP
    style ME1 fill: #99ff99
    style ME2 fill: #99ccff
    style ME3 fill: #ffcc99
    style ME4 fill: #ff99cc
    style CAP fill: #99ff99
```

---

## 10. CPU Affinity and NUMA

**Flow Explanation:**

This diagram illustrates CPU affinity and NUMA (Non-Uniform Memory Access) optimizations for minimizing memory latency.

**NUMA Architecture:**

- **2 CPU Sockets:** Each socket has local RAM
- **Local Access:** CPU accesses local RAM in 50-100ns
- **Remote Access:** CPU accesses remote RAM in 150-300ns (3× slower!)

**Optimization Strategy:**

1. **Pin Matching Engine:** Bind thread to specific CPU core (taskset)
2. **Allocate Local Memory:** Allocate order book data on same NUMA node
3. **Disable Hyperthreading:** Avoid sharing L1/L2 cache between threads

**Commands:**

```bash
# Pin to CPU core 0 (Socket 0)
taskset -c 0 ./matching_engine

# Allocate memory on NUMA node 0
numactl --cpunodebind=0 --membind=0 ./matching_engine

# Disable hyperthreading
echo 0 > /sys/devices/system/cpu/cpu1/online
```

**Performance Impact:**

| Optimization                  | Latency Reduction |
|-------------------------------|-------------------|
| CPU Pinning (avoid migration) | 10-20%            |
| NUMA Local Memory             | 30-40%            |
| Disable Hyperthreading        | 5-10%             |
| **Total**                     | **~50% faster**   |

**Trade-offs:**

- ✅ **30% latency reduction:** Remote RAM avoided
- ✅ **Predictable performance:** No CPU migration, no cache eviction
- ❌ **Reduced flexibility:** Cannot use all cores for other tasks

```mermaid
graph TB
    subgraph Server Hardware 2 Sockets
        subgraph Socket 0
            CPU0[CPU Cores 0 to 15<br/>L1 L2 Cache]
            RAM0[Local RAM 128 GB<br/>Fast Access 50 to 100ns]
            CPU0 -.->|Local Access<br/>Fast| RAM0
        end

        subgraph Socket 1
            CPU1[CPU Cores 16 to 31<br/>L1 L2 Cache]
            RAM1[Remote RAM 128 GB<br/>Slow Access 150 to 300ns]
            CPU1 -.->|Local Access<br/>Fast| RAM1
        end

        QPI[QPI Quick Path Interconnect<br/>Inter-Socket Communication<br/>High Latency]
        CPU0 -.->|Remote Access<br/>Slow 3x| QPI
        QPI -.-> RAM1
    end

    subgraph Optimization Strategy
        PIN[Pin Matching Engine<br/>to CPU Core 0<br/>taskset command]
        ALLOC[Allocate Order Book<br/>on Local RAM Socket 0<br/>numactl command]
        DISABLE[Disable Hyperthreading<br/>Avoid L1 Cache Sharing]
        PIN --> CPU0
        ALLOC --> RAM0
        DISABLE --> CPU0
    end

    subgraph Performance Impact
        BEFORE[Before NUMA Optimization<br/>Latency 150 to 300ns<br/>Remote RAM Access]
        AFTER[After NUMA Optimization<br/>Latency 50 to 100ns<br/>Local RAM Access<br/>30 percent Faster]
        BEFORE -->|Apply Optimizations| AFTER
    end

    style CPU0 fill: #99ff99
    style RAM0 fill: #99ff99
    style AFTER fill: #99ff99
    style QPI fill: #ff9999
```

---

## 11. Object Pool Memory Management

**Flow Explanation:**

This diagram shows the object pool pattern for avoiding malloc/free overhead in the hot path.

**malloc/free Problems:**

1. **Latency:** 50-500ns per allocation (syscall overhead)
2. **Fragmentation:** Long-running process accumulates fragmentation
3. **Non-Deterministic:** Latency varies based on heap state

**Object Pool Solution:**

1. **Pre-Allocate:** Allocate 10M Order objects at startup
2. **Free List:** Maintain linked list of available slots
3. **Allocate:** Pop from free list (O(1), 5-10ns)
4. **Deallocate:** Push to free list (O(1), 5-10ns)

**Implementation:**

```cpp
struct ObjectPool {
  Order* pool;           // Pre-allocated array (10M orders)
  Order* free_list;      // Linked list of available slots
  
  Order* allocate() {
    Order* obj = free_list;
    free_list = free_list->next;
    return obj;  // 5-10ns
  }
  
  void deallocate(Order* obj) {
    obj->next = free_list;
    free_list = obj;  // 5-10ns
  }
};
```

**Memory Layout:**

- **Pool Size:** 10M orders × 64 bytes = 640 MB
- **Cache-Line Aligned:** Each order on 64-byte boundary
- **NUMA Local:** Allocated on same node as matching engine

**Performance:**

- ✅ **10× faster:** 5-10ns vs 50-500ns (malloc)
- ✅ **Predictable:** Constant-time allocation
- ❌ **Fixed capacity:** Cannot grow beyond 10M orders

```mermaid
graph TB
    subgraph Object Pool Pre-Allocated
        POOL[Object Pool<br/>10M Order Objects<br/>640 MB<br/>Pre-Allocated at Startup]
        FREE[Free List<br/>Linked List<br/>Available Slots]
        USED[Used Objects<br/>Active Orders<br/>In Order Book]
    end

    subgraph Allocation Fast Path
        ALLOC[allocate Order<br/>Pop from Free List<br/>O 1 Operation<br/>5 to 10ns]
        POOL --> FREE
        FREE -->|Pop Head| ALLOC
        ALLOC --> USED
    end

    subgraph Deallocation Fast Path
        DEALLOC[deallocate Order<br/>Push to Free List<br/>O 1 Operation<br/>5 to 10ns]
        USED -->|Order Filled or Cancelled| DEALLOC
        DEALLOC -->|Push Head| FREE
    end

    subgraph Comparison with malloc
        MALLOC[malloc free<br/>50 to 500ns Latency<br/>Heap Fragmentation<br/>Non-Deterministic]
        POOL_PERF[Object Pool<br/>5 to 10ns Latency<br/>No Fragmentation<br/>Constant Time<br/>10x Faster]
        MALLOC -.->|Avoid| POOL_PERF
    end

    style ALLOC fill: #99ff99
    style DEALLOC fill: #99ff99
    style POOL_PERF fill: #99ff99
    style MALLOC fill: #ff9999
```

---

## 12. Multi-Region Deployment

**Flow Explanation:**

This diagram shows a multi-region deployment strategy for global exchanges with failover capabilities.

**Architecture:**

1. **Primary Region (US East):** Handles all trading, writes audit log
2. **Standby Region (US West):** Receives audit log replication, maintains shadow order book
3. **Disaster Recovery Region (EU):** Cold standby, only activated on regional failure

**Replication Strategy:**

- **Audit Log Replication:** Async replication (<10ms lag)
- **Order Book Snapshot:** Hourly snapshots to S3 (backup)
- **Failover Time:** <5 seconds (promote standby to primary)

**Latency Considerations:**

- **US East Client → US East Exchange:** 1-5ms
- **US West Client → US East Exchange:** 30-50ms (cross-coast)
- **EU Client → US East Exchange:** 80-120ms (cross-Atlantic)

**Optimization for Global Clients:**

- **Smart Order Routing:** Route to nearest region for market data
- **Execution Always Primary:** All trades execute in primary region (consistency)

**Trade-offs:**

- ✅ **High availability:** <5 seconds failover on regional outage
- ✅ **Disaster recovery:** EU region as last resort
- ❌ **Cost:** 3× infrastructure (primary + 2 standbys)

```mermaid
graph TB
    subgraph Primary Region US East
        ME1[Matching Engine<br/>Active Primary<br/>All Trading]
        AL1[Audit Log<br/>NVMe SSD<br/>Source of Truth]
        DB1[Ledger Database<br/>PostgreSQL<br/>User Balances]
        ME1 --> AL1
        ME1 --> DB1
    end

    subgraph Standby Region US West
        ME2[Matching Engine<br/>Hot Standby<br/>Shadow Order Book]
        AL2[Audit Log Replica<br/>Async Replication<br/>under 10ms Lag]
        DB2[Ledger Replica<br/>Read-Only<br/>Async Replication]
        AL1 -.->|Replicate<br/>Async| AL2
        AL2 --> ME2
        DB1 -.->|Replicate| DB2
    end

    subgraph Disaster Recovery EU
        ME3[Matching Engine<br/>Cold Standby<br/>Only on Failure]
        AL3[Audit Log Backup<br/>S3 Snapshots<br/>Hourly]
        AL1 -.->|Backup<br/>Hourly| AL3
        AL3 -.->|Restore| ME3
    end

    subgraph Failover Process
        FAIL[Primary Fails<br/>Regional Outage]
        PROMOTE[Promote US West<br/>to Primary<br/>under 5 Seconds]
        RESUME[Resume Trading<br/>New Primary<br/>US West]
        FAIL --> PROMOTE
        ME2 -.->|Failover| PROMOTE
        PROMOTE --> RESUME
    end

    subgraph Global Clients
        C1[US Clients<br/>1 to 5ms Latency]
        C2[EU Clients<br/>80 to 120ms Latency]
        C1 --> ME1
        C2 --> ME1
    end

    style ME1 fill: #99ff99
    style ME2 fill: #ffcc99
    style ME3 fill: #99ccff
    style RESUME fill: #99ff99
```

---

## 13. Circuit Breaker System

**Flow Explanation:**

This diagram illustrates the circuit breaker system for handling flash crashes and abnormal market conditions.

**Trigger Conditions:**

1. **Price Movement:** >10% change in 5 minutes
2. **Volume Spike:** 100× normal volume
3. **Order Imbalance:** 95% buy or 95% sell orders
4. **Manual Trigger:** Operator command

**Circuit Breaker Levels:**

**Level 1 (Warning):**

- Slow down order acceptance (rate limiting)
- Alert market participants
- Continue trading

**Level 2 (Pause):**

- Reject new orders for 5 minutes
- Cancel aggressive orders (market orders)
- Preserve limit orders in book

**Level 3 (Halt):**

- Stop all trading
- Cancel all orders
- Notify regulators
- Manual resume (30-60 minutes)

**Implementation:**

```
Monitor Thread (every 1 second):
  1. Check last_trade_price vs VWAP (volume-weighted average)
  2. If delta > 10%: Trigger Level 2
  3. If delta > 20%: Trigger Level 3
  4. Broadcast halt message to all clients
```

**Recovery:**

- **Level 1:** Auto-resume after 1 minute
- **Level 2:** Auto-resume after 5 minutes
- **Level 3:** Manual operator approval required

```mermaid
graph TB
    subgraph Normal Trading
        ME[Matching Engine<br/>Processing Orders]
        MON[Monitor Thread<br/>Every 1 Second<br/>Check Price Movement]
        ME -.->|Report Stats| MON
    end

    subgraph Trigger Conditions
        COND1[Price Movement<br/>greater than 10 percent in 5 Minutes]
        COND2[Volume Spike<br/>100x Normal]
        COND3[Order Imbalance<br/>95 percent Buy or Sell]
        COND4[Manual Trigger<br/>Operator Command]
        MON -->|Detect| COND1
        MON -->|Detect| COND2
        MON -->|Detect| COND3
    end

    subgraph Circuit Breaker Levels
        L1[Level 1 Warning<br/>Slow Down Orders<br/>Rate Limiting<br/>Continue Trading]
        L2[Level 2 Pause<br/>Reject New Orders<br/>5 Minute Pause<br/>Cancel Market Orders]
        L3[Level 3 Halt<br/>Stop All Trading<br/>Cancel All Orders<br/>Notify Regulators<br/>Manual Resume]
        COND1 -->|5 to 10 percent| L1
        COND1 -->|10 to 20 percent| L2
        COND1 -->|greater than 20 percent| L3
        COND2 --> L2
        COND3 --> L2
        COND4 --> L3
    end

    subgraph Recovery Process
        AUTO1[Level 1 Auto-Resume<br/>After 1 Minute]
        AUTO2[Level 2 Auto-Resume<br/>After 5 Minutes]
        MANUAL[Level 3 Manual Resume<br/>Operator Approval<br/>30 to 60 Minutes]
        L1 --> AUTO1
        L2 --> AUTO2
        L3 --> MANUAL
        AUTO1 --> ME
        AUTO2 --> ME
        MANUAL --> ME
    end

    style ME fill: #99ff99
    style L3 fill: #ff9999
    style MANUAL fill: #ffcc99
```

---

## 14. Monitoring and Observability

**Flow Explanation:**

This diagram shows the comprehensive monitoring and observability system for detecting performance degradation and
failures.

**Key Metrics:**

**Latency Metrics:**

- **p50 (Median):** <50μs (half of orders faster than this)
- **p99:** <100μs (99% of orders faster than this)
- **p99.9:** <1ms (worst-case outliers)

**Throughput Metrics:**

- **Orders/sec:** Target 1M, alert if <500K
- **Executions/sec:** Target 100K, alert if <50K
- **Order Reject Rate:** Target <0.1%, alert if >1%

**System Health:**

- **CPU Utilization:** Target 70-80% (matching engine), alert if >95%
- **Memory Usage:** Target <80%, alert if >90%
- **Audit Log Lag:** Target <10ms, alert if >100ms

**Tracing:**

```
Order ID: 12345 (Distributed Trace)
  0μs:    Gateway receive
  5μs:    Ring buffer enqueue
 10μs:    Matching engine dequeue
 50μs:    Match complete (executed)
 60μs:    Audit log written (async)
100μs:    Market data published

Bottleneck: Matching (40μs of 100μs)
Action: Optimize order book traversal
```

**Alerting:**

1. **Critical (Page Oncall):** Matching engine down, p99 >500μs
2. **Warning (Slack):** p99 >200μs, CPU >90%
3. **Info (Dashboard):** Normal metrics, trends

**Tools:**

- **Prometheus:** Metric collection (1-second granularity)
- **Grafana:** Dashboards (real-time visualization)
- **Jaeger:** Distributed tracing (per-order trace)
- **PagerDuty:** Alerting (oncall rotation)

```mermaid
graph TB
    subgraph Matching Engine Instrumentation
        ME[Matching Engine<br/>Emit Metrics<br/>Every 1 Second]
        LAT[Latency Metrics<br/>p50 p99 p99.9<br/>Histogram]
        THRU[Throughput Metrics<br/>Orders per sec<br/>Executions per sec<br/>Counter]
        SYS[System Metrics<br/>CPU Memory<br/>Audit Log Lag<br/>Gauge]
        ME --> LAT
        ME --> THRU
        ME --> SYS
    end

    subgraph Monitoring Stack
        PROM[Prometheus<br/>Time-Series Database<br/>1 Second Granularity]
        GRAF[Grafana<br/>Real-Time Dashboards<br/>Visualization]
        JAEG[Jaeger<br/>Distributed Tracing<br/>Per-Order Trace]
        LAT --> PROM
        THRU --> PROM
        SYS --> PROM
        PROM --> GRAF
        ME --> JAEG
    end

    subgraph Alerting
        ALERT[Alert Manager<br/>Rule Evaluation<br/>Every 10 Seconds]
        CRIT[Critical Alerts<br/>p99 greater than 500 microseconds<br/>Engine Down<br/>Page Oncall]
        WARN[Warning Alerts<br/>p99 greater than 200 microseconds<br/>CPU greater than 90 percent<br/>Slack Notification]
        INFO[Info Alerts<br/>Normal Metrics<br/>Dashboard Only]
        PROM --> ALERT
        ALERT --> CRIT
        ALERT --> WARN
        ALERT --> INFO
        CRIT -->|PagerDuty| PD[Oncall Engineer<br/>Immediate Response]
        WARN -->|Slack| SL[Team Channel<br/>Investigation]
    end

    style CRIT fill: #ff9999
    style WARN fill: #ffcc99
    style GRAF fill: #99ccff
    style JAEG fill: #99ff99
```

