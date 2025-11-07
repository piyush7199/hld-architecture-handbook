# Distributed ID Generator - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A distributed ID generator creates globally unique, time-sortable 64-bit integer IDs at scale without requiring
coordination between nodes during generation. The Snowflake algorithm (inspired by Twitter) generates millions of IDs
per second across multiple data centers.

**Key Characteristics:**

- **Globally unique** - Every ID is unique across all data centers
- **Time-sortable** - Newer IDs are numerically greater than older IDs (efficient database indexing)
- **64-bit integers** - Standard BIGINT type, efficient storage
- **No coordination during generation** - Each node generates IDs independently
- **High performance** - < 1ms latency, 100K+ IDs/sec per node
- **High availability** - 99.999% uptime (ID generation failure stops entire system)

---

## Requirements & Scale

### Functional Requirements

| Requirement           | Description                                                          | Priority    |
|-----------------------|----------------------------------------------------------------------|-------------|
| **Global Uniqueness** | Every generated ID must be globally unique across all data centers   | Must Have   |
| **Numerical**         | IDs must be 64-bit integers (BIGINT) for efficient database indexing | Must Have   |
| **Time-Sortable**     | Newer IDs should be numerically greater than older IDs (roughly)     | Must Have   |
| **No Coordination**   | ID generation should not require inter-node communication            | Must Have   |
| **High Availability** | Service must remain available even if some nodes fail                | Must Have   |
| **K-Sortable**        | IDs generated within same time window should be roughly sortable     | Should Have |

### Scale Estimation (Social Media Platform Example)

| Metric                  | Calculation              | Result               |
|-------------------------|--------------------------|----------------------|
| **Average QPS**         | 10K writes/sec baseline  | 10K IDs/sec          |
| **Peak QPS**            | 10x burst during events  | 100K IDs/sec peak    |
| **Daily IDs**           | 10K QPS sustained        | ~864 million IDs/day |
| **IDs per Node**        | 10 nodes sharing load    | 10K IDs/sec per node |
| **IDs per Millisecond** | Per node capacity        | 10 IDs/ms per node   |
| **Sequence Bits**       | Support 4096 IDs/ms      | 12 bits sufficient   |
| **Timestamp Lifespan**  | 41 bits at 1ms precision | ~69 years            |
| **Total Nodes**         | 10 bits for node ID      | Up to 1024 nodes     |

**64-bit ID Structure:**

```
├─ 1 bit (sign): Always 0 (positive number)
├─ 41 bits (timestamp): Milliseconds since epoch = 2^41 ms = 69 years
├─ 10 bits (worker ID): Up to 2^10 = 1024 worker nodes
└─ 12 bits (sequence): Up to 2^12 = 4,096 IDs per millisecond per node

Max IDs per second per node = 4,096 × 1,000 = 4,096,000 IDs/sec
Max IDs per second cluster (1024 nodes) = 4.2 billion IDs/sec
Our need: 100K IDs/sec (well within capacity)
```

---

## Key Components

| Component                | Responsibility                                    | Technology Options           |
|--------------------------|---------------------------------------------------|------------------------------|
| **ID Generator Node**    | Generate unique IDs using Snowflake algorithm     | Go, Java, Rust (stateless)   |
| **Load Balancer**        | Distribute requests across nodes                  | NGINX, HAProxy, AWS ALB      |
| **Coordination Service** | Assign unique worker IDs, service discovery       | ZooKeeper, etcd, Consul      |
| **Monitoring**           | Track ID generation rate, clock drift, duplicates | Prometheus, Grafana, DataDog |

---

## Snowflake ID Structure (64-bit)

### Bit Layout

```
┌────────────────────────────────────────────────────────────────┐
│ 0 │          41 bits          │  10 bits  │     12 bits       │
│───│───────────────────────────│───────────│───────────────────│
│ S │       Timestamp           │ Worker ID │    Sequence       │
│───│───────────────────────────│───────────│───────────────────│
│ 0 │ Milliseconds since epoch  │  Node ID  │ Counter (0-4095)  │
└────────────────────────────────────────────────────────────────┘

S: Sign bit (always 0 for positive)
```

| Component     | Bits | Range                  | Purpose                                     |
|---------------|------|------------------------|---------------------------------------------|
| **Sign**      | 1    | 0                      | Keep ID positive for Java compatibility     |
| **Timestamp** | 41   | 0 to 2,199,023,255,551 | Milliseconds since custom epoch (~69 years) |
| **Worker ID** | 10   | 0 to 1,023             | Unique node identifier (1024 nodes max)     |
| **Sequence**  | 12   | 0 to 4,095             | Counter within same millisecond per node    |

### Why This Structure?

**Time-Sortable:**

- Timestamp is the most significant bits
- Newer IDs are naturally greater than older IDs
- Database indexes can efficiently range-scan by ID (which correlates with time)

**Globally Unique:**

- Worker ID ensures different nodes never generate same ID
- Sequence ensures same node can't generate duplicates within millisecond

**Compact:**

- 64-bit fits in BIGINT (standard database type)
- Smaller than UUID (128 bits)
- Efficient for indexing and storage

---

## Snowflake Algorithm Implementation

### Core Algorithm Flow

**Initialization:**

- Assign unique worker ID (0-1023) to each generator instance
- Set custom epoch (e.g., January 1, 2020)
- Initialize sequence counter to 0
- Create mutex for thread-safe access

**ID Generation Steps:**

1. **Get Current Timestamp:**
    - Current time in milliseconds since epoch

2. **Handle Clock Drift:**
    - If clock moved backwards < 5ms: Wait for clock to catch up
    - If clock moved backwards > 5ms: Throw error (serious issue)

3. **Manage Sequence:**
    - **Same millisecond:** Increment sequence (0 to 4095)
    - **Sequence overflow:** Wait for next millisecond, reset to 0
    - **New millisecond:** Reset sequence to 0

4. **Compose ID Using Bit Operations:**
    - Shift timestamp left by 22 bits (upper 41 bits)
    - Shift worker_id left by 12 bits (middle 10 bits)
    - Add sequence (lower 12 bits)
    - Combine using bitwise OR

5. **Return Generated ID:**
    - 64-bit positive integer
    - Time-sortable, globally unique

**Key Properties:**

- **Thread-safe:** Mutex protects shared state
- **Capacity:** Up to 4,096 IDs per millisecond per worker
- **Total:** ~4.1 million IDs/sec per worker (4096 × 1000)
- **Parseable:** Can extract timestamp, worker ID, and sequence from ID

**Example:**

```
generator = SnowflakeIDGenerator(worker_id=1)
id1 = generator.generate_id()
// Output: 123456789012345678

parsed = generator.parse_id(id1)
// Output: {
//   timestamp: 1730000000000,
//   timestamp_human: '2024-10-27 00:00:00',
//   worker_id: 1,
//   sequence: 0
// }
```

---

## Worker ID Assignment

### Challenge

Each generator node needs a unique worker ID (0-1023). How do we assign these without conflicts?

### Solution 1: Coordination Service (Recommended)

**Worker ID Management with etcd:**

**How it works:**

1. Node starts, connects to etcd
2. Check if hostname already has assigned worker ID
3. If not, iterate through IDs 0-1023 to find available slot
4. Attempt atomic create with lease (TTL = 30 seconds)
5. If successful:
    - Start background heartbeat thread (refresh every 10s)
    - Store hostname → worker_id mapping
    - Return worker_id
6. If all IDs taken, throw error

**Key Features:**

- Automatic cleanup via TTL if node crashes
- Prevents duplicate assignments via atomic operations
- Heartbeat keeps worker ID alive while node is healthy
- Hostname-based lookup for recovery after restart

### Solution 2: Static Configuration

```yaml
# config.yaml
workers:
  - hostname: node-1.example.com
    worker_id: 1
  - hostname: node-2.example.com
    worker_id: 2
```

**Trade-offs:**

| Method                   | Pros                        | Cons                                  |
|--------------------------|-----------------------------|---------------------------------------|
| **Coordination Service** | Dynamic, automatic failover | Dependency on external service        |
| **Static Config**        | Simple, no dependencies     | Manual management, no auto-failover   |
| **MAC Address Hash**     | Automatic, no coordination  | Risk of collisions (birthday paradox) |
| **Database Assignment**  | Centralized, auditable      | Single point of failure, slower       |

---

## Clock Synchronization & Drift Handling

### Problem: Clock Drift

Real-world server clocks drift over time. NTP corrections can cause:

1. **Clock jumps forward:** Minor issue, just larger gap in IDs
2. **Clock jumps backward:** CATASTROPHIC - can cause duplicate IDs!

### Solution Strategies

**Strategy 1: Refuse to Generate IDs (Conservative)**

- Detect clock backwards, throw error
- Safest but causes service interruption

**Strategy 2: Wait for Clock to Catch Up (Tolerant) - Recommended**

- If clock moves backwards < 5ms: Wait
- If > 5ms: Throw error
- Balances safety and availability

**Strategy 3: Use Last Known Good Timestamp (Aggressive)**

- Continue from last timestamp
- Risk: Might exhaust sequence bits faster
- Dangerous

### Clock Synchronization Best Practices

| Practice                  | Description                    | Benefit              |
|---------------------------|--------------------------------|----------------------|
| **NTP Sync**              | Sync clocks every 1-5 minutes  | Keep drift < 1ms     |
| **Monotonic Clocks**      | Use system monotonic clock API | Never goes backwards |
| **Clock Skew Monitoring** | Alert if drift > 10ms          | Early detection      |
| **Graceful Degradation**  | Handle small backwards jumps   | Better availability  |

---

## Alternative ID Generation Strategies

### Comparison Matrix

| Strategy              | Size    | Sortable          | Performance | Coordination   | Best For             |
|-----------------------|---------|-------------------|-------------|----------------|----------------------|
| **Snowflake**         | 64-bit  | ✅ Time-sorted     | ⭐⭐⭐⭐⭐       | Worker ID only | **Recommended**      |
| **UUID v4**           | 128-bit | ❌ Random          | ⭐⭐⭐⭐        | None           | Simplicity           |
| **UUID v7**           | 128-bit | ✅ Time-sorted     | ⭐⭐⭐⭐        | None           | Sortable UUID needed |
| **ULID**              | 128-bit | ✅ Time-sorted     | ⭐⭐⭐⭐        | None           | UUID alternative     |
| **DB Auto-increment** | 64-bit  | ✅ Strictly sorted | ⭐⭐          | Every insert   | Small scale          |
| **MongoDB ObjectID**  | 96-bit  | ✅ Time-sorted     | ⭐⭐⭐⭐        | None           | MongoDB              |

### When to Use Each

| Scenario                                 | Recommended Strategy | Why                                |
|------------------------------------------|----------------------|------------------------------------|
| **High-scale distributed system**        | Snowflake            | Best performance, sortable, 64-bit |
| **Need strict ordering**                 | DB Auto-increment    | Only option for strict sequence    |
| **Global uniqueness, no coordination**   | UUID v7              | Standard, widely supported         |
| **Small-scale application**              | UUID v4              | Simplest, no infrastructure        |
| **Multi-region, no central coordinator** | ULID or UUID v7      | Sortable without coordination      |
| **MongoDB**                              | ObjectID             | Native support                     |

---

## Handling Edge Cases & Failures

### Sequence Exhaustion

**Problem:** Node needs to generate > 4,096 IDs in 1 millisecond

**Solutions:**

1. **Wait for Next Millisecond** - If sequence overflows, wait for next ms
2. **Increase Sequence Bits** - Custom implementation with 14 bits (16,384 IDs/ms)
3. **Add More Worker Nodes** - 1024 nodes × 4,096 IDs/ms = 4.2 million IDs/ms cluster-wide

### Worker ID Conflicts

**Problem:** Two nodes accidentally get same worker ID

**Prevention:**

1. **Coordination Service with Leases** - etcd/ZooKeeper assigns IDs atomically with TTL + heartbeat
2. **Duplicate Detection** - Store generated IDs in cache, detect duplicates, alert and throw error

### Clock Drift Monitoring

**Monitoring Metrics:**

- Query NTP server every minute
- Compare system time vs NTP time
- Calculate drift = |system_time - ntp_time|
- If drift > 10ms: Log warning
- If drift > 100ms: Alert ops team

---

## High Availability Architecture

### Multi-Region Deployment

**Worker ID Allocation by Region:**

- Region 1: Worker IDs 0-339
- Region 2: Worker IDs 340-679
- Region 3: Worker IDs 680-1023

**Benefits:**

- ✅ Each region operates independently
- ✅ Low latency for local applications
- ✅ No cross-region coordination needed

---

## Monitoring & Observability

### Key Metrics

| Metric                  | Target        | Alert Threshold | Why Important          |
|-------------------------|---------------|-----------------|------------------------|
| **Generation Rate**     | Baseline ±20% | > 2x baseline   | Detect unusual traffic |
| **Latency (p99)**       | < 1ms         | > 10ms          | User experience        |
| **Clock Drift**         | < 10ms        | > 50ms          | Duplicate ID risk      |
| **Sequence Resets**     | Per new ms    | > 100/sec       | Sequence exhaustion    |
| **Clock Backwards**     | 0             | > 0             | Critical issue         |
| **Worker ID Conflicts** | 0             | > 0             | Critical issue         |
| **Error Rate**          | < 0.01%       | > 0.1%          | Service health         |

---

## Common Anti-Patterns to Avoid

### 1. Not Handling Clock Backwards

❌ **Bad:** No clock backwards handling → If clock moves backwards, generates duplicate IDs!

✅ **Good:** Always check if `timestamp < last_timestamp`. If yes, throw error or wait.

### 2. Using System Time Directly

❌ **Bad:** System time can jump backwards

```python
timestamp = system_time_millis()  # Can jump backwards!
```

✅ **Good:** Use monotonic clock (never goes backwards) or track last timestamp and use `max(current, last)`.

### 3. No Worker ID Coordination

❌ **Bad:** Random worker ID (collision risk!)

```python
worker_id = random(0, 1023)
```

✅ **Good:** Coordinated assignment

```python
worker_id = acquire_worker_id_from_etcd()
```

### 4. Blocking on ID Generation

❌ **Bad:** Synchronous, blocks application

```python
id = http_get('http://id-service/generate').id  # Network call!
```

✅ **Good:** Embed generator in application (no network call)

```python
generator = SnowflakeIDGenerator(worker_id=1)
id = generator.generate_id()  # < 1ms, no network
```

---

## Performance Optimizations

### Batch Generation

Generate multiple IDs at once by holding lock once. Example: Generate 1000 IDs in single lock acquisition → Much faster
than 1000 individual calls.

### Lock-Free Implementation (Advanced)

**Lock-Free Snowflake:**

- Each thread maintains own sequence counter
- No mutex needed
- Higher throughput
- More complex implementation

---

## Capacity Planning

### Scaling Guidelines

| Load                   | Worker Nodes | Rationale         |
|------------------------|--------------|-------------------|
| **< 10K IDs/sec**      | 2-3 nodes    | Redundancy for HA |
| **10K - 100K IDs/sec** | 10-20 nodes  | Distribute load   |
| **100K - 1M IDs/sec**  | 50-100 nodes | High throughput   |
| **> 1M IDs/sec**       | 100+ nodes   | Maximum scale     |

### Cost Estimation (AWS)

```
Instance Type: t3.medium (2 vCPU, 4 GB RAM)
Cost: $0.0416/hour

10 nodes × $0.0416/hour × 730 hours/month = $304/month

Load Balancer (ALB): ~$20/month
etcd cluster (3 nodes, t3.micro): ~$20/month

Total: ~$350/month for 100K IDs/sec capacity
```

---

## Trade-offs Summary

| Decision         | Choice         | Alternative    | Why Chosen               | Trade-off                    |
|------------------|----------------|----------------|--------------------------|------------------------------|
| **ID Size**      | 64-bit         | 128-bit UUID   | Smaller, standard BIGINT | Can't store 128-bit metadata |
| **Coordination** | Worker ID only | None           | Prevents collisions      | Needs etcd/ZooKeeper         |
| **Sortability**  | Time-sorted    | Random         | Better DB performance    | Slight predictability        |
| **Generation**   | Local          | Remote service | No network latency       | Need to embed in apps        |
| **Clock Drift**  | Refuse or wait | Continue       | Prevent duplicates       | Temporary unavailability     |

---

## Key Takeaways

1. **Snowflake algorithm** provides globally unique, time-sortable 64-bit IDs
2. **Worker ID coordination** (etcd/ZooKeeper) prevents collisions
3. **Clock synchronization** (NTP) is critical - monitor drift continuously
4. **Handle clock backwards** gracefully (wait or refuse)
5. **Embed in application** (no network call) for < 1ms latency
6. **Sequence bits limit** capacity per node (4,096 IDs/ms)
7. **Horizontal scaling** via more worker nodes (up to 1024)
8. **Time-sortable** enables efficient database indexing
9. **64-bit** fits in standard BIGINT type
10. **No coordination during generation** - each node works independently

---

## Recommended Stack

- **Language:** Go (performance) or Java (enterprise)
- **Coordination:** etcd (lightweight) or ZooKeeper (battle-tested)
- **Clock Sync:** chrony or ntpd
- **Monitoring:** Prometheus + Grafana
- **Deployment:** Embedded in application services (no separate service)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (ID structure, generation flow, worker assignment)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (ID generation, clock drift handling,
  failover)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (Snowflake algorithm, worker ID management, clock
  handling)

