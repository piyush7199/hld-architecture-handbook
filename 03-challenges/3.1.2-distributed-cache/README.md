# 3.1.2 Design a Distributed Cache (Redis/Memcached)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions. 
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, consistent hashing, replication, eviction policies, and scaling
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows for cache operations, failover, and monitoring
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices and trade-offs
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for all core functions

---

## Problem Statement

Design a highly available, horizontally scalable, distributed in-memory cache system similar to Redis or Memcached. The
system should support millions of operations per second with sub-millisecond latency while providing fault tolerance and
minimal data reshuffling when nodes are added or removed.

---

## 1. Requirements and Scale Estimation

### Functional Requirements (FRs)

| Requirement                | Description                                                               | Priority     |
|----------------------------|---------------------------------------------------------------------------|--------------|
| **Key-Value Operations**   | Support `GET(key)`, `PUT(key, value)`, `DELETE(key)` with O(1) complexity | Must Have    |
| **Time-to-Live (TTL)**     | Keys must expire after a specified duration automatically                 | Must Have    |
| **Horizontal Scalability** | Add/remove cache nodes dynamically without downtime                       | Must Have    |
| **Data Types**             | Support strings, lists, sets, sorted sets, hashes                         | Should Have  |
| **Atomic Operations**      | Support atomic increment/decrement, compare-and-swap                      | Should Have  |
| **Persistence (Optional)** | Option to persist data to disk for durability                             | Nice to Have |

### Non-Functional Requirements (NFRs)

| Requirement             | Target                                     | Rationale                                     |
|-------------------------|--------------------------------------------|-----------------------------------------------|
| **Low Latency**         | < 1 ms (p99)                               | In-memory operations must be fast             |
| **High Throughput**     | > 100K QPS per node                        | Support high-traffic applications             |
| **High Availability**   | 99.99% uptime                              | Single node failure should not affect service |
| **Minimal Reshuffling** | < 1% keys moved when adding/removing nodes | Preserve cache hit ratio                      |
| **Memory Efficiency**   | < 20% metadata overhead                    | Maximize data storage                         |

---

### Scale Estimation

#### Scenario: Large E-commerce Platform

| Metric                | Assumption                                 | Calculation                 | Result                 |
|-----------------------|--------------------------------------------|-----------------------------|------------------------|
| **Total Data**        | 10% of 1.8 TB database in cache            | 1.8 TB × 0.1                | **~180 GB RAM needed** |
| **Average Key Size**  | Key: 64 bytes, Value: 1 KB                 | 64 B + 1 KB                 | ~1 KB per entry        |
| **Total Keys**        | 180 GB / 1 KB                              | 180 GB / 1 KB               | **~180 million keys**  |
| **Peak Read QPS**     | 80% of DB reads (from 1157 QPS baseline)   | 1157 × 0.8 × 5 (burst)      | **~4,600 reads/sec**   |
| **Peak Write QPS**    | 20% of reads (cache invalidation)          | 4,600 × 0.2                 | **~920 writes/sec**    |
| **Network Bandwidth** | 1 KB × 4,600 QPS                           | 1 KB × 4,600                | **~4.6 MB/sec**        |
| **Nodes Required**    | 180 GB / 64 GB per node (with replication) | 180 GB / 64 GB × 3 replicas | **~9 nodes**           |

#### Back-of-the-Envelope Calculations

```
Memory per Node = 64 GB RAM
Replication Factor = 3 (1 primary + 2 replicas)
Total Nodes = (180 GB / 64 GB) × 3 = ~9 nodes

QPS per Node = 4,600 QPS / 3 primaries = ~1,533 QPS/node
This is well within Redis capacity (100K+ QPS per node)

Network: 4.6 MB/sec is negligible (1 Gbps = 125 MB/sec)
```

---

## 2. High-Level Architecture

> 📊 **See detailed architecture diagrams:** [HLD Diagrams](./hld-diagram.md)

### Key Components

| Component               | Responsibility                                  | Technology Options           |
|-------------------------|-------------------------------------------------|------------------------------|
| **Client Library**      | Consistent hashing, routing, connection pooling | jedis, lettuce, go-redis     |
| **Cache Node**          | In-memory key-value storage                     | Redis, Memcached, KeyDB      |
| **Replication**         | Data redundancy, failover                       | Master-Replica (async/sync)  |
| **Cluster Manager**     | Health monitoring, failover orchestration       | Redis Sentinel, Cluster mode |
| **Configuration Store** | Node discovery, cluster topology                | etcd, Consul, ZooKeeper      |

---

## 3. Detailed Component Design

### 3.1 Data Partitioning Strategy

> 📊 **See visual representation:** [Consistent Hashing Ring Diagram](./hld-diagram.md#consistent-hashing-ring)

#### Why Consistent Hashing?

**Problem with Simple Modulo Hashing:**

```
// Simple hash: node = hash(key) % N
// If N = 3 nodes:
node = hash("user:123") % 3  // Goes to node 1

// Add 1 node (N = 4):
node = hash("user:123") % 4  // Now goes to node 3! ❌

// Result: ~75% of keys remapped → cache invalidation cascade!
```

**Solution: Consistent Hashing**

> 📊 **See implementation details:** [Consistent Hashing Addition Sequence](./sequence-diagrams.md#consistent-hashing-node-addition)

```
// Consistent Hashing with Virtual Nodes
ConsistentHash:
  ring: HashMap
  sorted_keys: SortedList
  virtual_nodes: integer = 150
  
  function initialize(nodes, virtual_nodes=150):
    ring = empty_hashmap
    sorted_keys = empty_list
    this.virtual_nodes = virtual_nodes
    
    for each node in nodes:
      add_node(node)
  
  function add_node(node):
    for i from 0 to virtual_nodes:
      virtual_key = node + ":" + i
      hash_value = hash_function(virtual_key)
      ring[hash_value] = node
      sorted_keys.append(hash_value)
    
    sorted_keys.sort()
  
  function get_node(key):
    if ring is empty:
      return null
    
    hash_value = hash_function(key)
    
    // Binary search for the next node on the ring
    idx = binary_search(sorted_keys, hash_value)
    if idx == length(sorted_keys):
      idx = 0
    
    return ring[sorted_keys[idx]]
  
  function hash_function(key):
    return md5_hash(key) as integer

// Usage
cache = ConsistentHash(['node1', 'node2', 'node3'])
node = cache.get_node('user:123')  // Returns 'node2'

// Add node4:
cache.add_node('node4')
node = cache.get_node('user:123')  // Still likely 'node2'
// Only ~25% of keys remapped instead of 75%!
```

#### Virtual Nodes (VNodes) Impact

| VNodes per Node | Distribution Uniformity   | Lookup Overhead | Recommendation     |
|-----------------|---------------------------|-----------------|--------------------|
| 10-50           | Poor (±20% imbalance)     | Very Low        | Not recommended    |
| 100-200         | Good (±5% imbalance)      | Low             | **Recommended**    |
| 500-1000        | Excellent (±1% imbalance) | Medium          | For large clusters |
| 5000+           | Perfect                   | High            | Overkill           |

---

### 3.2 Replication Strategy

> 📊 **See replication flows:** [Replication Sequence Diagrams](./sequence-diagrams.md#replication-flow)

#### Replication Options Comparison

| Strategy              | Latency         | Data Loss Risk      | Availability | Best For                          |
|-----------------------|-----------------|---------------------|--------------|-----------------------------------|
| **Async Replication** | Very Low (<1ms) | Low (1-5ms of data) | High         | **Recommended for cache**         |
| **Sync Replication**  | High (2-10ms)   | None                | Medium       | Financial systems (not cache)     |
| **Semi-Sync**         | Medium (1-5ms)  | Very Low            | High         | Balance consistency & performance |
| **No Replication**    | Lowest          | Complete on failure | Low          | Dev/test only                     |

**Why Async for Cache?**

- Cache is **not the source of truth** (database is)
- Sub-millisecond writes critical for performance
- 1-5ms of data loss acceptable (cache can be repopulated from DB)
- Losing 5ms of cached writes ≠ losing database writes

---

### 3.3 Write Policies

> 📊 **See write pattern flows:** [Cache Pattern Sequence Diagrams](./sequence-diagrams.md)

#### Cache-Aside (Lazy Loading) - Recommended for Distributed Cache

**Read Flow:**
1. Check cache first
2. **Cache Hit:** Return immediately (fast)
3. **Cache Miss:** Fetch from database, populate cache with TTL, return data

**Write Flow:**
1. Update database (source of truth)
2. Invalidate cache (don't update)
3. Next read will fetch fresh data

*See `pseudocode.md::get_from_cache_aside()` and `pseudocode.md::update_with_cache_aside()` for implementation*

**Why Cache-Aside Over Write-Through?**

| Aspect            | Cache-Aside                  | Write-Through                             |
|-------------------|------------------------------|-------------------------------------------|
| **Write Latency** | Low (DB only)                | High (DB + Cache sync)                    |
| **Cache Size**    | Smaller (only accessed data) | Larger (all written data)                 |
| **Consistency**   | Eventual                     | Strong                                    |
| **Best For**      | **Read-heavy workloads**     | Write-heavy, immediate consistency needed |

---

### 3.4 Eviction Policies

> 📊 **See eviction policy comparison:** [Eviction Policies Diagram](./hld-diagram.md#cache-eviction-policies)

#### LRU (Least Recently Used) Implementation

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
  
  function get(key):
    if key exists in cache:
      node = cache[key]
      remove_node(node)
      add_to_head(node)
      return node.value
    return null
  
  function put(key, value):
    if key exists in cache:
      remove_node(cache[key])
    
    node = Node(key, value)
    add_to_head(node)
    cache[key] = node
    
    if size(cache) > capacity:
      // Evict LRU (tail)
      lru = tail.prev
      remove_node(lru)
      delete cache[lru.key]

Node:
  key: any
  value: any
  prev: Node
  next: Node
```

#### Eviction Policy Comparison

| Policy     | Best For               | Pros                  | Cons                       | Redis Support                    |
|------------|------------------------|-----------------------|----------------------------|----------------------------------|
| **LRU**    | General purpose        | Good hit rate, simple | Ignores frequency          | ✅ `maxmemory-policy allkeys-lru` |
| **LFU**    | Stable access patterns | Keeps hot data        | Complex, cold start issues | ✅ `allkeys-lfu`                  |
| **TTL**    | Time-sensitive data    | Predictable expiry    | May evict useful data      | ✅ `volatile-ttl`                 |
| **Random** | Low overhead           | Very simple           | Poor hit rate              | ✅ `allkeys-random`               |

---

### 3.5 TTL (Time-to-Live) Management

#### Implementation Strategies

**Option 1: Active Expiration (Lazy Delete)**

Check TTL when key is accessed. If expired, delete and return null. Simple, no background process needed.

**Option 2: Passive Expiration (Background Scan)**

Background process samples 100 random keys every 100ms, deletes expired ones. Cleans up unused keys but uses CPU.

*See `pseudocode.md::TTL Management` for detailed implementations*

**Redis Approach: Hybrid**

- Active: Check on access
- Passive: Sample 20 keys every 100ms, delete expired
- Probabilistic: If > 25% expired, repeat immediately

---

### 3.6 Concurrency Control: Preventing Cache Stampede

> 📊 **See stampede prevention solutions:** [Cache Stampede Prevention Diagrams](./sequence-diagrams.md#cache-stampede-prevention)

#### Problem: Cache Stampede / Thundering Herd

```
Hot Key Expires
        │
        ├─ Request 1 ──┐
        ├─ Request 2 ──┼─▶ All hit DB simultaneously
        ├─ Request 3 ──┤
        ├─ ...         │
        └─ Request 10K ┘

Result: Database overwhelmed, cascading failure
```

#### Solution 1: Single Flight Request (Recommended)

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
      return value

**Example:** 10,000 concurrent requests for same user → Only 1 DB query, other 9,999 wait and read from cache.

*See `pseudocode.md::SingleFlightCache` for implementation*

#### Solution 2: Probabilistic Early Expiration

Refresh cache proactively before TTL expires. If TTL < 10% remaining, probability increases to trigger async background refresh. Cache never goes cold.
  
  return value
```

---

## 4. Availability and Fault Tolerance

> 📊 **See failover process:** [Failover Sequence Diagram](./sequence-diagrams.md#automatic-failover-with-sentinel)

### 4.1 Failover Strategy

#### Automatic Failover with Sentinel

**Configuration:**

```conf
# Redis Sentinel configuration
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000

# Quorum = 2 (need 2 sentinels to agree master is down)
```

**Failover Process:**

1. **Detection** (5s): Sentinel detects master unresponsive
2. **Quorum Vote** (1s): Sentinels vote if master is really down
3. **Leader Election** (1s): One sentinel leads the failover
4. **Promotion** (1-3s): Promote replica to master
5. **Reconfiguration** (1s): Update clients

**Total Downtime: ~9-13 seconds**

---

### 4.2 Cluster Mode (Sharding + Replication)

> 📊 **See cluster architecture:** [Cluster Sharding Diagram](./hld-diagram.md#cluster-sharding-redis-cluster)

```
Redis Cluster:
16,384 Hash Slots distributed across nodes

Slot 0-5460  → Node 1 (Master) + Replica 1A
Slot 5461-10922 → Node 2 (Master) + Replica 2A
Slot 10923-16383 → Node 3 (Master) + Replica 3A

Client calculates slot: CRC16(key) % 16384
Node redirects if wrong slot (MOVED response)
```

**Advantages:**

- Automatic sharding
- Built-in failover
- Horizontal scalability

**Disadvantages:**

- More complex setup
- Cross-slot operations not supported
- Requires client library support

---

## 5. Bottlenecks and Optimizations

### 5.1 Performance Bottlenecks

| Bottleneck                | Symptoms                   | Solution                                       |
|---------------------------|----------------------------|------------------------------------------------|
| **Network Saturation**    | High latency, packet loss  | Use 10 Gbps network, connection pooling        |
| **Single-threaded Redis** | CPU at 100%, high latency  | Use Redis Cluster for parallel processing      |
| **Memory Fragmentation**  | Memory usage > actual data | Restart Redis periodically, use `activedefrag` |
| **Slow Commands**         | Occasional latency spikes  | Use `SLOWLOG`, avoid `KEYS *`, `FLUSHALL`      |
| **Large Values**          | High bandwidth usage       | Compress values, split large objects           |

### 5.2 Optimization Techniques

> 📊 **See optimization patterns:** [Performance Optimization Diagram](./hld-diagram.md#performance-optimization-techniques)

#### Connection Pooling

❌ **Bad:** Create new connection per request → TCP handshake overhead (~3ms each)

✅ **Good:** Use connection pool with 50 max connections → Reuse connections → No TCP overhead → 3x faster

*See `pseudocode.md::ConnectionPool` for implementation*

#### Pipelining (Batch Operations)

> 📊 **See pipelining benefits:** [Pipelining Sequence Diagram](./sequence-diagrams.md#pipelining-for-batch-operations)

❌ **Bad:** 1000 operations = 1000 network round-trips (~500ms total)

✅ **Good:** Pipeline batches 1000 operations into 1 network round-trip (~0.5ms total) → 1000x faster

*See `pseudocode.md::Pipeline` for implementation*

#### Compression for Large Values

For values > 1 KB, compress before storing. Example: 10 KB JSON → 1 KB compressed (10x memory savings, worth the CPU cost).

*See `pseudocode.md::set_compressed()` and `pseudocode.md::get_compressed()` for implementation*

---

## 6. Monitoring and Observability

> 📊 **See monitoring setup:** [Monitoring Dashboard Diagram](./hld-diagram.md#monitoring-dashboard)

### Key Metrics to Monitor

| Metric            | Target         | Alert Threshold | Why Important               |
|-------------------|----------------|-----------------|-----------------------------|
| **Hit Rate**      | > 80%          | < 70%           | Low hit rate = wasted cache |
| **Latency (p99)** | < 1ms          | > 5ms           | User experience impact      |
| **Memory Usage**  | < 80%          | > 90%           | Prevent eviction storms     |
| **Evicted Keys**  | Minimal        | > 1000/sec      | Indicates undersized cache  |
| **Expired Keys**  | Normal         | > 10000/sec     | Check TTL settings          |
| **Network I/O**   | < 70% capacity | > 80%           | Bottleneck indicator        |
| **Commands/sec**  | Baseline ±20%  | > 2x baseline   | Unusual traffic pattern     |

### Redis Monitoring Commands

```bash
# Get stats
redis-cli INFO stats
redis-cli INFO memory

# Monitor real-time commands
redis-cli MONITOR

# Check slow queries
redis-cli SLOWLOG GET 10

# Get memory usage per key
redis-cli --bigkeys

# Check latency
redis-cli --latency
```

---

## 7. Alternative Approaches

> 📊 **See technology comparison:** [Technology Comparison Diagram](./hld-diagram.md#technology-comparison)

### Comparison: Redis vs Memcached vs KeyDB

| Feature             | Redis                                   | Memcached             | KeyDB                  | Best For                      |
|---------------------|-----------------------------------------|-----------------------|------------------------|-------------------------------|
| **Data Structures** | Rich (lists, sets, sorted sets, hashes) | Key-value only        | Same as Redis          | Complex data: Redis           |
| **Persistence**     | RDB, AOF                                | None                  | RDB, AOF               | Durability: Redis             |
| **Threading**       | Single-threaded                         | Multi-threaded        | Multi-threaded         | High CPU: KeyDB               |
| **Replication**     | Built-in                                | None (requires proxy) | Built-in               | HA: Redis/KeyDB               |
| **Clustering**      | Built-in                                | Client-side           | Built-in               | Scalability: Redis            |
| **Performance**     | 100K+ QPS                               | 300K+ QPS             | 200K+ QPS              | Pure speed: Memcached         |
| **Memory Overhead** | Higher (~20-30%)                        | Lower (~10%)          | Higher (~20-30%)       | Memory constrained: Memcached |
| **Use Case**        | General purpose                         | Pure caching          | High-performance Redis | See "Best For"                |

---

## 8. Trade-offs and Design Decisions Summary

| Decision         | Choice             | Alternative     | Why Chosen                          | Trade-off                            |
|------------------|--------------------|-----------------|-------------------------------------|--------------------------------------|
| **Partitioning** | Consistent Hashing | Modulo Hashing  | Minimal reshuffling                 | Slightly more complex                |
| **Replication**  | Master-Replica     | Masterless (AP) | Simple failover, strong consistency | Write bottleneck at master           |
| **Write Policy** | Cache-Aside        | Write-Through   | Low write latency                   | Temporary inconsistency              |
| **Eviction**     | LRU                | LFU             | Good balance, simple                | May evict hot data accessed long ago |
| **Persistence**  | None (optional)    | RDB/AOF         | Maximum performance                 | Data loss on crash                   |
| **Consistency**  | Eventual           | Strong          | Sub-ms latency                      | Stale reads possible                 |

---

## 9. Common Pitfalls and Anti-Patterns

### Anti-Pattern 1: Not Setting TTL

```
// ❌ Bad: No TTL, cache grows forever
cache.set('user:123', user_data)

// ✅ Good: Always set TTL
cache.set('user:123', user_data, TTL=300)  // 5 minutes
```

### Anti-Pattern 2: Cache Stampede

Already covered in Section 3.6 with solutions.

### Anti-Pattern 3: Storing Large Objects

```
// ❌ Bad: 10 MB object
cache.set('users:all', all_users_list)  // 10 MB!

// ✅ Good: Paginate or split
for i from 0 to length(all_users) step 100:
  chunk = all_users[i : i+100]
  cache.set("users:page:" + (i/100), chunk, TTL=300)
```

### Anti-Pattern 4: Using Cache as Primary Storage

```
// ❌ Bad: Only in cache
cache.set('session:abc', session_data, TTL=3600)
// If cache fails, data is lost!

// ✅ Good: Cache as secondary storage
db.save_session(session_data)  // Primary
cache.set('session:abc', session_data, TTL=3600)  // Secondary
```

---

## 10. Capacity Planning Guidelines

> 📊 **See scaling strategy:** [Scaling Strategy Diagram](./hld-diagram.md#scaling-strategy)

### Scaling Strategy

| Metric      | When to Scale      | How to Scale                             |
|-------------|--------------------|------------------------------------------|
| **Memory**  | > 80% usage        | Add more nodes (horizontal)              |
| **CPU**     | > 70% usage        | Use KeyDB (multi-threaded) or add nodes  |
| **Network** | > 70% bandwidth    | Upgrade NIC, add nodes                   |
| **QPS**     | > 80K QPS per node | Add more master nodes                    |
| **Latency** | p99 > 5ms          | Optimize queries, add replicas for reads |

### Cost Estimation (AWS Example)

```
Instance Type: r6g.2xlarge (64 GB RAM, 8 vCPU)
Cost: $0.504/hour

9 nodes × $0.504/hour × 730 hours/month = $3,311/month

With 3-year reserved instance:
9 nodes × $0.303/hour × 730 hours/month = $1,991/month

Data Transfer: ~$0.01/GB (within AZ, negligible)

Total: ~$2,000-3,300/month for 180 GB cache capacity
```

---

## Summary

A distributed cache system requires careful design decisions balancing:

- **Performance** (sub-millisecond latency)
- **Availability** (failover, replication)
- **Scalability** (consistent hashing, sharding)
- **Consistency** (eventual vs strong)

**Key Takeaways:**

1. Use **Consistent Hashing** with virtual nodes for minimal reshuffling
2. **Async replication** provides best performance for cache use cases
3. **Cache-Aside** pattern ideal for read-heavy workloads
4. **Single Flight Request** prevents cache stampede
5. **Monitor hit rate** and latency continuously
6. **Always set TTL** to prevent unbounded growth
7. Cache is **secondary storage** - database is source of truth

**Recommended Stack:**

- **Redis** for general purpose (rich features, good performance)
- **KeyDB** for extreme performance (multi-threaded)
- **Memcached** for pure key-value with minimal overhead
- **Redis Sentinel** for automatic failover
- **Redis Cluster** for horizontal scalability

