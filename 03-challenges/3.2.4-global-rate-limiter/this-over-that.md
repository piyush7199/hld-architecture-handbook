# Global Rate Limiter - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made in the Global Rate Limiter system,
including alternatives considered, trade-offs, and conditions that would change these decisions.

---

## Table of Contents

1. [Rate Limiting Algorithm: Token Bucket vs Sliding Window vs Fixed Window](#1-rate-limiting-algorithm-token-bucket-vs-sliding-window-vs-fixed-window)
2. [Storage System: Redis vs RDBMS vs In-Memory](#2-storage-system-redis-vs-rdbms-vs-in-memory)
3. [Consistency Model: Atomic INCR vs Distributed Locks](#3-consistency-model-atomic-incr-vs-distributed-locks)
4. [Failure Strategy: Fail-Open vs Fail-Close](#4-failure-strategy-fail-open-vs-fail-close)
5. [Scaling Strategy: Sharding by User ID vs Global Counter](#5-scaling-strategy-sharding-by-user-id-vs-global-counter)
6. [Hot Key Mitigation: L1 Cache vs Replication vs Ignore](#6-hot-key-mitigation-l1-cache-vs-replication-vs-ignore)
7. [Configuration Management: etcd vs Database vs Hardcoded](#7-configuration-management-etcd-vs-database-vs-hardcoded)

---

## 1. Rate Limiting Algorithm: Token Bucket vs Sliding Window vs Fixed Window

### The Problem

Choose an algorithm that:

- Prevents abuse (no boundary bursts)
- Handles bursty traffic fairly
- Provides sub-millisecond latency
- Minimizes storage requirements

**Constraints:**

- Must support 500K QPS
- < 1ms latency per check
- Minimize Redis operations

### Options Considered

| Algorithm                  | Accuracy | Storage    | Latency | Burst Handling                          | Complexity |
|----------------------------|----------|------------|---------|-----------------------------------------|------------|
| **Token Bucket**           | High     | 2 values   | ~0.8ms  | ✅ Excellent (allows accumulated tokens) | Medium     |
| **Sliding Window Counter** | High     | 2 counters | ~1ms    | ✅ Good (weighted sum)                   | Medium     |
| **Sliding Log**            | Perfect  | N entries  | ~10ms   | ✅ Perfect (exact count)                 | High       |
| **Fixed Window**           | Low      | 1 counter  | ~0.5ms  | ❌ Poor (boundary bursts)                | Low        |
| **Leaky Bucket**           | High     | 2 values   | ~0.8ms  | ❌ Poor (strict rate, no bursts)         | Medium     |

#### Option A: Token Bucket ✅

**How It Works:**

```
Bucket capacity: 10 tokens
Refill rate: 10 tokens/second

Time 0s: tokens = 10
→ Make 10 requests → tokens = 0
→ Wait 1 second
Time 1s: tokens = 0 + (10 × 1) = 10
→ Make 15 requests → tokens = 0 (capped at capacity)
```

**Storage:**

```
user:12345:tokens → 5 (current token count)
user:12345:last_refill → 1704067200 (Unix timestamp)
```

**Pros:**

- ✅ **Burst Tolerance:** Allows using accumulated tokens (better UX)
- ✅ **Smooth:** Refills continuously (not reset at boundaries)
- ✅ **Fair:** Unused capacity doesn't go to waste
- ✅ **Production-Proven:** Used by Stripe, AWS, Shopify

**Cons:**

- ❌ **Slightly More Storage:** 2 values vs 1 counter
- ❌ **More Complex:** Refill calculation required

#### Option B: Sliding Window Counter ✅

**How It Works:**

```
Limit: 100 requests/second
Current time: 1.5 seconds into minute

Previous window (0-1s): 80 requests
Current window (1-2s): 30 requests

Rate = 80 × (1 - 0.5) + 30 = 40 + 30 = 70
70 < 100 → Allow ✅
```

**Storage:**

```
user:12345:window:1704067201 → 30 (current window)
user:12345:window:1704067200 → 80 (previous window)
```

**Pros:**

- ✅ **More Accurate:** No boundary burst problem
- ✅ **Less Storage:** 2 counters vs full log
- ✅ **Good Approximation:** Close to true sliding window

**Cons:**

- ❌ **Approximation:** Not 100% accurate (weighted sum)
- ❌ **2 Redis Reads:** Current + previous window

#### Option C: Sliding Log (Not Chosen)

**How It Works:**

- Store timestamp of every request
- Count requests in last N seconds
- Perfect accuracy

**Storage:**

```
user:12345:log → [1704067200, 1704067201, ..., 1704067250] (50 entries)
```

**Pros:**

- ✅ **Perfect Accuracy:** Exact request count

**Cons:**

- ❌ **Too Much Storage:** N entries per user (100 entries for 100 QPS limit)
- ❌ **Slow:** Must scan all entries to count (~10ms)
- ❌ **Cannot Scale:** 1M users × 100 entries = 100M Redis entries

**Why Not Chosen:** Storage and latency requirements not met.

#### Option D: Fixed Window (Not Recommended)

**How It Works:**

```
Window: 10:00:00 - 10:00:59 (limit: 100)

User makes 100 requests at 10:00:59
User makes 100 requests at 10:01:00 (new window)

Total: 200 requests in 1 second! ❌
```

**Pros:**

- ✅ **Simplest:** Single counter
- ✅ **Fastest:** Single INCR (~0.5ms)

**Cons:**

- ❌ **Boundary Burst:** Can exceed 2× limit at window boundaries
- ❌ **Not Production-Ready:** Too inaccurate

**Why Not Chosen:** Boundary burst problem unacceptable for strict rate limiting.

### Decision Made

**✅ Token Bucket for bursty APIs, Sliding Window for strict enforcement**

**Rationale:**

1. **Token Bucket:**
    - Best for APIs with natural bursts (video uploads, file downloads)
    - Better user experience (accumulated tokens)
    - Used by major providers (Stripe, AWS)
2. **Sliding Window:**
    - Best for strict limits (financial APIs, write operations)
    - Prevents boundary bursts
    - Good balance of accuracy and performance

**Implementation:**

- Default: Token Bucket (better UX)
- Override per endpoint: Sliding Window (critical operations)
- Never use: Fixed Window (too inaccurate), Sliding Log (too slow)

### Trade-offs Accepted

| Trade-off         | Impact                                   | Mitigation                                        |
|-------------------|------------------------------------------|---------------------------------------------------|
| **Complexity**    | Token Bucket requires refill calculation | Well-documented algorithm, proven implementations |
| **Storage**       | 2 values vs 1 counter                    | Negligible (2 bytes vs 1 byte per user)           |
| **Approximation** | Sliding Window not 100% accurate         | <1% error acceptable                              |

### When to Reconsider

**Switch to Sliding Window if:**

- Boundary bursts become a problem
- Strict enforcement required (financial, security)

**Switch to Fixed Window if:**

- Latency becomes critical (need <0.5ms)
- Willing to accept boundary bursts
- Coarse limits only (hourly/daily quotas)

---

## 2. Storage System: Redis vs RDBMS vs In-Memory

### The Problem

Store rate limit counters for 1M users with 1M ops/sec throughput.

**Requirements:**

- Sub-millisecond latency (<1ms)
- Atomic operations (prevent race conditions)
- High throughput (1M ops/sec)
- Durable (counters persist across restarts)

### Options Considered

| System                | Latency | Throughput | Atomicity     | Durability        | Cost   |
|-----------------------|---------|------------|---------------|-------------------|--------|
| **Redis**             | <1ms    | 100K/node  | ✅ INCR        | ⚠️ AOF/RDB        | Medium |
| **PostgreSQL**        | 10-50ms | 5K/node    | ✅ Row locks   | ✅ WAL             | High   |
| **In-Memory (Local)** | <0.1ms  | 1M+        | ❌ No global   | ❌ Lost on restart | Low    |
| **Memcached**         | <1ms    | 100K/node  | ❌ No INCR     | ❌ None            | Low    |
| **DynamoDB**          | 5-10ms  | Variable   | ✅ Conditional | ✅ Replicated      | High   |

#### Option A: Redis ✅ (Chosen)

**Pros:**

- ✅ **Sub-Millisecond:** <1ms latency (in-memory)
- ✅ **Atomic Operations:** INCR, INCRBY (single-threaded execution)
- ✅ **High Throughput:** 100K+ ops/sec per node
- ✅ **Horizontal Scaling:** Redis Cluster supports sharding
- ✅ **Durability:** AOF/RDB persistence (optional)
- ✅ **TTL Support:** Auto-expire keys
- ✅ **Production-Proven:** Used by all major tech companies

**Cons:**

- ❌ **Memory Cost:** ~$500/TB (expensive)
- ❌ **Durability Risk:** Async persistence can lose seconds of data
- ❌ **Ops Complexity:** Requires cluster management

**Example:**

```
INCR user:12345:count    // Atomic, O(1), <1ms
EXPIRE user:12345:count 3600  // Auto-cleanup
```

#### Option B: PostgreSQL (Not Chosen)

**Pros:**

- ✅ **ACID:** Strong consistency, durable writes
- ✅ **Familiar:** Well-understood, mature ecosystem
- ✅ **Cheaper Storage:** ~$100/TB (disk-based)

**Cons:**

- ❌ **Too Slow:** 10-50ms per operation (disk I/O + locking)
- ❌ **Low Throughput:** ~5K ops/sec per node (vs 100K for Redis)
- ❌ **Lock Contention:** Row-level locks at high concurrency
- ❌ **Cannot Scale:** 1M ops/sec would require 200 nodes

**Why Not Chosen:** Latency (10-50ms) exceeds requirement (<1ms). Throughput insufficient.

#### Option C: In-Memory Local Counters (Not Chosen)

**Pros:**

- ✅ **Fastest:** <0.1ms latency (no network)
- ✅ **Highest Throughput:** Millions of ops/sec
- ✅ **Simplest:** No external dependencies

**Cons:**

- ❌ **No Global Enforcement:** User can hit different servers
    - Example: User makes 10 requests to Server 1, 10 to Server 2 = 20 total (limit exceeded)
- ❌ **Lost on Restart:** Counters reset when server restarts
- ❌ **No Coordination:** Cannot sync across servers

**Why Not Chosen:** Fails "global enforcement" requirement.

#### Option D: Memcached (Not Chosen)

**Pros:**

- ✅ **Fast:** Sub-millisecond latency
- ✅ **Simple:** Easy to operate

**Cons:**

- ❌ **No Atomic INCR:** Race conditions possible
- ❌ **No Persistence:** Lost on restart
- ❌ **Limited Features:** No TTL per key

**Why Not Chosen:** Lack of atomic operations critical for rate limiting.

### Decision Made

**✅ Redis Cluster (10 nodes, sharded by user_id)**

**Rationale:**

1. **Latency:** <1ms meets requirement
2. **Throughput:** 100K ops/sec × 10 nodes = 1M ops/sec
3. **Atomic:** INCR prevents race conditions
4. **TTL:** Auto-expire keys (bounded storage)
5. **Production-Proven:** Industry standard for rate limiting

**Configuration:**

```
Cluster: 10 master nodes + 20 replicas (RF=3)
Memory: 2 GB per node (1 GB data + 1 GB overhead)
Persistence: AOF (append-only file) for durability
Eviction: No eviction (use TTL instead)
```

### Trade-offs Accepted

| Trade-off           | Impact                                | Mitigation                                                                 |
|---------------------|---------------------------------------|----------------------------------------------------------------------------|
| **Memory Cost**     | $500/TB (~$15K/year for 30TB cluster) | Acceptable for critical service                                            |
| **Durability Risk** | Async AOF can lose 1-2 seconds        | Acceptable (rate limit counters can tolerate minor loss)                   |
| **Ops Complexity**  | Requires Redis expertise              | Hire specialists, use managed services (AWS ElastiCache, Redis Enterprise) |

### When to Reconsider

**Switch to PostgreSQL if:**

- Latency requirement relaxes to >10ms
- Throughput drops below 10K ops/sec
- Strong ACID guarantees become critical

**Switch to Local Counters if:**

- Single-server deployment only
- Global enforcement not required

---

## 3. Consistency Model: Atomic INCR vs Distributed Locks

### The Problem

Prevent race conditions when multiple API gateways check rate limit for same user simultaneously.

**Scenario:**

```
Gateway 1 and Gateway 2 both receive requests for user_id=12345 at same time.
Both check: current_count = 99 (limit = 100)
Both allow: count increments to 100, then 101
Result: Limit exceeded! ❌
```

### Options Considered

| Approach                       | Latency  | Correctness       | Complexity | Cost   |
|--------------------------------|----------|-------------------|------------|--------|
| **Atomic INCR**                | ~0.5ms   | ✅ Perfect         | Low        | Low    |
| **Distributed Lock (Redlock)** | ~10-50ms | ✅ Perfect         | High       | High   |
| **Optimistic Locking**         | ~2ms     | ⚠️ Retry needed   | Medium     | Medium |
| **No Synchronization**         | ~0.5ms   | ❌ Race conditions | Low        | Low    |

#### Option A: Atomic INCR ✅ (Chosen)

**How It Works:**

```
Gateway 1: INCR user:12345:count → 100
Gateway 2: INCR user:12345:count → 101

Gateway 1: Check (100 <= 100) → Allow ✅
Gateway 2: Check (101 > 100) → Reject ✅
```

**Why It Works:**

- Redis single-threaded execution model
- Commands processed sequentially (never parallel)
- INCR is atomic operation (cannot interleave)

**Pros:**

- ✅ **Fast:** O(1) operation, <0.5ms
- ✅ **Simple:** No lock management
- ✅ **No Deadlocks:** No locks to deadlock
- ✅ **Correct:** Prevents all race conditions

**Cons:**

- ❌ **Slight Over-Counting:** May exceed limit by 1 (acceptable)

#### Option B: Distributed Lock - Redlock (Not Chosen)

**How It Works:**

```
1. Gateway 1: Acquire lock on user:12345
2. Gateway 1: Read count
3. Gateway 1: Check limit
4. Gateway 1: Increment if under limit
5. Gateway 1: Release lock

Gateway 2 waits for lock...
```

**Pros:**

- ✅ **Perfect Consistency:** No over-counting possible

**Cons:**

- ❌ **Slow:** 10-50ms per operation (vs 0.5ms for INCR)
- ❌ **Complex:** Requires managing locks, timeouts, deadlocks
- ❌ **Failure Modes:** Lock can't be released (timeout), deadlock
- ❌ **Throughput:** Serializes all requests for a user

**Why Not Chosen:** 50× slower than atomic INCR, no benefit for rate limiting use case.

#### Option C: Optimistic Locking (Not Chosen)

**How It Works:**

```
1. Read count + version
2. Check limit
3. INCR with version check (CAS - Compare and Swap)
4. If version changed: Retry
```

**Pros:**

- ✅ **No Locks:** Lock-free design
- ✅ **Correct:** Prevents race conditions

**Cons:**

- ❌ **Retry Overhead:** Under contention, many retries
- ❌ **Complexity:** Retry logic, exponential backoff
- ❌ **Latency Variance:** 1-5 retries = 1-5ms

**Why Not Chosen:** Unnecessary complexity. Atomic INCR simpler and faster.

### Decision Made

**✅ Redis Atomic INCR (Increment-Then-Check)**

**Rationale:**

1. **Fastest:** Single Redis operation, <0.5ms
2. **Simplest:** No lock management, no retry logic
3. **Correct:** Single-threaded Redis guarantees atomicity
4. **Scalable:** No bottlenecks, no contention

**Implementation:**

```
// Increment first (atomic)
new_count = INCR user:12345:count

// Then check
if new_count > limit:
    return 429  // Reject
else:
    return 200  // Allow
```

### Trade-offs Accepted

| Trade-off                | Impact                                  | Mitigation                                          |
|--------------------------|-----------------------------------------|-----------------------------------------------------|
| **Slight Over-Counting** | May exceed limit by 1 request           | Acceptable (1 extra request out of 100 is 1% error) |
| **No Perfect Accuracy**  | Not suitable for financial transactions | Use distributed locks for financial APIs if needed  |

### When to Reconsider

**Switch to Distributed Locks if:**

- Perfect accuracy required (financial transactions)
- Willing to accept 50× higher latency
- Single over-count unacceptable

**Switch to Optimistic Locking if:**

- Atomic operations unavailable
- Need retry logic anyway

---

## 4. Failure Strategy: Fail-Open vs Fail-Close

### The Problem

What happens when Redis cluster is unavailable?

**Scenario:**

- Redis cluster down for 10 minutes
- 500K API requests/sec
- Must decide: Block all or allow all?

### Options Considered

| Strategy       | API Availability | Quota Enforcement | Business Impact | Security           |
|----------------|------------------|-------------------|-----------------|--------------------|
| **Fail-Open**  | ✅ 100%           | ❌ Disabled        | ✅ Minimal       | ⚠️ Temporary abuse |
| **Fail-Close** | ❌ 0%             | ✅ Strict          | ❌ $1M+ loss     | ✅ No abuse         |
| **Hybrid**     | ⚠️ Degraded      | ⚠️ Partial        | ⚠️ Complex      | ⚠️ Partial abuse   |

#### Option A: Fail-Open ✅ (Chosen)

**Behavior:** Allow all requests (disable rate limiting) when Redis unavailable.

**Pros:**

- ✅ **High Availability:** API remains available
- ✅ **Business Continuity:** Revenue continues
- ✅ **Customer Satisfaction:** No service disruption
- ✅ **Industry Standard:** Used by Stripe, GitHub, AWS

**Cons:**

- ❌ **Temporary Abuse:** Malicious users can exceed limits
- ❌ **DDoS Risk:** Attackers can overwhelm system
- ❌ **Must Rely on Other Protections:** WAF, L7 firewall, DDoS mitigation

**Example:**

```
Redis down for 10 minutes
→ Rate limiter disabled
→ 500K QPS continues flowing
→ Some abusive users exceed limits
→ Revenue: $0 lost (vs $1M with fail-close)
→ Redis recovers, rate limiting resumes
```

#### Option B: Fail-Close (Not Chosen)

**Behavior:** Block all API requests when Redis unavailable.

**Pros:**

- ✅ **Strict Enforcement:** No quota leakage
- ✅ **Security:** No abuse possible

**Cons:**

- ❌ **Self-Inflicted DDoS:** Rate limiter becomes SPOF
- ❌ **Revenue Loss:** $100K/minute lost revenue
- ❌ **Customer Churn:** Users switch to competitors
- ❌ **SLA Breach:** Violates 99.99% uptime SLA

**Example:**

```
Redis down for 10 minutes
→ All API requests blocked
→ Revenue: $1M lost
→ Customers: Angry, filing support tickets
→ Business: Catastrophic impact
```

**Why Not Chosen:** Unacceptable business impact. Rate limiter must never be the reason API is down.

#### Option C: Hybrid - Degraded Mode (Not Chosen)

**Behavior:** Switch to less accurate local rate limiting when Redis down.

**Pros:**

- ✅ **Partial Availability:** API continues
- ✅ **Partial Enforcement:** Some protection

**Cons:**

- ❌ **Complex:** Two rate limiting implementations
- ❌ **Inaccurate:** Local counters not globally consistent
- ❌ **Testing Difficulty:** Hard to test degraded mode

**Why Not Chosen:** Complexity not justified. Fail-open simpler and more reliable.

### Decision Made

**✅ Fail-Open (Allow requests when Redis unavailable)**

**Rationale:**

1. **Business Continuity:** API availability > strict enforcement
2. **Industry Standard:** Stripe, AWS, GitHub all use fail-open
3. **Rare Failure:** Redis cluster highly available (99.99% uptime)
4. **Other Protections:** WAF, DDoS mitigation provide backup
5. **Quick Recovery:** Redis issues typically resolved in <15 minutes

**Implementation (Circuit Breaker):**

```
1. Monitor Redis error rate
2. If error rate > 50% for 10 seconds:
   - Open circuit (fail-open mode)
   - Allow all requests
3. After 60 seconds:
   - Test Redis with single request
   - If success: Close circuit (resume rate limiting)
```

### Trade-offs Accepted

| Trade-off           | Impact                                            | Mitigation                                                  |
|---------------------|---------------------------------------------------|-------------------------------------------------------------|
| **Temporary Abuse** | Malicious users can exceed limits for ~10 minutes | Monitor for abuse, ban IPs via WAF, rely on DDoS protection |
| **Revenue Risk**    | If abuse causes backend overload                  | Backend has separate capacity limits, auto-scaling          |
| **Security Risk**   | Attackers can brute-force during outage           | WAF rate limits, DDoS mitigation active                     |

### When to Reconsider

**Switch to Fail-Close if:**

- Government/defense system (security > availability)
- Financial API (strict quota enforcement critical)
- Willing to accept downtime

**Switch to Hybrid if:**

- Have resources to maintain two implementations
- Local rate limiting acceptable accuracy

---

## 5. Scaling Strategy: Sharding by User ID vs Global Counter

### The Problem

Redis single instance can handle ~100K ops/sec. We need 1M ops/sec.

**Options:**

- Global counter (all users share one counter)
- Shard by user_id (each user on specific shard)

### Options Considered

| Strategy                | Throughput     | Consistency       | Complexity | Scalability     |
|-------------------------|----------------|-------------------|------------|-----------------|
| **Sharding by user_id** | ✅ 1M+ ops/sec  | ✅ Per-user strong | Low        | ✅ Linear        |
| **Global counter**      | ❌ 100K ops/sec | ✅ Perfect         | Low        | ❌ Vertical only |
| **Random sharding**     | ✅ 1M+ ops/sec  | ❌ Weak            | Low        | ✅ Linear        |

#### Option A: Sharding by user_id ✅ (Chosen)

**How It Works:**

```
shard_id = hash(user_id) % num_shards

User 12345 → hash(12345) % 10 = 5 → Shard 5
User 67890 → hash(67890) % 10 = 0 → Shard 0
```

**Pros:**

- ✅ **High Throughput:** 100K/shard × 10 shards = 1M ops/sec
- ✅ **Strong Consistency:** All requests for a user hit same shard
- ✅ **Linear Scaling:** Add more shards as needed
- ✅ **Isolation:** Hot user on Shard 5 doesn't affect Shard 1

**Cons:**

- ❌ **No Cross-User Atomicity:** Cannot atomically update two users
- ❌ **Resharding Complexity:** Adding shards requires data migration

#### Option B: Global Counter (Not Chosen)

**How It Works:**

```
All users share single counter: api_global_count
```

**Pros:**

- ✅ **Perfect Consistency:** One source of truth
- ✅ **Simple:** No sharding logic

**Cons:**

- ❌ **Throughput Bottleneck:** Limited to 100K ops/sec
- ❌ **Cannot Scale Horizontally:** Single instance limit
- ❌ **Hot Key:** All requests hit one Redis key

**Why Not Chosen:** Cannot meet 1M ops/sec requirement.

### Decision Made

**✅ Sharding by user_id with 10 Redis master nodes**

**Rationale:**

1. **Throughput:** 10 nodes × 100K ops/sec = 1M ops/sec
2. **Consistency:** Per-user consistency sufficient (don't need global)
3. **Scalability:** Add more nodes as traffic grows

**Configuration:**

```
Nodes: 10 masters + 20 replicas (RF=3)
Sharding: Consistent hashing by user_id
Load: ~100K ops/sec per node
```

### Trade-offs Accepted

| Trade-off            | Impact                                                | Mitigation                                    |
|----------------------|-------------------------------------------------------|-----------------------------------------------|
| **No Global Limits** | Cannot enforce "100K requests total across all users" | Not a requirement for rate limiting           |
| **Uneven Load**      | Some users more active than others                    | Consistent hashing provides good distribution |

### When to Reconsider

**Switch to Global Counter if:**

- Need to enforce global limits (not per-user)
- Throughput drops below 100K ops/sec

---

## 6. Hot Key Mitigation: L1 Cache vs Replication vs Ignore

### The Problem

Malicious IP makes 100K requests/sec, overwhelming single Redis shard.

### Options Considered

| Strategy                   | Load Reduction | Accuracy | Complexity | Cost   |
|----------------------------|----------------|----------|------------|--------|
| **L1 Cache (Local)**       | ✅ 99%          | ⚠️ ~90%  | Medium     | Low    |
| **Hot Key Replication**    | ✅ 70%          | ✅ 100%   | High       | Medium |
| **Ignore (No Mitigation)** | ❌ 0%           | ✅ 100%   | Low        | Low    |

#### Option A: L1 Cache ✅ (Chosen)

**How It Works:**

```
1. Gateway maintains local in-memory cache
2. Check L1 cache first (microsecond latency)
3. If under limit locally, allow
4. Batch updates to Redis every 100ms
```

**Pros:**

- ✅ **Massive Load Reduction:** 99% fewer Redis ops
- ✅ **Sub-Millisecond Latency:** No network call
- ✅ **Simple:** Just add local cache

**Cons:**

- ❌ **Slightly Less Accurate:** ~10% quota overage possible
- ❌ **Cache Invalidation:** Must sync across gateways

#### Option B: Hot Key Replication (Not Chosen for Default)

**How It Works:**

```
1. Detect hot key (>10K QPS)
2. Replicate to 3 Redis nodes
3. Load balance reads across replicas
```

**Pros:**

- ✅ **Load Distribution:** Spreads across nodes
- ✅ **Perfect Accuracy:** No overage

**Cons:**

- ❌ **Complex:** Replication coordination
- ❌ **Higher Memory:** 3× storage for hot keys

### Decision Made

**✅ L1 Cache (with hot key detection trigger)**

**Rationale:**

- Default: No L1 cache (simplest)
- When hot key detected: Enable L1 cache automatically
- 99% load reduction with <10% accuracy loss acceptable

---

## 7. Configuration Management: etcd vs Database vs Hardcoded

### The Problem

Store and update rate limit rules dynamically (no restarts).

### Decision Made

**✅ etcd (distributed key-value store)**

**Rationale:**

- Watch API for instant updates
- Distributed consensus (consistent)
- Fast (<10ms reads)
- Used by Kubernetes (proven)

**Alternatives:**

- Database: Too slow, no watch API
- Hardcoded: Requires restart, no flexibility

---

## Summary Table: All Decisions

| Component       | Choice           | Alternative      | Key Reason                        |
|-----------------|------------------|------------------|-----------------------------------|
| **Algorithm**   | Token Bucket     | Fixed Window     | Allows bursts, no boundary issue  |
| **Storage**     | Redis            | PostgreSQL       | Sub-ms latency, 1M ops/sec        |
| **Consistency** | Atomic INCR      | Distributed Lock | 50× faster, simpler               |
| **Failure**     | Fail-Open        | Fail-Close       | Availability > strict enforcement |
| **Scaling**     | Shard by user_id | Global counter   | Linear scaling to 1M+ ops/sec     |
| **Hot Key**     | L1 Cache         | Ignore           | 99% load reduction                |
| **Config**      | etcd             | Database         | Watch API, instant updates        |

---

**All design decisions prioritize:**

1. **Availability:** System must remain available (fail-open)
2. **Latency:** Sub-millisecond rate limit checks
3. **Scalability:** Linear scaling to millions of QPS
4. **Simplicity:** Avoid unnecessary complexity (no locks)
