# 3.4.1 Design a Stock Exchange Matching Engine

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, order book structure, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Order matching flows, execution paths, failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## 1. Problem Statement

Design an ultra-low-latency stock exchange matching engine that:

- **Matches orders in <100μs** (microseconds)
- **Handles 1M orders/sec** peak throughput
- **Guarantees ACID** transactions for financial correctness
- **Publishes real-time market data** to millions of subscribers

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

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

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

### Non-Functional Requirements (NFRs)

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

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

### Component Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLIENT LAYER (Traders)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ HFT Firm │  │  Retail  │  │ Algo Bot │  │ Market   │        │
│  │          │  │  Trader  │  │          │  │ Maker    │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
│       │             │             │             │                │
│       └─────────────┴─────────────┴─────────────┘                │
│                         │                                         │
└─────────────────────────┼─────────────────────────────────────────┘
                          │
                          ▼ FIX Protocol / gRPC
┌─────────────────────────────────────────────────────────────────┐
│                     LOW-LATENCY GATEWAY                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  - TCP/gRPC endpoint (kernel bypass: DPDK)                 │ │
│  │  - Fast AuthN/AuthZ (JWT cached in memory)                 │ │
│  │  - Order validation (basic checks)                         │ │
│  │  - Route to matching engine via shared memory              │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────┼─────────────────────────────────────────┘
                          │
                          ▼ Shared Memory Queue (Ring Buffer)
┌─────────────────────────────────────────────────────────────────┐
│              MATCHING ENGINE (Single-Threaded Core)              │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  ORDER BOOK (In-Memory)                                     │ │
│  │                                                             │ │
│  │  BID SIDE (Buy Orders)   │   ASK SIDE (Sell Orders)        │ │
│  │  ┌──────────────────┐    │    ┌──────────────────┐        │ │
│  │  │ $100.50 → [O1]   │    │    │ $100.52 → [O5]   │        │ │
│  │  │ $100.49 → [O2,O3]│    │    │ $100.53 → [O6,O7]│        │ │
│  │  │ $100.48 → [O4]   │    │    │ $100.54 → [O8]   │        │ │
│  │  └──────────────────┘    │    └──────────────────┘        │ │
│  │       (Sorted DESC)       │        (Sorted ASC)            │ │
│  │                                                             │ │
│  │  Price-Time Priority Queue (Red-Black Tree)                │ │
│  │  - O(log N) insert/delete                                  │ │
│  │  - O(1) best bid/ask                                       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Processing Loop:                                                │
│  1. Dequeue order from ring buffer                              │
│  2. Match against opposite side                                 │
│  3. Update order book (add/remove)                              │
│  4. Generate execution events                                   │
│  5. Write to audit log (async)                                  │
│                                                                  │
│  Single-Threaded: No locks, no context switches                 │
│  Latency: <100μs per order                                      │
└─────────────────────────┼─────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  AUDIT LOG   │  │ MARKET DATA  │  │   LEDGER     │
│  (WAL)       │  │  PUBLISHER   │  │   SERVICE    │
├──────────────┤  ├──────────────┤  ├──────────────┤
│ Append-only  │  │ WebSocket    │  │ PostgreSQL   │
│ NVMe SSD     │  │ broadcast    │  │ (ACID)       │
│ 128 MB/sec   │  │ 100K msg/sec │  │ Async update │
│              │  │              │  │ User balance │
│ Durability:  │  │ Subscribers: │  │              │
│ All orders   │  │ 1M+ clients  │  │ Final truth  │
│ + trades     │  │              │  │ for balances │
└──────────────┘  └──────────────┘  └──────────────┘
```

**Flow Summary:**

1. **Client** sends order via FIX protocol (TCP with kernel bypass)
2. **Gateway** validates, writes to shared memory ring buffer
3. **Matching Engine** dequeues order, matches against order book, generates execution
4. **Audit Log** persists order + execution (durability)
5. **Market Data Publisher** broadcasts to subscribers (WebSocket)
6. **Ledger Service** updates user balances (async, eventual consistency)

---

## 4. Data Model and Order Book Structure

### 4.1 Order Structure

```
Order {
  order_id: u64,           // Unique ID (timestamp + sequence)
  user_id: u32,            // Owner
  symbol: u32,             // Stock symbol (encoded as int for speed)
  side: u8,                // 0 = BUY, 1 = SELL
  order_type: u8,          // 0 = LIMIT, 1 = MARKET, 2 = STOP
  price: u64,              // Price in cents (avoid float)
  quantity: u32,           // Number of shares
  filled_quantity: u32,    // Shares already filled
  timestamp: u64,          // Nanosecond timestamp
  time_in_force: u8,       // 0 = GTC, 1 = IOC, 2 = FOK
  status: u8               // 0 = PENDING, 1 = PARTIAL, 2 = FILLED, 3 = CANCELLED
}

Size: 64 bytes (cache-line aligned for CPU efficiency)
```

**Why u64 for price instead of float64?**

- Avoid floating-point rounding errors (critical for finance)
- Integer operations faster than floating-point (2-3× speedup)
- Price = $100.50 stored as 10050 cents

### 4.2 Order Book Structure

**Data Structure Choice: Red-Black Tree + Linked List**

```
OrderBook {
  symbol: u32,
  
  // Bid side (buy orders): sorted descending by price
  bids: RBTree<Price, PriceLevel>,
  best_bid: Price,
  
  // Ask side (sell orders): sorted ascending by price
  asks: RBTree<Price, PriceLevel>,
  best_ask: Price,
  
  // Order lookup (for cancellation)
  orders: HashMap<OrderID, *Order>,
  
  // Statistics
  total_bid_volume: u64,
  total_ask_volume: u64,
  last_trade_price: Price
}

PriceLevel {
  price: Price,
  orders: LinkedList<*Order>,  // FIFO queue at this price
  total_quantity: u32
}
```

**Why Red-Black Tree?**

- O(log N) insert/delete
- O(1) best bid/ask (tree root or leftmost/rightmost)
- Self-balancing (no worst-case O(N) like BST)
- Better cache locality than AVL tree

**Why Linked List at each price level?**

- Time priority: First order at price level executes first (FIFO)
- O(1) append to end
- O(1) remove from front

*See [pseudocode.md::insert_order()](pseudocode.md) for implementation.*

---

## 5. Matching Algorithm (Price-Time Priority)

### 5.1 Matching Rules

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

### 5.2 Matching Flow

```
1. Incoming Order: SELL 500 shares @ $100.48 (aggressive limit)
2. Check best bid: $100.50 (higher than ask price → can match!)
3. Execute against O1 (100 shares @ $100.50)
4. Remaining: 400 shares
5. Check next best bid: $100.49 (O2 has 200 shares)
6. Execute against O2 (200 shares @ $100.49)
7. Remaining: 200 shares
8. Check next best bid: $100.49 (O3 has 300 shares)
9. Execute against O3 (200 shares @ $100.49)
10. Fully filled! (O1 removed, O2 removed, O3 has 100 shares left)
```

**Latency Breakdown:**

- Step 1-2: Tree traversal: 10 CPU cycles
- Step 3-10: Matching loop: 50 CPU cycles per match (×3 = 150 cycles)
- Total: 160 CPU cycles = **53 nanoseconds** at 3 GHz

*See [pseudocode.md::match_order()](pseudocode.md) for implementation.*

---

## 6. Durability and Fault Tolerance (Audit Log)

### 6.1 Write-Ahead Log (WAL)

**Problem:** In-memory order book volatile (lost on crash). Need durability.

**Solution:** Append-only audit log persists all events before processing.

**Audit Log Structure:**

```
LogEntry {
  sequence_number: u64,    // Monotonic counter
  timestamp: u64,          // Nanosecond precision
  event_type: u8,          // 0 = ORDER, 1 = EXECUTION, 2 = CANCEL
  payload: [u8; 256]       // Serialized event data
  checksum: u32            // CRC32 for integrity
}

Size: 272 bytes per entry
Throughput: 1M entries/sec × 272 bytes = 272 MB/sec
Storage: NVMe SSD (3 GB/sec write bandwidth → no bottleneck)
```

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

*See [pseudocode.md::write_audit_log()](pseudocode.md) for implementation.*

### 6.2 Crash Recovery

```
1. Read audit log from disk (sequential read: fast)
2. Replay all ORDER events → Rebuild order book
3. Replay all EXECUTION events → Update order states
4. Rebuild in-memory structures (order book, hash map)
5. Resume normal operation
```

**Recovery Time:**

- 1M orders in log: 272 MB
- Sequential read: 3 GB/sec → 90ms read time
- Replay processing: 1M orders × 1μs = 1 second
- **Total: ~1 second downtime**

---

## 7. Market Data Feed (Real-Time Broadcast)

### 7.1 Market Data Levels

**Level 1 (Top-of-Book):**

```json
{
  "symbol": "AAPL",
  "best_bid": 100.50,
  "best_bid_size": 100,
  "best_ask": 100.52,
  "best_ask_size": 200,
  "last_trade_price": 100.51,
  "timestamp": 1640000000000001
}
```

**Level 2 (Full Depth):**

```json
{
  "symbol": "AAPL",
  "bids": [
    {
      "price": 100.50,
      "quantity": 100
    },
    {
      "price": 100.49,
      "quantity": 500
    },
    {
      "price": 100.48,
      "quantity": 200
    }
  ],
  "asks": [
    {
      "price": 100.52,
      "quantity": 200
    },
    {
      "price": 100.53,
      "quantity": 700
    }
  ]
}
```

**Level 3 (Order-by-Order):**

```json
{
  "order_id": 12345,
  "symbol": "AAPL",
  "side": "BUY",
  "price": 100.50,
  "quantity": 100,
  "timestamp": 1640000000000001
}
```

### 7.2 Broadcast Strategy

**WebSocket Fanout:**

```
Matching Engine → Kafka Topic → Market Data Service → WebSocket Server → Clients

Kafka:
  - Topic: market-data-feed
  - Partitions: 100 (shard by symbol)
  - Throughput: 100K messages/sec

Market Data Service:
  - Subscribe to Kafka
  - Filter by subscription (clients choose symbols)
  - Broadcast via WebSocket

WebSocket Server:
  - 1M concurrent connections
  - 100K updates/sec broadcast
  - Compression: Protocol Buffers (10× smaller than JSON)
```

**Latency:**

- Matching Engine → Kafka: 100μs
- Kafka → Market Data Service: 500μs
- Market Data Service → Client: 500μs
- **Total: 1.1ms** (acceptable for most traders)

**HFT Requirement:**

- High-Frequency Traders need <10μs
- Solution: Direct feed from matching engine (bypass Kafka)
- Co-location: HFT servers in same datacenter as exchange

---

## 8. Single-Threaded vs Multi-Threaded Design

### 8.1 Why Single-Threaded?

**Multi-Threaded Challenges:**

```
Thread 1: Process BUY order → Lock order book → Match → Unlock
Thread 2: Process SELL order → Wait for lock → Match → Unlock

Lock overhead:
- Mutex acquire/release: 50-100 CPU cycles
- Cache line contention: 100-500 cycles
- Context switch: 1000-10000 cycles

Result: 10-50μs latency per order (too slow!)
```

**Single-Threaded Benefits:**

```
Single thread: Process orders sequentially
- No locks needed (no concurrent access)
- No cache line ping-pong (all data in L1 cache)
- No context switches
- Predictable latency: 0.1-1μs per order

Result: <100μs end-to-end (10-100× faster!)
```

**Trade-off:**

- Single-threaded limits throughput to ~1-2M orders/sec (single CPU core)
- Scaling: Deploy separate matching engine per symbol (AAPL engine, GOOG engine, etc.)

### 8.2 LMAX Disruptor Pattern

**Ring Buffer (Lock-Free Queue):**

```
Producer (Gateway) → Ring Buffer → Consumer (Matching Engine)

Ring Buffer:
  - Pre-allocated array (1M slots)
  - Single producer, single consumer (no locks!)
  - Write: Increment write index (atomic CAS)
  - Read: Increment read index (atomic CAS)
  - Latency: <100 nanoseconds

Key: CPU cache line alignment (64 bytes)
- Write index: cache line 0
- Read index: cache line 1
- Data slots: cache lines 2-N
- No false sharing!
```

*See [pseudocode.md::ring_buffer_enqueue()](pseudocode.md) for implementation.*

---

## 9. Low-Latency Optimizations

### 9.1 Kernel Bypass Networking (DPDK)

**Problem:** Linux kernel network stack adds 10-50μs latency.

**Solution:** DPDK (Data Plane Development Kit)

```
Traditional:
  Packet → NIC → Kernel → Network Stack → Application
  Latency: 10-50μs

DPDK (Kernel Bypass):
  Packet → NIC → Application (direct memory access)
  Latency: 1-5μs

How it works:
- Poll NIC directly (no interrupts)
- Zero-copy (no kernel buffer)
- Huge pages (reduce TLB misses)
```

**Trade-off:**

- Requires dedicated CPU cores (100% utilization)
- Complex programming model
- Not portable (tied to specific NICs)

### 9.2 CPU Affinity and NUMA

**CPU Pinning:**

```bash
# Pin matching engine to CPU core 0
taskset -c 0 ./matching_engine

# Disable hyperthreading (avoid sharing L1 cache)
echo 0 > /sys/devices/system/cpu/cpu1/online
```

**NUMA (Non-Uniform Memory Access):**

```
Server: 2 CPU sockets
Socket 0: CPUs 0-15, Local RAM (faster access)
Socket 1: CPUs 16-31, Remote RAM (slower access)

Optimization:
- Allocate matching engine data on local RAM
- Pin thread to CPU on same socket
- Result: 30% latency reduction
```

### 9.3 Cache Optimization

**Cache Line Alignment:**

```cpp
struct Order {
  u64 order_id;
  u32 user_id;
  // ... 56 more bytes ...
} __attribute__((aligned(64)));  // 64-byte cache line

Why?
- CPU fetches entire cache line (64 bytes)
- If struct spans 2 cache lines → 2× memory reads
- Alignment ensures single cache line fetch
```

**Prefetching:**

```cpp
// Tell CPU to prefetch next order (hide memory latency)
__builtin_prefetch(next_order, 0, 3);
process_order(current_order);
```

### 9.4 Avoid Memory Allocation

**Problem:** malloc/free adds 50-500 nanoseconds latency.

**Solution: Object Pool**

```
Pre-allocate 10M Order objects at startup
- Pool: Array of Orders
- Free list: Linked list of available slots
- Allocate: Pop from free list (O(1))
- Deallocate: Push to free list (O(1))
- Latency: 5-10 nanoseconds
```

*See [pseudocode.md::object_pool_allocate()](pseudocode.md) for implementation.*

---

## 10. Bottlenecks and Scaling Strategies

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

**Cross-Symbol Orders:**

- Rare: <1% of orders span multiple symbols
- Handle via inter-engine communication (slow path)

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

## 11. Common Anti-Patterns

### ❌ Anti-Pattern 1: Using Floating-Point for Prices

**Problem:**

```cpp
double price = 100.50;
double quantity = 1000.0;
double total = price * quantity;  // 100500.0000000001 (rounding error!)
```

**Impact:** Financial errors, audit failures, regulatory fines.

**✅ Best Practice:** Use fixed-point integers.

```cpp
u64 price_cents = 10050;  // $100.50 = 10050 cents
u32 quantity = 1000;
u64 total_cents = price_cents * quantity;  // 10050000 cents = $100,500.00 (exact!)
```

---

### ❌ Anti-Pattern 2: Multi-Threaded Order Book with Locks

**Problem:**

```cpp
std::mutex order_book_lock;

void process_order(Order order) {
  std::lock_guard<std::mutex> lock(order_book_lock);  // 50-100μs overhead!
  match_order(order);
}
```

**Impact:** 50-100μs latency per order (500× slower than target).

**✅ Best Practice:** Single-threaded matching engine + lock-free ring buffer.

---

### ❌ Anti-Pattern 3: Synchronous Audit Log Writes

**Problem:**

```cpp
void process_order(Order order) {
  match_order(order);
  fsync(audit_log_fd);  // 10-50μs disk write! (blocks matching engine)
}
```

**Impact:** Disk I/O blocks matching engine, latency spikes to milliseconds.

**✅ Best Practice:** Async audit log writes with batching.

```cpp
void process_order(Order order) {
  match_order(order);
  async_write_to_audit_log(order);  // Non-blocking (100ns)
}
```

---

### ❌ Anti-Pattern 4: JSON for Market Data Feed

**Problem:**

```json
{
  "symbol": "AAPL",
  "price": 100.50,
  "quantity": 100
}
```

- Size: 48 bytes (after minification)
- Encoding latency: 1-5μs per message
- 100K msg/sec × 5μs = 500ms total CPU time (one core!)

**✅ Best Practice:** Protocol Buffers or FlatBuffers.

```protobuf
message Trade {
  uint32 symbol = 1;  // 4 bytes (encoded as int)
  uint64 price = 2;   // 8 bytes
  uint32 quantity = 3; // 4 bytes
}
```

- Size: 16 bytes (3× smaller)
- Encoding latency: 0.1-0.5μs per message (10× faster)

---

## 12. Alternative Approaches

### Alternative 1: Multi-Threaded with Fine-Grained Locks

**Why NOT Chosen:**

| Factor          | Single-Threaded (Chosen)         | Multi-Threaded + Locks                |
|-----------------|----------------------------------|---------------------------------------|
| **Latency**     | ✅ <100μs (no locks)              | ❌ 1-10ms (lock contention)            |
| **Throughput**  | ⚠️ 1-2M orders/sec (single core) | ✅ 5-10M orders/sec (multi-core)       |
| **Complexity**  | ✅ Simple (no race conditions)    | ❌ Complex (deadlocks, races)          |
| **Determinism** | ✅ Predictable (sequential)       | ❌ Non-deterministic (race conditions) |

**When to Use Multi-Threaded:**

- Lower latency requirement (1-10ms acceptable)
- Higher throughput priority (>10M orders/sec)
- Willing to accept complexity

### Alternative 2: Distributed Matching Engine (Sharded by Price)

**Architecture:**

```
Price $100.00-$100.50 → Engine 1
Price $100.51-$101.00 → Engine 2
Price $101.01-$101.50 → Engine 3
```

**Why NOT Chosen:**

**Problem:** Cross-price matching requires coordination.

```
Order: BUY @ $101.00 (spans Engine 1 and Engine 2)
  → Requires 2PC (Two-Phase Commit)
  → Adds 1-10ms latency
```

**When to Use:**

- Extremely high throughput (>100M orders/sec)
- Price ranges rarely overlap (e.g., options trading)

### Alternative 3: Cloud-Based Matching Engine

**Why NOT Chosen:**

| Factor      | On-Premise (Chosen)           | Cloud (AWS, GCP)           |
|-------------|-------------------------------|----------------------------|
| **Latency** | ✅ <100μs (dedicated hardware) | ❌ 500μs-5ms (multi-tenant) |
| **Network** | ✅ Kernel bypass (DPDK)        | ❌ Standard networking      |
| **Cost**    | ✅ $100K capex (one-time)      | ⚠️ $50K/month (ongoing)    |
| **Control** | ✅ Full hardware control       | ❌ Limited control          |

**When to Use Cloud:**

- Non-latency-critical (crypto exchanges: 5-50ms OK)
- Lower upfront cost (no capex)
- Rapid scaling (add instances)

---

## 13. Monitoring and Observability

### 13.1 Key Metrics

| Metric                       | Target | Alert Threshold | Action                     |
|------------------------------|--------|-----------------|----------------------------|
| **Matching Latency (p50)**   | <50μs  | >100μs          | Investigate hot order book |
| **Matching Latency (p99)**   | <100μs | >500μs          | Check CPU load, memory     |
| **Matching Latency (p99.9)** | <1ms   | >5ms            | Restart matching engine    |
| **Order Reject Rate**        | <0.1%  | >1%             | Check validation logic     |
| **Execution Rate**           | 10%    | <5% or >20%     | Market anomaly detection   |
| **Audit Log Lag**            | <10ms  | >100ms          | Disk I/O bottleneck        |
| **Market Data Lag**          | <1ms   | >10ms           | Network congestion         |

### 13.2 Distributed Tracing

**Per-Order Trace:**

```
Order ID: 12345
Timestamp 0μs:    Order received at gateway
Timestamp 5μs:    Written to ring buffer
Timestamp 10μs:   Dequeued by matching engine
Timestamp 50μs:   Matching complete (executed)
Timestamp 60μs:   Audit log written (async)
Timestamp 100μs:  Market data published

Total: 100μs end-to-end
Breakdown:
  - Gateway: 5μs
  - Ring buffer: 5μs
  - Matching: 40μs ← Bottleneck
  - Audit log: 10μs
  - Market data: 40μs
```

**Tools:**

- Custom instrumentation (RDTSC CPU cycle counter)
- Low-overhead logging (circular buffer in shared memory)
- Post-mortem analysis (replay audit log with tracing)

---

## 14. Real-World Examples

### 15.1 NASDAQ (US Stock Exchange)

**Scale:**

- 4M messages/sec peak
- 10,000 symbols
- 10B shares/day
- $20 trillion market cap

**Architecture:**

- Single-threaded matching engine (INET system)
- Kernel bypass networking (Solarflare NICs)
- Order book: Custom C++ implementation
- Latency: <50μs matching, <500μs end-to-end

**Innovations:**

- Dynamic order types (pegged orders, hidden orders)
- Halt/resume trading (circuit breakers)
- Multi-tier market data (Level 1, 2, 3)

### 15.2 CME Group (Futures Exchange)

**Scale:**

- 5M messages/sec peak
- 100,000+ contracts
- 3B contracts/day
- $1 quadrillion notional value

**Architecture:**

- CME Globex platform (C++)
- FPGA-accelerated matching (hardware acceleration)
- Order book: ASIC custom chip
- Latency: <10μs matching (hardware), <50μs end-to-end

**Unique Features:**

- Hardware matching engine (FPGA/ASIC)
- Sub-10μs latency (fastest in industry)
- 24/7 trading (futures never close)

### 15.3 Coinbase (Crypto Exchange)

**Scale:**

- 15K orders/sec (lower than traditional)
- 500 trading pairs
- 100M users

**Architecture:**

- Multi-threaded matching engine (Golang)
- Cloud-based (AWS, GCP)
- Order book: PostgreSQL (!) + Redis cache
- Latency: 5-50ms (acceptable for crypto)

**Why Different:**

- Lower latency requirements (crypto traders less sensitive)
- Cloud-first (scalability over latency)
- Retail focus (not institutional HFT)

---

## 15. Interview Discussion Points

### Q1: Why not use a database for the order book?

**Answer:**

**Database (PostgreSQL):**

- Disk-based: 1-10ms latency per query
- ACID overhead: Row locks, WAL writes
- Network overhead: TCP round-trip

**In-Memory Order Book:**

- RAM-based: <1μs latency
- No locks (single-threaded)
- No network (same process)

**When Database is Acceptable:**

- Non-latency-critical (>10ms OK)
- Need ACID across multiple tables
- Complex queries (joins, aggregations)

---

### Q2: How do you handle a flash crash?

**Answer:**

**Circuit Breakers:**

```
If price moves >10% in 5 minutes:
  1. Halt trading (reject all orders)
  2. Notify exchanges, regulators
  3. Manual review (30-60 minutes)
  4. Resume trading

Implementation:
  - Monitor last_trade_price every 1 second
  - Compare to VWAP (volume-weighted average price)
  - If delta > 10%: Trigger halt
```

**Kill Switch:**

```
Operator command: HALT_ALL_TRADING
  - Stop accepting new orders
  - Cancel all open orders
  - Preserve audit log
  - Notify clients

Resume: Manually restart matching engines
```

---

### Q3: What if matching engine crashes?

**Answer:**

**Crash Recovery:**

1. **Detect Crash:** Heartbeat monitor (1-second timeout)
2. **Failover:** Backup matching engine takes over (hot standby)
3. **Replay Audit Log:** Rebuild order book from WAL
4. **Resume Trading:** ~1 second downtime

**Hot Standby:**

- Backup engine receives same audit log (async replication)
- Maintains shadow order book (eventual consistency)
- On crash: Promote backup to primary (<1 second)

**Cold Start:**

- No backup: Replay audit log from scratch (10-60 seconds)

---

## 16. Trade-offs Summary

| What We Gain                                          | What We Sacrifice                                              |
|-------------------------------------------------------|----------------------------------------------------------------|
| ✅ **Ultra-low latency** (<100μs matching)             | ❌ **Single-core throughput limit** (1-2M orders/sec)           |
| ✅ **Simple single-threaded design** (no locks)        | ❌ **Sharding required for scale** (separate engine per symbol) |
| ✅ **Deterministic execution** (sequential processing) | ⚠️ **Custom hardware required** (kernel bypass, NVMe)          |
| ✅ **Strong consistency** (ACID via audit log)         | ❌ **Eventual consistency for ledger** (async balance updates)  |
| ✅ **Financial correctness** (no double-spending)      | ⚠️ **High infrastructure cost** ($1.7M capex, $2.2M/year opex) |

**Best For:**

- Institutional trading (HFT, market makers)
- High-volume exchanges (NYSE, NASDAQ)
- Latency-sensitive markets (equities, futures)

**NOT For:**

- Retail-only platforms (latency less critical)
- Low-volume markets (<1K orders/sec)
- Cost-sensitive startups (cloud alternatives cheaper)

---

## 17. References

### Related System Design Components

- **[2.5.3 Distributed Locking](../../02-components/2.5-algorithms/2.5.3-distributed-locking.md)** - Why single-threaded
  avoids locks
- **[2.3.1 Asynchronous Communication](../../02-components/2.3-messaging-streaming/2.3.1-asynchronous-communication.md)
  ** - Audit log async writes
- **[2.0.3 Real-Time Communication](../../02-components/2.0-communication/2.0.3-real-time-communication.md)** -
  WebSocket market data
- **[2.1.7 PostgreSQL Deep Dive](../../02-components/2.1-databases/2.1.7-postgresql-deep-dive.md)** - WAL (Write-Ahead
  Log) patterns
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)** - Market data feed distribution

### Related Design Challenges

- **[3.4.4 Recommendation System](../3.4.4-recommendation-system/)** - Low-latency serving patterns
- **[3.1.2 Distributed Cache](../3.1.2-distributed-cache/)** - In-memory data structures

### External Resources

- **LMAX Disruptor Pattern:** [Disruptor Documentation](https://lmax-exchange.github.io/disruptor/) - High-performance
  inter-thread messaging
- **Understanding Latency:
  ** [Jeff Dean's Latency Numbers](https://people.eecs.berkeley.edu/~rcs/research/interactive_latency.html) - System
  latency reference
- **Matching Engine Design:** [Stanford EE380 Talk](https://web.stanford.edu/class/ee380/Abstracts/121017-slides.pdf) -
  Exchange architecture
- **Quickfix:** [FIX Protocol Implementation](https://github.com/quickfix/quickfix) - C++ FIX library
- **Disruptor:** [Low-Latency Ring Buffer](https://github.com/LMAX-Exchange/disruptor) - Java ring buffer
- **DPDK:** [Kernel Bypass Networking](https://www.dpdk.org/) - Data Plane Development Kit

### Books

- *Trading and Exchanges* by Larry Harris - Market microstructure and exchange design
- *Flash Boys* by Michael Lewis - HFT and latency arbitrage
- *The Linux Programming Interface* by Michael Kerrisk - Low-level system optimization

