# Stock Exchange Matching Engine - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

An ultra-low-latency stock exchange matching engine that matches orders in <100μs (microseconds), handles 1M orders/sec
peak throughput, guarantees ACID transactions for financial correctness, and publishes real-time market data to millions
of subscribers.

**Real-World Examples:**

- **NASDAQ:** Handles 4M messages/sec, <50μs matching latency
- **NYSE:** Processes 10B orders/day, <500μs end-to-end latency
- **CME Group:** Futures exchange, <10μs matching engine latency
- **Coinbase:** Crypto exchange, 15K orders/sec, ~5ms latency

**The Core Challenge:**

Traditional databases are too slow (millisecond latency). We need:

1. **In-Memory Processing:** All order book state in RAM
2. **Single-Threaded Engine:** Avoid lock overhead (no mutexes)
3. **Kernel Bypass Networking:** Direct memory access (DPDK, RDMA)
4. **Custom Data Structures:** Price-time priority queues

---

## Requirements & Scale

### Functional Requirements

1. **Order Types:**
    - Limit Order: Buy/Sell at specific price or better
    - Market Order: Buy/Sell at best available price
    - Stop Order: Trigger when price reaches threshold

2. **Order Matching:**
    - Price-Time Priority: Highest bid, lowest ask, FIFO within price level
    - Immediate-or-Cancel (IOC): Execute immediately or cancel
    - Fill-or-Kill (FOK): Execute entire order or cancel

3. **Order Book:**
    - Level 1: Best bid/ask only
    - Level 2: Full depth (all price levels)
    - Level 3: Individual orders (full transparency)

4. **Trade Execution:**
    - Update buyer and seller accounts
    - Generate execution report
    - Publish to market data feed

### Non-Functional Requirements

| Requirement             | Target        | Rationale                                |
|-------------------------|---------------|------------------------------------------|
| **Matching Latency**    | <100μs (p99)  | Competitive with NYSE, NASDAQ            |
| **Throughput**          | 1M orders/sec | Handle peak trading (market open, close) |
| **Availability**        | 99.99%        | 52 minutes downtime/year                 |
| **Consistency**         | ACID          | No double-spending, no lost trades       |
| **Durability**          | 100%          | All orders/trades persisted (audit log)  |
| **Market Data Latency** | <1ms          | Real-time price discovery                |

### Scale Estimation

| Metric               | Assumption                   | Calculation                       | Result                 |
|----------------------|------------------------------|-----------------------------------|------------------------|
| **Peak Orders**      | 1M orders/sec                | Market open/close spikes          | 1M QPS                 |
| **Order Size**       | 128 bytes per order          | Order ID, symbol, price, quantity | 128 MB/sec ingestion   |
| **Order Book Depth** | 1000 price levels per symbol | Bid + ask sides                   | 2000 levels total      |
| **Symbols**          | 10,000 stocks traded         | NYSE + NASDAQ combined            | 10K order books        |
| **Execution Rate**   | 10% of orders execute        | 90% passive orders sit in book    | 100K trades/sec        |
| **Market Data Feed** | 100K updates/sec             | Executions + top-of-book changes  | 12.8 MB/sec broadcast  |
| **Audit Log**        | 1M orders × 128 bytes        | Write-ahead log for durability    | 128 MB/sec disk writes |

**Key Insight:** 100μs latency = 100,000 nanoseconds. At 3 GHz CPU, that's only 300,000 CPU cycles. Every operation must
be optimized.

---

## Key Components

| Component               | Responsibility                                 | Technology              | Scalability     |
|-------------------------|------------------------------------------------|-------------------------|-----------------|
| **Low-Latency Gateway** | TCP/gRPC endpoint, fast auth, order validation | DPDK (kernel bypass)    | Horizontal      |
| **Matching Engine**     | Single-threaded order matching                 | C++ (custom)            | Shard by symbol |
| **Order Book**          | In-memory price-time priority queue            | Red-Black Tree          | Per-symbol      |
| **Audit Log**           | Write-ahead log for durability                 | NVMe SSD (async writes) | Single writer   |
| **Market Data Feed**    | Real-time broadcast to subscribers             | Kafka + WebSocket       | Horizontal      |

---

## Architecture Overview

**Flow Summary:**

1. **Client** sends order via FIX Protocol/gRPC to **Gateway**
2. **Gateway** validates order, routes to **Matching Engine** via shared memory (ring buffer)
3. **Matching Engine** (single-threaded) processes order:
    - Match against opposite side (price-time priority)
    - Update order book (add/remove)
    - Generate execution events
4. **Matching Engine** writes to **Audit Log** (async, batched)
5. **Matching Engine** publishes to **Market Data Feed** (Kafka)
6. **Market Data Servers** broadcast to clients via WebSocket

**Key Design:**

- **Single-threaded matching engine:** No locks, no context switches
- **In-memory order book:** All state in RAM (no disk I/O)
- **Ring buffer:** Lock-free queue between gateway and engine
- **Async audit log:** Non-blocking writes (batched)

---

## Matching Algorithm (Price-Time Priority)

### Matching Rules

**Price Priority:**

- Buy orders: Highest price first
- Sell orders: Lowest price first

**Time Priority:**

- Within same price level: FIFO (first-in-first-out)

**Example:**

```
Order Book:
BID (Buy)          |  ASK (Sell)
$100.50 → [O1]     |  $100.52 → [O5]
$100.49 → [O2, O3] |  $100.53 → [O6, O7]

Incoming: Market SELL 100 shares

Matching:
1. Best bid = $100.50 (O1)
2. Execute: O1 buys 100 shares at $100.50
3. Remove O1 from book (fully filled)
4. New best bid = $100.49 (O2)
```

**Latency Breakdown:**

- Tree traversal: 10 CPU cycles
- Matching loop: 50 CPU cycles per match (×3 = 150 cycles)
- Total: 160 CPU cycles = **53 nanoseconds** at 3 GHz

---

## Durability and Fault Tolerance (Audit Log)

### Write-Ahead Log (WAL)

**Problem:** In-memory order book volatile (lost on crash). Need durability.

**Solution:** Append-only audit log persists all events before processing.

**Async Write Strategy:**

```
1. Order received → Copy to ring buffer (in-memory)
2. Matching engine processes order (matching, execution)
3. Async thread writes to audit log (batched writes)
4. If crash: Replay audit log from last checkpoint
```

**Why Async?**

- Synchronous write: 10-50μs latency (too slow!)
- Async write: 0.1μs latency (just copy to buffer)
- Trade-off: Risk of losing last 1-10ms of data if crash before flush

**Mitigation: Battery-Backed Write Cache**

- NVMe SSD with capacitor backup
- Guarantees writes persisted even on power loss
- Cost: $500-$1000 per drive

---

## Low-Latency Optimizations

### 1. Single-Threaded Design

**Why?**

- No mutex locks (50-100μs overhead eliminated)
- No context switches (CPU cache stays hot)
- Predictable latency (no thread contention)

**Trade-off:** Can't utilize multiple CPU cores (but can shard by symbol)

### 2. Kernel Bypass Networking (DPDK)

**Why?**

- Traditional TCP: Kernel overhead (10-50μs)
- DPDK: Direct memory access (0.1-1μs)

**How:**

- Bypass kernel network stack
- Direct access to NIC memory
- User-space networking

### 3. Custom Data Structures

**Order Book: Red-Black Tree**

- O(log N) insert/delete
- O(1) best bid/ask (cached)
- Memory-efficient (no heap allocations)

**Ring Buffer: Lock-Free Queue**

- Single producer (gateway)
- Single consumer (matching engine)
- No locks, no CAS (compare-and-swap)

### 4. Fixed-Point Arithmetic

**Why?**

- Floating-point: Rounding errors (financial incorrectness)
- Fixed-point: Exact calculations (cents as integers)

**Example:**

```cpp
u64 price_cents = 10050;  // $100.50 = 10050 cents
u32 quantity = 1000;
u64 total_cents = price_cents * quantity;  // Exact!
```

---

## Bottlenecks & Solutions

### Bottleneck 1: Single-Core CPU Limit

**Problem:** Single-threaded engine maxes out at 1-2M orders/sec (one CPU core).

**Solution: Shard by Symbol**

```
Symbol: AAPL → Matching Engine 1 (CPU core 0)
Symbol: GOOG → Matching Engine 2 (CPU core 1)
Symbol: MSFT → Matching Engine 3 (CPU core 2)
...

Result: 10K symbols × 1M orders/sec each = 10B orders/sec total capacity
```

### Bottleneck 2: Audit Log Disk I/O

**Problem:** 1M orders/sec × 272 bytes = 272 MB/sec disk writes.

**Solution: Batch Writes**

```
Instead of: Write every order individually (1M syscalls/sec)
Use: Batch 1000 orders, write once (1K syscalls/sec)

Batch size: 1000 orders × 272 bytes = 272 KB
Write latency: 272 KB / 3 GB/sec = 90μs
Syscall overhead reduction: 1000× faster
```

**Trade-off:** Increased crash window (lose last 1-10ms of orders).

**Mitigation:** Battery-backed write cache (persist to NVMe capacitor).

### Bottleneck 3: Market Data Broadcast

**Problem:** 100K updates/sec × 1M clients = 100B messages/sec.

**Solution: Hierarchical Fanout**

```
Matching Engine → Kafka (100K msg/sec)
                     ↓
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
  WS Server 1   WS Server 2   WS Server N
  (333K clients) (333K clients) (333K clients)

Each WS server: 100K msg/sec ÷ 333K clients = 0.3 msg/sec/client (manageable)
```

**Further Optimization: Conflation**

```
Updates within 10ms window:
  - $100.50 → $100.51 → $100.52 → $100.50

Conflated (send only last):
  - $100.50

Result: 10× reduction in messages (10 updates/100ms → 1 update/100ms)
```

---

## Common Anti-Patterns to Avoid

### 1. Using Floating-Point for Prices

❌ **Bad:**

```cpp
double price = 100.50;
double quantity = 1000.0;
double total = price * quantity;  // 100500.0000000001 (rounding error!)
```

✅ **Good:** Use fixed-point integers

```cpp
u64 price_cents = 10050;  // $100.50 = 10050 cents
u32 quantity = 1000;
u64 total_cents = price_cents * quantity;  // Exact!
```

### 2. Multi-Threaded Order Book with Locks

❌ **Bad:**

```cpp
std::mutex order_book_lock;
void process_order(Order order) {
  std::lock_guard<std::mutex> lock(order_book_lock);  // 50-100μs overhead!
  match_order(order);
}
```

✅ **Good:** Single-threaded matching engine + lock-free ring buffer

### 3. Synchronous Audit Log Writes

❌ **Bad:**

```cpp
void process_order(Order order) {
  match_order(order);
  fsync(audit_log_fd);  // 10-50μs disk write! (blocks matching engine)
}
```

✅ **Good:** Async audit log writes with batching

```cpp
void process_order(Order order) {
  match_order(order);
  async_write_to_audit_log(order);  // Non-blocking (100ns)
}
```

### 4. JSON for Market Data Feed

❌ **Bad:**

```json
{
  "symbol": "AAPL",
  "price": 100.50,
  "quantity": 100
}
```

- Size: 48 bytes
- Encoding latency: 1-5μs per message

✅ **Good:** Protocol Buffers or FlatBuffers

```protobuf
message Trade {
  uint32 symbol = 1;  // 4 bytes
  uint64 price = 2;   // 8 bytes
  uint32 quantity = 3; // 4 bytes
}
```

- Size: 16 bytes (3× smaller)
- Encoding latency: 0.1-0.5μs per message (10× faster)

---

## Monitoring & Observability

### Key Metrics

| Metric                     | Target        | Alert Threshold         |
|----------------------------|---------------|-------------------------|
| **Matching Latency (p99)** | <100μs        | >200μs                  |
| **Throughput**             | 1M orders/sec | <500K orders/sec        |
| **Order Book Depth**       | 1000 levels   | <100 levels (thin book) |
| **Audit Log Lag**          | <10ms         | >100ms                  |
| **Market Data Latency**    | <1ms          | >5ms                    |
| **Error Rate**             | <0.01%        | >0.1%                   |

### Dashboards

1. **Real-Time Operations:**
    - Matching latency (p50, p95, p99)
    - Orders/sec (line chart)
    - Order book depth per symbol (heatmap)

2. **System Health:**
    - CPU usage (should be <80% for single-threaded)
    - Memory usage (order book size)
    - Disk I/O (audit log write rate)

3. **Market Data:**
    - Subscriber count
    - Message rate (updates/sec)
    - WebSocket connection health

---

## Trade-offs Summary

| What We Gain                      | What We Sacrifice                                              |
|-----------------------------------|----------------------------------------------------------------|
| ✅ Ultra-low latency (<100μs)      | ❌ Single-threaded (can't use all CPU cores)                    |
| ✅ High throughput (1M orders/sec) | ❌ Complex architecture (kernel bypass, custom data structures) |
| ✅ Strong consistency (ACID)       | ❌ High cost (specialized hardware, NVMe SSDs)                  |
| ✅ Real-time market data           | ❌ Operational complexity (monitoring, tuning)                  |
| ✅ Financial correctness           | ❌ Limited scalability (must shard by symbol)                   |

---

## Real-World Examples

### NASDAQ

- **Scale:** 4M messages/sec peak, 10,000 symbols, 10B shares/day
- **Architecture:** Single-threaded matching engine (INET system), kernel bypass networking (Solarflare NICs)
- **Latency:** <50μs matching, <500μs end-to-end

### CME Group

- **Scale:** 5M messages/sec peak, 100,000+ contracts, 3B contracts/day
- **Architecture:** FPGA-accelerated matching (hardware acceleration), ASIC custom chip
- **Latency:** <10μs matching (hardware), <50μs end-to-end

### Coinbase

- **Scale:** 15K orders/sec, 500 trading pairs, 100M users
- **Architecture:** Multi-threaded matching engine (Golang), cloud-based (AWS, GCP)
- **Latency:** 5-50ms (acceptable for crypto)

---

## Key Takeaways

1. **Single-threaded matching engine** eliminates lock overhead (50-100μs saved)
2. **In-memory order book** provides <1μs access (vs 1-10ms for disk)
3. **Kernel bypass networking (DPDK)** reduces latency by 10-50× (0.1μs vs 10-50μs)
4. **Fixed-point arithmetic** ensures financial correctness (no rounding errors)
5. **Async audit log** prevents blocking (0.1μs vs 10-50μs)
6. **Ring buffer** enables lock-free communication (no mutexes)
7. **Shard by symbol** scales horizontally (10K symbols × 1M orders/sec each)
8. **Batch writes** reduce syscall overhead (1000× reduction)
9. **Protocol Buffers** reduce message size and encoding latency (3× smaller, 10× faster)
10. **Battery-backed write cache** guarantees durability even on power loss

---

## Recommended Stack

- **Matching Engine:** C++ (single-threaded, custom data structures)
- **Gateway:** C++ with DPDK (kernel bypass networking)
- **Order Book:** Red-Black Tree (in-memory, O(log N) operations)
- **Audit Log:** NVMe SSD with battery-backed write cache
- **Market Data:** Kafka (distribution) + WebSocket (client delivery)
- **Monitoring:** Prometheus + Grafana (metrics, dashboards)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, order book structure, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (order matching, execution paths,
  failure scenarios)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (matching algorithm, audit log, market data)

