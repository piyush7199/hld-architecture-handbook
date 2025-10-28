# E-commerce Flash Sale System (High-Contention Inventory)

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

Design a flash sale system that handles extreme traffic spikes when selling limited-quantity products (e.g., 100 limited-edition iPhones). The system must prevent overselling, handle 100K+ concurrent purchase attempts per second, maintain strong consistency for inventory counts, and process payments atomically while ensuring fairness.

**Core Challenge:** Managing a single "hot key" (inventory counter) receiving 100,000 writes per second when only 100 items are available (0.1% success rate), while maintaining ACID properties and preventing double-charging.

---

## Requirements and Scale Estimation

### Functional Requirements

| Requirement | Description | Priority |
|-------------|-------------|----------|
| **Inventory Protection** | Prevent overselling (exactly 100 items sold) | Must Have |
| **Reservation System** | Temporary item reservation before payment | Must Have |
| **Payment Processing** | Complete payment within 10-minute window | Must Have |
| **Automatic Refund** | Cancel and refund if payment fails/times out | Must Have |
| **Fair Queue** | First-come-first-served (FCFS) ordering | Should Have |
| **Real-Time Stock Updates** | Show remaining inventory to users | Should Have |

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Strong Consistency** | 100% accurate inventory | Overselling causes legal issues |
| **High Throughput** | 100K writes/sec | Peak load during flash sale start |
| **Low Latency** | < 50ms (p99) | Users abandon if slow |
| **Fault Tolerance** | Auto-rollback on failure | No partial transactions |
| **Availability** | 99.9% uptime | Downtime = lost revenue |

### Scale Estimation

| Metric | Assumption | Result |
|--------|-----------|--------|
| **Concurrent Users** | 1 Million users waiting | 1M users |
| **Peak Purchase Attempts** | 100,000 clicks per second | 100K QPS |
| **Available Inventory** | 100 limited-edition items | 100 items |
| **Success Rate** | 100 / 100,000 | 0.1% |
| **Rejection Rate** | 99,900 / 100,000 | 99.9% |
| **Critical Section Ops** | Check + decrement atomic | 200K ops/sec on 1 key |

---

## High-Level Architecture

```
           ┌────────────────────────────────────┐
           │      User (1M concurrent)          │
           └─────────────┬──────────────────────┘
                         │ 100K QPS spike
           ┌─────────────▼──────────────────────┐
           │     API Gateway + Rate Limiter     │
           │   - Token Bucket (load shedding)   │
           │   - Bot detection (CAPTCHA)        │
           └─────────────┬──────────────────────┘
                         │ 10K QPS (90% rejected)
           ┌─────────────▼──────────────────────┐
           │   Inventory Reservation Service    │
           │  - Atomic Redis DECR               │
           │  - Idempotency check               │
           └─────┬────────────────┬──────────────┘
                 │                │
  ┌──────────────▼──┐      ┌──────▼──────────────┐
  │ Redis Cluster   │      │ Reservation DB      │
  │ Hot Key Counter │      │ (PostgreSQL)        │
  │ iphone:stock    │      │ TTL: 10 minutes     │
  └──────────────┬──┘      └──────┬──────────────┘
                 │                │
           ┌─────▼────────────────▼──────────────┐
           │     Kafka Message Queue             │
           │   Topic: inventory-reserved         │
           └─────┬───────────────────────────────┘
                 │ Async payment processing
           ┌─────▼───────────────────────────────┐
           │  Payment Saga Orchestrator          │
           │  - Step 1: Charge (Stripe)          │
           │  - Step 2: Create order             │
           │  - Compensate: Refund + Release     │
           └─────────────────────────────────────┘
```

**Components:**
1. **API Gateway + Rate Limiter:** Multi-layer rate limiting (CDN edge → API Gateway → Service)
2. **Inventory Reservation Service:** Core service managing atomic inventory operations
3. **Redis Cluster:** In-memory hot key for inventory counter (atomic DECR)
4. **Reservation DB:** Tracks temporary reservations with 10-minute TTL
5. **Kafka:** Decouples reservation from slow payment processing
6. **Payment Saga Orchestrator:** Manages distributed transaction with compensation
7. **Payment Service:** Stripe API integration with idempotency keys

---

## Detailed Component Design

### Inventory Management (Redis Atomic DECR)

**Challenge:** 100K concurrent writes to single Redis key (iphone:stock).

**Solution:** Use Redis atomic DECR operation with Lua script.

**Why Redis over Database Row Lock?**

| Factor | Redis DECR | PostgreSQL Row Lock |
|--------|------------|---------------------|
| **Throughput** | ✅ 100K ops/sec per key | ❌ 1K ops/sec (lock contention) |
| **Latency** | ✅ <1ms (in-memory) | ❌ 10-50ms (disk I/O) |
| **Atomicity** | ✅ Native atomic operations | ✅ ACID compliant |
| **Durability** | ⚠️ RDB snapshots (1-5s risk) | ✅ WAL (zero data loss) |

**Lua Script for Atomic Check-and-Decrement:**

```lua
-- Atomic check and decrement
local stock_key = KEYS[1]
local current_stock = tonumber(redis.call('GET', stock_key))

if current_stock > 0 then
  redis.call('DECR', stock_key)
  return 1  -- Success
else
  return 0  -- Sold out
end
```

**Benefits:**
- Atomic operation (no race condition)
- Fast (<1ms latency)
- Prevents overselling (strong consistency)

**Trade-offs:**
- Single point of contention
- Potential data loss on Redis crash (mitigated by replication)

---

### Reservation System (Soft Lock with TTL)

**Why Soft Lock?**

Users need 10 minutes to complete payment. Can't hold hard lock (blocks others) or immediately create final order (payment might fail).

**Design:**
1. User reserves → Redis DECR succeeds → Create reservation record (TTL: 10 min)
2. Payment window → User has 10 minutes to pay
3. Outcomes:
   - **Success:** Payment succeeds → Delete reservation → Create order
   - **Failure:** Payment fails → Delete reservation → Redis INCR (return stock)
   - **Timeout:** TTL expires → Background worker → Redis INCR → Notify user

**Data Model:**

```sql
CREATE TABLE reservations (
    reservation_id UUID PRIMARY KEY,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    status VARCHAR(20),  -- PENDING, PAID, EXPIRED, CANCELLED
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,  -- created_at + 10 minutes
    INDEX idx_expires (expires_at, status)
);
```

**Benefits:**
- Fair FCFS reservation
- Automatic cleanup via TTL
- Payment decoupled from reservation

**Trade-offs:**
- Temporary overselling (stock reserved but not paid)
- Background worker overhead
- 10-minute eventual consistency window

---

### Load Shedding and Rate Limiting

**Challenge:** 1M users attempting to buy 100 items (999,900 will fail). Handling all wastes resources.

**Solution: Multi-Layer Rate Limiting**

**Layer 1: CDN Edge Rate Limiting**
- Token Bucket: 10 requests/sec per IP
- Result: 90% traffic blocked at edge (100K → 10K requests)

**Layer 2: API Gateway Rate Limiting**
- Leaky Bucket: 10K QPS total (all users)
- Result: Queue excess traffic, serve 10K QPS

**Layer 3: CAPTCHA Challenge**
- Trigger: Suspicious behavior (100+ requests/min)
- Result: Slow down bots, ensure fair human access

**Why Token Bucket at Edge?**

| Factor | Token Bucket | Fixed Window | Sliding Window Log |
|--------|-------------|--------------|-------------------|
| **Burst Handling** | ✅ Allows bursts | ❌ Abrupt cutoff | ✅ Smooth but expensive |
| **Edge Deployment** | ✅ Runs at CDN | ✅ Simple | ❌ Too complex |
| **Resource Cost** | ✅ O(1) memory | ✅ O(1) memory | ❌ O(N) memory |

**Benefits:**
- 90% traffic blocked at CDN (cheap)
- Fair burst handling
- Bot protection via CAPTCHA

**Trade-offs:**
- 90% users rejected immediately
- Legitimate users may hit limits
- CAPTCHA adds 2-5s delay

---

### Payment Processing (Saga Pattern)

**Challenge:** Payment is slow (500ms-2s Stripe API) and can fail. Can't process 100K payments synchronously.

**Solution: Saga Pattern with Compensation**

**Saga Steps:**

1. **Reserve Inventory** (Local transaction)
   - Redis DECR (inventory)
   - Create reservation record
   - Compensation: Redis INCR, delete reservation

2. **Charge Customer** (Remote transaction)
   - Call Stripe API with idempotency key
   - Compensation: Stripe refund API

3. **Create Order** (Local transaction)
   - Insert into orders table
   - Compensation: Soft delete order

**Why Saga over 2PC?**

| Factor | Saga Pattern | Two-Phase Commit (2PC) |
|--------|-------------|------------------------|
| **Performance** | ✅ Async, non-blocking | ❌ Synchronous, blocks all |
| **Availability** | ✅ Continues if one down | ❌ Blocks if coordinator down |
| **Scalability** | ✅ High throughput | ❌ Low throughput (global lock) |
| **Consistency** | ⚠️ Eventual | ✅ Strong (ACID) |

**Benefits:**
- Async (reservation instant, payment async)
- Resilient (automatic compensation)
- Scalable (no global locks)

**Trade-offs:**
- Eventual consistency
- Complex rollback logic
- Need idempotency keys

---

### Idempotency (Preventing Double-Charge)

**Challenge:** Network failures cause retries. User might be charged twice.

**Solution: Idempotency Key**

Every payment request includes unique UUID. If duplicate request arrives (same key), return cached result instead of charging again.

**Data Model:**

```sql
CREATE TABLE idempotency_store (
    idempotency_key UUID PRIMARY KEY,
    request_payload JSONB,
    response_payload JSONB,
    status VARCHAR(20),  -- PROCESSING, SUCCESS, FAILED
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,  -- 24 hours
    INDEX idx_expires (expires_at)
);
```

**Flow:**
1. Client generates UUID
2. Server checks idempotency store
3. If exists → Return cached response (no charge)
4. If not exists → Process payment → Cache response

**Benefits:**
- Exactly-once semantics
- Network fault tolerance
- Audit trail

**Trade-offs:**
- 24-hour cache per request
- Storage overhead
- Cleanup required

---

### Hot Key Optimization (Split Counter)

**Challenge:** Even Redis, single key at 100K writes/sec becomes bottleneck.

**Solution: Split Counter (Sharding the Hot Key)**

Instead of one key (`iphone:stock = 100`), split into 10 keys:
- `iphone:stock:0 = 10`
- `iphone:stock:1 = 10`
- ... (10 keys, 10 items each)

**Algorithm:**
1. Hash user_id → shard_id (0-9)
2. Try Redis DECR on `iphone:stock:{shard_id}`
3. If success → Reserved
4. If failure → Try next shard (round-robin)
5. If all fail → Sold out

**Benefits:**
- 10× throughput (10K × 10 = 100K ops/sec)
- Load distribution across 10 instances
- Reduced contention per key

**Trade-offs:**
- Complexity (track multiple keys)
- Uneven distribution
- Eventual consistency for total

---

## Key Algorithms

### Atomic Inventory Reservation

**Description:** Check-and-decrement with Lua script prevents race conditions.

**Steps:**
1. Client sends reservation request
2. Server generates idempotency_key
3. Check idempotency store
4. Execute Lua script: check_and_decr
5. If success: Create reservation, publish to Kafka
6. If sold out: Return error

**Time Complexity:** O(1)

---

### Reservation Cleanup

**Description:** Background worker scans for expired reservations and releases inventory.

**Steps:**
1. Worker wakes every 60 seconds
2. Query expired reservations: `expires_at < NOW() AND status = 'PENDING'`
3. For each: Redis INCR, UPDATE status = 'EXPIRED'
4. Send notification to user

**Time Complexity:** O(N) where N = expired reservations

---

### Payment Saga

**Description:** Multi-step transaction with compensation.

**Steps:**
1. Reserve inventory (Redis DECR)
2. Charge customer (Stripe API)
3. Create order (PostgreSQL INSERT)
4. If any step fails: Execute compensation in reverse

**Time Complexity:** O(1) per step

---

## Data Models

### Reservations Table

```sql
CREATE TABLE reservations (
    reservation_id UUID PRIMARY KEY,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT DEFAULT 1,
    status VARCHAR(20),
    idempotency_key UUID UNIQUE,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,
    amount DECIMAL(10, 2),
    INDEX idx_expires (expires_at, status),
    INDEX idx_idempotency (idempotency_key)
);
```

### Orders Table

```sql
CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    reservation_id UUID NOT NULL,
    product_id BIGINT NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    payment_id VARCHAR(255),  -- Stripe charge_id
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user (user_id),
    INDEX idx_reservation (reservation_id)
);
```

### Redis Data Structures

**Inventory Counter:**
```
KEY: product:{product_id}:stock
TYPE: STRING (integer)
VALUE: 100 → 0
```

**Split Counter:**
```
KEY: product:{product_id}:stock:0
VALUE: 10

KEY: product:{product_id}:stock:1
VALUE: 10

... (10 keys)
```

**Idempotency Cache:**
```
KEY: idempotency:{key}
TYPE: HASH
FIELDS: status, response, timestamp
TTL: 86400 (24 hours)
```

---

## Bottlenecks and Optimizations

### Redis Hot Key Contention

**Problem:** Single Redis key receives 100K writes/sec, causing high latency.

**Solution:**
1. Split Counter: 10 keys (10K writes/sec each)
2. Redis Cluster: Shard across multiple nodes
3. Read Replicas: Route reads to replicas

**Gain:** 10× throughput

---

### Database Write Overload

**Problem:** 100 concurrent DB writes overwhelm PostgreSQL.

**Solution:**
1. Async Order Creation: Kafka queue buffers events
2. Batch Inserts: 10 orders per INSERT
3. Connection Pooling: Reuse connections

**Gain:** 10× throughput via batching

---

### Payment Gateway Rate Limiting

**Problem:** Stripe limits 100 req/sec. Need to process 100 payments.

**Solution:**
1. Async Processing: Spread over 10-20 seconds
2. Multiple Accounts: 5 Stripe accounts (500 req/sec)
3. Retry with Backoff: Handle 429 errors

**Gain:** 5× throughput via sharding

---

## Monitoring and Observability

### Key Metrics

**Business:**
- Sold items (total)
- Success rate (0.1% expected)
- Revenue (total payment amount)
- Abandoned cart rate

**System:**
- Redis throughput (ops/sec on hot key)
- Redis latency (p50, p99, p999)
- API Gateway QPS (capped at 10K)
- Rejection rate (99.9% expected)
- Kafka consumer lag
- Payment success rate

**Alerts:**
- P0: Redis cluster down, overselling detected
- P1: Redis latency >100ms, payment rate <80%
- P2: Kafka lag >1000 messages

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Synchronous Payment

**Problem:** User waits 2 seconds for payment → High abandonment

**Solution:** ✅ Async payment via Kafka → Instant response (<50ms)

---

### ❌ Anti-Pattern 2: No Idempotency

**Problem:** Network retry → Double charge

**Solution:** ✅ Idempotency key → Exactly-once semantics

---

### ❌ Anti-Pattern 3: No Load Shedding

**Problem:** All 1M requests → Backend overload → System crash

**Solution:** ✅ Multi-layer rate limiting → 90% rejected at edge

---

### ❌ Anti-Pattern 4: No Reservation Expiration

**Problem:** Abandoned cart → Stock locked forever

**Solution:** ✅ 10-minute TTL → Auto-release via worker

---

### ❌ Anti-Pattern 5: Database Row Lock

**Problem:** 100K concurrent locks → Deadlocks, timeouts

**Solution:** ✅ Redis atomic DECR → No locks, 100K ops/sec

---

## Alternative Approaches

### Queue-Based Fair Assignment

**Approach:** All users join FIFO queue. Server processes sequentially.

**Pros:** Perfectly fair, predictable
**Cons:** Slow, complex, poor UX

**When:** Fairness > speed (concert tickets)

---

### Lottery System

**Approach:** Users enter lottery. Random selection after registration.

**Pros:** Fair, no hot key
**Cons:** Poor UX (no instant gratification)

**When:** Demand >>> supply (1M users, 10 items)

---

### Pre-Reservation (Waitlist)

**Approach:** Users join waitlist before sale. Process in order.

**Pros:** Smoother traffic, fair
**Cons:** Advance planning required

**When:** Want to spread load over hours/days

---

## Trade-offs Summary

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ High throughput (100K QPS) | ❌ Complex architecture |
| ✅ Strong consistency | ❌ Eventual consistency (10-min window) |
| ✅ Fast UX (<50ms) | ❌ High infrastructure cost |
| ✅ Fault tolerance | ❌ Complex error handling |
| ✅ Scalable | ❌ Operational overhead |

---

## Real-World Examples

### Alibaba Singles Day (11.11)

**Scale:** 583M users, $74B GMV, 583K orders/sec peak

**Architecture:**
- Redis cluster with 1000 shards per product
- Custom payment processor (100K TPS)
- Multi-layer rate limiting

---

### Amazon Prime Day

**Scale:** 250M items sold, 100M Prime members

**Architecture:**
- DynamoDB with optimistic locking
- SQS for async processing
- Lightning Deals with 4-hour windows

---

### Supreme Streetwear Drops

**Scale:** 100-500 items, 10K concurrent users (90% bots)

**Architecture:**
- Aggressive bot detection
- Virtual waiting room (10-min queue)
- Redis atomic DECR

---

## Conclusion

Flash sale systems require careful balancing of throughput, latency, consistency, and cost. Key challenges:

1. **Hot Key Contention:** 100K writes/sec on single Redis key
2. **Strong Consistency:** Zero tolerance for overselling
3. **Graceful Degradation:** Reject 99.9% without crashing
4. **Payment Decoupling:** Async via Saga pattern
5. **Fault Tolerance:** Automatic compensation

**Critical Decisions:**
- Redis atomic operations for inventory
- Saga pattern for distributed transactions
- Multi-layer rate limiting for load shedding
- Idempotency keys for exactly-once payment
- Split counter for 10× throughput

**Real-World Validation:**
- Alibaba: 583K orders/sec
- Amazon: 250M items sold
- Supreme: 90% bot traffic blocked

The architecture scales from 100 to 10,000+ items by sharding Redis keys and increasing capacity. Patterns apply to any high-contention system: concert tickets, sneakers, vaccine appointments, etc.

---

## References

- **Related Chapters:**
  - [2.5.1 Rate Limiting Algorithms](../../02-components/2.5-algorithms/2.5.1-rate-limiting-algorithms.md)
  - [2.3.4 Distributed Transactions & Idempotency](../../02-components/2.3-messaging-streaming/2.3.4-distributed-transactions-and-idempotency.md)
  - [2.1.11 Redis Deep Dive](../../02-components/2.1-databases/2.1.11-redis-deep-dive.md)
  - [2.5.3 Distributed Locking](../../02-components/2.5-algorithms/2.5.3-distributed-locking.md)

- **Official Documentation:**
  - [Redis Transactions](https://redis.io/topics/transactions)
  - [Stripe Idempotency](https://stripe.com/docs/api/idempotent_requests)

- **Industry Articles:**
  - [Alibaba Singles Day Architecture](https://www.alibabacloud.com/blog/596631)
  - [Amazon Prime Day 2021](https://aws.amazon.com/blogs/aws/prime-day-2021-two-chart-topping-days/)
