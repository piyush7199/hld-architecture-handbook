# 4.1.1 Design a Stock Exchange Matching Engine

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions. 
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, order book structure, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Order matching flows, execution paths, failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## Problem Statement

Design an ultra-low-latency stock exchange matching engine that processes financial orders in under 100 microseconds while maintaining ACID guarantees and handling millions of orders per second.

**Real-World Context:**
- **NASDAQ:** 4M messages/sec, <50μs matching latency
- **NYSE:** 10B orders/day, <500μs end-to-end
- **CME Group:** 5M messages/sec, <10μs with FPGA acceleration
- **Coinbase:** 15K orders/sec, 5-50ms (crypto market acceptable)

**Core Technical Challenges:**

1. **Microsecond Latency:** 100μs = 300,000 CPU cycles at 3 GHz
2. **Million QPS:** 1M orders/sec = 1 order every microsecond
3. **Financial Correctness:** No lost trades, no double-spending (ACID)
4. **Durability:** All orders persisted for audit and crash recovery

---

## Requirements and Scale Estimation

### Functional Requirements

**Order Types:**
- Limit Order: Execute at specified price or better
- Market Order: Execute immediately at best available price
- Stop Order: Trigger when threshold price reached
- IOC (Immediate-or-Cancel): Execute or cancel, no partial fills
- FOK (Fill-or-Kill): Complete fill or full cancel

**Matching Rules:**
- Price Priority: Best price executes first (highest bid, lowest ask)
- Time Priority: Within same price, FIFO (first-in-first-out)
- Pro-rata: Some markets split by order size percentage

**Order Book Transparency:**
- Level 1: Top-of-book only (best bid/ask)
- Level 2: Full depth (all price levels, aggregated)
- Level 3: Order-by-order (full transparency, rare)

### Non-Functional Requirements

| Requirement | Target | Industry Benchmark |
|------------|--------|-------------------|
| **Matching Latency (p99)** | <100μs | NASDAQ: 50μs, CME: 10μs |
| **Peak Throughput** | 1M orders/sec | NYSE: ~200K, NASDAQ: ~500K |
| **Availability** | 99.99% | 52 min downtime/year |
| **Consistency** | ACID (strict) | Financial regulations mandate |
| **Durability** | 100% | Audit log replay on crash |
| **Market Data Latency** | <1ms | Broadcast to 1M+ subscribers |

### Scale Estimation

| Metric | Calculation | Result |
|--------|-------------|--------|
| **Peak Orders** | Market open/close spikes | 1,000,000 QPS |
| **Order Size** | 128 bytes each | 128 MB/sec ingestion |
| **Order Book Depth** | 1000 levels × 2 sides | 2000 price levels/symbol |
| **Active Symbols** | NYSE + NASDAQ | 10,000 stocks |
| **Execution Rate** | 10% of orders execute | 100,000 trades/sec |
| **Market Data** | 100K updates/sec | 12.8 MB/sec broadcast |
| **Audit Log** | All orders + trades | 128 MB/sec disk writes |
| **Storage (Daily)** | 1M orders/sec × 86400 sec | 11 TB/day (compressed 1 TB) |

**Latency Budget Breakdown:**

```
Target: 100μs end-to-end

Gateway validation:        5μs  (5%)
Ring buffer transfer:      5μs  (5%)
Matching engine:          40μs (40%)  ← Core bottleneck
Audit log write (async): 10μs (10%)
Market data publish:      40μs (40%)
Total:                   100μs
```

---

## High-Level Architecture

### Component Overview

```
CLIENT LAYER (Traders, Algorithms, Market Makers)
            │
            ▼ FIX Protocol / gRPC (TCP with kernel bypass)
┌──────────────────────────────────────────────────┐
│        LOW-LATENCY GATEWAY                        │
│  - DPDK kernel bypass networking                 │
│  - Fast AuthN (JWT cache)                        │
│  - Order validation                              │
│  - Route to matching engine                      │
└──────────────┬───────────────────────────────────┘
               │
               ▼ Ring Buffer (Lock-Free, Shared Memory)
┌──────────────────────────────────────────────────┐
│   MATCHING ENGINE (Single-Threaded, In-Memory)   │
│                                                   │
│   ORDER BOOK (Red-Black Tree + Linked Lists)     │
│   ┌─────────────┬─────────────┐                  │
│   │  BID SIDE   │  ASK SIDE   │                  │
│   │  (Buy)      │  (Sell)     │                  │
│   ├─────────────┼─────────────┤                  │
│   │ $100.50 [3] │ $100.52 [2] │  ← Best prices   │
│   │ $100.49 [5] │ $100.53 [7] │                  │
│   │ $100.48 [2] │ $100.54 [1] │                  │
│   └─────────────┴─────────────┘                  │
│                                                   │
│   Processing: <100μs per order                   │
│   No locks, No context switches                  │
└──────────────┬───────────────────────────────────┘
               │
     ┌─────────┼─────────┐
     │         │         │
     ▼         ▼         ▼
┌────────┐ ┌──────────┐ ┌─────────┐
│ AUDIT  │ │ MARKET   │ │ LEDGER  │
│ LOG    │ │ DATA     │ │ SERVICE │
│ (WAL)  │ │ FEED     │ │ (ACID)  │
├────────┤ ├──────────┤ ├─────────┤
│ NVMe   │ │ Kafka +  │ │ Postgres│
│ SSD    │ │ WebSocket│ │ Async   │
│ 128MB/s│ │ 100K/sec │ │ Updates │
└────────┘ └──────────┘ └─────────┘
```

**Design Principles:**

1. **Single-Threaded Core:** Avoid lock overhead (50-100μs per lock)
2. **In-Memory State:** All order book data in RAM (no disk I/O)
3. **Async I/O:** Audit log and ledger updates non-blocking
4. **Lock-Free Communication:** Ring buffer between gateway and engine
5. **Kernel Bypass:** DPDK for <5μs network latency

---

## Data Model and Order Book

### Order Structure

```
struct Order {
  order_id: u64,           // Timestamp + sequence (unique)
  user_id: u32,            // Account ID
  symbol: u32,             // Stock symbol (int-encoded for speed)
  side: u8,                // 0=BUY, 1=SELL
  order_type: u8,          // 0=LIMIT, 1=MARKET, 2=STOP
  price: u64,              // Price in cents (no floats!)
  quantity: u32,           // Shares
  filled_quantity: u32,    // Shares executed
  timestamp: u64,          // Nanosecond precision
  time_in_force: u8,       // 0=GTC, 1=IOC, 2=FOK
  status: u8               // 0=PENDING, 1=PARTIAL, 2=FILLED, 3=CANCELLED
}

Size: 64 bytes (cache-line aligned)
```

**Why Fixed-Point Integers (Not Floats)?**

```
❌ Float64: 100.50 × 1000 = 100500.0000000001 (rounding error!)
✅ Int64:   10050 × 1000 = 10050000 (exact!)

Financial Rule: Price in cents (10050 = $100.50)
```

### Order Book Data Structure

**Choice: Red-Black Tree + Linked List**

```
OrderBook {
  symbol: u32,
  
  bids: RBTree<Price, PriceLevel>,  // Buy orders (sorted DESC)
  asks: RBTree<Price, PriceLevel>,  // Sell orders (sorted ASC)
  
  best_bid: Price,  // Cached for O(1) access
  best_ask: Price,
  
  orders: HashMap<OrderID, *Order>,  // Fast lookup for cancellation
}

PriceLevel {
  price: Price,
  orders: LinkedList<*Order>,  // FIFO time priority
  total_quantity: u32
}
```

**Why Red-Black Tree?**

| Operation | Red-Black Tree | Binary Search Tree | B-Tree |
|-----------|---------------|-------------------|--------|
| **Insert** | O(log N) | O(log N) avg, O(N) worst | O(log N) |
| **Delete** | O(log N) | O(log N) avg, O(N) worst | O(log N) |
| **Best Price** | O(1) cached | O(log N) | O(log N) |
| **Balance** | Self-balancing | Manual rebalancing | Auto |

**Cache Locality:** Red-Black Tree better than B-Tree for in-memory (no disk I/O).

*Implementation: [pseudocode.md::insert_order()](pseudocode.md)*

---

## Matching Algorithm (Price-Time Priority)

### Matching Rules

**1. Price Priority:** Best price executes first
- Buy orders: Highest price first
- Sell orders: Lowest price first

**2. Time Priority:** Within same price level, FIFO
- Order 1 at $100.50 executes before Order 2 at $100.50

**Example Matching:**

```
Initial Order Book:
BID (Buy)          │  ASK (Sell)
$100.50 → [O1=100] │  $100.52 → [O5=200]
$100.49 → [O2=200] │  $100.53 → [O6=700]

Incoming: Market SELL 150 shares

Step 1: Match against best bid ($100.50, O1=100 shares)
  → Execute 100 shares at $100.50
  → O1 fully filled (remove from book)
  → Remaining: 50 shares

Step 2: New best bid = $100.49 (O2=200 shares)
  → Execute 50 shares at $100.49
  → O2 partially filled (150 shares remain)
  → Remaining: 0 shares

Result:
  - Executed: 100 @ $100.50, 50 @ $100.49
  - New order book: BID $100.49 → [O2=150], ...
```

### Latency Breakdown

```
Microsecond Accounting (100μs total):

1. Tree traversal (best bid/ask):        10 cycles = 3ns
2. Match loop (check 3 orders):          150 cycles = 50ns
3. Update order book (remove O1):        100 cycles = 33ns
4. Hash map update (order lookup):       50 cycles = 17ns
5. Generate execution events (2):        200 cycles = 67ns
6. Ring buffer enqueue (audit log):      100 cycles = 33ns
7. Market data publish (Kafka):          300 cycles = 100ns
                                         ─────────────────
Total matching engine:                   910 cycles = 303ns

Remaining budget: 100μs - 0.3μs = 99.7μs (network, gateway, etc.)
```

*Implementation: [pseudocode.md::match_order()](pseudocode.md)*

---

## Durability (Write-Ahead Log)

### Problem

In-memory order book is volatile (lost on crash). Need 100% durability for audit and recovery.

### Solution: Audit Log (WAL)

```
LogEntry {
  sequence_number: u64,    // Monotonic
  timestamp: u64,          // Nanoseconds
  event_type: u8,          // 0=ORDER, 1=EXECUTION, 2=CANCEL
  payload: [u8; 256],      // Serialized data
  checksum: u32            // CRC32
}

Size: 272 bytes per entry
Rate: 1M entries/sec
Throughput: 272 MB/sec
Storage: NVMe SSD (3 GB/sec write → no bottleneck)
```

### Async Write Strategy

```
Synchronous (SLOW):
  Order → Match → fsync(audit_log) → Return
  Latency: 10-50μs disk write (blocks matching!)

Asynchronous (FAST):
  Order → Match → async_write(audit_log) → Return
  Latency: 100ns (just copy to ring buffer)
  
Trade-off: Risk losing last 1-10ms if crash before flush
```

**Mitigation: Battery-Backed Write Cache**
- NVMe SSD with capacitor backup
- Guarantees persistence on power loss
- Cost: +$500 per drive

### Crash Recovery

```
1. Read audit log from disk (sequential)
   - 1M orders × 272 bytes = 272 MB
   - Read time: 272 MB ÷ 3 GB/sec = 90ms

2. Replay log (rebuild order book)
   - Process: 1M orders × 1μs = 1 second

Total downtime: ~1 second
```

*Implementation: [pseudocode.md::replay_audit_log()](pseudocode.md)*

---

## Market Data Feed (Real-Time Broadcast)

### Market Data Levels

**Level 1 (Top-of-Book):**
```json
{
  "symbol": "AAPL",
  "best_bid": 100.50,
  "best_ask": 100.52,
  "last_trade": 100.51,
  "timestamp": 1640000000000001
}
```

**Level 2 (Full Depth):**
```json
{
  "bids": [
    {"price": 100.50, "qty": 100},
    {"price": 100.49, "qty": 500}
  ],
  "asks": [
    {"price": 100.52, "qty": 200}
  ]
}
```

### Broadcast Architecture

```
Matching Engine → Kafka (100K msg/sec)
                     ↓
     ┌───────────────┼───────────────┐
     ▼               ▼               ▼
WS Server 1    WS Server 2    WS Server N
(333K clients) (333K clients) (333K clients)

Per-client rate: 100K msg/sec ÷ 1M clients = 0.1 msg/sec
```

**Optimization: Conflation**

```
Raw updates (within 10ms):
  $100.50 → $100.51 → $100.52 → $100.50

Conflated (send only last):
  $100.50

Reduction: 10× fewer messages
```

**Protocol: Protocol Buffers (Not JSON)**

| Format | Size | Encoding Latency |
|--------|------|-----------------|
| JSON | 48 bytes | 5μs |
| ProtoBuf | 16 bytes | 0.5μs |
| **Reduction** | **3× smaller** | **10× faster** |

---

## Single-Threaded Design (LMAX Disruptor)

### Why NOT Multi-Threaded?

**Multi-Threaded Problems:**

```
Thread 1: Lock order book → Match → Unlock
Thread 2: Wait for lock → Match

Lock overhead:
- Mutex acquire/release: 50-100 cycles
- Cache line ping-pong: 100-500 cycles
- Context switch: 1000-10000 cycles

Result: 10-50μs latency (50× slower!)
```

**Single-Threaded Benefits:**

```
✅ No locks (no concurrent access)
✅ No cache coherency issues
✅ No context switches
✅ Predictable latency: <1μs

Trade-off: Limited to ~1-2M orders/sec (single core)
```

### Scaling: Shard by Symbol

```
Symbol AAPL → Matching Engine 1 (CPU core 0)
Symbol GOOG → Matching Engine 2 (CPU core 1)
Symbol MSFT → Matching Engine 3 (CPU core 2)
...

Total: 10K symbols × 1M orders/sec = 10B orders/sec capacity
```

### Ring Buffer (Lock-Free Queue)

```
Producer (Gateway) → Ring Buffer → Consumer (Matching Engine)

Structure:
  - Pre-allocated array (1M slots)
  - Write index (atomic CAS)
  - Read index (atomic CAS)
  - Cache line aligned (no false sharing)

Latency: <100 nanoseconds
```

*Implementation: [pseudocode.md::ring_buffer_enqueue()](pseudocode.md)*

---

## Low-Latency Optimizations

### 1. Kernel Bypass Networking (DPDK)

**Traditional Stack:**
```
Packet → NIC → Kernel → Network Stack → Application
Latency: 10-50μs
```

**DPDK (Kernel Bypass):**
```
Packet → NIC → Application (direct memory access)
Latency: 1-5μs

How:
- Poll NIC (no interrupts)
- Zero-copy (no kernel buffer)
- Huge pages (reduce TLB misses)
```

**Trade-off:** Dedicated CPU cores (100% utilization)

### 2. CPU Affinity (NUMA)

```bash
# Pin matching engine to CPU core 0
taskset -c 0 ./matching_engine

# Allocate memory on same NUMA node
numactl --cpunodebind=0 --membind=0 ./matching_engine
```

**NUMA Benefit:** 30% latency reduction (local RAM vs remote RAM)

### 3. Cache Line Alignment

```cpp
struct Order {
  u64 order_id;
  // ... 56 more bytes ...
} __attribute__((aligned(64)));  // 64-byte cache line

Why? CPU fetches 64 bytes at once (entire cache line)
If struct spans 2 cache lines → 2× memory reads
```

### 4. Object Pool (Avoid malloc)

```
malloc/free: 50-500ns latency

Object Pool:
- Pre-allocate 10M Order objects
- Free list: Linked list
- Allocate: Pop from list (O(1))
- Latency: 5-10ns (10× faster!)
```

*Implementation: [pseudocode.md::object_pool_allocate()](pseudocode.md)*

---

## Bottlenecks and Solutions

### Bottleneck 1: Single-Core CPU Limit

**Problem:** 1-2M orders/sec max on single core

**Solution:** Shard by symbol (10K engines = 10B orders/sec)

### Bottleneck 2: Audit Log Disk I/O

**Problem:** 1M orders/sec × 272 bytes = 272 MB/sec

**Solution:** Batch writes (1000 orders → 1 syscall)
- Reduction: 1000× fewer syscalls
- Trade-off: Larger crash window (10ms)

### Bottleneck 3: Market Data Broadcast

**Problem:** 100K updates/sec × 1M clients = 100B messages/sec

**Solution:** Hierarchical fanout + conflation
- Kafka → 100 WS servers (10K clients each)
- Conflate: 10× reduction (send only last price)

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Float for Prices

**Problem:**
```cpp
double price = 100.50;
double total = price * 1000;  // 100500.0000000001 (error!)
```

**✅ Solution:** Fixed-point integers
```cpp
u64 price_cents = 10050;  // $100.50 = 10050 cents
u64 total = price_cents * 1000;  // 10050000 (exact!)
```

---

### ❌ Anti-Pattern 2: Multi-Threaded with Locks

**Problem:** Lock overhead 50-100μs (too slow!)

**✅ Solution:** Single-threaded + lock-free ring buffer

---

### ❌ Anti-Pattern 3: Sync Audit Log

**Problem:** fsync() blocks matching engine (10-50μs)

**✅ Solution:** Async writes with battery-backed cache

---

### ❌ Anti-Pattern 4: JSON Market Data

**Problem:** 48 bytes, 5μs encoding latency

**✅ Solution:** Protocol Buffers (16 bytes, 0.5μs)

---

## Alternative Approaches

### Alternative 1: Multi-Threaded + Locks

| Factor | Single-Threaded (Chosen) | Multi-Threaded |
|--------|----------------------|---------------|
| Latency | ✅ <100μs | ❌ 1-10ms |
| Throughput | ⚠️ 1-2M/sec | ✅ 5-10M/sec |
| Complexity | ✅ Simple | ❌ Complex |

**When to Use:** Higher throughput priority, 1-10ms latency OK

---

### Alternative 2: Cloud-Based (AWS/GCP)

| Factor | On-Premise (Chosen) | Cloud |
|--------|------------------|-------|
| Latency | ✅ <100μs | ❌ 500μs-5ms |
| Cost | ✅ $100K capex | ⚠️ $50K/month |
| Control | ✅ Full hardware | ❌ Limited |

**When to Use:** Crypto exchanges (5-50ms OK), no capex budget

---

### Alternative 3: FPGA/ASIC Acceleration

**Example: CME Group**
- Hardware matching engine (FPGA)
- Sub-10μs latency
- Cost: $500K-$1M per system

**When to Use:** Ultra-HFT (high-frequency trading), willing to pay premium

---

## Monitoring and Observability

| Metric | Target | Alert |
|--------|--------|-------|
| **Matching Latency (p50)** | <50μs | >100μs |
| **Matching Latency (p99)** | <100μs | >500μs |
| **Audit Log Lag** | <10ms | >100ms |
| **Market Data Lag** | <1ms | >10ms |
| **Order Reject Rate** | <0.1% | >1% |

**Tracing:**

```
Order ID: 12345
  0μs: Gateway receive
  5μs: Ring buffer enqueue
 10μs: Matching engine dequeue
 50μs: Match complete
 60μs: Audit log (async)
100μs: Market data published

Bottleneck: Matching (40μs)
```

---

## Cost Analysis

### Hardware Costs

| Component | Spec | Cost |
|-----------|------|------|
| CPU | Intel Xeon Gold 6348 (28 cores) | $4,000 |
| RAM | 256 GB DDR4-3200 ECC | $2,000 |
| Storage | 2× NVMe SSD 1TB (battery-backed) | $1,000 |
| NIC | Mellanox ConnectX-6 (100 Gbps, DPDK) | $2,000 |
| Motherboard | Dual-socket, PCIe 4.0 | $1,000 |
| **Total per server** | | **$10,000** |

**Cluster:**

| Component | Quantity | Total |
|-----------|----------|-------|
| Matching Engines | 100 servers | $1,000,000 |
| Gateway Servers | 20 servers | $200,000 |
| Market Data Servers | 50 servers | $500,000 |
| **Total** | | **$1,700,000** |

### Operational Costs

| Item | Annual |
|------|--------|
| Colocation | $500,000 |
| Network | $200,000 |
| Personnel (10 engineers) | $1,500,000 |
| **Total** | **$2,200,000/year** |

**Revenue:**
- Trading fees: $0.001 per trade
- 100K trades/sec × 86400 sec/day = 8.64B trades/day
- Revenue: $8.64M/day = **$3.15B/year**

**Profit:** $3.15B - $2.2M = **$3.148B/year** (99.93% margin!)

---

## Real-World Examples

### NASDAQ (US Stock Exchange)

- **Scale:** 4M messages/sec, 10K symbols
- **Latency:** <50μs matching, <500μs end-to-end
- **Architecture:** Single-threaded INET system, kernel bypass

### CME Group (Futures)

- **Scale:** 5M messages/sec, 100K contracts
- **Latency:** <10μs (FPGA-accelerated)
- **Architecture:** Hardware matching engine (ASIC)

### Coinbase (Crypto)

- **Scale:** 15K orders/sec, 500 pairs
- **Latency:** 5-50ms (acceptable for crypto)
- **Architecture:** Multi-threaded Golang, cloud-based

---

## Interview Discussion Points

### Q1: Why not use a database?

**Database:** 1-10ms latency (disk I/O, locks)
**In-Memory Order Book:** <1μs latency

**When Database OK:** Non-latency-critical (>10ms acceptable)

---

### Q2: Handle flash crash?

**Circuit Breakers:**
- If price moves >10% in 5 minutes: Halt trading
- Manual review (30-60 min)
- Resume after investigation

---

### Q3: Matching engine crash?

**Recovery:**
1. Detect crash (1-second timeout)
2. Failover to hot standby (<1 second)
3. Replay audit log (1 second)
4. Resume trading

Total downtime: ~1 second

---

## Trade-offs Summary

| Gain | Sacrifice |
|------|-----------|
| ✅ Ultra-low latency (<100μs) | ❌ Single-core throughput limit |
| ✅ Simple design (no locks) | ❌ Sharding required |
| ✅ Deterministic execution | ⚠️ Custom hardware needed |
| ✅ Strong consistency (ACID) | ❌ Eventual consistency (ledger) |
| ✅ Financial correctness | ⚠️ High infrastructure cost |

**Best For:**
- Institutional trading (HFT, market makers)
- High-volume exchanges (NYSE, NASDAQ)
- Latency-sensitive markets

**NOT For:**
- Retail platforms (latency less critical)
- Low-volume markets (<1K orders/sec)
- Cost-sensitive startups

---

## References

### Academic Papers
- [LMAX Disruptor Pattern](https://lmax-exchange.github.io/disruptor/) - Lock-free messaging
- [Understanding Latency](https://people.eecs.berkeley.edu/~rcs/research/interactive_latency.html) - Jeff Dean's numbers

### Related Chapters
- [2.5.3 Distributed Locking](../../02-components/2.5-algorithms/2.5.3-distributed-locking.md)
- [2.3.1 Asynchronous Communication](../../02-components/2.3-messaging-streaming/2.3.1-asynchronous-communication.md)
- [2.0.3 Real-Time Communication](../../02-components/2.0-communication/2.0.3-real-time-communication.md)

### Open Source
- [Quickfix](https://github.com/quickfix/quickfix) - FIX protocol (C++)
- [Disruptor](https://github.com/LMAX-Exchange/disruptor) - Ring buffer (Java)
- [DPDK](https://www.dpdk.org/) - Kernel bypass framework

### Books
- "Trading and Exchanges" by Larry Harris
- "Flash Boys" by Michael Lewis
- "The Linux Programming Interface" by Michael Kerrisk

