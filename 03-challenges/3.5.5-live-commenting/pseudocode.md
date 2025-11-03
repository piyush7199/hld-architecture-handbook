# Live Commenting - Pseudocode Implementations

This document contains detailed algorithm implementations for the Live Commenting system. The main challenge
document references these functions.

---

## Table of Contents

1. [Connection Management](#1-connection-management)
2. [Comment Ingestion](#2-comment-ingestion)
3. [Broadcast and Fanout](#3-broadcast-and-fanout)
4. [Moderation](#4-moderation)
5. [Adaptive Throttling](#5-adaptive-throttling)
6. [Reaction Aggregation](#6-reaction-aggregation)
7. [Rate Limiting](#7-rate-limiting)
8. [Cache Management](#8-cache-management)
9. [Circuit Breaker](#9-circuit-breaker)
10. [Connection Registry](#10-connection-registry)

---

## 1. Connection Management

### establish_websocket_connection()

**Purpose:** Establishes a WebSocket connection, authenticates user, and registers connection in Redis.

**Parameters:**

- `client_request` (WebSocketRequest): WebSocket handshake request
- `jwt_token` (string): JWT authentication token

**Returns:**

- `connection_id` (string): Unique connection identifier
- `ws_server_id` (string): WebSocket server identifier

**Algorithm:**

```
function establish_websocket_connection(client_request, jwt_token):
  // Step 1: Validate JWT token
  user_data = validate_jwt_token(jwt_token)
  if user_data is None:
    return error("Invalid token")
  
  user_id = user_data.user_id
  stream_id = extract_stream_id(client_request.path)
  
  // Step 2: Accept WebSocket connection
  ws_connection = accept_websocket_handshake(client_request)
  
  // Step 3: Register connection in Redis
  ws_server_id = get_current_server_id()  // "ws-server-42"
  connection_id = generate_connection_id(user_id, stream_id)
  
  redis.HSET(
    key: f"conn:user:{user_id}",
    fields: {
      "stream_id": stream_id,
      "ws_server": ws_server_id,
      "connected_at": current_timestamp(),
      "connection_id": connection_id
    },
    ttl: 7200  // 2 hours
  )
  
  // Step 4: Subscribe to Redis Pub/Sub channel
  redis_channel = f"stream:{stream_id}:comments"
  redis.SUBSCRIBE(redis_channel)
  
  // Step 5: Setup heartbeat
  start_heartbeat_timer(ws_connection, interval=30 seconds)
  
  // Step 6: Return connection info
  return {
    "connection_id": connection_id,
    "ws_server_id": ws_server_id,
    "status": "connected"
  }
```

**Time Complexity:** O(1)

**Example Usage:**

```
result = establish_websocket_connection(
  client_request=ws_request,
  jwt_token="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
)
// Returns: {"connection_id": "conn_123_789", "ws_server_id": "ws-42"}
```

---

### handle_heartbeat()

**Purpose:** Processes WebSocket heartbeat (ping/pong) to keep connections alive and detect dead connections.

**Parameters:**

- `connection_id` (string): Connection identifier
- `heartbeat_type` (string): "ping" or "pong"

**Returns:**

- `is_alive` (boolean): Whether connection is still alive

**Algorithm:**

```
function handle_heartbeat(connection_id, heartbeat_type):
  if heartbeat_type == "ping":
    // Server sends ping, expect pong
    send_pong_frame(connection_id)
    return true
  
  else if heartbeat_type == "pong":
    // Client responded to ping
    last_heartbeat = current_timestamp()
    
    // Update last heartbeat in Redis
    user_id = extract_user_id_from_connection(connection_id)
    redis.HSET(
      key: f"conn:user:{user_id}",
      field: "last_heartbeat",
      value: last_heartbeat
    )
    
    return true
  
  // Check for dead connections (no pong in 60 seconds)
  last_heartbeat = redis.HGET(f"conn:user:{user_id}", "last_heartbeat")
  if (current_timestamp() - last_heartbeat) > 60:
    return false  // Connection is dead
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
is_alive = handle_heartbeat("conn_123_789", "pong")
if not is_alive:
  cleanup_connection("conn_123_789")
```

---

### cleanup_connection()

**Purpose:** Removes connection from registry and unsubscribes from channels when connection closes.

**Parameters:**

- `connection_id` (string): Connection identifier
- `user_id` (int): User identifier

**Returns:**

- `success` (boolean): Whether cleanup was successful

**Algorithm:**

```
function cleanup_connection(connection_id, user_id):
  // Remove from connection registry
  redis.HDEL(f"conn:user:{user_id}")
  
  // Unsubscribe from Redis Pub/Sub channels
  stream_id = extract_stream_id_from_connection(connection_id)
  redis_channel = f"stream:{stream_id}:comments"
  redis.UNSUBSCRIBE(redis_channel)
  
  // Decrement viewer count
  redis.PFREM(f"stream:viewers:{stream_id}", f"user_{user_id}")
  
  // Close WebSocket connection
  close_websocket(connection_id)
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
cleanup_connection("conn_123_789", user_id=123)
```

---

## 2. Comment Ingestion

### submit_comment()

**Purpose:** Accepts comment from client, validates, and publishes to Kafka.

**Parameters:**

- `user_id` (int): User submitting comment
- `stream_id` (int): Stream identifier
- `message` (string): Comment text
- `client_ip` (string): Client IP address

**Returns:**

- `comment_id` (string): Generated comment identifier
- `status` (string): "published" or "rate_limited"

**Algorithm:**

```
function submit_comment(user_id, stream_id, message, client_ip):
  // Step 1: Rate limiting check
  rate_limit_result = check_rate_limits(user_id, stream_id, client_ip)
  if not rate_limit_result.allowed:
    return error(rate_limit_result.message)
  
  // Step 2: Validate comment
  if len(message) == 0 or len(message) > 500:
    return error("Comment must be 1-500 characters")
  
  if contains_invalid_characters(message):
    return error("Comment contains invalid characters")
  
  // Step 3: Generate comment ID
  comment_id = generate_comment_id()  // UUID or Snowflake ID
  
  // Step 4: Create comment object
  comment = {
    "comment_id": comment_id,
    "user_id": user_id,
    "stream_id": stream_id,
    "message": message,
    "created_at": current_timestamp(),
    "status": "pending_moderation"
  }
  
  // Step 5: Publish to Kafka (partition by stream_id)
  kafka_partition = hash(stream_id) % 100
  kafka.PUBLISH(
    topic: "comments",
    partition: kafka_partition,
    message: comment
  )
  
  // Step 6: Return success (optimistic UI)
  return {
    "comment_id": comment_id,
    "status": "published",
    "message": "Comment submitted successfully"
  }
```

**Time Complexity:** O(1)

**Example Usage:**

```
result = submit_comment(
  user_id=123,
  stream_id=789,
  message="Great stream!",
  client_ip="1.2.3.4"
)
// Returns: {"comment_id": "cmt_abc123", "status": "published"}
```

---

### generate_comment_id()

**Purpose:** Generates unique comment identifier using Snowflake algorithm or UUID.

**Parameters:**

- None

**Returns:**

- `comment_id` (string): Unique comment identifier

**Algorithm:**

```
function generate_comment_id():
  // Option 1: Snowflake ID (64-bit integer)
  timestamp = current_timestamp_ms() - epoch_start
  machine_id = get_machine_id()  // 10 bits
  sequence = atomic_increment(sequence_counter) % 4096  // 12 bits
  
  snowflake_id = (timestamp << 22) | (machine_id << 12) | sequence
  return f"cmt_{snowflake_id}"
  
  // Option 2: UUID (128-bit)
  // return str(uuid.uuid4())
```

**Time Complexity:** O(1)

**Example Usage:**

```
comment_id = generate_comment_id()
// Returns: "cmt_1234567890123456789"
```

---

## 3. Broadcast and Fanout

### broadcast_comment()

**Purpose:** Broadcasts comment to all viewers of a stream using two-tier fanout.

**Parameters:**

- `comment` (dict): Comment object with all fields
- `stream_id` (int): Stream identifier

**Returns:**

- `fanout_count` (int): Number of servers that received the message

**Algorithm:**

```
function broadcast_comment(comment, stream_id):
  // Step 1: Serialize comment (MessagePack)
  serialized_comment = msgpack.pack(comment)
  
  // Step 2: Calculate Redis shard (for load distribution)
  num_shards = 100
  base_channel = f"stream:{stream_id}:comments"
  
  // Step 3: Publish to all Redis Pub/Sub shards (parallel)
  fanout_count = 0
  for shard_id in range(num_shards):
    channel = f"{base_channel}:shard{shard_id}"
    redis.PUBLISH(channel, serialized_comment)
    fanout_count += 1
  
  // Step 4: Return fanout count
  return fanout_count
```

**Time Complexity:** O(k) where k = number of shards (typically 100)

**Example Usage:**

```
fanout_count = broadcast_comment(
  comment={"comment_id": "cmt_123", "message": "Great!"},
  stream_id=789
)
// Returns: 100 (published to 100 shards)
```

---

### fanout_to_websocket_servers()

**Purpose:** Receives comment from Redis Pub/Sub and pushes to all connected clients on this WebSocket server.

**Parameters:**

- `comment` (dict): Comment object
- `server_id` (string): WebSocket server identifier

**Returns:**

- `pushed_count` (int): Number of clients that received the message

**Algorithm:**

```
function fanout_to_websocket_servers(comment, server_id):
  // Step 1: Get all connections for this server
  connections = get_server_connections(server_id)
  
  // Step 2: Serialize comment (MessagePack)
  serialized_comment = msgpack.pack(comment)
  
  // Step 3: Batch push to all connections
  batch_size = 100
  pushed_count = 0
  
  for i in range(0, len(connections), batch_size):
    batch = connections[i:i+batch_size]
    
    // Send to batch in parallel
    for connection in batch:
      try:
        connection.send(serialized_comment)
        pushed_count += 1
      except ConnectionError:
        // Connection closed, remove from registry
        cleanup_connection(connection.id, connection.user_id)
  
  return pushed_count
```

**Time Complexity:** O(n) where n = number of connections

**Example Usage:**

```
pushed_count = fanout_to_websocket_servers(
  comment={"comment_id": "cmt_123", "message": "Great!"},
  server_id="ws-server-42"
)
// Returns: 1000000 (1M connections on this server)
```

---

## 4. Moderation

### moderate_comment()

**Purpose:** Processes comment through moderation pipeline (rule-based + ML model) and returns moderation decision.

**Parameters:**

- `comment` (dict): Comment object with message field

**Returns:**

- `decision` (string): "ACCEPT", "REJECT", "SHADOW_BAN", or "FLAG"
- `reason` (string): Reason for decision
- `confidence` (float): Confidence score (0.0 to 1.0)

**Algorithm:**

```
function moderate_comment(comment):
  message = comment["message"]
  user_id = comment["user_id"]
  
  // Stage 1: Rule-based filter (fast, 5ms)
  blacklisted_words = ["spam", "hate", "offensive"]
  if contains_blacklisted_words(message, blacklisted_words):
    return {
      "decision": "REJECT",
      "reason": "blacklisted_word",
      "confidence": 1.0
    }
  
  // Stage 2: ML model (slower, 50ms)
  toxicity_score = ml_model.predict(message)
  
  // Stage 3: User reputation check
  user_reputation = get_user_reputation(user_id)
  threshold = 0.9  // Default threshold
  
  if user_reputation.has_violations:
    threshold = 0.7  // Lower threshold for repeat offenders
  
  // Decision logic
  if toxicity_score > threshold:
    if user_reputation.violation_count >= 3:
      return {
        "decision": "SHADOW_BAN",
        "reason": "repeat_offender",
        "confidence": toxicity_score
      }
    else:
      return {
        "decision": "REJECT",
        "reason": "high_toxicity",
        "confidence": toxicity_score
      }
  
  else if toxicity_score > 0.5:
    return {
      "decision": "FLAG",
      "reason": "moderate_toxicity",
      "confidence": toxicity_score
    }
  
  else:
    return {
      "decision": "ACCEPT",
      "reason": "clean",
      "confidence": 1.0 - toxicity_score
    }
```

**Time Complexity:** O(1) for rule-based, O(n) for ML model where n = message length

**Example Usage:**

```
result = moderate_comment({
  "comment_id": "cmt_123",
  "user_id": 123,
  "message": "Great stream!"
})
// Returns: {"decision": "ACCEPT", "reason": "clean", "confidence": 0.95}
```

---

### process_moderation_decision()

**Purpose:** Processes moderation decision and takes appropriate action (delete, shadow ban, or flag).

**Parameters:**

- `comment_id` (string): Comment identifier
- `decision` (string): Moderation decision
- `reason` (string): Reason for decision

**Returns:**

- `action_taken` (string): "deleted", "shadow_banned", "flagged", or "none"

**Algorithm:**

```
function process_moderation_decision(comment_id, decision, reason):
  if decision == "REJECT":
    // Publish delete event
    delete_event = {
      "type": "comment.deleted",
      "comment_id": comment_id,
      "reason": reason,
      "timestamp": current_timestamp()
    }
    
    kafka.PUBLISH(topic: "comment.events", message: delete_event)
    
    // Mark as deleted in Cassandra
    cassandra.UPDATE(
      table: "comments",
      set: {"is_deleted": true},
      where: {"comment_id": comment_id}
    )
    
    return "deleted"
  
  else if decision == "SHADOW_BAN":
    // Don't broadcast, but store for analytics
    cassandra.UPDATE(
      table: "comments",
      set: {"is_deleted": true, "shadow_banned": true},
      where: {"comment_id": comment_id}
    )
    
    return "shadow_banned"
  
  else if decision == "FLAG":
    // Flag for manual review, but keep visible
    cassandra.UPDATE(
      table: "comments",
      set: {"flagged": true, "flag_reason": reason},
      where: {"comment_id": comment_id}
    )
    
    // Notify moderators
    notify_moderators(comment_id, reason)
    
    return "flagged"
  
  else:
    return "none"
```

**Time Complexity:** O(1)

**Example Usage:**

```
action = process_moderation_decision(
  comment_id="cmt_123",
  decision="REJECT",
  reason="high_toxicity"
)
// Returns: "deleted"
```

---

## 5. Adaptive Throttling

### adaptive_throttle()

**Purpose:** Determines whether to broadcast comment based on current comment rate and adaptive throttling rules.

**Parameters:**

- `stream_id` (int): Stream identifier
- `comment` (dict): Comment object

**Returns:**

- `should_broadcast` (boolean): Whether to broadcast comment
- `sample_rate` (float): Current sample rate (0.0 to 1.0)

**Algorithm:**

```
function adaptive_throttle(stream_id, comment):
  // Step 1: Get current comment rate
  current_rate = get_comment_rate(stream_id)  // Comments per second
  
  // Step 2: Determine sample rate based on current rate
  if current_rate < 5000:
    sample_rate = 1.0  // 100% broadcast (normal traffic)
  else if current_rate < 20000:
    sample_rate = 0.5  // 50% broadcast (moderate traffic)
  else if current_rate < 50000:
    sample_rate = 0.1  // 10% broadcast (high traffic)
  else:
    sample_rate = 0.05  // 5% broadcast (extreme traffic)
  
  // Step 3: Random sampling
  random_value = random()  // 0.0 to 1.0
  
  if random_value < sample_rate:
    should_broadcast = true
  else:
    should_broadcast = false
  
  // Step 4: Always store comment (even if not broadcast)
  store_comment(comment)
  
  // Step 5: Notify users if sampling is active
  if sample_rate < 1.0:
    notify_users_sampling_active(stream_id, sample_rate)
  
  return {
    "should_broadcast": should_broadcast,
    "sample_rate": sample_rate
  }
```

**Time Complexity:** O(1)

**Example Usage:**

```
result = adaptive_throttle(
  stream_id=789,
  comment={"comment_id": "cmt_123", "message": "Great!"}
)
// Returns: {"should_broadcast": true, "sample_rate": 0.1}
```

---

### get_comment_rate()

**Purpose:** Gets current comment rate (comments per second) for a stream.

**Parameters:**

- `stream_id` (int): Stream identifier

**Returns:**

- `rate` (float): Comments per second

**Algorithm:**

```
function get_comment_rate(stream_id):
  // Use sliding window counter (last 10 seconds)
  window_size = 10  // seconds
  
  current_time = current_timestamp()
  window_start = current_time - window_size
  
  // Count comments in window using Redis sorted set
  key = f"rate:stream:{stream_id}"
  
  // Remove old entries (outside window)
  redis.ZREMRANGEBYSCORE(key, 0, window_start)
  
  // Count entries in window
  count = redis.ZCOUNT(key, window_start, current_time)
  
  // Calculate rate
  rate = count / window_size
  
  // Add current comment to window
  redis.ZADD(key, score=current_time, member=generate_id())
  
  return rate
```

**Time Complexity:** O(log n + m) where n = sorted set size, m = removed entries

**Example Usage:**

```
rate = get_comment_rate(stream_id=789)
// Returns: 25000.0 (25k comments/sec)
```

---

## 6. Reaction Aggregation

### aggregate_reactions()

**Purpose:** Aggregates emoji reactions locally before broadcasting to reduce network traffic.

**Parameters:**

- `reaction` (dict): Reaction object with type and user_id
- `stream_id` (int): Stream identifier

**Returns:**

- `aggregated_count` (int): Total count for this reaction type after aggregation

**Algorithm:**

```
function aggregate_reactions(reaction, stream_id):
  reaction_type = reaction["type"]  // "fire", "heart", etc.
  user_id = reaction["user_id"]
  
  // Step 1: Get local buffer (server-specific)
  buffer_key = f"reactions:buffer:{stream_id}:{reaction_type}"
  local_buffer = get_local_buffer(buffer_key)
  
  // Step 2: Increment local buffer
  local_buffer[reaction_type] = local_buffer.get(reaction_type, 0) + 1
  
  // Step 3: Check if flush interval elapsed (500ms)
  last_flush_time = local_buffer.get("last_flush", 0)
  current_time = current_timestamp_ms()
  
  if (current_time - last_flush_time) >= 500:
    // Flush to Redis
    count = local_buffer[reaction_type]
    
    redis.HINCRBY(
      key: f"reactions:stream:{stream_id}",
      field: reaction_type,
      increment: count
    )
    
    // Clear local buffer
    local_buffer[reaction_type] = 0
    local_buffer["last_flush"] = current_time
    
    // Broadcast aggregated update
    total_count = redis.HGET(f"reactions:stream:{stream_id}", reaction_type)
    broadcast_reaction_update(stream_id, reaction_type, total_count)
  
  else:
    // Store in local buffer, wait for flush
    set_local_buffer(buffer_key, local_buffer)
  
  // Step 4: Get current total
  total_count = redis.HGET(f"reactions:stream:{stream_id}", reaction_type)
  
  return total_count
```

**Time Complexity:** O(1)

**Example Usage:**

```
total = aggregate_reactions(
  reaction={"type": "fire", "user_id": 123},
  stream_id=789
)
// Returns: 15000 (total fire reactions)
```

---

### broadcast_reaction_update()

**Purpose:** Broadcasts aggregated reaction counts to all viewers.

**Parameters:**

- `stream_id` (int): Stream identifier
- `reaction_type` (string): Reaction type ("fire", "heart", etc.)
- `total_count` (int): Total count for this reaction type

**Returns:**

- `broadcasted` (boolean): Whether broadcast was successful

**Algorithm:**

```
function broadcast_reaction_update(stream_id, reaction_type, total_count):
  // Get all reaction types and counts
  all_reactions = redis.HGETALL(f"reactions:stream:{stream_id}")
  
  // Create update message
  update_message = {
    "type": "reactions_update",
    "stream_id": stream_id,
    "reactions": all_reactions,
    "timestamp": current_timestamp()
  }
  
  // Serialize (MessagePack)
  serialized = msgpack.pack(update_message)
  
  // Broadcast via Redis Pub/Sub
  channel = f"stream:{stream_id}:reactions"
  redis.PUBLISH(channel, serialized)
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
broadcast_reaction_update(
  stream_id=789,
  reaction_type="fire",
  total_count=15000
)
```

---

## 7. Rate Limiting

### check_rate_limits()

**Purpose:** Checks multi-layer rate limits (per-user, per-IP, per-stream, global) before allowing comment.

**Parameters:**

- `user_id` (int): User identifier
- `stream_id` (int): Stream identifier
- `client_ip` (string): Client IP address

**Returns:**

- `allowed` (boolean): Whether comment is allowed
- `message` (string): Error message if not allowed

**Algorithm:**

```
function check_rate_limits(user_id, stream_id, client_ip):
  current_time = current_timestamp()
  
  // Layer 1: Per-user rate limit (10 comments/min)
  user_key = f"rate:user:{user_id}"
  user_count = redis.GET(user_key)
  
  if user_count is None:
    user_count = 0
  
  if user_count >= 10:
    return {
      "allowed": false,
      "message": "Rate limit exceeded: 10 comments per minute"
    }
  
  // Layer 2: Per-IP rate limit (100 comments/min)
  ip_key = f"rate:ip:{client_ip}"
  ip_count = redis.GET(ip_key)
  
  if ip_count is None:
    ip_count = 0
  
  if ip_count >= 100:
    return {
      "allowed": false,
      "message": "Rate limit exceeded: 100 comments per minute per IP"
    }
  
  // Layer 3: Per-stream rate limit (10k comments/sec)
  stream_key = f"rate:stream:{stream_id}"
  stream_count = redis.GET(stream_key)
  
  if stream_count is None:
    stream_count = 0
  
  if stream_count >= 10000:
    return {
      "allowed": false,
      "message": "Stream rate limit exceeded: 10k comments per second"
    }
  
  // Layer 4: Global rate limit (1M comments/sec)
  global_key = "rate:global"
  global_count = redis.GET(global_key)
  
  if global_count is None:
    global_count = 0
  
  if global_count >= 1000000:
    return {
      "allowed": false,
      "message": "Global rate limit exceeded: 1M comments per second"
    }
  
  // All checks passed, increment counters
  redis.INCR(user_key)
  redis.EXPIRE(user_key, 60)  // 1 minute TTL
  
  redis.INCR(ip_key)
  redis.EXPIRE(ip_key, 60)
  
  redis.INCR(stream_key)
  redis.EXPIRE(stream_key, 1)  // 1 second TTL
  
  redis.INCR(global_key)
  redis.EXPIRE(global_key, 1)
  
  return {
    "allowed": true,
    "message": "Rate limits OK"
  }
```

**Time Complexity:** O(1)

**Example Usage:**

```
result = check_rate_limits(
  user_id=123,
  stream_id=789,
  client_ip="1.2.3.4"
)
// Returns: {"allowed": true, "message": "Rate limits OK"}
```

---

## 8. Cache Management

### get_stream_metadata()

**Purpose:** Retrieves stream metadata (title, viewer count, etc.) from cache or database.

**Parameters:**

- `stream_id` (int): Stream identifier

**Returns:**

- `metadata` (dict): Stream metadata object

**Algorithm:**

```
function get_stream_metadata(stream_id):
  // Step 1: Check Redis cache
  cache_key = f"stream:{stream_id}"
  cached_metadata = redis.HGETALL(cache_key)
  
  if cached_metadata is not None and len(cached_metadata) > 0:
    return cached_metadata
  
  // Step 2: Cache miss, fetch from PostgreSQL
  metadata = postgres.SELECT(
    table: "streams",
    where: {"stream_id": stream_id}
  )
  
  if metadata is None:
    return error("Stream not found")
  
  // Step 3: Cache in Redis (write-through)
  redis.HMSET(cache_key, metadata)
  redis.EXPIRE(cache_key, 3600)  // 1 hour TTL
  
  return metadata
```

**Time Complexity:** O(1) for cache hit, O(1) for database query

**Example Usage:**

```
metadata = get_stream_metadata(stream_id=789)
// Returns: {"stream_id": 789, "title": "Live Stream", "viewer_count": 5000000}
```

---

### invalidate_stream_cache()

**Purpose:** Invalidates stream cache when metadata is updated.

**Parameters:**

- `stream_id` (int): Stream identifier

**Returns:**

- `invalidated` (boolean): Whether invalidation was successful

**Algorithm:**

```
function invalidate_stream_cache(stream_id):
  cache_key = f"stream:{stream_id}"
  
  // Delete from Redis
  redis.DEL(cache_key)
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
invalidate_stream_cache(stream_id=789)
```

---

## 9. Circuit Breaker

### CircuitBreaker class

**Purpose:** Implements circuit breaker pattern to prevent cascading failures when downstream services fail.

**Algorithm:**

```
class CircuitBreaker:
  state: "CLOSED" | "OPEN" | "HALF_OPEN"
  failure_count: int
  failure_threshold: int = 50  // 50% failure rate
  success_threshold: int = 5  // 5 consecutive successes to close
  timeout: int = 30  // 30 seconds before half-open
  last_failure_time: timestamp
  
  function call(service_function, *args, **kwargs):
    // Check circuit state
    if self.state == "OPEN":
      if (current_timestamp() - self.last_failure_time) > self.timeout:
        self.state = "HALF_OPEN"
      else:
        return error("Circuit breaker is OPEN")
    
    // Try calling service
    try:
      result = service_function(*args, **kwargs)
      
      // Success
      if self.state == "HALF_OPEN":
        self.success_count += 1
        if self.success_count >= self.success_threshold:
          self.state = "CLOSED"
          self.failure_count = 0
          self.success_count = 0
      
      return result
      
    except Exception as e:
      // Failure
      self.failure_count += 1
      self.last_failure_time = current_timestamp()
      
      // Calculate failure rate
      total_requests = self.failure_count + self.success_count
      failure_rate = self.failure_count / total_requests
      
      if failure_rate > (self.failure_threshold / 100.0):
        self.state = "OPEN"
      
      raise e
```

**Time Complexity:** O(1)

**Example Usage:**

```
circuit_breaker = CircuitBreaker()

// Protect Redis Pub/Sub calls
try:
  result = circuit_breaker.call(
    redis.PUBLISH,
    channel="stream:789:comments",
    message=comment
  )
except CircuitBreakerOpen:
  // Fallback to Kafka direct broadcast
  fallback_broadcast(comment)
```

---

## 10. Connection Registry

### lookup_user_connection()

**Purpose:** Looks up which WebSocket server holds a specific user's connection.

**Parameters:**

- `user_id` (int): User identifier

**Returns:**

- `ws_server_id` (string): WebSocket server identifier or None
- `stream_id` (int): Stream identifier or None

**Algorithm:**

```
function lookup_user_connection(user_id):
  // Query Redis connection registry
  registry_key = f"conn:user:{user_id}"
  connection_data = redis.HGETALL(registry_key)
  
  if connection_data is None or len(connection_data) == 0:
    return {
      "ws_server_id": None,
      "stream_id": None,
      "found": false
    }
  
  return {
    "ws_server_id": connection_data["ws_server"],
    "stream_id": int(connection_data["stream_id"]),
    "connected_at": connection_data["connected_at"],
    "found": true
  }
```

**Time Complexity:** O(1)

**Example Usage:**

```
result = lookup_user_connection(user_id=123)
// Returns: {"ws_server_id": "ws-server-42", "stream_id": 789, "found": true}
```

---

### route_message_to_user()

**Purpose:** Routes a message directly to a specific user's connection (bypassing broadcast).

**Parameters:**

- `user_id` (int): Target user identifier
- `message` (dict): Message to send

**Returns:**

- `delivered` (boolean): Whether message was delivered

**Algorithm:**

```
function route_message_to_user(user_id, message):
  // Step 1: Lookup user's connection
  connection_info = lookup_user_connection(user_id)
  
  if not connection_info["found"]:
    return {
      "delivered": false,
      "reason": "User not connected"
    }
  
  ws_server_id = connection_info["ws_server_id"]
  
  // Step 2: Route message to specific server
  ws_server = get_websocket_server(ws_server_id)
  
  if ws_server is None:
    return {
      "delivered": false,
      "reason": "Server not found"
    }
  
  // Step 3: Send message directly
  serialized_message = msgpack.pack(message)
  
  success = ws_server.send_to_user(user_id, serialized_message)
  
  return {
    "delivered": success,
    "ws_server_id": ws_server_id
  }
```

**Time Complexity:** O(1)

**Example Usage:**

```
result = route_message_to_user(
  user_id=123,
  message={"type": "ban", "reason": "spam"}
)
// Returns: {"delivered": true, "ws_server_id": "ws-server-42"}
```

---

This completes the pseudocode implementations for the Live Commenting system. Each function includes detailed
algorithms, complexity analysis, and usage examples.

