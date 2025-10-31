# Payment Gateway - Pseudocode Implementations

This document contains detailed algorithm implementations for the Payment Gateway system. The main challenge document
references these functions.

---

## Table of Contents

1. [Payment Processing](#1-payment-processing)
2. [Idempotency Management](#2-idempotency-management)
3. [Tokenization](#3-tokenization)
4. [Fraud Detection](#4-fraud-detection)
5. [Refund Processing](#5-refund-processing)
6. [Webhook Delivery](#6-webhook-delivery)
7. [Circuit Breaker](#7-circuit-breaker)
8. [Subscription Billing](#8-subscription-billing)
9. [Sharding](#9-sharding)
10. [3D Secure](#10-3d-secure)

---

## 1. Payment Processing

### authorize_payment()

**Purpose:** Create payment authorization and reserve funds

**Parameters:**

- merchant_id (string): Merchant identifier
- amount_cents (integer): Payment amount in cents
- currency (string): ISO currency code (USD, EUR)
- token_id (string): Tokenized card reference
- idempotency_key (string): UUID for duplicate prevention

**Returns:**

- Payment: {payment_id, status, auth_code, expires_at}

**Algorithm:**

```
function authorize_payment(merchant_id, amount_cents, currency, token_id, idempotency_key):
    // Step 1: Check idempotency
    cached_result = check_idempotency(merchant_id, idempotency_key)
    if cached_result is not null:
        return cached_result
    
    // Step 2: Validate inputs
    validate_merchant(merchant_id)
    validate_amount(amount_cents, currency)
    validate_token(token_id, merchant_id)
    
    // Step 3: Get card details from token
    card = detokenize_card(token_id)
    if card is null:
        throw InvalidTokenError()
    
    // Step 4: Fraud detection
    fraud_score = calculate_fraud_score(merchant_id, card, amount_cents)
    if fraud_score > 0.8:
        return {status: "blocked", reason: "fraud_suspected"}
    if fraud_score > 0.5:
        return {status: "requires_action", next_action: "3ds_challenge"}
    
    // Step 5: Call bank API
    try:
        bank_response = bank_api.authorize(
            card_number: decrypt(card.encrypted_number),
            exp_month: card.exp_month,
            exp_year: card.exp_year,
            amount: amount_cents,
            currency: currency
        )
    catch BankAPIError as e:
        handle_bank_error(e)
        throw PaymentFailedError(e.message)
    
    // Step 6: Create payment record
    payment_id = generate_id("pay")
    expires_at = now() + days(7)
    
    payment = db.insert({
        table: "payments",
        values: {
            payment_id: payment_id,
            merchant_id: merchant_id,
            amount_cents: amount_cents,
            currency: currency,
            token_id: token_id,
            status: "authorized",
            auth_code: bank_response.auth_code,
            expires_at: expires_at,
            created_at: now()
        }
    })
    
    // Step 7: Cache idempotency result
    cache_idempotency(merchant_id, idempotency_key, payment, ttl_seconds: 86400)
    
    // Step 8: Trigger events
    publish_event("payment.authorized", payment)
    
    return payment
```

**Time Complexity:** O(1) average case (O(log n) for database insert)

**Example Usage:**

```
payment = authorize_payment(
    merchant_id: "merch_123",
    amount_cents: 10000,  // $100.00
    currency: "usd",
    token_id: "tok_abc456",
    idempotency_key: "idem_xyz789"
)
// Returns: {payment_id: "pay_123", status: "authorized", auth_code: "ABC123"}
```

---

### capture_payment()

**Purpose:** Capture previously authorized payment and transfer funds

**Parameters:**

- payment_id (string): Payment identifier
- amount_cents (integer): Optional partial capture amount
- idempotency_key (string): UUID for duplicate prevention

**Returns:**

- Payment: Updated payment object with status "captured"

**Algorithm:**

```
function capture_payment(payment_id, amount_cents, idempotency_key):
    // Step 1: Check idempotency
    cache_key = f"capture:{payment_id}:{idempotency_key}"
    cached_result = redis.get(cache_key)
    if cached_result:
        return cached_result
    
    // Step 2: Load payment
    payment = db.select("payments").where(payment_id: payment_id).first()
    if payment is null:
        throw PaymentNotFoundError()
    
    // Step 3: Validate can capture
    if payment.status != "authorized":
        throw InvalidStateError(f"Cannot capture payment in status {payment.status}")
    
    if now() > payment.expires_at:
        throw AuthorizationExpiredError()
    
    // Validate amount (partial capture)
    if amount_cents is not null:
        if amount_cents > payment.amount_cents:
            throw InvalidAmountError("Cannot capture more than authorized")
        capture_amount = amount_cents
    else:
        capture_amount = payment.amount_cents
    
    // Step 4: Call bank API to transfer funds
    try:
        bank_response = bank_api.capture(
            auth_code: payment.auth_code,
            amount: capture_amount
        )
    catch BankAPIError as e:
        handle_bank_error(e)
        throw CaptureFailedError(e.message)
    
    // Step 5: Update payment record
    db.update("payments").where(payment_id: payment_id).set({
        status: "captured",
        captured_amount_cents: capture_amount,
        settlement_id: bank_response.settlement_id,
        captured_at: now()
    })
    
    payment.status = "captured"
    payment.captured_amount_cents = capture_amount
    
    // Step 6: Cache result
    redis.setex(cache_key, 86400, payment)
    
    // Step 7: Trigger webhook
    publish_event("payment.captured", payment)
    
    return payment
```

**Time Complexity:** O(1)

**Example Usage:**

```
payment = capture_payment(
    payment_id: "pay_123",
    amount_cents: null,  // Full capture
    idempotency_key: "idem_capture_123"
)
// Returns: {payment_id: "pay_123", status: "captured"}
```

---

### cancel_payment()

**Purpose:** Cancel authorized payment and release hold

**Parameters:**

- payment_id (string): Payment identifier

**Returns:**

- Payment: Updated payment with status "canceled"

**Algorithm:**

```
function cancel_payment(payment_id):
    // Load payment
    payment = db.select("payments").where(payment_id: payment_id).first()
    
    if payment.status != "authorized":
        throw InvalidStateError("Can only cancel authorized payments")
    
    // Call bank API to void authorization
    try:
        bank_api.void(auth_code: payment.auth_code)
    catch BankAPIError as e:
        log_error(f"Bank void failed: {e}")
        // Continue anyway (best effort)
    
    // Update status
    db.update("payments").where(payment_id: payment_id).set({
        status: "canceled",
        canceled_at: now()
    })
    
    payment.status = "canceled"
    
    // Trigger event
    publish_event("payment.canceled", payment)
    
    return payment
```

**Time Complexity:** O(1)

---

## 2. Idempotency Management

### check_idempotency()

**Purpose:** Check if request already processed

**Parameters:**

- merchant_id (string): Merchant identifier
- idempotency_key (string): UUID provided by client

**Returns:**

- object: Cached result if exists, null otherwise

**Algorithm:**

```
function check_idempotency(merchant_id, idempotency_key):
    // Build cache key
    cache_key = f"idempotency:{merchant_id}:{idempotency_key}"
    
    // Check Redis
    result = redis.get(cache_key)
    
    if result is not null:
        log_info(f"Idempotency cache hit: {cache_key}")
        return deserialize(result)
    
    return null
```

**Time Complexity:** O(1)

---

### cache_idempotency()

**Purpose:** Store request result for future duplicate checks

**Parameters:**

- merchant_id (string): Merchant identifier
- idempotency_key (string): UUID
- result (object): Response to cache
- ttl_seconds (integer): Time to live (default 86400 = 24 hours)

**Returns:**

- boolean: true if cached successfully

**Algorithm:**

```
function cache_idempotency(merchant_id, idempotency_key, result, ttl_seconds):
    cache_key = f"idempotency:{merchant_id}:{idempotency_key}"
    
    // Serialize result
    serialized = json.dumps(result)
    
    // Store in Redis with TTL
    success = redis.setex(cache_key, ttl_seconds, serialized)
    
    if not success:
        log_error(f"Failed to cache idempotency: {cache_key}")
    
    return success
```

**Time Complexity:** O(1)

**Example Usage:**

```
cache_idempotency(
    merchant_id: "merch_123",
    idempotency_key: "idem_xyz",
    result: {payment_id: "pay_123", status: "authorized"},
    ttl_seconds: 86400
)
```

---

## 3. Tokenization

### create_token()

**Purpose:** Convert sensitive card number to non-sensitive token

**Parameters:**

- card_number (string): Credit card number (16 digits)
- exp_month (integer): Expiration month (1-12)
- exp_year (integer): Expiration year (YYYY)
- cvc (string): Card verification code (3-4 digits)

**Returns:**

- Token: {token_id, last4, brand, fingerprint}

**Algorithm:**

```
function create_token(card_number, exp_month, exp_year, cvc):
    // Step 1: Validate card number (Luhn algorithm)
    if not validate_luhn(card_number):
        throw InvalidCardError("Card number failed Luhn check")
    
    // Step 2: Determine brand
    brand = detect_card_brand(card_number)
    
    // Step 3: Create fingerprint (for deduplication)
    fingerprint = sha256(card_number)
    
    // Step 4: Check if card already tokenized
    existing_token = db.select("tokens")
        .where(fingerprint: fingerprint)
        .where(exp_month: exp_month, exp_year: exp_year)
        .first()
    
    if existing_token is not null:
        return existing_token  // Return existing token
    
    // Step 5: Generate unique token ID
    token_id = generate_id("tok")
    
    // Step 6: Encrypt card number
    encryption_key = kms.get_data_key("card-encryption")
    encrypted_card = aes_256_encrypt(card_number, encryption_key)
    
    // Step 7: Store in database
    token = db.insert({
        table: "tokens",
        values: {
            token_id: token_id,
            encrypted_card: encrypted_card,
            fingerprint: fingerprint,
            last4: card_number[-4:],
            brand: brand,
            exp_month: exp_month,
            exp_year: exp_year,
            created_at: now()
        }
    })
    
    // Step 8: Cache token metadata
    redis.hset(f"token:{token_id}", {
        last4: token.last4,
        brand: brand,
        exp_month: exp_month,
        exp_year: exp_year
    })
    redis.expire(f"token:{token_id}", 3600)  // 1 hour TTL
    
    return {
        token_id: token_id,
        last4: token.last4,
        brand: brand,
        fingerprint: fingerprint
    }
```

**Time Complexity:** O(1)

**Example Usage:**

```
token = create_token(
    card_number: "4111111111111111",
    exp_month: 12,
    exp_year: 2025,
    cvc: "123"
)
// Returns: {token_id: "tok_abc123", last4: "1111", brand: "Visa"}
```

---

### validate_luhn()

**Purpose:** Validate credit card number using Luhn algorithm

**Parameters:**

- card_number (string): Credit card number

**Returns:**

- boolean: true if valid, false otherwise

**Algorithm:**

```
function validate_luhn(card_number):
    // Remove spaces and dashes
    digits = card_number.replace(/[^0-9]/g, '')
    
    if len(digits) < 13 or len(digits) > 19:
        return false
    
    // Luhn algorithm
    sum = 0
    parity = len(digits) % 2
    
    for i in range(len(digits)):
        digit = int(digits[i])
        
        if i % 2 == parity:
            digit = digit * 2
            if digit > 9:
                digit = digit - 9
        
        sum = sum + digit
    
    return (sum % 10) == 0
```

**Time Complexity:** O(n) where n = number of digits

**Example Usage:**

```
validate_luhn("4111111111111111")  // true (valid Visa test card)
validate_luhn("4111111111111112")  // false (invalid checksum)
```

---

### detokenize_card()

**Purpose:** Retrieve encrypted card data from token

**Parameters:**

- token_id (string): Token identifier

**Returns:**

- Card: {encrypted_number, exp_month, exp_year, brand, last4}

**Algorithm:**

```
function detokenize_card(token_id):
    // Check cache first
    cached_card = redis.hgetall(f"token:{token_id}")
    if cached_card is not null:
        // Load encrypted card from DB (cache doesn't store encrypted number)
        card = db.select("tokens").where(token_id: token_id).first()
        if card:
            return card
    
    // Load from database
    card = db.select("tokens").where(token_id: token_id).first()
    
    if card is null:
        throw TokenNotFoundError()
    
    // Cache metadata (not encrypted card!)
    redis.hset(f"token:{token_id}", {
        last4: card.last4,
        brand: card.brand,
        exp_month: card.exp_month,
        exp_year: card.exp_year
    })
    redis.expire(f"token:{token_id}", 3600)
    
    return card
```

**Time Complexity:** O(1)

---

### decrypt_card()

**Purpose:** Decrypt card number for bank API call

**Parameters:**

- encrypted_card (string): AES-256 encrypted card

**Returns:**

- string: Decrypted card number

**Algorithm:**

```
function decrypt_card(encrypted_card):
    // Get decryption key from HSM
    decryption_key = kms.get_data_key("card-encryption")
    
    // Decrypt
    card_number = aes_256_decrypt(encrypted_card, decryption_key)
    
    // Audit log (do NOT log card number!)
    audit_log({
        action: "card_decrypted",
        timestamp: now(),
        // NO card number in logs!
    })
    
    return card_number
```

**Time Complexity:** O(1)

---

## 4. Fraud Detection

### calculate_fraud_score()

**Purpose:** Calculate fraud risk score using ML model

**Parameters:**

- merchant_id (string): Merchant identifier
- card (object): Card details
- amount_cents (integer): Transaction amount

**Returns:**

- float: Risk score between 0.0 (safe) and 1.0 (fraud)

**Algorithm:**

```
function calculate_fraud_score(merchant_id, card, amount_cents):
    // Step 1: Extract features
    features = extract_features(merchant_id, card, amount_cents)
    
    // Step 2: Load ML model
    model = load_fraud_model("xgboost_v3")
    
    // Step 3: Run inference
    risk_score = model.predict(features)
    
    // Step 4: Log prediction
    log_fraud_prediction(merchant_id, card.fingerprint, risk_score)
    
    return risk_score
```

**Time Complexity:** O(1) for inference (20ms with GPU)

---

### extract_features()

**Purpose:** Extract features for fraud detection model

**Parameters:**

- merchant_id (string): Merchant identifier
- card (object): Card details
- amount_cents (integer): Transaction amount

**Returns:**

- array: 30-element feature vector

**Algorithm:**

```
function extract_features(merchant_id, card, amount_cents):
    features = []
    
    // Transaction features
    features.append(amount_cents)
    features.append(get_hour_of_day())
    features.append(get_day_of_week())
    features.append(is_weekend())
    
    // Card features
    features.append(encode_card_brand(card.brand))
    features.append(encode_card_country(card.country))
    
    // Historical features (from Redis)
    card_key = f"card_history:{card.fingerprint}"
    features.append(redis.get(f"{card_key}:decline_count") or 0)
    features.append(redis.get(f"{card_key}:total_transactions") or 0)
    features.append(redis.get(f"{card_key}:avg_amount") or 0)
    
    // Velocity features (transactions per hour)
    features.append(count_recent_transactions(card.fingerprint, hours: 1))
    features.append(count_recent_transactions(card.fingerprint, hours: 24))
    
    // Merchant features
    merchant_key = f"merchant_history:{merchant_id}"
    features.append(redis.get(f"{merchant_key}:fraud_rate") or 0)
    features.append(redis.get(f"{merchant_key}:avg_ticket") or 0)
    
    // Behavioral features
    features.append(is_first_transaction(card.fingerprint, merchant_id))
    features.append(amount_deviation_from_avg(card.fingerprint, amount_cents))
    
    // Geographic features
    features.append(encode_country(card.country))
    features.append(is_high_risk_country(card.country))
    
    // ... (30 total features)
    
    return features
```

**Time Complexity:** O(1) with Redis lookups

---

### update_fraud_features()

**Purpose:** Update historical features after transaction

**Parameters:**

- card_fingerprint (string): Card fingerprint
- merchant_id (string): Merchant identifier
- amount_cents (integer): Transaction amount
- result (string): "approved", "declined", "fraud"

**Returns:**

- void

**Algorithm:**

```
function update_fraud_features(card_fingerprint, merchant_id, amount_cents, result):
    card_key = f"card_history:{card_fingerprint}"
    
    // Update transaction count
    redis.incr(f"{card_key}:total_transactions")
    
    // Update decline count
    if result == "declined" or result == "fraud":
        redis.incr(f"{card_key}:decline_count")
    
    // Update average amount (running average)
    current_avg = float(redis.get(f"{card_key}:avg_amount") or 0)
    total_txns = int(redis.get(f"{card_key}:total_transactions"))
    new_avg = ((current_avg * (total_txns - 1)) + amount_cents) / total_txns
    redis.set(f"{card_key}:avg_amount", new_avg)
    
    // Update velocity (sliding window)
    redis.lpush(f"{card_key}:recent_txns", json.dumps({
        timestamp: now(),
        amount: amount_cents
    }))
    redis.ltrim(f"{card_key}:recent_txns", 0, 99)  // Keep last 100
    
    // Set expiry (retain for 90 days)
    redis.expire(f"{card_key}:total_transactions", 7776000)
```

**Time Complexity:** O(1)

---

## 5. Refund Processing

### create_refund()

**Purpose:** Refund captured payment (full or partial)

**Parameters:**

- payment_id (string): Payment identifier
- amount_cents (integer): Refund amount (null for full refund)
- reason (string): Refund reason
- idempotency_key (string): UUID

**Returns:**

- Refund: {refund_id, status, amount_cents}

**Algorithm:**

```
function create_refund(payment_id, amount_cents, reason, idempotency_key):
    // Check idempotency
    cache_key = f"refund:{payment_id}:{idempotency_key}"
    cached_result = redis.get(cache_key)
    if cached_result:
        return cached_result
    
    // Load payment
    payment = db.select("payments").where(payment_id: payment_id).first()
    
    if payment.status != "captured":
        throw InvalidStateError("Can only refund captured payments")
    
    // Calculate refund amount
    if amount_cents is null:
        refund_amount = payment.captured_amount_cents
    else:
        refund_amount = amount_cents
    
    // Validate refund amount
    refunded_so_far = db.sum("refunds", "amount_cents")
        .where(payment_id: payment_id, status: "completed")
    
    if (refunded_so_far + refund_amount) > payment.captured_amount_cents:
        throw InvalidAmountError("Cannot refund more than captured")
    
    // Create refund record
    refund_id = generate_id("ref")
    refund = db.insert({
        table: "refunds",
        values: {
            refund_id: refund_id,
            payment_id: payment_id,
            amount_cents: refund_amount,
            reason: reason,
            status: "pending",
            created_at: now()
        }
    })
    
    // Call bank API
    try:
        bank_response = bank_api.refund(
            settlement_id: payment.settlement_id,
            amount: refund_amount
        )
    catch BankAPIError as e:
        db.update("refunds").where(refund_id: refund_id).set({
            status: "failed",
            error: e.message
        })
        throw RefundFailedError(e.message)
    
    // Update refund status
    db.update("refunds").where(refund_id: refund_id).set({
        status: "completed",
        bank_refund_id: bank_response.refund_id,
        completed_at: now()
    })
    
    refund.status = "completed"
    
    // Cache result
    redis.setex(cache_key, 86400, refund)
    
    // Trigger event
    publish_event("payment.refunded", {
        payment_id: payment_id,
        refund_id: refund_id,
        amount_cents: refund_amount
    })
    
    return refund
```

**Time Complexity:** O(1)

---

## 6. Webhook Delivery

### deliver_webhook()

**Purpose:** Deliver event to merchant endpoint with retry logic

**Parameters:**

- merchant_id (string): Merchant identifier
- event_type (string): Event type (payment.captured, payment.refunded)
- payload (object): Event data

**Returns:**

- boolean: true if delivered successfully

**Algorithm:**

```
function deliver_webhook(merchant_id, event_type, payload):
    // Load merchant webhook config
    merchant = db.select("merchants").where(merchant_id: merchant_id).first()
    
    if merchant.webhook_url is null:
        log_info(f"Merchant {merchant_id} has no webhook URL")
        return false
    
    // Generate signature
    signature = generate_webhook_signature(merchant.webhook_secret, payload)
    
    // Attempt delivery
    max_attempts = 10
    attempt = 0
    
    while attempt < max_attempts:
        attempt = attempt + 1
        
        try:
            response = http.post(
                url: merchant.webhook_url,
                headers: {
                    "Content-Type": "application/json",
                    "X-Signature": f"sha256={signature}",
                    "X-Event-Type": event_type,
                    "X-Attempt": attempt
                },
                body: json.dumps(payload),
                timeout: 10  // 10 second timeout
            )
            
            if response.status_code == 200:
                log_info(f"Webhook delivered: {merchant_id} {event_type}")
                return true
            else:
                log_warning(f"Webhook failed (status {response.status_code})")
        
        catch HTTPError as e:
            log_error(f"Webhook delivery error: {e}")
        
        // Calculate backoff delay
        if attempt < max_attempts:
            delay = calculate_backoff_delay(attempt)
            log_info(f"Retrying webhook in {delay} seconds")
            sleep(delay)
    
    // All attempts failed
    log_error(f"Webhook delivery failed after {max_attempts} attempts")
    
    // Move to dead letter queue
    publish_to_dlq("webhook_failed", {
        merchant_id: merchant_id,
        event_type: event_type,
        payload: payload
    })
    
    // Alert merchant
    send_email(
        to: merchant.email,
        subject: "Webhook delivery failed",
        body: f"Unable to deliver {event_type} event after {max_attempts} attempts"
    )
    
    return false
```

**Time Complexity:** O(n) where n = retry attempts (max 10)

---

### generate_webhook_signature()

**Purpose:** Generate HMAC signature for webhook verification

**Parameters:**

- webhook_secret (string): Merchant's webhook secret
- payload (object): Event data

**Returns:**

- string: HMAC-SHA256 hex signature

**Algorithm:**

```
function generate_webhook_signature(webhook_secret, payload):
    // Serialize payload
    payload_string = json.dumps(payload, sort_keys: true)
    
    // Generate HMAC-SHA256
    signature = hmac_sha256(webhook_secret, payload_string)
    
    return signature.hex()
```

**Time Complexity:** O(n) where n = payload size

**Example Usage:**

```
signature = generate_webhook_signature(
    webhook_secret: "whsec_abc123",
    payload: {event: "payment.captured", payment_id: "pay_123"}
)
// Returns: "a3f5d8e9c2b1..."
```

---

### calculate_backoff_delay()

**Purpose:** Calculate exponential backoff delay for retries

**Parameters:**

- attempt (integer): Attempt number (1-10)

**Returns:**

- integer: Delay in seconds

**Algorithm:**

```
function calculate_backoff_delay(attempt):
    // Retry schedule:
    // 1: 1 minute
    // 2: 5 minutes
    // 3: 30 minutes
    // 4: 1 hour
    // 5-10: 6, 12, 24, 48, 72, 96 hours
    
    delays = [60, 300, 1800, 3600, 21600, 43200, 86400, 172800, 259200, 345600]
    
    if attempt <= len(delays):
        return delays[attempt - 1]
    else:
        return delays[-1]  // Max delay
```

**Time Complexity:** O(1)

---

## 7. Circuit Breaker

### CircuitBreaker Class

**Purpose:** Implement circuit breaker pattern for bank API calls

**Algorithm:**

```
class CircuitBreaker:
    function __init__(service_name, failure_threshold, cooldown_seconds):
        this.service_name = service_name
        this.failure_threshold = failure_threshold  // 0.5 = 50%
        this.cooldown_seconds = cooldown_seconds  // 60 seconds
        this.state = "CLOSED"  // CLOSED, OPEN, HALF_OPEN
        this.failure_count = 0
        this.success_count = 0
        this.total_count = 0
        this.last_failure_time = null
        this.window_seconds = 10  // Rolling window
    
    function call(func, *args):
        // Check if circuit is open
        if this.state == "OPEN":
            if now() - this.last_failure_time > this.cooldown_seconds:
                log_info(f"Circuit entering HALF_OPEN: {this.service_name}")
                this.state = "HALF_OPEN"
            else:
                throw CircuitBreakerOpenError(f"{this.service_name} circuit is open")
        
        // Execute function
        try:
            result = func(*args)
            this.record_success()
            return result
        
        catch Exception as e:
            this.record_failure()
            throw e
    
    function record_success():
        this.success_count = this.success_count + 1
        this.total_count = this.total_count + 1
        
        // Reset failure count in HALF_OPEN state
        if this.state == "HALF_OPEN":
            if this.success_count >= 5:  // 5 consecutive successes
                log_info(f"Circuit closing: {this.service_name}")
                this.state = "CLOSED"
                this.reset_counts()
    
    function record_failure():
        this.failure_count = this.failure_count + 1
        this.total_count = this.total_count + 1
        this.last_failure_time = now()
        
        // Open circuit if failure rate exceeds threshold
        failure_rate = this.failure_count / this.total_count
        
        if failure_rate > this.failure_threshold and this.total_count >= 10:
            log_error(f"Circuit opening: {this.service_name} (failure rate: {failure_rate})")
            this.state = "OPEN"
        
        // In HALF_OPEN, single failure reopens circuit
        if this.state == "HALF_OPEN":
            log_error(f"Circuit reopening: {this.service_name}")
            this.state = "OPEN"
    
    function reset_counts():
        this.failure_count = 0
        this.success_count = 0
        this.total_count = 0
    
    function get_state():
        return this.state
```

**Example Usage:**

```
// Initialize circuit breaker
bank_circuit = CircuitBreaker(
    service_name: "bank_api",
    failure_threshold: 0.5,  // 50% failure rate
    cooldown_seconds: 60
)

// Use circuit breaker
try:
    result = bank_circuit.call(bank_api.authorize, card, amount)
catch CircuitBreakerOpenError:
    return {error: "Service temporarily unavailable"}
```

---

## 8. Subscription Billing

### process_subscription_billing()

**Purpose:** Daily cron job to bill subscriptions due today

**Parameters:**

- None (run daily at 00:00 UTC)

**Returns:**

- Summary: {billed_count, failed_count, revenue_cents}

**Algorithm:**

```
function process_subscription_billing():
    log_info("Starting subscription billing")
    
    // Query subscriptions due today
    today = date_today()
    subscriptions = db.select("subscriptions")
        .where(current_period_end: today)
        .where(status: "active")
        .all()
    
    log_info(f"Found {len(subscriptions)} subscriptions due")
    
    billed_count = 0
    failed_count = 0
    revenue_cents = 0
    
    // Process each subscription
    for subscription in subscriptions:
        try:
            result = bill_subscription(subscription)
            if result.success:
                billed_count = billed_count + 1
                revenue_cents = revenue_cents + subscription.amount_cents
            else:
                failed_count = failed_count + 1
        
        catch Exception as e:
            log_error(f"Billing failed for {subscription.subscription_id}: {e}")
            failed_count = failed_count + 1
    
    log_info(f"Billing complete: {billed_count} succeeded, {failed_count} failed")
    
    return {
        billed_count: billed_count,
        failed_count: failed_count,
        revenue_cents: revenue_cents
    }
```

**Time Complexity:** O(n) where n = subscriptions due today

---

### bill_subscription()

**Purpose:** Bill a single subscription

**Parameters:**

- subscription (object): Subscription object

**Returns:**

- object: {success: boolean, payment_id: string}

**Algorithm:**

```
function bill_subscription(subscription):
    // Attempt to charge payment method
    try:
        payment = authorize_payment(
            merchant_id: subscription.merchant_id,
            amount_cents: subscription.amount_cents,
            currency: subscription.currency,
            token_id: subscription.payment_token_id,
            idempotency_key: f"sub_{subscription.subscription_id}_{subscription.current_period_end}"
        )
        
        // Auto-capture subscription payments
        capture_payment(payment.payment_id)
        
        // Extend subscription period
        db.update("subscriptions")
            .where(subscription_id: subscription.subscription_id)
            .set({
                current_period_start: subscription.current_period_end,
                current_period_end: subscription.current_period_end + days(30),
                last_payment_id: payment.payment_id,
                retry_count: 0
            })
        
        // Send success email
        send_email(
            to: subscription.customer_email,
            subject: "Payment successful",
            body: f"Your subscription has been renewed"
        )
        
        return {success: true, payment_id: payment.payment_id}
    
    catch PaymentFailedError as e:
        // Handle failed payment
        retry_count = subscription.retry_count + 1
        
        if retry_count < 3:
            // Schedule retry
            next_retry = calculate_retry_date(retry_count)
            db.update("subscriptions")
                .where(subscription_id: subscription.subscription_id)
                .set({
                    retry_count: retry_count,
                    next_retry_date: next_retry
                })
            
            send_email(
                to: subscription.customer_email,
                subject: "Payment failed",
                body: f"Your payment failed. We'll retry on {next_retry}"
            )
        else:
            // Mark past_due after 3 failures
            db.update("subscriptions")
                .where(subscription_id: subscription.subscription_id)
                .set({
                    status: "past_due",
                    past_due_since: now()
                })
            
            send_email(
                to: subscription.customer_email,
                subject: "Subscription past due",
                body: "Please update your payment method"
            )
        
        return {success: false, error: e.message}
```

**Time Complexity:** O(1)

---

## 9. Sharding

### get_shard_id()

**Purpose:** Calculate shard ID for merchant

**Parameters:**

- merchant_id (string): Merchant identifier

**Returns:**

- integer: Shard ID (0-63)

**Algorithm:**

```
function get_shard_id(merchant_id):
    // Hash merchant ID
    hash_value = hash_fnv1a(merchant_id)
    
    // Modulo to get shard (64 shards)
    shard_id = hash_value % 64
    
    return shard_id
```

**Time Complexity:** O(1)

**Example Usage:**

```
shard_id = get_shard_id("merch_12345")  // Returns: 34
```

---

### get_shard_connection()

**Purpose:** Get database connection for specific shard

**Parameters:**

- shard_id (integer): Shard ID (0-63)

**Returns:**

- Connection: Database connection object

**Algorithm:**

```
function get_shard_connection(shard_id):
    // Load shard configuration
    shard_config = config.shards[shard_id]
    
    // Connection pool (per shard)
    pool_key = f"shard_pool_{shard_id}"
    
    if pool_key not in connection_pools:
        connection_pools[pool_key] = create_connection_pool(
            host: shard_config.host,
            port: shard_config.port,
            database: shard_config.database,
            min_connections: 10,
            max_connections: 50
        )
    
    // Get connection from pool
    connection = connection_pools[pool_key].get_connection()
    
    return connection
```

**Time Complexity:** O(1)

---

### query_shard()

**Purpose:** Execute query on correct shard

**Parameters:**

- merchant_id (string): Merchant identifier
- query (string): SQL query
- params (object): Query parameters

**Returns:**

- Result: Query result

**Algorithm:**

```
function query_shard(merchant_id, query, params):
    // Get shard ID
    shard_id = get_shard_id(merchant_id)
    
    // Get connection
    conn = get_shard_connection(shard_id)
    
    // Execute query
    try:
        result = conn.execute(query, params)
        return result
    finally:
        conn.release()  // Return to pool
```

**Time Complexity:** O(1) + query complexity

---

## 10. 3D Secure

### initiate_3ds_challenge()

**Purpose:** Initiate 3D Secure authentication challenge

**Parameters:**

- payment_id (string): Payment identifier
- return_url (string): URL to redirect after challenge

**Returns:**

- object: {challenge_url, challenge_id}

**Algorithm:**

```
function initiate_3ds_challenge(payment_id, return_url):
    // Load payment
    payment = db.select("payments").where(payment_id: payment_id).first()
    
    // Get card details
    card = detokenize_card(payment.token_id)
    
    // Call 3DS provider
    try:
        challenge = threeds_provider.create_challenge(
            card_number: decrypt_card(card.encrypted_card),
            amount_cents: payment.amount_cents,
            currency: payment.currency,
            return_url: return_url
        )
    catch ThreeDSError as e:
        throw ThreeDSInitiationError(e.message)
    
    // Store challenge ID with payment
    db.update("payments").where(payment_id: payment_id).set({
        threeds_challenge_id: challenge.challenge_id,
        status: "requires_action"
    })
    
    return {
        challenge_url: challenge.challenge_url,
        challenge_id: challenge.challenge_id
    }
```

**Time Complexity:** O(1)

---

### complete_3ds_challenge()

**Purpose:** Complete payment after successful 3DS authentication

**Parameters:**

- payment_id (string): Payment identifier
- challenge_result (string): Result token from 3DS provider

**Returns:**

- Payment: Updated payment object

**Algorithm:**

```
function complete_3ds_challenge(payment_id, challenge_result):
    // Load payment
    payment = db.select("payments").where(payment_id: payment_id).first()
    
    if payment.status != "requires_action":
        throw InvalidStateError("Payment not awaiting 3DS completion")
    
    // Verify 3DS result
    try:
        verification = threeds_provider.verify_challenge(
            challenge_id: payment.threeds_challenge_id,
            result_token: challenge_result
        )
    catch ThreeDSError as e:
        db.update("payments").where(payment_id: payment_id).set({
            status: "failed",
            failure_reason: "3ds_failed"
        })
        throw ThreeDSVerificationError(e.message)
    
    if not verification.authenticated:
        db.update("payments").where(payment_id: payment_id).set({
            status: "failed",
            failure_reason: "3ds_authentication_failed"
        })
        return payment
    
    // Get card and retry authorization with 3DS proof
    card = detokenize_card(payment.token_id)
    
    try:
        bank_response = bank_api.authorize_with_3ds(
            card_number: decrypt_card(card.encrypted_card),
            amount: payment.amount_cents,
            currency: payment.currency,
            threeds_proof: verification.proof
        )
    catch BankAPIError as e:
        db.update("payments").where(payment_id: payment_id).set({
            status: "failed",
            failure_reason: e.message
        })
        throw PaymentFailedError(e.message)
    
    // Update payment with authorization
    db.update("payments").where(payment_id: payment_id).set({
        status: "authorized",
        auth_code: bank_response.auth_code,
        liability_shifted: true  // 3DS shifts liability to bank
    })
    
    payment.status = "authorized"
    payment.auth_code = bank_response.auth_code
    
    return payment
```

**Time Complexity:** O(1)
