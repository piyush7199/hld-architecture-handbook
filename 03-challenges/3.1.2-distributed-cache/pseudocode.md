# Distributed Cache - Pseudocode Implementations

This document contains detailed algorithm implementations for the Distributed Cache system. The main challenge document
references these functions.

---

## Table of Contents

1. [Consistent Hashing](#consistent-hashing)
2. [LRU Cache Implementation](#lru-cache-implementation)
3. [Cache-Aside Pattern](#cache-aside-pattern)
4. [Cache Stampede Prevention](#cache-stampede-prevention)
5. [TTL Management](#ttl-management)
6. [Replication](#replication)
7. [Connection Pooling](#connection-pooling)
8. [Batch Operations (Pipelining)](#batch-operations-pipelining)
9. [Compression for Large Values](#compression-for-large-values)
10. [Anti-Pattern Examples](#anti-pattern-examples)

---

## Consistent Hashing

### ConsistentHash Class

Implements consistent hashing with virtual nodes for distributed cache partitioning.

**Purpose:** Minimize data movement when nodes are added/removed

**Algorithm:**

```
ConsistentHash:
  ring: HashMap  // hash_value → node
  sorted_keys: SortedList  // All hash values in sorted order
  virtual_nodes: integer = 150  // VNodes per physical node
  
  function initialize(nodes, virtual_nodes=150):
    ring = empty_hashmap
    sorted_keys = empty_list
    this.virtual_nodes = virtual_nodes
    
    for each node in nodes:
      add_node(node)
```

### add_node()

Adds a node to the consistent hash ring with virtual nodes.

**Parameters:**

- `node` (string): Node identifier (e.g., "cache-1", "192.168.1.100:6379")

**Algorithm:**

```
function add_node(node):
  for i from 0 to virtual_nodes:
    virtual_key = node + ":" + i
    hash_value = hash_function(virtual_key)
    ring[hash_value] = node
    sorted_keys.append(hash_value)
  
  sorted_keys.sort()  // Keep sorted for binary search
```

### get_node()

Finds which node a key should be stored on.

**Parameters:**

- `key` (string): The cache key

**Returns:**

- `node` (string): The node that should store this key

**Algorithm:**

```
function get_node(key):
  if ring is empty:
    return null
  
  hash_value = hash_function(key)
  
  // Binary search for the next node on the ring
  idx = binary_search_ceiling(sorted_keys, hash_value)
  if idx == length(sorted_keys):
    idx = 0  // Wrap around to first node
  
  return ring[sorted_keys[idx]]
```

### remove_node()

Removes a node from the ring.

**Parameters:**

- `node` (string): Node to remove

**Algorithm:**

```
function remove_node(node):
  keys_to_remove = empty_list
  
  for each hash_value, stored_node in ring:
    if stored_node == node:
      keys_to_remove.append(hash_value)
  
  for each hash_value in keys_to_remove:
    delete ring[hash_value]
    remove hash_value from sorted_keys
```

### hash_function()

Hashes a string to an integer (0 to 2^32-1).

**Algorithm:**

```
function hash_function(key):
  // Use MD5 or MurmurHash for distribution
  return md5_hash(key) as 32_bit_integer
```

---

## LRU Cache Implementation

### LRUCache Class

Implements Least Recently Used eviction policy with O(1) operations.

**Data Structures:**

- Doubly-linked list (for LRU ordering)
- HashMap (for O(1) lookup)

**Algorithm:**

```
LRUCache:
  capacity: integer
  cache: HashMap  // key → Node
  head: Node  // Dummy head (most recently used)
  tail: Node  // Dummy tail (least recently used)
  
  function initialize(capacity):
    this.capacity = capacity
    cache = empty_hashmap
    head = Node(0, 0)  // Dummy
    tail = Node(0, 0)  // Dummy
    head.next = tail
    tail.prev = head
```

### get()

Retrieves value and marks as recently used.

**Parameters:**

- `key`: Cache key

**Returns:**

- `value`: Cached value, or null if not found

**Algorithm:**

```
function get(key):
  if key exists in cache:
    node = cache[key]
    
    // Move to front (most recently used)
    remove_node(node)
    add_to_head(node)
    
    return node.value
  
  return null
```

### put()

Inserts or updates a key-value pair.

**Parameters:**

- `key`: Cache key
- `value`: Value to cache

**Algorithm:**

```
function put(key, value):
  if key exists in cache:
    // Update existing
    remove_node(cache[key])
  
  // Create new node
  node = Node(key, value)
  add_to_head(node)
  cache[key] = node
  
  // Check capacity
  if size(cache) > capacity:
    // Evict LRU (tail)
    lru = tail.prev
    remove_node(lru)
    delete cache[lru.key]
```

### Helper Functions

```
function add_to_head(node):
  node.prev = head
  node.next = head.next
  head.next.prev = node
  head.next = node

function remove_node(node):
  prev_node = node.prev
  next_node = node.next
  prev_node.next = next_node
  next_node.prev = prev_node
```

### Node Structure

```
Node:
  key: any
  value: any
  prev: Node
  next: Node
```

---

## Cache-Aside Pattern

### get_from_cache_aside()

Implements cache-aside (lazy loading) pattern.

**Parameters:**

- `key`: Data key
- `ttl`: Time-to-live in seconds (default: 300)

**Returns:**

- `value`: The data

**Algorithm:**

```
function get_from_cache_aside(key, ttl=300):
  // 1. Try cache first
  value = cache.get(key)
  
  if value is not null:
    return value  // Cache Hit ✅
  
  // 2. Cache miss - fetch from database
  value = database.query(key)
  
  if value is not null:
    // 3. Populate cache for next time
    cache.set(key, value, ttl)
  
  return value
```

### update_with_cache_aside()

Updates data using cache-aside pattern.

**Parameters:**

- `key`: Data key
- `new_value`: New value to store

**Algorithm:**

```
function update_with_cache_aside(key, new_value):
  // 1. Update database (source of truth)
  database.update(key, new_value)
  
  // 2. Invalidate cache (don't update!)
  cache.delete(key)
  
  // Next read will fetch fresh data from DB and populate cache
```

**Why Delete Instead of Update:**

- Simpler (no serialization needed)
- Safer (avoids inconsistency if DB update succeeds but cache fails)
- Lazy (don't waste effort on data that might not be read)

---

## Cache Stampede Prevention

### Single-Flight Request Pattern

Prevents multiple requests from hitting database when cache expires.

**Algorithm:**

```
SingleFlightCache:
  cache: HashMap
  locks: HashMap  // Per-key locks
  lock_mutex: Mutex  // Protects locks map
  
  function get_or_fetch(key, fetch_function, ttl=300):
    // Try cache first
    value = cache.get(key)
    if value is not null:
      return value
    
    // Get or create lock for this key
    acquire_lock(lock_mutex):
      if key not in locks:
        locks[key] = create_lock()
      lock = locks[key]
    
    // Acquire lock (only first request proceeds)
    acquire_lock(lock):
      // Double-check cache (another thread may have populated it)
      value = cache.get(key)
      if value is not null:
        return value
      
      // Fetch from DB (only once!)
      value = fetch_function()
      cache.set(key, value, ttl)
      
      // Cleanup lock
      acquire_lock(lock_mutex):
        delete locks[key]
      
      return value
```

**Usage Example:**

```
cache = SingleFlightCache()

function get_user(user_id):
  return cache.get_or_fetch(
    "user:" + user_id,
    lambda: database.query("SELECT * FROM users WHERE id = ?", user_id),
    ttl=300
  )

// 10,000 concurrent requests for same user:
// - Only 1 DB query executed
// - Other 9,999 requests wait for result
```

### Probabilistic Early Expiration

Alternative approach: refresh cache before expiration.

**Algorithm:**

```
function get_with_early_refresh(key, ttl=600):
  entry = cache.get_with_expiry(key)
  
  if entry is null:
    return fetch_and_cache(key, ttl)
  
  value, expires_at = entry
  time_to_expiry = expires_at - current_time()
  
  // Refresh probabilistically before expiry
  // Higher probability as expiration approaches
  if time_to_expiry < ttl * 0.1:  // Last 10% of TTL
    probability = 1 - (time_to_expiry / (ttl * 0.1))
    if random() < probability:
      // Asynchronously refresh in background
      async_execute(refresh_cache(key, ttl))
  
  return value

function refresh_cache(key, ttl):
  value = database.query(key)
  cache.set(key, value, ttl)
```

---

## TTL Management

### Active Expiration (Lazy Delete)

Check TTL on access.

**Algorithm:**

```
function get_with_ttl_check(key):
  entry = storage.get(key)
  
  if entry is null:
    return null
  
  // Check TTL on access
  if entry.expires_at and current_time() > entry.expires_at:
    storage.delete(key)  // Lazy delete
    return null
  
  return entry.value
```

### Passive Expiration (Background Scan)

Background job to clean up expired keys.

**Algorithm:**

```
function background_ttl_cleaner():
  while true:
    // Sample random keys
    sample = storage.random_sample(100)
    expired_count = 0
    
    for each key in sample:
      entry = storage.get(key)
      if entry.expires_at and current_time() > entry.expires_at:
        storage.delete(key)
        expired_count = expired_count + 1
    
    // If many expired, scan more aggressively
    if expired_count > 25:
      continue  // Repeat immediately
    else:
      sleep(100_milliseconds)  // Wait before next scan
```

### Redis-Style Hybrid TTL

Combines active and passive expiration.

**Algorithm:**

```
RedisStyleTTL:
  function on_access(key):
    // Active: Check on access
    if is_expired(key):
      delete(key)
      return null
    return get(key)
  
  function background_cleaner():
    while true:
      // Sample 20 keys
      sample = random_sample(20)
      expired_count = 0
      
      for each key in sample:
        if is_expired(key):
          delete(key)
          expired_count = expired_count + 1
      
      // Probabilistic repetition
      if expired_count > 5:  // > 25% expired
        continue  // Repeat immediately
      else:
        sleep(100_milliseconds)
```

---

## Replication

### Master-Replica Async Replication

**Algorithm:**

```
MasterNode:
  replicas: List[ReplicaNode]
  replication_buffer: Queue
  
  function write(key, value):
    // 1. Write to master
    local_storage.set(key, value)
    
    // 2. Add to replication buffer
    replication_buffer.enqueue({
      operation: "SET",
      key: key,
      value: value,
      timestamp: current_time()
    })
    
    // 3. Acknowledge immediately (async replication)
    return success
  
  function replication_worker():
    while true:
      // Batch replication commands
      batch = replication_buffer.dequeue_batch(100)
      
      if batch is empty:
        sleep(10_milliseconds)
        continue
      
      // Send to all replicas (fire-and-forget)
      for each replica in replicas:
        async_send(replica, batch)
```

```
ReplicaNode:
  master: MasterNode
  local_storage: HashMap
  lag_monitor: Monitor
  
  function receive_replication(batch):
    for each command in batch:
      if command.operation == "SET":
        local_storage.set(command.key, command.value)
      else if command.operation == "DELETE":
        local_storage.delete(command.key)
    
    // Update lag metrics
    last_replicated = batch.last().timestamp
    lag = current_time() - last_replicated
    lag_monitor.record(lag)
```

### Master Failover with Sentinel

**Algorithm:**

```
Sentinel:
  masters: HashMap  // master_name → master_info
  
  function monitor():
    while true:
      for each master_name, master_info in masters:
        if not ping(master_info.address):
          master_info.failures = master_info.failures + 1
          
          if master_info.failures >= threshold:
            initiate_failover(master_name)
        else:
          master_info.failures = 0
      
      sleep(1_second)
  
  function initiate_failover(master_name):
    master_info = masters[master_name]
    
    // 1. Quorum vote
    if not get_quorum_vote(master_name):
      return  // Not enough sentinels agree
    
    // 2. Select best replica
    best_replica = select_replica_with_least_lag(master_info.replicas)
    
    // 3. Promote replica
    send_command(best_replica, "REPLICAOF NO ONE")
    
    // 4. Update other replicas
    for each replica in master_info.replicas:
      if replica != best_replica:
        send_command(replica, "REPLICAOF " + best_replica.address)
    
    // 5. Update clients
    broadcast_topology_change(master_name, best_replica)
```

---

## Connection Pooling

### ConnectionPool Class

Manages reusable connections to cache nodes.

**Algorithm:**

```
ConnectionPool:
  available: Queue  // Available connections
  active: Set  // In-use connections
  max_connections: integer
  host: string
  port: integer
  
  function initialize(host, port, max_connections=50):
    this.host = host
    this.port = port
    this.max_connections = max_connections
    available = empty_queue
    active = empty_set
  
  function get_connection():
    // Try to get available connection
    if not available.is_empty():
      conn = available.dequeue()
      active.add(conn)
      return conn
    
    // Create new if under limit
    if size(active) < max_connections:
      conn = create_connection(host, port)
      active.add(conn)
      return conn
    
    // Wait for available connection (with timeout)
    conn = available.dequeue(timeout=5_seconds)
    if conn is null:
      raise error("Connection pool exhausted")
    active.add(conn)
    return conn
  
  function release_connection(conn):
    active.remove(conn)
    
    if conn.is_alive():
      available.enqueue(conn)
    else:
      conn.close()  // Don't return dead connections
```

---

## Batch Operations (Pipelining)

### Pipeline Class

Batches multiple operations into single network round-trip.

**Algorithm:**

```
Pipeline:
  commands: List
  connection: Connection
  
  function initialize(connection):
    this.connection = connection
    commands = empty_list
  
  function set(key, value):
    commands.append({
      operation: "SET",
      key: key,
      value: value
    })
  
  function get(key):
    commands.append({
      operation: "GET",
      key: key
    })
  
  function execute():
    // Send all commands in one network call
    results = connection.send_batch(commands)
    commands = empty_list
    return results
```

**Usage Example:**

```
// ❌ Bad: 1000 network round-trips
for i from 0 to 999:
  cache.set("key:" + i, i)

// ✅ Good: 1 network round-trip
pipeline = cache.create_pipeline()
for i from 0 to 999:
  pipeline.set("key:" + i, i)
pipeline.execute()
```

---

## Compression for Large Values

### set_compressed()

**Purpose:** Compress and store large values (> 1 KB) to save memory.

**Parameters:**

- `key`: Cache key
- `value`: Data to compress and store
- `ttl`: Time-to-live in seconds (optional)

**Algorithm:**

```
function set_compressed(key, value, ttl=null):
  // 1. Serialize value to JSON/binary
  serialized = serialize_to_json(value)
  
  // 2. Check if compression is worthwhile
  if length(serialized) < 1024:  // Less than 1 KB
    // Not worth compressing
    cache.set(key, serialized, ttl)
    return
  
  // 3. Compress using gzip or snappy
  compressed = compress_gzip(serialized)
  
  // 4. Store with metadata flag
  metadata = {
    'compressed': true,
    'algorithm': 'gzip',
    'original_size': length(serialized),
    'compressed_size': length(compressed)
  }
  
  cache_value = {
    'metadata': metadata,
    'data': compressed
  }
  
  cache.set(key, cache_value, ttl)
  
  log_info("Compressed " + key + 
           " from " + metadata.original_size + 
           " to " + metadata.compressed_size + " bytes " +
           "(" + (100 * metadata.compressed_size / metadata.original_size) + "% savings)")
```

### get_compressed()

**Purpose:** Retrieve and decompress stored values.

**Parameters:**

- `key`: Cache key

**Returns:** Decompressed original value or null if not found

**Algorithm:**

```
function get_compressed(key):
  // 1. Retrieve from cache
  cache_value = cache.get(key)
  
  if cache_value is null:
    return null
  
  // 2. Check if data is compressed
  if cache_value.metadata and cache_value.metadata.compressed:
    // 3. Decompress
    compressed_data = cache_value.data
    algorithm = cache_value.metadata.algorithm
    
    if algorithm == 'gzip':
      decompressed = decompress_gzip(compressed_data)
    else if algorithm == 'snappy':
      decompressed = decompress_snappy(compressed_data)
    else:
      throw error("Unknown compression algorithm: " + algorithm)
    
    // 4. Deserialize
    value = deserialize_from_json(decompressed)
    return value
  else:
    // Data not compressed, return as-is
    return deserialize_from_json(cache_value)
```

**Example Usage:**

```
// Store large user profile (10 KB)
user_profile = {
  id: 12345,
  name: "John Doe",
  bio: "..." // Long text
  posts: [...] // Array of 100 posts
}

// Compress and store (10 KB → 1 KB)
set_compressed('user:12345', user_profile, TTL=3600)

// Retrieve and decompress automatically
profile = get_compressed('user:12345')
```

**Benefits:**

- 10x memory savings for large objects
- Reduced network bandwidth
- Faster cache eviction (more data fits in memory)

**Trade-offs:**

- CPU overhead for compression/decompression (~1-5ms)
- Complexity in handling metadata

---

## Anti-Pattern Examples

### ❌ Not Setting TTL

**Problem:**

```
function cache_user_bad(user_id, user_data):
  cache.set('user:' + user_id, user_data)  // No TTL!
  // Cache grows forever, never expires
```

**Solution:**

```
function cache_user_good(user_id, user_data):
  cache.set('user:' + user_id, user_data, TTL=300)  // 5 minutes
```

### ❌ Storing Large Objects

**Problem:**

```
function cache_all_users_bad():
  all_users = database.query("SELECT * FROM users")  // 10 MB!
  cache.set('users:all', all_users)
  // Wastes memory, slow serialization
```

**Solution:**

```
function cache_all_users_good():
  all_users = database.query("SELECT * FROM users")
  
  // Paginate
  for i from 0 to length(all_users) step 100:
    chunk = all_users[i : i+100]
    page_num = i / 100
    cache.set("users:page:" + page_num, chunk, TTL=300)
```

### ❌ Using Cache as Primary Storage

**Problem:**

```
function save_session_bad(session_id, session_data):
  cache.set('session:' + session_id, session_data, TTL=3600)
  // If cache fails, data is lost!
```

**Solution:**

```
function save_session_good(session_id, session_data):
  // 1. Save to database (primary storage)
  database.insert_or_update('sessions', session_id, session_data)
  
  // 2. Cache for performance (secondary)
  cache.set('session:' + session_id, session_data, TTL=3600)
```

---

This pseudocode provides detailed implementations of all algorithms discussed in the main Distributed Cache design
document.

