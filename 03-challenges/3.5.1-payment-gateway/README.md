# 3.5.1 Design a Payment Gateway (Stripe / PayPal)

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

Design a secure **payment gateway** like Stripe or PayPal that processes credit card transactions for e-commerce
platforms. The system must handle authorization, capture, refunds, and chargebacks while maintaining **ACID guarantees
**, **PCI-DSS compliance**, and **idempotency** to prevent duplicate charges.

**Core Challenges:**

- Financial integrity (exactly-once processing, no double charges)
- Security (PCI-DSS Level 1 compliance, tokenization)
- Latency (< 500ms authorization time)
- Availability (99.99% uptime for critical payment flow)
- Fraud prevention (real-time ML-based detection)
- Auditability (immutable ledger for reconciliation)

---

## Requirements and Scale Estimation

### Functional Requirements

1. **Transaction Processing**: Accept, authorize, and capture credit card payments
2. **Idempotency**: Guarantee exactly-once processing using idempotency keys
3. **Refunds/Chargebacks**: Support reversing or disputing transactions
4. **Multi-Currency**: Handle payments in 150+ currencies
5. **Webhooks**: Notify merchants of payment events asynchronously
6. **Compliance**: Adhere to PCI-DSS, GDPR, SOC 2 regulations

### Non-Functional Requirements

1. **Strong Consistency**: Financial ledgers must be ACID compliant (atomicity, isolation, durability)
2. **High Availability**: 99.99% uptime (52 minutes downtime per year)
3. **Low Latency**: < 500ms authorization, < 200ms token generation
4. **Security**: End-to-end encryption, tokenization, zero-trust architecture
5. **Auditability**: Immutable transaction logs, full audit trails

### Scale Estimation

| Metric                    | Assumption             | Result                   |
|---------------------------|------------------------|--------------------------|
| **Peak Throughput**       | Black Friday traffic   | 20,000 QPS               |
| **Daily Transactions**    | 10% peak sustained     | ~170M transactions/day   |
| **Transaction Storage**   | 10 years retention     | 620 billion transactions |
| **Data Volume**           | 1 KB per transaction   | 620 TB storage           |
| **Authorization Latency** | Network + processing   | < 500ms target           |
| **Downtime Cost**         | $10M daily at 2.9% fee | **$350/minute**          |

---

## High-Level Architecture

The architecture centers around **dual-phase commit** (authorization → capture), **idempotency layer**, and **immutable
ledger**.

### Core Components

```
Merchant Website → API Gateway (TLS, Rate Limiting)
                         ↓
        ┌────────────────┼────────────────┐
        ↓                ↓                ↓
  Tokenization      Payment          Fraud
   Service          Service        Detection
  (PCI Scope)                      (ML-based)
        ↓                ↓                ↓
        └────────────────┼────────────────┘
                         ↓
                  Idempotency Store
                     (Redis)
                         ↓
                  Ledger Service
                   (PostgreSQL)
                  Sharded by Merchant
                         ↓
                  Bank Processor API
                (Visa, Mastercard)
```

**Key Design Decisions:**

1. **Two-Phase Commit**: Authorization (reserve funds) → Capture (transfer money)
2. **Idempotency**: Redis cache prevents duplicate charges on retry
3. **Tokenization**: Store tokens instead of raw cards (reduce PCI scope)
4. **ACID Ledger**: PostgreSQL for financial integrity (not NoSQL)
5. **Sharding**: 64 shards by merchant_id for horizontal scaling

---

## Detailed Component Design

### Payment Flow: Authorization vs Capture

**Why Two Steps?**

1. **Authorization**: Reserve funds, verify card validity (soft hold)
2. **Capture**: Transfer money after fulfillment checks pass

**Benefits:**

- Cancel orders before capture (no refund needed)
- Check inventory before charging
- Reduces chargeback risk

**Flow:**

```
Authorization:
  POST /payments {amount, card_token, idempotency_key}
  → Check idempotency cache
  → Call bank for authorization
  → Write AUTHORIZED to ledger
  → Return {payment_id, status: authorized}

Capture (hours/days later):
  POST /payments/{id}/capture
  → Verify payment authorized
  → Call bank to transfer funds
  → Write CAPTURED to ledger
  → Return {status: captured}
  → Trigger webhook: payment.captured
```

*See pseudocode.md::authorize_payment() and capture_payment() for details*

### Idempotency System

**The Problem**: Network failures cause retries. Without idempotency, retry charges customer twice.

**Solution**: Client provides `idempotency_key` (UUID). Server caches results in Redis.

**Implementation:**

```
1. Client generates UUID: "idem_abc123"
2. Server: GET idempotency:idem_abc123 from Redis
3. If exists → return cached result (same response)
4. If not exists:
   - Process payment
   - SET idempotency:idem_abc123 {response} EX 86400 (24h TTL)
   - Return response
```

**Properties:**

- **TTL**: 24 hours (balances memory vs safety)
- **Scope**: Per merchant
- **Storage**: Redis cluster (replicated)
- **Stale reads OK**: Idempotency is best-effort

*See pseudocode.md::check_idempotency() for implementation*

### Tokenization (PCI-DSS Compliance)

**Problem**: Storing credit card numbers requires expensive PCI-DSS Level 1 certification.

**Solution**: Replace card with non-sensitive token.

**Flow:**

```
1. POST /tokens {card_number: "4111111111111111"}
2. Tokenization Service:
   - Validate card (Luhn algorithm)
   - Generate token: "tok_abc123"
   - Encrypt card: AES-256(card_number)
   - Store: tokens_db[tok_abc123] = encrypted_card
   - Return token
3. POST /payments {amount, token: "tok_abc123"}
4. Payment Service: Fetch card, send to bank
```

**Security:**

- **Encryption**: AES-256 at rest, TLS 1.3 in transit
- **Key Management**: AWS KMS / HashiCorp Vault (HSM-backed)
- **PCI Scope**: Only Tokenization Service in scope

**Schema:**

```sql
CREATE TABLE tokens (
    token_id VARCHAR(32) PRIMARY KEY,
    card_fingerprint CHAR(64),  -- SHA-256 for dedup
    encrypted_card BYTEA,       -- AES-256
    card_brand VARCHAR(20),
    last4 CHAR(4),
    exp_month INT,
    exp_year INT,
    merchant_id VARCHAR(32),
    created_at TIMESTAMP
);
```

*See pseudocode.md::tokenize_card() for implementation*

### Ledger Database Design

**Requirements:**

- **ACID**: Atomicity, Consistency, Isolation, Durability
- **Immutable**: Append-only, never update
- **Auditable**: Every state change logged
- **Reconcilable**: Match bank statements exactly

**Choice: Sharded PostgreSQL**

**Why PostgreSQL over NoSQL?**

| Feature           | PostgreSQL  | Cassandra         | DynamoDB          |
|-------------------|-------------|-------------------|-------------------|
| **ACID**          | ✅ Full      | ❌ Eventual        | ❌ Eventual        |
| **Transactions**  | ✅ Multi-row | ❌ Single-row      | ❌ Limited         |
| **Financial Use** | ✅ Standard  | ❌ Not recommended | ❌ Not recommended |

**Schema:**

```sql
CREATE TABLE payments (
    payment_id VARCHAR(32) PRIMARY KEY,
    merchant_id VARCHAR(32) NOT NULL,
    idempotency_key VARCHAR(64) UNIQUE,
    amount_cents BIGINT NOT NULL,  -- Store as cents
    currency CHAR(3),
    status VARCHAR(20),  -- authorized, captured, refunded
    card_token VARCHAR(32),
    authorization_code VARCHAR(64),
    created_at TIMESTAMP,
    INDEX (merchant_id, created_at),
    INDEX (idempotency_key)
);

CREATE TABLE payment_events (
    event_id BIGSERIAL PRIMARY KEY,
    payment_id VARCHAR(32),
    event_type VARCHAR(32),
    previous_status VARCHAR(20),
    new_status VARCHAR(20),
    metadata JSONB,
    created_at TIMESTAMP
);
```

**Sharding:**

- **Shard Key**: merchant_id (all payments for one merchant on same shard)
- **Shard Count**: 64 shards (300 writes/sec per shard)
- **Replication**: 3 replicas per shard (primary + 2 read replicas)

*See pseudocode.md::write_to_ledger() for implementation*

---

## Security and Compliance

### PCI-DSS Level 1 Compliance

**Requirements** (6M+ transactions/year):

1. **Secure Network**: Firewalls, no default passwords
2. **Protect Cardholder Data**: AES-256 encryption, TLS 1.3
3. **Vulnerability Management**: Regular updates, anti-virus
4. **Access Control**: Need-to-know basis, unique IDs
5. **Monitor Networks**: Track all card data access
6. **Security Policy**: Documented for all personnel

**Implementation:**

- **Tokenization Service**: Only component in PCI scope
- **Encryption**: AES-256 for card data
- **Key Rotation**: Monthly via AWS KMS
- **Access Logs**: Every card access logged immutably
- **Network Segmentation**: Tokenization in isolated VPC

### Fraud Detection

**Challenge**: Detect fraud in < 50ms (part of authorization flow).

**Solution**: Real-time ML-based risk scoring.

**Features:**

1. **Transaction**: Amount, currency, merchant category
2. **Card**: Brand, issuing bank, decline rate
3. **Customer**: Email, IP, billing vs shipping mismatch
4. **Behavioral**: Spending pattern deviation, velocity

**Pipeline:**

```
Real-Time (Synchronous):
- Load features from Redis
- Score with decision tree
- If risk > 0.8 → BLOCK
- If 0.5-0.8 → FLAG (3D Secure)
- If < 0.5 → ALLOW

Offline (Daily):
- Kafka → Spark → Feature engineering
- Train XGBoost on labeled data
- Deploy new model to Redis
```

**Actions:**

- **Low Risk** (< 0.5): Auto-approve
- **Medium Risk** (0.5-0.8): 3D Secure challenge
- **High Risk** (> 0.8): Block transaction

*See pseudocode.md::calculate_fraud_score() for implementation*

---

## Refunds and Chargebacks

### Refunds

**Types:**

1. **Full Refund**: Return entire amount
2. **Partial Refund**: Return portion

**Flow:**

```
1. POST /payments/{id}/refunds {amount}
2. Verify payment captured
3. Create refund record in ledger
4. Send refund to bank
5. Update ledger with status
6. Webhook: payment.refunded
```

**Idempotency**: Refunds use idempotency keys (prevent double refunds).

**Schema:**

```sql
CREATE TABLE refunds (
    refund_id VARCHAR(32) PRIMARY KEY,
    payment_id VARCHAR(32),
    amount_cents BIGINT,
    reason VARCHAR(255),
    status VARCHAR(20),
    created_at TIMESTAMP,
    FOREIGN KEY (payment_id) REFERENCES payments(payment_id)
);
```

### Chargebacks

**Definition**: Customer disputes charge with bank (not merchant).

**Reasons:**

- Fraudulent transaction
- Product not received
- Duplicate charge

**Flow:**

```
1. Bank notifies gateway of chargeback
2. Create chargeback record
3. Webhook: payment.dispute.created
4. Merchant has 7-21 days to provide evidence
5. If win → funds returned
6. If lose → funds deducted, fee applied ($15-$25)
```

**Impact:**

- High chargeback rate (> 1%) → Gateway may terminate merchant
- Chargeback fee: $15-$25 per chargeback
- Reserve funds: Gateway may hold 10% of revenue

*See pseudocode.md::process_chargeback() for implementation*

---

## Multi-Currency Support

**Components:**

1. **Exchange Rate Service**:
    - Fetch rates from providers
    - Cache in Redis (TTL: 1 hour)
    - Update every 30 minutes

2. **Presentment vs Settlement**:
    - **Presentment**: Currency shown to customer (EUR)
    - **Settlement**: Currency paid to merchant (USD)

**Example:**

```
Customer in Europe:
- Sees: €100.00
- Exchange rate: 1 EUR = 1.10 USD
- Merchant receives: $110.00
- Fee (2.9% + $0.30): $3.49
- Net: $106.51
```

**Schema:**

```sql
CREATE TABLE exchange_rates (
    rate_id BIGSERIAL PRIMARY KEY,
    base_currency CHAR(3),
    quote_currency CHAR(3),
    rate DECIMAL(18, 8),
    valid_from TIMESTAMP,
    valid_until TIMESTAMP
);
```

*See pseudocode.md::convert_currency() for implementation*

---

## Webhook System

**Purpose**: Notify merchants of async events.

**Events:**

- `payment.authorized`
- `payment.captured`
- `payment.failed`
- `payment.refunded`
- `payment.dispute.created`

**Delivery:**

- **At-least-once**: Retry up to 10 times
- **Signature**: HMAC-SHA256 in header
- **Idempotency**: Merchant handles duplicates

**Retry Schedule:**

1. Immediate
2. 1 minute
3. 5 minutes
4. 30 minutes
   5-10. 1h, 6h, 12h, 24h

*See pseudocode.md::deliver_webhook() for implementation*

---

## Availability and Fault Tolerance

### Database Failover

**Setup**: Multi-master PostgreSQL (3 nodes per shard).

**Failure:**

```
Normal: Primary (writes) → Replica 1 → Replica 2

Primary Fails:
1. Health check detects (< 10s)
2. Replica 1 promoted to primary
3. Traffic rerouted
4. Failed node replaced

RTO: < 30 seconds
RPO: 0 (synchronous replication)
```

### Circuit Breaker for Bank API

**Problem**: Bank API slow → system backs up.

**Solution**: Circuit breaker pattern.

**States:**

1. **Closed**: All requests go through
2. **Open**: Fail fast, don't call bank
3. **Half-Open**: Test request after cooldown

**Config:**

- **Failure threshold**: 50% error rate over 10s
- **Cooldown**: 60 seconds
- **Success threshold**: 5 consecutive successes

*See pseudocode.md::circuit_breaker() for implementation*

---

## Common Anti-Patterns

### ❌ 1. Using Floats for Money

**Problem**: Floating-point arithmetic is imprecise.

```
0.1 + 0.2 = 0.30000000000000004
```

**Solution**: Store as **cents** (integers).

```
✅ CORRECT: amount_cents = 10050  // $100.50
❌ WRONG: amount_dollars = 100.50  // Precision loss
```

### ❌ 2. Skipping Idempotency

**Problem**: Network failures → retries → double charges.

**Solution**: Always require `idempotency_key`.

```
✅ CORRECT:
POST /payments {"amount": 10000, "idempotency_key": "idem_abc123"}

❌ WRONG:
POST /payments {"amount": 10000}  // No idempotency
```

### ❌ 3. Storing Raw Credit Cards

**Problem**: PCI-DSS Level 1 certification costs millions.

**Solution**: Tokenize immediately.

```
✅ CORRECT: tokens_db[tok_abc] = AES-256("4111...")
❌ WRONG: payments_db[id] = {"card": "4111..."}
```

### ❌ 4. Missing Audit Trail

**Problem**: Can't reconcile with bank statements.

**Solution**: Append-only event log.

```
✅ CORRECT:
payment_events:
- event_id=1: pending → authorized
- event_id=2: authorized → captured

❌ WRONG:
UPDATE payments SET status='captured'  // Lost history
```

### ❌ 5. Synchronous Webhook Delivery

**Problem**: Merchant endpoint down → block payment.

**Solution**: Async webhook delivery.

```
✅ CORRECT:
1. Capture payment
2. Publish to Kafka
3. Return success
4. Worker delivers webhook async

❌ WRONG:
1. Capture payment
2. Call webhook (blocking)
3. If fails → fail payment
```

---

## Monitoring and Observability

### Key Metrics

**Business Metrics:**

| Metric                         | Target       | Alert |
|--------------------------------|--------------|-------|
| **Authorization Success Rate** | > 98%        | < 95% |
| **Fraud Detection Rate**       | 0.1% blocked | > 1%  |
| **Chargeback Rate**            | < 0.5%       | > 1%  |

**System Metrics:**

| Metric                  | Target  | Alert    |
|-------------------------|---------|----------|
| **API Latency (p95)**   | < 500ms | > 1000ms |
| **DB Query Time (p95)** | < 50ms  | > 200ms  |
| **Cache Hit Rate**      | > 80%   | < 50%    |

**Dashboards:**

1. **Transaction Health**: Authorization rate, latency, errors
2. **Financial Health**: Revenue, refunds, chargebacks
3. **Infrastructure**: DB CPU, Redis hit rate, Kafka lag

---

## Cost Analysis

**Monthly Cost** (170M transactions/month):

| Component                  | Cost            |
|----------------------------|-----------------|
| **API Gateway**            | $15k            |
| **Payment Service**        | $40k            |
| **PostgreSQL** (64 shards) | $120k           |
| **Redis Cluster**          | $10k            |
| **Tokenization**           | $30k            |
| **Fraud Detection**        | $35k            |
| **Total**                  | **$345k/month** |

**Revenue** (2.9% + $0.30 fee):

```
170M transactions × $50 avg = $8.5B volume
- Percentage: $8.5B × 2.9% = $246.5M
- Fixed: 170M × $0.30 = $51M
Total: $297.5M/month

Net margin: 99.9% (infrastructure negligible)
```

---

## Trade-offs Summary

| Decision             | Gain                  | Sacrifice          |
|----------------------|-----------------------|--------------------|
| **Two-Phase Commit** | ✅ Flexibility         | ❌ Complexity       |
| **Tokenization**     | ✅ PCI scope reduction | ❌ Extra service    |
| **PostgreSQL**       | ✅ ACID guarantees     | ❌ Harder to scale  |
| **Idempotency**      | ✅ Exactly-once        | ❌ Client UUID      |
| **Async Webhooks**   | ✅ Non-blocking        | ❌ Async complexity |

---

## Real-World Examples

### Stripe

- **Architecture**: Ruby on Rails → microservices
- **Database**: Sharded PostgreSQL
- **Cache**: Redis for idempotency
- **Scale**: $640B volume (2021)
- **Innovation**: Developer-first API

### PayPal

- **Architecture**: Java microservices
- **Database**: Oracle → distributed systems
- **Scale**: $1.36T volume (2021)
- **Innovation**: Buyer/seller protection

### Square

- **Architecture**: Go, Ruby microservices
- **Database**: MySQL
- **Scale**: $112B volume (2020)
- **Innovation**: Unified online + offline commerce

---

## References

**Related Chapters:**

- [PostgreSQL Deep Dive](../../02-components/2.1-databases/2.1.1-postgresql-deep-dive.md) - ACID transactions
- [Redis Deep Dive](../../02-components/2.2-caching/2.2.1-redis-deep-dive.md) - Idempotency cache
- [Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md) - Event streaming

**External Resources:**

- Stripe API Documentation: https://stripe.com/docs/api
- PCI-DSS Standards: https://www.pcisecuritystandards.org/
- Idempotency in APIs: https://stripe.com/blog/idempotency

**Books:**

- *Designing Data-Intensive Applications* by Martin Kleppmann
- *Site Reliability Engineering* by Google

---

## Interview Discussion Points

### Question 1: How do you prevent double charging?

**Answer**: Idempotency keys.

**Deep Dive:**

1. Client generates UUID for each request
2. Server checks Redis cache for key
3. If exists → return cached result
4. If not → process payment, cache result with 24h TTL
5. Scope: per merchant

**Follow-up**: What if Redis fails?

- Graceful degradation: Query database
- Slower but correct
- Alert ops team

### Question 2: Why PostgreSQL instead of NoSQL?

**Answer**: ACID guarantees required for financial transactions.

**Deep Dive:**

- **Atomicity**: All-or-nothing
- **Consistency**: Referential integrity
- **Isolation**: No concurrent interference
- **Durability**: Survives crashes

**Scaling**: Horizontal sharding (64 shards by merchant_id)

### Question 3: How do you handle bank API failures?

**Answer**: Circuit breaker + multiple processors.

**Circuit Breaker States:**

1. **Closed**: Normal operation
2. **Open**: Fail fast
3. **Half-Open**: Testing recovery

**Config**: 50% error threshold, 60s cooldown

### Question 4: How do you scale to 20k QPS?

**Answer**: Multi-level approach.

**Horizontal Scaling:**

- API Gateway: 100 instances
- Payment Service: 200 instances
- Database: 64 shards
- Redis: 10-node cluster

**Vertical Optimizations:**

- Connection pooling
- Redis caching
- Async webhooks
- Read replicas

### Question 5: How do you ensure PCI-DSS compliance?

**Answer**: Tokenization + network segmentation.

**Strategy:**

1. **Minimize Scope**: Only Tokenization Service stores cards
2. **Secure Service**: Isolated VPC, HSM for keys, AES-256 encryption
3. **Regular Audits**: Annual PCI assessment, quarterly scans
4. **Employee Training**: Security awareness, background checks

---

## Advanced Features

### 3D Secure (3DS) Authentication

**Purpose**: Additional authentication for high-risk transactions.

**Flow:**

```
1. Risk score: 0.65 (medium)
2. Redirect to 3DS challenge
3. Customer enters OTP
4. Bank verifies
5. If success → liability shifts to bank
```

**Benefits**:

- Reduces fraud
- Required for EU SCA
- Shifts chargeback liability

### Recurring Payments / Subscriptions

**Use Case**: Netflix, Spotify billing.

**Lifecycle:**

```
1. Create subscription
2. Store with billing cycle
3. Daily cron charges on due date
4. Retry failed charges (Day 0, 3, 7)
5. After 30 days → Cancel
```

**Schema:**

```sql
CREATE TABLE subscriptions (
    subscription_id VARCHAR(32) PRIMARY KEY,
    customer_id VARCHAR(32),
    plan_id VARCHAR(32),
    payment_method_token VARCHAR(32),
    status VARCHAR(20),
    current_period_end DATE
);
```

### Payment Links

**Use Case**: Send payment via email/SMS.

**Flow:**

```
1. Merchant: POST /payment_links {amount, description}
2. Response: {url: "https://pay.gateway.com/plink_abc"}
3. Send to customer
4. Customer pays on hosted page
5. Merchant receives webhook
```

**Benefits**:

- No website needed
- Gateway handles PCI
- Mobile-optimized

### Split Payments (Marketplace)

**Use Case**: Uber, Airbnb platforms.

**Scenario**:

- Customer pays $100
- Platform takes $20 commission
- Driver receives $80

**Flow:**

```
POST /payments {
  amount: 10000,
  application_fee_amount: 2000,
  transfer_data: {destination: "acct_driver"}
}

Result:
- Customer: -$100
- Platform: +$20
- Driver: +$80 (auto-transfer)
```

---

## Performance Optimization

### Database Connection Pooling

**Problem**: New connection per request = 100ms overhead.

**Solution**: Connection pool.

**Config:**

- Min: 50 connections per instance
- Max: 200 connections per instance
- Total: 60 instances × 200 = 12k connections
- Per shard: 12k / 64 = 187 connections

### Read Replicas

**Problem**: Dashboard queries slow down payments.

**Solution**: Route analytics to read replicas.

```
Write Path: API → Primary DB
Read Path: Dashboard → Read Replica (1s lag OK)
```

### Multi-Level Caching

**Levels:**

1. **Application Cache**: In-memory (100 MB per instance)
    - Exchange rates (1h TTL)
    - Merchant configs (10m TTL)

2. **Redis Cache**: Distributed (50 GB cluster)
    - Idempotency keys (24h TTL)
    - Token metadata (1h TTL)
    - Fraud features (5m TTL)

3. **Database**: Source of truth

### API Rate Limiting

**Strategy**: Token bucket per merchant.

**Limits:**

- Tier 1 (Small): 100 req/sec
- Tier 2 (Medium): 1,000 req/sec
- Tier 3 (Enterprise): 10,000 req/sec

**Implementation**: Redis INCR with TTL.

---

## Deployment Strategy

### Multi-Region Architecture

**Regions:**

- US-East-1 (Primary)
- US-West-2 (Secondary)
- EU-West-1 (GDPR compliance)
- AP-Southeast-1 (Asian users)

**Per Region:**

- API Gateway: 30 instances
- Payment Service: 60 instances
- PostgreSQL: 16 shards × 3 replicas
- Redis: 6 nodes
- Kafka: 15 brokers

**Disaster Recovery:**

- RTO: 5 minutes
- RPO: 0 (sync within region)
- Automated failover scripts

### Database Sharding

**Strategy**: Hash-based sharding by merchant_id.

**Shard Calculation:**

```
shard_id = hash(merchant_id) % 64
```

**Benefits:**

- All merchant payments on same shard
- Enables merchant-scoped analytics
- Even distribution

**Rebalancing**: Consistent hashing when adding shards (only 50% move).

---

## Testing

### Unit Tests

**Critical Functions:**

1. **Idempotency**: Same key → same result
2. **Amount Calculation**: Integer arithmetic only
3. **Fraud Scoring**: Known patterns flagged
4. **Tokenization**: Encryption/decryption

### Integration Tests

**Scenarios:**

1. **Successful Flow**: Authorize → Capture
2. **Idempotency**: Retry returns same payment_id
3. **Fraud Block**: High-risk transaction blocked

### Load Testing

**Black Friday Simulation:**

- Tool: JMeter / Locust
- Profile: 0 → 20k QPS over 10 min
- Sustain: 20k QPS for 40 min
- Mix: 70% auth, 20% token, 10% other

**Success Criteria:**

- p95 latency < 500ms
- Error rate < 0.1%
- Resource utilization < 80%

### Chaos Engineering

**Experiments:**

1. **Database Failover**: Kill primary, verify auto-promotion
2. **Bank API Outage**: Simulate 500 errors, verify circuit breaker
3. **Network Partition**: Isolate AZ, verify traffic reroute
4. **Cache Failure**: Stop Redis, verify graceful degradation

---

## Compliance and Auditing

### Audit Logging

**What to Log:**

1. **Card Data Access**: Who, what, when, where, result
2. **Payment State Changes**: Previous → new state, trigger, reason
3. **Configuration Changes**: Settings, API keys, webhooks
4. **Security Events**: Failed auth, rate limits, fraud

**Schema:**

```sql
CREATE TABLE audit_logs (
    log_id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(64),
    actor_id VARCHAR(32),
    resource_type VARCHAR(32),
    action VARCHAR(32),
    previous_value JSONB,
    new_value JSONB,
    created_at TIMESTAMP
);
```

**Retention**: 7 years (PCI requirement: 1 year minimum)

### SOC 2 Compliance

**Trust Criteria:**

1. **Security**: Authorized access controls
2. **Availability**: 99.99% uptime SLA
3. **Processing Integrity**: Complete, valid, accurate
4. **Confidentiality**: Protected information
5. **Privacy**: Personal data handling

**Controls:**

- MFA for all employees
- Principle of least privilege
- 24/7 SOC monitoring
- Weekly DR drills

---

## Key Architectural Decisions Summary

### 1. Two-Phase Commit (Authorization → Capture)

**Why**: Flexibility for merchants to cancel before capture.

**Trade-off**: More complexity but reduces refunds.

### 2. Idempotency Keys

**Why**: Prevent double charges on network retry.

**Trade-off**: Client must generate UUIDs.

### 3. Tokenization

**Why**: Reduce PCI scope (only one service stores cards).

**Trade-off**: Additional service to maintain.

### 4. PostgreSQL with Sharding

**Why**: ACID guarantees for financial data.

**Trade-off**: Harder to scale than NoSQL (need sharding).

### 5. Async Webhooks

**Why**: Don't block payment flow on merchant endpoint.

**Trade-off**: Merchant must handle eventual delivery.

### 6. Real-Time Fraud Detection

**Why**: Block fraud before money moves.

**Trade-off**: Adds 50ms latency to authorization.

### 7. Circuit Breaker for Bank APIs

**Why**: Prevent cascading failures.

**Trade-off**: Some requests fail fast (better than timeout).

### 8. Multi-Region Deployment

**Why**: Low latency globally + high availability.

**Trade-off**: 2x infrastructure cost.

---

## Comparison with Real Systems

### Stripe

**Similarities:**

- PostgreSQL for ledger
- Redis for idempotency
- Tokenization for PCI scope reduction
- Webhook delivery system

**Differences:**

- Started as Rails monolith (we start with microservices)
- More emphasis on developer experience (SDKs, docs)

### PayPal

**Similarities:**

- Two-phase commit (authorization → capture)
- Fraud detection with ML
- Multi-region deployment

**Differences:**

- Oracle database historically (we use PostgreSQL)
- Buyer/seller protection (dispute resolution)
- More focus on consumer payments (we focus on merchant)

### Square

**Similarities:**

- MySQL/PostgreSQL for transactions
- Real-time authorization
- Webhook notifications

**Differences:**

- Unified online + offline (point-of-sale hardware)
- Focus on small businesses
- Simpler API (we have more advanced features)

---

## Conclusion

This payment gateway design demonstrates:

- **Financial Integrity**: ACID transactions, idempotency, immutable ledger
- **Security**: PCI-DSS compliance, tokenization, encryption
- **Scalability**: 20k QPS via sharding, caching, horizontal scaling
- **Availability**: 99.99% uptime via multi-region, circuit breakers
- **Fraud Prevention**: Real-time ML scoring, 3D Secure
- **Developer Experience**: Clean API, webhooks, comprehensive docs

**Key Takeaways:**

1. Use ACID database (PostgreSQL) for financial transactions
2. Implement idempotency to prevent double charges
3. Tokenize cards to reduce PCI scope
4. Shard database by merchant_id for horizontal scaling
5. Use circuit breakers to handle bank API failures
6. Deploy multi-region for low latency and high availability
7. Real-time fraud detection with ML models
8. Comprehensive testing (unit, integration, load, chaos)
9. Audit logging for compliance
10. Monitor business and system metrics

The system handles **$297M/month revenue** on **$345k/month infrastructure** (99.9% margin), processing **170M
transactions/day** at **20k QPS peak** with **< 500ms latency** and **99.99% uptime**.


---

## Additional Technical Deep Dives

### Idempotency Implementation Details

**Redis Storage:**

```
Key: idempotency:{merchant_id}:{idempotency_key}
Value: {
  payment_id,
  status,
  amount_cents,
  response_payload,
  created_at
}
TTL: 86400 seconds (24 hours)
```

**Edge Cases:**

1. **Concurrent Requests**: Use Redis SETNX for atomic check-and-set
2. **Partial Success**: Store intermediate state for resume
3. **Key Expiry**: After 24h, allow reprocessing (merchant should use new key)

### Fraud Detection Model Features

**Transaction Features (10):**

- Amount, currency, merchant_category
- Time of day, day of week
- Device fingerprint
- IP address, geolocation
- Billing vs shipping address match
- Velocity (transactions per hour)

**Card Features (8):**

- Card brand, issuing bank
- Card country vs IP country
- Previous decline rate
- First transaction on card
- Card age (days since first use)
- BIN (Bank Identification Number) risk score
- Credit vs debit

**Customer Features (12):**

- Email domain reputation
- Phone number verification
- Account age (days since registration)
- Previous successful payments
- Chargeback history
- Device history (known vs new)
- Browser/user agent
- Operating system
- Screen resolution (fraud bots often have unusual resolutions)
- Timezone match with IP
- Language/locale
- Referrer URL

**Model**: XGBoost with 30 features, trained daily on 1M labeled transactions.

**Performance**:

- True Positive Rate: 85% (correctly block fraud)
- False Positive Rate: 2% (incorrectly block legitimate)
- Inference Time: 20ms (on GPU)

### Database Schema Complete

**Full Schema (7 tables):**

```sql
-- Core payment tracking
CREATE TABLE payments (
    payment_id VARCHAR(32) PRIMARY KEY,
    merchant_id VARCHAR(32) NOT NULL,
    customer_id VARCHAR(32),
    idempotency_key VARCHAR(64) UNIQUE,
    amount_cents BIGINT NOT NULL,
    currency CHAR(3) DEFAULT 'USD',
    status VARCHAR(20),
    card_token VARCHAR(32),
    authorization_code VARCHAR(64),
    captured_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_merchant_created (merchant_id, created_at),
    INDEX idx_status (status),
    INDEX idx_customer (customer_id, created_at)
);

-- Immutable event log
CREATE TABLE payment_events (
    event_id BIGSERIAL PRIMARY KEY,
    payment_id VARCHAR(32) NOT NULL,
    event_type VARCHAR(32),
    previous_status VARCHAR(20),
    new_status VARCHAR(20),
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_payment (payment_id, created_at)
);

-- Tokenized cards
CREATE TABLE tokens (
    token_id VARCHAR(32) PRIMARY KEY,
    card_fingerprint CHAR(64),
    encrypted_card BYTEA,
    card_brand VARCHAR(20),
    last4 CHAR(4),
    exp_month INT,
    exp_year INT,
    merchant_id VARCHAR(32),
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_merchant (merchant_id, created_at),
    INDEX idx_fingerprint (card_fingerprint)
);

-- Refunds
CREATE TABLE refunds (
    refund_id VARCHAR(32) PRIMARY KEY,
    payment_id VARCHAR(32) NOT NULL,
    amount_cents BIGINT,
    reason VARCHAR(255),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    FOREIGN KEY (payment_id) REFERENCES payments(payment_id),
    INDEX idx_payment (payment_id)
);

-- Exchange rates
CREATE TABLE exchange_rates (
    rate_id BIGSERIAL PRIMARY KEY,
    base_currency CHAR(3),
    quote_currency CHAR(3),
    rate DECIMAL(18, 8),
    valid_from TIMESTAMP,
    valid_until TIMESTAMP,
    INDEX idx_currencies (base_currency, quote_currency, valid_from)
);

-- Webhooks
CREATE TABLE webhooks (
    webhook_id BIGSERIAL PRIMARY KEY,
    merchant_id VARCHAR(32),
    event_type VARCHAR(64),
    payload JSONB,
    url TEXT,
    status VARCHAR(20),
    attempts INT DEFAULT 0,
    last_attempt_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_merchant_status (merchant_id, status),
    INDEX idx_created (created_at)
);

-- Audit logs
CREATE TABLE audit_logs (
    log_id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(64),
    actor_id VARCHAR(32),
    resource_type VARCHAR(32),
    resource_id VARCHAR(32),
    action VARCHAR(32),
    previous_value JSONB,
    new_value JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_resource (resource_type, resource_id),
    INDEX idx_actor (actor_id, created_at)
);
```

### API Endpoints Summary

**Payment Operations:**

- POST /tokens - Tokenize card
- POST /payments - Authorize payment
- POST /payments/{id}/capture - Capture authorized payment
- POST /payments/{id}/cancel - Cancel authorization
- GET /payments/{id} - Get payment details
- POST /payments/{id}/refunds - Create refund

**Subscription Operations:**

- POST /subscriptions - Create subscription
- GET /subscriptions/{id} - Get subscription
- PUT /subscriptions/{id} - Update subscription
- DELETE /subscriptions/{id} - Cancel subscription

**Marketplace Operations:**

- POST /accounts - Create connected account
- POST /transfers - Create transfer
- POST /payouts - Create payout

**Utility Operations:**

- POST /payment_links - Create payment link
- GET /exchange_rates - Get current rates
- POST /webhooks - Register webhook endpoint

### Error Codes and Handling

**HTTP Status Codes:**

- 200 OK: Success
- 400 Bad Request: Invalid parameters
- 401 Unauthorized: Invalid API key
- 402 Payment Required: Insufficient funds
- 404 Not Found: Resource not found
- 409 Conflict: Duplicate idempotency key with different params
- 429 Too Many Requests: Rate limit exceeded
- 500 Internal Server Error: Our fault
- 502 Bad Gateway: Bank API error
- 503 Service Unavailable: Maintenance mode

**Error Response Format:**

```json
{
  "error": {
    "type": "card_error",
    "code": "insufficient_funds",
    "message": "Your card has insufficient funds.",
    "param": "amount",
    "payment_id": "pay_abc123"
  }
}
```

**Error Categories:**

1. **Card Errors**: insufficient_funds, expired_card, incorrect_cvc
2. **API Errors**: invalid_request, authentication_required
3. **Rate Limit Errors**: rate_limit_exceeded
4. **Processing Errors**: processing_error, bank_timeout

