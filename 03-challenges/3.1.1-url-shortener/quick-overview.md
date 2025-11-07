# URL Shortener - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A URL shortener converts long URLs into short, memorable links (e.g., `bit.ly/abc123`). The system must handle:

- **Billions of URLs** stored (1.825B records over 5 years)
- **100:1 read-to-write ratio** (100M redirects/day vs 1M creations/day)
- **Sub-100ms redirect latency** (critical for UX - users expect instant redirects)
- **99.99% availability** (downtime affects millions of users)
- **Custom aliases** (users can specify their own short codes)
- **URL expiration** (optional time-based link expiration)
- **Basic analytics** (click tracking, referrers, geographic data)

---

## Requirements & Scale

### Functional Requirements

| Requirement         | Description                                    | Priority     |
|---------------------|------------------------------------------------|--------------|
| **URL Redirection** | Given short URL, redirect to original long URL | Must Have    |
| **URL Shortening**  | Generate unique 7-8 character short URL        | Must Have    |
| **Custom Aliases**  | Allow users to specify custom short aliases    | Must Have    |
| **URL Expiration**  | Support optional expiration time for links     | Should Have  |
| **Analytics**       | Track clicks, referrers, basic statistics      | Should Have  |
| **Bulk Operations** | Allow batch URL creation                       | Nice to Have |

### Scale Estimation

| Metric                    | Calculation                             | Result             |
|---------------------------|-----------------------------------------|--------------------|
| **URL Creates (Writes)**  | 1M/day ÷ (24h × 3600s)                  | ~12 QPS            |
| **URL Redirects (Reads)** | 100M/day ÷ (24h × 3600s)                | ~1,157 QPS         |
| **Storage (5 Years)**     | 1M/day × 365 × 5 = 1.825B records × 1KB | ~1.8 TB            |
| **Read-to-Write Ratio**   | 1,157 ÷ 12                              | 100:1 (read-heavy) |

---

## Key Components

| Component               | Purpose                              | Technology                              | Scalability              |
|-------------------------|--------------------------------------|-----------------------------------------|--------------------------|
| **Load Balancer**       | Distribute traffic, SSL termination  | NGINX, AWS ALB                          | Horizontal (multi-AZ)    |
| **Shortening Service**  | Generate unique aliases, write to DB | Stateless API servers (Go/Java/Python)  | Horizontal scaling       |
| **Redirection Service** | Fast lookups, HTTP redirects         | Stateless API servers (Go/Rust/Node.js) | Horizontal scaling       |
| **Redis Cache**         | Hot URL mappings, reduce DB load     | Redis Cluster (Cache-Aside pattern)     | Horizontal (sharding)    |
| **PostgreSQL**          | Persistent storage, source of truth  | Sharded by alias hash                   | Horizontal (sharding)    |
| **ID Generator**        | Globally unique sequential IDs       | Snowflake algorithm                     | Horizontal (distributed) |
| **Analytics**           | Click tracking, metrics              | Kafka + ClickHouse/Cassandra            | Horizontal (partitioned) |

---

## Write Path (Create Short URL)

**Complete Flow:** Client → Load Balancer → Shortening Service → ID Generator → Base62 Encode → Database → Cache →
Response

**Detailed Steps:**

1. **Client Request** (0ms): User submits long URL via `POST /api/v1/shorten` with optional custom alias
2. **Load Balancer** (2ms): Routes request to available Shortening Service instance (round-robin/least-connections)
3. **Shortening Service** (5ms):
    - Validates URL format (scheme, domain, length)
    - Checks rate limits (IP-based, User ID-based, API Key-based)
    - Validates URL against blacklist (malware/phishing prevention)
4. **Two Paths:**
    - **Custom Alias Path:** Check cache for availability → Atomic INSERT to DB with UNIQUE constraint → If conflict:
      Return error
    - **Auto-Generated Path:** Get unique ID from ID Generator (Snowflake) → Base62 encode → Store in DB
5. **ID Generator** (10ms): Generates unique sequential ID using Snowflake algorithm (timestamp + machine ID + sequence)
6. **Base62 Encoding** (1ms): Converts numeric ID to short 7-character alias (e.g., 123456789 → "8M0kX")
    - Uses alphabet: `0-9, A-Z, a-z` (62 characters)
    - 7 chars = 3.5 trillion unique IDs
7. **Database Write** (20ms): Stores mapping `(short_alias, long_url, user_id, created_at, expires_at, status)` in
   sharded PostgreSQL
    - Sharding: `hash(short_alias) % num_shards`
    - Primary key: `short_alias` (clustered index for fast lookups)
8. **Cache Write** (5ms): Stores mapping in Redis with 24-hour TTL (non-blocking, fire-and-forget)
    - Key: `url:{short_alias}`
    - Value: `long_url`
    - TTL: 24 hours (balances freshness and performance)
9. **Response** (~50ms total): Returns JSON `{short_url: "https://short.ly/8M0kX"}` to client

**Latency Breakdown:**

- Load balancer: 2ms
- Validation & rate limiting: 5ms
- ID generation: 10ms
- Base62 encoding: 1ms
- Database write: 20ms
- Cache write: 5ms
- Network overhead: ~7ms
- **Total: ~50ms**

---

## Read Path (Redirect)

**Complete Flow:** Client → Load Balancer → Redirection Service → Redis Cache → (if miss) Database → Redirect → Async
Analytics

### Fast Path (90% - Cache Hit)

**Steps:**

1. **Client Request** (0ms): User visits short URL via `GET /{shortCode}` (e.g., `GET /8M0kX`)
2. **Load Balancer** (2ms): Routes request to available Redirection Service instance
3. **Cache Check** (1ms): Queries Redis with key `url:{shortCode}`
4. **Cache Hit:** Redis returns `long_url` immediately
5. **Async Analytics** (non-blocking): Publishes click event to Kafka queue
    - Event: `{short_alias, clicked_at, user_agent, ip, referer, country}`
    - Fire-and-forget (doesn't block redirect)
6. **HTTP 302 Redirect** (1ms): Returns `302 Found` with `Location: long_url` header
7. **Browser Redirect** (1ms): Browser automatically follows redirect to long URL

**Total Latency: < 5ms** (90% of requests)

### Slow Path (10% - Cache Miss)

**Steps:**

1. **Client Request** (0ms): User visits short URL
2. **Load Balancer** (2ms): Routes to Redirection Service
3. **Cache Check** (1ms): Queries Redis → **MISS**
4. **Database Query** (50ms):
    - Query: `SELECT long_url, expires_at, status FROM urls WHERE short_alias = ?`
    - Uses primary key index (fast lookup)
    - Validates: URL exists, not expired (`expires_at > NOW()`), status is `ACTIVE`
5. **Error Handling:**
    - If not found: Return `404 Not Found`
    - If expired: Return `410 Gone`
    - If blocked: Return `403 Forbidden`
6. **Cache Update** (5ms): Stores mapping in Redis with 24-hour TTL (prevents future misses)
7. **Async Analytics** (non-blocking): Publishes click event to Kafka
8. **HTTP 302 Redirect** (2ms): Returns redirect to long URL

**Total Latency: ~100ms** (10% of requests)

**Cache Hit Rate:** 85-90% for popular URLs (reduces database load by 9x)

---

## Key Design Decisions

### 1. Base62 Encoding

**Why Chosen:**

- ✅ **Guaranteed unique:** Sequential IDs from ID generator ensure no collisions
- ✅ **Short:** 7 characters = 3.5 trillion unique IDs (sufficient for decades)
- ✅ **URL-safe:** Uses only `0-9, A-Z, a-z` (no special encoding needed)
- ✅ **Predictable:** Sequential IDs are easy to manage and debug

**Alternatives Considered:**

- ❌ **MD5/SHA Hash:** Collision handling complex, not sortable, same URL creates duplicates
- ❌ **Random Generation:** Birthday paradox, requires DB checks for uniqueness
- ❌ **UUID:** Too long (128-bit = ~22 chars when Base62 encoded)

**Example:**

```
ID: 123456789
Base62 Encode: "8M0kX"
Short URL: https://short.ly/8M0kX
```

### 2. Cache-Aside Pattern

**Why Chosen:**

- ✅ **Optimize for reads:** 100:1 read-to-write ratio means cache is critical
- ✅ **High hit rate:** 85-90% of redirects served from cache (< 5ms)
- ✅ **Reduces DB load:** Only 10-15% of reads hit database
- ✅ **Simple to implement:** Application manages cache explicitly

**How It Works:**

- **Read:** Check cache → If miss, query DB → Update cache → Return result
- **Write:** Write to DB → Invalidate cache (or write-through for consistency)
- **TTL:** 24 hours (balances freshness and performance)

**Alternatives Considered:**

- ❌ **Write-Through:** Every write updates cache (unnecessary for read-heavy workload)
- ❌ **Write-Behind:** Risk of data loss if cache fails before DB write

### 3. PostgreSQL (Sharded)

**Why Chosen:**

- ✅ **ACID guarantees:** Critical for uniqueness constraints (custom aliases)
- ✅ **Manageable data volume:** ~1.8TB over 5 years (fits in relational DB)
- ✅ **Uniqueness constraints:** Database enforces `UNIQUE(short_alias)` atomically
- ✅ **Relations:** Can join with user tables for analytics

**Sharding Strategy:**

- **Shard Key:** `hash(short_alias) % num_shards`
- **Distribution:** Even distribution across shards (e.g., Shard 0: a-m, Shard 1: n-z)
- **Lookups:** Primary key lookups are fast across shards
- **Scaling:** Add shards as data grows (requires rehashing)

**Alternatives Considered:**

- ❌ **NoSQL (DynamoDB/Cassandra):** Harder to enforce uniqueness, eventual consistency issues
- ❌ **Single Database:** Doesn't scale beyond ~100M URLs

### 4. HTTP 302 (Temporary Redirect)

**Why Chosen:**

- ✅ **Track every click:** Every visit goes through server (enables analytics)
- ✅ **Flexible:** Can change destination URL without breaking links
- ✅ **Analytics:** Track repeat visits, geographic distribution, referrers

**Trade-offs:**

- ❌ **Higher server load:** Every request hits server (vs 301 which browsers cache)
- ❌ **Consistent latency:** ~50ms every time (vs 301 which is < 1ms after first visit)

**Alternative:**

- **HTTP 301 (Permanent):** Browsers cache redirect, reduces server load, but can't track repeat visits

**Recommendation:** Use **302 for analytics**, **301 for permanent links** (if analytics not needed)

### 5. Asynchronous Analytics

**Why Chosen:**

- ✅ **Don't block redirect:** Analytics processing doesn't slow down redirect response
- ✅ **Decoupled services:** Analytics failure doesn't affect redirects
- ✅ **Better scalability:** Analytics processing scales independently
- ✅ **Batch processing:** More efficient writes to analytics database

**Implementation:**

- **Fire-and-forget:** Publish click event to Kafka queue (non-blocking)
- **Event Schema:** `{short_alias, clicked_at, user_agent, ip, referer, country}`
- **Processing:** Background workers consume from Kafka and write to ClickHouse/Cassandra
- **Real-time counters:** Optional Redis counters for immediate stats

**Alternative:**

- ❌ **Synchronous analytics:** Blocks redirect response, adds 50-100ms latency, couples services

---

## Data Model

### URLs Table Schema

```sql
CREATE TABLE urls (
    short_alias VARCHAR(8) PRIMARY KEY,      -- Clustered index, unique
    long_url VARCHAR(2048) NOT NULL,         -- Destination URL
    created_at TIMESTAMP DEFAULT NOW(),      -- Record creation time
    user_id BIGINT,                          -- Foreign key to users table
    expires_at TIMESTAMP,                    -- Optional expiration time
    status ENUM('ACTIVE', 'EXPIRED', 'BLOCKED', 'DELETED') DEFAULT 'ACTIVE',
    INDEX(user_id, created_at),              -- For user dashboards
    INDEX(expires_at)                        -- For cleanup jobs
);
```

**Storage per record:** ~1KB (including indexes)
**Total storage (5 years):** 1.825B records × 1KB = ~1.8TB

### Sharding Strategy

- **Shard Key:** `hash(short_alias) % num_shards`
- **Example:** 4 shards → Shard 0 (a-f), Shard 1 (g-m), Shard 2 (n-s), Shard 3 (t-z)
- **Per-Shard Setup:** 1 Primary (read+write) + 2 Replicas (read-only)
- **Replication:** Async replication with < 1 second lag

---

## Scalability Path

| Scale            | Architecture                            | Key Changes                                              |
|------------------|-----------------------------------------|----------------------------------------------------------|
| **0-1M URLs**    | Single DB + Redis                       | Simple monolithic setup, good for MVP                    |
| **1M-100M URLs** | DB replication + Redis cluster + CDN    | Add read replicas, Redis cluster, CDN for static content |
| **100M-1B URLs** | Database sharding by alias hash         | Horizontal scaling unlocked, 16+ shards                  |
| **1B+ URLs**     | Multi-region + Distributed ID generator | Global scale with low latency, active-active regions     |

**Key Insight:** Start simple, scale incrementally based on actual traffic patterns and bottlenecks.

---

## Common Anti-Patterns to Avoid

### 1. Race Conditions on Custom Aliases

❌ **Bad:** Check cache → Insert to DB (race condition window)

```
Time 0ms: Request A checks cache → alias available
Time 1ms: Request B checks cache → alias available
Time 2ms: Request A inserts → Success
Time 3ms: Request B inserts → Conflict!
```

✅ **Good:** Use database UNIQUE constraint, let DB enforce atomicity

```sql
INSERT INTO urls (short_alias, long_url) VALUES (?, ?)
-- If constraint violation: Return "Alias already taken"
```

### 2. Synchronous Analytics Blocking Redirects

❌ **Bad:** Wait for analytics DB write before redirecting (adds 50-100ms latency)

✅ **Good:** Fire-and-forget async tracking to Kafka queue (redirect in < 5ms)

### 3. Cache Stampede on Popular URLs

❌ **Bad:** When cache expires, 1,000+ concurrent requests all hit database

✅ **Good:** Distributed locking (only first request queries DB) or probabilistic early expiration

### 4. No Rate Limiting

❌ **Bad:** Attackers can create millions of URLs, exhaust storage

✅ **Good:** Multi-layered rate limiting (IP: 10/hour, User: 100/hour, API Key: custom)

### 5. Not Validating URLs

❌ **Bad:** Accept `javascript:alert('xss')` or `http://localhost:6379` (XSS/SSRF attacks)

✅ **Good:** Multi-layer validation (scheme, domain, length, blacklist, reachability check)

### 6. Storing Analytics in Same Table

❌ **Bad:** `UPDATE urls SET clicks = clicks + 1` on every redirect (write amplification)

✅ **Good:** Separate write-optimized analytics DB (Cassandra/ClickHouse), Redis counters for real-time

### 7. No Cache Fallback

❌ **Bad:** If Redis fails, entire service fails

✅ **Good:** Graceful degradation (try-catch, fallback to database, slower but available)

### 8. Sequential IDs Without Encoding

❌ **Bad:** Aliases like `1`, `2`, `3` are predictable, easy to enumerate

✅ **Good:** Base62 encode (e.g., `7234891234` → `aB3xY9`, appears random)

---

## Performance Metrics

| Metric                        | Target            | Actual              | Notes                                      |
|-------------------------------|-------------------|---------------------|--------------------------------------------|
| **Write Latency**             | < 100ms           | ~50ms               | ID generation + DB write + cache write     |
| **Read Latency (Cache Hit)**  | < 10ms            | < 5ms               | 90% of requests (Redis GET)                |
| **Read Latency (Cache Miss)** | < 200ms           | ~100ms              | 10% of requests (Redis + DB query)         |
| **Cache Hit Rate**            | > 80%             | 85-90%              | Popular URLs cached, reduces DB load by 9x |
| **Throughput**                | 1K+ redirects/sec | 100K+ with scaling  | Horizontal scaling via load balancer       |
| **Availability**              | 99.99%            | Multi-AZ + failover | < 52 minutes downtime/year                 |
| **Database Load**             | -                 | 10-15% of reads     | 85-90% served from cache                   |

---

## Technology Stack

| Layer             | Technology                                         | Rationale                                                 |
|-------------------|----------------------------------------------------|-----------------------------------------------------------|
| **Load Balancer** | NGINX or AWS ALB                                   | SSL termination, health checks, traffic distribution      |
| **API Servers**   | Go (performance) or Python/FastAPI (dev speed)     | Stateless, horizontally scalable                          |
| **Cache**         | Redis Cluster (multi-AZ, auto-failover)            | In-memory, sub-millisecond latency, high throughput       |
| **Database**      | PostgreSQL (sharded, primary-replica)              | ACID guarantees, uniqueness constraints, manageable scale |
| **ID Generator**  | Snowflake algorithm (embedded or separate service) | Distributed, no coordination, globally unique             |
| **Analytics**     | Kafka + ClickHouse/Cassandra                       | High-throughput event streaming, write-optimized storage  |
| **Monitoring**    | Prometheus + Grafana + Jaeger                      | Metrics, dashboards, distributed tracing                  |

---

## Availability & Fault Tolerance

### Single Points of Failure (SPOFs)

| Component           | Impact if Failed                | Mitigation                                                      |
|---------------------|---------------------------------|-----------------------------------------------------------------|
| **ID Generator**    | No new URLs can be created      | Distributed Snowflake (each server generates IDs independently) |
| **Redis Cache**     | High DB load, increased latency | Redis Cluster with replication, circuit breaker pattern         |
| **Database Master** | No writes (creates/updates)     | PostgreSQL automatic failover (Patroni + etcd), < 30s failover  |
| **API Servers**     | Reduced capacity                | Auto-scaling group with health checks, 3-20 instances           |

### High Availability Strategies

**Database Layer:**

- Primary-Replica setup: 1 Primary (writes) + 2+ Replicas (reads)
- Automatic failover: < 30 seconds
- Replication lag: < 1 second

**Cache Layer:**

- Redis Cluster: 6 nodes (3 masters + 3 replicas)
- Automatic failover with Redis Sentinel
- Circuit breaker: If cluster fails → Serve from DB (slower but functional)

**Application Layer:**

- Auto-scaling: 3-20 instances across 3 AZs
- Scale up: CPU > 70% for 2 minutes
- Scale down: CPU < 30% for 5 minutes
- Health checks: GET /health every 10 seconds

---

## Quick Reference

### Scale Estimation

- **1M URL creations/day** = ~12 QPS writes
- **100M redirects/day** = ~1,157 QPS reads
- **5 years storage** = ~1.8TB (1.825B records × 1KB)
- **Read-to-write ratio** = 100:1 (read-heavy workload)

### Capacity

- **7-char Base62** = 3.5 trillion unique URLs (62^7)
- **6-char Base62** = 56.8 billion unique URLs (62^6)
- **5-char Base62** = 916 million unique URLs (62^5)

### Cost Estimate

- **~$1,500-$3,000/month** for 1M creations + 100M redirects/day
- **Optimizations:** CDN (reduce data transfer by 80%), reserved instances (save 40%)

### Key Algorithms

- **Base62 Encoding:** Convert numeric ID to short string (see pseudocode.md)
- **Snowflake ID Generation:** Distributed unique ID generation (timestamp + machine ID + sequence)
- **Cache-Aside Pattern:** Check cache → If miss, query DB → Update cache
- **Sharding:** `hash(short_alias) % num_shards` for even distribution

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, components, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (write path, read path, failure
  scenarios)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (Base62, ID generation, caching)