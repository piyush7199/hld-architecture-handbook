# Distributed ID Generator (Snowflake) - Pseudocode Implementations

This document contains detailed algorithm implementations for the Distributed ID Generator system. The main challenge document references these functions.

---

## Table of Contents

1. [Snowflake ID Structure](#snowflake-id-structure)
2. [ID Generation Algorithm](#id-generation-algorithm)
3. [Worker ID Assignment](#worker-id-assignment)
4. [Clock Synchronization](#clock-synchronization)
5. [Alternative ID Strategies](#alternative-id-strategies)
6. [Batch ID Generation](#batch-id-generation)
7. [Sequence Exhaustion Handling](#sequence-exhaustion-handling)
8. [Anti-Pattern Examples](#anti-pattern-examples)

---

## Snowflake ID Structure

### 64-bit ID Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ 1 bit │   41 bits      │  10 bits  │      12 bits              │
│ Sign  │  Timestamp     │ Worker ID │  Sequence Number          │
│   0   │  (milliseconds)│  (0-1023) │  (0-4095 per millisecond) │
└─────────────────────────────────────────────────────────────────┘
```

**Bit Distribution:**
- **Sign bit (1 bit):** Always 0 (keeps ID positive)
- **Timestamp (41 bits):** Milliseconds since custom epoch (2023-01-01)
- **Worker ID (10 bits):** Unique identifier for generator instance (0-1023)
- **Sequence (12 bits):** Auto-increment per millisecond (0-4095)

**Capacity:**
- 41 bits of timestamp: ~69 years from epoch
- 10 bits of worker ID: 1,024 unique workers
- 12 bits of sequence: 4,096 IDs per millisecond per worker
- **Total:** ~4.1 million IDs/second per worker

---

## ID Generation Algorithm

### SnowflakeGenerator Class

Main class for generating distributed IDs.

**Algorithm:**
```
SnowflakeGenerator:
  worker_id: integer (10 bits, 0-1023)
  datacenter_id: integer (optional, 5 bits if used)
  sequence: integer = 0
  last_timestamp: integer = -1
  epoch: integer = 1672531200000  // 2023-01-01 00:00:00 UTC in ms
  
  // Bit positions
  WORKER_ID_BITS: integer = 10
  SEQUENCE_BITS: integer = 12
  TIMESTAMP_SHIFT: integer = WORKER_ID_BITS + SEQUENCE_BITS  // 22
  WORKER_ID_SHIFT: integer = SEQUENCE_BITS  // 12
  SEQUENCE_MASK: integer = (1 << SEQUENCE_BITS) - 1  // 4095
  
  function initialize(worker_id):
    if worker_id < 0 or worker_id >= 1024:
      raise error("Worker ID must be between 0 and 1023")
    
    this.worker_id = worker_id
```

### next_id()

Generates the next unique ID.

**Returns:**
- `id` (integer): 64-bit unique ID

**Algorithm:**
```
function next_id():
  // 1. Get current timestamp
  timestamp = current_time_millis()
  
  // 2. Handle clock moving backwards
  if timestamp < last_timestamp:
    offset = last_timestamp - timestamp
    
    if offset > 5:  // More than 5ms backwards
      raise error("Clock moved backwards by " + offset + "ms")
    
    // Small drift: wait it out
    sleep_until(last_timestamp)
    timestamp = current_time_millis()
  
  // 3. Same millisecond: increment sequence
  if timestamp == last_timestamp:
    sequence = (sequence + 1) & SEQUENCE_MASK
    
    // Sequence overflow: wait for next millisecond
    if sequence == 0:
      timestamp = wait_next_millis(last_timestamp)
  
  // 4. New millisecond: reset sequence
  else:
    sequence = 0
  
  // 5. Update last timestamp
  last_timestamp = timestamp
  
  // 6. Compose ID from components
  id = compose_id(timestamp, worker_id, sequence)
  
  return id
```

### compose_id()

Combines components into a single 64-bit ID.

**Parameters:**
- `timestamp` (integer): Milliseconds since epoch
- `worker_id` (integer): Worker identifier (0-1023)
- `sequence` (integer): Sequence number (0-4095)

**Returns:**
- `id` (integer): 64-bit ID

**Algorithm:**
```
function compose_id(timestamp, worker_id, sequence):
  // Shift timestamp to upper 41 bits
  timestamp_component = (timestamp - epoch) << TIMESTAMP_SHIFT
  
  // Shift worker_id to middle 10 bits
  worker_component = worker_id << WORKER_ID_SHIFT
  
  // Sequence in lower 12 bits (no shift needed)
  sequence_component = sequence
  
  // Combine using bitwise OR
  id = timestamp_component | worker_component | sequence_component
  
  return id
```

### decompose_id()

Extracts components from a Snowflake ID.

**Parameters:**
- `id` (integer): 64-bit Snowflake ID

**Returns:**
- `components` (object): {timestamp, worker_id, sequence}

**Algorithm:**
```
function decompose_id(id):
  // Extract timestamp (upper 41 bits)
  timestamp = (id >> TIMESTAMP_SHIFT) + epoch
  
  // Extract worker_id (middle 10 bits)
  worker_id = (id >> WORKER_ID_SHIFT) & ((1 << WORKER_ID_BITS) - 1)
  
  // Extract sequence (lower 12 bits)
  sequence = id & SEQUENCE_MASK
  
  return {
    timestamp: timestamp,
    worker_id: worker_id,
    sequence: sequence,
    datetime: timestamp_to_datetime(timestamp)
  }
```

### Helper Functions

```
function current_time_millis():
  // Returns current time in milliseconds since Unix epoch
  return system_time_in_milliseconds()

function wait_next_millis(last_timestamp):
  // Busy-wait until next millisecond
  timestamp = current_time_millis()
  
  while timestamp <= last_timestamp:
    timestamp = current_time_millis()
  
  return timestamp

function sleep_until(target_timestamp):
  // Sleep until target timestamp is reached
  while current_time_millis() < target_timestamp:
    sleep_microseconds(100)
```

---

## Worker ID Assignment

### Centralized Assignment (etcd)

Uses distributed coordination service for worker ID assignment.

**Algorithm:**
```
WorkerIDManager:
  etcd_client: EtcdClient
  lease_ttl: integer = 10  // seconds
  heartbeat_interval: integer = 3  // seconds
  
  function acquire_worker_id():
    // 1. Acquire distributed lock
    lock_key = "/snowflake/assignment_lock"
    lock = etcd_client.acquire_lock(lock_key, timeout=5)
    
    try:
      // 2. Find first available worker ID
      for worker_id from 0 to 1023:
        key = "/snowflake/workers/" + worker_id
        
        // Try to create key with lease
        lease = etcd_client.create_lease(lease_ttl)
        success = etcd_client.put_if_not_exists(key, {
          hostname: get_hostname(),
          ip: get_ip_address(),
          pid: get_process_id(),
          started_at: current_time()
        }, lease_id=lease.id)
        
        if success:
          // Start heartbeat to keep lease alive
          start_heartbeat_worker(worker_id, lease)
          return worker_id
      
      // All IDs taken
      raise error("No available worker IDs")
    
    finally:
      lock.release()
  
  function start_heartbeat_worker(worker_id, lease):
    // Background thread to keep lease alive
    async_loop:
      sleep(heartbeat_interval)
      
      try:
        etcd_client.keep_alive(lease)
      catch:
        // Lost connection: try to reacquire
        log_error("Lost heartbeat for worker " + worker_id)
        exit_process()  // Force restart and reacquire
```

### Hostname-Based Assignment

Uses deterministic hashing of hostname for worker ID.

**Algorithm:**
```
function get_worker_id_from_hostname():
  hostname = get_hostname()  // e.g., "app-server-042"
  
  // Extract numeric suffix if present
  match = regex_match(hostname, r"(\d+)$")
  if match:
    worker_id = parse_integer(match.group(1)) % 1024
    return worker_id
  
  // Fallback: hash hostname
  hash_value = hash_function(hostname)
  worker_id = hash_value % 1024
  
  return worker_id
```

### IP-Based Assignment

Uses IP address to derive worker ID.

**Algorithm:**
```
function get_worker_id_from_ip():
  ip_address = get_ip_address()  // e.g., "192.168.10.42"
  
  // Parse IP octets
  octets = ip_address.split(".")
  last_octet = parse_integer(octets[3])
  second_last_octet = parse_integer(octets[2])
  
  // Combine last two octets to create 10-bit worker ID
  worker_id = ((second_last_octet % 4) << 8) | last_octet
  worker_id = worker_id % 1024  // Ensure within range
  
  return worker_id
```

---

## Clock Synchronization

### NTP Synchronization Check

Validates that system clock is synchronized with NTP servers.

**Algorithm:**
```
function check_clock_sync():
  // Query NTP server
  ntp_server = "pool.ntp.org"
  ntp_time = query_ntp_server(ntp_server)
  local_time = current_time_millis()
  
  // Calculate drift
  drift = abs(ntp_time - local_time)
  
  // Alert if drift exceeds threshold
  if drift > 5000:  // 5 seconds
    log_critical("Clock drift: " + drift + "ms")
    alert_ops_team("Clock drift detected")
    return false
  
  return true

function query_ntp_server(server):
  // Send NTP packet to server
  socket = create_udp_socket()
  socket.send_to(server, 123, NTP_PACKET)
  
  response = socket.receive(timeout=2000)
  if response is null:
    raise error("NTP server timeout")
  
  // Parse NTP response to get timestamp
  ntp_timestamp = parse_ntp_response(response)
  return ntp_timestamp
```

### monitor_clock_drift()

**Purpose:** Continuously monitor clock drift in background to detect issues early.

**Algorithm:**
```
function monitor_clock_drift():
  // Monitoring metrics
  clock_drift_gauge = create_gauge('snowflake_clock_drift_ms')
  clock_backwards_counter = create_counter('snowflake_clock_backwards_total')
  
  // Run in background thread
  while true:
    try:
      // 1. Get current system time
      system_time = current_time_millis()
      
      // 2. Query NTP server for reference time
      ntp_time = query_ntp_server("pool.ntp.org")
      
      // 3. Calculate drift
      drift = abs(system_time - ntp_time)
      clock_drift_gauge.set(drift)
      
      // 4. Alert based on drift severity
      if drift > 100:  // > 100ms - CRITICAL
        log_critical("CRITICAL clock drift: " + drift + "ms")
        alert_ops_team("Clock drift critical", {
          drift_ms: drift,
          system_time: system_time,
          ntp_time: ntp_time
        })
      else if drift > 10:  // > 10ms - WARNING
        log_warning("Clock drift detected: " + drift + "ms")
      
      // 5. Check for backwards clock movement
      if system_time < this.last_timestamp:
        clock_backwards_counter.increment()
        log_critical("Clock moved backwards!")
        alert_ops_team("Clock moved backwards", {
          current: system_time,
          last: this.last_timestamp,
          difference: this.last_timestamp - system_time
        })
      
      this.last_timestamp = system_time
      
    catch error:
      log_error("Clock drift monitoring error: " + error)
    
    // 6. Check every minute
    sleep(60000)  // 60 seconds
```

**Monitoring Actions:**
- Log drift continuously to metrics system
- Alert if drift > 10ms (warning)
- Alert immediately if drift > 100ms (critical)
- Alert if clock moves backwards
- Ops team can proactively fix NTP sync issues

### Clock Drift Detection

Monitors for clock moving backwards.

**Algorithm:**
```
ClockDriftMonitor:
  last_timestamp: integer = 0
  drift_threshold: integer = 5  // milliseconds
  
  function monitor():
    while true:
      current = current_time_millis()
      
      // Check for backwards movement
      if current < last_timestamp:
        drift = last_timestamp - current
        
        if drift > drift_threshold:
          log_critical("Clock moved backwards: " + drift + "ms")
          alert_ops_team("Clock drift detected")
          
          // Options:
          // 1. Wait it out (< 5ms)
          // 2. Crash and restart (> 5ms)
          // 3. Use last_timestamp + 1 (dangerous)
        
        else:
          // Small drift: wait
          sleep_until(last_timestamp)
      
      last_timestamp = current
      sleep(100)  // Check every 100ms
```

---

## Alternative ID Strategies

### UUID v4 (Random)

128-bit universally unique identifier.

**Algorithm:**
```
function generate_uuid_v4():
  // Generate 16 random bytes
  random_bytes = generate_random_bytes(16)
  
  // Set version (4) and variant bits
  random_bytes[6] = (random_bytes[6] & 0x0F) | 0x40  // Version 4
  random_bytes[8] = (random_bytes[8] & 0x3F) | 0x80  // Variant 10
  
  // Format as UUID string
  uuid = format_as_uuid(random_bytes)
  // e.g., "550e8400-e29b-41d4-a716-446655440000"
  
  return uuid

function format_as_uuid(bytes):
  return hex(bytes[0:4]) + "-" +
         hex(bytes[4:6]) + "-" +
         hex(bytes[6:8]) + "-" +
         hex(bytes[8:10]) + "-" +
         hex(bytes[10:16])
```

**Pros:**
- No coordination needed
- Universally unique across all systems
- Cryptographically random

**Cons:**
- 128 bits (2x larger than Snowflake)
- Not sortable by time
- Random (no indexing benefits)

### ULID (Universally Unique Lexicographically Sortable Identifier)

128-bit time-sortable ID with random component.

**Algorithm:**
```
function generate_ulid():
  // 48-bit timestamp (milliseconds)
  timestamp = current_time_millis()
  
  // 80-bit random component
  random_component = generate_random_bytes(10)
  
  // Combine
  ulid_bytes = timestamp_to_bytes(timestamp, 6) + random_component
  
  // Encode as Crockford Base32 (26 characters)
  ulid = base32_encode(ulid_bytes)
  // e.g., "01ARZ3NDEKTSV4RRFFQ69G5FAV"
  
  return ulid
```

**Pros:**
- Time-sortable (like Snowflake)
- Universally unique (no coordination)
- String format (URL-safe)

**Cons:**
- 128 bits (larger than Snowflake)
- Less compact than integer IDs

### Database Auto-Increment

Uses database sequence for ID generation.

**Algorithm:**
```
// PostgreSQL Example
function generate_db_id():
  result = db.query("SELECT nextval('user_id_seq')")
  return result.value

// With pre-allocation (for performance)
DatabaseIDGenerator:
  current_block_start: integer = 0
  current_block_end: integer = 0
  current_id: integer = 0
  block_size: integer = 1000
  
  function next_id():
    // Allocate new block if exhausted
    if current_id >= current_block_end:
      allocate_new_block()
    
    current_id = current_id + 1
    return current_id
  
  function allocate_new_block():
    // Fetch 1000 IDs at once
    result = db.query(
      "SELECT nextval('user_id_seq') FROM generate_series(1, " + block_size + ")"
    )
    
    current_block_start = result.first()
    current_block_end = result.last()
    current_id = current_block_start
```

**Pros:**
- Simple implementation
- Guaranteed uniqueness
- Sortable by time (roughly)

**Cons:**
- Database dependency (single point of failure)
- Limited throughput (bottleneck)
- Not distributed by default

### Redis INCR

Uses Redis atomic counter for ID generation.

**Algorithm:**
```
function generate_redis_id():
  id = redis.incr("global_id_counter")
  return id

// With sharding (for higher throughput)
function generate_sharded_redis_id():
  // Rotate through Redis shards
  shard_id = current_millisecond() % num_shards
  
  // Each shard generates IDs with different bases
  base_id = redis[shard_id].incr("id_counter")
  
  // Compose: shard_id in upper bits, counter in lower bits
  id = (shard_id << 53) | base_id
  
  return id
```

**Pros:**
- Very fast (in-memory)
- Atomic operations
- Simple to implement

**Cons:**
- Redis dependency
- Not time-sortable
- Need persistence (AOF/RDS) to avoid ID reuse

---

## Batch ID Generation

### generate_batch()

**Purpose:** Generate multiple IDs at once to reduce overhead for bulk operations.

**Parameters:**
- `count`: Number of IDs to generate

**Returns:** Array of unique IDs

**Algorithm:**

```
function generate_batch(count):
  // Pre-allocate array
  ids = new_array(count)
  
  // Acquire lock once (instead of once per ID)
  acquire_lock(id_generation_lock):
    for i from 0 to count - 1:
      // Generate ID without lock overhead
      timestamp = current_time_millis()
      
      // Handle timestamp changes
      if timestamp != last_timestamp:
        sequence = 0
        last_timestamp = timestamp
      else:
        sequence = sequence + 1
        
        // Handle sequence exhaustion within batch
        if sequence > MAX_SEQUENCE:
          // Wait for next millisecond
          timestamp = wait_next_millis(last_timestamp)
          sequence = 0
          last_timestamp = timestamp
      
      // Compose ID
      id = (timestamp << 22) | (worker_id << 12) | sequence
      ids[i] = id
  
  release_lock(id_generation_lock)
  
  return ids
```

**Optimized Version (Lock-Free with Pre-allocation):**

```
function generate_batch_optimized(count):
  ids = new_array(count)
  
  // 1. Check if we can generate all IDs in current millisecond
  timestamp = current_time_millis()
  available_in_current_ms = MAX_SEQUENCE - sequence
  
  if count <= available_in_current_ms and timestamp == last_timestamp:
    // Fast path: All IDs fit in current millisecond
    start_sequence = atomic_add(sequence, count)
    
    for i from 0 to count - 1:
      ids[i] = (timestamp << 22) | (worker_id << 12) | (start_sequence + i)
    
    return ids
  
  // Slow path: Spans multiple milliseconds
  acquire_lock(id_generation_lock):
    for i from 0 to count - 1:
      ids[i] = next_id()
  release_lock(id_generation_lock)
  
  return ids
```

**Example Usage:**

```
// Generate 1000 IDs at once
ids = generate_batch(1000)

// Bulk insert into database
for id in ids:
  database.insert("INSERT INTO users (id, ...) VALUES (?, ...)", id, ...)
```

**Performance Benefits:**
- **Single lock acquisition:** ~10x faster than individual calls
- **Reduced function call overhead:** No per-ID function invocation cost
- **Better cache locality:** Sequential memory access
- **Network efficiency:** Single round-trip for batch requests

**Benchmarks:**
- Individual: 1000 calls × 1μs = 1ms
- Batch: 1 lock + 1000 generations = ~0.1ms (10x faster)

---

## Sequence Exhaustion Handling

### Wait for Next Millisecond

Default strategy: block until next millisecond.

**Algorithm:**
```
function handle_sequence_exhaustion():
  // Busy-wait for next millisecond
  target = last_timestamp + 1
  
  while current_time_millis() < target:
    // Spin-wait (very short sleep)
    sleep_nanoseconds(100)
  
  return current_time_millis()
```

**Pros:**
- Simple
- Guarantees uniqueness

**Cons:**
- Blocks request (adds latency)
- Wastes CPU cycles

### Use Multiple Workers

Alternative: distribute load across multiple worker IDs.

**Algorithm:**
```
SnowflakePool:
  workers: List[SnowflakeGenerator]
  round_robin_index: integer = 0
  
  function initialize(worker_ids):
    for each worker_id in worker_ids:
      workers.append(SnowflakeGenerator(worker_id))
  
  function next_id():
    // Round-robin across workers
    worker = workers[round_robin_index]
    round_robin_index = (round_robin_index + 1) % length(workers)
    
    return worker.next_id()
```

**Pros:**
- No blocking
- Higher throughput (4096 × N per millisecond)

**Cons:**
- Requires multiple worker IDs
- More complex management

---

## Anti-Pattern Examples

### ❌ Ignoring Clock Drift

**Problem:**
```
function next_id_bad():
  timestamp = current_time_millis()
  
  // Just use current timestamp, ignore drift!
  if timestamp == last_timestamp:
    sequence = sequence + 1
  else:
    sequence = 0
  
  last_timestamp = timestamp
  return compose_id(timestamp, worker_id, sequence)

// If clock moves backwards:
// - IDs may not be unique
// - IDs may not be monotonically increasing
```

**Solution:**
```
function next_id_good():
  timestamp = current_time_millis()
  
  // Detect and handle clock moving backwards
  if timestamp < last_timestamp:
    offset = last_timestamp - timestamp
    
    if offset > 5:
      raise error("Clock moved backwards")
    
    // Wait for clock to catch up
    sleep_until(last_timestamp)
    timestamp = current_time_millis()
  
  // Continue normal flow...
```

### ❌ Hardcoded Worker ID

**Problem:**
```
// All instances use same worker ID!
generator = SnowflakeGenerator(worker_id=1)

// Result: ID collisions across multiple servers
```

**Solution:**
```
// Dynamically assign worker ID
worker_id = acquire_worker_id_from_etcd()
generator = SnowflakeGenerator(worker_id)
```

### ❌ Not Handling Sequence Overflow

**Problem:**
```
function next_id_bad():
  sequence = sequence + 1  // No mask!
  
  // sequence becomes 4096, 4097, etc.
  // Overwrites worker_id bits!
  
  return compose_id(timestamp, worker_id, sequence)
```

**Solution:**
```
function next_id_good():
  sequence = (sequence + 1) & SEQUENCE_MASK  // 4096 → 0
  
  if sequence == 0:
    // Wait for next millisecond
    timestamp = wait_next_millis(last_timestamp)
  
  return compose_id(timestamp, worker_id, sequence)
```

---

This pseudocode provides detailed implementations of all algorithms discussed in the main Distributed ID Generator design document.

