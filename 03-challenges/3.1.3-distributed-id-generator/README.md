# 3.1.3 Design a Distributed ID Generator (Snowflake)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## 1. Problem Statement

Design a highly available, distributed ID generation service similar to Twitter's Snowflake that can generate globally
unique,
time-sortable, 64-bit integer IDs at scale. The system must generate millions of IDs per second across multiple data
centers while maintaining uniqueness guarantees without requiring coordination between nodes during ID generation.

---

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

| Requirement           | Description                                                          | Priority    |
|-----------------------|----------------------------------------------------------------------|-------------|
| **Global Uniqueness** | Every generated ID must be globally unique across all data centers   | Must Have   |
| **Numerical**         | IDs must be 64-bit integers (BIGINT) for efficient database indexing | Must Have   |
| **Time-Sortable**     | Newer IDs should be numerically greater than older IDs (roughly)     | Must Have   |
| **No Coordination**   | ID generation should not require inter-node communication            | Must Have   |
| **High Availability** | Service must remain available even if some nodes fail                | Must Have   |
| **K-Sortable**        | IDs generated within same time window should be roughly sortable     | Should Have |

### Non-Functional Requirements (NFRs)

| Requirement           | Target                | Rationale                                 |
|-----------------------|-----------------------|-------------------------------------------|
| **Low Latency**       | < 1 ms per ID         | Should not slow down application          |
| **High Throughput**   | 100K IDs/sec per node | Support high-traffic applications         |
| **High Availability** | 99.999% uptime        | ID generation failure stops entire system |
| **Scalability**       | Linear horizontal     | Add nodes without coordination overhead   |
| **Clock Tolerance**   | Handle clock drift    | Real-world clocks are never perfect       |

---

### Scale Estimation

#### Scenario: Large Social Media Platform

| Metric                  | Assumption               | Calculation                | Result                   |
|-------------------------|--------------------------|----------------------------|--------------------------|
| **Average QPS**         | 10K writes/sec baseline  | 10,000 QPS                 | **10K IDs/sec**          |
| **Peak QPS**            | 10x burst during events  | 10,000 × 10                | **100K IDs/sec peak**    |
| **Daily IDs**           | 10K QPS sustained        | 10,000 × 86,400 sec/day    | **~864 million IDs/day** |
| **IDs per Node**        | 10 nodes sharing load    | 100K / 10                  | **10K IDs/sec per node** |
| **IDs per Millisecond** | Per node capacity        | 10,000 / 1,000 ms          | **10 IDs/ms per node**   |
| **Sequence Bits**       | Support 4096 IDs/ms      | 2^12 = 4096                | **12 bits sufficient**   |
| **Timestamp Lifespan**  | 41 bits at 1ms precision | 2^41 / (1000×60×60×24×365) | **~69 years**            |
| **Total Nodes**         | 10 bits for node ID      | 2^10                       | **Up to 1024 nodes**     |

#### Back-of-the-Envelope Calculations

```
64-bit ID Structure:
├─ 1 bit (sign): Always 0 (positive number)
├─ 41 bits (timestamp): Milliseconds since epoch = 2^41 ms = 69 years
├─ 10 bits (worker ID): Up to 2^10 = 1024 worker nodes
└─ 12 bits (sequence): Up to 2^12 = 4,096 IDs per millisecond per node

Max IDs per second per node = 4,096 × 1,000 = 4,096,000 IDs/sec
Max IDs per second cluster (1024 nodes) = 4.2 billion IDs/sec

Our need: 100K IDs/sec (well within capacity)
```

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

### System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Application Layer                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  Service │  │  Service │  │  Service │  │  Service │        │
│  │    A     │  │    B     │  │    C     │  │    D     │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
└───────┼─────────────┼─────────────┼─────────────┼──────────────┘
        │             │             │             │
        └─────────────┴──────┬──────┴─────────────┘
                             │
                    ┌────────▼────────┐
                    │  Load Balancer  │
                    │   (Round Robin) │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ ID Generator │     │ ID Generator │     │ ID Generator │
│   Node 1     │     │   Node 2     │     │   Node N     │
│ Worker ID: 1 │     │ Worker ID: 2 │     │ Worker ID: N │
│              │     │              │     │              │
│ ┌──────────┐ │     │ ┌──────────┐ │     │ ┌──────────┐ │
│ │Snowflake │ │     │ │Snowflake │ │     │ │Snowflake │ │
│ │Algorithm │ │     │ │Algorithm │ │     │ │Algorithm │ │
│ └──────────┘ │     │ └──────────┘ │     │ └──────────┘ │
│              │     │              │     │              │
│ Sequence: 0  │     │ Sequence: 0  │     │ Sequence: 0  │
│ Last TS: ... │     │ Last TS: ... │     │ Last TS: ... │
└──────────────┘     └──────────────┘     └──────────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Coordination   │
                    │    Service      │
                    │ (ZooKeeper/etcd)│
                    │                 │
                    │ - Assigns Worker│
                    │   IDs on startup│
                    │ - Health checks │
                    └─────────────────┘
```

### Key Components

| Component                | Responsibility                                    | Technology Options           |
|--------------------------|---------------------------------------------------|------------------------------|
| **ID Generator Node**    | Generate unique IDs using Snowflake algorithm     | Go, Java, Rust (stateless)   |
| **Load Balancer**        | Distribute requests across nodes                  | NGINX, HAProxy, AWS ALB      |
| **Coordination Service** | Assign unique worker IDs, service discovery       | ZooKeeper, etcd, Consul      |
| **Monitoring**           | Track ID generation rate, clock drift, duplicates | Prometheus, Grafana, DataDog |

---

## 4. Detailed Component Design

### 3.1 Snowflake ID Structure

#### 64-bit Layout

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

#### Why This Structure?

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

### 3.2 Snowflake Algorithm Implementation

#### Core Algorithm

**How Snowflake ID Generation Works:**

**Initialization:**

- Assign unique worker ID (0-1023) to each generator instance
- Set custom epoch (e.g., January 1, 2020)
- Initialize sequence counter to 0
- Create mutex for thread-safe access

**ID Generation Flow:**

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

*See `pseudocode.md::SnowflakeGenerator` for detailed implementation with all helper functions*

**Example Usage:**

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

### 3.3 Worker ID Assignment

#### Challenge

Each generator node needs a unique worker ID (0-1023). How do we assign these without conflicts?

#### Solution 1: Coordination Service (Recommended)

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

*See `pseudocode.md::WorkerIDManager` for detailed implementation*

#### Solution 2: Static Configuration

```yaml
# config.yaml
workers:
  - hostname: node-1.example.com
    worker_id: 1
  - hostname: node-2.example.com
    worker_id: 2
  - hostname: node-3.example.com
    worker_id: 3
```

**Trade-offs:**

| Method                   | Pros                        | Cons                                  |
|--------------------------|-----------------------------|---------------------------------------|
| **Coordination Service** | Dynamic, automatic failover | Dependency on external service        |
| **Static Config**        | Simple, no dependencies     | Manual management, no auto-failover   |
| **MAC Address Hash**     | Automatic, no coordination  | Risk of collisions (birthday paradox) |
| **Database Assignment**  | Centralized, auditable      | Single point of failure, slower       |

---

### 3.4 Clock Synchronization & Drift Handling

#### Problem: Clock Drift

Real-world server clocks drift over time. NTP corrections can cause:

1. **Clock jumps forward**: Minor issue, just larger gap in IDs
2. **Clock jumps backward**: CATASTROPHIC - can cause duplicate IDs!

#### Solution Strategies

**Strategy 1: Refuse to Generate IDs (Conservative)**

Detect clock backwards, throw error. Safest but causes service interruption.

**Strategy 2: Wait for Clock to Catch Up (Tolerant) - Recommended**

If clock moves backwards < 5ms: Wait. If > 5ms: Throw error. Balances safety and availability.

**Strategy 3: Use Last Known Good Timestamp (Aggressive)**

Continue from last timestamp. Risk: Might exhaust sequence bits faster. Dangerous.

*See `pseudocode.md::next_id()` for all strategy implementations*

#### Clock Synchronization Best Practices

| Practice                  | Description                    | Benefit              |
|---------------------------|--------------------------------|----------------------|
| **NTP Sync**              | Sync clocks every 1-5 minutes  | Keep drift < 1ms     |
| **Monotonic Clocks**      | Use system monotonic clock API | Never goes backwards |
| **Clock Skew Monitoring** | Alert if drift > 10ms          | Early detection      |
| **Graceful Degradation**  | Handle small backwards jumps   | Better availability  |

---

## 5. Alternative ID Generation Strategies

### Comparison Matrix

| Strategy              | Size    | Sortable          | Performance | Coordination   | Best For             |
|-----------------------|---------|-------------------|-------------|----------------|----------------------|
| **Snowflake**         | 64-bit  | ✅ Time-sorted     | ⭐⭐⭐⭐⭐       | Worker ID only | **Recommended**      |
| **UUID v4**           | 128-bit | ❌ Random          | ⭐⭐⭐⭐        | None           | Simplicity           |
| **UUID v7**           | 128-bit | ✅ Time-sorted     | ⭐⭐⭐⭐        | None           | Sortable UUID needed |
| **ULID**              | 128-bit | ✅ Time-sorted     | ⭐⭐⭐⭐        | None           | UUID alternative     |
| **DB Auto-increment** | 64-bit  | ✅ Strictly sorted | ⭐⭐          | Every insert   | Small scale          |
| **MongoDB ObjectID**  | 96-bit  | ✅ Time-sorted     | ⭐⭐⭐⭐        | None           | MongoDB              |

### Detailed Comparison

#### UUID v4 (Random)

```
// Generate random UUID v4
id = generate_uuid_v4()

// Example: 'f47ac10b-58cc-4372-a567-0e02b2c3d479'
```

**Pros:**

- ✅ Globally unique (virtually impossible to collide)
- ✅ No coordination needed
- ✅ Built into all languages

**Cons:**

- ❌ 128 bits (2x storage of Snowflake)
- ❌ Not sortable (bad for database indexes)
- ❌ Random insertion pattern (poor B-tree performance)

---

#### UUID v7 (Time-ordered)

```
// Generate time-ordered UUID v7
id = generate_uuid_v7()

// Example: '018c4c3c-8f1f-7000-8000-0242ac120002'
//          └───timestamp───┘└────random────┘
```

**Pros:**

- ✅ Time-sortable
- ✅ No coordination needed
- ✅ Better index performance than UUID v4

**Cons:**

- ❌ Still 128 bits
- ❌ Less human-readable

---

#### ULID (Universally Unique Lexicographically Sortable Identifier)

```
// Generate ULID
id = generate_ulid()

// Example: '01ARZ3NDEKTSV4RRFFQ69G5FAV'
```

**Structure:**

```
 01ARZ3NDEKTSV4RRFFQ69G5FAV
 ├─────────┬────────────────┘
 Timestamp   Random (80 bits)
 (48 bits)
```

**Pros:**

- ✅ Time-sortable
- ✅ Lexicographically sortable (as strings)
- ✅ No coordination

**Cons:**

- ❌ 128 bits
- ❌ Less adoption than UUID

---

#### Database Auto-Increment

```sql
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);

INSERT INTO users (name) VALUES ('Alice');
-- Returns id = 1
```

**Pros:**

- ✅ Simple
- ✅ Strictly monotonic
- ✅ 64-bit

**Cons:**

- ❌ **Single point of failure**
- ❌ **Write bottleneck** (all inserts contend for same sequence)
- ❌ **Doesn't scale horizontally**
- ❌ Requires database round-trip for every ID

---

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

## 6. Handling Edge Cases & Failures

### 5.1 Sequence Exhaustion

**Problem:** Node needs to generate > 4,096 IDs in 1 millisecond

**Solutions:**

**Option 1: Wait for Next Millisecond**

```
if sequence == 0:  // Overflowed
  timestamp = wait_next_millis(last_timestamp)
```

**Option 2: Increase Sequence Bits (Custom Implementation)**

```
┌─────────────────────────────────────────────┐
│ 1 │  41 bits  │  8 bits   │   14 bits      │
│───│───────────│───────────│────────────────│
│ 0 │ Timestamp │ Worker ID │   Sequence     │
└─────────────────────────────────────────────┘
       69 years     256 nodes    16,384 IDs/ms
```

**Option 3: Add More Worker Nodes**

- 1024 nodes × 4,096 IDs/ms = 4.2 million IDs/ms cluster-wide

---

### 5.2 Worker ID Conflicts

**Problem:** Two nodes accidentally get same worker ID

**Prevention:**

1. **Coordination Service with Leases**
    - etcd/ZooKeeper assigns IDs atomically
    - TTL + heartbeat ensures stale assignments expire

2. **Duplicate Detection**

```
IDGeneratorWithDuplicateDetection:
  worker_id: integer
  cache_client: CacheClient
  
  generate_id() - With duplicate detection:
    - Generate Snowflake ID
    - Store in cache with worker_id (if not exists)
    - If already exists: CRITICAL duplicate detected → alert and throw error
    - Set 24-hour TTL on cache key for cleanup

*See `pseudocode.md::generate_id()` for implementation*
    
    return id
```

---

### 5.3 Clock Drift Monitoring

```
// Monitoring metrics
clock_drift_gauge = create_gauge('snowflake_clock_drift_ms')
clock_backwards_counter = create_counter('snowflake_clock_backwards_total')

Background monitoring thread:
- Query NTP server every minute
- Compare system time vs NTP time
- Calculate drift = |system_time - ntp_time|
- Record metric
- If drift > 10ms: Log warning
- If drift > 100ms: Alert ops team

*See `pseudocode.md::monitor_clock_drift()` for implementation*
```

---

## 7. High Availability Architecture

### Multi-Region Deployment

```
Region US-East:
┌────────────────────────────────┐
│  ID Generators (Worker ID 0-99)│
│  ┌──────┐  ┌──────┐  ┌──────┐ │
│  │Node 1│  │Node 2│  │Node N│ │
│  └──────┘  └──────┘  └──────┘ │
└────────────────────────────────┘

Region EU-West:
┌────────────────────────────────┐
│ID Generators (Worker ID 100-199)│
│  ┌──────┐  ┌──────┐  ┌──────┐ │
│  │Node 1│  │Node 2│  │Node N│ │
│  └──────┘  └──────┘  └──────┘ │
└────────────────────────────────┘

Region AP-South:
┌────────────────────────────────┐
│ID Generators (Worker ID 200-299)│
│  ┌──────┐  ┌──────┐  ┌──────┐ │
│  │Node 1│  │Node 2│  │Node N│ │
│  └──────┘  └──────┘  └──────┘ │
└────────────────────────────────┘
```

**Benefits:**

- ✅ Each region operates independently
- ✅ Low latency for local applications
- ✅ No cross-region coordination needed

**Worker ID Allocation by Region:**

- Region 1: Worker IDs 0-339
- Region 2: Worker IDs 340-679
- Region 3: Worker IDs 680-1023

---

## 8. Monitoring & Observability

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

### Monitoring Dashboard

```
// Define metrics
ids_generated = create_counter('snowflake_ids_generated_total')
generation_latency = create_histogram('snowflake_generation_seconds')
sequence_resets = create_counter('snowflake_sequence_resets_total')
clock_drift = create_gauge('snowflake_clock_drift_milliseconds')

**Monitored ID generation:**
- Track start time
- Generate ID
- Increment counter
- Track sequence resets
- Record latency

*See `pseudocode.md` for implementation*
    
    return id
```

---

## 9. Common Anti-Patterns

### Anti-Pattern 1: Not Handling Clock Backwards

**Problem:**

❌ **Bad:** No clock backwards handling → If clock moves backwards, generates duplicate IDs!

✅ **Good:** Always check if `timestamp < last_timestamp`. If yes, throw error or wait.

*See `pseudocode.md::next_id()` for implementation*

---

### Anti-Pattern 2: Using System Time Directly

**Problem:**

```
❌ System time can jump

timestamp = system_time_millis()  // Can jump backwards!
```

**Better:**

✅ **Solution:** Use monotonic clock (never goes backwards) or track last timestamp and use `max(current, last)`.

*See `pseudocode.md::current_time_millis()` for implementation*

---

### Anti-Pattern 3: No Worker ID Coordination

**Problem:**

```
❌ Random worker ID (collision risk!)

worker_id = random(0, 1023)
```

**Better:**

```
✅ Coordinated assignment

worker_id = acquire_worker_id_from_etcd()
```

---

### Anti-Pattern 4: Blocking on ID Generation

**Problem:**

```
❌ Synchronous, blocks application

id = http_get('http://id-service/generate').id  // Network call!
```

**Better:**

```
✅ Embed generator in application (no network call)

generator = SnowflakeIDGenerator(worker_id=1)
id = generator.generate_id()  // < 1ms, no network
```

---

## 10. Performance Optimizations

### Batch Generation

Generate multiple IDs at once by holding lock once. Example: Generate 1000 IDs in single lock acquisition → Much faster
than 1000 individual calls.

*See `pseudocode.md::generate_batch()` for implementation*

### Lock-Free Implementation (Advanced)

```
LockFreeSnowflake:
  worker_id: integer
  thread_local_counter: ThreadLocal
  
  generate_id() - Lock-free with thread-local counters:
    - Each thread maintains own sequence counter
    - No mutex needed
    - Higher throughput

*See `pseudocode.md` for implementation*
    
    thread_local_counter.last_ts = timestamp
    
    return construct_id(timestamp, worker_id, thread_local_counter.sequence)
```

---

## 11. Capacity Planning

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

## 12. Trade-offs Summary

| Decision         | Choice         | Alternative    | Why Chosen               | Trade-off                    |
|------------------|----------------|----------------|--------------------------|------------------------------|
| **ID Size**      | 64-bit         | 128-bit UUID   | Smaller, standard BIGINT | Can't store 128-bit metadata |
| **Coordination** | Worker ID only | None           | Prevents collisions      | Needs etcd/ZooKeeper         |
| **Sortability**  | Time-sorted    | Random         | Better DB performance    | Slight predictability        |
| **Generation**   | Local          | Remote service | No network latency       | Need to embed in apps        |
| **Clock Drift**  | Refuse or wait | Continue       | Prevent duplicates       | Temporary unavailability     |

---

## Summary

A distributed ID generator using Snowflake algorithm provides:

**Key Advantages:**

1. ✅ **Globally unique** without coordination during generation
2. ✅ **Time-sortable** for efficient database indexing
3. ✅ **High performance** (< 1ms, millions of IDs/sec)
4. ✅ **64-bit** standard integer (efficient storage)
5. ✅ **Horizontally scalable** (up to 1024 nodes)

**Key Challenges:**

1. ⚠️ Clock synchronization (NTP required)
2. ⚠️ Worker ID management (needs coordination service)
3. ⚠️ Clock backwards handling (graceful degradation)
4. ⚠️ Sequence exhaustion at extreme load per node

**Recommended Stack:**

- **Language:** Go (performance) or Java (enterprise)
- **Coordination:** etcd (lightweight) or ZooKeeper (battle-tested)
- **Clock Sync:** chrony or ntpd
- **Monitoring:** Prometheus + Grafana
- **Deployment:** Embedded in application services (no separate service)

The Snowflake algorithm is the **gold standard** for distributed ID generation in high-scale systems.

---

## 13. References

### Related System Design Components

- **[2.5.2 Consensus Algorithms](../../02-components/2.5-algorithms/2.5.2-consensus-algorithms.md)** - Coordination
  patterns for distributed systems
- **[1.1.2 Latency, Throughput, and Scale](../../01-principles/1.1.2-latency-throughput-scale.md)** - Performance
  fundamentals
- **[1.1.6 Failure Modes and Fault Tolerance](../../01-principles/1.1.6-failure-modes-fault-tolerance.md)** - High
  availability patterns
- **[2.2.2 Consistent Hashing](../../02-components/2.2-caching/2.2.2-consistent-hashing.md)** - Distributed system
  partitioning

### Re-ated Design Challenges

- **[3.1.1 URL Shortener](../3.1.1-url-shortener/)** - Uses distributed ID generator (Snowflake) for unique aliases
- **[3.1.2 Distributed Cache](../3.1.2-distributed-cache/)** - Coordination service patterns (etcd/ZooKeeper)
- **[3.2.4 Global Rate Limiter](../3.2.4-global-rate-limiter/)** - Distributed coordination patterns

### External Resources

- **Twitter's Snowflake:
  ** [Original Snowflake Blog Post](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake.html) -
  Original announcement and design
- **Snowflake Algorithm:** [GitHub - Twitter Snowflake](https://github.com/twitter-archive/snowflake) - Reference
  implementation
- **UUID v7 Specification:** [RFC 4122](https://www.rfc-editor.org/rfc/rfc4122) - Time-ordered UUID standard
- **ULID Specification:** [ULID GitHub](https://github.com/ulid/spec) - Universally Unique Lexicographically Sortable
  Identifier
- **etcd Documentation:** [etcd.io](https://etcd.io/docs/) - Distributed coordination service
- **ZooKeeper Documentation:** [Apache ZooKeeper](https://zookeeper.apache.org/) - Distributed coordination service

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Chapters on distributed systems and coordination
- *Distributed Systems: Concepts and Design* by George Coulouris - Coordination and consensus algorithms
- *System Design Interview Vol 1* by Alex Xu - ID generation patterns and trade-offs