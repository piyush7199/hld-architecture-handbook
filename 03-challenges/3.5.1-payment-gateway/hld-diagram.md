# Payment Gateway - High-Level Design Diagrams

This document contains Mermaid diagrams showing the architecture and data flow for the Payment Gateway system.

---

## Table of Contents

1. [Overall System Architecture](#1-overall-system-architecture)
2. [Payment Flow (Authorization)](#2-payment-flow-authorization)
3. [Idempotency System](#3-idempotency-system)
4. [Tokenization Service](#4-tokenization-service)
5. [Database Sharding Architecture](#5-database-sharding-architecture)
6. [Fraud Detection Pipeline](#6-fraud-detection-pipeline)
7. [Webhook Delivery System](#7-webhook-delivery-system)
8. [Multi-Region Deployment](#8-multi-region-deployment)
9. [Circuit Breaker Pattern](#9-circuit-breaker-pattern)
10. [Subscription Billing Flow](#10-subscription-billing-flow)
11. [Split Payment Architecture](#11-split-payment-architecture)
12. [Monitoring and Observability](#12-monitoring-and-observability)

---

## 1. Overall System Architecture

**Flow Explanation:**

High-level view of all major components in the payment gateway system.

**Key Components:**

1. API Gateway handles TLS termination and rate limiting
2. Payment Service orchestrates transaction flow
3. Tokenization Service manages card data (PCI scope)
4. Fraud Detection scores transactions in real-time
5. Ledger Database stores immutable transaction records
6. Bank Processor integrates with Visa/Mastercard networks

**Performance:**

- Total latency: < 500ms for authorization
- Throughput: 20,000 QPS peak
- Availability: 99.99% uptime

```mermaid
graph TB
    Client[Merchant Website]

    subgraph API_Layer
        Gateway[API Gateway<br/>TLS, Rate Limiting<br/>Load Balancing]
    end

    subgraph Application_Layer
        PaymentSvc[Payment Service<br/>Orchestration<br/>200 instances]
        TokenSvc[Tokenization Service<br/>PCI Scope<br/>50 instances]
        FraudSvc[Fraud Detection<br/>ML-based<br/>30 instances]
    end

    subgraph Data_Layer
        Redis[Redis Cluster<br/>Idempotency Cache<br/>10 nodes]
        Postgres[(PostgreSQL<br/>Ledger DB<br/>64 shards)]
    end

    subgraph External
        Bank[Bank Processor API<br/>Visa, Mastercard]
    end

    Client --> Gateway
    Gateway --> PaymentSvc
    Gateway --> TokenSvc
    PaymentSvc --> Redis
    PaymentSvc --> FraudSvc
    PaymentSvc --> Postgres
    PaymentSvc --> Bank
    TokenSvc --> Postgres
    style PaymentSvc fill: #99ccff
    style Redis fill: #ff9999
    style Postgres fill: #99ff99
```

---

## 2. Payment Flow (Authorization)

**Flow Explanation:**

Two-phase commit pattern: Authorization reserves funds, Capture transfers money.

**Steps:**

1. Client sends payment request with idempotency key
2. Check Redis for duplicate request
3. Fraud detection scores transaction (< 50ms)
4. Call bank API for authorization
5. Write to ledger database
6. Return authorization result

**Benefits:**

- Merchant can cancel before capture
- Reduces refund rate
- Handles inventory checks

```mermaid
graph TB
    Client[Client/Merchant]

    subgraph Authorization_Flow
        API[API Gateway]
        IdemCheck{Idempotency<br/>Check}
        FraudCheck{Fraud<br/>Score}
        BankAuth[Bank<br/>Authorization]
        Ledger[(Ledger DB)]
    end

    Client -->|POST /payments<br/>idempotency_key| API
    API --> IdemCheck
    IdemCheck -->|Cache Hit| Client
    IdemCheck -->|Cache Miss| FraudCheck
    FraudCheck -->|Score < 0 . 5<br/>Allow| BankAuth
    FraudCheck -->|Score > 0 . 8<br/>Block| Client
    BankAuth -->|Auth Code| Ledger
    Ledger -->|Payment ID| API
    API -->|Status: authorized| Client
    style FraudCheck fill: #ffcc99
    style BankAuth fill: #99ccff
    style Ledger fill: #99ff99
```

---

## 3. Idempotency System

**Flow Explanation:**

Prevents double charging when clients retry failed requests.

**Mechanism:**

1. Client generates UUID (idempotency_key)
2. Server checks Redis cache
3. If exists, return cached result
4. If not, process and cache result (24h TTL)

**Edge Cases:**

- Concurrent requests: Use Redis SETNX for atomic check
- Expired keys: Allow reprocessing after 24h
- Redis failure: Fall back to database query

```mermaid
graph LR
    Client[Client Request<br/>idempotency_key]
    Redis[(Redis Cache<br/>TTL: 24h)]
    Process[Process Payment]
    DB[(Database)]
    Client -->|1 . Check| Redis
    Redis -->|2a . Hit<br/>Return cached| Client
    Redis -->|2b . Miss| Process
    Process -->|3 . Execute| DB
    DB -->|4 . Result| Process
    Process -->|5 . Cache| Redis
    Process -->|6 . Return| Client
    style Redis fill: #ff9999
    style Process fill: #99ccff
```

---

## 4. Tokenization Service

**Flow Explanation:**

Replaces sensitive card data with non-sensitive tokens to reduce PCI scope.

**Process:**

1. Client sends card number
2. Validate with Luhn algorithm
3. Generate unique token
4. Encrypt card with AES-256
5. Store encrypted card
6. Return token to client

**Security:**

- Only service in PCI scope
- Isolated VPC
- HSM for key management

```mermaid
graph TB
    Client[Client]

    subgraph Tokenization_PCI_Scope
        Validate[Validate Card<br/>Luhn Algorithm]
        Generate[Generate Token<br/>tok_abc123]
        Encrypt[Encrypt Card<br/>AES-256]
        Store[(Token DB<br/>Encrypted)]
    end

    KMS[AWS KMS<br/>Key Management]
    Client -->|POST /tokens<br/>4111111111111111| Validate
    Validate -->|Valid| Generate
    Generate --> Encrypt
    Encrypt -->|Get Key| KMS
    Encrypt --> Store
    Store -->|Token| Client
    style Tokenization_PCI_Scope fill: #ffcccc
    style KMS fill: #ff9999
```

---

## 5. Database Sharding Architecture

**Flow Explanation:**

Horizontal sharding by merchant_id distributes load across 64 PostgreSQL shards.

**Sharding Strategy:**

- Shard key: merchant_id
- Shard calculation: hash(merchant_id) % 64
- Each shard: 3 replicas (primary + 2 read replicas)

**Benefits:**

- Linear scalability
- All merchant data on same shard
- Enables merchant-scoped queries

```mermaid
graph TB
    Client[API Request<br/>merchant_id: 12345]
    Router[Shard Router]

    subgraph Shard_0
        S0P[(Shard 0<br/>Primary)]
        S0R1[(Replica 1)]
        S0R2[(Replica 2)]
    end

    subgraph Shard_1
        S1P[(Shard 1<br/>Primary)]
        S1R1[(Replica 1)]
        S1R2[(Replica 2)]
    end

    subgraph Shard_63
        S63P[(Shard 63<br/>Primary)]
        S63R1[(Replica 1)]
        S63R2[(Replica 2)]
    end

    Client --> Router
    Router -->|hash % 64 = 0| S0P
    Router -->|hash % 64 = 1| S1P
    Router -->|hash % 64 = 63| S63P
    S0P -.->|Replication| S0R1
    S0P -.->|Replication| S0R2
    style Router fill: #99ccff
    style S0P fill: #99ff99
```

---

## 6. Fraud Detection Pipeline

**Flow Explanation:**

Real-time ML-based fraud scoring happens synchronously during authorization.

**Features:**

- Transaction: amount, currency, time
- Card: brand, country, decline history
- Customer: email, IP, device fingerprint
- Behavioral: velocity, spending patterns

**Actions:**

- Risk < 0.5: Auto-approve
- Risk 0.5-0.8: 3D Secure challenge
- Risk > 0.8: Block transaction

```mermaid
graph TB
    Payment[Payment Request]

    subgraph Feature_Engineering
        TxnFeatures[Transaction Features<br/>Amount, Currency, Time]
        CardFeatures[Card Features<br/>Brand, Country, History]
        UserFeatures[User Features<br/>Email, IP, Device]
    end

    subgraph ML_Model
        XGBoost[XGBoost Model<br/>30 Features<br/>Inference: 20ms]
    end

    subgraph Decision_Engine
        LowRisk{Risk < 0.5}
        MedRisk{Risk 0.5-0.8}
        HighRisk{Risk > 0.8}
    end

    Payment --> TxnFeatures
    Payment --> CardFeatures
    Payment --> UserFeatures
    TxnFeatures --> XGBoost
    CardFeatures --> XGBoost
    UserFeatures --> XGBoost
    XGBoost --> LowRisk
    LowRisk -->|Yes| Approve[Approve]
    LowRisk -->|No| MedRisk
    MedRisk -->|Yes| Challenge[3DS Challenge]
    MedRisk -->|No| HighRisk
    HighRisk --> Block[Block Transaction]
    style XGBoost fill: #ffcc99
    style Approve fill: #99ff99
    style Block fill: #ff9999
```

---

## 7. Webhook Delivery System

**Flow Explanation:**

Asynchronous webhook delivery with retry logic ensures merchants receive event notifications.

**Delivery Guarantees:**

- At-least-once delivery
- Up to 10 retry attempts
- Exponential backoff: 1m, 5m, 30m, 1h, 6h, 12h, 24h

**Signature:**

- HMAC-SHA256 signature in header
- Merchant verifies authenticity

```mermaid
graph TB
    Event[Payment Event<br/>captured, refunded]

    subgraph Queue
        Kafka[Kafka Topic<br/>webhook-events]
    end

    subgraph Worker
        Consumer[Webhook Worker]
        Retry{Retry<br/>Logic}
    end

    Merchant[Merchant Endpoint<br/>POST webhook_url]
    DLQ[Dead Letter Queue<br/>After 10 retries]
    Event -->|Publish| Kafka
    Kafka --> Consumer
    Consumer -->|HMAC signature| Merchant
    Merchant -->|200 OK| Consumer
    Merchant -->|5xx/timeout| Retry
    Retry -->|Attempt < 10| Kafka
    Retry -->|Attempt > = 10| DLQ
    style Kafka fill: #99ccff
    style Merchant fill: #99ff99
    style DLQ fill: #ff9999
```

---

## 8. Multi-Region Deployment

**Flow Explanation:**

Active-active deployment across 4 regions for low latency and high availability.

**Regions:**

- US-East-1: Primary, largest capacity
- US-West-2: Secondary US
- EU-West-1: European users (GDPR)
- AP-Southeast-1: Asian users

**Routing:**

- Route53 latency-based routing
- Async database replication (1-2s lag)

```mermaid
graph TB
    Client[Global Clients]
    DNS[Route53<br/>Latency-Based Routing]

    subgraph US_East
        USE_API[API Gateway]
        USE_DB[(DB Shards<br/>16 per region)]
    end

    subgraph US_West
        USW_API[API Gateway]
        USW_DB[(DB Shards)]
    end

    subgraph EU_West
        EU_API[API Gateway]
        EU_DB[(DB Shards)]
    end

    subgraph Asia_Pacific
        AP_API[API Gateway]
        AP_DB[(DB Shards)]
    end

    Client --> DNS
    DNS -->|US users| USE_API
    DNS -->|EU users| EU_API
    DNS -->|Asia users| AP_API
    USE_API --> USE_DB
    EU_API --> EU_DB
    AP_API --> AP_DB
    USE_DB -.->|Async replication| USW_DB
    USE_DB -.->|Async replication| EU_DB
    USE_DB -.->|Async replication| AP_DB
    style DNS fill: #99ccff
```

---

## 9. Circuit Breaker Pattern

**Flow Explanation:**

Protects system from cascading failures when bank API is slow or unavailable.

**States:**

- Closed: Normal operation, requests pass through
- Open: Bank is down, fail fast (don't call bank)
- Half-Open: Testing recovery, send probe request

**Configuration:**

- Failure threshold: 50% error rate over 10 seconds
- Cooldown: 60 seconds
- Success threshold: 5 consecutive successes

```mermaid
stateDiagram-v2
    [*] --> Closed: Initial State
    Closed --> Open: Error rate > 50%<br/>over 10 seconds
    Closed --> Closed: Success
    Open --> HalfOpen: After 60s cooldown
    Open --> Open: Request blocked<br/>(fail fast)
    HalfOpen --> Closed: 5 consecutive<br/>successes
    HalfOpen --> Open: Single failure
    note right of Closed
        Normal operation
        All requests pass through
        Monitor error rate
    end note
    note right of Open
        Bank API down
        Fail fast (no calls)
        Wait for cooldown
    end note
    note right of HalfOpen
        Testing recovery
        Limited requests
        Quick decision
    end note
```

---

## 10. Subscription Billing Flow

**Flow Explanation:**

Automated recurring billing for subscription services (Netflix, Spotify model).

**Lifecycle:**

1. Customer creates subscription
2. Daily cron checks for due subscriptions
3. Charge payment method on billing date
4. Retry failed charges (Day 0, 3, 7)
5. Mark past_due if all retries fail
6. Cancel after 30 days past_due

```mermaid
graph TB
    Customer[Customer<br/>Creates Subscription]

    subgraph Subscription_DB
        Sub[(Subscriptions<br/>current_period_end)]
    end

    subgraph Billing_Cron
        Daily[Daily Job<br/>00:00 UTC]
        Check{Due<br/>Today?}
    end

    Charge[Charge Payment Method]
    Success{Success?}
    Retry{Retry<br/>Count}
    Customer --> Sub
    Daily -->|Query| Sub
    Sub --> Check
    Check -->|Yes| Charge
    Check -->|No| Daily
    Charge --> Success
    Success -->|Yes| Extend[Extend Period<br/>+1 month]
    Success -->|No| Retry
    Retry -->|< 3| Schedule[Schedule Retry<br/>Day 3, 7]
    Retry -->|> = 3| PastDue[Mark past_due<br/>Email customer]
    PastDue -->|30 days| Cancel[Cancel Subscription]
    style Charge fill: #99ccff
    style Extend fill: #99ff99
    style Cancel fill: #ff9999
```

---

## 11. Split Payment Architecture

**Flow Explanation:**

Marketplace model where platform takes commission and transfers funds to merchants (Uber, Airbnb).

**Money Flow:**

- Customer pays $100
- Platform receives $20 (application_fee)
- Merchant receives $80 (automatic transfer)

**Use Cases:**

- Ride-sharing platforms
- Freelance marketplaces
- E-commerce marketplaces

```mermaid
graph TB
    Customer[Customer]

    subgraph Platform
        API[Payment API]
        Fee[Application Fee<br/>20%]
    end

    subgraph Connected_Account
        Merchant[Merchant/Driver<br/>Connected Account]
        Transfer[Auto Transfer<br/>80%]
    end

    Bank[Bank Account<br/>Payout]
    Customer -->|Pay $100| API
    API -->|Platform keeps| Fee
    API -->|Create transfer| Transfer
    Transfer -->|$80| Merchant
    Merchant -->|Daily payout| Bank
    style API fill: #99ccff
    style Fee fill: #ffcc99
    style Transfer fill: #99ff99
```

---

## 12. Monitoring and Observability

**Flow Explanation:**

Comprehensive monitoring of business and system metrics.

**Key Metrics:**

- Authorization success rate (target: > 98%)
- Fraud detection rate (target: 0.1%)
- API latency p95 (target: < 500ms)
- Database query time (target: < 50ms)

**Alerting:**

- Critical: Authorization rate < 95% → Page on-call
- High: Latency > 1s → Slack alert
- Medium: Fraud spike → Email security team

```mermaid
graph TB
    subgraph Services
        API[API Gateway]
        Payment[Payment Service]
        DB[(Database)]
    end

    subgraph Metrics_Collection
        Prometheus[Prometheus<br/>Metrics Collection]
        Logs[Elasticsearch<br/>Log Aggregation]
    end

    subgraph Visualization
        Grafana[Grafana<br/>Dashboards]
    end

    subgraph Alerting
        AlertMgr[Alert Manager]
        PagerDuty[PagerDuty]
        Slack[Slack]
    end

    API -->|Metrics| Prometheus
    Payment -->|Metrics| Prometheus
    DB -->|Metrics| Prometheus
    API -->|Logs| Logs
    Payment -->|Logs| Logs
    Prometheus --> Grafana
    Logs --> Grafana
    Prometheus --> AlertMgr
    AlertMgr -->|Critical| PagerDuty
    AlertMgr -->|High| Slack
    style Prometheus fill: #99ccff
    style Grafana fill: #99ff99
    style PagerDuty fill: #ff9999
```