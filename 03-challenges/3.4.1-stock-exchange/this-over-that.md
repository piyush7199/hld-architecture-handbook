# Stock Exchange Matching Engine - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made for the stock exchange matching engine design, including rationale, alternatives considered, and trade-offs.

---

## Table of Contents

1. [Single-Threaded vs Multi-Threaded Matching Engine](#1-single-threaded-vs-multi-threaded-matching-engine)
2. [Red-Black Tree vs Other Data Structures for Order Book](#2-red-black-tree-vs-other-data-structures-for-order-book)
3. [DPDK Kernel Bypass vs Traditional Networking Stack](#3-dpdk-kernel-bypass-vs-traditional-networking-stack)
4. [Async Audit Log vs Synchronous Writes](#4-async-audit-log-vs-synchronous-writes)
5. [Fixed-Point Integers vs Floating-Point for Prices](#5-fixed-point-integers-vs-floating-point-for-prices)
6. [Ring Buffer vs Message Queue for Inter-Process Communication](#6-ring-buffer-vs-message-queue-for-inter-process-communication)
7. [On-Premise vs Cloud Deployment](#7-on-premise-vs-cloud-deployment)

---

## 1. Single-Threaded vs Multi-Threaded Matching Engine

### The Problem

The matching engine must process 1M orders/sec with <100μs latency. Should we use a single-threaded design (sequential processing) or multi-threaded design (concurrent processing with locks)?

### Options Considered

| Design | Pros | Cons | Latency | Throughput |
|--------|------|------|---------|------------|
| **Single-Threaded** (Chosen) | ✅ No locks (no contention)<br/>✅ No cache coherency overhead<br/>✅ Predictable latency<br/>✅ Simple (no race conditions) | ❌ Single-core throughput limit (1-2M orders/sec)<br/>❌ Cannot utilize multi-core | <100μs (p99) | 1-2M orders/sec |
| **Multi-Threaded + Locks** | ✅ Higher throughput (5-10M orders/sec)<br/>✅ Utilize multi-core CPUs | ❌ Lock contention (50-100μs per lock)<br/>❌ Cache line ping-pong<br/>❌ Non-deterministic latency<br/>❌ Complex (deadlocks, races) | 1-10ms (p99) | 5-10M orders/sec |
| **Multi-Threaded + Lock-Free** | ✅ No locks (lock-free data structures)<br/>✅ Utilize multi-core | ❌ Complex implementation<br/>❌ CAS retry loops<br/>❌ Still has cache coherency overhead | 500μs-1ms (p99) | 3-5M orders/sec |

### Decision Made

**Single-Threaded Matching Engine** (LMAX Disruptor Pattern)

### Rationale

1. **Lock Overhead Too High:**
   - Mutex acquire/release: 50-100 CPU cycles (17-33ns at 3 GHz)
   - At 1M orders/sec, this adds 50-100μs per order (50-100% of latency budget!)
   - Lock contention under high load pushes latency to milliseconds

2. **Cache Coherency Problem:**
   ```
   Thread 1 (Core 0): Modify order book → Invalidate cache line
   Thread 2 (Core 1): Access order book → Cache miss → Fetch from RAM (100-300ns)
   
   Result: 10-30% performance degradation from cache thrashing
   ```

3. **Predictability > Throughput:**
   - Financial systems value **predictable latency** over **raw throughput**
   - Single-threaded: Latency variance <10μs (p99 - p50)
   - Multi-threaded: Latency variance 1-10ms (unpredictable)

4. **Horizontal Scaling via Sharding:**
   - Deploy 10K matching engines (one per symbol)
   - Total capacity: 10K × 1M = **10B orders/sec**
   - No need for multi-threaded complexity

### Implementation Details

**LMAX Disruptor Pattern:**

```
Gateway (Producer) → Ring Buffer (Lock-Free) → Matching Engine (Consumer)

Key:
- Single producer, single consumer (no CAS retry loops)
- Pre-allocated ring buffer (no memory allocation overhead)
- Cache line alignment (no false sharing)
- Latency: <100ns
```

*See [pseudocode.md::ring_buffer_enqueue()](pseudocode.md) for implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Ultra-low latency** (<100μs consistently) | ❌ **Single-core throughput limit** (1-2M orders/sec) |
| ✅ **Predictable performance** (no lock contention) | ⚠️ **Must shard by symbol** (deploy multiple engines) |
| ✅ **Simple design** (no race conditions, no deadlocks) | ⚠️ **Higher operational complexity** (manage 10K engines) |

### When to Reconsider

**Use Multi-Threaded If:**

1. **Latency requirement >1ms:** Locks become acceptable (not sub-100μs critical)
2. **Cannot shard workload:** Cross-symbol orders dominate (rare in practice)
3. **Limited deployment resources:** Cannot deploy 10K+ engine instances

**Real-World Examples:**

- **NASDAQ, NYSE, CME:** Single-threaded design (proven at scale)
- **Coinbase (Crypto):** Multi-threaded Golang (5-50ms latency acceptable)
- **Traditional Banks:** Multi-threaded (compliance > latency)

---

## 2. Red-Black Tree vs Other Data Structures for Order Book

### The Problem

The order book must support:
- **Insert order:** O(log N) acceptable
- **Best bid/ask:** O(1) required (accessed every order)
- **Match and remove:** O(log N) acceptable
- **Memory efficiency:** Fit 10K orders in L2 cache (1-2 MB)

### Options Considered

| Data Structure | Insert | Best Price | Delete | Memory | Cache Locality |
|----------------|--------|------------|--------|--------|----------------|
| **Red-Black Tree** (Chosen) | O(log N) | O(1) cached | O(log N) | ✅ 40 bytes/node | ✅ Good |
| **AVL Tree** | O(log N) | O(1) cached | O(log N) | ⚠️ 48 bytes/node | ⚠️ Poor (more rotations) |
| **B-Tree** | O(log N) | O(log N) | O(log N) | ❌ 100+ bytes/node | ✅ Excellent (disk-optimized) |
| **Skip List** | O(log N) avg | O(1) | O(log N) avg | ⚠️ 50+ bytes/node | ❌ Poor (random pointers) |
| **Sorted Array** | O(N) | O(1) | O(N) | ✅ 8 bytes/entry | ✅ Perfect |
| **Hash Map** | O(1) | O(N) scan | O(1) | ✅ 16 bytes/entry | ❌ Poor (random access) |

### Decision Made

**Red-Black Tree + Linked List Hybrid**

- **Red-Black Tree:** Index price levels (sorted)
- **Linked List:** FIFO queue at each price level (time priority)
- **Hash Map:** O(1) lookup for order cancellation

### Rationale

1. **Why Red-Black Tree over AVL Tree?**
   - **Fewer rotations:** RB tree rotations ~2× per insert, AVL ~3-4× (due to stricter balancing)
   - **Better write performance:** Matching engine is write-heavy (insert/delete orders)
   - **Cache-friendly:** Fewer tree modifications = fewer cache line invalidations

2. **Why NOT B-Tree?**
   - **Optimized for disk I/O:** B-tree minimizes disk seeks (we're in-memory!)
   - **Larger nodes:** 100+ bytes per node (wastes memory for small key-value pairs)
   - **Overkill for in-memory:** Red-Black tree has better L1/L2 cache utilization

3. **Why NOT Skip List?**
   - **Non-deterministic:** Randomized levels → unpredictable latency
   - **Poor cache locality:** Random pointers across memory (cache thrashing)
   - **Harder to implement:** Balancing logic complex

4. **Why NOT Sorted Array?**
   - **O(N) insert/delete:** 10K orders × 1M orders/sec = impossible
   - **Only works for read-heavy:** Stock exchange is write-heavy (orders constantly change)

5. **Why Add Hash Map?**
   - **Fast cancellation:** O(1) lookup by order_id (common operation)
   - **Memory cost:** 16 bytes/order (acceptable for 10K orders = 160 KB)

### Implementation Details

```
OrderBook:
  bids: RBTree<Price, PriceLevel>  // Sorted DESC (highest first)
  asks: RBTree<Price, PriceLevel>  // Sorted ASC (lowest first)
  orders: HashMap<OrderID, *Order> // Fast cancellation
  
  best_bid: Price  // Cached at tree root (O(1) access)
  best_ask: Price

PriceLevel:
  price: Price
  orders: LinkedList<*Order>  // FIFO time priority
  total_quantity: u32
```

**Memory Layout:**

- Order: 64 bytes (cache-line aligned)
- Tree Node: 40 bytes (parent, left, right, color, data)
- Hash Map Entry: 16 bytes (key, value)
- **Total per order:** 64 + 40 + 16 = 120 bytes
- **10K orders:** 1.2 MB (fits in L2 cache: 1-2 MB)

*See [pseudocode.md::insert_order()](pseudocode.md) for implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **O(log N) insert/delete** (sub-microsecond for 10K orders) | ❌ **Memory overhead** (40% overhead from tree pointers) |
| ✅ **O(1) best bid/ask** (cached, no tree traversal) | ⚠️ **Rebalancing cost** (rotations add 10-20% overhead) |
| ✅ **Cache-friendly** (10K orders fit in L2 cache) | - |

### When to Reconsider

**Use B-Tree If:**

1. **Order book too large for RAM:** 100M+ orders (need disk-backed structure)
2. **Read-heavy workload:** Mostly queries, few inserts (B-tree optimizes reads)

**Use Skip List If:**

1. **Concurrent updates:** Multiple threads modifying order book (lock-free skip list)
2. **Simple implementation:** Don't want to implement Red-Black tree rotations

**Real-World Examples:**

- **NASDAQ, CME:** Custom Red-Black Tree variants
- **IEX (Investors Exchange):** Red-Black Tree + Cache optimization
- **Some crypto exchanges:** Skip lists (lower latency requirements)

---

## 3. DPDK Kernel Bypass vs Traditional Networking Stack

### The Problem

Network latency is a significant bottleneck:
- **Traditional stack:** 10-50μs (kernel processing, interrupts, context switches)
- **Target:** <5μs gateway latency (to fit in 100μs total budget)

### Options Considered

| Networking Stack | Latency | CPU Overhead | Complexity | Portability |
|------------------|---------|--------------|------------|-------------|
| **DPDK Kernel Bypass** (Chosen) | 1-5μs | 100% (dedicated cores) | High | Low (NIC-specific) |
| **Traditional Linux Stack** | 10-50μs | 5-20% | Low | High |
| **XDP (eBPF)** | 5-10μs | 20-40% | Medium | Medium |
| **User-Space TCP Stack (mTCP)** | 5-15μs | 50-80% | High | Medium |

### Decision Made

**DPDK (Data Plane Development Kit) with Kernel Bypass**

### Rationale

1. **10× Latency Reduction:**
   ```
   Traditional Stack:
   NIC → Interrupt → Kernel → Copy → Application
   Latency: 10-50μs
   
   DPDK:
   NIC → DMA → Application Memory (poll mode, no interrupts)
   Latency: 1-5μs
   ```

2. **Eliminate Interrupt Overhead:**
   - **Hardware interrupt:** 1000-10000 CPU cycles (context switch)
   - **DPDK polling:** 0 interrupts (100% CPU polling NIC)
   - **Result:** Predictable latency (no interrupt storms)

3. **Zero-Copy:**
   - **Traditional:** NIC buffer → Kernel buffer → User buffer (2 copies)
   - **DPDK:** NIC DMA → Application memory (0 copies)
   - **Result:** 50% memory bandwidth savings

4. **Huge Pages:**
   - **Traditional:** 4 KB pages (TLB thrashing with large buffers)
   - **DPDK:** 2 MB pages (99% fewer TLB misses)
   - **Result:** 10-20% performance improvement

### Implementation Details

**DPDK Setup:**

```bash
# 1. Reserve huge pages (2 MB pages)
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# 2. Bind NIC to DPDK driver (unbind from kernel)
dpdk-devbind.py --bind=vfio-pci 0000:02:00.0

# 3. Pin DPDK threads to CPU cores
taskset -c 0-3 ./gateway_dpdk
```

**Poll Mode Driver (PMD):**

```cpp
while (true) {
  // Poll NIC descriptor ring (no interrupts!)
  num_packets = rte_eth_rx_burst(port_id, queue_id, packets, BURST_SIZE);
  
  for (i = 0; i < num_packets; i++) {
    process_packet(packets[i]);  // FIX protocol parsing
  }
}
```

**Latency Breakdown:**

| Step | Traditional | DPDK | Improvement |
|------|-------------|------|-------------|
| **NIC → Memory** | 1μs (interrupt) | 0.5μs (DMA) | 2× |
| **Kernel Processing** | 10μs | 0μs (userspace) | ∞ |
| **Copy to User** | 2μs | 0μs (zero-copy) | ∞ |
| **Total** | **13μs** | **0.5μs** | **26×** |

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **10× latency reduction** (50μs → 5μs) | ❌ **100% CPU overhead** (polling = 100% utilization) |
| ✅ **Zero-copy** (50% memory bandwidth savings) | ❌ **NIC-specific** (Intel X710, Mellanox only) |
| ✅ **Predictable latency** (no interrupts) | ❌ **Complex setup** (userspace TCP/IP stack required) |
| ✅ **Huge pages** (10-20% performance gain) | ⚠️ **Cost** (dedicated cores = 4-8 cores for gateway) |

### When to Reconsider

**Use Traditional Stack If:**

1. **Latency requirement >10ms:** Kernel overhead acceptable
2. **Cost-sensitive:** Cannot dedicate CPU cores to polling (cloud, multi-tenant)
3. **Portability:** Need to run on any NIC (not just Intel/Mellanox)

**Use XDP (eBPF) If:**

1. **Middle ground:** 5-10μs latency acceptable (between DPDK and kernel)
2. **Simpler than DPDK:** Still use kernel drivers (no userspace TCP/IP)
3. **DDoS protection:** eBPF firewall at NIC (drop packets before kernel)

**Real-World Examples:**

- **NYSE, NASDAQ, CME:** DPDK + Solarflare/Mellanox NICs
- **Coinbase:** Traditional stack (5-50ms acceptable for crypto)
- **Cloudflare (DDoS):** XDP for packet filtering (not matching engine)

---

## 4. Async Audit Log vs Synchronous Writes

### The Problem

All orders and trades must be persisted for:
- **Audit compliance:** SEC regulations require 7-year retention
- **Crash recovery:** Rebuild order book state after failure
- **Durability:** No lost trades (financial correctness)

But synchronous disk writes add 10-50μs latency (50% of budget!).

### Options Considered

| Write Strategy | Durability | Latency | Throughput | Crash Window |
|----------------|------------|---------|------------|--------------|
| **Async Writes** (Chosen) | ✅ 99.99% (with battery backup) | <1μs (non-blocking) | 272 MB/sec | 1-10ms |
| **Sync Writes (fsync)** | ✅ 100% | 10-50μs (blocks engine) | 20 MB/sec | 0ms |
| **Sync + WAL** | ✅ 100% | 20-100μs | 10 MB/sec | 0ms |
| **No Durability** | ❌ 0% | 0μs | ∞ | ∞ |

### Decision Made

**Async Writes with Battery-Backed NVMe Cache**

### Rationale

1. **Non-Blocking Writes:**
   ```
   Synchronous (BAD):
   Order → Match → fsync() → Wait 10-50μs → Return
   
   Async (GOOD):
   Order → Match → Copy to ring buffer (100ns) → Return
                     ↓
                  Background thread → Batch write → fsync()
   ```

2. **Batch Amortization:**
   - **Sync:** 1M orders/sec × 1 fsync/order = 1M syscalls/sec (impossible!)
   - **Async:** Batch 1000 orders → 1 fsync/batch = 1K syscalls/sec (manageable)
   - **Result:** 1000× reduction in disk I/O overhead

3. **Throughput:**
   - **Sync:** 1M orders/sec × 50μs = 50 seconds of disk I/O per second (impossible!)
   - **Async:** 1M orders/sec × 272 bytes = 272 MB/sec (NVMe SSD: 3 GB/sec → no bottleneck)

4. **Battery-Backed Cache:**
   - **Problem:** Async writes at risk if crash before flush (1-10ms window)
   - **Solution:** NVMe SSD with capacitor backup (persists on power loss)
   - **Cost:** +$500 per drive (acceptable for financial system)

### Implementation Details

**Async Write Architecture:**

```
Matching Engine → Ring Buffer (Lock-Free) → Background Thread
                                               ↓
                                          Batch Buffer (1000 entries)
                                               ↓
                                          NVMe SSD (fsync every 1ms)
```

**Log Entry Structure:**

```cpp
struct LogEntry {
  u64 sequence_number;  // Monotonic counter
  u64 timestamp;        // Nanosecond precision
  u8 event_type;        // 0=ORDER, 1=EXECUTION, 2=CANCEL
  u8 payload[256];      // Serialized event data
  u32 checksum;         // CRC32 integrity
};

Size: 272 bytes per entry
Rate: 1M entries/sec
Throughput: 272 MB/sec
```

**Crash Recovery:**

```
1. Read audit log from disk (sequential)
   - 272 MB at 3 GB/sec = 90ms

2. Replay all events (rebuild order book)
   - 1M orders × 1μs = 1 second

Total downtime: ~1 second
```

*See [pseudocode.md::write_audit_log()](pseudocode.md) for implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Non-blocking writes** (matching engine latency unaffected) | ⚠️ **Crash window** (last 1-10ms at risk) |
| ✅ **High throughput** (272 MB/sec sustainable) | ⚠️ **Cost** ($500 battery-backed cache per drive) |
| ✅ **Batch amortization** (1000× fewer syscalls) | ❌ **Eventual durability** (not immediate like sync) |

### When to Reconsider

**Use Sync Writes If:**

1. **Zero data loss required:** Regulatory requirement for immediate persistence
2. **Low throughput:** <10K orders/sec (sync overhead acceptable)
3. **Latency budget >50μs:** Can afford disk I/O in critical path

**Use No Durability If:**

1. **Test environment:** Non-production systems
2. **Ephemeral data:** Order book can be reconstructed from external source

**Real-World Examples:**

- **NYSE, NASDAQ:** Async writes with NVRAM (non-volatile RAM, even faster than NVMe)
- **CME Group:** Hybrid (sync for trade executions, async for order acks)
- **Crypto exchanges:** Sync writes (lower throughput, stricter audit requirements)

---

## 5. Fixed-Point Integers vs Floating-Point for Prices

### The Problem

Financial systems must represent prices with **exact precision** (no rounding errors). Should we use floating-point (float64) or fixed-point integers?

### Options Considered

| Representation | Precision | Performance | Complexity | Use Case |
|----------------|-----------|-------------|------------|----------|
| **Fixed-Point Integers** (Chosen) | ✅ Exact (no rounding) | ✅ Fast (integer ops) | ✅ Simple | Stock prices |
| **Floating-Point (float64)** | ❌ Rounding errors | ⚠️ Slower (FPU ops) | ✅ Simple | Scientific computing |
| **Decimal (BigDecimal)** | ✅ Exact | ❌ Slow (software emulation) | ❌ Complex | Banking (rare in HFT) |

### Decision Made

**Fixed-Point Integers (Cents Representation)**

### Rationale

1. **Rounding Error Problem:**
   ```cpp
   // Floating-Point (BAD):
   double price = 100.50;
   double quantity = 1000.0;
   double total = price * quantity;
   // Result: 100500.0000000001 (ROUNDING ERROR!)
   
   // Fixed-Point (GOOD):
   u64 price_cents = 10050;  // $100.50 = 10050 cents
   u32 quantity = 1000;
   u64 total_cents = price_cents * quantity;
   // Result: 10050000 cents = $100,500.00 (EXACT!)
   ```

2. **Performance:**
   - **Integer multiply:** 1-3 CPU cycles
   - **Float multiply:** 3-5 CPU cycles (FPU pipeline)
   - **Result:** 2-3× faster for financial calculations

3. **Regulatory Compliance:**
   - SEC requires **exact** calculations (no rounding)
   - Auditors check for floating-point errors in trading systems
   - Fixed-point is **provably correct** (no rounding artifacts)

4. **Simplicity:**
   - **Fixed-point:** Simple integer operations (add, subtract, multiply, divide)
   - **Decimal:** Complex software emulation (slow)

### Implementation Details

**Price Representation:**

```cpp
// Stock price: $100.50
u64 price = 10050;  // Store as cents (100.50 × 100)

// Convert to dollars for display:
double price_dollars = (double)price / 100.0;

// Arithmetic:
u64 total = price * quantity;  // Exact integer multiplication
```

**Edge Case: Division**

```cpp
// Division requires careful rounding:
u64 avg_price = total_cents / quantity;
// Round to nearest cent (banker's rounding):
u64 remainder = total_cents % quantity;
if (remainder >= quantity / 2) {
  avg_price += 1;  // Round up
}
```

**Price Increments:**

| Asset Class | Tick Size (Minimum Increment) | Representation |
|-------------|-------------------------------|----------------|
| Stocks (US) | $0.01 (1 cent) | 1 unit |
| Futures | $0.25 (quarter) | 25 units |
| Options | $0.01 (penny) | 1 unit |
| Forex | $0.0001 (pip) | 0.01 units (store as 10000ths) |

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Exact precision** (no rounding errors) | ❌ **Manual scaling** (must convert cents ↔ dollars) |
| ✅ **Faster** (2-3× faster than float) | ⚠️ **Overflow risk** (must use u64, not u32) |
| ✅ **Regulatory compliance** (provably correct) | ❌ **Division complexity** (must handle rounding) |

### When to Reconsider

**Use Floating-Point If:**

1. **Scientific computing:** Physics simulations (rounding acceptable)
2. **Non-financial:** Gaming, graphics (precision not critical)
3. **Legacy systems:** Existing codebase uses float (migration cost)

**Use Decimal (BigDecimal) If:**

1. **Banking:** Exact decimal arithmetic required (e.g., interest calculations)
2. **Accounting:** Ledger systems (not latency-critical)
3. **Crypto:** Variable precision (Bitcoin uses satoshis, similar to fixed-point)

**Real-World Examples:**

- **All stock exchanges:** Fixed-point integers (NYSE, NASDAQ, CME)
- **Banks:** BigDecimal (Java) or Decimal (Python) for accounting
- **Crypto exchanges:** Integer satoshis (Bitcoin), wei (Ethereum)

---

## 6. Ring Buffer vs Message Queue for Inter-Process Communication

### The Problem

Gateway and matching engine run in separate processes (for fault isolation). How should they communicate?

### Options Considered

| IPC Mechanism | Latency | Throughput | Complexity | Ordering |
|---------------|---------|------------|------------|----------|
| **Ring Buffer (Shared Memory)** (Chosen) | <100ns | 10M msg/sec | Medium | FIFO |
| **Unix Domain Socket** | 1-5μs | 1M msg/sec | Low | FIFO |
| **TCP Loopback** | 10-50μs | 500K msg/sec | Low | FIFO (with TCP overhead) |
| **Kafka / RabbitMQ** | 1-10ms | 100K msg/sec | Low | FIFO (with persistence overhead) |
| **Shared Memory (Custom)** | <100ns | 10M msg/sec | High | Manual |

### Decision Made

**Lock-Free Ring Buffer in Shared Memory (LMAX Disruptor Pattern)**

### Rationale

1. **100× Faster than Unix Socket:**
   - **Ring buffer:** <100ns (just memory copy + atomic CAS)
   - **Unix socket:** 1-5μs (kernel syscall overhead)
   - **Result:** 100× latency reduction

2. **Lock-Free (Single Producer, Single Consumer):**
   - **No CAS retry loops:** Write index and read index independent
   - **No contention:** Only one writer, one reader
   - **Result:** Predictable latency (no lock contention spikes)

3. **Cache-Line Aligned:**
   ```
   Write Index: Cache Line 0 (64 bytes)
   Read Index:  Cache Line 1 (64 bytes)
   Data Slots:  Cache Lines 2-N
   
   Result: No false sharing (cache line ping-pong avoided)
   ```

4. **Pre-Allocated:**
   - **Allocate 1M slots at startup** (no dynamic allocation)
   - **No malloc/free overhead** (5-10ns vs 50-500ns)

### Implementation Details

**Ring Buffer Structure:**

```cpp
struct RingBuffer {
  Order slots[1000000];  // Pre-allocated (64 MB)
  
  alignas(64) atomic<u64> write_index;  // Cache line 0
  alignas(64) atomic<u64> read_index;   // Cache line 1
  
  u64 size = 1000000;  // Power of 2 (for fast modulo)
};

// Enqueue (producer):
u64 write_idx = write_index.fetch_add(1, memory_order_relaxed);
slots[write_idx & (size - 1)] = order;  // Bitwise AND (fast modulo)

// Dequeue (consumer):
u64 read_idx = read_index.load(memory_order_relaxed);
if (read_idx == write_index.load(memory_order_acquire)) {
  return; // Empty
}
Order order = slots[read_idx & (size - 1)];
read_index.store(read_idx + 1, memory_order_release);
```

**Memory Layout:**

```
Address 0x1000:  Write Index (64 bytes)
Address 0x1040:  Read Index (64 bytes)
Address 0x1080:  Slot 0 (64 bytes)
Address 0x10C0:  Slot 1 (64 bytes)
...
```

*See [pseudocode.md::ring_buffer_enqueue()](pseudocode.md) for implementation.*

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Ultra-low latency** (<100ns) | ❌ **Fixed size** (must pre-allocate, cannot grow) |
| ✅ **High throughput** (10M msg/sec) | ❌ **Backpressure handling** (if full, producer must block or drop) |
| ✅ **Lock-free** (no contention) | ⚠️ **Shared memory setup** (more complex than sockets) |

### When to Reconsider

**Use Unix Domain Socket If:**

1. **Latency requirement >1μs:** Socket overhead acceptable
2. **Dynamic size:** Cannot predict buffer size at startup
3. **Portability:** Shared memory not available (Docker, Kubernetes restrictions)

**Use Kafka / RabbitMQ If:**

1. **Persistence required:** Need message replay (audit log)
2. **Multi-consumer:** Multiple matching engines consuming same feed
3. **Latency >1ms:** Not latency-critical (background processing)

**Real-World Examples:**

- **LMAX (London Multi-Asset Exchange):** Invented Disruptor pattern (open-source)
- **NYSE, NASDAQ:** Custom ring buffer implementations
- **Crypto exchanges:** Mix of ring buffers (hot path) and Kafka (warm path)

---

## 7. On-Premise vs Cloud Deployment

### The Problem

Should the matching engine run on dedicated on-premise hardware or in the cloud (AWS, GCP, Azure)?

### Options Considered

| Deployment | Latency | Cost (5 Years) | Control | Scalability |
|------------|---------|---------------|---------|-------------|
| **On-Premise** (Chosen) | <100μs | $1.7M capex + $11M opex = **$12.7M** | ✅ Full hardware control | ⚠️ Manual scaling |
| **Cloud (Bare Metal)** | 500μs-1ms | $50K/month × 60 months = **$3M** | ⚠️ Limited control | ✅ Easy scaling |
| **Cloud (VMs)** | 1-10ms | $30K/month × 60 months = **$1.8M** | ❌ No hardware control | ✅ Auto-scaling |
| **Hybrid** | Mixed | $6M (on-prem) + $1M (cloud) = **$7M** | ⚠️ Partial control | ✅ Flexible |

### Decision Made

**On-Premise Deployment with Dedicated Hardware**

### Rationale

1. **Latency Requirement:**
   - **Target:** <100μs (p99)
   - **On-premise:** 50-100μs (dedicated hardware, kernel bypass)
   - **Cloud bare metal:** 500μs-1ms (multi-tenant, no kernel bypass)
   - **Cloud VMs:** 1-10ms (virtualization overhead, noisy neighbors)
   - **Result:** Cloud cannot meet latency target

2. **Kernel Bypass (DPDK):**
   - **On-premise:** Full DPDK support (Intel X710, Mellanox NICs)
   - **Cloud:** Limited DPDK support (AWS Nitro, GCP gVNIC require custom setup)
   - **Result:** On-premise has 10× better network latency

3. **Cost (5-Year TCO):**
   - **On-premise:** $1.7M capex + $2.2M/year × 5 = **$12.7M**
   - **Cloud bare metal:** $50K/month × 60 months = **$3M** (cheaper!)
   - **But:** On-premise has **predictable costs** (no egress fees, no instance spikes)

4. **Revenue vs Cost:**
   - **Revenue:** $3.15B/year (100K trades/sec × $0.001 fee × 86400 sec/day)
   - **Cost:** $2.2M/year (opex)
   - **Profit:** $3.148B/year (**99.93% margin**)
   - **Result:** $12.7M capex is negligible (0.4% of first-year revenue)

5. **Control:**
   - **NUMA optimization:** Pin threads to CPU cores, allocate local RAM
   - **NIC tuning:** Direct access to NIC registers (DPDK)
   - **No noisy neighbors:** Dedicated hardware (no multi-tenancy)

### Implementation Details

**Hardware Configuration:**

| Component | On-Premise | Cloud (c6i.metal) |
|-----------|------------|-------------------|
| **CPU** | Intel Xeon Gold 6348 (28 cores, 3.5 GHz, pinned to matching engine) | Intel Xeon Platinum 8375C (shared, virtualized) |
| **RAM** | 256 GB DDR4-3200 ECC (local NUMA node) | 128 GB DDR4 (remote NUMA) |
| **NIC** | Mellanox ConnectX-6 (100 Gbps, DPDK) | AWS Nitro (100 Gbps, limited DPDK) |
| **Storage** | NVMe SSD (battery-backed write cache) | EBS gp3 (network-attached, 10× slower) |
| **Latency** | <100μs | 500μs-1ms |

**Colocation:**

- **Data Center:** Equinix NY4 (low-latency financial hub)
- **Proximity to Markets:** <1ms to NYSE, NASDAQ
- **HFT Co-location:** Sell rack space to HFT firms (additional revenue: $500K/year)

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Ultra-low latency** (<100μs) | ❌ **High capex** ($1.7M upfront) |
| ✅ **Full hardware control** (DPDK, NUMA, NIC tuning) | ❌ **Manual scaling** (cannot auto-scale like cloud) |
| ✅ **Predictable costs** (no egress fees) | ⚠️ **Operational burden** (manage 10K servers) |
| ✅ **HFT co-location revenue** ($500K/year) | ❌ **Long procurement cycle** (6-12 months to add capacity) |

### When to Reconsider

**Use Cloud If:**

1. **Latency requirement >1ms:** Cloud latency acceptable (e.g., crypto exchanges: 5-50ms)
2. **Low upfront capital:** Startup with limited funding ($1.7M capex too high)
3. **Variable workload:** Spiky traffic (auto-scaling valuable)
4. **Non-HFT:** Retail-only platform (no institutional HFT clients)

**Use Hybrid If:**

1. **Hot/cold split:** On-prem for matching engine (hot path), cloud for analytics (cold path)
2. **Disaster recovery:** On-prem primary, cloud backup (failover in minutes)
3. **Geographic expansion:** On-prem in US, cloud in Asia/Europe (lower latency for local clients)

**Real-World Examples:**

- **NYSE, NASDAQ, CME:** 100% on-premise (latency-critical)
- **Coinbase, Kraken (Crypto):** 100% cloud (AWS, GCP) - latency less critical
- **IEX (Investors Exchange):** On-premise with cloud analytics (hybrid)
- **Robinhood:** Cloud (retail focus, no HFT)

---

## Summary Comparison

| Decision | Chosen | Alternative | Key Trade-off |
|----------|--------|-------------|---------------|
| **1. Threading Model** | Single-Threaded | Multi-Threaded + Locks | Latency (<100μs) vs Throughput (>5M orders/sec) |
| **2. Order Book** | Red-Black Tree | AVL, B-Tree, Skip List | O(log N) insert/delete, O(1) best price |
| **3. Networking** | DPDK Kernel Bypass | Traditional Stack | 10× latency reduction vs 100% CPU overhead |
| **4. Audit Log** | Async Writes | Sync Writes | Non-blocking vs 1-10ms crash window |
| **5. Price Format** | Fixed-Point Integers | Floating-Point | Exact precision vs Manual scaling |
| **6. IPC** | Ring Buffer | Unix Socket, Kafka | <100ns latency vs Fixed size |
| **7. Deployment** | On-Premise | Cloud | <100μs latency vs $1.7M capex |

**Overall Philosophy:**

**Latency First, Cost Second** - Ultra-low latency is the primary goal. Infrastructure costs ($12.7M over 5 years) are negligible compared to revenue ($3.15B/year = 99.93% margin).

