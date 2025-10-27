# 3.2.4 Design a Global Rate Limiter

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## Problem Statement

Design a globally distributed rate limiter service that enforces request limits across multiple API gateway nodes,
preventing users or IPs from exceeding their allocated quota (e.g., 10 requests/second) while maintaining
sub-millisecond latency and handling 500,000 QPS.

---

## Requirements and Scale Estimation

### Functional Requirements

- **Global Enforcement:** Limit user/IP to N requests per window across all servers
- **Different Tiers:** Support varying limits (basic vs paid users)
- **HTTP 429 Response:** Reject excess requests with proper status code
- **Rate Limit Headers:** Return limit info in response headers

### Non-Functional Requirements

| Requirement            | Target       | Rationale                         |
|------------------------|--------------|-----------------------------------|
| **Low Latency**        | < 1 ms (p99) | Minimal overhead on API requests  |
| **High Throughput**    | 500K QPS     | Support entire API fleet          |
| **High Availability**  | 99.99%       | Failure blocks all traffic (SPOF) |
| **Strong Consistency** | No drift     | Prevent quota leakage             |

### Scale Estimation

| Metric           | Value                |
|------------------|----------------------|
| API Throughput   | 500,000 QPS          |
| Active Users     | 1 Million            |
| Storage          | ~1 GB (counter data) |
| Redis Operations | ~1 Million ops/sec   |

---

## High-Level Architecture

> 📊 **See detailed architecture diagrams:** [HLD Diagrams](./hld-diagram.md)

### Key Components

| Component               | Responsibility                   | Technology                   |
|-------------------------|----------------------------------|------------------------------|
| **API Gateway**         | Routes requests, enforces limits | NGINX, Kong, AWS API Gateway |
| **Rate Limiter Module** | Implements algorithms            | Lua script, Go service       |
| **Redis Cluster**       | Distributed counter store        | Redis Cluster, KeyDB         |
| **Config Service**      | Rate limit rules                 | etcd, Consul, DynamoDB       |

---

## Rate Limiting Algorithms

### 1. Token Bucket ✅

**How:** Bucket fills with tokens at constant rate, each request consumes token.

**Pros:** ✅ Allows bursts, ✅ Smooth limiting
**Cons:** ❌ More storage

**Use For:** APIs with bursty traffic (video, file uploads)

*See [pseudocode.md::token_bucket_check()](pseudocode.md)*

### 2. Sliding Window Counter ✅

**How:** Weighted sum of current + previous window.

**Pros:** ✅ Accurate, ✅ Prevents boundary bursts
**Cons:** ❌ Approximation, ❌ 2 counter reads

**Use For:** Strict rate limiting (financial APIs)

*See [pseudocode.md::sliding_window_check()](pseudocode.md)*

### 3. Fixed Window Counter

**How:** Count requests in fixed time buckets.

**Pros:** ✅ Simplest, ✅ Fastest
**Cons:** ❌ Boundary burst problem (2× requests)

**Use For:** Coarse limits only

---

## Data Model (Redis)

**Token Bucket:**

```
user:12345:tokens → 5
user:12345:last_refill → 1704067200
```

**Sliding Window:**

```
user:12345:window:1704067200 → 45
user:12345:window:1704067199 → 52
```

**TTL:** Auto-expire keys (2× window size)

---

## Key Design Decisions

### Decision 1: Redis vs RDBMS

**Chosen:** Redis (in-memory, atomic ops)

**Why Not RDBMS:**

- ❌ Too slow (10-50ms vs <1ms)
- ❌ Cannot handle 1M ops/sec
- ❌ Lock contention

*See [this-over-that.md](this-over-that.md) for detailed analysis*

### Decision 2: Fail-Open vs Fail-Close

**Chosen:** Fail-Open (allow requests if Redis down)

**Why:**

- ✅ API remains available
- ✅ High availability prioritized
- ❌ Trade-off: Temporary abuse possible

### Decision 3: Atomic INCR vs Locks

**Chosen:** Atomic INCR (no distributed locks)

**Why:**

- ✅ Redis single-threaded = atomic
- ✅ No lock overhead
- ✅ Sub-millisecond latency

---

## Global Synchronization

### Challenge: Race Conditions

Two gateways check limit simultaneously → both allow → quota exceeded

**Solution:** Redis atomic INCR

```
Gateway 1: INCR → 10
Gateway 2: INCR → 11
Gateway 2: Check (11 > 10) → Reject ✅
```

**Why Atomic:**

- ✅ Single-threaded Redis
- ✅ O(1) operation
- ✅ No locks needed

---

## Distributed Scaling

### Sharding by User ID

```
shard_id = hash(user_id) % num_shards
```

**Benefits:**

- ✅ Even load distribution
- ✅ All user requests hit same shard
- ✅ Strong consistency per user

**Cluster Size:**

- Redis: ~100K ops/sec per node
- Required: 1M ops/sec
- **Nodes: 10 Redis nodes**

---

## Bottlenecks and Solutions

### Bottleneck 1: Hot Keys

**Problem:** Malicious IP makes 100K requests/sec → one Redis node overloaded

**Solution 1:** Local L1 cache on gateway (reduces Redis load 90%)
**Solution 2:** Hot key replication (distribute across nodes)

### Bottleneck 2: Network Latency

**Problem:** Even 1ms adds up at 500K QPS

**Solution 1:** Co-locate Redis in same AZ as gateway
**Solution 2:** Connection pooling (reuse connections)

### Bottleneck 3: Storage Growth

**Problem:** Counter keys accumulate

**Solution:** TTL on Redis keys (auto-expire)

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Using RDBMS

Storing counters in PostgreSQL.

**Why Bad:** 10-50ms latency, cannot handle 1M ops/sec

**Solution:** Use Redis with atomic INCR

### ❌ Anti-Pattern 2: Check-Then-Increment

Read count, check locally, then increment (race condition).

**Solution:** Increment-then-check (atomic)

### ❌ Anti-Pattern 3: No Timeout

Blocking Redis call without timeout.

**Solution:** 5ms timeout + circuit breaker

### ❌ Anti-Pattern 4: Distributed Locks

Using Redlock for consistency (adds 10-50ms).

**Solution:** Redis atomic ops (no locks needed)

*See [pseudocode.md](pseudocode.md) for implementations*

---

## Alternative Approaches

### A. Application-Level (No Redis)

**Pros:** Fastest, simplest
**Cons:** ❌ No global enforcement (user hits different servers)

**Why Not:** Fails global requirement

### B. Database-Based

**Pros:** ACID, familiar
**Cons:** ❌ Too slow (10-50ms), cannot scale

**Why Not:** Latency/throughput requirements

### C. CDN-Based

**Pros:** Managed, global edge
**Cons:** ❌ Limited customization, expensive

**When to Use:** Public APIs with simple limits

---

## Monitoring

**Critical Metrics:**

- Rate limit latency (P99): < 1ms
- Redis operations: 1M ops/sec
- Rejection rate: < 5%
- Circuit breaker state: Closed

**Alerts:**

- Latency > 5ms → Scale Redis
- Error rate > 5% → Redis health check
- Hot key detected → Enable mitigation

---

## Trade-offs Summary

### What We Gained

✅ Global enforcement across all servers
✅ Sub-millisecond latency (<1ms)
✅ High throughput (500K QPS)
✅ Strong consistency (atomic ops)
✅ Horizontal scaling

### What We Sacrificed

❌ Fail-open allows temporary abuse
❌ Redis ops complexity
❌ Infrastructure cost (10+ nodes)
❌ L1 cache (if used) allows ~10% overage
❌ Redis SPOF (must be HA)

---

## Real-World Examples

**Stripe:** Token Bucket, Redis, fail-open
**GitHub:** Fixed window (hourly), tiered limits
**Twitter:** Sliding window (15-min), per-endpoint limits

---

## References

- [Rate Limiting Algorithms](../../02-components/2.5.1-rate-limiting-algorithms.md)
- [Redis Deep Dive](../../02-components/2.2.1-caching-deep-dive.md)
- [Consistent Hashing](../../02-components/2.2.2-consistent-hashing.md)
- [API Gateway](../../01-principles/1.2.3-api-gateway-servicemesh.md)

---

For complete details, algorithms, and trade-off analysis, see the *
*[Full Design Document](3.2.4-design-global-rate-limiter.md)**.

