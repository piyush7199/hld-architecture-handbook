# E-commerce Flash Sale System - Sequence Diagrams

## Table of Contents

1. [User Reservation Flow (Happy Path)](#1-user-reservation-flow-happy-path)
2. [Payment Saga Success Flow](#2-payment-saga-success-flow)
3. [Payment Failure and Compensation](#3-payment-failure-and-compensation)
4. [Reservation Expiration and Cleanup](#4-reservation-expiration-and-cleanup)
5. [Concurrent Reservation Race Condition](#5-concurrent-reservation-race-condition)
6. [Split Counter Reservation Flow](#6-split-counter-reservation-flow)
7. [Idempotency Check Flow](#7-idempotency-check-flow)
8. [Rate Limiting Multi-Layer Flow](#8-rate-limiting-multi-layer-flow)
9. [Redis Failover Flow](#9-redis-failover-flow)
10. [Bot Detection and CAPTCHA Challenge](#10-bot-detection-and-captcha-challenge)
11. [Kafka Consumer Lag Recovery](#11-kafka-consumer-lag-recovery)
12. [Complete End-to-End Purchase Flow](#12-complete-end-to-end-purchase-flow)

---

## 1. User Reservation Flow (Happy Path)

**Flow:**

Shows the complete happy path from user clicking "Buy Now" to receiving reservation confirmation in under 50ms.

**Steps:**
1. **User clicks** (0ms): User taps "Buy Now" button
2. **API Gateway** (10ms): Token bucket rate limit check passes
3. **Reservation Service** (20ms): Generates idempotency key (UUID)
4. **Redis check** (25ms): Execute Lua script check_and_decr
5. **Lua execution** (26ms): GET stock (100), Check > 0, DECR (99)
6. **Create reservation** (35ms): INSERT into reservations table with TTL
7. **Kafka publish** (40ms): Publish reservation event (async)
8. **Response** (45ms): Return reservation_id to user
9. **Payment processing** (async): Kafka consumer processes payment (2-10s)

**Performance:**
- **User response time:** 45ms (instant feedback)
- **Success rate:** 0.1% (100 out of 100,000 attempts)
- **Payment latency:** 2-10 seconds (async, doesn't block user)

**Benefits:**
- Fast user experience (< 50ms)
- Async payment (doesn't block)
- Strong consistency (atomic DECR)

**Trade-offs:**
- Eventual consistency (payment async)
- High rejection rate (99.9% fail)

```mermaid
sequenceDiagram
    participant User
    participant Gateway as API Gateway
    participant ReservationSvc as Reservation Service
    participant Redis
    participant ReservationDB as Reservation DB
    participant Kafka
    
    User->>Gateway: POST /buy<br/>{product_id: iphone, user_id: 123}
    Note right of User: Click Buy Now<br/>at 12:00:00.000
    
    Gateway->>Gateway: Rate limit check<br/>Token bucket: OK
    
    Gateway->>ReservationSvc: Forward request
    
    ReservationSvc->>ReservationSvc: Generate UUID<br/>idempotency_key
    
    ReservationSvc->>Redis: EVAL check_and_decr<br/>KEYS[1] = iphone:stock
    Note right of Redis: Lua script atomic
    
    Redis->>Redis: GET iphone:stock → 100
    Redis->>Redis: Check: 100 gt 0 ✓
    Redis->>Redis: DECR: 100 → 99
    Redis->>Redis: Return 1 (SUCCESS)
    
    Redis-->>ReservationSvc: 1 (success)
    
    ReservationSvc->>ReservationDB: INSERT INTO reservations<br/>(id, user_id, expires_at)<br/>TTL: 10 minutes
    ReservationDB-->>ReservationSvc: OK
    
    ReservationSvc->>Kafka: Publish reservation event<br/>(async, non-blocking)
    Note right of Kafka: Event buffered<br/>Consumer processes later
    
    ReservationSvc-->>User: 200 OK<br/>{reservation_id, expires_at}<br/>(45ms total)
    
    Note over User: Success!<br/>User has 10 minutes to pay
```

---

## 2. Payment Saga Success Flow

**Flow:**

Shows the complete Saga pattern with three steps: reserve inventory, charge customer, create order. All steps succeed in this happy path.

**Steps:**
1. **Kafka consumer** pulls reservation event
2. **Saga Step 1:** Check reservation still valid (not expired)
3. **Saga Step 2:** Charge customer via Stripe API (idempotency key)
4. **Stripe processing:** 500ms-2s latency
5. **Saga Step 3:** Create final order in PostgreSQL (ACID)
6. **Cleanup:** Delete reservation record
7. **Notify user:** Send confirmation email + push notification

**Performance:**
- **Step 1:** 10ms (DB query)
- **Step 2:** 500ms-2s (Stripe API)
- **Step 3:** 50ms (DB insert)
- **Total:** 560ms-2.06s end-to-end

**Benefits:**
- Async processing (doesn't block user)
- Idempotent payment (no double-charge)
- ACID order creation

**Trade-offs:**
- Long latency (2+ seconds)
- Complex error handling
- Requires compensation logic

```mermaid
sequenceDiagram
    participant Kafka
    participant Saga as Payment Saga Orchestrator
    participant ReservationDB as Reservation DB
    participant Stripe as Stripe API
    participant OrderDB as Order DB
    participant User
    
    Kafka->>Saga: Pull reservation event<br/>{reservation_id, user_id, amount}
    
    Note over Saga: Saga Step 1:<br/>Validate Reservation
    
    Saga->>ReservationDB: SELECT * FROM reservations<br/>WHERE id = ? AND status = PENDING
    ReservationDB-->>Saga: Reservation valid ✓
    
    Note over Saga: Saga Step 2:<br/>Charge Customer
    
    Saga->>Stripe: POST /v1/charges<br/>{amount: 999,<br/>idempotency_key: UUID}
    Note right of Stripe: Idempotency key<br/>prevents double-charge
    
    Stripe->>Stripe: Process payment<br/>(500ms-2s)
    Stripe-->>Saga: {charge_id: ch_123,<br/>status: succeeded}
    
    Note over Saga: Saga Step 3:<br/>Create Order
    
    Saga->>OrderDB: BEGIN TRANSACTION
    Saga->>OrderDB: INSERT INTO orders<br/>(reservation_id, payment_id, status)
    Saga->>OrderDB: DELETE FROM reservations<br/>WHERE id = ?
    Saga->>OrderDB: COMMIT
    OrderDB-->>Saga: Order created ✓
    
    Saga->>User: Push notification<br/>Order confirmed!
    Saga->>User: Email receipt
    
    Note over User: Purchase complete!<br/>Total time: 2-10 seconds
```

---

## 3. Payment Failure and Compensation

**Flow:**

Shows the Saga compensation flow when payment fails. The system automatically rolls back the reservation and returns inventory to Redis.

**Steps:**
1. **Saga Step 1:** Validate reservation (success)
2. **Saga Step 2:** Charge customer (FAILURE - card declined)
3. **Compensation begins:** Rollback Step 1
4. **Redis INCR:** Return inventory (99 → 100)
5. **Update reservation:** SET status = FAILED
6. **Notify user:** Payment declined, please update card
7. **Result:** System state consistent (no partial transactions)

**Performance:**
- **Failure detection:** Immediate (Stripe API returns error)
- **Compensation time:** 50ms (Redis INCR + DB update)
- **Inventory returned:** Within 100ms

**Benefits:**
- Automatic rollback (no manual intervention)
- Inventory returned immediately
- User notified of failure

**Trade-offs:**
- Complex compensation logic
- Multiple DB writes (performance cost)
- Brief period of inconsistency (eventual consistency)

```mermaid
sequenceDiagram
    participant Saga as Payment Saga
    participant ReservationDB
    participant Stripe
    participant Redis
    participant User
    
    Note over Saga: Saga Step 1: Validate
    Saga->>ReservationDB: Check reservation
    ReservationDB-->>Saga: Valid ✓
    
    Note over Saga: Saga Step 2: Charge (FAILS)
    Saga->>Stripe: POST /v1/charges<br/>{amount: 999}
    Stripe-->>Saga: ERROR: card_declined<br/>{code: CARD_DECLINED}
    
    Note over Saga: Compensation: Rollback Step 1
    
    Saga->>Saga: Payment failed!<br/>Execute compensation
    
    Saga->>Redis: INCR iphone:stock<br/>(99 → 100)
    Note right of Redis: Return inventory<br/>immediately
    Redis-->>Saga: OK (stock: 100)
    
    Saga->>ReservationDB: UPDATE reservations<br/>SET status = FAILED<br/>WHERE id = ?
    ReservationDB-->>Saga: Updated
    
    Saga->>User: Push notification<br/>Payment declined
    Saga->>User: Email: Update payment method
    
    Note over User: Purchase failed<br/>Inventory returned<br/>Can try again
```

---

## 4. Reservation Expiration and Cleanup

**Flow:**

Shows the background worker process that scans for expired reservations (TTL elapsed) and automatically returns inventory to Redis.

**Steps:**
1. **Worker wakes** (every 60 seconds): Cron trigger
2. **Query expired:** `WHERE expires_at < NOW() AND status = PENDING`
3. **Found 10 expired:** Users didn't pay within 10 minutes
4. **For each reservation:**
   - Redis INCR (return inventory)
   - UPDATE status = EXPIRED
   - Notify user (reservation expired)
5. **Sleep 60 seconds:** Wait for next scan

**Performance:**
- **Worker frequency:** Every 60 seconds
- **Cleanup lag:** Max 60 seconds
- **Batch size:** 1,000 reservations per scan
- **Processing time:** 5-10 seconds per batch

**Benefits:**
- Automatic cleanup (no manual intervention)
- Inventory returned for next users
- Fair access (time-bound reservations)

**Trade-offs:**
- Cleanup lag (up to 60s delay)
- Worker overhead (continuous scanning)
- Race condition risk (user pays while worker expires)

```mermaid
sequenceDiagram
    participant Worker as Cleanup Worker
    participant ReservationDB
    participant Redis
    participant User
    
    Note over Worker: Cron: Every 60 seconds
    
    Worker->>Worker: Wake up<br/>Scan for expired
    
    Worker->>ReservationDB: SELECT * FROM reservations<br/>WHERE expires_at lt NOW()<br/>AND status = PENDING<br/>LIMIT 1000
    
    ReservationDB-->>Worker: 10 expired reservations
    
    loop For each expired reservation
        Worker->>Redis: INCR product:stock<br/>(Return inventory)
        Redis-->>Worker: OK (stock increased)
        
        Worker->>ReservationDB: UPDATE reservations<br/>SET status = EXPIRED<br/>WHERE id = ?
        ReservationDB-->>Worker: Updated
        
        Worker->>User: Push notification<br/>Reservation expired
        Worker->>User: Email: Time limit exceeded
    end
    
    Note over Worker: Processed 10 expirations<br/>Returned 10 items to inventory
    
    Worker->>Worker: Sleep 60 seconds
    
    Note over Worker: Next scan in 60s
```

---

## 5. Concurrent Reservation Race Condition

**Flow:**

Shows how two users attempting to reserve the last item simultaneously are handled atomically by Redis Lua script, preventing overselling.

**Steps:**
1. **User A and User B** both click "Buy Now" for last item (stock = 1)
2. **User A request** arrives at Redis first (by microseconds)
3. **Lua script for User A** locks Redis (single-threaded execution)
4. **Lua execution:** GET stock (1) → Check (1 > 0) → DECR (0) → Return SUCCESS
5. **User B request** waits until User A's Lua script completes
6. **Lua script for User B:** GET stock (0) → Check (0 > 0 fails) → Return SOLD_OUT
7. **Result:** User A succeeds, User B receives error (no overselling)

**Performance:**
- User A latency: <1ms (Lua script execution)
- User B latency: <1ms (queued, then executes)
- Total difference: <1ms (microseconds apart)

**Benefits:**
- Prevents race condition (atomicity)
- No deadlocks (Lua script is atomic)
- Accurate inventory (exactly 1 item sold)

**Trade-offs:**
- Sequential execution (B waits for A)
- Microsecond delays (negligible)

```mermaid
sequenceDiagram
    participant UserA as User A
    participant UserB as User B
    participant Service as Reservation Service
    participant Redis
    
    Note over Redis: Stock: 1 (last item)
    
    UserA->>Service: POST /reserve (T=0ms)
    UserB->>Service: POST /reserve (T=0.001ms)
    
    Service->>Redis: User A: EVAL check_and_decr
    Note over Redis: Lua script locks Redis<br/>(single-threaded)
    
    Redis->>Redis: User A Lua:<br/>GET stock → 1
    Redis->>Redis: Check: 1 > 0 ✓
    Redis->>Redis: DECR: 1 → 0
    
    Note over UserB,Redis: User B waits...<br/>(queued)
    
    Redis-->>Service: User A: SUCCESS (stock: 0)
    Service-->>UserA: 200 OK<br/>Reserved!
    
    Note over Redis: Lua unlocked
    
    Service->>Redis: User B: EVAL check_and_decr
    
    Redis->>Redis: User B Lua:<br/>GET stock → 0
    Redis->>Redis: Check: 0 > 0 ✗
    Redis->>Redis: RETURN: SOLD_OUT
    
    Redis-->>Service: User B: SOLD_OUT (stock: 0)
    Service-->>UserB: 410 Gone<br/>Sold Out
    
    Note over UserA,UserB: Result: A succeeds, B fails<br/>No overselling!
```

---

## 6. Split Counter Reservation Flow

**Flow:**

Shows how split counter optimization distributes load across 10 Redis keys, with fallback to next shard if current shard is empty.

**Steps:**
1. **User makes reservation** request
2. **Hash user_id** → shard_id (e.g., user_id 12345 mod 10 = 5)
3. **Try shard 5:** EVAL check_and_decr on `iphone:stock:5`
4. **If success:** Return reservation (fast path)
5. **If failure (shard empty):** Try next shard (round-robin)
6. **Try shards 6, 7, 8...** until success or all 10 fail
7. **If all fail:** Return SOLD_OUT

**Performance:**
- **Best case:** First try success (<1ms)
- **Worst case:** Try all 10 shards (~10ms)
- **Average case:** 2-3 tries (~2ms)

**Benefits:**
- 10× throughput (distributed load)
- No single bottleneck
- Automatic fallback (resilient)

**Trade-offs:**
- Uneven distribution (some shards empty first)
- Multiple Redis calls (worst case: 10)
- Complex logic (shard selection)

```mermaid
sequenceDiagram
    participant User
    participant Service as Reservation Service
    participant Router as Hash Router
    participant Shard5 as Redis Shard 5
    participant Shard6 as Redis Shard 6
    participant Shard7 as Redis Shard 7
    
    User->>Service: POST /reserve<br/>{user_id: 12345}
    
    Service->>Router: Hash user_id
    Router-->>Service: shard_id = 12345 mod 10 = 5
    
    Service->>Shard5: EVAL check_and_decr<br/>iphone:stock:5
    
    Shard5->>Shard5: GET stock → 0<br/>(shard empty!)
    Shard5-->>Service: SOLD_OUT (shard 5)
    
    Note over Service: Try next shard (round-robin)
    
    Service->>Shard6: EVAL check_and_decr<br/>iphone:stock:6
    
    Shard6->>Shard6: GET stock → 0<br/>(shard empty!)
    Shard6-->>Service: SOLD_OUT (shard 6)
    
    Note over Service: Try next shard
    
    Service->>Shard7: EVAL check_and_decr<br/>iphone:stock:7
    
    Shard7->>Shard7: GET stock → 3<br/>Check: 3 > 0 ✓<br/>DECR: 3 → 2
    Shard7-->>Service: SUCCESS (shard 7)
    
    Service-->>User: 200 OK<br/>{reservation_id}<br/>(3 tries, 2ms total)
    
    Note over User: Reserved from shard 7<br/>after 2 failed attempts
```

---

## 7. Idempotency Check Flow

**Flow:**

Shows how idempotency key prevents double-charging when network failure causes client retry.

**Steps:**
1. **Client generates** UUID idempotency_key
2. **First request:** POST /charge with key
3. **Server checks** idempotency store (key not found)
4. **Server inserts** key with status=PROCESSING
5. **Call Stripe API** (charge customer)
6. **Network timeout!** Client doesn't receive response
7. **Client retries** (same idempotency key)
8. **Server checks** idempotency store (key exists, status=SUCCESS)
9. **Server returns** cached response (no Stripe call)
10. **Result:** Customer charged exactly once

**Performance:**
- First request: 500ms-2s (Stripe latency)
- Retry: <1ms (cached response)
- Cache hit rate: ~5% (network failures)

**Benefits:**
- Exactly-once semantics
- Network fault tolerance
- No double-charge risk

**Trade-offs:**
- Storage overhead (24h cache)
- Additional Redis lookup
- Key management complexity

```mermaid
sequenceDiagram
    participant Client
    participant Service as Payment Service
    participant IdempotencyStore as Idempotency Store<br/>(Redis)
    participant Stripe
    
    Note over Client: Generate UUID<br/>key = abc-123
    
    Client->>Service: POST /charge<br/>idempotency_key: abc-123<br/>amount: 999
    
    Service->>IdempotencyStore: GET abc-123
    IdempotencyStore-->>Service: Key not found
    
    Service->>IdempotencyStore: SET abc-123<br/>status: PROCESSING
    
    Service->>Stripe: POST /charges<br/>amount: 999
    
    Note over Stripe: Processing...<br/>(500ms-2s)
    
    Stripe-->>Service: {charge_id: ch_123,<br/>status: succeeded}
    
    Service->>IdempotencyStore: SET abc-123<br/>status: SUCCESS<br/>response: {charge_id}
    
    Note over Client,Service: Network timeout!<br/>Response lost
    
    Note over Client: No response received<br/>Retry request
    
    Client->>Service: POST /charge<br/>idempotency_key: abc-123<br/>(RETRY - same key)
    
    Service->>IdempotencyStore: GET abc-123
    IdempotencyStore-->>Service: Found!<br/>status: SUCCESS<br/>response: {charge_id}
    
    Service-->>Client: 200 OK<br/>{charge_id: ch_123}<br/>(from cache, <1ms)
    
    Note over Client: Success!<br/>Customer charged once
```

---

## 8. Rate Limiting Multi-Layer Flow

**Flow:**

Shows complete multi-layer rate limiting from CDN to service layer, progressively reducing traffic.

**Steps:**
1. **1M users** send requests simultaneously
2. **CDN Layer:** Token bucket (10 req/sec per IP) → 900K blocked
3. **100K pass** to API Gateway
4. **Gateway Layer:** Leaky bucket queue (max 100K) → 90K queued
5. **Gateway drains** 10K QPS to service
6. **Service Layer:** Concurrency limit (10K max) → Processes requests
7. **Result:** 100 succeed, 9,900 sold out, 990K rejected

**Performance:**
- CDN blocking: <1ms (instant rejection)
- Queue wait: 0-30 seconds (position dependent)
- Service processing: 50ms (reservation)

**Benefits:**
- Cost optimization (block at cheap CDN)
- System protection (no overload)
- Fair access (FIFO queue)

**Trade-offs:**
- High rejection rate (99%)
- Queue timeouts (frustrating)
- False positives possible

```mermaid
sequenceDiagram
    participant Users as 1M Users
    participant CDN as CDN Edge<br/>Token Bucket
    participant Gateway as API Gateway<br/>Leaky Bucket
    participant Service as Reservation Service
    participant Redis
    
    Users->>CDN: 1,000,000 requests<br/>(T=0s)
    
    CDN->>CDN: Token Bucket Check<br/>10 req/sec per IP
    
    CDN-->>Users: 900,000 rejected<br/>429 Too Many Requests<br/>(<1ms each)
    
    CDN->>Gateway: 100,000 forwarded
    
    Gateway->>Gateway: Leaky Bucket Queue<br/>Max 100K, Drain 10K/sec
    
    Note over Gateway: 10K processed immediately<br/>90K queued (0-9s wait)
    
    Gateway->>Service: 10,000 QPS<br/>(sustained rate)
    
    Service->>Redis: EVAL check_and_decr<br/>(atomic)
    
    Redis-->>Service: 100 SUCCESS<br/>9,900 SOLD_OUT
    
    Service-->>Gateway: 100 OK<br/>9,900 Gone
    
    Gateway-->>Users: Return responses
    
    Note over Gateway: After 30s timeout
    
    Gateway-->>Users: 80K timeout<br/>408 Request Timeout<br/>(queue wait exceeded)
    
    Note over Users: Final result:<br/>100 success (0.01%)<br/>9,900 sold out (0.99%)<br/>980K blocked (98%)<br/>80K timeout (1%)
```

---

## 9. Redis Failover Flow

**Flow:**

Shows automatic failover when Redis master crashes, Sentinel promotes replica, and service resumes.

**Steps:**
1. **T=0:** Flash sale active, 50 items sold
2. **T=30s:** Redis master crashes (hardware failure)
3. **T=30-33s:** Client requests timeout (no response)
4. **T=33s:** Sentinel detects failure (heartbeat timeout)
5. **T=33-35s:** Sentinels vote (quorum 2/3)
6. **T=35s:** Replica promoted to master
7. **T=36s:** Clients reconnect to new master
8. **T=36s+:** Service resumes, remaining 50 items sold

**Performance:**
- Failure detection: 3 seconds (heartbeat)
- Failover time: 3 seconds (election + promotion)
- Total downtime: 6 seconds
- Data loss: 0-5 seconds of writes (depends on replication lag)

**Benefits:**
- Automatic recovery (no manual intervention)
- Minimal downtime (6 seconds)
- No lost reservations (queued)

**Trade-offs:**
- 6-second service interruption
- Potential data loss (last 1-5 seconds)
- Requires Sentinel cluster

```mermaid
sequenceDiagram
    participant Client as Reservation Service
    participant Sentinel1
    participant Sentinel2
    participant Master as Redis Master
    participant Replica as Redis Replica
    
    Note over Master: T=0: Normal operation<br/>50 items sold
    
    Client->>Master: DECR iphone:stock
    Master-->>Client: OK (stock: 49)
    
    Master->>Replica: Async replication
    
    Note over Master: T=30s: Hardware failure!<br/>Master crashes
    
    Client->>Master: DECR iphone:stock
    Note over Client: Timeout (3s)
    
    Sentinel1->>Master: Heartbeat ping
    Note over Sentinel1: No response
    
    Sentinel1->>Sentinel1: T=33s: Master down!<br/>Start failover
    
    Sentinel1->>Sentinel2: Vote: Promote replica?
    Sentinel2-->>Sentinel1: Vote: Yes (quorum 2/3)
    
    Note over Sentinel1,Sentinel2: T=34-35s: Election complete
    
    Sentinel1->>Replica: SLAVEOF NO ONE<br/>Become master
    
    Replica->>Replica: Promote to master
    Note over Replica: Now accepting writes
    
    Sentinel1->>Client: Config update:<br/>New master IP
    
    Client->>Replica: Reconnect<br/>DECR iphone:stock
    Replica-->>Client: OK (stock: 48)
    
    Note over Client,Replica: T=36s: Service resumed<br/>6 seconds total downtime<br/>Flash sale continues
```

---

## 10. Bot Detection and CAPTCHA Challenge

**Flow:**

Shows bot detection pipeline with fingerprinting, behavioral analysis, and CAPTCHA challenge.

**Steps:**
1. **Request arrives** from suspicious IP
2. **Stage 1:** Rate limit check (100+ req/min) → Suspicious
3. **Stage 2:** Browser fingerprint (headless browser detected)
4. **ML model scores** behavior (bot score: 85/100)
5. **Trigger CAPTCHA** (score > 80)
6. **User solves** CAPTCHA challenge
7. **CAPTCHA fails** (bot script can't solve)
8. **Block request** with 403 Forbidden (1-hour ban)

**Detection Signals:**
- Rate: 100+ req/min (humans: <10)
- Fingerprint: Missing WebGL, Canvas
- Behavior: No mouse movement, instant clicks
- Session: Fresh account (<1 day old)

**Performance:**
- Detection: <10ms per request
- CAPTCHA: 2-5 seconds (user time)
- Block duration: 1 hour

**Benefits:**
- 90% bot traffic blocked
- Fair human access
- Scalper prevention

**Trade-offs:**
- False positives (1%)
- CAPTCHA friction
- Sophisticated bots may bypass

```mermaid
sequenceDiagram
    participant Bot as Bot Script
    participant CDN as CDN Edge
    participant Gateway as API Gateway
    participant Detector as Bot Detector<br/>(ML Model)
    participant CAPTCHA as CAPTCHA Service
    
    Bot->>CDN: POST /reserve<br/>(100th request in 1 min)
    
    CDN->>CDN: Stage 1: Rate Check<br/>100 req/min > 10 threshold
    Note over CDN: Suspicious! Flag request
    
    CDN->>Gateway: Forward (flagged)
    
    Gateway->>Detector: Stage 2: Fingerprint<br/>Check User-Agent, WebGL
    
    Detector->>Detector: Missing WebGL ✗<br/>Missing Canvas ✗<br/>No mouse events ✗
    Note over Detector: Headless browser!
    
    Detector->>Detector: Stage 3: ML Model<br/>Behavioral analysis
    
    Detector-->>Gateway: Bot Score: 85/100<br/>(Likely bot)
    
    Note over Gateway: Score > 80<br/>Trigger CAPTCHA
    
    Gateway->>CAPTCHA: Challenge request
    CAPTCHA-->>Gateway: CAPTCHA image
    
    Gateway-->>Bot: 403 Forbidden<br/>CAPTCHA challenge required
    
    Bot->>Bot: Attempt to solve<br/>(automated script)
    
    Bot->>Gateway: CAPTCHA response<br/>(incorrect)
    
    Gateway->>CAPTCHA: Verify response
    CAPTCHA-->>Gateway: Failed
    
    Gateway-->>Bot: 403 Forbidden<br/>Blocked for 1 hour
    
    Note over Bot: Bot blocked<br/>Human users proceed
```

---

## 11. Kafka Consumer Lag Recovery

**Flow:**

Shows recovery process when payment service crashes and Kafka consumer lag builds up.

**Steps:**
1. **T=0:** Normal operation, 0 lag
2. **T=60s:** Payment service crashes (OOM error)
3. **T=60-120s:** Messages pile up (lag: 5000)
4. **T=120s:** Alert triggers (lag > 1000)
5. **T=125s:** Auto-scaling triggers (5 → 20 pods)
6. **T=130s:** New consumers start processing backlog
7. **T=180s:** Lag cleared (back to 0)
8. **Impact:** Reservations delayed 2-3 minutes (TTL extended)

**Performance:**
- Detection: 60 seconds (monitoring lag)
- Scaling: 5 seconds (Kubernetes auto-scale)
- Recovery: 60 seconds (process 5000 messages)
- Total impact: 2-3 minutes delayed payments

**Benefits:**
- Automatic recovery (no manual intervention)
- No lost messages (Kafka durability)
- Auto-scaling handles burst

**Trade-offs:**
- Temporary delays (2-3 minutes)
- Increased costs (20 pods vs 5)
- TTL extension needed

```mermaid
sequenceDiagram
    participant Kafka
    participant Consumer1 as Payment Consumer<br/>(Pod 1)
    participant Monitor as Monitoring<br/>(Prometheus)
    participant AutoScaler as Kubernetes<br/>Auto-Scaler
    participant NewConsumers as New Payment Pods<br/>(Pods 2-20)
    
    Note over Consumer1: T=0: Normal operation<br/>Lag: 0
    
    Kafka->>Consumer1: Process messages<br/>100/sec
    Consumer1->>Consumer1: Payment processing
    
    Note over Consumer1: T=60s: Pod crashes!<br/>Out of memory (OOM)
    
    Kafka->>Kafka: Messages pile up<br/>(no consumers)
    
    Note over Monitor: T=60-120s<br/>Lag increases: 0 → 5000
    
    Monitor->>Monitor: T=120s: Alert!<br/>Lag > 1000 threshold
    
    Monitor->>AutoScaler: Trigger scale-up<br/>Target: 20 pods
    
    AutoScaler->>AutoScaler: T=125s: Launch pods<br/>2-20 (18 new pods)
    
    AutoScaler->>NewConsumers: Start pods
    
    NewConsumers->>Kafka: T=130s: Join consumer group<br/>Rebalance partitions
    
    Note over Kafka,NewConsumers: 20 consumers processing<br/>2000 messages/sec
    
    Kafka->>NewConsumers: Distribute 5000 backlog<br/>(250 messages each)
    
    NewConsumers->>NewConsumers: Process payments<br/>(30 seconds)
    
    Note over Monitor: T=180s: Lag cleared<br/>Back to 0
    
    Monitor->>AutoScaler: Scale down to 5 pods<br/>(normal load)
    
    Note over Kafka,NewConsumers: Recovery complete<br/>60 seconds total<br/>2-3 min payment delay
```

---

## 12. Complete End-to-End Purchase Flow

**Flow:**

Shows complete journey from user clicking "Buy Now" to receiving order confirmation, including all systems.

**Steps:**
1. **User clicks** "Buy Now" (T=0ms)
2. **CDN + API Gateway:** Rate limiting, authentication (T=10ms)
3. **Reservation Service:** Generate UUID, atomic Redis DECR (T=50ms)
4. **User receives** reservation confirmation (T=50ms - fast!)
5. **Kafka:** Publish reservation event (async, T=100ms)
6. **Payment Saga:** Consume event, start processing (T=1s)
7. **Stripe API:** Charge customer (T=2-3s)
8. **Create Order:** PostgreSQL INSERT (T=3s)
9. **Send Notification:** Email + push notification (T=5s)
10. **User receives** order confirmation (T=5s total)

**Performance:**
- User feedback: 50ms (reservation)
- Payment processing: 2-5 seconds (async)
- Order confirmation: 5-10 seconds total

**Benefits:**
- Fast user experience (50ms)
- Async payment (doesn't block)
- Reliable (all systems involved)

**Trade-offs:**
- Complex (8 systems)
- Eventual consistency
- Cost (~$0.10 per transaction)

```mermaid
sequenceDiagram
    participant User
    participant CDN
    participant Gateway as API Gateway
    participant ReservationSvc as Reservation Service
    participant Redis
    participant Kafka
    participant PaymentSaga as Payment Saga
    participant Stripe
    participant OrderDB as Order DB
    participant Notification as Notification Service
    
    User->>CDN: Click Buy Now<br/>T=0ms
    CDN->>Gateway: Forward (rate limit OK)<br/>T=10ms
    Gateway->>ReservationSvc: POST /reserve<br/>T=20ms
    
    ReservationSvc->>ReservationSvc: Generate UUID<br/>idempotency_key
    ReservationSvc->>Redis: EVAL check_and_decr<br/>T=25ms
    Redis-->>ReservationSvc: SUCCESS (stock: 99)<br/>T=26ms
    
    ReservationSvc->>ReservationSvc: Create reservation record<br/>T=35ms
    ReservationSvc-->>User: 200 OK<br/>{reservation_id}<br/>T=50ms ✅
    
    Note over User: Success! 50ms response<br/>User has 10 min to pay
    
    ReservationSvc->>Kafka: Publish event<br/>T=100ms (async)
    
    Kafka->>PaymentSaga: Pull event<br/>T=1000ms
    
    PaymentSaga->>Stripe: POST /charges<br/>T=1500ms
    Note over Stripe: Processing...<br/>(500ms-2s)
    Stripe-->>PaymentSaga: charge_id<br/>T=3000ms
    
    PaymentSaga->>OrderDB: INSERT order<br/>T=3100ms
    OrderDB-->>PaymentSaga: order_id<br/>T=3150ms
    
    PaymentSaga->>Notification: Send email + push<br/>T=3200ms
    Notification-->>User: Order confirmed!<br/>T=5000ms ✅
    
    Note over User: Total: 5 seconds<br/>Reservation: 50ms<br/>Order confirmation: 5s
```

