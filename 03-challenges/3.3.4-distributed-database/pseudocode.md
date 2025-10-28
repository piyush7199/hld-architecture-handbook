# Distributed Database - Pseudocode Implementations

This document contains detailed algorithm implementations for the Distributed Database system. The main challenge document references these functions.

---

## Table of Contents

1. [Raft Consensus](#1-raft-consensus)
2. [Distributed Transactions (2PC)](#2-distributed-transactions-2pc)
3. [Range Management](#3-range-management)
4. [Timestamp Oracle](#4-timestamp-oracle)
5. [LSM Tree Operations](#5-lsm-tree-operations)
6. [Read Path](#6-read-path)
7. [Metadata Management](#7-metadata-management)
8. [Failure Recovery](#8-failure-recovery)
9. [Online Schema Migration](#9-online-schema-migration)
10. [Monitoring and Chaos Engineering](#10-monitoring-and-chaos-engineering)

---

## 1. Raft Consensus

### raft_append_entries()

**Purpose:** Leader replicates log entries to followers using Raft consensus.

**Parameters:**
- entries: List[LogEntry] - New log entries to replicate
- term: int - Current term number
- leader_commit: int - Leader's commit index

**Returns:**
- bool - True if quorum achieved, False otherwise

**Algorithm:**

```
function raft_append_entries(entries, term, leader_commit):
  // Step 1: Append entries to leader's log
  for entry in entries:
    log.append(entry)
    entry.term = term
    entry.index = log.length()
  
  // Step 2: Send AppendEntries RPC to all followers (parallel)
  acks = []
  for follower in followers:
    rpc = AppendEntriesRPC{
      term: term,
      leader_id: self.node_id,
      prev_log_index: follower.next_index - 1,
      prev_log_term: log[follower.next_index - 1].term,
      entries: entries,
      leader_commit: leader_commit
    }
    
    // Send RPC asynchronously (non-blocking)
    future = send_async(follower, rpc)
    acks.append(future)
  
  // Step 3: Wait for quorum ACKs (2 out of 3)
  quorum_size = floor(cluster_size / 2) + 1  // e.g., 2 for 3 nodes
  acks_received = 1  // Leader counts as 1
  timeout = 1000  // ms
  start_time = now()
  
  while acks_received < quorum_size and (now() - start_time) < timeout:
    for future in acks:
      if future.is_ready():
        response = future.get()
        if response.success and response.term == term:
          acks_received += 1
          follower.next_index = response.match_index + 1
  
  // Step 4: Check if quorum achieved
  if acks_received >= quorum_size:
    // Mark entries as committed
    for entry in entries:
      entry.committed = true
      commit_index = max(commit_index, entry.index)
    
    // Apply to state machine (RocksDB)
    for entry in entries:
      state_machine.apply(entry.key, entry.value)
    
    return true
  else:
    // Quorum not achieved, entries remain uncommitted
    return false
```

**Time Complexity:** O(N) where N = number of followers

**Space Complexity:** O(M) where M = number of entries

**Example Usage:**

```
entries = [
  LogEntry{key: "user123", value: "Alice", operation: "INSERT"},
  LogEntry{key: "user456", value: "Bob", operation: "INSERT"}
]

success = raft_append_entries(entries, term=5, leader_commit=999)
if success:
  return "SUCCESS" to client
else:
  return "TIMEOUT" to client (client retries)
```

---

### raft_request_vote()

**Purpose:** Candidate requests votes from peers during leader election.

**Parameters:**
- candidate_id: int - ID of the candidate requesting votes
- term: int - Candidate's current term
- last_log_index: int - Index of candidate's last log entry
- last_log_term: int - Term of candidate's last log entry

**Returns:**
- vote_granted: bool - True if vote granted, False otherwise

**Algorithm:**

```
function raft_request_vote(candidate_id, term, last_log_index, last_log_term):
  // Step 1: Check term (reject stale candidates)
  if term < current_term:
    return false, current_term
  
  // Step 2: Update term if higher
  if term > current_term:
    current_term = term
    voted_for = null  // Reset vote in new term
    state = FOLLOWER  // Step down if was leader/candidate
  
  // Step 3: Check if already voted in this term
  if voted_for != null and voted_for != candidate_id:
    return false, current_term
  
  // Step 4: Check candidate's log is at least as up-to-date
  my_last_log_index = log.length() - 1
  my_last_log_term = log[my_last_log_index].term
  
  candidate_log_ok = (last_log_term > my_last_log_term) or 
                     (last_log_term == my_last_log_term and last_log_index >= my_last_log_index)
  
  if not candidate_log_ok:
    return false, current_term
  
  // Step 5: Grant vote
  voted_for = candidate_id
  reset_election_timeout()  // Reset timer
  
  return true, current_term
```

**Time Complexity:** O(1)

**Example Usage:**

```
// Candidate requests vote
vote_granted, term = raft_request_vote(
  candidate_id=2,
  term=6,
  last_log_index=1000,
  last_log_term=5
)

if vote_granted:
  votes_received += 1
  if votes_received >= quorum_size:
    become_leader()
```

---

## 2. Distributed Transactions (2PC)

### two_phase_commit()

**Purpose:** Coordinate distributed transaction across multiple shards using Two-Phase Commit.

**Parameters:**
- transaction: Transaction - Transaction object with list of operations
- participants: List[ShardID] - List of shards involved

**Returns:**
- result: TransactionResult - SUCCESS or ABORT with reason

**Algorithm:**

```
function two_phase_commit(transaction, participants):
  coordinator_id = generate_uuid()
  
  // Phase 1: PREPARE
  prepare_results = []
  for shard in participants:
    // Send PREPARE request
    request = PrepareRequest{
      transaction_id: transaction.id,
      coordinator_id: coordinator_id,
      operations: transaction.operations_for_shard(shard),
      timeout: 30000  // 30 seconds
    }
    
    try:
      response = send_rpc(shard, request, timeout=30000)
      prepare_results.append(response)
    catch TimeoutException:
      // Participant didn't respond, must ABORT
      prepare_results.append(PrepareResponse{vote: NO, reason: "timeout"})
  
  // Check if all voted YES
  all_yes = true
  for result in prepare_results:
    if result.vote != YES:
      all_yes = false
      abort_reason = result.reason
      break
  
  // Phase 2: COMMIT or ABORT
  if all_yes:
    // COMMIT phase
    for shard in participants:
      request = CommitRequest{
        transaction_id: transaction.id,
        coordinator_id: coordinator_id
      }
      
      // Retry COMMIT forever (idempotent)
      retry_count = 0
      while true:
        try:
          response = send_rpc(shard, request, timeout=30000)
          if response.status == COMMITTED:
            break
        catch TimeoutException:
          retry_count += 1
          sleep(1000 * min(retry_count, 10))  // Exponential backoff
          // Continue retrying (participant will eventually respond)
    
    // Log commit success
    coordinator_log.write(LogEntry{
      transaction_id: transaction.id,
      status: COMMITTED,
      timestamp: now()
    })
    
    return TransactionResult{status: SUCCESS}
  
  else:
    // ABORT phase
    for shard in participants:
      request = AbortRequest{
        transaction_id: transaction.id,
        coordinator_id: coordinator_id,
        reason: abort_reason
      }
      
      // Best effort ABORT (don't retry forever)
      try:
        send_rpc(shard, request, timeout=5000)
      catch TimeoutException:
        // Participant will timeout and abort on its own
        pass
    
    // Log abort
    coordinator_log.write(LogEntry{
      transaction_id: transaction.id,
      status: ABORTED,
      reason: abort_reason,
      timestamp: now()
    })
    
    return TransactionResult{status: ABORT, reason: abort_reason}
```

**Time Complexity:** O(N) where N = number of participants

**Example Usage:**

```
transaction = Transaction{
  id: "txn-123",
  operations: [
    Operation{shard: 1, type: UPDATE, key: "accountA", value: "balance-50"},
    Operation{shard: 2, type: UPDATE, key: "accountB", value: "balance+50"}
  ]
}

result = two_phase_commit(transaction, participants=[1, 2])
if result.status == SUCCESS:
  return "Transaction committed"
else:
  return "Transaction aborted: " + result.reason
```

---

## 3. Range Management

### split_range()

**Purpose:** Split a hot or oversized range into two ranges.

**Parameters:**
- range_id: int - ID of range to split
- split_key: bytes - Key to split at (median key)

**Returns:**
- new_range_id: int - ID of newly created range

**Algorithm:**

```
function split_range(range_id, split_key):
  // Step 1: Lock range for split (reads/writes continue)
  range = get_range(range_id)
  range.status = SPLITTING
  
  // Step 2: Calculate split metadata
  old_start_key = range.start_key
  old_end_key = range.end_key
  
  new_range_left = Range{
    range_id: range_id,  // Keep original ID
    start_key: old_start_key,
    end_key: split_key,
    leader_node: range.leader_node,  // Keep original leader
    replicas: range.replicas
  }
  
  new_range_right = Range{
    range_id: generate_new_range_id(),
    start_key: split_key,
    end_key: old_end_key,
    leader_node: allocate_new_leader_node(),  // Different nodes
    replicas: allocate_new_replica_nodes(count=3)
  }
  
  // Step 3: Create new Raft group for right range
  for node in new_range_right.replicas:
    send_rpc(node, CreateRaftGroupRequest{
      range_id: new_range_right.range_id,
      start_key: new_range_right.start_key,
      end_key: new_range_right.end_key
    })
  
  // Step 4: Copy data [split_key...end_key] to new group (background)
  copy_worker = BackgroundWorker()
  copy_worker.start(task=copy_data, args={
    source_range: range_id,
    dest_range: new_range_right.range_id,
    start_key: split_key,
    end_key: old_end_key
  })
  
  // Step 5: Dual-write phase (writes go to both ranges)
  range.dual_write = true
  range.dual_write_target = new_range_right.range_id
  
  // Wait for copy to complete
  while not copy_worker.is_complete():
    sleep(1000)
    progress = copy_worker.get_progress()
    log.info("Split progress: " + progress + "%")
  
  // Step 6: Update metadata (atomic swap)
  etcd.transaction(
    delete(range_metadata_key(range_id)),
    put(range_metadata_key(new_range_left.range_id), serialize(new_range_left)),
    put(range_metadata_key(new_range_right.range_id), serialize(new_range_right))
  )
  
  // Step 7: Delete old data from left range
  delete_keys_in_range(range_id, start_key=split_key, end_key=old_end_key)
  
  // Step 8: Mark split complete
  new_range_left.status = ACTIVE
  new_range_right.status = ACTIVE
  
  // Notify gateways to invalidate cache
  broadcast_metadata_change(ranges=[new_range_left.range_id, new_range_right.range_id])
  
  return new_range_right.range_id
```

**Time Complexity:** O(M) where M = amount of data to copy

**Example Usage:**

```
// Monitor detects hotspot
if range_qps > 10000 and duration > 60_seconds:
  median_key = calculate_median_key(range_id)
  new_range_id = split_range(range_id, median_key)
  log.info("Split complete: " + range_id + " → " + range_id + " + " + new_range_id)
```

---

### detect_hotspot()

**Purpose:** Monitor ranges and detect hotspots that need splitting.

**Parameters:**
- None (runs continuously as background thread)

**Returns:**
- None (triggers split_range() when hotspot detected)

**Algorithm:**

```
function detect_hotspot():
  while true:
    // Step 1: Query metrics for all ranges
    ranges = get_all_ranges()
    
    for range in ranges:
      // Step 2: Get metrics from monitoring system
      metrics = get_range_metrics(range.range_id, window=60_seconds)
      
      qps = metrics.queries_per_second
      size_mb = metrics.size_mb
      cpu_percent = metrics.cpu_percent
      
      // Step 3: Check split thresholds
      should_split = false
      reason = ""
      
      if qps > 10000 and metrics.duration > 60_seconds:
        should_split = true
        reason = "QPS hotspot: " + qps + " QPS"
      
      elif size_mb > 64:
        should_split = true
        reason = "Size limit: " + size_mb + " MB"
      
      elif cpu_percent > 80 and metrics.duration > 300_seconds:
        should_split = true
        reason = "CPU hotspot: " + cpu_percent + "%"
      
      // Step 4: Trigger split
      if should_split:
        log.warn("Hotspot detected on range " + range.range_id + ": " + reason)
        
        // Calculate split key (median key in range)
        split_key = calculate_median_key(range.range_id)
        
        // Trigger split (async)
        async_split_range(range.range_id, split_key)
    
    // Sleep before next check
    sleep(10_seconds)
```

**Time Complexity:** O(N) where N = number of ranges

**Example Usage:**

```
// Start monitoring thread
monitor_thread = Thread(target=detect_hotspot)
monitor_thread.start()

// Monitor runs continuously in background
```

---

## 4. Timestamp Oracle

### get_timestamp()

**Purpose:** Get globally unique, monotonically increasing timestamp for transaction.

**Parameters:**
- None

**Returns:**
- timestamp: int - Unique 64-bit timestamp

**Algorithm:**

```
function get_timestamp():
  // Step 1: Acquire lock (oracle is single-threaded for simplicity)
  lock.acquire()
  
  try:
    // Step 2: Get physical time (milliseconds since epoch)
    physical_time = now_millis()
    
    // Step 3: Check if time moved backwards (NTP adjustment)
    if physical_time < last_physical_time:
      // Clock went backwards! Wait until it catches up
      sleep_ms = last_physical_time - physical_time + 1
      sleep(sleep_ms)
      physical_time = now_millis()
    
    // Step 4: Increment logical counter
    if physical_time == last_physical_time:
      // Same millisecond, increment logical counter
      logical_counter += 1
      
      // Check overflow (max 16 bits = 65535)
      if logical_counter > 65535:
        // Wait for next millisecond
        sleep(1)
        physical_time = now_millis()
        logical_counter = 0
    else:
      // New millisecond, reset logical counter
      logical_counter = 0
      last_physical_time = physical_time
    
    // Step 5: Combine into 64-bit timestamp
    // [48 bits: physical time][16 bits: logical counter]
    timestamp = (physical_time << 16) | logical_counter
    
    // Step 6: Log to Raft for durability (async)
    raft_log_async(LogEntry{
      type: TIMESTAMP_ISSUED,
      timestamp: timestamp,
      term: current_term
    })
    
    return timestamp
  
  finally:
    lock.release()
```

**Time Complexity:** O(1)

**Space Complexity:** O(1)

**Example Usage:**

```
// Transaction begins
txn_start_ts = get_timestamp()  // e.g., 1640000000000001

// All writes in transaction tagged with this timestamp
write(key="X", value="Y", timestamp=txn_start_ts)

// Read at specific timestamp
value = read(key="X", timestamp=txn_start_ts)
```

---

## 5. LSM Tree Operations

### lsm_tree_write()

**Purpose:** Write key-value pair to LSM tree (RocksDB).

**Parameters:**
- key: bytes - Key to write
- value: bytes - Value to write
- timestamp: int - MVCC timestamp

**Returns:**
- None (write is durable after WAL + memtable)

**Algorithm:**

```
function lsm_tree_write(key, value, timestamp):
  // Step 1: Write to WAL (Write-Ahead Log) for durability
  wal_entry = WALEntry{
    key: key,
    value: value,
    timestamp: timestamp,
    checksum: crc32(key + value)
  }
  wal.append(wal_entry)
  wal.sync()  // fsync() to disk
  
  // Step 2: Write to Memtable (in-memory sorted tree)
  versioned_key = VersionedKey{
    key: key,
    timestamp: timestamp
  }
  
  memtable.put(versioned_key, value)
  
  // Step 3: Check if Memtable is full (64 MB)
  if memtable.size() >= 64_MB:
    // Freeze current memtable
    frozen_memtable = memtable
    memtable = new_memtable()
    
    // Flush to SSTable (background thread)
    flush_thread = Thread(target=flush_memtable_to_sstable, args=frozen_memtable)
    flush_thread.start()
  
  // Return immediately (write is durable in WAL)
  return
```

**Time Complexity:** O(log M) where M = memtable size (typically <1ms)

**Example Usage:**

```
// Write key-value pair
lsm_tree_write(key="user123", value="Alice", timestamp=1640000000001)

// Write is durable immediately (in WAL + memtable)
```

---

### lsm_tree_read()

**Purpose:** Read value for key at specific timestamp (MVCC).

**Parameters:**
- key: bytes - Key to read
- timestamp: int - MVCC timestamp to read at

**Returns:**
- value: bytes - Value at timestamp, or None if not found

**Algorithm:**

```
function lsm_tree_read(key, timestamp):
  // Step 1: Check memtable (most recent writes)
  versioned_key = VersionedKey{key: key, timestamp: timestamp}
  
  value = memtable.get(versioned_key)
  if value != None:
    return value
  
  // Step 2: Check frozen memtables (being flushed)
  for frozen_memtable in frozen_memtables:
    value = frozen_memtable.get(versioned_key)
    if value != None:
      return value
  
  // Step 3: Check SSTables (disk)
  // Level 0: 4 SSTables (overlapping keys)
  for sstable in level0_sstables:
    // Check bloom filter first (probabilistic skip)
    if not sstable.bloom_filter.contains(key):
      continue
    
    value = sstable.get(versioned_key)
    if value != None:
      return value
  
  // Level 1+: No overlapping keys, binary search
  for level in [1, 2, 3, 4]:
    sstable = find_sstable_containing_key(level, key)
    if sstable != None:
      value = sstable.get(versioned_key)
      if value != None:
        return value
  
  // Not found
  return None
```

**Time Complexity:** O(log N) where N = number of SSTables (typically 2-5ms)

**Example Usage:**

```
// Read at specific timestamp
value = lsm_tree_read(key="user123", timestamp=1640000000001)
// Returns: "Alice"

// Read at earlier timestamp
value = lsm_tree_read(key="user123", timestamp=1640000000000)
// Returns: None (not yet written at that time)
```

---

## 6. Read Path

### linearizable_read()

**Purpose:** Read with linearizability guarantee (latest committed value).

**Parameters:**
- key: bytes - Key to read

**Returns:**
- value: bytes - Latest committed value

**Algorithm:**

```
function linearizable_read(key):
  // Step 1: Check if I am leader
  if self.state != LEADER:
    // Forward to leader
    return forward_to_leader(key)
  
  // Step 2: Ensure I am still leader (prevent stale reads from old leader)
  // Read index: Record commit_index, send heartbeat, wait for quorum ACK
  read_index = commit_index
  
  heartbeat_acks = send_heartbeat_to_followers()
  if heartbeat_acks < quorum_size:
    // Lost leadership, reject read
    return ERROR("Not leader, retry")
  
  // Step 3: Wait for apply_index to catch up to read_index
  // (Ensures all committed writes are applied to state machine)
  while apply_index < read_index:
    sleep(1)  // Wait for apply thread
  
  // Step 4: Read from state machine (RocksDB)
  value = lsm_tree_read(key, timestamp=latest)
  
  return value
```

**Time Complexity:** O(1) for leader, O(RTT) if forwarded to leader

**Example Usage:**

```
// Read latest value (linearizable)
value = linearizable_read(key="user123")
// Guarantees: Returns latest committed value
```

---

## 7. Metadata Management

### lookup_range_for_key()

**Purpose:** Find which range owns a given key (used by gateway for routing).

**Parameters:**
- key: bytes - Key to lookup
- use_cache: bool - Whether to use local cache (default: True)

**Returns:**
- range: Range - Range object with leader_node, start_key, end_key

**Algorithm:**

```
function lookup_range_for_key(key, use_cache=true):
  // Step 1: Check local cache (99% hit rate)
  if use_cache:
    cached_range = cache.get(key)
    if cached_range != None and cached_range.ttl > now():
      return cached_range
  
  // Step 2: Cache miss, query etcd
  ranges = etcd.get(key="/ranges/", range_query=true)
  
  // Binary search to find range containing key
  left = 0
  right = ranges.length() - 1
  
  while left <= right:
    mid = (left + right) / 2
    range = ranges[mid]
    
    if key < range.start_key:
      right = mid - 1
    elif key >= range.end_key:
      left = mid + 1
    else:
      // Found! key is in [range.start_key, range.end_key)
      
      // Cache for future lookups
      cache.put(key, range, ttl=60_seconds)
      
      return range
  
  // Key not found (should never happen)
  return ERROR("Key not in any range")
```

**Time Complexity:** O(1) cache hit, O(log N) cache miss

**Example Usage:**

```
key = "user123"
range = lookup_range_for_key(key)

// Route query to range's leader
leader_node = range.leader_node
response = send_query_to_node(leader_node, key)
```

---

## 8. Failure Recovery

### recover_from_wal()

**Purpose:** Recover node state from Write-Ahead Log after crash.

**Parameters:**
- wal_path: str - Path to WAL file on disk

**Returns:**
- recovered_state: State - Recovered state machine state

**Algorithm:**

```
function recover_from_wal(wal_path):
  state = new_state_machine()
  
  // Step 1: Open WAL file
  wal_file = open(wal_path, mode="read")
  
  // Step 2: Read all entries sequentially
  entry_count = 0
  last_committed_index = 0
  
  while not wal_file.eof():
    entry = wal_file.read_entry()
    
    // Verify checksum
    if not verify_checksum(entry):
      log.error("Corrupt WAL entry at position " + entry_count)
      break
    
    // Replay entry if committed
    if entry.committed:
      state.apply(entry.key, entry.value, entry.timestamp)
      last_committed_index = entry.index
    
    entry_count += 1
  
  // Step 3: Close WAL
  wal_file.close()
  
  // Step 4: Truncate WAL (remove uncommitted entries)
  wal_file = open(wal_path, mode="write")
  wal_file.truncate(last_committed_index)
  wal_file.close()
  
  log.info("Recovered from WAL: " + entry_count + " entries, committed up to " + last_committed_index)
  
  return state
```

**Time Complexity:** O(N) where N = number of WAL entries

**Example Usage:**

```
// Node restarts after crash
state = recover_from_wal("/var/lib/db/wal/node-1.log")

// Resume normal operation
node.state_machine = state
node.rejoin_raft_cluster()
```

---

## 9. Online Schema Migration

### online_schema_change()

**Purpose:** Add column to table without downtime (multi-phase migration).

**Parameters:**
- table_name: str - Table to modify
- new_column: Column - New column definition

**Returns:**
- None (migration completes in background)

**Algorithm:**

```
function online_schema_change(table_name, new_column):
  // Phase 1: Write Both (Week 1)
  log.info("Phase 1: Write Both - Starting")
  
  schema_version_new = schema_version + 1
  
  // Update schema metadata (add new column)
  schema_new = get_schema(table_name)
  schema_new.add_column(new_column)
  schema_new.version = schema_version_new
  
  etcd.put("/schema/" + table_name + "/version_" + schema_version_new, serialize(schema_new))
  
  // Deploy new code: Writes include new column
  // Old rows: (user_id, name)
  // New rows: (user_id, name, email)
  // Reads: Still return old schema (backwards compatible)
  
  wait_for_deployment()
  log.info("Phase 1 complete")
  
  // Phase 2: Backfill (Week 2-4)
  log.info("Phase 2: Backfill - Starting")
  
  worker = BackgroundWorker()
  worker.start(task=backfill_column, args={
    table_name: table_name,
    column: new_column,
    batch_size: 10000
  })
  
  // Monitor progress
  while not worker.is_complete():
    sleep(60_seconds)
    progress = worker.get_progress()
    log.info("Backfill progress: " + progress + "%")
  
  log.info("Phase 2 complete")
  
  // Phase 3: Read New (Week 5)
  log.info("Phase 3: Read New - Starting")
  
  // Gradual rollout: 10% → 50% → 100% of reads use new schema
  for percentage in [10, 50, 100]:
    etcd.put("/schema/" + table_name + "/read_new_percentage", percentage)
    wait_for_validation(duration=1_day)
  
  log.info("Phase 3 complete")
  
  // Phase 4: Cleanup (Week 6)
  log.info("Phase 4: Cleanup - Starting")
  
  // Drop old schema version
  etcd.delete("/schema/" + table_name + "/version_" + schema_version)
  
  // Reclaim storage space (compaction)
  trigger_compaction(table_name)
  
  log.info("Phase 4 complete - Migration done!")
```

**Time Complexity:** O(N) where N = number of rows (backfill phase)

**Example Usage:**

```
// Add email column to users table
new_column = Column{
  name: "email",
  type: VARCHAR(255),
  nullable: true,
  default: null
}

online_schema_change(table_name="users", new_column=new_column)
// Migration runs in background (4-6 weeks for billion rows)
```

---

## 10. Monitoring and Chaos Engineering

### chaos_kill_leader()

**Purpose:** Chaos engineering test - kill leader node and verify automatic failover.

**Parameters:**
- range_id: int - Range to test

**Returns:**
- failover_duration_ms: int - Time taken for new leader election

**Algorithm:**

```
function chaos_kill_leader(range_id):
  // Step 1: Identify current leader
  range = get_range(range_id)
  old_leader_node = range.leader_node
  
  log.info("Chaos test: Killing leader " + old_leader_node + " for range " + range_id)
  
  // Step 2: Kill leader process
  start_time = now_millis()
  
  send_rpc(old_leader_node, KillProcessRequest{signal: SIGKILL})
  
  // Step 3: Monitor for new leader election
  new_leader_node = None
  timeout = 5000  // 5 seconds max
  
  while (now_millis() - start_time) < timeout:
    sleep(100)
    
    // Query etcd for new leader
    range = get_range(range_id)
    if range.leader_node != old_leader_node:
      new_leader_node = range.leader_node
      break
  
  if new_leader_node == None:
    log.error("Chaos test FAILED: No new leader elected within 5 seconds")
    return -1
  
  // Step 4: Verify new leader is functional
  failover_duration_ms = now_millis() - start_time
  
  // Send test write to new leader
  try:
    response = send_write_to_leader(new_leader_node, key="chaos_test", value="success")
    if response.status == SUCCESS:
      log.info("Chaos test PASSED: New leader " + new_leader_node + " elected in " + failover_duration_ms + "ms")
    else:
      log.error("Chaos test FAILED: New leader not functional")
  catch:
    log.error("Chaos test FAILED: New leader unreachable")
  
  // Step 5: Restart old leader (for cleanup)
  send_rpc(old_leader_node, RestartProcessRequest{})
  
  return failover_duration_ms
```

**Time Complexity:** O(1)

**Example Usage:**

```
// Run chaos test on range 1
failover_ms = chaos_kill_leader(range_id=1)

// Expected: 1000-2000ms (1-2 seconds)
assert failover_ms < 3000, "Failover too slow!"
```

---

### hybrid_logical_clock()

**Purpose:** Generate monotonic timestamp even with clock skew (HLC).

**Parameters:**
- physical_time: int - Current physical time (milliseconds)
- received_timestamp: int - Timestamp from remote node (for sync)

**Returns:**
- hlc_timestamp: int - Hybrid Logical Clock timestamp

**Algorithm:**

```
function hybrid_logical_clock(physical_time, received_timestamp=None):
  // HLC = (physical_time, logical_counter)
  // Stored as 64-bit: [48 bits physical][16 bits logical]
  
  if received_timestamp != None:
    // Received timestamp from remote node (e.g., in RPC)
    remote_physical = received_timestamp >> 16
    remote_logical = received_timestamp & 0xFFFF
    
    // Update local clock to max of local and remote
    if remote_physical > physical_time:
      // Remote is ahead, adopt their physical time
      local_physical = remote_physical
      local_logical = remote_logical + 1
    elif remote_physical == physical_time:
      // Same physical time, increment logical counter
      local_physical = physical_time
      local_logical = max(hlc_logical_counter, remote_logical) + 1
    else:
      // Local is ahead, use local time
      local_physical = physical_time
      local_logical = hlc_logical_counter + 1
  else:
    // No received timestamp, just generate local HLC
    if physical_time > hlc_physical_time:
      // Physical time advanced, reset logical counter
      local_physical = physical_time
      local_logical = 0
    else:
      // Same physical time, increment logical counter
      local_physical = hlc_physical_time
      local_logical = hlc_logical_counter + 1
  
  // Update global HLC state
  hlc_physical_time = local_physical
  hlc_logical_counter = local_logical
  
  // Combine into 64-bit timestamp
  hlc_timestamp = (local_physical << 16) | local_logical
  
  return hlc_timestamp
```

**Time Complexity:** O(1)

**Example Usage:**

```
// Node 1: Generate HLC
node1_time = hybrid_logical_clock(physical_time=1640000000000, received_timestamp=None)
// Returns: 1640000000000 << 16 | 0 = 1640000000000:0

// Node 2: Receive message from Node 1, sync clocks
node2_time = hybrid_logical_clock(physical_time=1639999999000, received_timestamp=node1_time)
// Returns: 1640000000000 << 16 | 1 = 1640000000000:1
// (Adopted Node 1's time + incremented logical)
```

