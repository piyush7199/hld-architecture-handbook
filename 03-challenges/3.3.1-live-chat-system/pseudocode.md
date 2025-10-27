# Live Chat System - Pseudocode Implementations

This document contains detailed algorithm implementations for the Live Chat System. The main challenge document references these functions.

---

## Table of Contents

1. [WebSocket Connection Management](#1-websocket-connection-management)
2. [Message Send Operations](#2-message-send-operations)
3. [Message Receive Operations](#3-message-receive-operations)
4. [Sequence ID Generation](#4-sequence-id-generation)
5. [Presence Service](#5-presence-service)
6. [Group Chat Fanout](#6-group-chat-fanout)
7. [Offline Message Handling](#7-offline-message-handling)
8. [Read Receipt Operations](#8-read-receipt-operations)
9. [Message Editing](#9-message-editing)
10. [Message Deletion](#10-message-deletion)

---

## 1. WebSocket Connection Management

### handle_connection()

**Purpose:** Handle new WebSocket connection, authenticate, register, and maintain.

**Parameters:**
- websocket_connection: WebSocket - The incoming WebSocket connection
- server_id: string - This server's unique identifier

**Returns:**
- boolean - True if connection established successfully

**Algorithm:**
```
function handle_connection(websocket_connection, server_id):
    // Step 1: Perform HTTP upgrade
    try:
        upgrade_to_websocket(websocket_connection)
    catch UpgradeError as e:
        return false
    
    // Step 2: Wait for AUTH message (timeout: 10 seconds)
    auth_message = websocket_connection.receive_with_timeout(10000)
    if auth_message == null:
        websocket_connection.close(4001, "Auth timeout")
        return false
    
    // Step 3: Validate JWT token
    jwt_token = auth_message.token
    user_data = validate_jwt(jwt_token)
    if user_data == null:
        websocket_connection.close(4003, "Invalid token")
        return false
    
    user_id = user_data.user_id
    
    // Step 4: Register connection in local memory
    connection_pool[user_id] = {
        socket: websocket_connection,
        user_id: user_id,
        server_id: server_id,
        connected_at: now(),
        last_heartbeat: now()
    }
    
    // Step 5: Register in distributed registry (Redis)
    redis.set("user:" + user_id + ":ws_server", server_id, ttl=600)
    
    // Step 6: Update presence
    set_user_online(user_id)
    
    // Step 7: Fetch and deliver offline messages
    offline_messages = fetch_offline_messages(user_id)
    if offline_messages.length > 0:
        for message in offline_messages:
            websocket_connection.send(message)
        clear_offline_queue(user_id)
    
    // Step 8: Send READY message
    websocket_connection.send({
        type: "READY",
        timestamp: now(),
        server_id: server_id
    })
    
    // Step 9: Start heartbeat loop
    spawn_thread(heartbeat_loop, user_id, websocket_connection)
    
    // Step 10: Start message receive loop
    message_loop(user_id, websocket_connection)
    
    return true
```

**Time Complexity:** O(1) for connection setup, O(n) for offline messages where n is message count

**Space Complexity:** O(1) for connection state

**Example Usage:**
```
websocket_server.on_connection = function(ws):
    success = handle_connection(ws, "ws-server-42")
    if not success:
        log_error("Connection failed")
```

---

### heartbeat_loop()

**Purpose:** Maintain connection health and presence status with periodic heartbeats.

**Parameters:**
- user_id: string - User identifier
- websocket_connection: WebSocket - Active connection

**Returns:**
- void - Runs until connection closed

**Algorithm:**
```
function heartbeat_loop(user_id, websocket_connection):
    while websocket_connection.is_open():
        try:
            // Send PING frame
            websocket_connection.send_ping()
            
            // Wait for PONG (timeout: 5 seconds)
            pong_received = websocket_connection.wait_pong(5000)
            
            if not pong_received:
                // No pong, connection dead
                log_warning("Heartbeat timeout for user " + user_id)
                websocket_connection.close(4002, "Heartbeat timeout")
                break
            
            // Update presence TTL in Redis
            redis.expire("user:" + user_id + ":presence", 60)
            
            // Update local connection state
            connection_pool[user_id].last_heartbeat = now()
            
            // Sleep for 30 seconds before next heartbeat
            sleep(30000)
            
        catch ConnectionClosed:
            // Connection closed gracefully
            break
        catch Exception as e:
            log_error("Heartbeat error for user " + user_id + ": " + e)
            break
    
    // Cleanup when connection closes
    cleanup_connection(user_id)
```

**Time Complexity:** O(1) per heartbeat

**Performance:** 1 byte every 30 seconds, minimal overhead

**Example Usage:**
```
// Spawned automatically in handle_connection()
spawn_thread(heartbeat_loop, "12345", websocket)
```

---

### cleanup_connection()

**Purpose:** Clean up all resources when WebSocket connection closes.

**Parameters:**
- user_id: string - User identifier

**Returns:**
- void

**Algorithm:**
```
function cleanup_connection(user_id):
    // Step 1: Remove from local connection pool
    if user_id in connection_pool:
        delete connection_pool[user_id]
    
    // Step 2: Delete presence in Redis
    redis.delete("user:" + user_id + ":presence")
    
    // Step 3: Remove from connection registry
    redis.delete("user:" + user_id + ":ws_server")
    
    // Step 4: Unsubscribe from Pub/Sub channels
    user_chats = get_user_chats(user_id)
    for chat_id in user_chats:
        redis.unsubscribe("typing:" + chat_id)
    
    // Step 5: Log disconnection
    log_info("User " + user_id + " disconnected")
    
    // Step 6: Emit offline event (for analytics)
    emit_event("user.offline", {
        user_id: user_id,
        disconnected_at: now()
    })
```

**Time Complexity:** O(n) where n is number of chat subscriptions

**Example Usage:**
```
// Called automatically when connection closes
cleanup_connection("12345")
```

---

## 2. Message Send Operations

### send_message()

**Purpose:** Handle message send from user, validate, sequence, and publish to Kafka.

**Parameters:**
- user_id: string - Sender's user ID
- chat_id: string - Chat identifier
- content: string - Message content
- message_type: string - "text", "image", "video", etc.
- request_id: string - Client-generated unique ID for deduplication

**Returns:**
- object - {success: boolean, seq_id: number, error: string}

**Algorithm:**
```
function send_message(user_id, chat_id, content, message_type, request_id):
    // Step 1: Deduplication check
    dedup_key = "msg_dedup:" + user_id + ":" + request_id
    if redis.exists(dedup_key):
        // Message already processed (client retry)
        cached_response = redis.get(dedup_key)
        return json_parse(cached_response)
    
    // Step 2: Validate inputs
    validation_result = validate_message(user_id, chat_id, content, message_type)
    if not validation_result.valid:
        return {success: false, error: validation_result.error}
    
    // Step 3: Rate limit check
    rate_limit_key = "rate_limit:" + user_id
    current_count = redis.incr(rate_limit_key)
    if current_count == 1:
        redis.expire(rate_limit_key, 60)  // 1-minute window
    
    if current_count > 100:  // Max 100 messages per minute
        return {success: false, error: "Rate limit exceeded"}
    
    // Step 4: Request sequence ID
    seq_id = generate_sequence_id()
    
    // Step 5: Build message object
    message = {
        seq_id: seq_id,
        chat_id: chat_id,
        sender_id: user_id,
        content: content,
        message_type: message_type,
        timestamp: now(),
        request_id: request_id
    }
    
    // Step 6: Publish to Kafka
    partition = hash(chat_id) % KAFKA_PARTITION_COUNT
    kafka_result = kafka.publish("messages", message, partition)
    
    if not kafka_result.success:
        return {success: false, error: "Failed to publish"}
    
    // Step 7: Cache deduplication response
    response = {success: true, seq_id: seq_id}
    redis.setex(dedup_key, 300, json_stringify(response))  // 5-minute cache
    
    // Step 8: Return success
    return response
```

**Time Complexity:** O(1) for all operations

**Space Complexity:** O(1)

**Example Usage:**
```
result = send_message(
    user_id="12345",
    chat_id="abc-123",
    content="Hello!",
    message_type="text",
    request_id="client-uuid-xyz"
)

if result.success:
    send_ack_to_client(result.seq_id)
```

---

### validate_message()

**Purpose:** Validate message content and permissions.

**Parameters:**
- user_id: string - Sender's user ID
- chat_id: string - Chat identifier
- content: string - Message content
- message_type: string - Type of message

**Returns:**
- object - {valid: boolean, error: string}

**Algorithm:**
```
function validate_message(user_id, chat_id, content, message_type):
    // Step 1: Check if user is member of chat
    is_member = redis.sismember("chat:" + chat_id + ":members", user_id)
    if not is_member:
        return {valid: false, error: "User not a member of chat"}
    
    // Step 2: Check if user is blocked/banned
    is_banned = redis.sismember("chat:" + chat_id + ":banned", user_id)
    if is_banned:
        return {valid: false, error: "User is banned from chat"}
    
    // Step 3: Validate content length
    if message_type == "text":
        if content.length == 0:
            return {valid: false, error: "Empty message"}
        if content.length > 5000:  // Max 5000 characters
            return {valid: false, error: "Message too long"}
    
    // Step 4: Check for spam patterns
    if contains_spam(content):
        return {valid: false, error: "Spam detected"}
    
    // Step 5: Content moderation (profanity filter)
    if contains_profanity(content):
        // Option 1: Reject
        // return {valid: false, error: "Inappropriate content"}
        
        // Option 2: Flag for review (preferred)
        flag_for_moderation(user_id, chat_id, content)
    
    // All checks passed
    return {valid: true}
```

**Time Complexity:** O(n) where n is content length (for spam/profanity check)

**Example Usage:**
```
validation = validate_message("12345", "abc-123", "Hello!", "text")
if not validation.valid:
    return_error(validation.error)
```

---

## 3. Message Receive Operations

### deliver_message()

**Purpose:** Deliver message to recipient, handling both online and offline cases.

**Parameters:**
- message: object - Full message object from Kafka
- recipient_id: string - Recipient's user ID

**Returns:**
- object - {delivered: boolean, method: string, latency_ms: number}

**Algorithm:**
```
function deliver_message(message, recipient_id):
    start_time = now()
    
    // Step 1: Check if recipient is online
    presence = redis.get("user:" + recipient_id + ":presence")
    
    if presence == "online":
        // Online path: Push via WebSocket
        result = deliver_to_online_user(message, recipient_id)
        
        if result.success:
            // Update delivery status
            update_delivery_status(message.seq_id, recipient_id, "delivered")
            
            latency_ms = (now() - start_time)
            return {
                delivered: true,
                method: "websocket",
                latency_ms: latency_ms
            }
        else:
            // Delivery failed, fallback to offline
            queue_offline_message(message, recipient_id)
            latency_ms = (now() - start_time)
            return {
                delivered: false,
                method: "queued_after_failure",
                latency_ms: latency_ms
            }
    else:
        // Offline path: Queue message + push notification
        queue_offline_message(message, recipient_id)
        send_push_notification(message, recipient_id)
        
        latency_ms = (now() - start_time)
        return {
            delivered: false,
            method: "offline_queue",
            latency_ms: latency_ms
        }
```

**Time Complexity:** O(1) for online, O(1) for offline

**Average Latency:** 50ms (online), 10ms (offline queue)

**Example Usage:**
```
// Delivery worker consumes from Kafka
message = kafka.consume("messages")
for recipient_id in message.recipients:
    result = deliver_message(message, recipient_id)
    log_delivery_metrics(result)
```

---

### deliver_to_online_user()

**Purpose:** Push message to online user's WebSocket connection.

**Parameters:**
- message: object - Full message object
- recipient_id: string - Recipient's user ID

**Returns:**
- object - {success: boolean, error: string}

**Algorithm:**
```
function deliver_to_online_user(message, recipient_id):
    // Step 1: Find which WebSocket server has the connection
    ws_server_id = redis.get("user:" + recipient_id + ":ws_server")
    
    if ws_server_id == null:
        return {success: false, error: "Connection not found"}
    
    // Step 2: Check if it's this server
    if ws_server_id == THIS_SERVER_ID:
        // Local delivery
        if recipient_id in connection_pool:
            connection = connection_pool[recipient_id]
            try:
                connection.socket.send({
                    type: "MESSAGE",
                    data: message
                })
                return {success: true}
            catch ConnectionClosed:
                return {success: false, error: "Connection closed"}
        else:
            return {success: false, error: "Connection not in pool"}
    else:
        // Remote delivery via gRPC
        try:
            grpc_client = get_grpc_client(ws_server_id)
            response = grpc_client.DeliverMessage({
                user_id: recipient_id,
                message: message
            })
            return {success: response.delivered}
        catch RPCError as e:
            return {success: false, error: "RPC failed: " + e}
```

**Time Complexity:** O(1)

**Latency:** <10ms local, ~20ms remote (gRPC)

**Example Usage:**
```
result = deliver_to_online_user(message, "67890")
if result.success:
    metrics.increment("messages.delivered.online")
```

---

## 4. Sequence ID Generation

### generate_sequence_id()

**Purpose:** Generate globally unique, time-ordered 64-bit Snowflake ID.

**Parameters:**
- None (uses server state)

**Returns:**
- number - 64-bit sequence ID

**Algorithm:**
```
// Global state
last_timestamp = 0
sequence_counter = 0
NODE_ID = get_node_id()  // 10 bits: 0-1023

function generate_sequence_id():
    current_timestamp = now_milliseconds()
    
    if current_timestamp == last_timestamp:
        // Same millisecond: increment sequence
        sequence_counter = (sequence_counter + 1) & 8191  // Mask to 13 bits (0-8191)
        
        if sequence_counter == 0:
            // Sequence exhausted (>8192 IDs in 1ms), wait for next millisecond
            while now_milliseconds() <= last_timestamp:
                // Busy wait (typically < 1ms)
                pass
            current_timestamp = now_milliseconds()
            sequence_counter = 0
    else:
        // New millisecond: reset sequence
        sequence_counter = 0
    
    // Prevent clock going backward
    if current_timestamp < last_timestamp:
        throw ClockMovedBackwardException(
            "Clock moved backward by " + (last_timestamp - current_timestamp) + "ms"
        )
    
    last_timestamp = current_timestamp
    
    // Build 64-bit ID:
    // [41 bits timestamp] [10 bits node_id] [13 bits sequence]
    id = (current_timestamp << 23) | (NODE_ID << 13) | sequence_counter
    
    return id
```

**Time Complexity:** O(1) typical, O(n) worst-case if sequence exhausted (n ≤ 1ms)

**Throughput:** 8,192 IDs per millisecond per node = 8.192 million IDs/second/node

**Space Complexity:** O(1)

**Example Usage:**
```
seq_id = generate_sequence_id()
// seq_id = 879609302220800
// Decode:
//   timestamp = seq_id >> 23 = 104976465 ms since epoch
//   node_id = (seq_id >> 13) & 1023 = 42
//   sequence = seq_id & 8191 = 0
```

---

### decode_sequence_id()

**Purpose:** Decode Snowflake ID into components for debugging.

**Parameters:**
- seq_id: number - 64-bit sequence ID

**Returns:**
- object - {timestamp: number, node_id: number, sequence: number}

**Algorithm:**
```
function decode_sequence_id(seq_id):
    // Extract timestamp (41 bits, shifted left by 23)
    timestamp = seq_id >> 23
    
    // Extract node_id (10 bits, shifted left by 13)
    node_id = (seq_id >> 13) & 1023  // Mask to 10 bits
    
    // Extract sequence (13 bits, rightmost)
    sequence = seq_id & 8191  // Mask to 13 bits
    
    return {
        timestamp: timestamp,
        timestamp_human: milliseconds_to_datetime(timestamp),
        node_id: node_id,
        sequence: sequence
    }
```

**Time Complexity:** O(1)

**Example Usage:**
```
decoded = decode_sequence_id(879609302220800)
// {
//   timestamp: 104976465,
//   timestamp_human: "2024-10-27 10:30:00.000",
//   node_id: 42,
//   sequence: 0
// }
```

---

## 5. Presence Service

### set_user_online()

**Purpose:** Mark user as online in Redis with TTL.

**Parameters:**
- user_id: string - User identifier

**Returns:**
- boolean - Success status

**Algorithm:**
```
function set_user_online(user_id):
    key = "user:" + user_id + ":presence"
    
    // Set key with 60-second TTL
    result = redis.set(key, "online", ex=60)
    
    if result:
        // Publish presence event for real-time updates
        redis.publish("presence_updates", json_stringify({
            user_id: user_id,
            status: "online",
            timestamp: now()
        }))
        
        return true
    else:
        return false
```

**Time Complexity:** O(1)

**TTL Strategy:** 60-second TTL with 30-second heartbeat refresh

**Example Usage:**
```
set_user_online("12345")
// Key: "user:12345:presence" = "online" (expires in 60s)
```

---

### is_user_online()

**Purpose:** Check if user is currently online.

**Parameters:**
- user_id: string - User identifier

**Returns:**
- boolean - True if online, false otherwise

**Algorithm:**
```
function is_user_online(user_id):
    key = "user:" + user_id + ":presence"
    
    // Check if key exists
    presence = redis.get(key)
    
    return presence == "online"
```

**Time Complexity:** O(1)

**Latency:** <1ms (Redis GET operation)

**Example Usage:**
```
if is_user_online("67890"):
    deliver_via_websocket(message)
else:
    queue_for_offline(message)
```

---

### batch_check_presence()

**Purpose:** Check presence of multiple users efficiently (for group chat).

**Parameters:**
- user_ids: array[string] - List of user IDs to check

**Returns:**
- map[string, boolean] - Map of user_id to online status

**Algorithm:**
```
function batch_check_presence(user_ids):
    if user_ids.length == 0:
        return {}
    
    // Build keys
    keys = []
    for user_id in user_ids:
        keys.append("user:" + user_id + ":presence")
    
    // Batch GET (single roundtrip)
    values = redis.mget(keys)
    
    // Build result map
    result = {}
    for i in range(user_ids.length):
        user_id = user_ids[i]
        presence = values[i]
        result[user_id] = (presence == "online")
    
    return result
```

**Time Complexity:** O(n) where n is number of users

**Latency:** ~1-2ms for 50 users (vs 50ms with individual queries)

**Example Usage:**
```
group_members = ["12345", "67890", "11111", "22222", ...]  // 50 members
online_status = batch_check_presence(group_members)

for user_id in group_members:
    if online_status[user_id]:
        deliver_immediately(user_id)
    else:
        queue_offline(user_id)
```

---

## 6. Group Chat Fanout

### fanout_group_message()

**Purpose:** Distribute message to all group members efficiently.

**Parameters:**
- message: object - Full message object
- group_id: string - Group identifier

**Returns:**
- object - {online_delivered: number, offline_queued: number, total_latency_ms: number}

**Algorithm:**
```
function fanout_group_message(message, group_id):
    start_time = now()
    
    // Step 1: Get all group members
    members = redis.smembers("group:" + group_id + ":members")
    
    if members.length == 0:
        return {online_delivered: 0, offline_queued: 0, total_latency_ms: 0}
    
    // Step 2: Batch check presence
    online_status = batch_check_presence(members)
    
    online_members = []
    offline_members = []
    
    for member_id in members:
        if online_status[member_id]:
            online_members.append(member_id)
        else:
            offline_members.append(member_id)
    
    // Step 3: Choose fanout strategy based on group size
    if members.length < 50:
        // Small group: Synchronous fanout
        fanout_result = synchronous_fanout(message, online_members, offline_members)
    else:
        // Large group: Asynchronous fanout via worker queue
        fanout_result = asynchronous_fanout(message, online_members, offline_members)
    
    total_latency_ms = (now() - start_time)
    
    return {
        online_delivered: fanout_result.online_count,
        offline_queued: fanout_result.offline_count,
        total_latency_ms: total_latency_ms
    }
```

**Time Complexity:** O(n) where n is group size

**Average Latency:** 100-200ms (small groups), 500ms-1s (large groups)

**Example Usage:**
```
result = fanout_group_message(message, "team-engineering")
// {online_delivered: 32, offline_queued: 18, total_latency_ms: 150}
```

---

### synchronous_fanout()

**Purpose:** Immediately fan out to all members in parallel (for small groups).

**Parameters:**
- message: object - Message to deliver
- online_members: array[string] - Online user IDs
- offline_members: array[string] - Offline user IDs

**Returns:**
- object - {online_count: number, offline_count: number}

**Algorithm:**
```
function synchronous_fanout(message, online_members, offline_members):
    online_count = 0
    offline_count = 0
    
    // Fanout to online members in parallel
    parallel_tasks = []
    
    for member_id in online_members:
        task = spawn_async(deliver_to_online_user, message, member_id)
        parallel_tasks.append(task)
    
    // Wait for all online deliveries (with timeout)
    results = wait_all(parallel_tasks, timeout_ms=1000)
    
    for result in results:
        if result.success:
            online_count += 1
        else:
            // Failed to deliver, queue offline
            offline_members.append(result.user_id)
    
    // Queue for offline members
    for member_id in offline_members:
        queue_offline_message(message, member_id)
        send_push_notification(message, member_id)
        offline_count += 1
    
    return {online_count: online_count, offline_count: offline_count}
```

**Time Complexity:** O(n) with parallel execution, ~O(1) wall clock time

**Performance:** 10 members delivered in ~100ms (parallel), vs 1000ms (sequential)

**Example Usage:**
```
result = synchronous_fanout(message, online_members, offline_members)
log("Delivered to " + result.online_count + " online users")
```

---

## 7. Offline Message Handling

### queue_offline_message()

**Purpose:** Add message to user's offline queue for later delivery.

**Parameters:**
- message: object - Full message object
- user_id: string - Recipient user ID

**Returns:**
- number - Queue length after insertion

**Algorithm:**
```
function queue_offline_message(message, user_id):
    queue_key = "offline_messages:" + user_id
    
    // Add message to head of list (LPUSH = left push, FIFO)
    queue_length = redis.lpush(queue_key, json_stringify(message))
    
    // Set TTL if this is first message (queue was empty)
    if queue_length == 1:
        redis.expire(queue_key, 604800)  // 7 days = 604800 seconds
    
    // Check if queue is too large
    if queue_length > 10000:
        // Queue too large, trim to last 10000 messages
        redis.ltrim(queue_key, 0, 9999)
        log_warning("Offline queue for user " + user_id + " exceeded 10K, trimming")
    
    return queue_length
```

**Time Complexity:** O(1) for LPUSH, O(n) for LTRIM (rare)

**Storage:** ~1 KB per message, 10K max = 10 MB per user max

**Example Usage:**
```
queue_length = queue_offline_message(message, "67890")
// Queue: "offline_messages:67890" length = 5
```

---

### fetch_offline_messages()

**Purpose:** Retrieve all pending offline messages for user.

**Parameters:**
- user_id: string - User identifier

**Returns:**
- array[object] - List of message objects

**Algorithm:**
```
function fetch_offline_messages(user_id):
    queue_key = "offline_messages:" + user_id
    
    // Get all messages (0 to -1 = all elements)
    message_strings = redis.lrange(queue_key, 0, -1)
    
    if message_strings.length == 0:
        return []
    
    // Parse JSON
    messages = []
    for msg_str in message_strings:
        message = json_parse(msg_str)
        messages.append(message)
    
    // Reverse to get chronological order (oldest first)
    messages.reverse()
    
    return messages
```

**Time Complexity:** O(n) where n is number of queued messages

**Average Case:** n < 100 messages, ~5-10ms

**Example Usage:**
```
messages = fetch_offline_messages("67890")
for message in messages:
    deliver_to_websocket(message)
clear_offline_queue("67890")
```

---

### clear_offline_queue()

**Purpose:** Delete offline message queue after successful delivery.

**Parameters:**
- user_id: string - User identifier

**Returns:**
- boolean - Success status

**Algorithm:**
```
function clear_offline_queue(user_id):
    queue_key = "offline_messages:" + user_id
    
    // Delete the entire list
    result = redis.delete(queue_key)
    
    return result > 0
```

**Time Complexity:** O(1)

**Example Usage:**
```
clear_offline_queue("67890")
```

---

## 8. Read Receipt Operations

### mark_message_read()

**Purpose:** Record that a message was read by the recipient.

**Parameters:**
- message_id: number - Sequence ID of message
- reader_id: string - User who read the message
- chat_id: string - Chat identifier

**Returns:**
- boolean - Success status

**Algorithm:**
```
function mark_message_read(message_id, reader_id, chat_id):
    // Step 1: Publish read event to Kafka
    read_event = {
        message_id: message_id,
        chat_id: chat_id,
        reader_id: reader_id,
        read_at: now()
    }
    
    partition = hash(chat_id) % KAFKA_PARTITION_COUNT
    kafka.publish("read-receipts", read_event, partition)
    
    // Step 2: Update local cache (optimistic)
    cache_key = "read_receipt:" + message_id + ":" + reader_id
    redis.setex(cache_key, 3600, "read")  // Cache for 1 hour
    
    return true
```

**Time Complexity:** O(1)

**Latency:** ~10ms (Kafka publish)

**Example Usage:**
```
// User views message
mark_message_read(879609302220800, "67890", "abc-123")
```

---

### process_read_receipt()

**Purpose:** Consumer worker processes read receipt event from Kafka.

**Parameters:**
- read_event: object - Read receipt event

**Returns:**
- void

**Algorithm:**
```
function process_read_receipt(read_event):
    message_id = read_event.message_id
    reader_id = read_event.reader_id
    read_at = read_event.read_at
    
    // Step 1: Update database
    cassandra.execute(
        "UPDATE message_status SET read_at = ? WHERE message_id = ? AND reader_id = ?",
        [read_at, message_id, reader_id]
    )
    
    // Step 2: Lookup sender
    message_info = cassandra.execute(
        "SELECT sender_id FROM messages WHERE seq_id = ?",
        [message_id]
    )
    
    if message_info == null:
        return  // Message not found
    
    sender_id = message_info.sender_id
    
    // Step 3: Notify sender if online
    if is_user_online(sender_id):
        notification = {
            type: "READ_RECEIPT",
            message_id: message_id,
            read_by: reader_id,
            read_at: read_at
        }
        
        deliver_to_online_user(notification, sender_id)
```

**Time Complexity:** O(1)

**Example Usage:**
```
// Kafka consumer loop
while true:
    event = kafka.consume("read-receipts")
    process_read_receipt(event)
```

---

## 9. Message Editing

### edit_message()

**Purpose:** Edit previously sent message (within time window).

**Parameters:**
- message_id: number - ID of message to edit
- editor_id: string - User requesting edit (must be sender)
- new_content: string - Updated message content

**Returns:**
- object - {success: boolean, error: string}

**Algorithm:**
```
function edit_message(message_id, editor_id, new_content):
    // Step 1: Fetch original message
    original = cassandra.execute(
        "SELECT sender_id, content, sent_at FROM messages WHERE seq_id = ?",
        [message_id]
    )
    
    if original == null:
        return {success: false, error: "Message not found"}
    
    // Step 2: Verify editor is sender
    if original.sender_id != editor_id:
        return {success: false, error: "Only sender can edit"}
    
    // Step 3: Check time window (15 minutes)
    time_diff_minutes = (now() - original.sent_at) / 60000
    if time_diff_minutes > 15:
        return {success: false, error: "Edit window expired (15 minutes)"}
    
    // Step 4: Validate new content
    if new_content.length == 0:
        return {success: false, error: "Content cannot be empty"}
    if new_content.length > 5000:
        return {success: false, error: "Content too long"}
    
    // Step 5: Publish edit event to Kafka
    edit_event = {
        message_id: message_id,
        old_content: original.content,
        new_content: new_content,
        edited_at: now(),
        editor_id: editor_id
    }
    
    kafka.publish("message-edits", edit_event)
    
    return {success: true}
```

**Time Complexity:** O(1)

**Edit Window:** 15 minutes (configurable)

**Example Usage:**
```
result = edit_message(879609302220800, "12345", "Updated message")
if result.success:
    send_ack_to_client()
```

---

## 10. Message Deletion

### delete_message()

**Purpose:** Soft-delete message (mark as deleted, keep for compliance).

**Parameters:**
- message_id: number - ID of message to delete
- deleter_id: string - User requesting deletion
- delete_type: string - "me" or "everyone"

**Returns:**
- object - {success: boolean, error: string}

**Algorithm:**
```
function delete_message(message_id, deleter_id, delete_type):
    // Step 1: Fetch original message
    original = cassandra.execute(
        "SELECT sender_id, chat_id, sent_at FROM messages WHERE seq_id = ?",
        [message_id]
    )
    
    if original == null:
        return {success: false, error: "Message not found"}
    
    // Step 2: Verify permissions
    if delete_type == "everyone":
        // Only sender can delete for everyone
        if original.sender_id != deleter_id:
            return {success: false, error: "Only sender can delete for everyone"}
        
        // Check time window (1 hour)
        time_diff_minutes = (now() - original.sent_at) / 60000
        if time_diff_minutes > 60:
            return {success: false, error: "Delete window expired (1 hour)"}
    
    // Step 3: Publish delete event to Kafka
    delete_event = {
        message_id: message_id,
        chat_id: original.chat_id,
        delete_type: delete_type,
        deleted_by: deleter_id,
        deleted_at: now()
    }
    
    kafka.publish("message-deletes", delete_event)
    
    return {success: true}
```

**Time Complexity:** O(1)

**Delete Window:** 1 hour for "delete for everyone"

**Example Usage:**
```
result = delete_message(879609302220800, "12345", "everyone")
if result.success:
    send_ack_to_client()
```

---

## Summary

This pseudocode document provides complete implementations for all core operations in the Live Chat System:

1. **WebSocket Management:** Connection lifecycle, heartbeats, cleanup
2. **Message Sending:** Validation, sequencing, publishing to Kafka
3. **Message Receiving:** Online/offline delivery, presence checks
4. **Sequence IDs:** Snowflake algorithm for distributed ID generation
5. **Presence:** Online/offline status tracking with TTL
6. **Group Chat:** Efficient fanout to multiple recipients
7. **Offline Handling:** Message queuing and delivery on reconnect
8. **Read Receipts:** Tracking and notifying message read status
9. **Message Editing:** Within-window content updates
10. **Message Deletion:** Soft-delete with compliance retention

All algorithms are optimized for:
- **Low Latency:** <500ms end-to-end message delivery
- **High Throughput:** 870K messages/sec peak
- **Scalability:** 100M concurrent connections
- **Reliability:** Zero message loss with Kafka durability

**Performance Characteristics:**
- Message send: O(1) - ~25ms
- Message receive: O(1) - ~50ms
- Group fanout: O(n) - ~100-500ms
- Sequence ID generation: O(1) - <0.1ms
- Presence check: O(1) - <1ms

See [hld-diagram.md](hld-diagram.md) for architecture diagrams and [sequence-diagrams.md](sequence-diagrams.md) for detailed interaction flows.
