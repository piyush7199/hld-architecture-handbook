# Pseudocode Implementations: Notification Service

This document contains detailed algorithm implementations for the Notification Service design challenge. The main design document describes WHAT and WHY, while this file shows HOW each algorithm works.

## Table of Contents

1. [Notification Service Core](#1-notification-service-core)
   - [processNotification()](#processnotification)
   - [selectChannels()](#selectchannels)
   - [enrichTemplate()](#enrichtemplate)
2. [Idempotency & Deduplication](#2-idempotency--deduplication)
   - [isDuplicate()](#isduplicate)
   - [storeIdempotencyKey()](#storeidempotencykey)
3. [Rate Limiting](#3-rate-limiting)
   - [TokenBucket class](#tokenbucket-class)
   - [consumeTokens()](#consumetokens)
4. [Circuit Breaker](#4-circuit-breaker)
   - [CircuitBreaker class](#circuitbreaker-class)
   - [recordSuccess()](#recordsuccess)
   - [recordFailure()](#recordfailure)
5. [Channel Workers](#5-channel-workers)
   - [EmailWorker.processMessage()](#emailworkerprocessmessage)
   - [SMSWorker.processMessage()](#smsworkerprocessmessage)
   - [PushWorker.batchSend()](#pushworkerbatchsend)
6. [WebSocket Connection Management](#6-websocket-connection-management)
   - [WebSocketManager.registerConnection()](#websocketmanagerregisterconnection)
   - [WebSocketManager.sendNotification()](#websocketmanagersendnotification)
   - [WebSocketManager.handleDisconnect()](#websocketmanagerhandledisconnect)
7. [Cache Operations](#7-cache-operations)
   - [fetchUserPreferences()](#fetchuserpreferences)
   - [invalidateCache()](#invalidatecache)
8. [Retry Logic](#8-retry-logic)
   - [retryWithBackoff()](#retrywithbackoff)
9. [Dead Letter Queue](#9-dead-letter-queue)
   - [moveToDLQ()](#movetodlq)
10. [Metrics & Logging](#10-metrics--logging)
    - [logNotificationStatus()](#lognotificationstatus)
    - [emitMetrics()](#emitmetrics)

---

## 1. Notification Service Core

### processNotification()

**Purpose:** Main entry point for processing a notification request from API Gateway.

**Parameters:**
- `request`: object - HTTP request containing notification details
  - `user_id`: int - Target user ID
  - `event_type`: string - Event that triggered notification (e.g., "order_confirmed")
  - `template_id`: string - Template to use
  - `variables`: map - Variables for template substitution
  - `priority`: string - Notification priority ("critical", "high", "normal", "low")
  - `idempotency_key`: string - Unique key to prevent duplicates

**Returns:** object - Response with notification_id and status

**Algorithm:**
```
function processNotification(request):
    // Step 1: Validate request
    if not validateRequest(request):
        return error("Invalid request", 400)
    
    // Step 2: Check idempotency
    if isDuplicate(request.idempotency_key):
        notification_id = getExistingNotificationId(request.idempotency_key)
        return {
            status: "already_processed",
            notification_id: notification_id,
            http_code: 200
        }
    
    // Step 3: Generate notification ID
    notification_id = generateUUID()
    
    // Step 4: Store idempotency key
    storeIdempotencyKey(request.idempotency_key, notification_id, ttl=86400) // 24 hours
    
    // Step 5: Fetch user preferences
    user_prefs = fetchUserPreferences(request.user_id)
    if not user_prefs:
        return error("User not found", 404)
    
    // Step 6: Fetch and enrich template
    template = fetchTemplate(request.template_id)
    if not template:
        return error("Template not found", 404)
    
    enriched_content = enrichTemplate(template, request.variables)
    
    // Step 7: Select channels based on user preferences
    channels = selectChannels(user_prefs, request.priority)
    
    // Step 8: Publish to Kafka topics for each channel
    for each channel in channels:
        message = {
            notification_id: notification_id,
            user_id: request.user_id,
            channel: channel,
            content: enriched_content[channel],
            priority: request.priority,
            timestamp: current_time()
        }
        
        publishToKafka(topic=channel + "-notifications", message=message)
    
    // Step 9: Log creation event
    logNotificationStatus(notification_id, "created", channels)
    
    // Step 10: Return response
    return {
        status: "accepted",
        notification_id: notification_id,
        channels: channels,
        http_code: 202
    }
```

**Time Complexity:** $O(n)$ where $n$ is number of channels (typically 1-4)  
**Space Complexity:** $O(1)$

**Example Usage:**
```
request = {
    user_id: 123456,
    event_type: "order_confirmed",
    template_id: "order_confirmation",
    variables: {
        order_id: "ORD-789",
        amount: "$99.99",
        delivery_date: "2024-01-15"
    },
    priority: "high",
    idempotency_key: "order-789-confirmation"
}

response = processNotification(request)
// Returns: {status: "accepted", notification_id: UUID, channels: ["email", "push", "web"], http_code: 202}
```

---

### selectChannels()

**Purpose:** Determine which notification channels to use based on user preferences and notification priority.

**Parameters:**
- `user_prefs`: object - User notification preferences
  - `email_enabled`: boolean
  - `sms_enabled`: boolean
  - `push_enabled`: boolean
  - `web_enabled`: boolean
  - `quiet_hours_start`: time
  - `quiet_hours_end`: time
- `priority`: string - Notification priority level

**Returns:** array - List of channels to use

**Algorithm:**
```
function selectChannels(user_prefs, priority):
    channels = []
    current_time = getCurrentTime()
    
    // Critical notifications ignore quiet hours and user preferences
    if priority == "critical":
        if user_prefs.sms_enabled:
            channels.append("sms")
        channels.append("push")
        channels.append("email")
        channels.append("web")
        return channels
    
    // Check if current time is within quiet hours
    in_quiet_hours = isInQuietHours(current_time, user_prefs.quiet_hours_start, user_prefs.quiet_hours_end)
    
    // During quiet hours, only send to non-disruptive channels
    if in_quiet_hours:
        if user_prefs.email_enabled:
            channels.append("email")
        if user_prefs.web_enabled:
            channels.append("web")
        return channels
    
    // Normal hours: respect all user preferences
    if user_prefs.email_enabled:
        channels.append("email")
    
    if user_prefs.sms_enabled:
        channels.append("sms")
    
    if user_prefs.push_enabled:
        channels.append("push")
    
    if user_prefs.web_enabled:
        channels.append("web")
    
    // If all channels disabled, at least use email (important notifications)
    if channels.isEmpty():
        channels.append("email")
    
    return channels
```

**Time Complexity:** $O(1)$  
**Space Complexity:** $O(1)$

---

### enrichTemplate()

**Purpose:** Substitute variables in notification template with actual values.

**Parameters:**
- `template`: object - Notification template
  - `subject`: string - Subject line with {{variables}}
  - `body`: string - Body content with {{variables}}
- `variables`: map<string, string> - Variable name to value mapping

**Returns:** object - Enriched content for all channels

**Algorithm:**
```
function enrichTemplate(template, variables):
    enriched = {}
    
    // Enrich subject (for email)
    enriched.subject = template.subject
    for each (key, value) in variables:
        placeholder = "{{" + key + "}}"
        enriched.subject = enriched.subject.replace(placeholder, value)
    
    // Enrich body
    enriched.body = template.body
    for each (key, value) in variables:
        placeholder = "{{" + key + "}}"
        enriched.body = enriched.body.replace(placeholder, value)
    
    // Generate channel-specific content
    enriched.email = {
        subject: enriched.subject,
        body: enriched.body,
        html: renderHTML(enriched.body)  // Convert to HTML
    }
    
    enriched.sms = {
        message: truncate(enriched.body, 160)  // SMS character limit
    }
    
    enriched.push = {
        title: enriched.subject,
        body: truncate(enriched.body, 100)  // Push notification limit
    }
    
    enriched.web = {
        title: enriched.subject,
        body: enriched.body,
        icon: template.icon_url
    }
    
    return enriched
```

**Time Complexity:** $O(m \times n)$ where $m$ = template length, $n$ = number of variables  
**Space Complexity:** $O(m)$

---

## 2. Idempotency & Deduplication

### isDuplicate()

**Purpose:** Check if a notification request with the given idempotency key has already been processed.

**Parameters:**
- `idempotency_key`: string - Unique key for the notification request

**Returns:** boolean - True if duplicate, False otherwise

**Algorithm:**
```
function isDuplicate(idempotency_key):
    // Query Redis for idempotency key
    redis_key = "idempotency:" + idempotency_key
    result = redis.get(redis_key)
    
    if result == null:
        return False  // Not a duplicate
    else:
        return True   // Duplicate found
```

**Time Complexity:** $O(1)$  
**Space Complexity:** $O(1)$

---

### storeIdempotencyKey()

**Purpose:** Store idempotency key with notification ID to prevent duplicate processing.

**Parameters:**
- `idempotency_key`: string - Unique key
- `notification_id`: string - Generated notification UUID
- `ttl`: int - Time to live in seconds

**Returns:** void

**Algorithm:**
```
function storeIdempotencyKey(idempotency_key, notification_id, ttl):
    redis_key = "idempotency:" + idempotency_key
    value = {
        notification_id: notification_id,
        timestamp: current_time()
    }
    
    // Store in Redis with TTL
    redis.setex(redis_key, ttl, serialize(value))
```

**Time Complexity:** $O(1)$  
**Space Complexity:** $O(1)$

---

## 3. Rate Limiting

### TokenBucket class

**Purpose:** Implement Token Bucket algorithm for rate limiting API calls to third-party providers.

**Algorithm:**
```
class TokenBucket:
    capacity: int           // Maximum tokens (burst size)
    tokens: float           // Current available tokens
    refill_rate: float      // Tokens added per second
    last_refill: timestamp  // Last time tokens were refilled
    lock: mutex             // For thread safety
    
    function initialize(capacity, refill_rate):
        this.capacity = capacity
        this.tokens = capacity
        this.refill_rate = refill_rate
        this.last_refill = current_time()
        this.lock = new_mutex()
    
    function consume(count=1):
        acquire_lock(this.lock)
        
        try:
            // Refill tokens based on elapsed time
            now = current_time()
            elapsed = now - this.last_refill
            new_tokens = elapsed * this.refill_rate
            this.tokens = min(this.tokens + new_tokens, this.capacity)
            this.last_refill = now
            
            // Try to consume requested tokens
            if this.tokens >= count:
                this.tokens -= count
                return True  // Request allowed
            else:
                return False  // Request blocked
        finally:
            release_lock(this.lock)
    
    function wait_for_tokens(count=1):
        while not this.consume(count):
            // Calculate wait time
            wait_seconds = (count - this.tokens) / this.refill_rate
            sleep(wait_seconds)
```

**Time Complexity:** $O(1)$ per consume operation  
**Space Complexity:** $O(1)$

---

### consumeTokens()

**Purpose:** Distributed token bucket implementation using Redis for multiple workers.

**Parameters:**
- `key`: string - Redis key for the bucket (e.g., "ratelimit:email:bucket")
- `count`: int - Number of tokens to consume
- `capacity`: int - Bucket capacity
- `refill_rate`: float - Tokens per second

**Returns:** boolean - True if tokens consumed, False if insufficient

**Algorithm:**
```
function consumeTokens(key, count, capacity, refill_rate):
    // Redis Lua script for atomic token consumption
    lua_script = """
    local key = KEYS[1]
    local count = tonumber(ARGV[1])
    local capacity = tonumber(ARGV[2])
    local refill_rate = tonumber(ARGV[3])
    local now = tonumber(ARGV[4])
    
    local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
    local tokens = tonumber(bucket[1]) or capacity
    local last_refill = tonumber(bucket[2]) or now
    
    -- Refill tokens
    local elapsed = now - last_refill
    tokens = math.min(tokens + (elapsed * refill_rate), capacity)
    
    -- Try to consume
    if tokens >= count then
        tokens = tokens - count
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)  -- 1 hour TTL
        return 1  -- Success
    else
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)
        return 0  -- Insufficient tokens
    end
    """
    
    result = redis.eval(lua_script, 
                        keys=[key], 
                        args=[count, capacity, refill_rate, current_time()])
    
    return result == 1
```

**Time Complexity:** $O(1)$ (atomic Redis operation)  
**Space Complexity:** $O(1)$

---

## 4. Circuit Breaker

### CircuitBreaker class

**Purpose:** Implement Circuit Breaker pattern to prevent cascading failures when third-party APIs are down.

**Algorithm:**
```
class CircuitBreaker:
    state: string              // "CLOSED", "OPEN", "HALF_OPEN"
    failure_count: int         // Consecutive failures
    success_count: int         // Consecutive successes (in HALF_OPEN)
    failure_threshold: int     // Failures before opening
    success_threshold: int     // Successes to close from HALF_OPEN
    timeout: int               // Seconds before testing recovery
    last_failure_time: timestamp
    lock: mutex
    
    function initialize(failure_threshold=5, timeout=60, success_threshold=3):
        this.state = "CLOSED"
        this.failure_count = 0
        this.success_count = 0
        this.failure_threshold = failure_threshold
        this.timeout = timeout
        this.success_threshold = success_threshold
        this.lock = new_mutex()
    
    function is_request_allowed():
        acquire_lock(this.lock)
        
        try:
            if this.state == "CLOSED":
                return True  // Normal operation
            
            if this.state == "OPEN":
                // Check if timeout elapsed
                if (current_time() - this.last_failure_time) >= this.timeout:
                    this.state = "HALF_OPEN"
                    this.success_count = 0
                    return True  // Allow test request
                else:
                    return False  // Still in timeout
            
            if this.state == "HALF_OPEN":
                return True  // Allow test request
        finally:
            release_lock(this.lock)
    
    function record_success():
        acquire_lock(this.lock)
        
        try:
            if this.state == "CLOSED":
                this.failure_count = 0  // Reset failure count
            
            if this.state == "HALF_OPEN":
                this.success_count += 1
                if this.success_count >= this.success_threshold:
                    this.state = "CLOSED"
                    this.failure_count = 0
                    log("Circuit breaker CLOSED - Service recovered")
        finally:
            release_lock(this.lock)
    
    function record_failure():
        acquire_lock(this.lock)
        
        try:
            this.last_failure_time = current_time()
            
            if this.state == "CLOSED":
                this.failure_count += 1
                if this.failure_count >= this.failure_threshold:
                    this.state = "OPEN"
                    log("Circuit breaker OPEN - Service down")
                    alert("Circuit breaker opened", severity="HIGH")
            
            if this.state == "HALF_OPEN":
                this.state = "OPEN"
                this.success_count = 0
                log("Circuit breaker back to OPEN - Service still down")
        finally:
            release_lock(this.lock)
```

**Time Complexity:** $O(1)$ per operation  
**Space Complexity:** $O(1)$

---

### recordSuccess()

**Purpose:** Record a successful API call for circuit breaker state management.

(See CircuitBreaker class above)

---

### recordFailure()

**Purpose:** Record a failed API call for circuit breaker state management.

(See CircuitBreaker class above)

---

## 5. Channel Workers

### EmailWorker.processMessage()

**Purpose:** Process email notification message from Kafka and send via SendGrid.

**Parameters:**
- `message`: object - Kafka message containing notification details

**Returns:** void

**Algorithm:**
```
function processMessage(message):
    rate_limiter = getTokenBucket("email")
    circuit_breaker = getCircuitBreaker("email")
    
    // Step 1: Check circuit breaker
    if not circuit_breaker.is_request_allowed():
        log("Circuit breaker open - skipping message")
        return  // Don't commit offset, will retry later
    
    // Step 2: Wait for rate limiter
    if not rate_limiter.consume(1):
        sleep(calculateWaitTime(rate_limiter))
        if not rate_limiter.consume(1):
            log("Rate limit exceeded - delaying message")
            return  // Don't commit offset, will retry later
    
    // Step 3: Call SendGrid API with retry
    try:
        result = retryWithBackoff(
            function=sendGridAPI.sendEmail,
            args={
                to: getUserEmail(message.user_id),
                subject: message.content.subject,
                body: message.content.body,
                html: message.content.html
            },
            max_retries=3,
            backoff_factor=2
        )
        
        // Step 4: Record success
        circuit_breaker.record_success()
        
        // Step 5: Log delivery status
        logNotificationStatus(
            message.notification_id,
            "delivered",
            channel="email",
            provider_message_id=result.message_id
        )
        
        // Step 6: Commit Kafka offset
        commitOffset(message.partition, message.offset)
        
        // Step 7: Emit metrics
        emitMetrics("email.sent", 1, tags={priority: message.priority})
        
    catch APIError as e:
        circuit_breaker.record_failure()
        
        if e.is_permanent_error():  // 4xx errors
            // Move to Dead Letter Queue
            moveToDLQ(message, error=e.message, channel="email")
            commitOffset(message.partition, message.offset)
        else:
            // Transient error, will retry
            log("Transient error: " + e.message)
            // Don't commit offset
```

**Time Complexity:** $O(1)$ plus network I/O  
**Space Complexity:** $O(1)$

---

### SMSWorker.processMessage()

**Purpose:** Process SMS notification message from Kafka and send via Twilio.

**Algorithm:**
```
function processMessage(message):
    rate_limiter = getTokenBucket("sms")
    circuit_breaker = getCircuitBreaker("sms")
    
    // Check circuit breaker
    if not circuit_breaker.is_request_allowed():
        return  // Skip, will retry later
    
    // Wait for rate limiter (Twilio: 100 SMS/second)
    while not rate_limiter.consume(1):
        sleep(0.01)  // 10ms
    
    try:
        // Call Twilio API
        result = twilioAPI.sendSMS(
            to: getUserPhoneNumber(message.user_id),
            body: message.content.message
        )
        
        circuit_breaker.record_success()
        
        logNotificationStatus(
            message.notification_id,
            "sent",
            channel="sms",
            provider_message_id=result.sid
        )
        
        commitOffset(message.partition, message.offset)
        emitMetrics("sms.sent", 1)
        
    catch APIError as e:
        circuit_breaker.record_failure()
        
        if e.error_code == "invalid_phone_number":
            moveToDLQ(message, error=e.message, channel="sms")
            commitOffset(message.partition, message.offset)
```

**Time Complexity:** $O(1)$ plus network I/O  
**Space Complexity:** $O(1)$

---

### PushWorker.batchSend()

**Purpose:** Batch process push notifications and send via FCM/APNS.

**Parameters:**
- `messages`: array - Batch of notification messages

**Returns:** void

**Algorithm:**
```
function batchSend(messages):
    rate_limiter = getTokenBucket("push")
    circuit_breaker = getCircuitBreaker("push")
    
    if not circuit_breaker.is_request_allowed():
        return  // Skip batch
    
    // Step 1: Group messages by platform
    ios_messages = []
    android_messages = []
    
    for each message in messages:
        device_tokens = getDeviceTokens(message.user_id)
        for each token in device_tokens:
            if token.platform == "ios":
                ios_messages.append({
                    token: token.value,
                    notification: message.content
                })
            else if token.platform == "android":
                android_messages.append({
                    token: token.value,
                    notification: message.content
                })
    
    // Step 2: Send iOS batch via APNS
    if not ios_messages.isEmpty():
        if rate_limiter.consume(ios_messages.length):
            try:
                results = apnsAPI.sendBatch(ios_messages)
                processResults(results, ios_messages, "ios")
            catch APIError as e:
                circuit_breaker.record_failure()
    
    // Step 3: Send Android batch via FCM
    if not android_messages.isEmpty():
        if rate_limiter.consume(android_messages.length):
            try:
                results = fcmAPI.sendBatch(android_messages)
                processResults(results, android_messages, "android")
                circuit_breaker.record_success()
            catch APIError as e:
                circuit_breaker.record_failure()
    
    // Step 4: Commit offsets for all messages
    for each message in messages:
        commitOffset(message.partition, message.offset)
    
    emitMetrics("push.batch_sent", messages.length)
```

**Time Complexity:** $O(n)$ where $n$ = batch size  
**Space Complexity:** $O(n)$

---

## 6. WebSocket Connection Management

### WebSocketManager.registerConnection()

**Purpose:** Register a new WebSocket connection for a user.

**Parameters:**
- `user_id`: int - User identifier
- `connection`: object - WebSocket connection object
- `server_id`: string - ID of WebSocket server handling this connection

**Returns:** void

**Algorithm:**
```
function registerConnection(user_id, connection, server_id):
    // Store in Redis for distributed tracking
    redis_key = "connection:user:" + user_id
    connection_data = {
        server_id: server_id,
        connection_id: connection.id,
        connected_at: current_time()
    }
    
    // Set with expiry (heartbeat will refresh)
    redis.setex(redis_key, ttl=300, serialize(connection_data))  // 5 minutes
    
    // Store in local map for this server
    local_connections[user_id] = connection
    
    // Subscribe to Redis Pub/Sub for this server
    redis.subscribe("server:" + server_id + ":notifications", handleNotificationMessage)
    
    log("WebSocket registered", user_id=user_id, server=server_id)
```

**Time Complexity:** $O(1)$  
**Space Complexity:** $O(1)$

---

### WebSocketManager.sendNotification()

**Purpose:** Send notification to user via WebSocket.

**Parameters:**
- `user_id`: int - Target user
- `notification`: object - Notification content

**Returns:** boolean - True if sent, False if user not connected

**Algorithm:**
```
function sendNotification(user_id, notification):
    // Step 1: Lookup which server has user's connection
    redis_key = "connection:user:" + user_id
    connection_data = redis.get(redis_key)
    
    if connection_data == null:
        // User not connected, store for later retrieval
        storeOfflineNotification(user_id, notification)
        return False
    
    // Step 2: Check if connection is on this server
    if connection_data.server_id == this_server_id:
        // Send directly
        connection = local_connections[user_id]
        if connection and connection.is_open():
            try:
                connection.send(serialize(notification))
                return True
            catch ConnectionError:
                handleDisconnect(user_id)
                return False
        else:
            handleDisconnect(user_id)
            return False
    else:
        // Connection is on different server, use Redis Pub/Sub
        message = {
            user_id: user_id,
            notification: notification
        }
        redis.publish("server:" + connection_data.server_id + ":notifications", serialize(message))
        return True
```

**Time Complexity:** $O(1)$  
**Space Complexity:** $O(1)$

---

### WebSocketManager.handleDisconnect()

**Purpose:** Clean up when WebSocket connection is lost.

**Parameters:**
- `user_id`: int - Disconnected user

**Returns:** void

**Algorithm:**
```
function handleDisconnect(user_id):
    // Remove from Redis
    redis_key = "connection:user:" + user_id
    redis.delete(redis_key)
    
    // Remove from local map
    if user_id in local_connections:
        local_connections.remove(user_id)
    
    log("WebSocket disconnected", user_id=user_id)
    
    emitMetrics("websocket.disconnected", 1)
```

**Time Complexity:** $O(1)$  
**Space Complexity:** $O(1)$

---

## 7. Cache Operations

### fetchUserPreferences()

**Purpose:** Fetch user notification preferences from cache or database.

**Parameters:**
- `user_id`: int - User identifier

**Returns:** object - User preferences

**Algorithm:**
```
function fetchUserPreferences(user_id):
    // Step 1: Try cache first
    cache_key = "user:prefs:" + user_id
    cached_prefs = redis.get(cache_key)
    
    if cached_prefs != null:
        emitMetrics("cache.hit", 1, tags={type: "user_prefs"})
        return deserialize(cached_prefs)
    
    // Step 2: Cache miss, fetch from database
    emitMetrics("cache.miss", 1, tags={type: "user_prefs"})
    
    db_prefs = db.query("""
        SELECT user_id, email_enabled, sms_enabled, push_enabled, web_enabled,
               frequency_limit, quiet_hours_start, quiet_hours_end
        FROM user_notification_preferences
        WHERE user_id = ?
    """, user_id)
    
    if db_prefs == null:
        // User not found, return defaults
        db_prefs = getDefaultPreferences()
    
    // Step 3: Store in cache
    redis.setex(cache_key, ttl=300, serialize(db_prefs))  // 5 minutes TTL
    
    return db_prefs
```

**Time Complexity:** $O(1)$ cache hit, $O(log n)$ cache miss (database B-tree lookup)  
**Space Complexity:** $O(1)$

---

### invalidateCache()

**Purpose:** Invalidate cache when user updates preferences.

**Parameters:**
- `user_id`: int - User whose cache to invalidate

**Returns:** void

**Algorithm:**
```
function invalidateCache(user_id):
    cache_key = "user:prefs:" + user_id
    redis.delete(cache_key)
    
    // Optionally publish invalidation event to other servers
    redis.publish("cache:invalidate", serialize({
        key: cache_key,
        user_id: user_id
    }))
    
    log("Cache invalidated", user_id=user_id)
```

**Time Complexity:** $O(1)$  
**Space Complexity:** $O(1)$

---

## 8. Retry Logic

### retryWithBackoff()

**Purpose:** Retry a function with exponential backoff on failure.

**Parameters:**
- `function`: callable - Function to retry
- `args`: object - Arguments to pass to function
- `max_retries`: int - Maximum retry attempts
- `backoff_factor`: float - Multiplier for backoff delay

**Returns:** result of function or throws exception

**Algorithm:**
```
function retryWithBackoff(function, args, max_retries=3, backoff_factor=2):
    attempt = 0
    
    while attempt < max_retries:
        try:
            result = function(args)
            return result  // Success
        catch TransientError as e:
            attempt += 1
            if attempt >= max_retries:
                throw e  // Max retries exceeded
            
            // Calculate backoff delay: 1s, 2s, 4s, ...
            delay = backoff_factor ** attempt
            log("Retry attempt " + attempt + " after " + delay + "s", error=e.message)
            sleep(delay)
        catch PermanentError as e:
            // Don't retry permanent errors (4xx)
            throw e
    
    throw MaxRetriesExceeded("Failed after " + max_retries + " attempts")
```

**Time Complexity:** $O(r)$ where $r$ = max_retries  
**Space Complexity:** $O(1)$

**Example Usage:**
```
result = retryWithBackoff(
    function=sendGridAPI.sendEmail,
    args={to: "user@example.com", subject: "Test", body: "Hello"},
    max_retries=3,
    backoff_factor=2
)
// Retries: 0s → 1s → 2s → 4s before giving up
```

---

## 9. Dead Letter Queue

### moveToDLQ()

**Purpose:** Move failed message to Dead Letter Queue for investigation.

**Parameters:**
- `message`: object - Original Kafka message
- `error`: string - Error message
- `channel`: string - Channel that failed (email, sms, push, web)

**Returns:** void

**Algorithm:**
```
function moveToDLQ(message, error, channel):
    dlq_message = {
        original_message: message,
        error: error,
        channel: channel,
        failed_at: current_time(),
        retry_count: message.retry_count || 0,
        original_topic: message.topic,
        original_partition: message.partition,
        original_offset: message.offset
    }
    
    // Publish to DLQ topic
    dlq_topic = "dlq-" + channel
    publishToKafka(topic=dlq_topic, message=dlq_message)
    
    // Log to database for auditing
    logNotificationStatus(
        message.notification_id,
        "failed",
        channel=channel,
        error=error
    )
    
    // Emit metrics
    emitMetrics("dlq.message_added", 1, tags={channel: channel})
    
    // Check if DLQ threshold exceeded
    dlq_count = getDLQMessageCount(channel)
    if dlq_count > 10000:
        alert("DLQ threshold exceeded", 
              severity="HIGH", 
              channel=channel, 
              count=dlq_count)
    
    log("Message moved to DLQ", notification_id=message.notification_id, channel=channel)
```

**Time Complexity:** $O(1)$  
**Space Complexity:** $O(1)$

---

## 10. Metrics & Logging

### logNotificationStatus()

**Purpose:** Log notification delivery status to Cassandra.

**Parameters:**
- `notification_id`: string - Notification UUID
- `status`: string - Status (created, sent, delivered, failed, opened)
- `channel`: string - Delivery channel
- `metadata`: map - Additional metadata

**Returns:** void

**Algorithm:**
```
function logNotificationStatus(notification_id, status, channel, metadata={}):
    // Async write to Cassandra (don't block)
    cassandra.execute_async("""
        INSERT INTO notification_logs 
        (notification_id, user_id, channel, status, created_at, delivered_at, error_message, metadata)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    """, [
        notification_id,
        metadata.get("user_id"),
        channel,
        status,
        current_time(),
        metadata.get("delivered_at"),
        metadata.get("error"),
        metadata
    ])
    
    // Also update status lookup table
    cassandra.execute_async("""
        UPDATE notification_status
        SET status = ?, updated_at = ?
        WHERE notification_id = ?
    """, [status, current_time(), notification_id])
```

**Time Complexity:** $O(1)$ (async operation)  
**Space Complexity:** $O(1)$

---

### emitMetrics()

**Purpose:** Emit metrics to Prometheus for monitoring.

**Parameters:**
- `metric_name`: string - Metric identifier
- `value`: float - Metric value
- `tags`: map - Additional tags for grouping

**Returns:** void

**Algorithm:**
```
function emitMetrics(metric_name, value, tags={}):
    // Construct metric with tags
    full_metric_name = metric_name
    tag_string = ""
    
    for each (key, val) in tags:
        tag_string += key + "=" + val + ","
    
    if tag_string != "":
        full_metric_name += "{" + tag_string.trim_end(",") + "}"
    
    // Send to Prometheus pushgateway
    prometheus.push(full_metric_name, value)
    
    // Also log for debugging
    if value > ANOMALY_THRESHOLD:
        log("Metric anomaly detected", metric=metric_name, value=value, tags=tags)
```

**Time Complexity:** $O(1)$  
**Space Complexity:** $O(1)$

---

## Summary

This document provides detailed pseudocode implementations for 20+ key functions in the Notification Service:

| Category | Functions | Key Algorithms |
|----------|-----------|----------------|
| **Core Service** | 3 functions | Request processing, channel selection, template enrichment |
| **Idempotency** | 2 functions | Duplicate detection, key storage |
| **Rate Limiting** | 2 functions | Token bucket (local & distributed) |
| **Circuit Breaker** | 3 functions | State management, success/failure recording |
| **Channel Workers** | 3 functions | Email, SMS, push processing with retries |
| **WebSocket** | 3 functions | Connection management, real-time delivery |
| **Caching** | 2 functions | Cache-aside pattern, invalidation |
| **Retry** | 1 function | Exponential backoff |
| **DLQ** | 1 function | Failed message handling |
| **Metrics** | 2 functions | Logging, monitoring |

**Key Complexity Analysis:**
- Most operations are $O(1)$ constant time
- Cache misses require $O(log n)$ database lookups
- Batch operations are $O(n)$ where $n$ is batch size
- All algorithms use $O(1)$ or $O(n)$ space

**Next Steps:**
- Review **[3.2.2-design-notification-service.md](./3.2.2-design-notification-service.md)** for complete system design
- Study **[this-over-that.md](./this-over-that.md)** for design decision rationale
- Examine **[hld-diagram.md](./hld-diagram.md)** and **[sequence-diagrams.md](./sequence-diagrams.md)** for visual flows
