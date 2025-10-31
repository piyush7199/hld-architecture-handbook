# Payment Gateway - Sequence Diagrams

This document contains detailed sequence diagrams showing interaction flows for the Payment Gateway system.

---

## Table of Contents

1. [Payment Authorization Flow](#1-payment-authorization-flow)
2. [Payment Capture Flow](#2-payment-capture-flow)
3. [Idempotent Retry Handling](#3-idempotent-retry-handling)
4. [Tokenization Flow](#4-tokenization-flow)
5. [Refund Processing](#5-refund-processing)
6. [Fraud Detection Flow](#6-fraud-detection-flow)
7. [3D Secure Authentication](#7-3d-secure-authentication)
8. [Webhook Delivery with Retries](#8-webhook-delivery-with-retries)
9. [Subscription Billing](#9-subscription-billing)
10. [Circuit Breaker Activation](#10-circuit-breaker-activation)

---

## 1. Payment Authorization Flow

**Flow:**

Complete flow from client payment request to authorized status.

**Steps:**

1. Client sends payment request (0ms)
2. API Gateway validates and routes (10ms)
3. Check idempotency cache (5ms)
4. Fraud detection scores transaction (50ms)
5. Call bank API for authorization (200ms)
6. Write to ledger database (20ms)
7. Cache idempotency result (5ms)
8. Return authorization response (10ms)

**Total Latency**: ~300ms

```mermaid
sequenceDiagram
    participant Client
    participant APIGateway
    participant PaymentService
    participant Redis
    participant FraudService
    participant BankAPI
    participant LedgerDB
    Client ->> APIGateway: POST /payments<br/>{amount, token, idempotency_key}
    APIGateway ->> APIGateway: Validate API key<br/>Rate limit check
    APIGateway ->> PaymentService: Forward request
    PaymentService ->> Redis: GET idempotency:key
    Redis -->> PaymentService: Cache miss
    PaymentService ->> FraudService: Calculate fraud score
    FraudService ->> FraudService: Load features<br/>Run ML model
    FraudService -->> PaymentService: Risk score: 0.3 (low)
    PaymentService ->> BankAPI: Authorize payment
    BankAPI ->> BankAPI: Check funds<br/>Reserve amount
    BankAPI -->> PaymentService: Auth code: ABC123
    PaymentService ->> LedgerDB: INSERT payment<br/>status=authorized
    LedgerDB -->> PaymentService: payment_id: pay_123
    PaymentService ->> Redis: SET idempotency:key<br/>TTL 24h
    Redis -->> PaymentService: Cached
    PaymentService -->> APIGateway: {payment_id, status: authorized}
    APIGateway -->> Client: 200 OK<br/>{payment_id: pay_123}
    Note over Client, LedgerDB: Total latency: ~300ms
```

---

## 2. Payment Capture Flow

**Flow:**

Merchant captures previously authorized payment (after shipment confirmation).

**Steps:**

1. Merchant requests capture (0ms)
2. Verify payment exists and is authorized (10ms)
3. Call bank API to transfer funds (200ms)
4. Update ledger with captured status (20ms)
5. Trigger webhook event (5ms async)
6. Return capture confirmation (10ms)

**Total Latency**: ~245ms

```mermaid
sequenceDiagram
    participant Merchant
    participant PaymentService
    participant LedgerDB
    participant BankAPI
    participant Kafka
    participant WebhookWorker
    Merchant ->> PaymentService: POST /payments/pay_123/capture
    PaymentService ->> LedgerDB: SELECT * FROM payments<br/>WHERE payment_id=pay_123
    LedgerDB -->> PaymentService: status=authorized,<br/>auth_code=ABC123
    PaymentService ->> PaymentService: Validate can capture<br/>(status must be authorized)
    PaymentService ->> BankAPI: Capture payment<br/>auth_code=ABC123
    BankAPI ->> BankAPI: Transfer funds<br/>Debit customer,<br/>Credit merchant
    BankAPI -->> PaymentService: Settlement ID: XYZ789
    PaymentService ->> LedgerDB: UPDATE payments<br/>SET status=captured,<br/>captured_at=NOW()
    LedgerDB -->> PaymentService: Updated
    PaymentService ->> Kafka: Publish event<br/>payment.captured
    Kafka -->> WebhookWorker: Event queued
    PaymentService -->> Merchant: 200 OK<br/>{status: captured}
    WebhookWorker ->> Merchant: POST webhook_url<br/>{event: payment.captured}
    Merchant -->> WebhookWorker: 200 OK
```

---

## 3. Idempotent Retry Handling

**Flow:**

Client retries failed request with same idempotency key (network issue).

**Steps:**

1. First request: Process normally, cache result
2. Network failure: Client doesn't receive response
3. Retry: Same idempotency key
4. Cache hit: Return cached result (no reprocessing)

**Key Point**: Payment only processed once, but client gets result twice.

```mermaid
sequenceDiagram
    participant Client
    participant PaymentService
    participant Redis
    participant BankAPI
    participant LedgerDB
    Note over Client: Request 1 (Original)
    Client ->> PaymentService: POST /payments<br/>idempotency_key=idem_123
    PaymentService ->> Redis: GET idempotency:idem_123
    Redis -->> PaymentService: null (cache miss)
    PaymentService ->> BankAPI: Authorize
    BankAPI -->> PaymentService: Authorized
    PaymentService ->> LedgerDB: Write payment
    PaymentService ->> Redis: SET idempotency:idem_123<br/>{payment_id: pay_abc, ...}
    PaymentService ->> Client: 200 OK {payment_id: pay_abc}
    Note over Client, PaymentService: Network failure!<br/>Client doesn't receive
    Note over Client: Request 2 (Retry)<br/>Same idempotency key
    Client ->> PaymentService: POST /payments<br/>idempotency_key=idem_123<br/>(same key!)
    PaymentService ->> Redis: GET idempotency:idem_123
    Redis -->> PaymentService: {payment_id: pay_abc, ...}<br/>(cache hit!)
    Note over PaymentService: No bank call!<br/>No DB write!<br/>Return cached result
    PaymentService -->> Client: 200 OK {payment_id: pay_abc}<br/>(same response)
    Note over Client, LedgerDB: Customer charged once,<br/>client gets result twice
```

---

## 4. Tokenization Flow

**Flow:**

Convert sensitive card number to non-sensitive token.

**Steps:**

1. Client sends card number (never stored raw)
2. Validate card with Luhn algorithm (10ms)
3. Generate unique token (5ms)
4. Encrypt card with AES-256 (10ms)
5. Store encrypted card in database (20ms)
6. Return token to client (10ms)

**Total Latency**: ~55ms

```mermaid
sequenceDiagram
    participant Client
    participant TokenService
    participant KMS
    participant TokenDB
    Client ->> TokenService: POST /tokens<br/>{card_number: 4111111111111111}
    TokenService ->> TokenService: Validate with<br/>Luhn algorithm
    TokenService ->> TokenService: Generate token<br/>tok_abc123
    TokenService ->> TokenService: Hash card<br/>SHA-256 fingerprint<br/>(for deduplication)
    TokenService ->> KMS: Get encryption key
    KMS -->> TokenService: AES-256 key
    TokenService ->> TokenService: Encrypt card<br/>encrypted_card=AES(card)
    TokenService ->> TokenDB: INSERT tokens<br/>(token_id, encrypted_card,<br/>fingerprint, last4)
    TokenDB -->> TokenService: Inserted
    TokenService -->> Client: 200 OK<br/>{token: tok_abc123,<br/>last4: 1111,<br/>brand: Visa}
    Note over Client, TokenDB: Raw card never stored<br/>Only encrypted version
```

---

## 5. Refund Processing

**Flow:**

Merchant initiates refund for captured payment.

**Steps:**

1. Merchant requests refund (0ms)
2. Verify payment captured and amount valid (10ms)
3. Create refund record (20ms)
4. Call bank API to process refund (200ms)
5. Update refund status (20ms)
6. Trigger webhook (5ms async)

**Total Latency**: ~255ms

```mermaid
sequenceDiagram
    participant Merchant
    participant PaymentService
    participant LedgerDB
    participant BankAPI
    participant Kafka
    Merchant ->> PaymentService: POST /payments/pay_123/refunds<br/>{amount: 5000, reason: "..."}
    PaymentService ->> LedgerDB: SELECT * FROM payments<br/>WHERE payment_id=pay_123
    LedgerDB -->> PaymentService: status=captured,<br/>amount=10000
    PaymentService ->> PaymentService: Validate:<br/>- Payment captured<br/>- Refund <= captured amount<br/>- Not already refunded
    PaymentService ->> LedgerDB: INSERT refunds<br/>(refund_id, payment_id,<br/>amount, status=pending)
    LedgerDB -->> PaymentService: refund_id: ref_xyz
    PaymentService ->> BankAPI: Process refund<br/>amount=5000
    BankAPI ->> BankAPI: Credit customer<br/>Debit merchant
    BankAPI -->> PaymentService: Refund ID: REFUND789
    PaymentService ->> LedgerDB: UPDATE refunds<br/>SET status=completed
    PaymentService ->> Kafka: Publish<br/>payment.refunded
    PaymentService -->> Merchant: 200 OK<br/>{refund_id: ref_xyz,<br/>status: completed}
    Note over Merchant, Kafka: Refund typically takes<br/>5-10 days to appear<br/>on customer statement
```

---

## 6. Fraud Detection Flow

**Flow:**

Real-time fraud scoring during authorization.

**Steps:**

1. Payment request arrives
2. Extract features (transaction, card, user)
3. Query feature store (Redis) for historical data
4. Run ML model (20ms)
5. Make decision based on score
6. Block high-risk, challenge medium-risk, approve low-risk

```mermaid
sequenceDiagram
    participant PaymentService
    participant FeatureStore
    participant MLModel
    participant BankAPI
    participant Customer
    PaymentService ->> PaymentService: Extract features<br/>- Amount: $10,000<br/>- New customer<br/>- High-risk country
    PaymentService ->> FeatureStore: Get historical features<br/>(card decline rate,<br/>user transaction history)
    FeatureStore -->> PaymentService: Features loaded
    PaymentService ->> MLModel: Score transaction<br/>30 features
    MLModel ->> MLModel: Run XGBoost model<br/>(20ms inference)
    MLModel -->> PaymentService: Risk score: 0.85<br/>(HIGH RISK)

    alt Risk > 0.8 (High Risk)
        PaymentService -->> PaymentService: Block transaction
        PaymentService -->> Customer: 402 Payment Required<br/>{error: fraud_suspected}
    else Risk 0.5-0.8 (Medium Risk)
        PaymentService -->> Customer: Require 3D Secure<br/>{next_action: 3ds_challenge}
    else Risk < 0.5 (Low Risk)
        PaymentService ->> BankAPI: Authorize payment
        BankAPI -->> PaymentService: Authorized
        PaymentService -->> Customer: 200 OK
    end

    Note over PaymentService, Customer: Decision made in < 50ms
```

---

## 7. 3D Secure Authentication

**Flow:**

Customer completes 3D Secure challenge for medium-risk transaction.

**Steps:**

1. Initial auth attempt → fraud score 0.65 (medium)
2. Redirect to 3DS page
3. Customer enters OTP sent to phone
4. Bank verifies OTP
5. Complete authorization
6. Liability shifts to bank

```mermaid
sequenceDiagram
    participant Customer
    participant Merchant
    participant PaymentService
    participant ThreeDSProvider
    participant Bank
    Customer ->> Merchant: Checkout<br/>Pay $500
    Merchant ->> PaymentService: POST /payments<br/>{amount, token}
    PaymentService ->> PaymentService: Fraud score: 0.65<br/>(Medium risk)
    PaymentService -->> Merchant: 200 OK<br/>{status: requires_action,<br/>next_action: {type: 3ds,<br/>url: https://3ds.com/...}}
    Merchant ->> Customer: Redirect to 3DS page
    Customer ->> ThreeDSProvider: Load 3DS challenge
    ThreeDSProvider ->> Bank: Request OTP
    Bank ->> Customer: SMS OTP: 123456
    Customer ->> ThreeDSProvider: Enter OTP: 123456
    ThreeDSProvider ->> Bank: Verify OTP
    Bank -->> ThreeDSProvider: Verified
    ThreeDSProvider ->> PaymentService: POST /payments/pay_123/confirm<br/>{3ds_result: success}
    PaymentService ->> Bank: Authorize with 3DS proof
    Bank -->> PaymentService: Authorized<br/>(Liability shifted)
    PaymentService -->> ThreeDSProvider: Success
    ThreeDSProvider ->> Customer: Redirect to success_url
    Note over Customer, Bank: Chargeback liability<br/>now with issuing bank
```

---

## 8. Webhook Delivery with Retries

**Flow:**

Webhook delivery with exponential backoff retry strategy.

**Steps:**

1. Payment event occurs (captured)
2. Publish to Kafka queue
3. Worker picks up event
4. Attempt delivery to merchant endpoint
5. If failure, retry with exponential backoff
6. After 10 retries, move to dead letter queue

```mermaid
sequenceDiagram
    participant PaymentService
    participant Kafka
    participant WebhookWorker
    participant Merchant
    participant DLQ
    PaymentService ->> Kafka: Publish event<br/>payment.captured
    Kafka -->> WebhookWorker: Poll event
    WebhookWorker ->> WebhookWorker: Generate HMAC signature<br/>HMAC-SHA256(payload, secret)
    Note over WebhookWorker, Merchant: Attempt 1 (Immediate)
    WebhookWorker ->> Merchant: POST webhook_url<br/>X-Signature: sha256=...
    Merchant -->> WebhookWorker: 500 Internal Error
    Note over WebhookWorker: Retry attempt 1 failed
    Note over WebhookWorker, Merchant: Attempt 2 (1 minute later)
    WebhookWorker ->> Merchant: POST webhook_url
    Merchant -->> WebhookWorker: Timeout (no response)
    Note over WebhookWorker, Merchant: Attempt 3 (5 minutes later)
    WebhookWorker ->> Merchant: POST webhook_url
    Merchant -->> WebhookWorker: 200 OK
    WebhookWorker ->> Kafka: ACK message<br/>(delivered successfully)

    alt All 10 retries fail
        WebhookWorker ->> DLQ: Move to DLQ<br/>for manual review
        WebhookWorker ->> PaymentService: Alert merchant<br/>webhook delivery failed
    end

    Note over WebhookWorker, DLQ: Retry schedule:<br/>1m, 5m, 30m, 1h,<br/>6h, 12h, 24h
```

---

## 9. Subscription Billing

**Flow:**

Automated monthly subscription billing.

**Steps:**

1. Daily cron checks for due subscriptions
2. Query subscriptions ending today
3. For each subscription, charge payment method
4. If success, extend billing period
5. If failure, retry with exponential backoff
6. After 3 failures, mark past_due

```mermaid
sequenceDiagram
    participant Cron
    participant SubscriptionService
    participant PaymentService
    participant DB
    participant Customer
    Cron ->> SubscriptionService: Trigger daily billing<br/>(00:00 UTC)
    SubscriptionService ->> DB: SELECT * FROM subscriptions<br/>WHERE current_period_end = TODAY()<br/>AND status = active
    DB -->> SubscriptionService: 1,000 subscriptions due

    loop For each subscription
        SubscriptionService ->> PaymentService: POST /payments<br/>{amount: 9.99,<br/>token: saved_payment_method}

        alt Payment succeeds
            PaymentService -->> SubscriptionService: 200 OK
            SubscriptionService ->> DB: UPDATE subscriptions<br/>SET current_period_end<br/>= current_period_end + 1 month
            SubscriptionService ->> Customer: Email: Payment successful
        else Payment fails
            PaymentService -->> SubscriptionService: 402 Payment Failed
            SubscriptionService ->> DB: UPDATE retry_count = 1,<br/>next_retry_date = TODAY() + 3
            SubscriptionService ->> Customer: Email: Payment failed,<br/>will retry in 3 days
        end
    end

    Note over Cron, Customer: Retry schedule:<br/>Day 0, 3, 7<br/>After 3 failures:<br/>mark past_due
```

---

## 10. Circuit Breaker Activation

**Flow:**

Circuit breaker opens when bank API fails repeatedly.

**Steps:**

1. Normal: Requests pass through (CLOSED state)
2. Bank API starts failing (50% error rate)
3. Circuit opens after threshold (OPEN state)
4. Requests fail fast (no bank calls)
5. After cooldown, test with probe (HALF-OPEN)
6. If successful, close circuit

```mermaid
sequenceDiagram
    participant Client
    participant PaymentService
    participant CircuitBreaker
    participant BankAPI
    Note over CircuitBreaker: State: CLOSED<br/>(Normal operation)

    loop Requests 1-10
        Client ->> PaymentService: POST /payments
        PaymentService ->> CircuitBreaker: Check state
        CircuitBreaker -->> PaymentService: CLOSED (allow)
        PaymentService ->> BankAPI: Authorize
        BankAPI -->> PaymentService: 500 Internal Error
        CircuitBreaker ->> CircuitBreaker: Track failure<br/>Error rate: 60%
    end

    CircuitBreaker ->> CircuitBreaker: Threshold exceeded<br/>Open circuit!
    Note over CircuitBreaker: State: OPEN<br/>(Fail fast)

    loop Requests 11-20
        Client ->> PaymentService: POST /payments
        PaymentService ->> CircuitBreaker: Check state
        CircuitBreaker -->> PaymentService: OPEN (block)
        PaymentService -->> Client: 503 Service Unavailable<br/>(No bank call)
    end

    Note over CircuitBreaker: Wait 60 seconds cooldown
    CircuitBreaker ->> CircuitBreaker: Enter HALF-OPEN<br/>Test with probe
    Note over CircuitBreaker: State: HALF-OPEN<br/>(Testing)
    Client ->> PaymentService: POST /payments (probe)
    PaymentService ->> BankAPI: Authorize (test)
    BankAPI -->> PaymentService: 200 OK (success!)
    CircuitBreaker ->> CircuitBreaker: Success!<br/>Close circuit
    Note over CircuitBreaker: State: CLOSED<br/>(Normal restored)
```