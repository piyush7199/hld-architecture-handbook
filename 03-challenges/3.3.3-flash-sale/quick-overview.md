# Flash Sale System - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A flash sale system for an e-commerce platform (like Amazon Prime Day, Alibaba Singles Day) that handles extreme traffic spikes when selling limited-quantity, high-demand products. The system prevents overselling (selling more than available stock), handles 100K+ concurrent purchase attempts per second, maintains strong consistency for inventory counts, and gracefully degrades under load while ensuring a fair purchasing experience.

**Core Challenge:** Managing a single "hot key" (inventory counter) that receives 100,000 writes per second when only 100 items are available (0.1% success rate), while maintaining ACID properties for successful purchases and preventing double-charging or overselling.

**Example Scenario:** Apple releases 100 limited-edition iPhones at 12:00 PM. 1 million users are ready to click "Buy Now" at exactly 12:00:00. The system must correctly sell exactly 100 phones, reject 999,900 users fairly, process payments atomically, and handle refunds if payments fail.

**Key Characteristics:**
- **Extreme contention** - 100K writes/sec on single Redis key
- **Strong consistency** - Zero overselling (ACID properties)
- **High rejection rate** - 99.9% of users fail (graceful error handling)
- **Time-bound** - Flash sale lasts 1-2 seconds (all items sold instantly)
- **Payment decoupling** - Async payment processing (can't process 100K payments synchronously)

---

## Requirements & Scale

### Functional Requirements

| Requirement | Description | Priority |
|-------------|-------------|----------|
| **Inventory Protection** | Prevent overselling (e.g., only 100 iPhones available) | Must Have |
| **Reservation System** | Allow users to temporarily reserve items before payment | Must Have |
| **Payment Processing** | Complete payment within time window (10 minutes) | Must Have |
| **Automatic Refund** | Cancel order and refund if payment fails or times out | Must Have |
| **Fair Queue** | First-come-first-served (FCFS) order for reservations | Should Have |
| **Real-Time Stock Updates** | Show remaining inventory count to users | Should Have |
| **Cancellation** | Allow users to cancel reservations voluntarily | Should Have |
| **Multi-Item Flash Sale** | Support multiple products simultaneously | Nice to Have |

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Strong Consistency** | 100% accurate inventory count | Overselling causes brand damage, legal issues |
| **High Throughput** | 100K writes/sec (purchase attempts) | Peak load during flash sale start |
| **Low Latency (Reservation)** | < 50ms (p99) | Users abandon if slow |
| **Fault Tolerance** | Auto-rollback on failure | Partial transactions unacceptable |
| **Availability** | 99.9% uptime during sale | Downtime = lost revenue |
| **Fairness** | No bot advantage | Prevent scalpers, ensure fair access |

### Scale Estimation

| Metric | Assumption | Calculation | Result |
|--------|-----------|-------------|--------|
| **Concurrent Users** | 1 Million users waiting at start time | - | 1M users |
| **Peak Purchase Attempts** | 100,000 "Buy Now" clicks per second | - | 100K QPS (writes) |
| **Available Inventory** | 100 limited-edition items | - | 100 items |
| **Success Rate** | Items / Attempts | 100 / 100,000 = 0.001 | 0.1% success rate |
| **Rejection Rate** | Failed attempts | 99,900 / 100,000 | 99.9% rejections |
| **Critical Section Ops** | Atomic inventory check + decrement | 100,000 × 2 = 200,000 | 200K ops/sec on 1 key |
| **Payment Processing** | Successful reservations | 100 payments | 100 payments total |
| **Reservation Timeout** | 10-minute window | - | 10 min TTL |

**Key Insights:**
- **Hot key contention:** Single Redis key receives 100K writes/sec (severe contention)
- **Extreme rejection rate:** 99.9% of users fail (need graceful error handling)
- **Time-bound:** Flash sale lasts 1-2 seconds (all 100 items sold instantly)
- **Payment decoupling critical:** Can't process payments synchronously at 100K QPS

---

## Key Components

| Component | Responsibility | Technology | Scalability |
|-----------|---------------|------------|-------------|
| **CDN / Load Balancer** | Geographic distribution, edge rate limiting | CloudFlare, Fastly | Horizontal |
| **API Gateway** | Rate limiting, bot detection, load shedding | NGINX, Kong | Horizontal |
| **Inventory Reservation** | Atomic Redis DECR, create reservation | Redis + Lua scripts | Horizontal (split counter) |
| **Reservation Store** | Store reservations with TTL | PostgreSQL | Horizontal (sharded) |
| **Payment Service** | Process payments asynchronously | Stripe API + Kafka | Horizontal |
| **Order Service** | Create orders after payment success | PostgreSQL | Horizontal (sharded) |
| **Cleanup Worker** | Release expired reservations | Cron job + Workers | Horizontal |

---

## Inventory Management Strategy

**Challenge:** 100K concurrent writes to a single Redis key (iphone:stock).

**Solution: Atomic Redis DECR with Lua Script**

Redis provides atomic operations (DECR, INCR) that can handle millions of operations per second on a single key. We use a Lua script to make check-and-decrement atomic.

**Why Redis DECR over Database Row Lock?**

| Factor | Redis DECR | Database Row Lock (PostgreSQL) |
|--------|------------|-------------------------------|
| **Throughput** | ✅ 100K ops/sec per key | ❌ 1K ops/sec (row lock contention) |
| **Latency** | ✅ <1ms (in-memory) | ❌ 10-50ms (disk I/O, lock wait) |
| **Atomicity** | ✅ Native atomic operations | ✅ ACID compliant |
| **Durability** | ⚠️ RDB snapshots (1-5s data loss risk) | ✅ WAL (zero data loss) |
| **Complexity** | ✅ Simple (single command) | ❌ Complex (transaction isolation) |

**Decision:** Use Redis for hot inventory counter, replicate final sales to PostgreSQL for durability.

**Lua Script for Atomic Check-and-Decrement:**
```lua
-- Check if stock available, decrement if yes
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
- **Atomic:** Check and decrement execute as one operation (no race condition)
- **Fast:** In-memory execution (<1ms latency)
- **Accurate:** Prevents overselling (strong consistency)

**Trade-offs:**
- **Single point of contention:** All writes hit one Redis master
- **Data loss risk:** Redis crash loses 1-5 seconds of data (solved by replication)

---

## Reservation System (Soft Lock with TTL)

**Why Soft Lock?**

Users need time to complete payment (10 minutes). We can't hold a hard lock (blocks other users) or immediately create a final order (payment might fail).

**Soft Lock Design:**

1. **User reserves:** Redis DECR succeeds, create reservation record
2. **Reservation TTL:** 10-minute expiration (auto-release if not paid)
3. **Payment window:** User has 10 minutes to pay
4. **Outcomes:**
   - **Success:** Payment succeeds → Delete reservation → Create order
   - **Failure:** Payment fails → Delete reservation → Redis INCR (return stock)
   - **Timeout:** TTL expires → Background worker → Redis INCR → Notify user

**Data Model:**
```sql
CREATE TABLE reservations (
    reservation_id UUID PRIMARY KEY,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    status VARCHAR(20),  -- PENDING, PAID, EXPIRED, CANCELLED
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,  -- created_at + 10 minutes
    INDEX idx_expires (expires_at, status)  -- For cleanup worker
);
```

**Benefits:**
- **Fair:** First-come-first-served (FCFS) reservation
- **Automatic cleanup:** TTL expires → worker releases stock
- **Payment decoupling:** Reservation instant, payment async

**Trade-offs:**
- **Temporary overselling:** Stock reserved but not paid (99% will timeout)
- **Cleanup overhead:** Background workers scan for expired reservations
- **Eventual consistency:** 10-minute delay between reserve and final order

---

## Load Shedding and Rate Limiting

**Challenge:** 1M users attempt to buy 100 items (999,900 will fail). Handling all 1M requests wastes resources.

**Solution: Multi-Layer Rate Limiting**

**Layer 1: CDN Edge Rate Limiting**
- **Token Bucket:** 10 requests/second per IP
- **Purpose:** Block bot traffic at edge (before reaching backend)
- **Result:** 90% traffic blocked (100K requests → 10K requests)

**Layer 2: API Gateway Rate Limiting**
- **Leaky Bucket:** 10K QPS total (all users combined)
- **Purpose:** Protect backend services from overload
- **Result:** Queue excess traffic, serve 10K QPS

**Layer 3: CAPTCHA Challenge**
- **Trigger:** Suspicious behavior (100+ requests/minute from single IP)
- **Purpose:** Prevent automated bots from scalpers
- **Result:** Slow down bots, ensure fair human access

**Why Token Bucket at Edge?**

| Factor | Token Bucket (Edge) | Fixed Window Counter | Sliding Window Log |
|--------|-------------------|---------------------|-------------------|
| **Burst Handling** | ✅ Allows short bursts (good UX) | ❌ Abrupt cutoff at window boundary | ✅ Smooth but expensive |
| **Edge Deployment** | ✅ Runs at CDN (low latency) | ✅ Simple to implement | ❌ Too complex for edge |
| **Resource Cost** | ✅ O(1) memory per IP | ✅ O(1) memory | ❌ O(N) memory (N = requests) |

**Benefits:**
- **Cost savings:** 90% traffic blocked at CDN (cheap), 10% reaches backend (expensive)
- **Fair access:** Token bucket allows short bursts (good for flash sale start)
- **Bot protection:** CAPTCHA challenges prevent automated scalpers

**Trade-offs:**
- **User frustration:** 90% users rejected immediately
- **False positives:** Legitimate users may hit rate limits
- **CAPTCHA friction:** Adds 2-5 seconds delay (reduces conversion)

---

## Payment Processing (Saga Pattern)

**Challenge:** Payment is slow (500ms-2s Stripe API call) and can fail. Can't process 100K payments synchronously.

**Solution: Saga Pattern with Compensation**

The Saga pattern breaks a distributed transaction into multiple local transactions, each with a compensation transaction to undo it if needed.

**Saga Steps:**

1. **Reserve Inventory** (Local transaction)
   - Redis DECR (inventory)
   - Create reservation record
   - **Compensation:** Redis INCR, delete reservation

2. **Charge Customer** (Remote transaction)
   - Call Stripe API with idempotency key
   - **Compensation:** Stripe refund API

3. **Create Order** (Local transaction)
   - Insert into orders table (PostgreSQL)
   - **Compensation:** Soft delete order

**Why Saga over 2PC (Two-Phase Commit)?**

| Factor | Saga Pattern | Two-Phase Commit (2PC) |
|--------|-------------|------------------------|
| **Performance** | ✅ Async, non-blocking | ❌ Synchronous, blocks all participants |
| **Availability** | ✅ Continues if one service down | ❌ Blocks if coordinator down |
| **Scalability** | ✅ High throughput | ❌ Low throughput (global lock) |
| **Consistency** | ⚠️ Eventual consistency | ✅ Strong consistency (ACID) |
| **Complexity** | ❌ Complex (compensation logic) | ✅ Simpler (atomic commit/rollback) |

**Decision:** Use Saga for flash sale (performance > consistency), use 2PC for financial ledger (consistency critical).

**Saga Orchestration Flow:**
```
Step 1: Reserve Inventory
├─ Success → Step 2
└─ Failure → Reject user

Step 2: Charge Customer (Stripe)
├─ Success → Step 3
└─ Failure → Compensate Step 1 (INCR inventory)

Step 3: Create Order
├─ Success → Complete
└─ Failure → Compensate Step 2 (refund) + Step 1 (INCR)
```

**Benefits:**
- **Async processing:** Payment doesn't block reservation
- **Fault tolerance:** Automatic compensation on failure
- **High throughput:** Process 100 payments in parallel

**Trade-offs:**
- **Complexity:** Compensation logic required
- **Eventual consistency:** Order created after payment (not atomic)
- **Error handling:** Must handle partial failures

---

## Bottlenecks & Solutions

### Bottleneck 1: Redis Hot Key Contention

**Problem:** Single Redis key receives 100K writes/sec, causing high latency and potential timeouts.

**Solution:**
1. **Split Counter:** Divide inventory across 10 keys (10K writes/sec per key)
2. **Redis Cluster:** Shard keys across multiple Redis nodes
3. **Read Replicas:** Route read queries (stock count display) to replicas

**Performance Gain:** 10× throughput (10K → 100K ops/sec)

### Bottleneck 2: Database Write Overload

**Problem:** 100 successful orders create 100 DB writes. If payment takes 2 seconds, 100 concurrent DB transactions overwhelm PostgreSQL.

**Solution:**
1. **Async Order Creation:** Kafka queue buffers order creation events
2. **Batch Inserts:** Workers batch 10 orders into single INSERT
3. **Connection Pooling:** Reuse DB connections (reduce overhead)

**Performance Gain:** 10× throughput via batching

### Bottleneck 3: Payment Gateway Rate Limiting

**Problem:** Stripe API has rate limits (100 requests/sec per account). 100 payments in 1 second exceed limit.

**Solution:**
1. **Client-Side Retry:** Exponential backoff on 429 errors
2. **Payment Queue:** Kafka queue spreads payments over 10-20 seconds
3. **Multiple Stripe Accounts:** Shard users across 5 Stripe accounts (500 req/sec total)

**Performance Gain:** 5× throughput via account sharding

### Bottleneck 4: Reservation Cleanup Lag

**Problem:** Background worker runs every 60 seconds. Expired reservations sit for up to 60 seconds before cleanup.

**Solution:**
1. **TTL-Based Expiration:** Redis key with TTL expires automatically (instant)
2. **Event-Driven Cleanup:** Publish "reservation_expired" event at expiration time
3. **Increase Worker Frequency:** Run every 10 seconds (trade-off: higher CPU)

**Performance Gain:** 6× faster cleanup (60s → 10s)

---

## Common Anti-Patterns to Avoid

### 1. Synchronous Payment Processing

❌ **Bad:** Process payment synchronously
```
User clicks "Buy Now" → Reserve inventory → Charge Stripe (2s) → Return response (2s total)
```
- With 100 concurrent users, payment processing becomes bottleneck
- Users wait 2+ seconds, abandonment increases

✅ **Good:** Async Payment
- Reserve inventory (<50ms) → Return immediately → Process payment async via Kafka
- User experience: 50ms vs 2 seconds
- Throughput: 100K QPS vs 500 QPS (limited by Stripe)

### 2. No Idempotency

❌ **Bad:** No idempotency key
- User clicks "Buy Now" twice (network retry)
- Two reservations created for same user
- Overselling risk

✅ **Good:** Idempotency Key
- Generate UUID per request
- Check idempotency store before processing
- Return same reservation_id for duplicate requests

### 3. Database Row Lock for Inventory

❌ **Bad:** Use PostgreSQL row lock
```sql
BEGIN;
SELECT * FROM inventory WHERE product_id = 123 FOR UPDATE;
UPDATE inventory SET stock = stock - 1 WHERE product_id = 123;
COMMIT;
```
- Row lock contention: 100K concurrent requests → massive queue
- Latency: 10-50ms per transaction
- Throughput: ~1K ops/sec (can't handle 100K QPS)

✅ **Good:** Redis Atomic DECR
- Single atomic operation: `DECR iphone:stock`
- Latency: <1ms
- Throughput: 100K ops/sec

### 4. No Rate Limiting

❌ **Bad:** Accept all 1M requests
- Backend overwhelmed (1M requests → 100K QPS)
- Wastes resources (99.9% will fail)
- Poor user experience (slow responses)

✅ **Good:** Multi-Layer Rate Limiting
- CDN edge: 10 req/sec per IP (blocks 90%)
- API Gateway: 10K QPS total (queues excess)
- Result: 10K requests reach backend (manageable)

---

## Monitoring & Observability

### Key Metrics

**Business Metrics:**
- **Sold items:** Total items sold
- **Success rate:** Sold / Attempts (should be ~0.1%)
- **Revenue:** Total payment amount
- **Abandoned cart rate:** Reservations expired / Total reservations

**System Metrics:**
- **Redis throughput:** Ops/sec on hot key
- **Redis latency:** p50, p99, p999 for DECR operations
- **API Gateway QPS:** Requests/sec (should be capped at 10K)
- **Rejection rate:** Requests rejected / Total requests
- **Kafka consumer lag:** Lag in processing payment events
- **Payment success rate:** Successful payments / Total attempts
- **Reservation expiration rate:** Expired / Total reservations

**Alerts:**
- **P0 (Critical):** Redis cluster down, overselling detected
- **P1 (High):** Redis latency >100ms, payment success rate <80%
- **P2 (Medium):** Kafka lag >1000 messages, reservation expiration >50%

---

## Trade-offs Summary

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ High throughput (100K QPS) | ❌ Complex architecture (many components) |
| ✅ Strong consistency (no overselling) | ❌ Eventual consistency (10-min reservation window) |
| ✅ Fast user experience (<50ms) | ❌ High infrastructure cost (Redis cluster, Kafka) |
| ✅ Fault tolerance (auto-compensation) | ❌ Complex error handling (saga pattern) |
| ✅ Scalable (split counter, sharding) | ❌ Operational overhead (monitoring, tuning) |

---

## Real-World Examples

### Alibaba Singles Day (11.11)
- **Scale:** 583 million users, $74 billion GMV in 24 hours, 583,000 orders per second (peak)
- **Architecture:** Redis cluster with split counters (1000 shards per product), Alipay (custom payment processor, 100K TPS), multi-layer rate limiting
- **Key Innovation:** "Pre-order" system where users add to cart before sale, reducing hot key contention

### Amazon Prime Day
- **Scale:** 250 million items sold, 100 million Prime members
- **Architecture:** DynamoDB with optimistic locking (version field), AWS-hosted payment gateway, SQS for async order processing
- **Key Innovation:** "Lightning Deals" with countdown timer, spreading demand over 4-hour windows

### Supreme Streetwear Drops
- **Scale:** 100-500 items, 10,000 concurrent users (bots)
- **Architecture:** CAPTCHA, browser fingerprinting, IP rate limiting, Redis atomic DECR, virtual waiting room (10-minute queue)
- **Key Challenge:** 90% traffic is bots. Aggressive bot detection and CAPTCHA challenges required.

---

## Key Takeaways

1. **Redis atomic DECR** handles 100K writes/sec on single key (vs 1K with DB locks)
2. **Lua scripts** ensure atomic check-and-decrement (prevents race conditions)
3. **Reservation system** with TTL decouples inventory from payment (10-minute window)
4. **Saga pattern** enables async payment processing (performance > consistency)
5. **Multi-layer rate limiting** blocks 90% traffic at edge (cost savings)
6. **Split counter** reduces hot key contention (10 keys → 10× throughput)
7. **Idempotency keys** prevent duplicate reservations (network retries)
8. **Kafka queue** buffers payment events (handles Stripe rate limits)
9. **Background workers** cleanup expired reservations (auto-release stock)
10. **Monitoring critical** (Redis latency, payment success rate, overselling detection)

---

## Recommended Stack

- **Inventory Counter:** Redis Cluster (atomic DECR, Lua scripts)
- **Reservation Store:** PostgreSQL (ACID transactions, TTL-based cleanup)
- **Message Queue:** Apache Kafka (async payment processing)
- **Payment Gateway:** Stripe (idempotency keys, webhooks)
- **Rate Limiting:** CDN edge (Token Bucket), API Gateway (Leaky Bucket)
- **Monitoring:** Prometheus + Grafana (metrics, dashboards)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, component design, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (reservation, payment saga, cleanup)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (atomic reservation, saga orchestration, cleanup)

