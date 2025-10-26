# 3.1.3 Design a Distributed ID Generator (Snowflake)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions. 
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, Snowflake ID structure, worker ID assignment, capacity analysis, and deployment strategies
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows for ID generation, clock drift handling, failover scenarios, and monitoring
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices and trade-offs
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for all core functions

---

## Problem Statement

Design a highly available, distributed ID generation service similar to Twitter's Snowflake that can generate globally
unique,
time-sortable, 64-bit integer IDs at scale. The system must generate millions of IDs per second across multiple data
centers while maintaining uniqueness guarantees without requiring coordination between nodes during ID generation.

---

## 1. Requirements and Scale Estimation

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

## 2. High-Level Architecture

> 📊 **See detailed architecture diagrams:** [HLD Diagrams](./hld-diagram.md)

### Key Components

| Component                | Responsibility                                    | Technology Options           |
|--------------------------|---------------------------------------------------|------------------------------|
| **ID Generator Node**    | Generate unique IDs using Snowflake algorithm     | Go, Java, Rust (stateless)   |
| **Load Balancer**        | Distribute requests across nodes                  | NGINX, HAProxy, AWS ALB      |
| **Coordination Service** | Assign unique worker IDs, service discovery       | ZooKeeper, etcd, Consul      |
| **Monitoring**           | Track ID generation rate, clock drift, duplicates | Prometheus, Grafana, DataDog |

---

## 3. Detailed Component Design

### 3.1 Snowflake ID Structure

> 📊 **See ID structure breakdown:** [Snowflake ID Structure Diagram](./hld-diagram.md#snowflake-id-structure-64-bit)

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

> 📊 **See generation flow:** [ID Generation Flow Diagram](./hld-diagram.md#id-generation-flow)
> 
> 📊 **See detailed sequence:** [ID Generation Sequence](./sequence-diagrams.md#id-generation-flow-happy-path)

#### Core Algorithm

```
SnowflakeIDGenerator:
  // Constants
  EPOCH = 1577836800000  // January 1, 2020 00:00:00 UTC (milliseconds)
  
  // Bit allocation
  WORKER_ID_BITS = 10
  SEQUENCE_BITS = 12
  
  // Max values
  MAX_WORKER_ID = (2^WORKER_ID_BITS) - 1  // 1023
  MAX_SEQUENCE = (2^SEQUENCE_BITS) - 1    // 4095
  
  // Bit shifts
  TIMESTAMP_SHIFT = WORKER_ID_BITS + SEQUENCE_BITS  // 22
  WORKER_ID_SHIFT = SEQUENCE_BITS                    // 12
  
  // State variables
  worker_id: integer
  sequence: integer = 0
  last_timestamp: integer = -1
  lock: Mutex  // Thread-safe access
  
  
  function initialize(worker_id):
    if worker_id < 0 or worker_id > MAX_WORKER_ID:
      raise error("Worker ID must be between 0 and " + MAX_WORKER_ID)
    
    this.worker_id = worker_id
  
  
  function generate_id():
    acquire_lock(lock):
      timestamp = current_millis()
      
      // Clock moved backwards! Handle clock drift
      if timestamp < last_timestamp:
        offset = last_timestamp - timestamp
        if offset > 5:  // More than 5ms backwards
          raise error("Clock moved backwards by " + offset + "ms. Refusing to generate ID.")
        
        // Wait for clock to catch up
        sleep(offset milliseconds)
        timestamp = current_millis()
      
      // Same millisecond - increment sequence
      if timestamp == last_timestamp:
        sequence = (sequence + 1) AND MAX_SEQUENCE  // Bitmask for wraparound
        
        // Sequence overflow - wait for next millisecond
        if sequence == 0:
          timestamp = wait_next_millis(last_timestamp)
      else:
        // New millisecond - reset sequence
        sequence = 0
      
      last_timestamp = timestamp
      
      // Construct ID using bit operations
      id = ((timestamp - EPOCH) << TIMESTAMP_SHIFT) OR
           (worker_id << WORKER_ID_SHIFT) OR
           sequence
      
      return id
  
  
  function current_millis():
    return current_system_time_in_milliseconds()
  
  
  function wait_next_millis(last_timestamp):
    timestamp = current_millis()
    while timestamp <= last_timestamp:
      timestamp = current_millis()
    return timestamp
  
  
  function parse_id(id):
    // Extract components from ID
    timestamp = ((id >> TIMESTAMP_SHIFT) + EPOCH)
    worker_id = (id >> WORKER_ID_SHIFT) AND MAX_WORKER_ID
    sequence = id AND MAX_SEQUENCE
    
    return {
      timestamp: timestamp,
      timestamp_human: format_timestamp(timestamp),
      worker_id: worker_id,
      sequence: sequence
    }
```

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

> 📊 **See assignment strategy:** [Worker ID Assignment Diagram](./hld-diagram.md#worker-id-assignment-strategy)
> 
> 📊 **See detailed sequence:** [Worker ID Assignment Sequence](./sequence-diagrams.md#worker-id-assignment-startup)

#### Challenge

Each generator node needs a unique worker ID (0-1023). How do we assign these without conflicts?

#### Solution 1: Coordination Service (Recommended)

```
WorkerIDManager:
  etcd_client: EtcdClient
  prefix: string = "/snowflake/workers/"
  
  function initialize(etcd_host, etcd_port):
    etcd_client = connect_to_etcd(etcd_host, etcd_port)
  
  function acquire_worker_id(node_hostname):
    // Try to find existing assignment
    existing = etcd_client.get(prefix + node_hostname)
    if existing is not null:
      return parse_int(existing)
    
    // Find available worker ID
    for worker_id from 0 to 1023:
      key = prefix + "id_" + worker_id
      
      // Try to create with lease (TTL for automatic cleanup)
      lease = etcd_client.create_lease(ttl=30_seconds)
      success = etcd_client.put_if_not_exists(key, node_hostname, lease)
      
      if success:
        // Start heartbeat thread to keep lease alive
        start_background_thread(heartbeat(lease))
        
        // Store hostname → worker_id mapping
        etcd_client.put(prefix + node_hostname, worker_id)
        return worker_id
    
    raise error("No available worker IDs")
  
  function heartbeat(lease):
    // Keep lease alive with periodic heartbeat
    while true:
      lease.refresh()
      sleep(10_seconds)  // Refresh every 10 seconds

// Usage
manager = WorkerIDManager(etcd_host="localhost", etcd_port=2379)
worker_id = manager.acquire_worker_id("node-1.example.com")
generator = SnowflakeIDGenerator(worker_id)
```

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

> 📊 **See drift handling flow:** [Clock Drift Handling Diagram](./hld-diagram.md#clock-drift-handling)
> 
> 📊 **See detailed scenarios:** [Clock Drift Detection Sequences](./sequence-diagrams.md#clock-drift-detection--handling)

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

*See `pseudocode.md::next_id()` for detailed implementations*

#### Clock Synchronization Best Practices

| Practice                  | Description                    | Benefit              |
|---------------------------|--------------------------------|----------------------|
| **NTP Sync**              | Sync clocks every 1-5 minutes  | Keep drift < 1ms     |
| **Monotonic Clocks**      | Use system monotonic clock API | Never goes backwards |
| **Clock Skew Monitoring** | Alert if drift > 10ms          | Early detection      |
| **Graceful Degradation**  | Handle small backwards jumps   | Better availability  |

---

## 4. Alternative ID Generation Strategies

> 📊 **See comparison:** [ID Strategy Comparison Diagram](./hld-diagram.md#comparison-with-alternative-id-generation-strategies)
> 
> 📊 **See performance comparison:** [Performance Comparison Sequences](./sequence-diagrams.md#comparison-snowflake-vs-uuid-vs-auto-increment)

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

## 5. Handling Edge Cases & Failures

> 📊 **See failure scenarios:** [Failure Scenarios Diagram](./hld-diagram.md#failure-scenarios--recovery)

### 5.1 Sequence Exhaustion

> 📊 **See exhaustion handling:** [Sequence Exhaustion Sequence](./sequence-diagrams.md#sequence-exhaustion-handling)

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
  
  function generate_id():
    id = generate_snowflake_id()
    
    // Check for duplicate (should never happen)
    key = "id:" + id
    if not cache_client.set_if_not_exists(key, worker_id):
      // CRITICAL: Duplicate detected!
      alert_duplicate(id)
      raise error("Duplicate ID generated: " + id)
    
    // Set expiry (cleanup old IDs)
    cache_client.expire(key, 86400)  // 24 hours
    
    return id
```

---

## 6. High Availability Architecture

> 📊 **See deployment architecture:** [Deployment Architecture Diagram](./hld-diagram.md#deployment-architecture)
> 
> 📊 **See multi-region setup:** [Multi-Region Deployment Diagram](./hld-diagram.md#multi-region-deployment)

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

## 7. Monitoring & Observability

> 📊 **See monitoring dashboard:** [Monitoring Dashboard Diagram](./hld-diagram.md#monitoring--observability-dashboard)
> 
> 📊 **See monitoring flow:** [Monitoring Sequence](./sequence-diagrams.md#monitoring--alerting-flow)

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

MonitoredSnowflakeGenerator extends SnowflakeIDGenerator:
  function generate_id():
    start_time = current_time()
    
    id = super.generate_id()
    ids_generated.increment()
    
    if this.sequence == 0:
      sequence_resets.increment()
    
    latency = current_time() - start_time
    generation_latency.observe(latency)
    
    return id
```

---

## 8. Common Anti-Patterns

### Anti-Pattern 1: Not Handling Clock Backwards

**Problem:** ❌ No clock backwards handling → If clock moves backwards, generates duplicate IDs!

**Solution:** ✅ Always check if `timestamp < last_timestamp`. If yes, throw error or wait.

*See `pseudocode.md::next_id()` for implementation*

---

### Anti-Pattern 2: Using System Time Directly

**Problem:**

```
❌ System time can jump

timestamp = system_time_millis()  // Can jump backwards!
```

**Solution:** ✅ Use monotonic clock (never goes backwards) or track last timestamp and use `max(current, last)`.

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

## 9. Performance Optimizations

> 📊 **See batch generation:** [Batch ID Generation Sequence](./sequence-diagrams.md#batch-id-generation-optimization)

### Batch Generation

Generate multiple IDs at once by holding lock once. Example: Generate 1000 IDs in single lock acquisition → Much faster than 1000 individual calls.

*See `pseudocode.md::generate_batch()` for implementation*

### Lock-Free Implementation (Advanced)

```
LockFreeSnowflake:
  worker_id: integer
  thread_local_counter: ThreadLocal
  
  function generate_id():
    // Each thread has its own counter (lock-free)
    if not thread_local_counter.initialized:
      thread_local_counter.sequence = 0
      thread_local_counter.last_ts = -1
    
    timestamp = current_millis()
    
    if timestamp == thread_local_counter.last_ts:
      thread_local_counter.sequence = (thread_local_counter.sequence + 1) AND 4095
    else:
      thread_local_counter.sequence = 0
    
    thread_local_counter.last_ts = timestamp
    
    return construct_id(timestamp, worker_id, thread_local_counter.sequence)
```

---

## 10. Capacity Planning

> 📊 **See scaling strategy:** [Scaling Strategy Diagram](./hld-diagram.md#scaling-strategy)

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

## 11. Trade-offs Summary

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

