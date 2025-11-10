# Payment Gateway - Quick Overview

## Core Concept

A payment gateway securely processes credit card transactions between merchants and banks. It handles authorization (reserve funds), capture (transfer money), tokenization (PCI-DSS compliance), idempotency (prevent duplicate charges), fraud detection, and webhooks (async notifications).

**Key Challenge**: Process 20k payments/sec with 99.99% uptime, ACID guarantees, and PCI-DSS Level 1 compliance.

---

## Requirements

### Functional
- **Authorization + Capture**: Two-phase commit (authorize first, capture later)
- **Tokenization**: Replace card numbers with tokens (PCI scope reduction)
- **Idempotency**: Prevent duplicate charges on retries
- **Webhooks**: Async notifications to merchants
- **Refunds**: Full and partial refunds
- **Multi-Currency**: Support USD, EUR, GBP, etc.

### Non-Functional
- **Throughput**: 20k payments/sec (1.7B payments/day)
- **Latency**: <500ms for authorization
- **Availability**: 99.99% uptime (52 minutes downtime/year)
- **Consistency**: ACID guarantees (no lost payments)
- **Security**: PCI-DSS Level 1 compliance
- **Durability**: Zero data loss (audit trail)

---

## Components

### 1. **API Gateway**
- Rate limiting (1000 req/sec per merchant)
- JWT authentication
- Request routing to payment service

### 2. **Payment Service**
- Authorization: Reserve funds, verify card
- Capture: Transfer money (hours/days later)
- Idempotency check (Redis lookup)
- Tokenization service integration

### 3. **Tokenization Service** (PCI Scope)
- Encrypts card numbers (AES-256)
- Generates tokens (tok_abc123)
- Stores encrypted cards in isolated database
- Reduces PCI scope (only this service handles raw cards)

### 4. **Ledger Database** (PostgreSQL)
- Sharded by merchant_id (64 shards)
- ACID transactions (atomicity, consistency)
- Immutable audit trail (payment_events table)
- 3 replicas per shard (primary + 2 read replicas)

### 5. **Idempotency Cache** (Redis)
- Key: `idempotency:{merchant_id}:{idempotency_key}`
- TTL: 24 hours
- Returns cached response on retry

### 6. **Fraud Detection Service**
- Real-time ML model (50ms latency)
- Rule-based filters (blacklist, velocity checks)
- Blocks suspicious transactions before capture

### 7. **Webhook Service**
- Publishes events to Kafka (`payment.events` topic)
- Retry logic (exponential backoff)
- Dead letter queue for failed deliveries

### 8. **Bank Integration**
- REST API to payment processors (Stripe, PayPal, Adyen)
- Circuit breaker pattern (fail fast on bank downtime)

---

## Architecture Flow

### Authorization Flow

```
1. Client → API Gateway: POST /payments {amount, token, idempotency_key}
2. API Gateway → Payment Service: Route request
3. Payment Service → Redis: Check idempotency (GET idempotency:key)
   - If exists → return cached response (HTTP 200)
   - If not → continue
4. Payment Service → Tokenization Service: Fetch card (GET /tokens/{token})
5. Payment Service → Bank API: Authorize payment (POST /authorize)
6. Bank → Payment Service: Authorization code (ABC123)
7. Payment Service → PostgreSQL: Write AUTHORIZED status (transaction)
8. Payment Service → Redis: Cache idempotency result (SET idempotency:key {response} EX 86400)
9. Payment Service → Kafka: Publish payment.authorized event
10. Payment Service → Client: {payment_id, status: authorized}
11. Webhook Service → Merchant: POST webhook_url {event: payment.authorized}
```

### Capture Flow

```
1. Client → API Gateway: POST /payments/{id}/capture
2. Payment Service → PostgreSQL: Read payment (status must be AUTHORIZED)
3. Payment Service → Bank API: Capture payment (POST /capture {auth_code})
4. Bank → Payment Service: Settlement ID (XYZ789)
5. Payment Service → PostgreSQL: Update status to CAPTURED (transaction)
6. Payment Service → Kafka: Publish payment.captured event
7. Webhook Service → Merchant: POST webhook_url {event: payment.captured}
```

---

## Key Design Decisions

### 1. **Authorization + Capture (Two-Phase Commit)**

**Why?**
- Merchant flexibility: Cancel order before capture (no refund needed)
- Reduces chargebacks (customer changed mind)
- Handles inventory checks (authorize first, capture only if in stock)

**Trade-off**: Two API calls (more complexity) vs single charge

### 2. **PostgreSQL for Ledger** (vs Cassandra/DynamoDB)

**Why PostgreSQL?**
- **ACID Guarantees**: Multi-row transactions (authorize + write event atomically)
- **Auditability**: Write-ahead log (WAL) for compliance
- **Financial Standard**: Industry standard for money systems

**Trade-off**: Harder to scale (need sharding) vs eventual consistency

### 3. **Tokenization** (PCI-DSS Compliance)

**Why?**
- PCI-DSS Level 1 certification costs millions
- Tokenization reduces scope (only Tokenization Service handles raw cards)
- Rest of system doesn't need PCI certification

**Trade-off**: Additional service to maintain vs cheaper compliance

### 4. **Idempotency Keys** (Client-Provided UUIDs)

**Why?**
- Network failures cause retries
- Without idempotency: customer charged twice
- Client generates UUID, server caches result

**Trade-off**: Client must generate UUIDs vs server-side deduplication

### 5. **Async Webhooks** (vs Synchronous Callbacks)

**Why?**
- Non-blocking payment flow (don't wait for merchant webhook)
- Retry logic for failed deliveries
- Dead letter queue for debugging

**Trade-off**: Merchant must handle async events vs immediate confirmation

---

## Security & Compliance

### PCI-DSS Level 1 Requirements

1. **Encryption at Rest**: AES-256 with key rotation (AWS KMS)
2. **Encryption in Transit**: TLS 1.3 only
3. **Tokenization**: Replace card numbers with tokens
4. **Access Control**: Need-to-know basis, unique IDs
5. **Audit Trail**: All cardholder data access logged
6. **Vulnerability Management**: Regular security testing
7. **Network Segmentation**: Firewalls between public/private networks

### Fraud Detection

- **Real-Time ML Model**: 50ms latency, blocks suspicious transactions
- **Rule-Based Filters**: Blacklist IPs, velocity checks (>10 payments/min)
- **Behavioral Analysis**: Unusual spending patterns
- **3D Secure**: Additional authentication for high-risk transactions

---

## Bottlenecks & Solutions

### 1. **PostgreSQL Write Throughput**

**Problem**: 20k writes/sec exceeds single PostgreSQL capacity (~5k writes/sec)

**Solution**:
- Shard by merchant_id (64 shards)
- 20k writes/sec ÷ 64 = ~300 writes/sec per shard
- 3 replicas per shard (read scaling)

### 2. **Idempotency Cache Hot Keys**

**Problem**: Popular merchants cause Redis hot keys

**Solution**:
- Shard Redis by merchant_id hash
- Redis Cluster (6 nodes, 3 masters + 3 replicas)
- Local buffering (batch writes)

### 3. **Bank API Latency**

**Problem**: Bank API takes 200-500ms (external dependency)

**Solution**:
- Circuit breaker (fail fast on bank downtime)
- Async processing for non-critical paths
- Connection pooling (reuse HTTP connections)

### 4. **Webhook Delivery Failures**

**Problem**: Merchant webhook endpoints down → lost events

**Solution**:
- Retry with exponential backoff (1s, 2s, 4s, 8s, 16s)
- Dead letter queue (store failed events)
- Manual retry endpoint for merchants

---

## Common Anti-Patterns

### ❌ **1. Synchronous Webhook Delivery**

**Problem**: Payment service waits for merchant webhook response (blocks payment flow)

**Bad**:
```
POST /payments → Authorize → Call merchant webhook (wait 5s) → Return response
```

**Good**:
```
POST /payments → Authorize → Publish to Kafka → Return response (async webhook)
```

### ❌ **2. No Idempotency**

**Problem**: Network retry charges customer twice

**Bad**:
```
POST /payments → Process payment → Return (no idempotency check)
```

**Good**:
```
POST /payments {idempotency_key} → Check Redis → If exists return cached, else process
```

### ❌ **3. Storing Card Numbers in Main Database**

**Problem**: Entire system needs PCI-DSS certification (millions of dollars)

**Bad**:
```
payments table: {payment_id, card_number, amount}  // PCI scope: entire system
```

**Good**:
```
payments table: {payment_id, token, amount}  // PCI scope: only tokenization service
tokens table: {token, encrypted_card}  // Isolated, PCI-certified
```

### ❌ **4. No Audit Trail**

**Problem**: Can't reconcile with bank statements, compliance issues

**Bad**:
```
payments table: {payment_id, status}  // No history
```

**Good**:
```
payments table: {payment_id, status}
payment_events table: {event_id, payment_id, event_type, previous_status, new_status, created_at}
```

---

## Monitoring & Observability

### Key Metrics

- **Payment Success Rate**: >99.5% (authorization success)
- **Latency P99**: <500ms (authorization)
- **Idempotency Hit Rate**: ~5% (retries)
- **Fraud Block Rate**: ~2% (suspicious transactions)
- **Webhook Delivery Rate**: >99% (successful deliveries)
- **Database Replication Lag**: <100ms

### Alerts

- **Payment Failure Rate** >1% (5-minute window)
- **Latency P99** >1s
- **Database Replication Lag** >500ms
- **Redis Memory Usage** >80%
- **Bank API Error Rate** >5%

---

## Trade-offs Summary

| Decision | What We Gain | What We Sacrifice |
|----------|--------------|-------------------|
| **Authorization + Capture** | ✅ Merchant flexibility | ❌ Two API calls, more complexity |
| **Tokenization** | ✅ PCI scope reduction | ❌ Additional service to maintain |
| **PostgreSQL Ledger** | ✅ ACID guarantees, audit trail | ❌ Harder to scale (need sharding) |
| **Idempotency Keys** | ✅ Exactly-once processing | ❌ Client must generate UUIDs |
| **Async Webhooks** | ✅ Non-blocking payment flow | ❌ Merchant must handle async events |
| **Sharded Database** | ✅ Horizontal scaling | ❌ Cross-shard queries difficult |
| **Multi-Region** | ✅ Low latency globally | ❌ Complex replication, data residency |
| **Real-Time Fraud** | ✅ Block fraud before capture | ❌ Adds 50ms latency |

---

## Real-World Examples

### Stripe
- **Architecture**: Ruby on Rails monolith → microservices, PostgreSQL (sharded), Redis (idempotency)
- **Scale**: $640B payment volume (2021), 100+ countries
- **Innovation**: Developer-first API (simple integration)

### PayPal
- **Architecture**: Java microservices, Oracle → distributed systems, multi-region active-active
- **Scale**: $1.36T payment volume (2021), 400M active accounts
- **Innovation**: ML-based fraud detection (PayPal Risk Models)

### Adyen
- **Architecture**: Microservices, PostgreSQL, Redis, Kafka
- **Scale**: €516B payment volume (2021)
- **Innovation**: Unified payment platform (single integration for all payment methods)

---

## Key Takeaways

1. **Two-Phase Commit**: Authorization (reserve) + Capture (transfer) provides flexibility
2. **Tokenization**: Reduces PCI scope (only tokenization service handles raw cards)
3. **Idempotency**: Client-provided UUIDs prevent duplicate charges on retries
4. **ACID Database**: PostgreSQL for financial data (audit trail, consistency)
5. **Async Webhooks**: Non-blocking payment flow with retry logic
6. **Fraud Detection**: Real-time ML model blocks suspicious transactions
7. **Sharding Strategy**: Shard by merchant_id (queries scoped to merchant)
8. **Circuit Breaker**: Fail fast on bank API downtime

---

## Recommended Stack

- **API Gateway**: NGINX / AWS API Gateway
- **Payment Service**: Go / Java (high throughput)
- **Tokenization Service**: Isolated service (PCI-certified)
- **Ledger Database**: PostgreSQL (sharded by merchant_id)
- **Idempotency Cache**: Redis Cluster
- **Message Queue**: Kafka (webhook events)
- **Fraud Detection**: Python (ML models), TensorFlow
- **Monitoring**: Prometheus, Grafana, ELK Stack

