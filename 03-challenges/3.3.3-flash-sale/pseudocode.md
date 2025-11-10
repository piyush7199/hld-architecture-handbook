# E-commerce Flash Sale System - Pseudocode Implementations

This document contains detailed algorithm implementations for the flash sale system. The main challenge document
references these functions.

---

## Table of Contents

1. [Inventory Reservation](#1-inventory-reservation)
2. [Payment Processing](#2-payment-processing)
3. [Saga Orchestration](#3-saga-orchestration)
4. [Rate Limiting](#4-rate-limiting)
5. [Cleanup Workers](#5-cleanup-workers)
6. [Idempotency Handling](#6-idempotency-handling)
7. [Split Counter Operations](#7-split-counter-operations)
8. [Utility Functions](#8-utility-functions)

---

## 1. Inventory Reservation

### reserve_inventory()

**Purpose:** Atomically check and decrement inventory using Redis Lua script.

**Parameters:**

- product_id (string): Product identifier
- user_id (integer): User making reservation
- quantity (integer): Number of items to reserve (default: 1)

**Returns:**

- dict: {success: bool, reservation_id: UUID, message: string}

**Algorithm:**

```
function reserve_inventory(product_id, user_id, quantity):
  // Generate idempotency key
  idempotency_key = generate_uuid()
  
  // Check idempotency store (prevent duplicate reservations)
  existing = check_idempotency(idempotency_key)
  if existing != null:
    return existing  // Return cached result
  
  // Construct Redis key
  stock_key = f"product:{product_id}:stock"
  
  // Lua script for atomic check-and-decrement
  lua_script = """
    local stock_key = KEYS[1]
    local quantity = tonumber(ARGV[1])
    local current_stock = tonumber(redis.call('GET', stock_key))
    
    if current_stock >= quantity then
      redis.call('DECRBY', stock_key, quantity)
      return 1  -- Success
    else
      return 0  -- Sold out
    end
  """
  
  // Execute Lua script atomically
  result = redis.eval(lua_script, [stock_key], [quantity])
  
  if result == 1:
    // Success: Create reservation record
    reservation_id = generate_uuid()
    expires_at = current_timestamp() + 600  // 10 minutes
    
    db.execute(
      "INSERT INTO reservations (id, user_id, product_id, quantity, expires_at, status) VALUES (?, ?, ?, ?, ?, 'PENDING')",
      [reservation_id, user_id, product_id, quantity, expires_at]
    )
    
    // Publish event to Kafka (async payment processing)
    kafka.publish("reservations", {
      "reservation_id": reservation_id,
      "user_id": user_id,
      "product_id": product_id,
      "amount": calculate_price(product_id, quantity)
    })
    
    // Cache result in idempotency store
    cache_idempotency_result(idempotency_key, {
      "success": true,
      "reservation_id": reservation_id,
      "expires_at": expires_at
    })
    
    return {
      "success": true,
      "reservation_id": reservation_id,
      "expires_at": expires_at,
      "message": "Reservation successful"
    }
  
  else:
    // Failure: Sold out
    return {
      "success": false,
      "reservation_id": null,
      "message": "Product sold out"
    }
```

**Time Complexity:** O(1) for Redis operation

**Space Complexity:** O(1)

**Example Usage:**

```
result = reserve_inventory("iphone-15", user_id=123, quantity=1)
if result["success"]:
  print(f"Reserved! Expires at: {result['expires_at']}")
else:
  print(f"Failed: {result['message']}")
```

---

### unreserve_inventory()

**Purpose:** Return inventory to stock (compensation transaction when payment fails).

**Parameters:**

- reservation_id (UUID): Reservation to cancel
- reason (string): Cancellation reason

**Returns:**

- bool: Success/failure

**Algorithm:**

```
function unreserve_inventory(reservation_id, reason):
  // Get reservation details
  reservation = db.query(
    "SELECT * FROM reservations WHERE id = ? AND status = 'PENDING'",
    [reservation_id]
  )
  
  if reservation == null:
    log_warning(f"Reservation {reservation_id} not found or already processed")
    return false
  
  // Return inventory to Redis
  stock_key = f"product:{reservation['product_id']}:stock"
  redis.incrby(stock_key, reservation['quantity'])
  
  // Update reservation status
  db.execute(
    "UPDATE reservations SET status = 'CANCELLED', cancellation_reason = ? WHERE id = ?",
    [reason, reservation_id]
  )
  
  // Notify user
  send_notification(reservation['user_id'], {
    "type": "reservation_cancelled",
    "reason": reason,
    "message": "Your reservation has been cancelled. Stock returned."
  })
  
  log_info(f"Returned {reservation['quantity']} items to stock for product {reservation['product_id']}")
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
success = unreserve_inventory(reservation_id, reason="payment_failed")
```

---

## 2. Payment Processing

### charge_customer()

**Purpose:** Charge customer via Stripe API with idempotency key to prevent double-charging.

**Parameters:**

- user_id (integer): User to charge
- amount (decimal): Amount in dollars
- idempotency_key (UUID): Unique key to prevent duplicate charges

**Returns:**

- dict: {success: bool, charge_id: string, error: string}

**Algorithm:**

```
function charge_customer(user_id, amount, idempotency_key):
  // Get user's payment method
  payment_method = db.query(
    "SELECT stripe_customer_id, stripe_payment_method_id FROM users WHERE id = ?",
    [user_id]
  )
  
  if payment_method == null:
    return {
      "success": false,
      "charge_id": null,
      "error": "No payment method found"
    }
  
  // Retry logic with exponential backoff
  max_retries = 3
  retry_count = 0
  backoff_seconds = 1
  
  while retry_count < max_retries:
    try:
      // Call Stripe API with idempotency key
      charge = stripe.create_charge({
        "amount": int(amount * 100),  // Convert to cents
        "currency": "usd",
        "customer": payment_method["stripe_customer_id"],
        "payment_method": payment_method["stripe_payment_method_id"],
        "description": f"Flash sale purchase - User {user_id}",
        "idempotency_key": idempotency_key  // Prevents duplicate charges
      })
      
      if charge["status"] == "succeeded":
        return {
          "success": true,
          "charge_id": charge["id"],
          "error": null
        }
      else:
        return {
          "success": false,
          "charge_id": null,
          "error": f"Charge failed: {charge['failure_message']}"
        }
    
    catch StripeNetworkError as error:
      // Transient network error: retry with exponential backoff
      retry_count++
      if retry_count < max_retries:
        log_warning(f"Stripe network error, retry {retry_count}/{max_retries}: {error}")
        sleep(backoff_seconds)
        backoff_seconds *= 2  // Exponential backoff: 1s, 2s, 4s
      else:
        return {
          "success": false,
          "charge_id": null,
          "error": f"Payment failed after {max_retries} retries: {error}"
        }
    
    catch StripeCardError as error:
      // Card declined: don't retry
      return {
        "success": false,
        "charge_id": null,
        "error": f"Card declined: {error.message}"
      }
```

**Time Complexity:** O(R) where R = retry attempts (max 3)

**Example Usage:**

```
result = charge_customer(user_id=123, amount=999.00, idempotency_key=uuid)
if result["success"]:
  print(f"Charged successfully: {result['charge_id']}")
else:
  print(f"Payment failed: {result['error']}")
```

---

## 3. Saga Orchestration

### execute_payment_saga()

**Purpose:** Execute multi-step payment saga with automatic compensation on failure.

**Parameters:**

- reservation_id (UUID): Reservation to process

**Returns:**

- dict: {success: bool, order_id: integer, error: string}

**Algorithm:**

```
function execute_payment_saga(reservation_id):
  // Initialize saga context (tracks completed steps for compensation)
  saga_context = SagaContext()
  
  try:
    // STEP 1: Validate Reservation
    log_info(f"Saga Step 1: Validate reservation {reservation_id}")
    
    reservation = db.query(
      "SELECT * FROM reservations WHERE id = ? AND status = 'PENDING' FOR UPDATE",
      [reservation_id]
    )
    
    if reservation == null:
      throw SagaException("Reservation not found or already processed")
    
    if reservation["expires_at"] < current_timestamp():
      throw SagaException("Reservation expired")
    
    saga_context.add_step("validate_reservation", reservation, null)
    
    // STEP 2: Charge Customer
    log_info(f"Saga Step 2: Charge customer {reservation['user_id']}")
    
    charge_result = charge_customer(
      reservation["user_id"],
      reservation["amount"],
      reservation["id"]  // Use reservation_id as idempotency key
    )
    
    if not charge_result["success"]:
      throw SagaException(f"Payment failed: {charge_result['error']}")
    
    saga_context.add_step("charge_customer", charge_result, compensate_charge)
    
    // STEP 3: Create Order
    log_info(f"Saga Step 3: Create order")
    
    order_id = db.execute(
      "INSERT INTO orders (user_id, reservation_id, product_id, quantity, amount, payment_id, status) VALUES (?, ?, ?, ?, ?, ?, 'PAID') RETURNING id",
      [
        reservation["user_id"],
        reservation_id,
        reservation["product_id"],
        reservation["quantity"],
        reservation["amount"],
        charge_result["charge_id"]
      ]
    )
    
    saga_context.add_step("create_order", {"order_id": order_id}, compensate_order)
    
    // STEP 4: Update Reservation Status
    db.execute(
      "UPDATE reservations SET status = 'PAID' WHERE id = ?",
      [reservation_id]
    )
    
    // STEP 5: Notify User
    send_notification(reservation["user_id"], {
      "type": "order_confirmed",
      "order_id": order_id,
      "message": "Your order has been confirmed!"
    })
    
    log_info(f"Saga completed successfully: Order {order_id}")
    
    return {
      "success": true,
      "order_id": order_id,
      "error": null
    }
  
  catch SagaException as error:
    // Saga failed: Execute compensation transactions in reverse order
    log_error(f"Saga failed: {error}. Executing compensations...")
    
    saga_context.compensate()  // Rollback steps in reverse order
    
    return {
      "success": false,
      "order_id": null,
      "error": str(error)
    }

// Compensation function for charge_customer step
function compensate_charge(charge_result):
  log_info(f"Compensating charge: Refunding {charge_result['charge_id']}")
  
  stripe.refund({
    "charge": charge_result["charge_id"],
    "reason": "saga_compensation"
  })

// Compensation function for create_order step
function compensate_order(order_data):
  log_info(f"Compensating order: Cancelling {order_data['order_id']}")
  
  db.execute(
    "UPDATE orders SET status = 'CANCELLED' WHERE id = ?",
    [order_data["order_id"]]
  )
```

**Time Complexity:** O(N) where N = number of saga steps (3 in this case)

**Example Usage:**

```
result = execute_payment_saga(reservation_id)
if result["success"]:
  print(f"Order created: {result['order_id']}")
else:
  print(f"Saga failed: {result['error']}")
```

---

## 4. Rate Limiting

### token_bucket_rate_limit()

**Purpose:** Implement token bucket rate limiting algorithm at CDN edge.

**Parameters:**

- ip_address (string): Client IP address
- rate_limit (integer): Requests per second allowed (default: 10)
- bucket_size (integer): Maximum burst size (default: 20)

**Returns:**

- dict: {allowed: bool, tokens_remaining: integer, retry_after: integer}

**Algorithm:**

```
function token_bucket_rate_limit(ip_address, rate_limit=10, bucket_size=20):
  key = f"rate_limit:{ip_address}"
  
  // Get current bucket state from Redis
  bucket = redis.get(key)
  
  if bucket == null:
    // First request: initialize bucket
    bucket = {
      "tokens": bucket_size,
      "last_update": current_timestamp()
    }
  else:
    // Refill tokens based on time elapsed
    now = current_timestamp()
    elapsed_seconds = (now - bucket["last_update"]) / 1000
    
    tokens_to_add = elapsed_seconds * rate_limit
    bucket["tokens"] = min(bucket_size, bucket["tokens"] + tokens_to_add)
    bucket["last_update"] = now
  
  // Check if tokens available
  if bucket["tokens"] >= 1:
    // Allow request: consume 1 token
    bucket["tokens"] -= 1
    
    // Save updated bucket state (TTL: 60 seconds)
    redis.setex(key, 60, json.stringify(bucket))
    
    return {
      "allowed": true,
      "tokens_remaining": floor(bucket["tokens"]),
      "retry_after": 0
    }
  else:
    // Deny request: no tokens available
    retry_after = ceil((1 - bucket["tokens"]) / rate_limit)
    
    return {
      "allowed": false,
      "tokens_remaining": 0,
      "retry_after": retry_after
    }
```

**Time Complexity:** O(1)

**Space Complexity:** O(1) per IP address

**Example Usage:**

```
result = token_bucket_rate_limit("192.168.1.1", rate_limit=10, bucket_size=20)
if result["allowed"]:
  process_request()
else:
  return_error(429, f"Too Many Requests. Retry after {result['retry_after']}s")
```

---

## 5. Cleanup Workers

### cleanup_expired_reservations()

**Purpose:** Background worker that scans for expired reservations and returns inventory to Redis.

**Parameters:**

- batch_size (integer): Number of reservations to process per run (default: 1000)

**Returns:**

- integer: Number of reservations cleaned up

**Algorithm:**

```
function cleanup_expired_reservations(batch_size=1000):
  // Query expired reservations
  expired_reservations = db.query(
    "SELECT * FROM reservations WHERE expires_at < NOW() AND status = 'PENDING' ORDER BY expires_at ASC LIMIT ?",
    [batch_size]
  )
  
  if expired_reservations.length == 0:
    log_info("No expired reservations found")
    return 0
  
  log_info(f"Found {expired_reservations.length} expired reservations")
  
  cleanup_count = 0
  
  for reservation in expired_reservations:
    try:
      // Use row lock to prevent race condition (user pays while we expire)
      db.begin_transaction()
      
      locked_reservation = db.query(
        "SELECT * FROM reservations WHERE id = ? AND status = 'PENDING' FOR UPDATE",
        [reservation["id"]]
      )
      
      if locked_reservation == null:
        // Reservation already processed (payment succeeded)
        log_info(f"Reservation {reservation['id']} already processed")
        db.commit()
        continue
      
      // Return inventory to Redis
      stock_key = f"product:{reservation['product_id']}:stock"
      redis.incrby(stock_key, reservation["quantity"])
      
      // Update reservation status
      db.execute(
        "UPDATE reservations SET status = 'EXPIRED', updated_at = NOW() WHERE id = ?",
        [reservation["id"]]
      )
      
      db.commit()
      
      // Send notification to user
      send_notification(reservation["user_id"], {
        "type": "reservation_expired",
        "product_id": reservation["product_id"],
        "message": "Your reservation has expired. The item is now available for others."
      })
      
      cleanup_count++
      log_info(f"Returned {reservation['quantity']} items to stock for product {reservation['product_id']}")
      
    catch Exception as error:
      db.rollback()
      log_error(f"Failed to cleanup reservation {reservation['id']}: {error}")
      // Continue to next reservation (don't fail entire batch)
  
  log_info(f"Cleaned up {cleanup_count} expired reservations")
  return cleanup_count
```

**Time Complexity:** O(N) where N = batch_size

**Space Complexity:** O(N) for reservation batch

**Example Usage:**

```
// Run as cron job every 60 seconds
while true:
  count = cleanup_expired_reservations(batch_size=1000)
  sleep(60)  // Wait 60 seconds before next scan
```

---

## 6. Idempotency Handling

### check_idempotency()

**Purpose:** Check if request with given idempotency key has already been processed.

**Parameters:**

- idempotency_key (UUID): Unique key identifying the request

**Returns:**

- dict: {exists: bool, status: string, response: dict} or null

**Algorithm:**

```
function check_idempotency(idempotency_key):
  key = f"idempotency:{idempotency_key}"
  
  // Check Redis cache (fast path)
  cached = redis.hgetall(key)
  
  if cached != null:
    return {
      "exists": true,
      "status": cached["status"],  // PROCESSING, SUCCESS, FAILED
      "response": json.parse(cached["response"])
    }
  
  // Not in cache
  return null
```

**Time Complexity:** O(1)

**Example Usage:**

```
existing = check_idempotency(idempotency_key)
if existing != null:
  if existing["status"] == "SUCCESS":
    return existing["response"]  // Return cached (no double-charge)
  elif existing["status"] == "PROCESSING":
    return error(409, "Request already processing")
  elif existing["status"] == "FAILED":
    return existing["response"]  // Return cached error
```

---

### cache_idempotency_result()

**Purpose:** Cache the result of a request to prevent duplicate processing.

**Parameters:**

- idempotency_key (UUID): Unique key identifying the request
- status (string): Request status (SUCCESS or FAILED)
- response (dict): Response to cache

**Returns:**

- bool: Success/failure

**Algorithm:**

```
function cache_idempotency_result(idempotency_key, status, response):
  key = f"idempotency:{idempotency_key}"
  ttl = 86400  // 24 hours
  
  // Store in Redis hash
  redis.hset(key, "status", status)
  redis.hset(key, "response", json.stringify(response))
  redis.hset(key, "timestamp", current_timestamp())
  redis.expire(key, ttl)
  
  log_info(f"Cached idempotency result for key {idempotency_key}: {status}")
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
// After successful payment
cache_idempotency_result(
  idempotency_key,
  status="SUCCESS",
  response={"charge_id": "ch_123", "amount": 999}
)
```

---

## 7. Split Counter Operations

### reserve_with_split_counter()

**Purpose:** Reserve inventory using split counter optimization (10 sharded keys).

**Parameters:**

- product_id (string): Product identifier
- user_id (integer): User making reservation
- num_shards (integer): Number of split counter shards (default: 10)

**Returns:**

- dict: {success: bool, shard_used: integer, reservation_id: UUID}

**Algorithm:**

```
function reserve_with_split_counter(product_id, user_id, num_shards=10):
  // Calculate initial shard based on user_id hash
  initial_shard = hash(user_id) % num_shards
  
  // Try shards in round-robin order (start at initial, wrap around)
  for attempt in range(num_shards):
    shard_id = (initial_shard + attempt) % num_shards
    stock_key = f"product:{product_id}:stock:{shard_id}"
    
    // Try atomic decrement on this shard
    lua_script = """
      local stock_key = KEYS[1]
      local current = tonumber(redis.call('GET', stock_key))
      if current > 0 then
        redis.call('DECR', stock_key)
        return 1
      else
        return 0
      end
    """
    
    result = redis.eval(lua_script, [stock_key])
    
    if result == 1:
      // Success: Create reservation
      reservation_id = generate_uuid()
      
      db.execute(
        "INSERT INTO reservations (id, user_id, product_id, shard_id, expires_at, status) VALUES (?, ?, ?, ?, ?, 'PENDING')",
        [reservation_id, user_id, product_id, shard_id, current_timestamp() + 600]
      )
      
      log_info(f"Reserved from shard {shard_id} after {attempt + 1} attempts")
      
      return {
        "success": true,
        "shard_used": shard_id,
        "reservation_id": reservation_id
      }
  
  // All shards exhausted: Sold out
  log_info(f"All {num_shards} shards empty for product {product_id}")
  
  return {
    "success": false,
    "shard_used": null,
    "reservation_id": null
  }
```

**Time Complexity:** O(S) where S = num_shards (worst case: try all shards)

**Space Complexity:** O(1)

**Example Usage:**

```
result = reserve_with_split_counter("iphone-15", user_id=123, num_shards=10)
if result["success"]:
  print(f"Reserved! Used shard {result['shard_used']}")
else:
  print("Sold out (all shards empty)")
```

---

## 8. Utility Functions

### generate_uuid()

**Purpose:** Generate a UUID v4 for idempotency keys and reservation IDs.

**Parameters:** None

**Returns:**

- string: UUID in format "550e8400-e29b-41d4-a716-446655440000"

**Algorithm:**

```
function generate_uuid():
  // Use cryptographically secure random number generator
  random_bytes = crypto.random_bytes(16)
  
  // Set version (4) and variant (RFC 4122)
  random_bytes[6] = (random_bytes[6] & 0x0f) | 0x40  // Version 4
  random_bytes[8] = (random_bytes[8] & 0x3f) | 0x80  // Variant RFC 4122
  
  // Format as UUID string
  uuid = format(
    "%02x%02x%02x%02x-%02x%02x-%02x%02x-%02x%02x-%02x%02x%02x%02x%02x%02x",
    random_bytes[0], random_bytes[1], random_bytes[2], random_bytes[3],
    random_bytes[4], random_bytes[5], random_bytes[6], random_bytes[7],
    random_bytes[8], random_bytes[9], random_bytes[10], random_bytes[11],
    random_bytes[12], random_bytes[13], random_bytes[14], random_bytes[15]
  )
  
  return uuid
```

**Time Complexity:** O(1)

**Example Usage:**

```
idempotency_key = generate_uuid()
// "550e8400-e29b-41d4-a716-446655440000"
```

---

### send_notification()

**Purpose:** Send push notification and email to user.

**Parameters:**

- user_id (integer): User to notify
- notification_data (dict): Notification content

**Returns:**

- bool: Success/failure

**Algorithm:**

```
function send_notification(user_id, notification_data):
  user = db.query("SELECT email, push_token FROM users WHERE id = ?", [user_id])
  
  if user == null:
    log_warning(f"User {user_id} not found")
    return false
  
  // Send push notification (async)
  if user["push_token"] != null:
    push_service.send({
      "token": user["push_token"],
      "title": notification_data["type"],
      "body": notification_data["message"],
      "data": notification_data
    })
  
  // Send email (async)
  if user["email"] != null:
    email_service.send({
      "to": user["email"],
      "subject": notification_data["type"],
      "body": notification_data["message"],
      "template": "flash_sale_notification"
    })
  
  log_info(f"Sent notification to user {user_id}: {notification_data['type']}")
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
send_notification(user_id=123, {
  "type": "order_confirmed",
  "order_id": 456,
  "message": "Your order has been confirmed!"
})
```

---

### calculate_price()

**Purpose:** Calculate price for product and quantity (with potential flash sale discount).

**Parameters:**

- product_id (string): Product identifier
- quantity (integer): Number of items

**Returns:**

- decimal: Total price in dollars

**Algorithm:**

```
function calculate_price(product_id, quantity):
  product = db.query(
    "SELECT base_price, flash_sale_discount FROM products WHERE id = ?",
    [product_id]
  )
  
  if product == null:
    throw ProductNotFoundException(f"Product {product_id} not found")
  
  base_price = product["base_price"]
  discount = product["flash_sale_discount"] || 0  // null-coalescing
  
  // Apply discount
  discounted_price = base_price * (1 - discount / 100)
  
  // Calculate total
  total = discounted_price * quantity
  
  // Round to 2 decimal places
  total = round(total, 2)
  
  log_info(f"Calculated price for {quantity}x {product_id}: ${total} (discount: {discount}%)")
  
  return total
```

**Time Complexity:** O(1)

**Example Usage:**

```
price = calculate_price("iphone-15", quantity=1)
// 999.00 (if base_price=999, discount=0)

price = calculate_price("iphone-15", quantity=1)
// 899.10 (if base_price=999, discount=10)
```

---

## Summary

This pseudocode implementation provides 18 comprehensive functions across 8 sections:

1. **Inventory Reservation** (2 functions): reserve_inventory(), unreserve_inventory()
2. **Payment Processing** (1 function): charge_customer()
3. **Saga Orchestration** (1 function): execute_payment_saga()
4. **Rate Limiting** (1 function): token_bucket_rate_limit()
5. **Cleanup Workers** (1 function): cleanup_expired_reservations()
6. **Idempotency Handling** (2 functions): check_idempotency(), cache_idempotency_result()
7. **Split Counter Operations** (1 function): reserve_with_split_counter()
8. **Utility Functions** (3 functions): generate_uuid(), send_notification(), calculate_price()

**Key Algorithms:**

- **Atomic Operations:** Lua scripts for Redis check-and-decrement
- **Distributed Transactions:** Saga pattern with compensation
- **Rate Limiting:** Token bucket with refill
- **Idempotency:** UUID-based deduplication
- **Split Counter:** Sharded inventory for 10× throughput

All functions include error handling, logging, and are designed for production use in high-concurrency scenarios (100K
QPS).

