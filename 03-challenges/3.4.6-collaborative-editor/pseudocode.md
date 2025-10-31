# Collaborative Editor - Pseudocode Implementations

This document contains detailed algorithm implementations for the Collaborative Editor system. The main challenge document references these functions.

---

## Table of Contents

1. [Operational Transformation (OT)](#1-operational-transformation-ot)
2. [CRDT (Conflict-Free Replicated Data Types)](#2-crdt-conflict-free-replicated-data-types)
3. [Document State Management](#3-document-state-management)
4. [WebSocket Server](#4-websocket-server)
5. [Event Store](#5-event-store)
6. [Snapshot Manager (CQRS)](#6-snapshot-manager-cqrs)
7. [Offline Sync](#7-offline-sync)
8. [Presence and Cursor Tracking](#8-presence-and-cursor-tracking)
9. [Rate Limiting](#9-rate-limiting)
10. [Failure Recovery](#10-failure-recovery)

---

## 1. Operational Transformation (OT)

### transform()

**Purpose:** Transforms an operation against another operation to resolve conflicts.

**Parameters:**
- op1 (Operation): First operation (already applied to document)
- op2 (Operation): Second operation (needs transformation)

**Returns:**
- Operation: Transformed op2 that accounts for op1

**Algorithm:**

```
function transform(op1, op2):
    // Handle insert vs insert
    if op1.type == "insert" and op2.type == "insert":
        if op1.position < op2.position:
            // op1 is before op2, no change needed
            return op2
        else if op1.position > op2.position:
            // op1 is after op2, no change needed
            return op2
        else:  // Same position
            // Tie-breaker: use user_id (deterministic)
            if op1.user_id < op2.user_id:
                op2.position += len(op1.content)
            return op2
    
    // Handle insert vs delete
    else if op1.type == "insert" and op2.type == "delete":
        if op1.position <= op2.position:
            // op1 inserted before delete range, shift delete position
            op2.position += len(op1.content)
        else if op1.position < op2.position + op2.length:
            // op1 inserted within delete range, expand delete length
            op2.length += len(op1.content)
        return op2
    
    // Handle delete vs insert
    else if op1.type == "delete" and op2.type == "insert":
        if op1.position + op1.length <= op2.position:
            // Delete is before insert, shift insert position
            op2.position -= op1.length
        else if op1.position < op2.position:
            // Delete overlaps insert position
            op2.position = op1.position
        return op2
    
    // Handle delete vs delete
    else if op1.type == "delete" and op2.type == "delete":
        if op1.position + op1.length <= op2.position:
            // Delete 1 is before delete 2, shift delete 2 position
            op2.position -= op1.length
        else if op1.position >= op2.position + op2.length:
            // Delete 1 is after delete 2, no change
            return op2
        else:
            // Deletes overlap, adjust position and length
            overlap_start = max(op1.position, op2.position)
            overlap_end = min(op1.position + op1.length, op2.position + op2.length)
            overlap_length = overlap_end - overlap_start
            
            op2.length -= overlap_length
            if op1.position < op2.position:
                op2.position -= (op2.position - op1.position)
        return op2
    
    // Handle format operations (commutative)
    else if op1.type == "format" and op2.type == "format":
        // Formatting is commutative, no transformation needed
        return op2
    
    return op2
```

**Time Complexity:** O(1) - constant time transformation

**Example Usage:**

```
op1 = {type: "insert", position: 5, content: "X", user_id: "alice"}
op2 = {type: "insert", position: 5, content: "Y", user_id: "bob"}

transformed_op2 = transform(op1, op2)
// Result: {type: "insert", position: 6, content: "Y", user_id: "bob"}
```

---

### apply_operation()

**Purpose:** Applies an operation to a document state.

**Parameters:**
- document (Document): Current document state
- operation (Operation): Operation to apply

**Returns:**
- Document: Updated document state

**Algorithm:**

```
function apply_operation(document, operation):
    if operation.type == "insert":
        // Insert content at position
        before = document.content[0:operation.position]
        after = document.content[operation.position:]
        document.content = before + operation.content + after
        document.version += 1
        return document
    
    else if operation.type == "delete":
        // Delete content from position to position + length
        before = document.content[0:operation.position]
        after = document.content[operation.position + operation.length:]
        document.content = before + after
        document.version += 1
        return document
    
    else if operation.type == "format":
        // Apply formatting to range
        for i in range(operation.range[0], operation.range[1]):
            if not document.formatting[i]:
                document.formatting[i] = {}
            document.formatting[i][operation.attribute] = operation.value
        document.version += 1
        return document
    
    return document
```

**Time Complexity:** 
- Insert: O(n) where n is document length (string concatenation)
- Delete: O(n)
- Format: O(m) where m is range length

---

## 2. CRDT (Conflict-Free Replicated Data Types)

### CRDTDocument

**Purpose:** Represents a document using CRDT (Replicated Growable Array).

**Data Structure:**

```
class CRDTCharacter:
    id: {siteId: string, logicalClock: int}
    content: string
    deleted: boolean

class CRDTDocument:
    characters: Array<CRDTCharacter>
    siteId: string
    logicalClock: int
```

---

### generate_crdt_id()

**Purpose:** Generates a globally unique ID for a character.

**Parameters:**
- siteId (string): Unique site identifier (user_id)
- logicalClock (int): Monotonically increasing counter

**Returns:**
- dict: CRDT ID {siteId, logicalClock}

**Algorithm:**

```
function generate_crdt_id(siteId, logicalClock):
    return {
        siteId: siteId,
        logicalClock: logicalClock
    }
```

---

### crdt_insert()

**Purpose:** Inserts a character into CRDT document.

**Parameters:**
- document (CRDTDocument): CRDT document
- position (int): Position to insert
- content (string): Content to insert

**Returns:**
- Operation: CRDT insert operation

**Algorithm:**

```
function crdt_insert(document, position, content):
    // Generate unique ID for new character
    document.logicalClock += 1
    char_id = generate_crdt_id(document.siteId, document.logicalClock)
    
    // Create new character
    new_char = CRDTCharacter(
        id: char_id,
        content: content,
        deleted: false
    )
    
    // Insert at position (using fractional indexing for IDs)
    if position == 0:
        // Insert at beginning
        if len(document.characters) == 0:
            new_char.id.logicalClock = 1.0
        else:
            new_char.id.logicalClock = document.characters[0].id.logicalClock - 1
    else if position >= len(document.characters):
        // Insert at end
        new_char.id.logicalClock = document.characters[-1].id.logicalClock + 1
    else:
        // Insert between two characters (fractional indexing)
        prev_clock = document.characters[position - 1].id.logicalClock
        next_clock = document.characters[position].id.logicalClock
        new_char.id.logicalClock = (prev_clock + next_clock) / 2.0
    
    document.characters.insert(position, new_char)
    
    return {
        type: "crdt_insert",
        character: new_char
    }
```

**Time Complexity:** O(n) - insert into array

---

### crdt_delete()

**Purpose:** Deletes a character from CRDT document (tombstone).

**Parameters:**
- document (CRDTDocument): CRDT document
- position (int): Position to delete

**Returns:**
- Operation: CRDT delete operation

**Algorithm:**

```
function crdt_delete(document, position):
    // Mark character as deleted (tombstone)
    char = document.characters[position]
    char.deleted = true
    
    return {
        type: "crdt_delete",
        character_id: char.id
    }
```

**Time Complexity:** O(1) - just mark as deleted

---

### crdt_merge()

**Purpose:** Merges two CRDT document states.

**Parameters:**
- doc1 (CRDTDocument): First document state
- doc2 (CRDTDocument): Second document state

**Returns:**
- CRDTDocument: Merged document

**Algorithm:**

```
function crdt_merge(doc1, doc2):
    // Merge character sets (union)
    all_characters = {}
    
    // Add all characters from doc1
    for char in doc1.characters:
        all_characters[char.id] = char
    
    // Add all characters from doc2 (overwrites if same ID)
    for char in doc2.characters:
        if char.id in all_characters:
            // Keep the one with deleted=true (deletions win)
            if char.deleted:
                all_characters[char.id] = char
        else:
            all_characters[char.id] = char
    
    // Sort characters by ID (siteId, logicalClock)
    sorted_chars = sorted(all_characters.values(), 
                          key=lambda c: (c.id.siteId, c.id.logicalClock))
    
    // Filter out deleted characters (tombstones)
    visible_chars = [c for c in sorted_chars if not c.deleted]
    
    // Create merged document
    merged = CRDTDocument(
        characters: visible_chars,
        siteId: doc1.siteId,
        logicalClock: max(doc1.logicalClock, doc2.logicalClock)
    )
    
    return merged
```

**Time Complexity:** O(n log n) - sorting characters

**Example Usage:**

```
doc1 = CRDTDocument with "Hello"
doc2 = CRDTDocument with "World"

merged = crdt_merge(doc1, doc2)
// Result: Characters sorted by ID, deterministic merge
```

---

## 3. Document State Management

### ClientDocument

**Purpose:** Client-side document model with optimistic updates.

**Data Structure:**

```
class ClientDocument:
    content: string
    version: int
    pending_operations: Queue<Operation>
    server_version: int
    formatting: Map<int, Map<string, any>>
```

---

### apply_optimistic()

**Purpose:** Applies an operation optimistically on client.

**Parameters:**
- document (ClientDocument): Client document
- operation (Operation): Operation to apply

**Returns:**
- void (modifies document in place)

**Algorithm:**

```
function apply_optimistic(document, operation):
    // Apply operation immediately to local state
    apply_operation(document, operation)
    
    // Add to pending operations queue
    document.pending_operations.enqueue(operation)
    
    // Send to server asynchronously
    send_to_server(operation)
```

**Time Complexity:** O(n) - apply operation

---

### reconcile()

**Purpose:** Reconciles local state with server acknowledgment.

**Parameters:**
- document (ClientDocument): Client document
- server_operation (Operation): Operation acknowledged by server

**Returns:**
- void (modifies document in place)

**Algorithm:**

```
function reconcile(document, server_operation):
    // Remove acknowledged operation from pending queue
    local_op = document.pending_operations.dequeue()
    
    if local_op.id != server_operation.id:
        // Server transformed our operation, need to reconcile
        
        // Rollback local operation
        rollback_operation(document, local_op)
        
        // Apply server's transformed operation
        apply_operation(document, server_operation)
        
        // Reapply all pending operations (transform against server op)
        for pending_op in document.pending_operations:
            transformed = transform(server_operation, pending_op)
            document.pending_operations.update(pending_op, transformed)
    
    // Update server version
    document.server_version = server_operation.version
```

**Time Complexity:** O(p) where p is number of pending operations

---

## 4. WebSocket Server

### WebSocketServer

**Purpose:** Manages WebSocket connections and broadcasts operations.

**Data Structure:**

```
class WebSocketServer:
    connection_pool: Map<doc_id, Set<Connection>>
    presence: Map<doc_id, Set<user_id>>
    redis_client: RedisClient
```

---

### on_connect()

**Purpose:** Handles new WebSocket connection.

**Parameters:**
- connection (Connection): New WebSocket connection
- doc_id (string): Document ID
- user_id (string): User ID

**Returns:**
- void

**Algorithm:**

```
function on_connect(connection, doc_id, user_id):
    // Add connection to pool
    if doc_id not in connection_pool:
        connection_pool[doc_id] = Set()
    connection_pool[doc_id].add(connection)
    
    // Add to presence
    redis_client.sadd(f"doc:{doc_id}:presence", user_id)
    redis_client.expire(f"doc:{doc_id}:presence", 10)  // 10 second TTL
    
    // Broadcast join event to other collaborators
    join_event = {type: "presence", event: "join", user_id: user_id}
    broadcast_to_document(doc_id, join_event, exclude=connection)
    
    // Send current presence list to new user
    presence_list = redis_client.smembers(f"doc:{doc_id}:presence")
    connection.send({type: "presence", users: presence_list})
```

**Time Complexity:** O(c) where c is number of collaborators

---

### on_message()

**Purpose:** Handles incoming operation from client.

**Parameters:**
- connection (Connection): WebSocket connection
- message (dict): Operation message

**Returns:**
- void

**Algorithm:**

```
function on_message(connection, message):
    operation = parse_operation(message)
    
    // Validate operation
    if not validate_operation(operation):
        connection.send({error: "Invalid operation"})
        return
    
    // Check permissions
    if not has_permission(operation.user_id, operation.doc_id, "edit"):
        connection.send({error: "Permission denied"})
        return
    
    // Check rate limit
    if is_rate_limited(operation.user_id):
        connection.send({error: "Rate limit exceeded"})
        return
    
    // Publish to Kafka (for OT processing)
    kafka_producer.send(
        topic: "document-edits",
        key: operation.doc_id,
        value: operation
    )
    
    // Send acknowledgment to client
    connection.send({type: "ack", operation_id: operation.id})
```

**Time Complexity:** O(1) - constant time processing

---

### broadcast_to_document()

**Purpose:** Broadcasts operation to all collaborators on a document.

**Parameters:**
- doc_id (string): Document ID
- operation (Operation): Operation to broadcast
- exclude (Connection): Optional connection to exclude (sender)

**Returns:**
- void

**Algorithm:**

```
function broadcast_to_document(doc_id, operation, exclude=null):
    // Get all connections for this document
    connections = connection_pool.get(doc_id, Set())
    
    // Broadcast to all connections (in-memory, fast)
    for connection in connections:
        if connection != exclude:
            connection.send(operation)
    
    // Also publish to Redis Pub/Sub for cross-server broadcast
    redis_client.publish(
        channel: f"doc:{doc_id}:ops",
        message: operation
    )
```

**Time Complexity:** O(c) where c is number of collaborators

---

### on_heartbeat()

**Purpose:** Handles heartbeat from client (keep presence alive).

**Parameters:**
- connection (Connection): WebSocket connection
- doc_id (string): Document ID
- user_id (string): User ID

**Returns:**
- void

**Algorithm:**

```
function on_heartbeat(connection, doc_id, user_id):
    // Refresh presence TTL
    redis_client.expire(f"doc:{doc_id}:presence", 10)  // Reset to 10 seconds
    
    // Send heartbeat ACK
    connection.send({type: "heartbeat_ack"})
```

**Time Complexity:** O(1)

---

## 5. Event Store

### EventStore

**Purpose:** Manages event log persistence (Event Sourcing).

**Data Structure:**

```
class EventStore:
    db: PostgreSQLConnection
    cache: RedisClient
```

---

### append_event()

**Purpose:** Appends an event to the event log.

**Parameters:**
- doc_id (string): Document ID
- user_id (string): User ID
- operation (Operation): Operation to persist

**Returns:**
- int: Event ID

**Algorithm:**

```
function append_event(doc_id, user_id, operation):
    // Insert into PostgreSQL event log (append-only)
    query = "INSERT INTO document_events (doc_id, user_id, operation_type, position, content, metadata) VALUES (?, ?, ?, ?, ?, ?) RETURNING event_id"
    
    event_id = db.execute(query, [
        doc_id,
        user_id,
        operation.type,
        operation.position,
        operation.content,
        json.dumps(operation.metadata)
    ])
    
    // Cache recent events in Redis (for fast replay)
    cache.lpush(f"doc:{doc_id}:events", json.dumps(operation))
    cache.ltrim(f"doc:{doc_id}:events", 0, 999)  // Keep last 1000 events
    cache.expire(f"doc:{doc_id}:events", 3600)  // 1 hour TTL
    
    return event_id
```

**Time Complexity:** O(1) - single INSERT

---

### fetch_events_since()

**Purpose:** Fetches events since a specific version (for replay).

**Parameters:**
- doc_id (string): Document ID
- since_version (int): Event ID to fetch from

**Returns:**
- Array<Operation>: List of events

**Algorithm:**

```
function fetch_events_since(doc_id, since_version):
    // Try Redis cache first (last 1000 events)
    cached_events = cache.lrange(f"doc:{doc_id}:events", 0, -1)
    
    if cached_events and len(cached_events) > 0:
        // Filter events after since_version
        events = [json.loads(e) for e in cached_events if e.event_id > since_version]
        if len(events) > 0:
            return events
    
    // Cache miss, query PostgreSQL
    query = "SELECT * FROM document_events WHERE doc_id = ? AND event_id > ? ORDER BY event_id ASC"
    rows = db.execute(query, [doc_id, since_version])
    
    events = [parse_event(row) for row in rows]
    return events
```

**Time Complexity:** O(n) where n is number of events

---

## 6. Snapshot Manager (CQRS)

### SnapshotManager

**Purpose:** Manages document snapshots for fast reads (CQRS pattern).

**Data Structure:**

```
class SnapshotManager:
    db: PostgreSQLConnection
    cache: RedisClient
    snapshot_interval: int = 100  // Snapshot every 100 operations
```

---

### should_create_snapshot()

**Purpose:** Checks if it's time to create a snapshot.

**Parameters:**
- doc_id (string): Document ID
- current_version (int): Current event version

**Returns:**
- boolean: True if snapshot should be created

**Algorithm:**

```
function should_create_snapshot(doc_id, current_version):
    last_snapshot_version = get_last_snapshot_version(doc_id)
    operations_since_snapshot = current_version - last_snapshot_version
    
    return operations_since_snapshot >= snapshot_interval
```

**Time Complexity:** O(1)

---

### create_snapshot()

**Purpose:** Creates a snapshot of current document state.

**Parameters:**
- doc_id (string): Document ID
- document (Document): Current document state

**Returns:**
- void

**Algorithm:**

```
function create_snapshot(doc_id, document):
    // Serialize document state
    snapshot_data = {
        version: document.version,
        content: document.content,
        formatting: document.formatting,
        metadata: {
            created_at: now(),
            size_bytes: len(document.content)
        }
    }
    
    // Write to Redis (hot cache)
    cache.set(
        key: f"doc:{doc_id}:snapshot",
        value: json.dumps(snapshot_data),
        ttl: 3600  // 1 hour
    )
    
    // Write to PostgreSQL (warm storage)
    query = "INSERT INTO document_snapshots (doc_id, snapshot_version, content, metadata, created_at) VALUES (?, ?, ?, ?, ?) ON CONFLICT (doc_id) DO UPDATE SET snapshot_version = ?, content = ?, metadata = ?, created_at = ?"
    
    db.execute(query, [
        doc_id,
        document.version,
        document.content,
        json.dumps(snapshot_data.metadata),
        now(),
        document.version,
        document.content,
        json.dumps(snapshot_data.metadata),
        now()
    ])
    
    // Archive to S3 (cold storage, async)
    async_upload_to_s3(
        bucket: "document-snapshots",
        key: f"{doc_id}/v{document.version}.json",
        data: snapshot_data
    )
```

**Time Complexity:** O(n) where n is document size

---

### load_document()

**Purpose:** Loads document from snapshot + events (fast read).

**Parameters:**
- doc_id (string): Document ID

**Returns:**
- Document: Reconstructed document

**Algorithm:**

```
function load_document(doc_id):
    // 1. Try Redis cache (hottest path)
    cached_snapshot = cache.get(f"doc:{doc_id}:snapshot")
    if cached_snapshot:
        return json.loads(cached_snapshot)  // < 10ms
    
    // 2. Load snapshot from PostgreSQL (warm path)
    query = "SELECT * FROM document_snapshots WHERE doc_id = ? ORDER BY snapshot_version DESC LIMIT 1"
    snapshot_row = db.execute(query, [doc_id])
    
    if not snapshot_row:
        // No snapshot, reconstruct from all events (cold path)
        events = fetch_events_since(doc_id, 0)
        document = Document(content="", version=0)
        for event in events:
            apply_operation(document, event)
        return document
    
    // 3. Reconstruct from snapshot + events since snapshot
    snapshot = parse_snapshot(snapshot_row)
    document = Document(
        content: snapshot.content,
        version: snapshot.version,
        formatting: snapshot.formatting
    )
    
    // 4. Fetch events since snapshot
    events = fetch_events_since(doc_id, snapshot.version)
    
    // 5. Replay events (typically < 100 events)
    for event in events:
        apply_operation(document, event)
    
    // 6. Cache result in Redis
    cache.set(
        key: f"doc:{doc_id}:snapshot",
        value: json.dumps(document.to_dict()),
        ttl: 3600
    )
    
    return document
```

**Time Complexity:** O(e) where e is number of events since snapshot (typically < 100)

---

## 7. Offline Sync

### mergeOfflineEdits()

**Purpose:** Merges offline edits when client reconnects.

**Parameters:**
- doc_id (string): Document ID
- client_last_version (int): Last server version client knew
- offline_operations (Array<Operation>): Operations made offline

**Returns:**
- dict: Merged state and missing operations

**Algorithm:**

```
function mergeOfflineEdits(doc_id, client_last_version, offline_operations):
    // 1. Load current server document state
    server_document = load_document(doc_id)
    
    // 2. Fetch operations client missed while offline
    missed_operations = fetch_events_since(doc_id, client_last_version)
    
    // 3. Convert operations to CRDT for commutative merge
    server_crdt = convert_to_crdt(server_document)
    client_crdt = convert_to_crdt(client_last_version, offline_operations)
    
    // 4. Merge CRDT states (commutative, deterministic)
    merged_crdt = crdt_merge(server_crdt, client_crdt)
    
    // 5. Convert back to operation log
    merged_document = crdt_to_document(merged_crdt)
    
    // 6. Persist merged operations
    for operation in offline_operations:
        append_event(doc_id, operation.user_id, operation)
    
    // 7. Create snapshot if needed
    if should_create_snapshot(doc_id, merged_document.version):
        create_snapshot(doc_id, merged_document)
    
    // 8. Return merged state and missed operations
    return {
        merged_document: merged_document,
        missed_operations: missed_operations,
        conflicts_resolved: len(offline_operations)
    }
```

**Time Complexity:** O(n log n) where n is total operations (CRDT merge)

**Example Usage:**

```
result = mergeOfflineEdits(
    doc_id: "abc123",
    client_last_version: 42,
    offline_operations: [
        {type: "insert", pos: 5, content: "offline edit"}
    ]
)

// Result: {
//   merged_document: Document with both offline and server edits,
//   missed_operations: [...server ops 43-50...],
//   conflicts_resolved: 1
// }
```

---

## 8. Presence and Cursor Tracking

### PresenceManager

**Purpose:** Manages user presence and cursor positions.

**Data Structure:**

```
class PresenceManager:
    redis_client: RedisClient
    presence_ttl: int = 10  // seconds
```

---

### update_presence()

**Purpose:** Updates user presence (online status).

**Parameters:**
- doc_id (string): Document ID
- user_id (string): User ID

**Returns:**
- void

**Algorithm:**

```
function update_presence(doc_id, user_id):
    key = f"doc:{doc_id}:presence"
    
    // Add user to presence set with TTL
    redis_client.sadd(key, user_id)
    redis_client.expire(key, presence_ttl)
```

**Time Complexity:** O(1)

---

### get_presence()

**Purpose:** Gets list of users currently online.

**Parameters:**
- doc_id (string): Document ID

**Returns:**
- Set<string>: Set of user IDs

**Algorithm:**

```
function get_presence(doc_id):
    key = f"doc:{doc_id}:presence"
    return redis_client.smembers(key)
```

**Time Complexity:** O(c) where c is number of collaborators

---

### update_cursor()

**Purpose:** Updates user's cursor position (throttled).

**Parameters:**
- doc_id (string): Document ID
- user_id (string): User ID
- position (int): Cursor position
- selection (Array<int>): Optional selection range

**Returns:**
- void

**Algorithm:**

```
function update_cursor(doc_id, user_id, position, selection=null):
    // Check throttle (max 10 updates/sec per user)
    throttle_key = f"cursor_throttle:{user_id}"
    last_update = redis_client.get(throttle_key)
    
    if last_update and (now() - last_update) < 0.1:  // 100ms
        return  // Throttled, skip this update
    
    // Update throttle timestamp
    redis_client.set(throttle_key, now(), ttl=0.1)
    
    // Broadcast cursor position to all collaborators
    cursor_event = {
        type: "cursor",
        user_id: user_id,
        position: position,
        selection: selection
    }
    
    broadcast_to_document(doc_id, cursor_event)
```

**Time Complexity:** O(c) where c is number of collaborators

---

## 9. Rate Limiting

### is_rate_limited()

**Purpose:** Checks if user has exceeded rate limit.

**Parameters:**
- user_id (string): User ID
- limit (int): Max operations per second (default 100)

**Returns:**
- boolean: True if rate limited

**Algorithm:**

```
function is_rate_limited(user_id, limit=100):
    key = f"ratelimit:{user_id}"
    
    // Use Redis Lua script for atomic increment + check
    lua_script = """
    local key = KEYS[1]
    local limit = tonumber(ARGV[1])
    local current = redis.call("INCR", key)
    if current == 1 then
        redis.call("EXPIRE", key, 1)
    end
    if current > limit then
        return 1  -- Rate limited
    else
        return 0  -- Allow
    end
    """
    
    result = redis_client.eval(lua_script, [key], [limit])
    return result == 1
```

**Time Complexity:** O(1) - atomic Redis operation

---

## 10. Failure Recovery

### handle_ot_worker_failure()

**Purpose:** Handles OT worker failure and recovery.

**Parameters:**
- failed_worker_id (string): ID of failed worker

**Returns:**
- void

**Algorithm:**

```
function handle_ot_worker_failure(failed_worker_id):
    // 1. Kafka consumer group automatically rebalances
    // (handled by Kafka, no code needed)
    
    // 2. New worker resumes from last committed offset
    last_offset = get_last_committed_offset(failed_worker_id)
    
    // 3. May reprocess last few messages (idempotent, safe)
    // (handled by Kafka consumer, no code needed)
    
    // 4. Start new worker (auto-scaling)
    start_new_ot_worker()
    
    log("OT worker failed, rebalancing...")
```

---

### handle_websocket_disconnection()

**Purpose:** Handles WebSocket disconnection and reconnection.

**Parameters:**
- connection (Connection): Disconnected connection
- doc_id (string): Document ID
- user_id (string): User ID

**Returns:**
- void

**Algorithm:**

```
function handle_websocket_disconnection(connection, doc_id, user_id):
    // 1. Remove from connection pool
    connection_pool[doc_id].remove(connection)
    
    // 2. Presence will auto-expire after 10 seconds (no heartbeat)
    // (handled by Redis TTL, no code needed)
    
    // 3. Client will reconnect with exponential backoff
    // (handled by client, no code needed)
```

---

### handle_websocket_reconnection()

**Purpose:** Handles WebSocket reconnection after disconnection.

**Parameters:**
- connection (Connection): New connection
- doc_id (string): Document ID
- user_id (string): User ID
- last_event_id (int): Last event client received

**Returns:**
- void

**Algorithm:**

```
function handle_websocket_reconnection(connection, doc_id, user_id, last_event_id):
    // 1. Fetch missed events
    missed_events = fetch_events_since(doc_id, last_event_id)
    
    // 2. Send missed events to client
    for event in missed_events:
        connection.send(event)
    
    // 3. Re-add to connection pool
    if doc_id not in connection_pool:
        connection_pool[doc_id] = Set()
    connection_pool[doc_id].add(connection)
    
    // 4. Update presence
    update_presence(doc_id, user_id)
    
    // 5. Broadcast reconnection to others
    join_event = {type: "presence", event: "rejoin", user_id: user_id}
    broadcast_to_document(doc_id, join_event, exclude=connection)
    
    log(f"User {user_id} reconnected to doc {doc_id}, replayed {len(missed_events)} events")
```

**Time Complexity:** O(e) where e is number of missed events