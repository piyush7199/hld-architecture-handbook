# Distributed Cache - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A distributed cache is a horizontally scalable, in-memory key-value store that provides sub-millisecond latency for
millions of operations per second. It acts as a secondary storage layer between applications and databases, dramatically
reducing database load and improving response times.

**Key Characteristics:**

- **Sub-millisecond latency** (p99 < 1ms) - Critical for high-performance applications
- **High throughput** (> 100K QPS per node) - Supports high-traffic workloads
- **Horizontal scalability** - Add/remove nodes dynamically without downtime
- **Minimal reshuffling** (< 1% keys moved on node add/remove) - Preserves cache hit ratio
- **High availability** (99.99% uptime) - Single node failure doesn't affect service
- **Fault tolerance** - Automatic failover and replication

---

## Requirements & Scale

### Functional Requirements

| Requirement                | Description                                         | Priority     |
|----------------------------|-----------------------------------------------------|--------------|
| **Key-Value Operations**   | GET, PUT, DELETE with O(1) complexity               | Must Have    |
| **Time-to-Live (TTL)**     | Keys expire automatically after specified duration  | Must Have    |
| **Horizontal Scalability** | Add/remove cache nodes dynamically without downtime | Must Have    |
| **Data Types**             | Support strings, lists, sets, sorted sets, hashes   | Should Have  |
| **Atomic Operations**      | Atomic increment/decrement, compare-and-swap        | Should Have  |
| **Persistence (Optional)** | Option to persist data to disk for durability       | Nice to Have |

### Scale Estimation (E-commerce Platform Example)

| Metric                | Calculation                                | Result             |
|-----------------------|--------------------------------------------|--------------------|
| **Total Data**        | 10% of 1.8 TB database in cache            | ~180 GB RAM needed |
| **Average Key Size**  | Key: 64 bytes, Value: 1 KB                 | ~1 KB per entry    |
| **Total Keys**        | 180 GB / 1 KB                              | ~180 million keys  |
| **Peak Read QPS**     | 80% of DB reads (1157 QPS × 0.8 × 5 burst) | ~4,600 reads/sec   |
| **Peak Write QPS**    | 20% of reads (cache invalidation)          | ~920 writes/sec    |
| **Network Bandwidth** | 1 KB × 4,600 QPS                           | ~4.6 MB/sec        |
| **Nodes Required**    | 180 GB / 64 GB per node × 3 replicas       | ~9 nodes           |

**Back-of-the-Envelope:**

- Memory per Node: 64 GB RAM
- Replication Factor: 3 (1 primary + 2 replicas)
- Total Nodes: (180 GB / 64 GB) × 3 = ~9 nodes
- QPS per Node: 4,600 QPS / 3 primaries = ~1,533 QPS/node (well within Redis capacity of 100K+ QPS)
- Network: 4.6 MB/sec is negligible (1 Gbps = 125 MB/sec)

---

## Key Components

| Component               | Responsibility                                  | Technology Options           |
|-------------------------|-------------------------------------------------|------------------------------|
| **Client Library**      | Consistent hashing, routing, connection pooling | jedis, lettuce, go-redis     |
| **Cache Node**          | In-memory key-value storage                     | Redis, Memcached, KeyDB      |
| **Replication**         | Data redundancy, failover                       | Master-Replica (async/sync)  |
| **Cluster Manager**     | Health monitoring, failover orchestration       | Redis Sentinel, Cluster mode |
| **Configuration Store** | Node discovery, cluster topology                | etcd, Consul, ZooKeeper      |

---

## Data Partitioning: Consistent Hashing

### Why Consistent Hashing?

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

- Only ~25% of keys remapped when adding/removing nodes (vs 75% with modulo)
- Uses hash ring with virtual nodes for uniform distribution
- O(log N) lookup time via binary search

**Virtual Nodes (VNodes) Impact:**

| VNodes per Node | Distribution Uniformity   | Lookup Overhead | Recommendation     |
|-----------------|---------------------------|-----------------|--------------------|
| 10-50           | Poor (±20% imbalance)     | Very Low        | Not recommended    |
| 100-200         | Good (±5% imbalance)      | Low             | **Recommended**    |
| 500-1000        | Excellent (±1% imbalance) | Medium          | For large clusters |
| 5000+           | Perfect                   | High            | Overkill           |

**Implementation:**

- **Data Structures:** HashMap (hash → node), Sorted list (for binary search)
- **Virtual Nodes:** 150 virtual nodes per physical node (recommended)
- **Hash Function:** MD5 or MurmurHash for uniform distribution
- **Operations:** `get_node(key)` - O(log N) via binary search

---

## Replication Strategy

### Master-Replica (Leader-Follower) Architecture

**How It Works:**

- **Master (Primary):** Handles all writes, replicates to replicas
- **Replicas (Followers):** Read-only, receive async replication from master
- **Replication Lag:** 1-5ms (acceptable for cache use case)

**Replication Options Comparison:**

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

## Write Policies: Cache-Aside (Lazy Loading)

### How Cache-Aside Works

**Read Flow:**

1. Application checks cache first for the data
2. **Cache Hit:** Return data immediately (fast path)
3. **Cache Miss:**
    - Fetch from database (slow path)
    - Populate cache with fetched data (TTL = 5 minutes)
    - Return data to application
4. Subsequent reads hit cache (fast)

**Write Flow:**

1. Update database first (source of truth)
2. **Invalidate** cache entry (don't update it!)
3. Next read will fetch fresh data and populate cache

**Why Invalidate Instead of Update:**

- **Simpler:** No serialization issues
- **Safer:** Avoids inconsistency if DB update succeeds but cache update fails
- **Lazy:** Don't waste effort on data that might not be read again

**Cache-Aside vs Write-Through:**

| Aspect            | Cache-Aside                  | Write-Through                             |
|-------------------|------------------------------|-------------------------------------------|
| **Write Latency** | Low (DB only)                | High (DB + Cache sync)                    |
| **Cache Size**    | Smaller (only accessed data) | Larger (all written data)                 |
| **Consistency**   | Eventual                     | Strong                                    |
| **Best For**      | **Read-heavy workloads**     | Write-heavy, immediate consistency needed |

---

## Eviction Policies

### LRU (Least Recently Used) - Recommended

**Implementation:**

- **Data Structures:** HashMap (key → node), Doubly-Linked List (access order)
- **Operations:** All O(1)
    - GET: Lookup in HashMap, move to head (mark as recently used)
    - PUT: Add to head, evict tail if capacity exceeded
- **Space Complexity:** O(capacity)

**Eviction Policy Comparison:**

| Policy     | Best For               | Pros                  | Cons                       | Redis Support      |
|------------|------------------------|-----------------------|----------------------------|--------------------|
| **LRU**    | General purpose        | Good hit rate, simple | Ignores frequency          | ✅ `allkeys-lru`    |
| **LFU**    | Stable access patterns | Keeps hot data        | Complex, cold start issues | ✅ `allkeys-lfu`    |
| **TTL**    | Time-sensitive data    | Predictable expiry    | May evict useful data      | ✅ `volatile-ttl`   |
| **Random** | Low overhead           | Very simple           | Poor hit rate              | ✅ `allkeys-random` |

---

## TTL (Time-to-Live) Management

### Redis Hybrid Approach (Best Practice)

Combines both strategies:

- **Active:** Check on every access (lazy delete)
- **Passive:** Sample 20 keys every 100ms, delete expired (background scan)
- **Probabilistic:** If > 25% expired, scan again immediately (aggressive mode)

**Why Hybrid?**

- Active ensures expired keys are never returned
- Passive cleans up unused expired keys
- Probabilistic prevents memory bloat from many expired keys

---

## Concurrency Control: Preventing Cache Stampede

### Problem: Cache Stampede / Thundering Herd

When a hot key expires, thousands of concurrent requests all hit the database simultaneously, overwhelming it.

**Example:** 10,000 concurrent requests for same user → 10,000 DB queries → Database overload

### Solution 1: Single Flight Request (Recommended)

**How It Works:**

1. Request arrives: Check cache first
2. Cache miss: Acquire distributed lock for this specific key
3. **Lock acquired (first request):**
    - Double-check cache (another thread may have populated it)
    - Fetch from database
    - Populate cache
    - Release lock
4. **Lock waiting (other requests):**
    - Wait for first request to complete
    - Read from cache (now populated)
    - No database query needed

**Benefits:**

- Prevents database overload (10,000 requests → 1 DB query)
- Transparent to application code
- Works across multiple servers (distributed lock)

**Trade-offs:**

- Adds slight latency (lock acquisition overhead ~1-5ms)
- Requires distributed lock infrastructure (Redis, etcd)

### Solution 2: Probabilistic Early Expiration

**How It Works:**

- Get entry with expiry time
- If TTL < 10% remaining, calculate refresh probability
- Probability increases as expiration approaches
- Trigger async background refresh
- Cache never goes completely cold

---

## Availability & Fault Tolerance

### Automatic Failover with Sentinel

**Failover Process:**

1. **Detection** (5s): Sentinel detects master unresponsive
2. **Quorum Vote** (1s): Sentinels vote if master is really down
3. **Leader Election** (1s): One sentinel leads the failover
4. **Promotion** (1-3s): Promote replica to master
5. **Reconfiguration** (1s): Update clients

**Total Downtime: ~9-13 seconds**

**Configuration:**

```conf
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000
# Quorum = 2 (need 2 sentinels to agree master is down)
```

### Cluster Mode (Sharding + Replication)

**Redis Cluster:**

- 16,384 Hash Slots distributed across nodes
- Client calculates slot: `CRC16(key) % 16384`
- Node redirects if wrong slot (MOVED response)
- Automatic sharding and built-in failover

**Advantages:**

- Automatic sharding
- Built-in failover
- Horizontal scalability

**Disadvantages:**

- More complex setup
- Cross-slot operations not supported
- Requires client library support

---

## Performance Optimizations

### Connection Pooling

❌ **Bad:** New connection per request → TCP handshake ~3ms overhead each

✅ **Good:** Connection pool (max 50 connections) → Reuse connections → No handshake → 3x faster

### Pipelining (Batch Operations)

❌ **Bad:** Round-trip for each operation

```
for i from 0 to 999:
  cache.set("key:" + i, i)
// 1000 network round-trips (~500ms at 0.5ms each)
```

✅ **Good:** Pipeline (batch)

```
pipeline = cache.create_pipeline()
for i from 0 to 999:
  pipeline.set("key:" + i, i)
pipeline.execute()
// 1 network round-trip (~0.5ms)
```

### Compression for Large Values

For large values (> 1 KB):

- Serialize to JSON
- Compress (gzip/snappy)
- Store compressed
- On retrieval: decompress and deserialize

**Example:** 10 KB JSON → 1 KB compressed (10x memory savings)

---

## Monitoring & Observability

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

## Technology Comparison

### Redis vs Memcached vs KeyDB

| Feature             | Redis                                   | Memcached             | KeyDB            | Best For                      |
|---------------------|-----------------------------------------|-----------------------|------------------|-------------------------------|
| **Data Structures** | Rich (lists, sets, sorted sets, hashes) | Key-value only        | Same as Redis    | Complex data: Redis           |
| **Persistence**     | RDB, AOF                                | None                  | RDB, AOF         | Durability: Redis             |
| **Threading**       | Single-threaded                         | Multi-threaded        | Multi-threaded   | High CPU: KeyDB               |
| **Replication**     | Built-in                                | None (requires proxy) | Built-in         | HA: Redis/KeyDB               |
| **Clustering**      | Built-in                                | Client-side           | Built-in         | Scalability: Redis            |
| **Performance**     | 100K+ QPS                               | 300K+ QPS             | 200K+ QPS        | Pure speed: Memcached         |
| **Memory Overhead** | Higher (~20-30%)                        | Lower (~10%)          | Higher (~20-30%) | Memory constrained: Memcached |

---

## Common Anti-Patterns to Avoid

### 1. Not Setting TTL

❌ **Bad:** No TTL, cache grows forever

```python
cache.set('user:123', user_data)
```

✅ **Good:** Always set TTL

```python
cache.set('user:123', user_data, TTL=300)  # 5 minutes
```

### 2. Cache Stampede

Already covered with Single Flight Request and Probabilistic Early Expiration solutions.

### 3. Storing Large Objects

❌ **Bad:** 10 MB object

```python
cache.set('users:all', all_users_list)  # 10 MB!
```

✅ **Good:** Paginate or split

```python
for i from 0 to length(all_users) step 100:
  chunk = all_users[i : i+100]
  cache.set("users:page:" + (i/100), chunk, TTL=300)
```

### 4. Using Cache as Primary Storage

❌ **Bad:** Only in cache

```python
cache.set('session:abc', session_data, TTL=3600)
# If cache fails, data is lost!
```

✅ **Good:** Cache as secondary storage

```python
db.save_session(session_data)  # Primary
cache.set('session:abc', session_data, TTL=3600)  # Secondary
```

---

## Trade-offs Summary

| Decision         | Choice             | Alternative     | Why Chosen                          | Trade-off                            |
|------------------|--------------------|-----------------|-------------------------------------|--------------------------------------|
| **Partitioning** | Consistent Hashing | Modulo Hashing  | Minimal reshuffling                 | Slightly more complex                |
| **Replication**  | Master-Replica     | Masterless (AP) | Simple failover, strong consistency | Write bottleneck at master           |
| **Write Policy** | Cache-Aside        | Write-Through   | Low write latency                   | Temporary inconsistency              |
| **Eviction**     | LRU                | LFU             | Good balance, simple                | May evict hot data accessed long ago |
| **Persistence**  | None (optional)    | RDB/AOF         | Maximum performance                 | Data loss on crash                   |
| **Consistency**  | Eventual           | Strong          | Sub-ms latency                      | Stale reads possible                 |

---

## Capacity Planning

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

Total: ~$2,000-3,300/month for 180 GB cache capacity
```

---

## Key Takeaways

1. **Use Consistent Hashing** with virtual nodes (150 per node) for minimal reshuffling
2. **Async replication** provides best performance for cache use cases
3. **Cache-Aside** pattern ideal for read-heavy workloads
4. **Single Flight Request** prevents cache stampede
5. **Monitor hit rate** and latency continuously
6. **Always set TTL** to prevent unbounded growth
7. **Cache is secondary storage** - database is source of truth
8. **LRU eviction** provides good balance for general purpose
9. **Connection pooling** and **pipelining** for performance
10. **Compress large values** (> 1 KB) to save memory

---

## Recommended Stack

- **Redis** for general purpose (rich features, good performance)
- **KeyDB** for extreme performance (multi-threaded)
- **Memcached** for pure key-value with minimal overhead
- **Redis Sentinel** for automatic failover
- **Redis Cluster** for horizontal scalability

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, consistent hashing, replication)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (cache operations, failover)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (consistent hashing, LRU, TTL management)

