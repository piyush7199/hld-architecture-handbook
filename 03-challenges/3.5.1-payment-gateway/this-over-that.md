# Payment Gateway - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made when designing a payment gateway like
Stripe or PayPal.

---

## Table of Contents

1. [PostgreSQL vs NoSQL for Ledger](#1-postgresql-vs-nosql-for-ledger)
2. [Two-Phase Commit vs Single-Phase](#2-two-phase-commit-vs-single-phase)
3. [Idempotency Keys vs Request Deduplication](#3-idempotency-keys-vs-request-deduplication)
4. [Tokenization vs Direct Card Storage](#4-tokenization-vs-direct-card-storage)
5. [Redis vs Memcached for Caching](#5-redis-vs-memcached-for-caching)
6. [Synchronous vs Asynchronous Webhooks](#6-synchronous-vs-asynchronous-webhooks)
7. [Sharding vs Vertical Scaling](#7-sharding-vs-vertical-scaling)
8. [Circuit Breaker vs Simple Retry](#8-circuit-breaker-vs-simple-retry)
9. [Real-Time Fraud Detection vs Batch Processing](#9-real-time-fraud-detection-vs-batch-processing)
10. [Multi-Region vs Single-Region Deployment](#10-multi-region-vs-single-region-deployment)

---

## 1. PostgreSQL vs NoSQL for Ledger

### The Problem

Store 620 billion financial transactions with ACID guarantees, auditability, and reconciliation capabilities.

### Options Considered

| Feature                    | PostgreSQL                   | Cassandra                  | MongoDB                 | DynamoDB                |
|----------------------------|------------------------------|----------------------------|-------------------------|-------------------------|
| **ACID Transactions**      | ✅ Full ACID                  | ❌ Eventual consistency     | ⚠️ Limited transactions | ⚠️ Limited transactions |
| **Multi-Row Transactions** | ✅ Yes                        | ❌ Single partition only    | ✅ Yes (4.0+)            | ⚠️ Max 25 items         |
| **Referential Integrity**  | ✅ Foreign keys               | ❌ Application-level        | ❌ Application-level     | ❌ Application-level     |
| **Query Flexibility**      | ✅ Full SQL, JOINs            | ❌ Limited to partition key | ✅ Rich queries          | ❌ Key-value lookups     |
| **Auditability**           | ✅ WAL, triggers              | ⚠️ Harder                  | ⚠️ Harder               | ⚠️ Harder               |
| **Scaling**                | ⚠️ Vertical, sharding needed | ✅ Horizontal               | ✅ Horizontal            | ✅ Auto-scaling          |
| **Financial Industry Use** | ✅ Standard                   | ❌ Rare                     | ⚠️ Growing              | ⚠️ Some use             |

### Decision Made

**PostgreSQL with Horizontal Sharding (64 shards by merchant_id)**

### Rationale

1. **ACID Requirements**: Financial transactions require atomicity (all-or-nothing), consistency (referential
   integrity), isolation (no dirty reads), and durability (crash recovery)
2. **Regulatory Compliance**: Auditors expect SQL databases with transaction logs
3. **Proven Track Record**: Banks, payment processors, and financial institutions use PostgreSQL
4. **Query Flexibility**: Merchants need complex reports (revenue by date, refund rates, etc.)
5. **Data Integrity**: Foreign key constraints prevent orphaned records

**Why NOT Cassandra**:

- Eventual consistency unacceptable for money (can't have "eventually correct" balance)
- No multi-row transactions (can't atomically update payment + refund)
- Harder to audit and reconcile with bank statements

**Why NOT MongoDB**:

- Transactions added recently (4.0), less battle-tested for finance
- Document model doesn't fit relational payment data well
- Smaller ecosystem for financial tools

**Why NOT DynamoDB**:

- Transaction limitations (max 25 items per transaction)
- Eventual consistency by default
- Vendor lock-in (AWS only)

### Implementation Details

**Sharding Strategy:**

```
Shard Calculation:
  shard_id = hash(merchant_id) % 64

Benefits:
  - All merchant data on same shard (fast queries)
  - Even distribution across shards
  - 300 writes/sec per shard (total 19,200 writes/sec)

Replication:
  - Primary + 2 read replicas per shard
  - Synchronous replication to 1 replica (RPO = 0)
  - Async replication to 2nd replica
```

**Connection Pooling:**

```
Settings:
  - Min connections: 50 per instance
  - Max connections: 200 per instance
  - 60 instances × 200 = 12,000 total connections
  - 12,000 / 64 shards = 187 connections per shard
  - PostgreSQL max_connections: 500 (safe margin)
```

### Trade-offs Accepted

| What We Gain                             | What We Sacrifice                              |
|------------------------------------------|------------------------------------------------|
| ✅ ACID guarantees (financial integrity)  | ❌ Harder to scale horizontally (need sharding) |
| ✅ Regulatory compliance (auditors happy) | ❌ More complex operations (manage 64 shards)   |
| ✅ Query flexibility (complex reports)    | ❌ Cross-shard queries expensive                |
| ✅ Data integrity (foreign keys)          | ❌ Schema migrations across shards complex      |

### When to Reconsider

**Switch to NoSQL if:**

- Scale exceeds 100k writes/sec (would need 300+ shards)
- Don't need ACID (non-financial use case)
- Need global distribution with multi-master writes

**Switch to NewSQL (CockroachDB, YugabyteDB) if:**

- Want distributed SQL with automatic sharding
- Need global multi-region writes
- Can tolerate higher latency (distributed consensus)

### Real-World Examples

- **Stripe**: PostgreSQL (sharded by merchant)
- **PayPal**: Oracle historically, now distributed SQL
- **Square**: MySQL (similar to PostgreSQL)
- **Adyen**: PostgreSQL with heavy replication

---

## 2. Two-Phase Commit vs Single-Phase

### The Problem

Should payments be processed in one step (charge immediately) or two steps (authorize then capture)?

### Options Considered

| Approach                            | Merchant Flexibility | Refund Rate                | Complexity      | Use Case        |
|-------------------------------------|----------------------|----------------------------|-----------------|-----------------|
| **Two-Phase** (Auth + Capture)      | ✅ High (can cancel)  | ✅ Lower (cancel vs refund) | ⚠️ More complex | ✅ E-commerce    |
| **Single-Phase** (Charge immediate) | ❌ Low (must refund)  | ❌ Higher                   | ✅ Simple        | ✅ Digital goods |

### Decision Made

**Two-Phase Commit (Authorization → Capture)**

### Rationale

1. **Merchant Flexibility**: Merchants can cancel orders before fulfillment (out of stock, fraud detected)
2. **Reduce Refunds**: Canceling authorization doesn't incur refund fees ($0.30 per refund)
3. **Inventory Checks**: Authorize first, capture only if item available
4. **Fraud Window**: Time to review suspicious orders before charging
5. **Industry Standard**: Visa/Mastercard networks support two-phase

**Why NOT Single-Phase**:

- No way to cancel (must issue refund)
- Higher chargeback risk (customer changed mind)
- Inventory mismatch (charge but can't fulfill)

### Implementation Details

**Authorization:**

```
POST /payments
{
  "amount": 10000,  // $100.00
  "currency": "usd",
  "token": "tok_abc123",
  "idempotency_key": "idem_xyz"
}

Response:
{
  "payment_id": "pay_123",
  "status": "authorized",
  "auth_code": "ABC123",
  "expires_at": "2024-01-08"  // 7 days
}

Bank Action:
- Soft hold on customer account
- Customer sees "pending" charge
- No money transferred yet
```

**Capture (hours/days later):**

```
POST /payments/pay_123/capture

Response:
{
  "payment_id": "pay_123",
  "status": "captured",
  "settlement_id": "XYZ789"
}

Bank Action:
- Transfer money from customer to merchant
- Customer sees final charge
- Money settled
```

**Cancel (if needed):**

```
POST /payments/pay_123/cancel

Response:
{
  "payment_id": "pay_123",
  "status": "canceled"
}

Bank Action:
- Release hold
- Customer never charged
- No refund needed (saves $0.30 fee)
```

### Trade-offs Accepted

| What We Gain                                   | What We Sacrifice                          |
|------------------------------------------------|--------------------------------------------|
| ✅ Merchant flexibility (cancel before capture) | ❌ Two API calls instead of one             |
| ✅ Lower refund rate                            | ❌ Must track authorization expiry (7 days) |
| ✅ Fraud review window                          | ❌ More complex state machine               |
| ✅ Inventory verification                       | ❌ Capture can fail (authorization expired) |

### When to Reconsider

**Use Single-Phase if:**

- Digital goods (no inventory, instant delivery)
- Subscription billing (recurring charges)
- Micropayments (< $1, complexity not worth it)
- Immediate delivery required

### Real-World Examples

- **Stripe**: Two-phase (authorize + capture)
- **PayPal**: Two-phase (authorize + capture)
- **Square**: Two-phase for online, single-phase for in-person
- **Spotify**: Single-phase (subscription, digital goods)

---

## 3. Idempotency Keys vs Request Deduplication

### The Problem

Network failures cause clients to retry requests. How do we prevent charging customers twice?

### Options Considered

| Approach                                    | Reliability                   | Performance                 | Client Complexity            | Cost                 |
|---------------------------------------------|-------------------------------|-----------------------------|------------------------------|----------------------|
| **Idempotency Keys** (Client-provided UUID) | ✅ Excellent (100% guaranteed) | ✅ Fast (Redis lookup)       | ⚠️ Client must generate UUID | ✅ Low (Redis)        |
| **Server-Side Dedup** (Fingerprint request) | ⚠️ Good (95% reliable)        | ⚠️ Slower (hash all fields) | ✅ Simple (no change)         | ⚠️ Medium            |
| **No Protection**                           | ❌ Poor (duplicate charges)    | ✅ Fastest                   | ✅ Simplest                   | ❌ High (chargebacks) |

### Decision Made

**Idempotency Keys (Client-Provided UUID)**

### Rationale

1. **100% Guarantee**: Client controls uniqueness (server can't guess)
2. **Fast Lookup**: Redis GET by key (1ms vs 10ms database query)
3. **Industry Standard**: Stripe, PayPal, Square all use this approach
4. **Explicit Intent**: Different keys = different payments (not accidental retry)
5. **Debugging**: Can track retries by idempotency key

**Why NOT Server-Side Dedup**:

- Can't distinguish intentional duplicate (customer buys 2 items) from accidental retry
- Hash collisions possible (different requests, same hash)
- Requires hashing all request fields (slow, error-prone)

**Why NOT No Protection**:

- Network failures common (5% of requests timeout)
- Double charges cause chargebacks (costs $15-$25 + losing customer)
- Damages reputation

### Implementation Details

**Client Side:**

```
import uuid

# Generate unique key for each payment
idempotency_key = str(uuid.uuid4())  // "idem_abc123"

# Retry-safe request
response = requests.post(
    "https://api.gateway.com/payments",
    json={
        "amount": 10000,
        "token": "tok_xyz",
        "idempotency_key": idempotency_key  // Required!
    }
)

# If network fails, retry with SAME key
if response.status_code == 500:
    response = requests.post(..., json={..., "idempotency_key": idempotency_key})
    # Server returns cached result (no double charge)
```

**Server Side:**

```
1. Request arrives: {amount, token, idempotency_key}

2. Check cache:
   result = redis.get(f"idempotency:{merchant_id}:{idempotency_key}")
   
3. If cache hit:
   return result  // Same response, no processing

4. If cache miss:
   a. Process payment
   b. Cache result: redis.setex(key, 86400, result)  // 24h TTL
   c. Return result

5. Scope: Per merchant (merchant_A and merchant_B can use same key)
```

**Key Format:**

```
idempotency:{merchant_id}:{idempotency_key}

Example:
  idempotency:merch_123:idem_abc456

TTL: 24 hours
  - Long enough for retries
  - Short enough to not waste memory
  - After expiry, same key can be reused (merchant should use new key)
```

### Trade-offs Accepted

| What We Gain                          | What We Sacrifice                        |
|---------------------------------------|------------------------------------------|
| ✅ 100% duplicate prevention           | ❌ Client must generate UUIDs             |
| ✅ Fast performance (Redis 1ms)        | ❌ Must maintain Redis cluster            |
| ✅ Explicit intent (keys show retries) | ❌ Key expiry complexity (24h)            |
| ✅ Industry standard                   | ❌ Increased API payload size (+32 bytes) |

### When to Reconsider

**Use Server-Side Dedup if:**

- Can't change client code (legacy systems)
- Request uniqueness defined by business logic (order_id)
- Idempotency less critical (non-financial operations)

**Skip Idempotency if:**

- Read-only operations (GET /payments/{id})
- Inherently idempotent operations (set status to X)
- Not financially critical

### Real-World Examples

- **Stripe**: Required idempotency_key for POST/PATCH/DELETE
- **PayPal**: Uses PayPal-Request-Id header
- **Square**: Uses idempotency_key parameter
- **AWS**: Uses ClientRequestToken for financial APIs

---

## 4. Tokenization vs Direct Card Storage

### The Problem

Storing credit card numbers requires expensive PCI-DSS Level 1 compliance ($1M+ annually). How do we handle card data
securely?

### Options Considered

| Approach                          | PCI Scope               | Compliance Cost         | Security                 | Flexibility            |
|-----------------------------------|-------------------------|-------------------------|--------------------------|------------------------|
| **Tokenization**                  | ✅ Minimal (one service) | ✅ Low ($50k-$100k)      | ✅ High (cards encrypted) | ✅ High                 |
| **Direct Storage**                | ❌ Entire system         | ❌ Very high ($1M+)      | ⚠️ Medium (if encrypted) | ✅ High                 |
| **Outsource** (Stripe, Braintree) | ✅ Zero (vendor handles) | ✅ Very low (pay per tx) | ✅ High                   | ❌ Low (vendor lock-in) |

### Decision Made

**Tokenization (In-House)**

### Rationale

1. **Reduce PCI Scope**: Only Tokenization Service stores cards (rest of system out of scope)
2. **Cost Savings**: $50k-$100k compliance vs $1M+ for entire system
3. **Security**: Cards encrypted with AES-256, keys in HSM
4. **Flexibility**: Full control over data, no vendor lock-in
5. **Competitive Advantage**: Can offer better rates than outsourcing

**Why NOT Direct Storage**:

- Every system that touches cards is in PCI scope
- Compliance audits cover entire infrastructure ($1M+ annually)
- Higher risk (breach = massive liability)
- Slows development (every change requires PCI review)

**Why NOT Outsource**:

- Vendor fees higher (2.9% + $0.30 vs 0.5% internal)
- Vendor lock-in (hard to migrate)
- Less control (can't customize)
- Data sovereignty issues (vendor may store abroad)

### Implementation Details

**Tokenization Flow:**

```
1. Client: POST /tokens
   {
     "card_number": "4111111111111111",
     "exp_month": 12,
     "exp_year": 2025,
     "cvc": "123"
   }

2. Tokenization Service (PCI Scope):
   a. Validate card (Luhn algorithm)
   b. Generate token: tok_abc123
   c. Create fingerprint: SHA-256(card_number)
   d. Encrypt card: AES-256(card_number, key_from_HSM)
   e. Store in database:
      tokens[tok_abc123] = {
        fingerprint: "abc123...",
        encrypted_card: "xyz789...",
        last4: "1111",
        brand: "Visa"
      }
   f. Return token

3. Client: POST /payments
   {
     "amount": 10000,
     "token": "tok_abc123"  // No raw card!
   }

4. Payment Service (NOT in PCI scope):
   - Never sees raw card
   - Only handles tokens
   - Fetches encrypted card from Tokenization Service when needed
```

**Security Layers:**

```
Layer 1: Network Isolation
  - Tokenization Service in private VPC
  - No internet access
  - Only accessible from Payment Service

Layer 2: Encryption at Rest
  - AES-256 encryption
  - Keys stored in HSM (Hardware Security Module)
  - Key rotation every 30 days

Layer 3: Encryption in Transit
  - TLS 1.3 for all communication
  - Mutual TLS (mTLS) between services
  - Certificate pinning

Layer 4: Access Control
  - Service accounts only (no human access)
  - All access logged to immutable audit log
  - Require 2-person approval for key operations

Layer 5: Compliance
  - Annual PCI-DSS audit (QSA)
  - Quarterly vulnerability scans (ASV)
  - Penetration testing
```

### Trade-offs Accepted

| What We Gain                                      | What We Sacrifice                |
|---------------------------------------------------|----------------------------------|
| ✅ 90% reduction in PCI scope                      | ❌ Additional service to maintain |
| ✅ $900k annual compliance savings                 | ❌ Tokenization Service is SPOF   |
| ✅ Faster development (most services out of scope) | ❌ Token-to-card lookup adds 10ms |
| ✅ Better security (isolation)                     | ❌ Must manage HSM, key rotation  |

### When to Reconsider

**Use Direct Storage if:**

- Already have PCI-DSS certification
- Very small scale (< $1M revenue)
- Tokenization overhead unacceptable

**Outsource to Stripe/Braintree if:**

- Small business (< $100k revenue)
- Fast time-to-market critical
- Don't want compliance burden
- Can accept higher fees

### Real-World Examples

- **Stripe**: In-house tokenization (competitive advantage)
- **PayPal**: In-house tokenization
- **Square**: In-house tokenization
- **Small merchants**: Outsource to Stripe/Braintree

---

## 5. Redis vs Memcached for Caching

### The Problem

Cache idempotency keys, token metadata, and fraud features for fast lookup.

### Options Considered

| Feature               | Redis                                 | Memcached               |
|-----------------------|---------------------------------------|-------------------------|
| **Data Structures**   | ✅ String, Hash, Set, Sorted Set, List | ❌ Key-Value only        |
| **Persistence**       | ✅ Optional (RDB, AOF)                 | ❌ In-memory only        |
| **Replication**       | ✅ Master-slave                        | ⚠️ Limited              |
| **Atomic Operations** | ✅ INCR, SETNX, etc.                   | ⚠️ Limited              |
| **Pub/Sub**           | ✅ Yes                                 | ❌ No                    |
| **Performance**       | ✅ 100k ops/sec per node               | ✅ 200k ops/sec per node |
| **Memory Efficiency** | ⚠️ Moderate                           | ✅ Better (simpler)      |

### Decision Made

**Redis**

### Rationale

1. **Rich Data Structures**: Need Hash for idempotency (multiple fields), Set for deduplication
2. **Atomic Operations**: SETNX for idempotency check-and-set (race condition safe)
3. **Replication**: Master-slave for high availability
4. **Persistence**: Optional AOF for critical data (though we primarily use as cache)
5. **Ecosystem**: Better monitoring tools, client libraries

**Why NOT Memcached**:

- Key-value only (would need to serialize/deserialize objects)
- No atomic SETNX (race condition when checking idempotency)
- No persistence (lose everything on restart)
- Limited replication

### Implementation Details

**Idempotency Storage:**

```
Key: idempotency:{merchant_id}:{idempotency_key}
Value: JSON {
  payment_id,
  status,
  amount_cents,
  response_payload
}
TTL: 86400 seconds (24 hours)

Operation: SETEX (set with expiry)
```

**Token Metadata:**

```
Key: token:{token_id}
Value: Hash {
  card_brand: "Visa",
  last4: "1111",
  exp_month: 12,
  exp_year: 2025,
  merchant_id: "merch_123"
}
TTL: 3600 seconds (1 hour)

Operation: HGETALL (get all hash fields)
```

**Rate Limiting:**

```
Key: rate_limit:{merchant_id}:second
Value: Integer (request count)
TTL: 1 second

Operation: INCR (atomic increment)
```

### Trade-offs Accepted

| What We Gain                       | What We Sacrifice                |
|------------------------------------|----------------------------------|
| ✅ Rich data structures (Hash, Set) | ❌ Slightly slower than Memcached |
| ✅ Atomic operations (SETNX)        | ❌ More memory usage              |
| ✅ Replication (high availability)  | ❌ More complex to operate        |
| ✅ Better tooling                   | ❌ Higher infrastructure cost     |

### When to Reconsider

**Use Memcached if:**

- Only need simple key-value (no complex data structures)
- Performance critical (need that extra 2x throughput)
- Memory constrained (Memcached more efficient)

### Real-World Examples

- **Stripe**: Redis for caching
- **PayPal**: Redis + Memcached (hybrid)
- **Square**: Redis
- **Twitter**: Memcached historically, migrating to Redis

---

## 6. Synchronous vs Asynchronous Webhooks

### The Problem

Should webhook delivery block the payment flow or be asynchronous?

### Options Considered

| Approach         | Payment Latency      | Reliability                           | Merchant Impact               | Complexity  |
|------------------|----------------------|---------------------------------------|-------------------------------|-------------|
| **Synchronous**  | ❌ High (blocks)      | ❌ Low (merchant down = payment fails) | ❌ High (must be always up)    | ✅ Simple    |
| **Asynchronous** | ✅ Low (non-blocking) | ✅ High (retry logic)                  | ✅ Low (can tolerate downtime) | ⚠️ Moderate |

### Decision Made

**Asynchronous Webhooks with Retry Logic**

### Rationale

1. **Non-Blocking**: Payment success doesn't depend on merchant endpoint
2. **Fault Tolerance**: Merchant endpoint down → payment still succeeds
3. **Retry Logic**: Up to 10 retries over 7 days (merchant will eventually get event)
4. **At-Least-Once Delivery**: Guaranteed delivery (merchant handles duplicates)
5. **Industry Standard**: Stripe, PayPal, Square all use async webhooks

**Why NOT Synchronous**:

- Merchant endpoint down → payment fails (terrible UX)
- Merchant endpoint slow → payment latency high
- Merchant can't tolerate downtime (must be 100% up)

### Implementation Details

**Flow:**

```
1. Payment Captured:
   - Write to database: status = captured
   - Return to customer: 200 OK (done!)
   - Publish event to Kafka: payment.captured

2. Webhook Worker:
   - Poll Kafka for events
   - For each event:
     a. Generate HMAC signature
     b. POST to merchant webhook_url
     c. If 200 OK → ACK Kafka message
     d. If error → Retry with backoff

3. Retry Schedule:
   - Attempt 1: Immediate
   - Attempt 2: 1 minute later
   - Attempt 3: 5 minutes later
   - Attempt 4: 30 minutes later
   - Attempt 5-10: 1h, 6h, 12h, 24h
   
4. After 10 Retries:
   - Move to Dead Letter Queue
   - Alert merchant (email)
   - Manual review required
```

**Signature Verification:**

```
Server:
  payload = json.dumps(event)
  signature = HMAC-SHA256(payload, webhook_secret)
  
  POST merchant_url
  Headers:
    X-Signature: sha256=abc123...
  Body:
    {event: payment.captured, ...}

Merchant:
  expected_sig = HMAC-SHA256(request.body, webhook_secret)
  if expected_sig != request.headers['X-Signature']:
    return 401 Unauthorized
```

### Trade-offs Accepted

| What We Gain                            | What We Sacrifice                     |
|-----------------------------------------|---------------------------------------|
| ✅ Non-blocking payment flow             | ❌ Merchant must handle async events   |
| ✅ Fault tolerant (merchant downtime OK) | ❌ Eventual consistency (slight delay) |
| ✅ Retry logic (guaranteed delivery)     | ❌ Must handle duplicate events        |
| ✅ Better customer UX                    | ❌ More complex (Kafka, workers)       |

### When to Reconsider

**Use Synchronous if:**

- Immediate confirmation required
- Merchant always available (SLA 99.99%+)
- Simple architecture preferred

### Real-World Examples

- **Stripe**: Async webhooks with retries
- **PayPal**: Async webhooks (IPN)
- **Square**: Async webhooks
- **GitHub**: Async webhooks

---

## 7. Sharding vs Vertical Scaling

### The Problem

Database hits write limit (5,000 writes/sec). How to scale beyond?

### Options Considered

| Approach                         | Max Throughput              | Cost                     | Complexity | Flexibility |
|----------------------------------|-----------------------------|--------------------------|------------|-------------|
| **Vertical Scaling** (Bigger DB) | ⚠️ Limited (10k writes/sec) | ❌ Expensive ($50k/month) | ✅ Simple   | ❌ Limited   |
| **Horizontal Sharding**          | ✅ Unlimited (add shards)    | ✅ Cheaper ($500/shard)   | ❌ Complex  | ✅ High      |
| **Read Replicas Only**           | ❌ No write scaling          | ✅ Cheap                  | ✅ Simple   | ❌ Read-only |

### Decision Made

**Horizontal Sharding (64 shards by merchant_id)**

### Rationale

1. **Linear Scaling**: Need 20k writes/sec → 64 shards × 300/sec = 19,200/sec
2. **Cost Effective**: 64 × $500 = $32k vs $50k for single large instance
3. **Future Proof**: Can add more shards as needed
4. **Shard Key**: merchant_id ensures all merchant data on same shard (fast queries)

**Why NOT Vertical Scaling**:

- Hit hardware limits (even largest DB maxes at ~10k writes/sec)
- Single point of failure
- More expensive at scale

**Why NOT Read Replicas Only**:

- Doesn't help with write bottleneck
- Only helps read-heavy workloads (we're write-heavy)

### Implementation Details

**Shard Calculation:**

```
shard_id = hash(merchant_id) % 64

Example:
  merchant_id = "merch_12345"
  hash("merch_12345") = 3829485762
  3829485762 % 64 = 34
  → Route to Shard 34
```

**Shard Map:**

```sql
CREATE TABLE shard_map (
    merchant_id VARCHAR(32) PRIMARY KEY,
    shard_id INT,
    shard_host VARCHAR(255)
);

Query:
  SELECT shard_host FROM shard_map WHERE merchant_id = 'merch_123'
  → db-shard-34.postgres.local
```

### Trade-offs Accepted

| What We Gain                       | What We Sacrifice                      |
|------------------------------------|----------------------------------------|
| ✅ Linear scalability (add shards)  | ❌ Cross-shard queries expensive        |
| ✅ Cost effective at scale          | ❌ Complex operations (64 databases)    |
| ✅ No single bottleneck             | ❌ Shard rebalancing painful            |
| ✅ Geographic distribution possible | ❌ Developer complexity (routing logic) |

### When to Reconsider

**Use Vertical Scaling if:**

- Scale < 5k writes/sec
- Simple operations preferred
- Budget allows

**Use NewSQL (CockroachDB) if:**

- Want automatic sharding
- Need global distribution
- Can tolerate higher latency

### Real-World Examples

- **Stripe**: Sharded PostgreSQL (by merchant)
- **Uber**: Sharded MySQL (by city)
- **Instagram**: Sharded PostgreSQL (by user_id)

---

## 8. Circuit Breaker vs Simple Retry

### The Problem

Bank API fails. How to prevent cascading failures?

### Options Considered

| Approach            | Latency (Bank Down)         | Resource Usage             | Recovery Time    | Complexity  |
|---------------------|-----------------------------|----------------------------|------------------|-------------|
| **Simple Retry**    | ❌ High (timeout each retry) | ❌ High (threads blocked)   | ❌ Slow           | ✅ Simple    |
| **Circuit Breaker** | ✅ Low (fail fast)           | ✅ Low (no blocked threads) | ✅ Fast           | ⚠️ Moderate |
| **No Retry**        | ✅ Low                       | ✅ Low                      | ❌ Never recovers | ✅ Simple    |

### Decision Made

**Circuit Breaker Pattern**

### Rationale

1. **Fail Fast**: When bank is down, return error immediately (no wasted timeouts)
2. **Resource Protection**: Don't exhaust thread pool waiting for bank
3. **Automatic Recovery**: Periodically test if bank recovered
4. **Cascading Failure Prevention**: Stop calling failing service

**Why NOT Simple Retry**:

- Wastes resources (threads blocked waiting)
- Slow (timeout on every retry: 3 retries × 10s = 30s latency)
- Doesn't adapt (keeps calling failing service)

**Why NOT No Retry**:

- Transient failures cause permanent errors
- Never recovers automatically

### Implementation Details

**States:**

```
CLOSED (Normal):
  - All requests pass through
  - Monitor error rate
  - If error rate > 50% for 10s → OPEN

OPEN (Bank Down):
  - Fail fast (no bank calls)
  - Return 503 Service Unavailable
  - After 60s cooldown → HALF-OPEN

HALF-OPEN (Testing):
  - Send probe request
  - If success → CLOSED
  - If failure → OPEN (reset cooldown)
```

### Trade-offs Accepted

| What We Gain                | What We Sacrifice                    |
|-----------------------------|--------------------------------------|
| ✅ Fast failure (no timeout) | ❌ Some requests fail during cooldown |
| ✅ Resource protection       | ❌ More complex logic                 |
| ✅ Automatic recovery        | ❌ Configuration tuning needed        |

### When to Reconsider

**Use Simple Retry if:**

- Service highly reliable (99.99% uptime)
- Low traffic (resource exhaustion not a concern)

### Real-World Examples

- **Netflix**: Hystrix (circuit breaker library)
- **AWS**: Built-in circuit breakers
- **Stripe**: Circuit breakers for bank APIs

---

## 9. Real-Time Fraud Detection vs Batch Processing

### The Problem

Detect fraudulent transactions to prevent chargebacks.

### Options Considered

| Approach                   | Detection Time | Fraud Prevention      | Cost         | Complexity  |
|----------------------------|----------------|-----------------------|--------------|-------------|
| **Real-Time** (Sync)       | ✅ < 50ms       | ✅ Block before charge | ❌ High (GPU) | ❌ Complex   |
| **Near Real-Time** (Async) | ⚠️ 5 seconds   | ⚠️ Block after charge | ⚠️ Medium    | ⚠️ Moderate |
| **Batch** (Daily)          | ❌ 24 hours     | ❌ Too late            | ✅ Low        | ✅ Simple    |

### Decision Made

**Real-Time Fraud Detection (Synchronous)**

### Rationale

1. **Block Before Charging**: Stop fraud before money moves (better UX, lower chargeback)
2. **Liability Shift**: Decline high-risk → no liability
3. **Lower Chargeback Rate**: Real-time blocking reduces chargebacks from 2% to 0.5%
4. **Competitive Advantage**: Lower fraud = lower fees for merchants

**Why NOT Batch Processing**:

- Too late (money already moved)
- Higher chargeback rate (costs $15-$25 per chargeback)
- Bad UX (customer charged then refunded)

### Implementation Details

**ML Pipeline:**

```
1. Feature Extraction (10ms):
   - Transaction: amount, currency, time
   - Card: brand, country, decline history
   - User: email, IP, device fingerprint

2. Feature Store Lookup (10ms):
   - Query Redis for historical features
   - Card velocity (transactions per hour)
   - User spending pattern

3. Model Inference (20ms):
   - Load XGBoost model
   - 30 features → risk score (0.0 to 1.0)
   - GPU-accelerated

4. Decision (10ms):
   - Risk < 0.5 → Approve
   - Risk 0.5-0.8 → 3D Secure challenge
   - Risk > 0.8 → Block

Total: 50ms (part of 300ms authorization)
```

### Trade-offs Accepted

| What We Gain                         | What We Sacrifice           |
|--------------------------------------|-----------------------------|
| ✅ Block fraud before charge          | ❌ Adds 50ms latency         |
| ✅ Lower chargeback rate (0.5% vs 2%) | ❌ Expensive (GPU instances) |
| ✅ Better UX (no surprise refunds)    | ❌ Complex ML pipeline       |

### When to Reconsider

**Use Batch if:**

- Low fraud rate (< 0.1%)
- Latency critical (can't afford 50ms)
- Small scale (< 1k transactions/day)

### Real-World Examples

- **Stripe**: Real-time (Radar)
- **PayPal**: Real-time ML models
- **Square**: Real-time detection

---

## 10. Multi-Region vs Single-Region Deployment

### The Problem

Serve global customers with low latency and high availability.

### Options Considered

| Approach                          | Latency (Global)    | Availability | Cost                    | Complexity  |
|-----------------------------------|---------------------|--------------|-------------------------|-------------|
| **Single Region** (US-East)       | ❌ High (200ms Asia) | ⚠️ 99.9%     | ✅ Low ($100k/month)     | ✅ Simple    |
| **Multi-Region** (Active-Passive) | ⚠️ Medium (100ms)   | ✅ 99.95%     | ⚠️ Medium ($150k/month) | ⚠️ Moderate |
| **Multi-Region** (Active-Active)  | ✅ Low (20ms)        | ✅ 99.99%     | ❌ High ($200k/month)    | ❌ Complex   |

### Decision Made

**Multi-Region Active-Active (US-East, US-West, EU-West, AP-Southeast)**

### Rationale

1. **Low Latency**: Route users to nearest region (20ms vs 200ms)
2. **High Availability**: Region failure → automatic failover (RTO < 5 minutes)
3. **Data Residency**: EU data stays in EU (GDPR compliance)
4. **Load Distribution**: 100M users across 4 regions

**Why NOT Single Region**:

- Poor latency for international users (200ms Asia → US)
- Single point of failure (region outage = complete downtime)
- GDPR issues (EU data must stay in EU)

**Why NOT Active-Passive**:

- Wasted capacity (passive region idle)
- Still poor latency for distant users

### Implementation Details

**Per Region:**

```
US-East-1 (Primary):
  - API Gateway: 30 instances
  - Payment Service: 60 instances
  - PostgreSQL: 16 shards × 3 replicas
  - Redis: 6 nodes
  - Kafka: 15 brokers

EU-West-1, AP-Southeast-1, US-West-2:
  - Same configuration
  - Async database replication (1-2s lag)
```

**Routing:**

```
Route53 Latency-Based Routing:
  - DNS query: api.gateway.com
  - Response: Nearest region IP
  - US user → US-East
  - EU user → EU-West
  - Asia user → AP-Southeast
```

### Trade-offs Accepted

| What We Gain                       | What We Sacrifice                     |
|------------------------------------|---------------------------------------|
| ✅ 10x lower latency (200ms → 20ms) | ❌ 2x cost ($100k → $200k)             |
| ✅ 99.99% availability              | ❌ Complex deployment (4 regions)      |
| ✅ GDPR compliance                  | ❌ Cross-region replication lag (1-2s) |
| ✅ Load distribution                | ❌ Monitoring complexity               |

### When to Reconsider

**Use Single Region if:**

- Users in single geography (US-only)
- Budget constrained
- Simplicity preferred

### Real-World Examples

- **Stripe**: Multi-region active-active
- **PayPal**: Multi-region
- **Square**: US-focused (single region)

---

**Summary Table: All Design Decisions**

| Decision                 | Chosen Approach            | Key Rationale        | Alternative                      |
|--------------------------|----------------------------|----------------------|----------------------------------|
| **Ledger Database**      | PostgreSQL (sharded)       | ACID guarantees      | Cassandra (eventual consistency) |
| **Payment Flow**         | Two-phase (auth + capture) | Merchant flexibility | Single-phase (charge immediate)  |
| **Duplicate Prevention** | Idempotency keys           | 100% guarantee       | Server-side dedup                |
| **Card Storage**         | Tokenization               | Reduce PCI scope     | Direct storage                   |
| **Cache**                | Redis                      | Rich data structures | Memcached                        |
| **Webhooks**             | Asynchronous               | Non-blocking         | Synchronous                      |
| **Scaling**              | Horizontal sharding        | Linear scalability   | Vertical scaling                 |
| **Bank API Failures**    | Circuit breaker            | Fail fast            | Simple retry                     |
| **Fraud Detection**      | Real-time (sync)           | Block before charge  | Batch processing                 |
| **Deployment**           | Multi-region active-active | Low latency globally | Single region                    |
