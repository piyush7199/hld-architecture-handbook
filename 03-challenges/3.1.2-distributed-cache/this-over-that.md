# Distributed Cache - Design Decisions (This Over That)

This document provides an in-depth analysis of all major architectural decisions made for the Distributed Cache system design.

---

## Table of Contents

1. [Technology Choice: Redis vs Memcached vs KeyDB](#1-technology-choice-redis-vs-memcached-vs-keydb)
2. [Partitioning Strategy: Consistent Hashing vs Modulo Hashing](#2-partitioning-strategy-consistent-hashing-vs-modulo-hashing)
3. [Replication: Master-Replica vs Masterless](#3-replication-master-replica-vs-masterless)
4. [Write Policy: Cache-Aside vs Write-Through](#4-write-policy-cache-aside-vs-write-through)
5. [Eviction Policy: LRU vs LFU vs TTL](#5-eviction-policy-lru-vs-lfu-vs-ttl)
6. [Consistency: Eventual vs Strong](#6-consistency-eventual-vs-strong)
7. [Cluster Mode: Sentinel vs Redis Cluster](#7-cluster-mode-sentinel-vs-redis-cluster)

---

## 1. Technology Choice: Redis vs Memcached vs KeyDB

### The Problem

We need an in-memory caching system that can:
- Store 180 GB of data across multiple nodes
- Handle 100K+ operations per second
- Provide sub-millisecond latency
- Support horizontal scaling
- Offer high availability with automatic failover

### Options Considered

#### Option A: Redis ✅ **CHOSEN**

**Pros:**
- ✅ **Rich data structures** - Lists, sets, sorted sets, hashes (not just key-value)
- ✅ **Persistence options** - RDB snapshots, AOF logs (optional durability)
- ✅ **Built-in replication** - Master-replica setup with automatic promotion
- ✅ **Atomic operations** - INCR, DECR, compare-and-swap
- ✅ **Pub/Sub support** - Real-time messaging capabilities
- ✅ **Lua scripting** - Server-side complex operations
- ✅ **Active community** - Excellent documentation and tooling
- ✅ **Redis Cluster** - Built-in horizontal scaling
- ✅ **Redis Sentinel** - Automatic failover management

**Cons:**
- ❌ **Single-threaded** - One CPU core per instance (CPU bottleneck)
- ❌ **Higher memory overhead** - ~20-30% compared to Memcached
- ❌ **More complex** - Many features mean more to configure
- ❌ **Persistence overhead** - Snapshotting can cause latency spikes

**Performance:**
- 100K-200K QPS per instance (single-threaded)
- Sub-millisecond latency (p99 < 1ms)
- Memory efficiency: ~70-80% (20-30% overhead)

**Real-World Usage:**
- Twitter uses Redis for timeline caching
- GitHub uses Redis for session storage
- Stack Overflow uses Redis for vote counts

#### Option B: Memcached

**Pros:**
- ✅ **Multi-threaded** - Better CPU utilization, 300K+ QPS per instance
- ✅ **Lower memory overhead** - ~10-15% overhead (more data per GB)
- ✅ **Simple** - Easy to understand and operate
- ✅ **Mature** - Battle-tested for decades
- ✅ **Extremely fast** - Raw speed champion for simple key-value

**Cons:**
- ❌ **Key-value only** - No lists, sets, sorted sets
- ❌ **No persistence** - Data lost on restart (true cache only)
- ❌ **No replication** - Must handle at client or use mcrouter
- ❌ **No atomic operations** - Can't INCR/DECR atomically across nodes
- ❌ **No failover** - Manual handling required
- ❌ **Limited features** - Just basic get/set

**Performance:**
- 300K-500K QPS per instance (multi-threaded)
- Sub-millisecond latency (p99 < 1ms)
- Memory efficiency: ~85-90%

**Real-World Usage:**
- Facebook uses Memcached extensively (with mcrouter)
- YouTube uses Memcached for video metadata

#### Option C: KeyDB (Redis Fork)

**Pros:**
- ✅ **Multi-threaded Redis** - Same features, better performance
- ✅ **100% Redis-compatible** - Drop-in replacement
- ✅ **Active-active replication** - Multi-master writes
- ✅ **Better performance** - 2-5x throughput vs Redis
- ✅ **All Redis features** - Data structures, persistence, etc.

**Cons:**
- ❌ **Smaller community** - Less mature than Redis
- ❌ **Newer project** - Less battle-tested
- ❌ **Some compatibility issues** - Edge cases with Redis modules
- ❌ **Enterprise support** - Limited compared to Redis

**Performance:**
- 200K-500K QPS per instance (multi-threaded)
- Sub-millisecond latency
- Memory efficiency: ~70-80% (same as Redis)

**Real-World Usage:**
- Snap uses KeyDB for high-throughput caching
- Various startups for performance-critical caching

### Decision Made: Redis

**Why Redis Wins:**

1. **Rich Data Structures:** Not just key-value. We can store:
   - User sessions (hashes)
   - Real-time counters (atomic INCR/DECR)
   - Leaderboards (sorted sets)
   - Message queues (lists)
   - Sets for tag filtering

2. **Built-in High Availability:** Redis Sentinel provides automatic failover without additional infrastructure.

3. **Ecosystem Maturity:** Excellent client libraries in all languages, extensive documentation, proven at massive scale.

4. **Operational Simplicity:** While more complex than Memcached, the integrated clustering and replication reduce external dependencies.

5. **Future Flexibility:** Starting with Redis gives us options (persistence, Pub/Sub) even if not needed immediately.

### Trade-offs Accepted

- **Single-threaded Performance:** Accept 100K QPS per instance instead of Memcached's 300K
  - *Mitigation:* Scale horizontally (more instances)
- **Higher Memory Cost:** 20-30% overhead vs Memcached's 10%
  - *Acceptable:* Memory is cheap relative to operational complexity

### When to Reconsider

**Switch to Memcached if:**
- Only need simple key-value storage (no data structures)
- Raw throughput per node is critical (300K+ QPS needed)
- Memory cost is paramount (need every MB)
- Persistence is explicitly not wanted

**Switch to KeyDB if:**
- Need Redis features but CPU is bottleneck
- Multi-threaded performance critical
- Active-active replication across regions needed
- Comfortable with less mature ecosystem

---

## 2. Partitioning Strategy: Consistent Hashing vs Modulo Hashing

### The Problem

With 180 GB of data and 9 nodes, we need to distribute keys across nodes. When nodes are added/removed:
- Minimize data movement (cache hit rate preservation)
- Maintain even distribution (no hotspots)
- Support dynamic scaling (add/remove nodes easily)

### Options Considered

#### Option A: Consistent Hashing ✅ **CHOSEN**

**How it works:**
```
1. Hash each key to a point on a ring (0 to 2^32-1)
2. Hash each node (with virtual nodes) to points on ring
3. Key is assigned to first node clockwise from key's position

With virtual nodes (vnodes):
- Node 1: vnodes at positions 100, 500, 900, ...
- Node 2: vnodes at positions 200, 600, 1000, ...
- Node 3: vnodes at positions 300, 700, 1100, ...
```

**Pros:**
- ✅ **Minimal reshuffling** - Only ~1/N keys move when adding Nth node
- ✅ **Even distribution** - Virtual nodes balance load
- ✅ **Predictable** - Same key always goes to same node (unless topology changes)
- ✅ **Elastic scaling** - Add/remove nodes without full rehash

**Cons:**
- ❌ **Complex** - More code than modulo hashing
- ❌ **Virtual node overhead** - Must track 100-200 vnodes per physical node
- ❌ **Hotspot possible** - If few vnodes, distribution uneven

**Data Movement Example:**
```
3 nodes → 4 nodes:
- Modulo: 75% of keys move (re-hash all)
- Consistent Hashing: 25% of keys move (1/4 from existing nodes to new node)
```

**Real-World Usage:**
- Amazon DynamoDB uses consistent hashing
- Apache Cassandra uses consistent hashing
- Most distributed systems use variant of consistent hashing

#### Option B: Modulo Hashing (Simple Hash)

**How it works:**
```
node_id = hash(key) % num_nodes

Example with 3 nodes:
key "user:123" → hash = 5234 → 5234 % 3 = 2 → Node 2
```

**Pros:**
- ✅ **Extremely simple** - One line of code
- ✅ **Perfect distribution** - Mathematical guarantee of evenness
- ✅ **Fast** - Just modulo operation
- ✅ **No virtual nodes** - Simpler mental model

**Cons:**
- ❌ **Massive reshuffling** - Adding/removing node rehashes ~75-100% of keys
- ❌ **Cache invalidation cascade** - All data moves = cache miss storm
- ❌ **Scaling is disruptive** - Can't scale without downtime
- ❌ **Fixed cluster size** - Changing size is expensive

**Data Movement Example:**
```
3 nodes → 4 nodes:
key "user:123" → hash = 5234
Old: 5234 % 3 = 2 → Node 2
New: 5234 % 4 = 2 → Node 2 (lucky! but only 25% stay in same place)

Result: 75% of keys move to different nodes
```

**Real-World Usage:**
- Only used in fixed-size clusters
- Not suitable for production distributed caches

#### Option C: Range-Based Partitioning

**How it works:**
```
Node 1: keys a-f
Node 2: keys g-m  
Node 3: keys n-s
Node 4: keys t-z
```

**Pros:**
- ✅ **Simple** - Easy to understand
- ✅ **Range queries** - Can scan alphabetically
- ✅ **Minimal movement** - Only boundary keys move

**Cons:**
- ❌ **Hotspots** - Some ranges have more data (uneven)
- ❌ **Manual rebalancing** - Need to adjust ranges manually
- ❌ **Key structure dependent** - Assumes even key distribution

### Decision Made: Consistent Hashing

**Why Consistent Hashing Wins:**

1. **Preserves Cache Hit Rate:** When adding Node 4 to 3-node cluster:
   - Modulo: 75% cache miss storm (keys move)
   - Consistent Hash: 25% cache misses (only 1/4 move)

2. **Elastic Scaling:** Can add nodes during traffic without causing outage from cache miss storm.

3. **Industry Standard:** Proven pattern used by all major distributed systems.

4. **Virtual Nodes Solve Distribution:** With 150 vnodes per node, distribution is within ±5% (excellent).

### Implementation Details

**Virtual Nodes Configuration:**

| VNodes per Node | Distribution | Lookup Overhead | Recommendation |
|-----------------|--------------|-----------------|----------------|
| 10-50 | ±20% imbalance | Very Low | ❌ Not recommended |
| 100-200 | ±5% imbalance | Low | ✅ **Recommended** |
| 500-1000 | ±1% imbalance | Medium | For large clusters |

**Our Choice: 150 vnodes per node**
- Good balance: ±5% distribution variance
- Low lookup overhead: Binary search in sorted array
- Proven in production (Cassandra uses similar)

### Trade-offs Accepted

- **Complexity:** 100 lines of code vs 1 line for modulo
  - *Acceptable:* Use proven library (java-consistent-hash, hash-ring)
- **Lookup Overhead:** Binary search O(log N) vs modulo O(1)
  - *Negligible:* Microseconds vs milliseconds for cache lookup

### When to Reconsider

**Use Modulo Hashing if:**
- Cluster size is fixed forever (no scaling plans)
- Downtime for reshuffling is acceptable
- Simplicity trumps all other concerns

**Use Range Partitioning if:**
- Keys have natural ranges (timestamps, user IDs)
- Need range scan capabilities
- Can manually rebalance when needed

---

## 3. Replication: Master-Replica vs Masterless

### The Problem

Single node failures should not cause:
- Data loss
- Service unavailability
- Cache cold start (miss storm)

Need replication strategy for 99.99% availability.

### Options Considered

#### Option A: Master-Replica (Leader-Follower) ✅ **CHOSEN**

**Architecture:**
```
Master (Primary)
├─ Handles all writes
├─ Async replication to replicas
│  └─ Replica 1 (read-only, lags 1-5ms)
│  └─ Replica 2 (read-only, lags 1-5ms)
└─ Reads can go to master or replicas
```

**Pros:**
- ✅ **Simple consistency** - One source of truth for writes
- ✅ **Fast writes** - No coordination, just write to master
- ✅ **Read scaling** - Add replicas for more read capacity
- ✅ **Clear failover** - Promote replica to master
- ✅ **Redis native** - Built into Redis

**Cons:**
- ❌ **Write bottleneck** - All writes to one master
- ❌ **Replication lag** - Replicas 1-5ms behind (stale reads)
- ❌ **Failover downtime** - 5-15 seconds during promotion
- ❌ **Single master** - Can't write during master failure

**Failover Process:**
```
1. Sentinel detects master down (5 seconds)
2. Quorum vote among sentinels (1 second)
3. Promote replica to master (2-5 seconds)
4. Update clients with new master (1-2 seconds)
Total: 9-13 seconds of write unavailability
```

**Real-World Usage:**
- Redis Sentinel (standard approach)
- MySQL replication
- PostgreSQL streaming replication

#### Option B: Masterless (Leaderless, AP)

**Architecture:**
```
All nodes equal:
├─ Node 1 (read + write)
├─ Node 2 (read + write)
└─ Node 3 (read + write)

Write quorum: W=2 (must write to 2 nodes)
Read quorum: R=2 (must read from 2 nodes)
W + R > N ensures consistency
```

**Pros:**
- ✅ **No single point of failure** - Any node can accept writes
- ✅ **Higher write availability** - Continue with N-1 nodes
- ✅ **No failover delay** - No master promotion needed
- ✅ **Linear write scaling** - More nodes = more write capacity

**Cons:**
- ❌ **Complex** - Quorum logic, conflict resolution
- ❌ **Eventual consistency** - Different nodes may disagree
- ❌ **Higher write latency** - Must write to W nodes (network hops)
- ❌ **Network partition issues** - Split brain scenarios
- ❌ **Not native to Redis** - Must use Dynamo-style system

**Real-World Usage:**
- Apache Cassandra (masterless)
- Amazon DynamoDB (masterless)
- Riak (masterless)

#### Option C: Multi-Master (Active-Active)

**Architecture:**
```
Master 1 (US-East) ⟷ Master 2 (EU-West)
Both accept writes, bi-directional replication
```

**Pros:**
- ✅ **Geographic distribution** - Write locally in each region
- ✅ **Higher write capacity** - Multiple write nodes
- ✅ **Low latency writes** - Write to nearest master

**Cons:**
- ❌ **Conflict resolution** - What if same key written in both regions?
- ❌ **Complex** - Need conflict resolution strategy (LWW, CRDT)
- ❌ **Not standard Redis** - KeyDB supports this, not Redis
- ❌ **Subtle bugs** - Conflicts can cause data inconsistency

### Decision Made: Master-Replica

**Why Master-Replica Wins for Cache:**

1. **Cache is Not Source of Truth:** Unlike a database, losing 5ms of cached writes is acceptable. Database is source of truth.

2. **Read-Heavy Workload:** With 10:1 read-to-write ratio, read scaling (replicas) is more important than write scaling (multiple masters).

3. **Simplicity:** One master means clear consistency model. No conflict resolution needed.

4. **Redis Native:** Built-in support with Sentinel. No custom infrastructure.

5. **Acceptable Failover:** 9-13 seconds of write unavailability during master failure is tolerable for cache (reads still work from replicas).

### Replication Configuration

**Async vs Sync Replication:**

| Type | Write Latency | Data Loss Risk | Use Case |
|------|---------------|----------------|----------|
| **Async** ✅ | <1ms | 1-5ms of data | **Cache (chosen)** |
| Sync | 5-10ms | None | Databases |
| Semi-sync | 2-5ms | Minimal | Hybrid systems |

**Why Async for Cache:**
```
Async:
Client → Write to Master → ACK (1ms) ✅
         Master → Replicate to Replicas (async)

Sync:
Client → Write to Master → Wait for Replica ACK → ACK client (10ms) ❌

For cache, sub-millisecond writes are critical.
Losing 5ms of cached data is acceptable (can refetch from DB).
```

### Trade-offs Accepted

- **Replication Lag:** Replicas can be 1-5ms behind master
  - *Acceptable:* Cache can tolerate slight staleness
- **Write Unavailability During Failover:** 9-13 seconds
  - *Acceptable:* Reads still work, writes queue in application

### When to Reconsider

**Use Masterless if:**
- Need 99.999% write availability (no failover downtime)
- Write scaling more important than simplicity
- Can handle eventual consistency complexities
- Using Cassandra/DynamoDB instead of Redis

**Use Multi-Master if:**
- Multi-region writes required (can't afford cross-region latency)
- Using KeyDB or custom infrastructure
- Have robust conflict resolution strategy
- Geographic distribution is critical

---

## 4. Write Policy: Cache-Aside vs Write-Through

### The Problem

When application writes data to database, how does it sync with cache?

Options:
1. **Cache-Aside:** App writes DB, then invalidates cache (lazy load)
2. **Write-Through:** App writes to cache, cache writes to DB
3. **Write-Behind:** App writes to cache, async write to DB

### Options Considered

#### Option A: Cache-Aside (Lazy Loading) ✅ **CHOSEN**

**Flow:**
```
Write:
App → Write to DB → Delete from cache
Next read will populate cache

Read:
App → Check cache → Miss → Query DB → Populate cache
```

**Pros:**
- ✅ **Simple** - Cache doesn't need to know about DB
- ✅ **Memory efficient** - Only cache what's actually read
- ✅ **Resilient** - Cache failure doesn't affect writes
- ✅ **No write amplification** - Don't cache data that's never read

**Cons:**
- ❌ **Cache misses** - First read after write is slow
- ❌ **Stampede risk** - Popular key expiry causes many DB queries
- ❌ **Stale data possible** - Between invalidation and next read

**Real-World Usage:**
- Most web applications (Facebook, Twitter, Amazon)
- Standard pattern for read-heavy systems

#### Option B: Write-Through

**Flow:**
```
Write:
App → Write to cache → Cache writes to DB → ACK to app

Read:
App → Read from cache (always populated)
```

**Pros:**
- ✅ **Cache always fresh** - No stale data
- ✅ **No cache misses** - Cache warmed on every write
- ✅ **Consistent** - Cache matches DB

**Cons:**
- ❌ **Higher write latency** - Must wait for cache AND DB
- ❌ **Wasted memory** - Caches data that might never be read
- ❌ **Cache must know DB** - Tight coupling
- ❌ **Complex failure modes** - What if cache write succeeds but DB fails?

**Real-World Usage:**
- Write-heavy systems where reads must be fast
- Inventory management systems

#### Option C: Write-Behind (Write-Back)

**Flow:**
```
Write:
App → Write to cache → Async batch write to DB

Read:
App → Read from cache (always populated)
```

**Pros:**
- ✅ **Extremely fast writes** - No DB wait
- ✅ **Batching possible** - Write 1000 records at once to DB
- ✅ **Reduced DB load** - Fewer write operations

**Cons:**
- ❌ **Data loss risk** - If cache crashes before DB write
- ❌ **Complex** - Need persistent queue, retry logic
- ❌ **Inconsistency** - Cache and DB temporarily out of sync
- ❌ **Not suitable for critical data**

### Decision Made: Cache-Aside

**Why Cache-Aside Wins for Distributed Cache:**

1. **Cache Independence:** Cache should be orthogonal to application logic. With Cache-Aside, application controls data flow explicitly.

2. **Memory Efficiency:** In typical applications, 80% of reads hit 20% of data. Why waste cache memory on data nobody reads?

3. **Failure Isolation:** If cache goes down, application still works (slower, but works). With Write-Through, cache is in critical path.

4. **Read-Heavy Optimization:** For read-heavy workloads (which is when you use cache), optimizing reads is more important than writes.

### Implementation Pattern

```
function get_user(user_id):
  // 1. Try cache first
  cache_key = "user:" + user_id
  user = cache.get(cache_key)
  
  if user exists:
    return user  // Cache Hit
  
  // 2. Cache miss - query database
  user = database.query("SELECT * FROM users WHERE id = ?", user_id)
  
  if user exists:
    // 3. Populate cache for next time
    cache.set(cache_key, user, TTL=300)  // 5 minutes
  
  return user


function update_user(user_id, new_data):
  // 1. Update database (source of truth)
  database.execute("UPDATE users SET ... WHERE id = ?", new_data, user_id)
  
  // 2. Invalidate cache (don't update!)
  cache_key = "user:" + user_id
  cache.delete(cache_key)
  
  // Next read will fetch fresh data from DB and populate cache
```

**Why Delete Instead of Update:**
- Simpler: No need to serialize/deserialize data
- Safer: Avoids race conditions (write to DB succeeds, cache update fails)
- Lazy: Don't spend effort caching data that might not be read again

### Trade-offs Accepted

- **First Read Miss:** After every write, first read is slower (must query DB)
  - *Acceptable:* Most entities are read far more often than written
- **Stampede Possible:** Popular key expiry can cause many concurrent DB queries
  - *Mitigation:* Use distributed locks or probabilistic early expiration

### When to Reconsider

**Use Write-Through if:**
- Cache must always be consistent with DB
- Write and read rates are similar (not read-heavy)
- Can afford higher write latency
- Memory is abundant (can cache everything)

**Use Write-Behind if:**
- Write performance is absolutely critical
- Can tolerate data loss (cache is temporary source of truth)
- Have robust persistent queue infrastructure
- Example: Gaming leaderboards (eventual consistency OK)

---

## 5. Eviction Policy: LRU vs LFU vs TTL

### The Problem

Cache memory is limited. When full, must evict (remove) data to make room for new data. Which data to evict?

### Options Considered

#### Option A: LRU (Least Recently Used) ✅ **CHOSEN**

**Algorithm:**
```
Evict the item that was accessed longest time ago

Example timeline:
Access order: A, B, C, D, A, E
Cache full, need to evict
LRU order: B < C < D < A < E
Evict B (least recently used)
```

**Pros:**
- ✅ **Good balance** - Works well for most workloads
- ✅ **Simple** - Easy to implement (doubly-linked list + hashmap)
- ✅ **Temporal locality** - Recently accessed data likely accessed again
- ✅ **Fast** - O(1) lookup and eviction
- ✅ **Industry standard** - Proven pattern

**Cons:**
- ❌ **Scan resistance** - Sequential scan evicts hot data
- ❌ **Frequency ignored** - Doesn't consider how often accessed
- ❌ **One-time access problem** - Big scan pollutes cache

**Hit Rate:**
- General workload: 60-80%
- Temporal locality workload: 80-90%
- Sequential scan workload: 20-40% (poor)

**Real-World Usage:**
- Redis default (with approximation for efficiency)
- Most operating systems (page cache)
- Most applications

#### Option B: LFU (Least Frequently Used)

**Algorithm:**
```
Evict the item that was accessed fewest times

Example:
Item A: accessed 100 times
Item B: accessed 50 times
Item C: accessed 10 times
Cache full, need to evict
Evict C (least frequently used)
```

**Pros:**
- ✅ **Frequency awareness** - Keeps truly hot data
- ✅ **Better for steady workloads** - Captures long-term patterns
- ✅ **Scan resistant** - One-time scans don't evict hot data

**Cons:**
- ❌ **Complex** - Need to track access counts
- ❌ **Memory overhead** - Store counters for each item
- ❌ **Slow to adapt** - Old hot data stays even if no longer hot
- ❌ **Cold start problem** - New data has low count, immediately evicted

**Hit Rate:**
- Stable workload: 70-90%
- Changing workload: 40-60% (poor)

**Real-World Usage:**
- Redis offers LFU mode (`allkeys-lfu`)
- CDNs for video caching
- Some specialized systems

#### Option C: TTL (Time-To-Live)

**Algorithm:**
```
Evict items based on expiration time, not access pattern

Example:
Item A: expires in 10 seconds
Item B: expires in 60 seconds
Item C: expires in 300 seconds
Evict A first (soonest expiration)
```

**Pros:**
- ✅ **Predictable** - Know exactly when item will be evicted
- ✅ **Time-sensitive** - Good for session data, temporary data
- ✅ **No access tracking** - Simpler than LRU/LFU

**Cons:**
- ❌ **Not adaptive** - Doesn't consider actual usage
- ❌ **May evict hot data** - If TTL short but data frequently accessed
- ❌ **Requires setting TTL** - Application must choose expiration

**Real-World Usage:**
- Session storage (expire after inactivity)
- Temporary tokens (API keys, OTP codes)
- Rate limiting counters

#### Option D: Random Eviction

**Algorithm:**
```
Pick random item and evict it
```

**Pros:**
- ✅ **Simplest possible** - No tracking needed
- ✅ **Fast** - O(1) with no overhead

**Cons:**
- ❌ **Poor hit rate** - 30-50% (terrible)
- ❌ **No pattern recognition** - Pure chance

**Only use for:** Development/testing

### Decision Made: LRU

**Why LRU Wins:**

1. **Temporal Locality:** Most applications exhibit temporal locality - if data was accessed recently, likely to be accessed again soon.

2. **Simplicity + Performance:** O(1) operations with simple data structure (doubly-linked list).

3. **Industry Proven:** LRU is the default for a reason - works well across diverse workloads.

4. **Redis Default:** Redis uses approximated LRU (samples random keys), which is even faster.

### Redis LRU Implementation

**Exact LRU (theoretical):**
```
Doubly-linked list: [Most Recent] ⟷ ... ⟷ [Least Recent]
Hashmap: key → node pointer

On access: Move node to front (most recent)
On eviction: Remove from back (least recent)
```

**Redis Approximated LRU (actual):**
```
Each key has last_access_time timestamp
On eviction:
  1. Sample N random keys (default: 5)
  2. Evict the one with oldest last_access_time
  3. Repeat if need more space

Why approximation?
- Much less memory (no linked list pointers)
- Still achieves 95%+ hit rate of exact LRU
- Faster for large datasets
```

### Configuration in Redis

```
maxmemory 2gb
maxmemory-policy allkeys-lru

Options:
- allkeys-lru: Evict any key using LRU (recommended for cache)
- volatile-lru: Only evict keys with TTL set
- allkeys-lfu: Use LFU instead of LRU
- volatile-ttl: Evict keys with earliest expiration
- allkeys-random: Random eviction (don't use in production)
- noeviction: Return error when memory full (for persistent data)
```

### Trade-offs Accepted

- **Scan Vulnerability:** Large sequential scans can evict hot data temporarily
  - *Acceptable:* Rare in practice, hot data gets re-cached quickly
- **Frequency Ignored:** Doesn't track how often items are accessed
  - *Acceptable:* Recent access is good proxy for frequency in most cases

### When to Reconsider

**Use LFU if:**
- Workload is stable (same hot keys for days/weeks)
- Scan resistance critical (lots of sequential access patterns)
- Can afford cold start period
- Example: Video CDN (popular videos accessed millions of times)

**Use TTL if:**
- Data has natural expiration (sessions, tokens)
- Application controls lifetime
- Predictable eviction more important than hit rate
- Example: Session store, rate limiting

**Hybrid Approach:**
- Use TTL for session data (has natural expiry)
- Use LRU for application data (no natural expiry)
- Separate Redis instances for different use cases

---

## 6. Consistency: Eventual vs Strong

### The Problem

With replication (master + replicas), reads can hit different nodes. Should we:
- Guarantee all reads see latest write (strong consistency)
- Allow reads to be stale (eventual consistency)

### Options Considered

#### Option A: Eventual Consistency ✅ **CHOSEN**

**Model:**
```
Write to Master → Async replicate → Replica lags 1-5ms
Read from Replica → May see stale data (up to 5ms old)
```

**Guarantees:**
- Writes are durable once acknowledged by master
- Reads from replicas may be stale for 1-5ms
- Eventually (within 5ms), all replicas converge

**Pros:**
- ✅ **Low latency** - No coordination overhead
- ✅ **High availability** - Can read from any replica
- ✅ **Read scaling** - Add replicas for more capacity
- ✅ **Master performance** - No waiting for replicas

**Cons:**
- ❌ **Stale reads** - Might read old data
- ❌ **Read-after-write inconsistency** - Write then read might not see write
- ❌ **Ordering issues** - Different replicas may have different orderings

**Real-World Usage:**
- Redis replication (default)
- DNS (eventually consistent)
- Most web applications

#### Option B: Strong Consistency

**Model:**
```
Write to Master → Wait for quorum → Acknowledge

Read:
- Option 1: Always read from master (stale-free guarantee)
- Option 2: Read from quorum (majority must agree)
```

**Pros:**
- ✅ **Always see latest** - No stale reads
- ✅ **Linearizability** - Writes and reads have total order
- ✅ **Simpler reasoning** - No "eventually" behavior

**Cons:**
- ❌ **Higher write latency** - Wait for replicas (5-10ms vs <1ms)
- ❌ **Limited read scaling** - Must read from master or quorum
- ❌ **Lower availability** - If quorum unavailable, can't read
- ❌ **Not native to Redis** - Must use Redis Raft modules

**Real-World Usage:**
- PostgreSQL synchronous replication
- Google Spanner
- etcd, Consul

### Decision Made: Eventual Consistency

**Why Eventual Consistency Wins for Cache:**

1. **Cache is Not Source of Truth:** Database is source of truth. Cache staleness is tolerable since data can be refetched.

2. **Performance Critical:** For cache, sub-millisecond latency is paramount. Can't afford 5-10ms for strong consistency.

3. **Read Scaling:** Need to scale reads horizontally. Strong consistency limits read scaling (must read from master or quorum).

4. **Availability:** If master fails, can still serve reads from replicas (slightly stale, but available).

### Handling Staleness

**Problem: Read-Your-Own-Writes**
```
User updates profile → Write to master
User immediately views profile → Read from replica → Sees old data ❌
```

**Solution 1: Read from Master**
```
function get_user_profile(user_id, current_user_id):
  if user_id == current_user_id:
    // Reading own data - use master (fresh)
    return master.get("user:" + user_id)
  else:
    // Reading others' data - replica OK (stale acceptable)
    return replica.get("user:" + user_id)
```

**Solution 2: Client Session Tracking**
```
On write: Server returns last_write_timestamp
On read: Client sends last_write_timestamp
If replica is behind client timestamp, read from master
```

**Solution 3: Sticky Sessions**
```
Route user's requests to same replica for short period after write
Gives time for replication to catch up
```

### Trade-offs Accepted

- **Stale Reads:** Replicas can lag 1-5ms behind master
  - *Acceptable:* For cache, 5ms staleness is fine
- **Read-Your-Own-Writes:** Might not see own write immediately
  - *Mitigation:* Critical paths read from master

### When to Reconsider

**Use Strong Consistency if:**
- Cache stores critical financial data
- Staleness is unacceptable (regulatory requirement)
- Application logic can't handle stale reads
- Using database that supports strong consistency (etcd, Consul)

Note: For most applications, if strong consistency is truly required, don't use cache - read directly from database.

---

## 7. Cluster Mode: Sentinel vs Redis Cluster

### The Problem

Need high availability and horizontal scaling. Redis offers two solutions:
1. **Redis Sentinel:** Monitoring + automatic failover
2. **Redis Cluster:** Built-in sharding + replication

### Options Considered

#### Option A: Redis Sentinel ✅ **CHOSEN for Initial Setup**

**Architecture:**
```
Master → Replica 1, Replica 2
Sentinel 1, Sentinel 2, Sentinel 3 (monitoring)

On master failure:
Sentinels detect → Vote → Promote replica → Update clients
```

**Pros:**
- ✅ **Simple** - Easy to understand and operate
- ✅ **Automatic failover** - No manual intervention
- ✅ **Client-side sharding** - Flexible (consistent hashing)
- ✅ **Gradual adoption** - Start simple, scale later
- ✅ **Any number of shards** - Not limited by Redis Cluster's 16,384 slots

**Cons:**
- ❌ **Manual sharding** - Application must implement
- ❌ **More moving parts** - Sentinel + sharding logic in client
- ❌ **Complex topology** - Multiple master-replica groups to manage

**Failover Time:** 9-13 seconds

**Real-World Usage:**
- Twitter (with Twemproxy for sharding)
- GitHub (with custom sharding)
- Instagram (Sentinel + client-side sharding)

#### Option B: Redis Cluster

**Architecture:**
```
16,384 hash slots distributed across nodes
Each node: master + replicas
No Sentinel needed (gossip protocol for failure detection)
```

**Pros:**
- ✅ **Built-in sharding** - No client-side logic needed
- ✅ **Automatic rebalancing** - Add/remove nodes easily
- ✅ **Better scaling** - Supports 1000+ nodes
- ✅ **No Sentinel** - One less component

**Cons:**
- ❌ **Complex** - Learning curve for operations
- ❌ **Limited cross-slot operations** - Can't operate on keys in different slots
- ❌ **MOVED redirects** - Client must follow redirects
- ❌ **16,384 slot limitation** - May not align with use case
- ❌ **All-or-nothing** - Either use cluster or don't (can't mix)

**Real-World Usage:**
- Some large-scale Redis deployments
- AWS ElastiCache Cluster Mode

### Decision Made: Redis Sentinel (Initially)

**Why Sentinel First:**

1. **Start Simple:** Most applications don't need Redis Cluster's complexity initially. Sentinel provides HA without forcing clustering.

2. **Flexible Sharding:** Client-side sharding (consistent hashing) offers more control than Redis Cluster's fixed 16,384 slots.

3. **Gradual Growth:** Can start with Sentinel, add sharding as needed, migrate to Cluster later if required.

4. **Cross-Key Operations:** Application might need multi-key operations (like MGET multiple keys), which is hard in Redis Cluster if keys hash to different slots.

### Migration Path

**Stage 1: Single Master + Sentinel (0-10 GB data)**
```
Master → Replica 1, Replica 2
Sentinel monitoring
No sharding needed
```

**Stage 2: Manual Sharding + Sentinel (10-100 GB data)**
```
Shard 1: Master 1 → Replicas
Shard 2: Master 2 → Replicas
Shard 3: Master 3 → Replicas
Client-side consistent hashing
Sentinel per shard
```

**Stage 3: Redis Cluster (100+ GB data, complexity justified)**
```
Migrate to Redis Cluster
Remove client-side sharding logic
Leverage auto-scaling, rebalancing
```

### Configuration Comparison

**Sentinel Config:**
```
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
sentinel parallel-syncs mymaster 1
```

**Redis Cluster Config:**
```
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
```

### Trade-offs Accepted

- **Manual Sharding:** Must implement in client
  - *Acceptable:* Use proven library (Jedis, lettuce, go-redis all support)
- **More Components:** Sentinel adds operational complexity
  - *Acceptable:* Simpler than Cluster for small-medium scale

### When to Reconsider

**Switch to Redis Cluster if:**
- Data > 100 GB (too many manual shards)
- Need 100+ Redis instances
- Want automatic rebalancing
- Cross-slot operations not needed
- Team has Redis Cluster expertise

**Stick with Sentinel if:**
- Data < 100 GB
- Team comfortable with client-side sharding
- Need maximum flexibility
- Want gradual scaling path

---

## Summary: Decision Matrix

| Decision | Chosen | Alternative | Primary Reason | Key Trade-off |
|----------|--------|-------------|----------------|---------------|
| **Technology** | Redis | Memcached | Rich data structures | Single-threaded performance |
| **Partitioning** | Consistent Hashing | Modulo | Minimal reshuffling | Complexity |
| **Replication** | Master-Replica | Masterless | Simplicity | Write bottleneck |
| **Write Policy** | Cache-Aside | Write-Through | Memory efficiency | Cache misses |
| **Eviction** | LRU | LFU | Works for most workloads | Scan vulnerability |
| **Consistency** | Eventual | Strong | Low latency | Stale reads |
| **Cluster Mode** | Sentinel | Redis Cluster | Start simple | Manual sharding |

---

## Evolution Path

### Stage 1: MVP (< 10 GB, < 10K QPS)
- **Setup:** Single Redis master + 2 replicas + 3 Sentinels
- **Sharding:** None
- **Rationale:** Simplicity, sufficient for most applications

### Stage 2: Growth (10-100 GB, 10K-100K QPS)
- **Setup:** 3-5 manual shards, each with master + replicas + Sentinel
- **Sharding:** Client-side consistent hashing
- **Rationale:** Scale reads (replicas) and capacity (shards)

### Stage 3: Scale (100+ GB, 100K+ QPS)
- **Setup:** Redis Cluster or large Sentinel deployment (10+ shards)
- **Sharding:** Redis Cluster automatic or refined client-side
- **Rationale:** Automatic management at scale

### Stage 4: Massive Scale (1+ TB, 1M+ QPS)
- **Setup:** Redis Cluster with 50+ nodes, geo-replication
- **Additional:** Proxy layer (Twemproxy/Envoy), monitoring
- **Rationale:** Multi-region, advanced features

---

This document captures the reasoning behind each design choice and should be updated as the system evolves.

