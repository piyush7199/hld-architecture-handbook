# Global Rate Limiter - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A globally distributed rate limiter service that enforces request limits across multiple API gateway nodes. The system
prevents users or IP addresses from exceeding their allocated quota (e.g., 10 requests per second) while maintaining
sub-millisecond latency and handling 500,000 QPS across the entire API fleet. The solution must be strongly consistent
to prevent quota leakage while remaining highly available to avoid blocking legitimate traffic.

**Key Characteristics:**

- **Global enforcement** - Rate limits enforced across all API servers globally
- **Sub-millisecond latency** - Rate limit checks must add minimal overhead (<1ms P99)
- **High throughput** - Handle 500K QPS across entire API fleet
- **Strong consistency** - Atomic operations prevent quota leakage via race conditions
- **High availability** - Fail-open policy ensures API remains available even if rate limiter fails
- **Horizontal scalability** - Add Redis shards as traffic grows

---

## Requirements & Scale

### Functional Requirements

| Requirement             | Description                                                                              | Priority     |
|-------------------------|------------------------------------------------------------------------------------------|--------------|
| **Global Enforcement**  | Limit a user/IP to N requests per time window (e.g., 10 QPS) across all servers globally | Must Have    |
| **Different Limits**    | Support different limits based on user tier (basic vs paid subscriber)                   | Must Have    |
| **Response**            | Reject requests exceeding limit with HTTP 429 Too Many Requests                          | Must Have    |
| **Multiple Algorithms** | Support Token Bucket, Sliding Window Counter algorithms                                  | Should Have  |
| **Rate Limit Headers**  | Return headers showing limit, remaining, reset time                                      | Should Have  |
| **Analytics**           | Track rate limit violations for abuse detection                                          | Nice to Have |

### Non-Functional Requirements

| Requirement                | Target           | Rationale                                                |
|----------------------------|------------------|----------------------------------------------------------|
| **Low Latency**            | < 1 ms (p99)     | Checking limit must add minimal overhead to API requests |
| **High Throughput**        | Handle 500K QPS  | Must support entire API fleet's traffic                  |
| **High Availability**      | 99.99% uptime    | Rate limiter failure blocks all traffic (SPOF)           |
| **Strong Consistency**     | No counter drift | Prevent users from exceeding limits via race conditions  |
| **Horizontal Scalability** | Linear scaling   | Add nodes as traffic grows                               |

### Scale Estimation

| Metric                  | Assumption                           | Calculation  | Result                |
|-------------------------|--------------------------------------|--------------|-----------------------|
| **Peak API Throughput** | 500,000 requests per second          | -            | 500K QPS              |
| **Active Users**        | 1 Million users with rate limits     | -            | 1M users              |
| **Data Storage**        | 1M active users × 1 KB per counter   | 1M × 1 KB    | ~1 GB of counter data |
| **Redis Operations**    | Every API request = 1 Read + 1 Write | 500K × 2     | ~1 Million ops/sec    |
| **Network Bandwidth**   | 500K requests × 100 bytes            | 500K × 100 B | ~50 MB/sec            |

**Key Insight:** This is an extremely read+write heavy workload requiring sub-millisecond operations. Only in-memory
stores (Redis) can handle this scale.

---

## Key Components

| Component               | Responsibility                                | Technology Options           | Scalability            |
|-------------------------|-----------------------------------------------|------------------------------|------------------------|
| **API Gateway**         | Routes requests, enforces rate limits         | NGINX, Kong, AWS API Gateway | Horizontal (stateless) |
| **Rate Limiter Module** | Implements rate limiting algorithms           | Lua script, Go service       | Horizontal             |
| **Redis Cluster**       | Distributed counter store (atomic operations) | Redis Cluster, KeyDB         | Horizontal (sharding)  |
| **Config Service**      | Stores rate limit rules, user tier mappings   | etcd, Consul, DynamoDB       | Horizontal             |
| **Monitoring**          | Tracks violations, latency, throughput        | Prometheus, Grafana          | Horizontal             |

---

## Rate Limiting Algorithms

### Algorithm 1: Token Bucket

**How It Works:**

- Each user has a bucket with a maximum capacity of tokens
- Tokens refill at a constant rate (e.g., 10 tokens/second)
- Each request consumes 1 token
- If no tokens available, request is rejected

**Advantages:**

- ✅ Allows bursts (client can use accumulated tokens)
- ✅ Smooth rate limiting over time
- ✅ Simple to implement

**Disadvantages:**

- ❌ Requires storing: token count + last refill timestamp
- ❌ Slightly more complex than fixed window

**Use Case:** APIs where bursty traffic is acceptable (e.g., video streaming).

### Algorithm 2: Sliding Window Counter

**How It Works:**

- Divide time into fixed windows (e.g., 1-second windows)
- Count requests in current window and previous window
- Approximate sliding window using weighted sum

**Formula:**

```
rate = (prev_window_count × overlap_percent) + current_window_count
```

**Advantages:**

- ✅ More accurate than fixed window (no boundary bursts)
- ✅ Less storage than sliding log (only 2 counters)
- ✅ Good balance of accuracy and efficiency

**Disadvantages:**

- ❌ Approximation (not 100% accurate)
- ❌ Requires 2 counter reads (current + previous)

**Use Case:** Most general-purpose APIs.

### Algorithm 3: Fixed Window Counter

**How It Works:**

- Divide time into fixed windows (e.g., 1-minute buckets)
- Count requests in current window
- Reset counter when window expires

**Advantages:**

- ✅ Simplest algorithm
- ✅ Minimal storage (single counter)
- ✅ Fastest (single INCR operation)

**Disadvantages:**

- ❌ **Boundary Burst Problem:** User can make 2× requests at window boundary
    - Example: 100 requests at 00:59, 100 requests at 01:00 = 200 requests in 1 second
- ❌ Not suitable for strict rate limiting

**Use Case:** Coarse-grained limits where boundary bursts are acceptable.

### Algorithm Comparison

| Criteria       | Token Bucket                  | Sliding Window                  | Fixed Window          |
|----------------|-------------------------------|---------------------------------|-----------------------|
| **Accuracy**   | High (allows bursts)          | High (approximates sliding log) | Low (boundary bursts) |
| **Storage**    | 2 values (tokens + timestamp) | 2 counters (current + prev)     | 1 counter             |
| **Complexity** | Medium                        | Medium                          | Low                   |
| **Use Case**   | Bursty traffic                | Strict limits                   | Coarse limits         |

**Decision:** Use **Token Bucket** for APIs with bursty workloads (video, file uploads), **Sliding Window** for strict
enforcement (financial APIs).

---

## Data Model (Redis)

**Key Format:**

```
rate_limit:{user_id}:{window} → counter
```

**Token Bucket Schema:**

```
user:12345:tokens → 5          // Current token count
user:12345:last_refill → 1704067200  // Last refill timestamp (Unix)
```

**Sliding Window Schema:**

```
user:12345:window:1704067200 → 45   // Request count in current 1-second window
user:12345:window:1704067199 → 52   // Request count in previous window
```

**TTL:** Set TTL on keys to auto-expire (e.g., 2× window size).

---

## Global Synchronization and Consistency

The Rate Limiter must be consistent across all API Gateway nodes globally.

### Challenge: Race Conditions

**Problem:**
Two API Gateway nodes check the limit simultaneously for the same user:

```
Gateway 1: READ counter = 9
Gateway 2: READ counter = 9
Gateway 1: Allow (9 < 10), INCR → 10
Gateway 2: Allow (9 < 10), INCR → 11  ❌ User exceeded limit!
```

**Solution: Atomic Operations**

Use Redis **INCR** (atomic increment) to prevent race conditions:

```
Gateway 1: INCR → 10
Gateway 2: INCR → 11
Gateway 2: Check (11 > 10) → Reject ✅
```

**Why Redis INCR?**

- ✅ Atomic: Single-threaded execution model
- ✅ Fast: O(1) operation, sub-millisecond
- ✅ Simple: No distributed locks needed

---

## Distributed Scaling (Sharding)

The 1 Million ops/sec workload is too high for a single Redis instance.

**Sharding Strategy: Hash by User ID**

```
shard_id = hash(user_id) % num_shards
```

**Benefits:**

- ✅ All requests for a user hit the same Redis node (consistency)
- ✅ Load distributed evenly across shards
- ✅ Horizontal scaling (add more shards)

**Example:**

```
User 12345 → hash(12345) % 10 = 5 → Shard 5
User 67890 → hash(67890) % 10 = 0 → Shard 0
```

**Cluster Size Estimation:**

- Redis: ~100K ops/sec per node
- Required: 1M ops/sec
- Nodes needed: 1M / 100K = **10 Redis nodes**

---

## Handling Failure: Fail-Open vs Fail-Close

**Critical Decision:** What happens if Redis cluster fails?

### Option A: Fail-Close ❌

**Behavior:** Block all API requests

**Pros:**

- ✅ Strict enforcement (no quota leakage)

**Cons:**

- ❌ **Self-inflicted DDoS:** Rate limiter becomes SPOF
- ❌ Entire API unavailable during Redis outage
- ❌ Catastrophic for business

### Option B: Fail-Open ✅ (Recommended)

**Behavior:** Allow all requests (disable rate limiting)

**Pros:**

- ✅ API remains available
- ✅ High availability prioritized

**Cons:**

- ❌ Temporary abuse possible during outage
- ❌ Must rely on other protections (WAF, DDoS mitigation)

**Decision:** **Fail-Open** is standard for production. Rate limiter should never be the reason your API is down.

**Implementation:**
Add circuit breaker around Redis calls. If Redis error rate exceeds threshold (e.g., 50% errors for 10 seconds), open
circuit and allow all requests. Periodically attempt to close circuit (half-open state) to check if Redis recovered.

---

## Bottlenecks & Solutions

### Bottleneck 1: Redis Contention (Hot Keys)

**Problem:** A single "hot" key (e.g., malicious IP launching script) can overload the Redis node holding its counter.

**Solutions:**

1. **Local L1 Cache (Per-Gateway):**
    - Cache rate limit status for hot keys locally (5-second TTL)
    - Reduces Redis load by 80-90% for hot keys
    - Trade-off: ~10% quota overage possible (acceptable)

2. **Hot Key Detection:**
    - Monitor Redis key access patterns
    - Automatically enable L1 cache for keys exceeding threshold (e.g., >1K QPS)

3. **IP Blocking:**
    - If hot key is malicious IP, block at firewall/WAF level
    - Remove from rate limiter entirely

### Bottleneck 2: Redis Latency at Scale

**Problem:** As traffic grows, Redis roundtrip latency increases (network congestion, CPU saturation).

**Solutions:**

1. **Connection Pooling:**
    - Reuse Redis connections (avoid TCP handshake overhead)
    - Pool size: 10-20 connections per gateway node

2. **Pipeline Batching:**
    - Batch multiple INCR operations in single pipeline
    - Reduces network round-trips by 5-10x

3. **Local L1 Cache (Aggressive):**
    - Cache all rate limit checks for 100ms
    - Reduces Redis load by 90%+
    - Trade-off: Higher quota overage (~20%)

### Bottleneck 3: Config Service Latency

**Problem:** Fetching rate limit rules (user tier, limits) from Config Service adds latency.

**Solutions:**

1. **Local Cache:**
    - Cache user tier mappings in gateway memory (5-minute TTL)
    - Reduces Config Service calls by 99%

2. **Push Updates:**
    - Config Service pushes rule updates to gateways (WebSocket/pub-sub)
    - Avoids polling overhead

---

## Common Anti-Patterns to Avoid

### 1. Using RDBMS for Counters

❌ **Bad:** Storing counters in PostgreSQL

- Disk I/O: 10-50ms latency (vs <1ms for Redis)
- Row-level locks: Contention at high QPS
- Write amplification: WAL logs, replication overhead
- Cannot handle 1M ops/sec

✅ **Good:** Use Redis with atomic INCR

- In-memory operations provide sub-millisecond latency
- Single-threaded execution model eliminates locks

### 2. Check-Then-Increment (Race Condition)

❌ **Bad:** Non-atomic check

- Read current count from Redis, check if under limit locally, then increment
- Creates race condition where two requests can both see count = 9, both pass the check, and both increment to 10 and 11

✅ **Good:** Increment-Then-Check

- Increment counter first atomically (INCR), then check if result exceeds limit
- If over limit, reject request
- Slightly over-counts but guarantees atomicity

### 3. No Timeout on Redis Calls

❌ **Bad:** Blocking Redis call

- Call Redis without timeout
- If Redis is slow or unresponsive, API Gateway thread blocks indefinitely
- All gateway threads eventually blocked, entire API unavailable

✅ **Good:** Set Timeout + Circuit Breaker

- Set aggressive timeout on Redis calls (e.g., 5ms)
- If timeout exceeded, fail open (allow request)
- Implement circuit breaker to stop calling Redis if error rate too high

### 4. Global Lock for Consistency

❌ **Bad:** Using distributed lock

- Acquire distributed lock (e.g., Redlock) before checking/updating counter
- Ensures strict consistency but adds 10-50ms latency
- At 500K QPS, lock contention causes massive queuing

✅ **Good:** Redis Atomic Operations (No Locks)

- Redis single-threaded execution model provides atomicity without locks
- INCR, INCRBY, HINCRBY are all atomic
- Use these instead of explicit locking

---

## Monitoring & Observability

### Key Metrics

| Metric                       | Target           | Alert Threshold | Description                                |
|------------------------------|------------------|-----------------|--------------------------------------------|
| **Rate Limit Latency (P99)** | < 1 ms           | > 5 ms          | Time to check rate limit (Redis roundtrip) |
| **Redis Operations/sec**     | 1M ops/sec       | < 500K ops/sec  | Total throughput to Redis cluster          |
| **Rejection Rate**           | < 5%             | > 20%           | Percentage of requests rejected            |
| **Redis Error Rate**         | < 0.1%           | > 5%            | Failed Redis operations                    |
| **Circuit Breaker State**    | Closed           | Open for > 60s  | Is circuit breaker open (fail-open mode)?  |
| **Hot Key Detection**        | 0 keys > 10K QPS | > 5 hot keys    | Keys with abnormally high traffic          |

### Dashboards

**Dashboard 1: Rate Limiter Health**

- Rate limit check latency (P50, P99, P999)
- Redis cluster CPU/memory/connections
- Rejection rate by user tier
- Hot key leaderboard (top 10 keys by QPS)

**Dashboard 2: Abuse Detection**

- Top rejected users/IPs
- Rejection rate trends (hourly/daily)
- Geographic distribution of rejections

**Critical Alerts:**

- Rate limit latency > 5ms for > 1 minute → Scale Redis or add local cache
- Redis error rate > 5% → Check Redis cluster health, may trigger fail-open
- Hot key detected (> 10K QPS) → Enable hot key mitigation
- Circuit breaker open for > 5 minutes → Redis cluster unavailable

---

## Trade-offs Summary

| Decision               | Choice                        | Alternative      | Why Chosen                          | Trade-off                     |
|------------------------|-------------------------------|------------------|-------------------------------------|-------------------------------|
| **Storage**            | Redis Cluster                 | PostgreSQL       | Sub-millisecond latency, 1M ops/sec | Higher cost, in-memory only   |
| **Algorithm**          | Token Bucket / Sliding Window | Fixed Window     | No boundary bursts, accurate        | More complex, 2 values vs 1   |
| **Consistency**        | Atomic INCR                   | Distributed Lock | Fast (<1ms), no lock contention     | Slight over-counting possible |
| **Failure Mode**       | Fail-Open                     | Fail-Close       | API availability prioritized        | Temporary abuse during outage |
| **Sharding**           | Hash by User ID               | Round-robin      | User consistency, even load         | Resharding complexity         |
| **Hot Key Mitigation** | Local L1 Cache                | No mitigation    | Reduces Redis load 80-90%           | ~10% quota overage            |

### What We Gained

✅ **Global Enforcement:** Strict rate limits enforced across all API servers
✅ **Low Latency:** Sub-millisecond rate limit checks (<1ms P99)
✅ **High Throughput:** Handles 500K QPS with Redis cluster
✅ **Strong Consistency:** Atomic operations prevent quota leakage
✅ **Horizontal Scaling:** Add Redis shards as traffic grows

### What We Sacrificed

❌ **Eventual Consistency Risk:** Fail-open policy allows temporary abuse during Redis outage
❌ **Complexity:** Requires Redis cluster management (ops overhead)
❌ **Cost:** Redis cluster with 10+ nodes is expensive
❌ **Accuracy:** Local L1 cache (if used) can allow ~10% quota overage
❌ **Single Point of Failure:** Redis cluster health critical (must be highly available)

---

## Real-World Implementations

### Stripe API Rate Limiting

- **Algorithm:** Token Bucket (allows bursts for better UX)
- **Storage:** Redis cluster with automatic sharding
- **Fail Strategy:** Fail-open (availability over strict enforcement)
- **Headers:** Returns `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- **Key Insight:** Stripe uses generous rate limits to avoid customer frustration

### GitHub API Rate Limiting

- **Algorithm:** Fixed window (hourly buckets)
- **Tiers:** Unauthenticated (60/hour), Authenticated (5,000/hour)
- **Storage:** Redis with fallback to local counters
- **Headers:** Comprehensive rate limit info in every response
- **Key Insight:** Different limits for different auth methods (token vs OAuth)

### Twitter API Rate Limiting

- **Algorithm:** Sliding window (15-minute buckets)
- **Tiers:** Complex per-endpoint limits (varies by API endpoint)
- **Storage:** Internal distributed counter system
- **Key Insight:** Rate limits are per-user per-endpoint (e.g., 900 tweets/15min for timeline API)

---

## Key Takeaways

1. **Redis is essential** for sub-millisecond rate limiting at scale (1M ops/sec)
2. **Atomic operations (INCR)** prevent race conditions without distributed locks
3. **Fail-open policy** ensures API availability even if rate limiter fails
4. **Token Bucket or Sliding Window** are preferred over Fixed Window (no boundary bursts)
5. **Sharding by user ID** ensures consistency and even load distribution
6. **Local L1 cache** mitigates hot key problems (80-90% Redis load reduction)
7. **Circuit breaker** prevents cascading failures during Redis outages
8. **Monitoring** is critical (latency, hot keys, rejection rates, circuit breaker state)
9. **Different algorithms** for different use cases (Token Bucket for bursts, Sliding Window for strict limits)
10. **Horizontal scaling** via Redis sharding as traffic grows

---

## Recommended Stack

- **API Gateway:** NGINX, Kong, or AWS API Gateway
- **Rate Limiter:** Embedded Lua script or Go service
- **Storage:** Redis Cluster (10+ nodes for 1M ops/sec)
- **Config Service:** etcd, Consul, or DynamoDB
- **Monitoring:** Prometheus + Grafana
- **Circuit Breaker:** Custom implementation or Hystrix

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, component design, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (rate limit check, failure scenarios)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (Token Bucket, Sliding Window, atomic operations)

