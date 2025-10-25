# URL Shortener: Design Decisions - This Over That

This document provides an in-depth analysis of all major architectural decisions made for the URL Shortener system design.

---

## Table of Contents

1. [Database Choice: PostgreSQL vs NoSQL](#1-database-choice-postgresql-vs-nosql)
2. [Alias Generation: Base62 vs MD5 Hash](#2-alias-generation-base62-vs-md5-hash)
3. [ID Generator: Snowflake vs Database Auto-increment](#3-id-generator-snowflake-vs-database-auto-increment)
4. [Caching Strategy: Cache-Aside vs Write-Through](#4-caching-strategy-cache-aside-vs-write-through)
5. [HTTP Redirect: 301 vs 302](#5-http-redirect-301-vs-302)
6. [Analytics: Async vs Inline](#6-analytics-async-vs-inline)
7. [Sharding Strategy: By Alias vs By User ID](#7-sharding-strategy-by-alias-vs-by-user-id)

---

## 1. Database Choice: PostgreSQL vs NoSQL

### The Problem

We need to store billions of URL mappings with the following requirements:
- **Uniqueness constraints** on short aliases (NO duplicates allowed)
- **ACID properties** for custom alias creation (prevent race conditions)
- **Relational data** for analytics (clicks, user associations)
- **High read throughput** (100:1 read-to-write ratio)

### Options Considered

#### Option A: PostgreSQL (Relational Database) ✅ **CHOSEN**

**Pros:**
- ✅ **Strong ACID guarantees** - Perfect for preventing duplicate aliases
- ✅ **Unique constraints** enforced at database level
- ✅ **Relational model** supports analytics queries elegantly
- ✅ **Mature ecosystem** with excellent tooling
- ✅ **Transactions** for complex operations
- ✅ **Foreign keys** for data integrity

**Cons:**
- ❌ **Vertical scaling challenges** beyond certain size
- ❌ **Write scaling** requires sharding (more complex)
- ❌ **Join operations** can be slow at massive scale

**Real-World Usage:**
- Bitly uses PostgreSQL for URL mappings
- GitHub uses PostgreSQL for core data

#### Option B: DynamoDB/Cassandra (NoSQL)

**Pros:**
- ✅ **Horizontal scalability** out of the box
- ✅ **High write throughput** (handles 100K+ writes/sec easily)
- ✅ **Built-in replication** across regions
- ✅ **No single point of failure**

**Cons:**
- ❌ **Eventual consistency** - Risk of duplicate aliases during race conditions
- ❌ **No unique constraints** - Must implement at application level
- ❌ **Complex conditional writes** for custom aliases
- ❌ **More expensive** at small scale
- ❌ **Limited query flexibility** (no joins, limited indexes)

**Real-World Usage:**
- Amazon uses DynamoDB for shopping cart
- Netflix uses Cassandra for viewing history

#### Option C: MongoDB (Document Store)

**Pros:**
- ✅ **Flexible schema**
- ✅ **Unique indexes** support
- ✅ **Horizontal scaling** with sharding
- ✅ **Good developer experience**

**Cons:**
- ❌ **Eventual consistency** in sharded setups
- ❌ **Less mature** than PostgreSQL for transactional workloads
- ❌ **Memory intensive**

### Decision Made: PostgreSQL

**Why PostgreSQL Wins:**

1. **ACID is Critical:** Custom alias creation MUST be atomic. Two users requesting the same alias must result in one success, one failure. PostgreSQL's UNIQUE constraint handles this perfectly at the database level.

2. **Scale is Manageable:** At 12 writes/sec and 1,157 reads/sec baseline, PostgreSQL can easily handle the load. Even at 10x scale (120 writes/sec), it's well within capacity.

3. **Sharding Path is Clear:** When we outgrow a single instance, we can shard by `short_alias` hash. Since all lookups are by primary key, sharded queries remain efficient.

4. **Analytics Requirements:** The relational model makes it easy to join URLs with user data, click events, and generate reports.

### Trade-offs Accepted

- **Vertical scaling first:** We start with a single powerful instance and vertical scaling before implementing sharding
- **Write bottleneck:** All writes go to the primary (replicas are read-only)
- **Sharding complexity:** Eventually need to implement application-level sharding

### When to Reconsider

**Switch to NoSQL (DynamoDB/Cassandra) if:**
- Write QPS consistently exceeds 1,000/sec
- Need multi-region active-active writes
- Geographic distribution requires local writes
- URL volume exceeds 10 TB (beyond practical PostgreSQL limits)

**Migration Path:**
1. Start with PostgreSQL (years 1-2)
2. Implement sharded PostgreSQL (years 3-4)
3. Consider hybrid: PostgreSQL for URLs, Cassandra for analytics
4. Full migration to NoSQL only if absolutely necessary

---

## 2. Alias Generation: Base62 vs MD5 Hash

### The Problem

We need a 7-8 character short alias that is:
- **Unique** across billions of URLs
- **Short** (the whole point of URL shortening)
- **URL-safe** (no special characters)
- **Efficient** to generate (low latency)

### Options Considered

#### Option A: Base62 Encoding of Sequential ID ✅ **CHOSEN**

**How it works:**
```
Sequential ID: 123456789
Base62 Encode: "8M0kX"
Guaranteed unique because ID is unique
```

**Pros:**
- ✅ **Guaranteed unique** (ID generator ensures this)
- ✅ **No collision handling** needed
- ✅ **Short** (7 chars = 3.5 trillion IDs)
- ✅ **Predictable length**
- ✅ **Fast** (simple integer-to-string conversion)
- ✅ **K-sortable** (roughly time-ordered)

**Cons:**
- ❌ **Requires ID generator service** (added dependency)
- ❌ **Slightly predictable** (can estimate total URLs from ID)

**Real-World Usage:**
- Bitly uses Base62 encoding
- YouTube uses Base64 encoding for video IDs

#### Option B: MD5/SHA-256 Hash of URL

**How it works:**
```
URL: https://example.com/very/long/path
MD5 Hash: d41d8cd98f00b204e9800998ecf8427e
First 7 chars: "d41d8cd"
```

**Pros:**
- ✅ **Deterministic** (same URL → same alias)
- ✅ **No coordination** needed
- ✅ **Simple architecture**
- ✅ **Built-in deduplication** (same URL always gets same short link)

**Cons:**
- ❌ **Collision handling is complex** (birthday paradox)
- ❌ **Collision probability increases** with scale:
  - At 1 billion URLs, collision probability ~0.001% (still requires handling)
- ❌ **Multiple DB queries** on collision (retry with salt)
- ❌ **Custom aliases impossible** (hash is deterministic)
- ❌ **Not sortable**

**Collision Handling Complexity:**
```
1. Hash URL → "abc1234"
2. Try to insert into DB
3. If conflict, hash(URL + "1") → "xyz5678"
4. Try to insert again
5. Repeat until success (up to N times)
```

**Real-World Usage:**
- Some URL shorteners use hash for deduplication check only

#### Option C: Random Generation

**How it works:**
```
Generate random 7-character string from Base62 alphabet
```

**Pros:**
- ✅ **No coordination** needed
- ✅ **Simple to implement**
- ✅ **Unpredictable**

**Cons:**
- ❌ **Birthday paradox** - collisions likely:
  - At 1 million URLs: 0.001% collision chance
  - At 1 billion URLs: ~20% collision chance!
- ❌ **Requires DB check before insert** (latency)
- ❌ **Retry loops** on collision

**When to Use:**
- Only for small scale (< 1 million URLs)

### Decision Made: Base62 Encoding

**Why Base62 Wins:**

1. **Zero Collision Risk:** Sequential IDs from Snowflake/database are guaranteed unique. No collision handling logic needed.

2. **Performance:** Simple integer-to-Base62 conversion is extremely fast (<1ms) with no DB queries needed.

3. **Scalability:** 7 characters gives us 3.5 trillion IDs - enough for decades.

4. **Custom Aliases Supported:** Base62 works alongside custom aliases. Hash-based approach makes custom aliases awkward.

### Trade-offs Accepted

- **ID Generator Dependency:** Need to run Snowflake ID generator service (or use database sequences)
- **Slight Predictability:** Users can estimate total URL count from IDs (mitigation: use random starting epoch)

### When to Reconsider

**Use MD5 Hash if:**
- Deduplication is critical (same URL should always get same short link)
- No infrastructure for ID generator
- Write volume is very low (< 1K URLs/day)

**Use Random Generation if:**
- Scale is tiny (< 100K URLs total)
- Simplicity is more important than efficiency

---

## 3. ID Generator: Snowflake vs Database Auto-increment

### The Problem

We need to generate globally unique IDs for:
- URL records
- Custom aliases (validation only)
- Analytics events

Requirements:
- **Globally unique** across all servers
- **High throughput** (support 10K+ IDs/sec)
- **No coordination** during generation (low latency)

### Options Considered

#### Option A: Snowflake (Distributed ID Generator) ✅ **CHOSEN**

**How it works:**
```
64-bit ID structure:
[1 bit: sign][41 bits: timestamp][10 bits: worker ID][12 bits: sequence]

Example ID: 7234891234567890
```

**Pros:**
- ✅ **No single point of failure** (each node generates independently)
- ✅ **Horizontal scaling** (add more nodes without coordination)
- ✅ **Time-sortable** (newer IDs > older IDs)
- ✅ **High throughput** (4,096 IDs/ms per node)
- ✅ **Low latency** (<1ms, no network call needed)

**Cons:**
- ❌ **Worker ID coordination** needed (use etcd/ZooKeeper)
- ❌ **Clock synchronization** required (NTP)
- ❌ **Clock drift handling** adds complexity

**Real-World Usage:**
- Twitter created Snowflake for tweet IDs
- Instagram uses similar algorithm
- Discord uses Snowflake for message IDs

#### Option B: Database Auto-increment

**How it works:**
```sql
CREATE TABLE urls (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    ...
);
```

**Pros:**
- ✅ **Simple** (no external service)
- ✅ **Strictly monotonic** (perfect ordering)
- ✅ **Built into database**

**Cons:**
- ❌ **Single point of contention** (all writes go to one sequence)
- ❌ **Write bottleneck** (can't scale horizontally)
- ❌ **Network latency** (need to query DB for each ID)
- ❌ **Sharding is complex** (need to partition sequence)

**Real-World Usage:**
- Small applications with single database

#### Option C: UUID v4 (Random)

**Pros:**
- ✅ **No coordination** whatsoever
- ✅ **Globally unique** (virtually zero collision chance)
- ✅ **Simple** (built into all languages)

**Cons:**
- ❌ **128 bits** (2x storage of Snowflake)
- ❌ **Not sortable** (random order)
- ❌ **Poor index performance** (random insertion pattern)
- ❌ **Not numeric** (stored as string or binary)

### Decision Made: Snowflake

**Why Snowflake Wins:**

1. **Horizontal Scalability:** Each application server can embed its own ID generator. No central bottleneck.

2. **Performance:** IDs are generated locally (no network call). <1ms latency vs 10-50ms for database round-trip.

3. **Time-Sortable:** Database indexes benefit from sequential insertion pattern (better B-tree performance).

4. **Future-Proof:** Can generate 4+ million IDs/second per node. Plenty of headroom for growth.

### Trade-offs Accepted

- **Worker ID Management:** Need etcd/ZooKeeper for worker ID assignment
- **Clock Dependency:** Requires NTP for clock synchronization
- **Operational Complexity:** More moving parts than simple auto-increment

### When to Reconsider

**Use Database Auto-increment if:**
- Single database, no plans to scale horizontally
- Write volume < 100/sec
- Strict ordering is required
- Simplicity is paramount

**Use UUID v4 if:**
- Client-side ID generation is needed
- No central coordination possible
- Storage cost is not a concern

---

## 4. Caching Strategy: Cache-Aside vs Write-Through

### The Problem

URL redirects are extremely read-heavy (100:1 read-to-write ratio). We need caching to:
- Reduce database load by 90%+
- Achieve < 50ms redirect latency
- Handle traffic spikes gracefully

### Options Considered

#### Option A: Cache-Aside (Lazy Loading) ✅ **CHOSEN**

**Flow:**
```
Read: Check cache → Miss? → Query DB → Populate cache
Write: Update DB → Invalidate cache (next read will populate)
```

**Pros:**
- ✅ **Cache only what's accessed** (better memory utilization)
- ✅ **Cache misses are tolerable** (read-heavy workload)
- ✅ **Simple invalidation** (just delete from cache)
- ✅ **Works with any cache** (Redis, Memcached)

**Cons:**
- ❌ **Cache miss penalty** (requires DB query)
- ❌ **Stampede risk** (many requests hit DB on cache expiry)
- ❌ **Temporary inconsistency** (cache might serve stale data briefly)

**Real-World Usage:**
- Most read-heavy systems (Facebook, Twitter, Amazon)

#### Option B: Write-Through

**Flow:**
```
Write: Update DB AND cache simultaneously
Read: Always read from cache (cache is always warm)
```

**Pros:**
- ✅ **Cache always warm** (no cold misses)
- ✅ **Consistent reads** (cache matches DB)
- ✅ **Simple read path** (always from cache)

**Cons:**
- ❌ **Higher write latency** (wait for both DB and cache)
- ❌ **Cache pollution** (stores data that might never be read)
- ❌ **Wasted memory** (cache contains cold data)
- ❌ **More complex** (need to handle cache write failures)

**Real-World Usage:**
- Write-heavy systems where reads must be fast (inventory systems)

#### Option C: Write-Behind (Write-Back)

**Flow:**
```
Write: Update cache → Async write to DB
Read: Always from cache
```

**Pros:**
- ✅ **Extremely fast writes**
- ✅ **Batching possible**

**Cons:**
- ❌ **Data loss risk** (if cache fails before DB write)
- ❌ **Complex** (need persistent queue)
- ❌ **Not suitable for durability-critical data**

### Decision Made: Cache-Aside

**Why Cache-Aside Wins:**

1. **Read-Heavy Workload:** With 100:1 read-to-write ratio, optimizing reads is more important than writes. Cache-Aside does this perfectly.

2. **Memory Efficiency:** Only popular URLs end up in cache (80/20 rule - 20% of URLs get 80% of traffic). Why waste memory on URLs nobody clicks?

3. **Simple Invalidation:** When a URL is updated/deleted, just remove from cache. Next read will fetch fresh data from DB.

4. **Write Performance:** Writes don't wait for cache operations. Just update DB and optionally invalidate cache.

### Trade-offs Accepted

- **Cache Stampede Risk:** If a popular URL expires from cache, many concurrent requests might hit DB simultaneously
  - *Mitigation:* Use distributed locks or probabilistic early expiration
- **Temporary Stale Data:** Cache might serve old data briefly after update
  - *Acceptable:* URL mappings rarely change after creation

### When to Reconsider

**Use Write-Through if:**
- Write and read ratios are more balanced (e.g., 1:10 instead of 1:100)
- Stale data is unacceptable
- Cache must always be consistent with DB

**Use Write-Behind if:**
- Write performance is absolutely critical
- Can tolerate data loss (cache is source of truth temporarily)
- Have robust persistent queue infrastructure

---

## 5. HTTP Redirect: 301 vs 302

### The Problem

When a user visits a short URL, we redirect them to the long URL. HTTP offers two redirect status codes:
- **301 (Moved Permanently)**
- **302 (Found / Temporary Redirect)**

### Options Considered

#### Option A: HTTP 301 (Permanent Redirect)

**Behavior:**
```
First visit: Browser requests → Server returns 301 + long URL
Subsequent visits: Browser redirects directly (never hits server)
```

**Pros:**
- ✅ **Browser caching** (subsequent visits are instant)
- ✅ **Reduced server load** (90% of repeat visits bypass server)
- ✅ **Lower bandwidth costs**
- ✅ **Better user experience** (< 1ms after first visit)

**Cons:**
- ❌ **Can't track repeat visits** (analytics incomplete)
- ❌ **Can't update destination** (cached in browser forever)
- ❌ **Can't revoke short URLs** (cache persists)
- ❌ **No control** after first redirect

**Real-World Usage:**
- Permanent URL forwarding (domain changes)
- Static redirects that never change

#### Option B: HTTP 302 (Temporary Redirect) ✅ **CHOSEN for Analytics**

**Behavior:**
```
Every visit: Browser requests → Server returns 302 + long URL → Redirect
```

**Pros:**
- ✅ **Track every click** (complete analytics)
- ✅ **Can update destination** (changes take effect immediately)
- ✅ **Can revoke/expire** short URLs
- ✅ **Full control** over redirects
- ✅ **Can add features** (A/B testing, geographic routing)

**Cons:**
- ❌ **Higher server load** (every click hits server)
- ❌ **Slower** (~50ms instead of < 1ms for cached 301)
- ❌ **Higher bandwidth** costs

**Real-World Usage:**
- Bitly uses 302 (needs analytics)
- Marketing URL shorteners use 302 (track campaigns)

### Decision Made: HTTP 302 for Analytics Use Case

**Why 302 Wins for URL Shorteners:**

1. **Analytics is Core Value:** Users want click tracking. With 301, we lose visibility after first click.

2. **Flexibility:** Can update destination URL, expire links, add A/B testing, implement geographic routing.

3. **Security:** Can detect and block malicious destinations, revoke compromised links.

4. **Performance is Still Good:** 50ms latency is perfectly acceptable for most use cases. With caching, we can serve 90% of requests in < 5ms.

### Trade-offs Accepted

- **Higher Server Load:** Every click requires a server request
  - *Mitigation:* Use Redis cache (< 5ms for 90% of requests)
- **Slightly Slower:** 50ms vs < 1ms for browser-cached 301
  - *Acceptable:* Users won't notice 50ms delay

### When to Reconsider

**Use HTTP 301 if:**
- Analytics not required
- URLs never change or expire
- Need to minimize server costs
- Optimizing for absolute lowest latency

**Hybrid Approach:**
- Use 302 for authenticated users (track their clicks)
- Use 301 for anonymous users (reduce load)
- Use 302 for paid/premium links (analytics required)
- Use 301 for free/public links (no analytics needed)

---

## 6. Analytics: Async vs Inline

### The Problem

URL shorteners need to track clicks for analytics:
- Click counts
- Geographic distribution
- Referrer sources
- User agents (browser/device)
- Timestamps

Should we track clicks synchronously (before redirect) or asynchronously (after redirect)?

### Options Considered

#### Option A: Asynchronous Tracking ✅ **CHOSEN**

**Flow:**
```
Request → Cache Lookup → Fire-and-Forget Analytics Event → Immediate Redirect
                                        ↓
                                   Kafka Queue
                                        ↓
                              Analytics Consumer → ClickHouse
```

**Pros:**
- ✅ **No latency impact** on redirects (non-blocking)
- ✅ **Redirect performance** unaffected by analytics DB slowness
- ✅ **Scalable** (analytics processing separate from serving)
- ✅ **Resilient** (analytics failure doesn't break redirects)
- ✅ **Batching possible** (write 1000 events at once)

**Cons:**
- ❌ **Eventual consistency** (analytics lags by seconds)
- ❌ **Potential data loss** (if queue/consumer fails)
- ❌ **Complex** (requires message queue infrastructure)

**Real-World Usage:**
- Google Analytics uses async (beacon API)
- Most high-traffic services use async analytics

#### Option B: Inline Synchronous Tracking

**Flow:**
```
Request → Cache Lookup → Write to Analytics DB → Redirect
```

**Pros:**
- ✅ **Immediate consistency** (analytics available instantly)
- ✅ **Simple** (no queue infrastructure)
- ✅ **No data loss** (transaction with redirect)

**Cons:**
- ❌ **Adds latency** (50ms+ to redirect)
- ❌ **Analytics DB slowness** affects user experience
- ❌ **Single point of failure** (analytics DB down = redirects slow)
- ❌ **Scaling complexity** (analytics writes don't scale with redirects)

### Decision Made: Asynchronous Tracking

**Why Async Wins:**

1. **User Experience First:** Redirect latency is critical. Adding 50ms+ for analytics is unacceptable.

2. **Separation of Concerns:** Redirect serving (read-heavy, low-latency) has different requirements than analytics (write-heavy, high-throughput).

3. **Resilience:** Analytics DB problems shouldn't affect redirects. Fire-and-forget ensures redirects always work.

4. **Scalability:** Can scale redirect servers independently from analytics infrastructure.

### Implementation Details

**Immediate Counter Update (Redis):**
```
On redirect:
- Increment Redis counter: clicks:abc123
- Publish event to Kafka (fire-and-forget)
- Redirect immediately
```

**Batch Processing (ClickHouse/Cassandra):**
```
Analytics Consumer:
- Read 1000 events from Kafka
- Batch insert to ClickHouse
- Acknowledge Kafka offset
```

### Trade-offs Accepted

- **Eventual Consistency:** Analytics dashboards lag by 1-5 seconds
  - *Acceptable:* Real-time analytics not critical for most use cases
- **Potential Data Loss:** If Kafka fails, some events might be lost
  - *Mitigation:* Kafka provides durability guarantees (replication, persistence)

### When to Reconsider

**Use Synchronous Tracking if:**
- Real-time analytics absolutely required
- Very low traffic (< 100 requests/sec)
- No message queue infrastructure available
- Analytics failure should prevent redirect (audit/compliance requirement)

---

## 7. Sharding Strategy: By Alias vs By User ID

### The Problem

When database grows beyond single server capacity, need to shard (partition) data across multiple database instances.

Sharding key choices:
- Shard by `short_alias` (hash of the short URL)
- Shard by `user_id` (who created the URL)
- Shard by `created_at` (time-based)

### Options Considered

#### Option A: Shard by Short Alias (Hash) ✅ **CHOSEN**

**How it works:**
```
short_alias = "abc123"
shard_id = hash(short_alias) % num_shards
Query shard_id for URL lookup
```

**Pros:**
- ✅ **Perfect for redirect workload** (lookups by alias)
- ✅ **Even distribution** (hashing distributes uniformly)
- ✅ **Single-shard queries** (all lookups hit one shard)
- ✅ **No hotspots** (traffic distributed evenly)

**Cons:**
- ❌ **User queries scatter** (get all my URLs requires querying all shards)
- ❌ **Analytics complex** (need to aggregate across shards)

**Real-World Usage:**
- Bitly likely uses hash-based sharding
- Instagram shards photos by photo ID

#### Option B: Shard by User ID

**How it works:**
```
user_id = 12345
shard_id = hash(user_id) % num_shards
All URLs for user 12345 on same shard
```

**Pros:**
- ✅ **User queries efficient** (single shard for "my URLs")
- ✅ **Analytics easier** (user's data colocated)
- ✅ **Good for dashboard** (show my shortened URLs)

**Cons:**
- ❌ **Redirect lookup requires scatter-gather** (don't know which user owns alias)
  - Must query all shards to find alias!
- ❌ **Hotspot risk** (popular users cause uneven load)
- ❌ **Not suitable for anonymous URLs**

**Real-World Usage:**
- Social media platforms shard by user (Twitter shards tweets by user)

#### Option C: Shard by Time (Created_At)

**How it works:**
```
created_at = "2024-01"
shard_id = month_to_shard_mapping["2024-01"]
```

**Pros:**
- ✅ **Easy archival** (old shards can be moved to cold storage)
- ✅ **Time-based queries efficient** (recent URLs on hot shards)

**Cons:**
- ❌ **Uneven load** (newest shard gets all writes)
- ❌ **Redirect lookups scatter** (need to query all shards)
- ❌ **Hotspot on latest shard**

### Decision Made: Shard by Short Alias

**Why Sharding by Alias Wins:**

1. **Optimized for Read Path:** 99% of queries are redirects (lookup by alias). Single-shard queries are critical for performance.

2. **Even Load Distribution:** Hash-based sharding naturally distributes traffic evenly. No hotspots.

3. **Scalability:** Can add shards without rebalancing (use consistent hashing).

### Trade-offs Accepted

- **User Queries are Slow:** "Show me all my URLs" requires querying all shards
  - *Mitigation:* Maintain separate user_urls table sharded by user_id for dashboard queries
- **Analytics Complexity:** Need to aggregate data across shards
  - *Mitigation:* Use separate analytics database (Cassandra/ClickHouse) optimized for this

### When to Reconsider

**Shard by User ID if:**
- User dashboard is primary use case
- Redirect volume is low (can afford scatter-gather)
- Most URLs belong to registered users (not anonymous)

**Hybrid Approach:**
- Primary DB: Shard by alias (for redirects)
- Secondary DB: Shard by user_id (for user queries)
- Keep both in sync with CDC (Change Data Capture)

---

## Summary: Decision Matrix

| Decision | Chosen | Alternative | Primary Reason | Key Trade-off |
|----------|--------|-------------|----------------|---------------|
| **Database** | PostgreSQL | DynamoDB/Cassandra | ACID guarantees for uniqueness | Harder to scale writes |
| **Alias Generation** | Base62 | MD5 Hash | Zero collision risk | ID generator dependency |
| **ID Generator** | Snowflake | DB Auto-increment | Horizontal scalability | Clock synchronization needed |
| **Caching** | Cache-Aside | Write-Through | Memory efficiency | Possible cache stampede |
| **HTTP Redirect** | 302 | 301 | Analytics tracking | Higher server load |
| **Analytics** | Async | Inline | No latency impact | Eventual consistency |
| **Sharding** | By Alias | By User ID | Fast redirect lookups | User queries scatter |

---

## Evolution Path: How Decisions Change with Scale

### Stage 1: MVP (0-1M URLs, < 100 QPS)

- **Database:** Single PostgreSQL instance
- **Cache:** Single Redis instance
- **ID Generation:** PostgreSQL sequences (no Snowflake needed)
- **Sharding:** None

**Rationale:** Simplicity over scalability. Premature optimization is root of evil.

### Stage 2: Growth (1M-100M URLs, 100-1K QPS)

- **Database:** PostgreSQL with read replicas
- **Cache:** Redis Cluster (3-5 nodes)
- **ID Generation:** Introduce Snowflake
- **Sharding:** None yet (vertical scaling)

**Rationale:** Scale up before scaling out. Vertical scaling is simpler.

### Stage 3: Scale (100M-1B URLs, 1K-10K QPS)

- **Database:** Sharded PostgreSQL (by alias hash)
- **Cache:** Large Redis Cluster (10-20 nodes)
- **ID Generation:** Snowflake (multiple workers)
- **Sharding:** 8-16 shards
- **Analytics:** Separate Cassandra/ClickHouse cluster

**Rationale:** Horizontal scaling required. Separate analytics from serving.

### Stage 4: Massive Scale (1B+ URLs, 10K+ QPS)

- **Database:** Consider hybrid (PostgreSQL + DynamoDB)
- **Cache:** Global Redis with geo-replication
- **Multi-region deployment**
- **CDN for popular links**
- **Sharding:** 64+ shards
- **Analytics:** Dedicated data warehouse

**Rationale:** Multi-region for latency. Hybrid architecture for different workloads.

---

## Real-World Examples

### Bitly's Evolution

1. **Early Days:** Single PostgreSQL + Redis
2. **2015:** Sharded PostgreSQL, introduced GUID-based IDs
3. **2018:** Hybrid architecture (PostgreSQL + DynamoDB)
4. **2020:** Multi-region deployment, heavy use of caching

**Lessons Learned:**
- Started simple, added complexity only when needed
- Analytics became separate concern early
- Maintained backward compatibility throughout

### TinyURL's Simplicity

- **Database:** Likely simple MySQL/PostgreSQL
- **Caching:** Basic Redis
- **Scale:** Optimized for simplicity over features
- **No analytics** (keeps architecture simpler)

**Lessons Learned:**
- Can serve millions of redirects with simple architecture
- Not every URL shortener needs Bitly-level features
- Simplicity is a feature

---

This document should be revisited and updated as the system evolves and new patterns emerge.

