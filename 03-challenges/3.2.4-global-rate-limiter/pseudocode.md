# Global Rate Limiter - Pseudocode Implementations

This document contains detailed algorithm implementations for the Global Rate Limiter system. The main challenge
document references these functions.

---

## Table of Contents

1. [Token Bucket Algorithm](#1-token-bucket-algorithm)
2. [Sliding Window Counter Algorithm](#2-sliding-window-counter-algorithm)
3. [Fixed Window Counter Algorithm](#3-fixed-window-counter-algorithm)
4. [Atomic Counter Operations](#4-atomic-counter-operations)
5. [Circuit Breaker Implementation](#5-circuit-breaker-implementation)
6. [Hot Key Detection](#6-hot-key-detection)
7. [L1 Cache Implementation](#7-l1-cache-implementation)
8. [Multi-Tier Rate Limiting](#8-multi-tier-rate-limiting)
9. [Configuration Management](#9-configuration-management)

---

## 1. Token Bucket Algorithm

### token_bucket_check()

**Purpose:** Check if request is within rate limit using Token Bucket algorithm, which allows bursts by refilling tokens
at constant rate.

**Parameters:**

- `user_id` (string): Unique identifier for the user
- `capacity` (int): Maximum number of tokens in bucket (burst size)
- `refill_rate` (int): Tokens added per second
- `cost` (int): Number of tokens consumed per request (default: 1)

**Returns:**

- `(allowed: bool, remaining: int, reset_time: timestamp)` - Whether request is allowed, tokens remaining, when bucket
  refills

**Algorithm:**

```
function token_bucket_check(user_id, capacity, refill_rate, cost=1):
    // Generate Redis keys
    token_key = "user:" + user_id + ":tokens"
    refill_key = "user:" + user_id + ":last_refill"
    
    // Load current state from Redis
    current_tokens = redis.GET(token_key)
    last_refill_time = redis.GET(refill_key)
    
    // Initialize if first request
    if current_tokens is NULL:
        current_tokens = capacity
        last_refill_time = now()
    
    // Calculate time elapsed since last refill
    now_time = now()
    elapsed_seconds = now_time - last_refill_time
    
    // Calculate tokens to add based on elapsed time
    tokens_to_add = refill_rate * elapsed_seconds
    
    // Update token count (capped at capacity)
    current_tokens = current_tokens + tokens_to_add
    if current_tokens > capacity:
        current_tokens = capacity
    
    // Check if enough tokens available
    if current_tokens >= cost:
        // Allow request: Consume tokens
        current_tokens = current_tokens - cost
        allowed = true
        
        // Save updated state to Redis with TTL
        redis.SET(token_key, current_tokens, TTL=3600)
        redis.SET(refill_key, now_time, TTL=3600)
    else:
        // Reject request: Not enough tokens
        allowed = false
    
    // Calculate when bucket will refill
    time_until_refill = (cost - current_tokens) / refill_rate
    reset_time = now_time + time_until_refill
    
    return (allowed, current_tokens, reset_time)
```

**Time Complexity:** O(1) - Constant time operations

**Space Complexity:** O(1) - Fixed storage per user (2 values)

**Example Usage:**

```
// User with 10 requests/second limit, burst of 10
result = token_bucket_check("user_12345", capacity=10, refill_rate=10)

if result.allowed:
    process_request()
    set_headers("X-RateLimit-Remaining", result.remaining)
else:
    return_429_error("Rate limit exceeded", result.reset_time)
```

---

## 2. Sliding Window Counter Algorithm

### sliding_window_check()

**Purpose:** Check rate limit using Sliding Window Counter, which prevents boundary bursts by using weighted sum of
current and previous windows.

**Parameters:**

- `user_id` (string): Unique identifier for the user
- `limit` (int): Maximum requests per window
- `window_size` (int): Window size in seconds (default: 60)

**Returns:**

- `(allowed: bool, current_rate: float, remaining: int)` - Whether allowed, current calculated rate, requests remaining

**Algorithm:**

```
function sliding_window_check(user_id, limit, window_size=60):
    // Calculate window IDs
    now_timestamp = now()  // Unix timestamp in seconds
    current_window_id = floor(now_timestamp / window_size)
    previous_window_id = current_window_id - 1
    
    // Generate Redis keys
    current_key = "user:" + user_id + ":window:" + current_window_id
    previous_key = "user:" + user_id + ":window:" + previous_window_id
    
    // Read counters from Redis (pipeline for efficiency)
    pipeline = redis.pipeline()
    pipeline.GET(current_key)
    pipeline.GET(previous_key)
    results = pipeline.execute()
    
    current_count = results[0] or 0
    previous_count = results[1] or 0
    
    // Calculate position within current window (0.0 to 1.0)
    elapsed_in_window = now_timestamp % window_size
    overlap_percentage = elapsed_in_window / window_size
    
    // Calculate weighted rate (sliding window approximation)
    // As we progress through current window, previous window contributes less
    rate = previous_count * (1 - overlap_percentage) + current_count
    
    // Check if under limit
    if rate < limit:
        // Allow request: Increment current window counter
        new_count = redis.INCR(current_key)
        
        // Set TTL if this is a new key (2× window size for safety)
        if new_count == 1:
            redis.EXPIRE(current_key, window_size * 2)
        
        allowed = true
        remaining = limit - ceil(rate) - 1
    else:
        // Reject request
        allowed = false
        remaining = 0
    
    return (allowed, rate, remaining)
```

**Time Complexity:** O(1) - Constant time operations

**Space Complexity:** O(1) - Two counters per user

**Example Usage:**

```
// User with 100 requests/minute limit
result = sliding_window_check("user_67890", limit=100, window_size=60)

if result.allowed:
    process_request()
    log("Current rate: " + result.current_rate + "/100")
else:
    return_429_error("Rate limit: 100 requests/minute")
```

**Why It Works (Example):**

```
Time: 1.5 seconds into minute (50% overlap)
Previous window (0-60s): 80 requests
Current window (60-120s): 30 requests

Rate = 80 × (1 - 0.5) + 30
     = 80 × 0.5 + 30
     = 40 + 30
     = 70 requests

At 0% into window: rate = 80 × 1.0 + 0 = 80 (mostly previous)
At 50% into window: rate = 80 × 0.5 + 30 = 70 (equal weight)
At 99% into window: rate = 80 × 0.01 + 75 = 75.8 (mostly current)
```

---

## 3. Fixed Window Counter Algorithm

### fixed_window_check()

**Purpose:** Check rate limit using Fixed Window Counter (simplest but has boundary burst problem).

**Parameters:**

- `user_id` (string): Unique identifier
- `limit` (int): Maximum requests per window
- `window_size` (int): Window size in seconds

**Returns:**

- `(allowed: bool, count: int, reset_time: timestamp)` - Whether allowed, current count, when window resets

**Algorithm:**

```
function fixed_window_check(user_id, limit, window_size=60):
    // Calculate current window ID
    now_timestamp = now()
    window_id = floor(now_timestamp / window_size)
    
    // Generate Redis key
    key = "user:" + user_id + ":fixed:" + window_id
    
    // Atomically increment counter
    count = redis.INCR(key)
    
    // Set TTL if this is first request in window
    if count == 1:
        redis.EXPIRE(key, window_size * 2)
    
    // Check if under limit
    if count <= limit:
        allowed = true
    else:
        allowed = false
    
    // Calculate when window resets
    next_window_start = (window_id + 1) * window_size
    reset_time = next_window_start
    
    return (allowed, count, reset_time)
```

**Time Complexity:** O(1)

**Space Complexity:** O(1) - One counter per user per window

**⚠️ Boundary Burst Problem:**

```
Window 1 (0-60s): User makes 100 requests at t=59s
Window 2 (60-120s): User makes 100 requests at t=60s
Total: 200 requests in 1 second! (limit should be 100)
```

**Example Usage:**

```
// Simple hourly limit (acceptable for coarse limits)
result = fixed_window_check("user_99999", limit=1000, window_size=3600)

if result.allowed:
    process_request()
else:
    return_429_error("Try again at " + result.reset_time)
```

**When to Use:**

- ✅ Coarse limits (hourly, daily quotas)
- ✅ When boundary bursts acceptable
- ❌ Do not use for strict rate limiting

---

## 4. Atomic Counter Operations

### check_rate_limit_atomic()

**Purpose:** Atomic rate limit check using increment-then-check pattern to prevent race conditions.

**Parameters:**

- `user_id` (string): User identifier
- `limit` (int): Request limit
- `window` (int): Window size in seconds

**Returns:**

- `allowed` (bool): Whether request should be allowed

**Algorithm:**

```
function check_rate_limit_atomic(user_id, limit, window):
    // Generate key for current window
    window_id = floor(now() / window)
    key = "rate:" + user_id + ":" + window_id
    
    // CRITICAL: Increment FIRST (atomic operation)
    // This prevents race condition where two gateways
    // both read 99, both allow, both increment to 100+
    new_count = redis.INCR(key)
    
    // Set TTL if this is first request
    if new_count == 1:
        redis.EXPIRE(key, window * 2)
    
    // THEN check if over limit
    if new_count <= limit:
        return true  // Allow
    else:
        return false  // Reject (429)
```

**Why Increment-Then-Check:**

```
❌ WRONG (Check-Then-Increment):
  Gateway 1: count = GET(key) → 99
  Gateway 2: count = GET(key) → 99
  Gateway 1: if 99 < 100 → Allow, INCR → 100
  Gateway 2: if 99 < 100 → Allow, INCR → 101
  Result: Both allowed, limit exceeded!

✅ CORRECT (Increment-Then-Check):
  Gateway 1: count = INCR(key) → 100
  Gateway 2: count = INCR(key) → 101
  Gateway 1: if 100 <= 100 → Allow
  Gateway 2: if 101 > 100 → Reject
  Result: Only Gateway 1 allowed!
```

**Time Complexity:** O(1)

**Example Usage:**

```
allowed = check_rate_limit_atomic("user_555", limit=100, window=60)
if not allowed:
    return_429()
```

---

### redis_atomic_incr()

**Purpose:** Wrapper for Redis atomic increment with error handling.

**Parameters:**

- `key` (string): Redis key
- `amount` (int): Increment amount (default: 1)
- `ttl` (int): Expiration time in seconds (optional)

**Returns:**

- `new_value` (int): Value after increment

**Algorithm:**

```
function redis_atomic_incr(key, amount=1, ttl=null):
    try:
        // Use pipeline for efficiency (single roundtrip)
        pipeline = redis.pipeline()
        
        if amount == 1:
            pipeline.INCR(key)
        else:
            pipeline.INCRBY(key, amount)
        
        // Set TTL if provided and key is new
        if ttl is not null:
            pipeline.EXPIRE(key, ttl)
        
        results = pipeline.execute()
        new_value = results[0]
        
        return new_value
        
    catch RedisError as e:
        log_error("Redis INCR failed", key, e)
        raise e
```

**Time Complexity:** O(1)

**Example Usage:**

```
count = redis_atomic_incr("user:123:count", amount=1, ttl=3600)
```

---

### increment_then_check()

**Purpose:** Complete rate limit check with increment-then-check pattern, including headers.

**Parameters:**

- `user_id` (string): User identifier
- `limit` (int): Request limit
- `window` (int): Window size in seconds

**Returns:**

- `(allowed: bool, headers: dict)` - Allow decision and HTTP headers

**Algorithm:**

```
function increment_then_check(user_id, limit, window):
    window_id = floor(now() / window)
    key = "rate:" + user_id + ":" + window_id
    
    // Atomic increment
    try:
        count = redis.INCR(key)
        redis.EXPIRE(key, window * 2)
    catch RedisError:
        // Fail open: Allow request if Redis unavailable
        return (true, create_headers(limit, limit, now() + window))
    
    // Check limit
    allowed = (count <= limit)
    remaining = max(0, limit - count)
    reset_time = (window_id + 1) * window
    
    // Create standard rate limit headers (RFC 6585)
    headers = create_headers(limit, remaining, reset_time)
    
    return (allowed, headers)

function create_headers(limit, remaining, reset_time):
    return {
        "X-RateLimit-Limit": limit,
        "X-RateLimit-Remaining": remaining,
        "X-RateLimit-Reset": reset_time,
        "Retry-After": max(0, reset_time - now())
    }
```

**Time Complexity:** O(1)

**Example Usage:**

```
allowed, headers = increment_then_check("user_789", limit=100, window=60)

if allowed:
    return_200(response, headers)
else:
    return_429("Rate limit exceeded", headers)
```

---

## 5. Circuit Breaker Implementation

### check_rate_limit_with_circuit_breaker()

**Purpose:** Rate limit check with circuit breaker for graceful degradation when Redis fails.

**Parameters:**

- `user_id` (string): User identifier
- `limit` (int): Request limit
- `window` (int): Window size

**Returns:**

- `(allowed: bool, status: string)` - Whether allowed and circuit breaker status

**Algorithm:**

```
// Global circuit breaker state
circuit_breaker = {
    state: "CLOSED",  // CLOSED, OPEN, HALF_OPEN
    failure_count: 0,
    success_count: 0,
    last_failure_time: 0,
    error_threshold: 0.5,  // 50% error rate
    window_size: 10,  // 10 seconds
    timeout: 60,  // 60 seconds before testing recovery
    requests_in_window: []
}

function check_rate_limit_with_circuit_breaker(user_id, limit, window):
    // Check circuit breaker state
    if circuit_breaker.state == "OPEN":
        // Circuit open: Check if should test recovery
        if (now() - circuit_breaker.last_failure_time) > circuit_breaker.timeout:
            circuit_breaker.state = "HALF_OPEN"
            circuit_breaker.success_count = 0
        else:
            // Still open: Fail open (allow without checking Redis)
            log_warning("Circuit breaker OPEN, allowing request")
            return (true, "FAIL_OPEN")
    
    // Attempt rate limit check
    try:
        allowed = check_rate_limit_atomic(user_id, limit, window)
        
        // Record success
        record_success()
        
        // If half-open and success, close circuit
        if circuit_breaker.state == "HALF_OPEN":
            circuit_breaker.success_count += 1
            if circuit_breaker.success_count >= 5:
                circuit_breaker.state = "CLOSED"
                circuit_breaker.failure_count = 0
                log_info("Circuit breaker CLOSED (recovered)")
        
        return (allowed, "NORMAL")
        
    catch RedisError as e:
        // Record failure
        record_failure()
        
        // Check if should open circuit
        error_rate = calculate_error_rate()
        if error_rate > circuit_breaker.error_threshold:
            circuit_breaker.state = "OPEN"
            circuit_breaker.last_failure_time = now()
            log_critical("Circuit breaker OPEN (Redis unavailable)")
        
        // Fail open: Allow request
        return (true, "FAIL_OPEN")

function record_success():
    circuit_breaker.requests_in_window.append({
        timestamp: now(),
        success: true
    })
    cleanup_old_requests()

function record_failure():
    circuit_breaker.failure_count += 1
    circuit_breaker.requests_in_window.append({
        timestamp: now(),
        success: false
    })
    cleanup_old_requests()

function cleanup_old_requests():
    cutoff = now() - circuit_breaker.window_size
    circuit_breaker.requests_in_window = filter(
        circuit_breaker.requests_in_window,
        lambda r: r.timestamp > cutoff
    )

function calculate_error_rate():
    if len(circuit_breaker.requests_in_window) == 0:
        return 0
    
    failures = count(circuit_breaker.requests_in_window, lambda r: not r.success)
    total = len(circuit_breaker.requests_in_window)
    return failures / total
```

**Time Complexity:** O(1) amortized (cleanup is O(n) but infrequent)

**Space Complexity:** O(n) where n is requests in window

**Circuit Breaker States:**

- **CLOSED:** Normal operation, all requests check Redis
- **OPEN:** Redis failing, all requests allowed without check (fail-open)
- **HALF_OPEN:** Testing recovery, single request tries Redis

**Example Usage:**

```
allowed, status = check_rate_limit_with_circuit_breaker("user_123", 100, 60)

if status == "FAIL_OPEN":
    log_alert("Rate limiting disabled (Redis unavailable)")
    
if allowed:
    process_request()
else:
    return_429()
```

---

### redis_call_with_timeout()

**Purpose:** Call Redis with timeout and circuit breaker protection.

**Parameters:**

- `operation` (function): Redis operation to execute
- `timeout_ms` (int): Timeout in milliseconds (default: 5)
- `default_value` (any): Value to return on timeout/error

**Returns:**

- Result of operation or default_value on error

**Algorithm:**

```
function redis_call_with_timeout(operation, timeout_ms=5, default_value=null):
    try:
        // Set socket timeout on Redis connection
        redis.set_timeout(timeout_ms)
        
        // Execute operation
        result = operation()
        
        return result
        
    catch TimeoutError:
        log_warning("Redis timeout after " + timeout_ms + "ms")
        record_circuit_breaker_failure()
        return default_value
        
    catch ConnectionError:
        log_error("Redis connection failed")
        record_circuit_breaker_failure()
        return default_value
        
    catch RedisError as e:
        log_error("Redis error: " + e)
        record_circuit_breaker_failure()
        return default_value
        
    finally:
        // Reset timeout to default
        redis.set_timeout(1000)
```

**Time Complexity:** O(1) plus operation time

**Example Usage:**

```
count = redis_call_with_timeout(
    lambda: redis.INCR("user:123:count"),
    timeout_ms=5,
    default_value=0
)

if count == 0:
    // Redis failed, fail open
    return (true, "FAIL_OPEN")
```

---

## 6. Hot Key Detection

### detect_hot_keys()

**Purpose:** Monitor Redis access patterns and detect "hot keys" (keys accessed abnormally frequently).

**Parameters:**

- `monitoring_window` (int): Time window for detection in seconds (default: 60)
- `threshold` (int): Accesses per second to trigger alert (default: 10000)

**Returns:**

- `hot_keys` (list): List of detected hot keys with access counts

**Algorithm:**

```
// Global state for tracking key access
key_access_tracker = {}  // key → [timestamp1, timestamp2, ...]

function detect_hot_keys(monitoring_window=60, threshold=10000):
    hot_keys = []
    now_time = now()
    cutoff_time = now_time - monitoring_window
    
    // Iterate through all tracked keys
    for key, timestamps in key_access_tracker:
        // Remove old timestamps (outside monitoring window)
        recent_accesses = filter(timestamps, lambda t: t > cutoff_time)
        key_access_tracker[key] = recent_accesses
        
        // Calculate access rate (accesses per second)
        access_count = len(recent_accesses)
        access_rate = access_count / monitoring_window
        
        // Check if exceeds threshold
        if access_rate > threshold:
            hot_keys.append({
                key: key,
                access_rate: access_rate,
                access_count: access_count
            })
    
    // Sort by access rate (highest first)
    hot_keys.sort(key=lambda h: h.access_rate, reverse=true)
    
    return hot_keys

function record_key_access(key):
    // Record timestamp of this access
    if key not in key_access_tracker:
        key_access_tracker[key] = []
    
    key_access_tracker[key].append(now())
    
    // Periodically cleanup old entries (every 1000 accesses)
    if random() < 0.001:  // 0.1% of calls
        cleanup_old_accesses()

function cleanup_old_accesses():
    cutoff = now() - 300  // Keep last 5 minutes
    for key in key_access_tracker:
        key_access_tracker[key] = filter(
            key_access_tracker[key],
            lambda t: t > cutoff
        )
```

**Time Complexity:** O(n × m) where n = number of keys, m = accesses per key

**Space Complexity:** O(n × m)

**Example Usage:**

```
// In rate limiter module, record each access
record_key_access("user:12345:count")

// Periodically check for hot keys (every 60 seconds)
hot_keys = detect_hot_keys(monitoring_window=60, threshold=10000)

for hot_key in hot_keys:
    log_alert("Hot key detected: " + hot_key.key)
    log_alert("Access rate: " + hot_key.access_rate + " per second")
    
    // Trigger mitigation
    enable_l1_cache_for_key(hot_key.key)
```

---

## 7. L1 Cache Implementation

### check_rate_limit_with_l1_cache()

**Purpose:** Check rate limit using local L1 cache to reduce Redis load for hot keys.

**Parameters:**

- `user_id` (string): User identifier
- `limit` (int): Request limit
- `window` (int): Window size in seconds
- `batch_interval` (int): How often to sync to Redis in milliseconds (default: 100)

**Returns:**

- `(allowed: bool, source: string)` - Whether allowed and cache source (L1 or L2)

**Algorithm:**

```
// Local in-memory cache (per gateway node)
l1_cache = {}  // key → {count, last_sync, enabled}

function check_rate_limit_with_l1_cache(user_id, limit, window, batch_interval=100):
    key = "rate:" + user_id + ":" + floor(now() / window)
    
    // Check if L1 cache enabled for this key (hot key)
    if key not in l1_cache or not l1_cache[key].enabled:
        // L1 cache not enabled: Use normal Redis check
        return check_rate_limit_atomic(user_id, limit, window), "L2_REDIS"
    
    // L1 cache enabled: Check locally first
    cache_entry = l1_cache[key]
    
    // Increment local counter (fast, no network)
    cache_entry.count += 1
    
    // Check if under limit locally (approximate)
    // Allow 10% overage to account for distributed nature
    local_limit = limit * 1.1
    
    if cache_entry.count > local_limit:
        // Over local limit: Force sync to Redis
        sync_l1_to_redis(key, user_id, limit, window)
        return false, "L1_CACHE_REJECT"
    
    // Check if batch interval elapsed
    if (now() - cache_entry.last_sync) > batch_interval:
        // Async sync to Redis (don't wait for response)
        spawn_async(lambda: sync_l1_to_redis(key, user_id, limit, window))
    
    return true, "L1_CACHE_ALLOW"

function sync_l1_to_redis(key, user_id, limit, window):
    cache_entry = l1_cache[key]
    
    // Send batched increment to Redis
    local_count = cache_entry.count
    if local_count > 0:
        try:
            redis.INCRBY(key, local_count)
            
            // Reset local counter
            cache_entry.count = 0
            cache_entry.last_sync = now()
            
        catch RedisError:
            // Keep local count if Redis fails
            log_warning("L1 cache sync failed, retrying later")

function enable_l1_cache_for_key(key, limit):
    l1_cache[key] = {
        count: 0,
        last_sync: now(),
        enabled: true,
        limit: limit
    }
    log_info("L1 cache enabled for hot key: " + key)

function disable_l1_cache_for_key(key):
    if key in l1_cache:
        // Final sync before disabling
        sync_l1_to_redis(key, extract_user_id(key), l1_cache[key].limit, 60)
        l1_cache[key].enabled = false
    log_info("L1 cache disabled for key: " + key)
```

**Time Complexity:**

- L1 check: O(1) - in-memory, microseconds
- Redis sync: O(1) - async, doesn't block

**Space Complexity:** O(n) where n = number of hot keys cached

**Example Usage:**

```
// Detect hot key
hot_keys = detect_hot_keys()
for hot_key in hot_keys:
    enable_l1_cache_for_key(hot_key.key, hot_key.limit)

// Rate limit check (uses L1 cache if enabled)
allowed, source = check_rate_limit_with_l1_cache("user_attacker", 100, 60)

log("Request allowed: " + allowed + ", Source: " + source)
// Output: "Request allowed: true, Source: L1_CACHE_ALLOW"
```

**Performance Impact:**

```
Without L1 Cache:
  100K requests/sec for hot key
  → 100K Redis ops/sec
  → Redis overloaded

With L1 Cache:
  100K requests/sec for hot key
  → 99K handled by L1 (local)
  → 1K Redis ops/sec (batched every 100ms)
  → 99% load reduction!
```

---

## 8. Multi-Tier Rate Limiting

### get_user_tier_and_limit()

**Purpose:** Lookup user tier and corresponding rate limit from cache or database.

**Parameters:**

- `user_id` (string): User identifier

**Returns:**

- `(tier: string, limit: int, window: int)` - User tier, request limit, window size

**Algorithm:**

```
// Tier configuration
tier_config = {
    "free": {limit: 10, window: 1},
    "basic": {limit: 100, window: 1},
    "premium": {limit: 1000, window: 1},
    "enterprise": {limit: 10000, window: 1}
}

// Local LRU cache (10K users)
tier_cache = LRUCache(capacity=10000, ttl=300)  // 5-minute TTL

function get_user_tier_and_limit(user_id):
    // Check cache first
    cache_key = "tier:" + user_id
    cached = tier_cache.get(cache_key)
    
    if cached is not NULL:
        return cached  // Cache hit
    
    // Cache miss: Query database
    try:
        user_record = database.query(
            "SELECT tier FROM users WHERE id = ?",
            user_id
        )
        
        if user_record is NULL:
            // User not found: Default to free tier
            tier = "free"
        else:
            tier = user_record.tier
        
        // Lookup tier limits
        if tier not in tier_config:
            log_warning("Unknown tier: " + tier + ", using free")
            tier = "free"
        
        limit = tier_config[tier].limit
        window = tier_config[tier].window
        
        result = (tier, limit, window)
        
        // Cache for future requests
        tier_cache.set(cache_key, result)
        
        return result
        
    catch DatabaseError as e:
        log_error("Failed to query user tier: " + e)
        // Fail safe: Use free tier
        return ("free", 10, 1)
```

**Time Complexity:**

- Cache hit: O(1)
- Cache miss: O(1) database query

**Space Complexity:** O(n) where n = cache size (10K users)

**Example Usage:**

```
tier, limit, window = get_user_tier_and_limit("user_12345")
// tier="premium", limit=1000, window=1

allowed = check_rate_limit_atomic("user_12345", limit, window)
```

---

### check_rate_limit_multi_tier()

**Purpose:** Complete rate limit check with tier-based limits.

**Parameters:**

- `user_id` (string): User identifier

**Returns:**

- `(allowed: bool, tier: string, headers: dict)` - Allow decision, user tier, HTTP headers

**Algorithm:**

```
function check_rate_limit_multi_tier(user_id):
    // Lookup user tier and limits
    tier, limit, window = get_user_tier_and_limit(user_id)
    
    // Apply tier-specific rate limit
    window_id = floor(now() / window)
    key = "rate:" + user_id + ":" + tier + ":" + window_id
    
    // Atomic increment
    count = redis.INCR(key)
    redis.EXPIRE(key, window * 2)
    
    // Check limit
    allowed = (count <= limit)
    remaining = max(0, limit - count)
    reset_time = (window_id + 1) * window
    
    // Create headers with tier info
    headers = {
        "X-RateLimit-Limit": limit,
        "X-RateLimit-Remaining": remaining,
        "X-RateLimit-Reset": reset_time,
        "X-RateLimit-Tier": tier,
        "Retry-After": max(0, reset_time - now())
    }
    
    return (allowed, tier, headers)
```

**Time Complexity:** O(1)

**Example Usage:**

```
allowed, tier, headers = check_rate_limit_multi_tier("user_premium_123")

if allowed:
    return_200(response, headers)
else:
    return_429({
        "error": "Rate limit exceeded for " + tier + " tier",
        "upgrade_url": "https://example.com/upgrade"
    }, headers)
```

---

## 9. Configuration Management

### load_rate_limit_config()

**Purpose:** Load rate limit configuration from etcd with watch for updates.

**Parameters:**

- `config_path` (string): Path in etcd (default: "/rate_limiter/config")

**Returns:**

- `config` (dict): Loaded configuration

**Algorithm:**

```
// Global configuration state
current_config = {}
config_version = 0

function load_rate_limit_config(config_path="/rate_limiter/config"):
    try:
        // Read config from etcd
        response = etcd.get(config_path)
        config_json = response.value
        new_config = json.parse(config_json)
        
        // Validate configuration
        if not validate_config(new_config):
            log_error("Invalid configuration, keeping current")
            return current_config
        
        // Update global config
        current_config = new_config
        config_version = new_config.version
        
        log_info("Loaded config version " + config_version)
        return current_config
        
    catch etcd.Error as e:
        log_error("Failed to load config from etcd: " + e)
        return current_config  // Keep current config

function watch_config_updates(config_path="/rate_limiter/config"):
    // Set up watch on etcd key
    etcd.watch(config_path, callback=on_config_change)
    log_info("Watching for config updates at " + config_path)

function on_config_change(event):
    log_info("Config change detected")
    
    // Reload configuration
    new_config = load_rate_limit_config()
    
    // Clear tier cache to pick up new limits
    tier_cache.clear()
    
    // Clear L1 cache
    l1_cache.clear()
    
    log_info("Configuration reloaded, version " + config_version)

function validate_config(config):
    // Validate required fields
    required_fields = ["rate_limits", "version", "updated_at"]
    for field in required_fields:
        if field not in config:
            log_error("Missing required field: " + field)
            return false
    
    // Validate rate limits
    for tier, limits in config.rate_limits:
        if "limit" not in limits or "window" not in limits:
            log_error("Invalid limits for tier: " + tier)
            return false
        
        if limits.limit <= 0 or limits.window <= 0:
            log_error("Limits must be positive: " + tier)
            return false
    
    return true

// Initialize on startup
function init():
    load_rate_limit_config()
    watch_config_updates()
```

**Time Complexity:** O(1) for config lookup after load

**Example Usage:**

```
// On startup
init()

// Config automatically updates when changed in etcd
// Admin updates config:
//   $ etcdctl put /rate_limiter/config '{"rate_limits":{"premium":{"limit":200}},"version":43}'
//
// Within 1 second:
//   - watch_config_updates() triggers
//   - on_config_change() fires
//   - New config loaded
//   - Next request uses new limit (200)
```

---

### connection_pool_setup()

**Purpose:** Setup Redis connection pool for efficient connection reuse.

**Parameters:**

- `redis_hosts` (list): List of Redis host:port pairs
- `pool_size` (int): Connections per host (default: 100)

**Returns:**

- `pool` (RedisPool): Connection pool object

**Algorithm:**

```
function connection_pool_setup(redis_hosts, pool_size=100):
    // Calculate optimal pool size
    // Formula: (Gateway threads × Concurrent requests) / Redis shards
    // Example: (100 threads × 10 concurrent) / 10 shards = 100 connections/shard
    
    pool_config = {
        max_connections: pool_size,
        socket_timeout: 5,  // 5ms timeout
        socket_connect_timeout: 10,  // 10ms connect timeout
        socket_keepalive: true,
        socket_keepalive_options: {
            TCP_KEEPIDLE: 60,
            TCP_KEEPINTVL: 10,
            TCP_KEEPCNT: 3
        },
        retry_on_timeout: false,  // Fail fast
        health_check_interval: 30  // Check connection health every 30s
    }
    
    // Create connection pool
    pool = RedisConnectionPool(redis_hosts, pool_config)
    
    log_info("Redis connection pool created: " + pool_size + " connections per shard")
    
    return pool
```

**Time Complexity:** O(1) per Redis operation (reuses connections)

**Example Usage:**

```
// On startup
redis_hosts = ["redis-1:6379", "redis-2:6379", "redis-3:6379"]
redis_pool = connection_pool_setup(redis_hosts, pool_size=100)

// Use pool for all Redis operations
redis_client = redis_pool.get_connection()
count = redis_client.INCR("user:123:count")
redis_pool.release_connection(redis_client)
```

---

**Summary:** All algorithms prioritize:

1. **Atomicity:** Prevent race conditions
2. **Latency:** Sub-millisecond operations
3. **Scalability:** Horizontal scaling via sharding
4. **Reliability:** Fail-open strategy, circuit breakers
