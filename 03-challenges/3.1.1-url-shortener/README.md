# 3.1.1 Design URL Shortener (TinyURL/Bitly)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions. 
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## 1. Problem Statement

Design a highly available, scalable URL shortening service like TinyURL or Bitly that can convert long URLs into short,
memorable links. The system must handle billions of URLs, support extremely high read throughput (100:1 read-to-write
ratio),
provide sub-100ms redirect latency, and support custom aliases, expiration, and basic analytics.

---

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

| Requirement         | Description                                                    | Priority     |
|---------------------|----------------------------------------------------------------|--------------|
| **URL Redirection** | Given a short URL, redirect user to the original long URL      | Must Have    |
| **URL Shortening**  | Generate a unique, short URL (7-8 characters) for any long URL | Must Have    |
| **Custom Aliases**  | Allow users to specify custom short aliases (if available)     | Must Have    |
| **URL Expiration**  | Support optional expiration time for links                     | Should Have  |
| **Analytics**       | Track clicks, referrers, and basic statistics                  | Should Have  |
| **Bulk Operations** | Allow batch URL creation                                       | Nice to Have |

### Non-Functional Requirements (NFRs)

| Requirement           | Target              | Rationale                          |
|-----------------------|---------------------|------------------------------------|
| **Low Latency**       | < 50 ms (p99)       | Fast redirects critical for UX     |
| **High Availability** | 99.99% uptime       | Downtime affects millions of users |
| **High Throughput**   | 1000+ redirects/sec | Read-heavy workload                |
| **Durability**        | 99.999%             | URLs must never be lost            |
| **Scalability**       | Billions of URLs    | Long-term growth                   |

### Scale Estimation

| Metric                | Assumption                                                                       | Calculation                                               | Result                                      |
|-----------------------|----------------------------------------------------------------------------------|-----------------------------------------------------------|---------------------------------------------|
| Total Users           | 500 Million ($\text{MAU}$)                                                       | -                                                         | -                                           |
| URL Creates (Writes)  | 1 Million per day ($\text{QPS}_{w}$)                                             | $\frac{1 \text{M}}{24 \text{h} \times 3600 \text{s/h}}$   | $\sim 12$ Writes per second ($\text{QPS}$)  |
| URL Redirects (Reads) | 100 Million per day ($\text{QPS}_{r}$)                                           | $\frac{100 \text{M}}{24 \text{h} \times 3600 \text{s/h}}$ | $\sim 1157$ Reads per second ($\text{QPS}$) |
| Storage (5 Years)     | $1 \text{M} \text{URLs}/\text{day} \times 365 \times 5 = 1.825 \text{B}$ records | $1.825 \text{B} \times 1 \text{kB}/\text{record}$         | $\sim 1.8 \text{TB}$ total storage          |

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

### System Overview

```
                           ┌─────────────────────────────────┐
                           │         Users/Clients           │
                           └──────────────┬──────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                    ▼                     ▼                     ▼
          ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
          │  Load Balancer  │   │  Load Balancer  │   │  Load Balancer  │
          │   (US-East-1)   │   │   (EU-West-1)   │   │   (AP-South-1)  │
          └────────┬────────┘   └────────┬────────┘   └────────┬────────┘
                   │                     │                     │
                   └─────────────────────┼─────────────────────┘
                                         │
                         ┌───────────────┴───────────────┐
                         │                               │
            Write Path (12 QPS)            Read Path (1,157 QPS)
                         │                               │
                         ▼                               ▼
            ┌─────────────────────┐       ┌─────────────────────┐
            │ Shortening Service  │       │ Redirection Service │
            │  (Stateless)        │       │  (Stateless)        │
            │                     │       │                     │
            │ 1. Get unique ID    │       │ 1. Check cache      │
            │ 2. Base62 encode    │       │ 2. Return 301/302   │
            │ 3. Save to DB       │       │ 3. Track analytics  │
            │ 4. Cache result     │       │                     │
            └──────────┬──────────┘       └──────────┬──────────┘
                       │                             │
                       │                             ▼
                       │               ┌──────────────────────────┐
                       │               │    Redis Cache Cluster   │
                       │               │  (Cache-Aside Pattern)   │
                       │               │                          │
                       │               │  short_url → long_url    │
                       │               │  TTL: 24 hours           │
                       │               │  Hit Rate: 90%+          │
                       │               └──────────────────────────┘
                       │                             │
                       └──────────────┬──────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │  PostgreSQL (Sharded by alias hash) │
                    │                                     │
                    │  ┌──────────┐  ┌──────────┐       │
                    │  │ Shard 0  │  │ Shard 1  │  ...  │
                    │  │ a-m      │  │ n-z      │       │
                    │  └──────────┘  └──────────┘       │
                    │                                     │
                    │  Primary-Replica (Async Repl)      │
                    └─────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │   ID Generator Service (Snowflake)  │
                    │   Generates unique sequential IDs   │
                    └─────────────────────────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │  Analytics Service (Optional)       │
                    │  Kafka → ClickHouse/Cassandra       │
                    └─────────────────────────────────────┘
```

### Key Components

| Component               | Responsibility                       | Technology Options            | Scalability              |
|-------------------------|--------------------------------------|-------------------------------|--------------------------|
| **Load Balancer**       | Distribute traffic, SSL termination  | NGINX, HAProxy, AWS ALB       | Horizontal (multi-AZ)    |
| **Shortening Service**  | Generate unique aliases, write to DB | Go, Java, Python (stateless)  | Horizontal               |
| **Redirection Service** | Fast lookups, HTTP redirects         | Go, Rust, Node.js (stateless) | Horizontal               |
| **Cache Layer**         | Hot URL mappings, reduce DB load     | Redis Cluster, Memcached      | Horizontal (sharding)    |
| **Database**            | Persistent storage, source of truth  | PostgreSQL, MySQL (sharded)   | Horizontal (sharding)    |
| **ID Generator**        | Globally unique sequential IDs       | Snowflake, Redis INCR, DB seq | Horizontal (distributed) |
| **Analytics**           | Click tracking, metrics              | Kafka + ClickHouse/Cassandra  | Horizontal (partitioned) |

---

## 4. Detailed Component Design

### 3.1 Data Model and Storage

Since we require **ACID** properties (we cannot afford to lose the mapping or have conflicting keys) and the total data
volume is manageable ($\sim 1.8 \text{TB}$), we start with a **Relational Database (PostgreSQL)**.

#### Schema ($\text{URLs}$ Table)

| Field           | Data Type                     | Notes                                                     |
|-----------------|-------------------------------|-----------------------------------------------------------|
| **short_alias** | $\text{VARCHAR}(\text{8})$    | Primary Key (Clustered Index), unique.                    |
| **long_url**    | $\text{VARCHAR}(\text{2048})$ | The destination URL.                                      |
| **created_at**  | $\text{TIMESTAMP}$            | Record creation time.                                     |
| **user_id**     | $\text{BIGINT}$               | Foreign key to the User service (for custom links/stats). |
| **expires_at**  | $\text{TIMESTAMP}$            | Optional expiration time.                                 |
| **status**      | $\text{ENUM}$                 | $\text{ACTIVE}$, $\text{EXPIRED}$, $\text{BLOCKED}$.      |

#### Database Scaling (Sharding)

The table must be **sharded** horizontally by the `short_alias` to distribute the read/write load across multiple
database instances. This is because lookups are based on the primary key, making lookups efficient across shards.

### 3.2 Alias Generation Strategy

The alias must be short (7-8 characters), unique, and URL-safe.

#### Alias Generation Comparison

| Strategy               | Pros                                           | Cons                                     | When to Use              |
|------------------------|------------------------------------------------|------------------------------------------|--------------------------|
| **Base62 Encoding**    | ✅ Guaranteed unique<br>✅ Sequential<br>✅ Short | ❌ Requires ID generator                  | **Recommended (chosen)** |
| **MD5/SHA Hash**       | ✅ No coordination<br>✅ Deterministic           | ❌ Collision handling<br>❌ Not sortable   | Deduplication needed     |
| **Random Generation**  | ✅ Simple<br>✅ No coordination                  | ❌ Birthday paradox<br>❌ DB checks needed | Low scale (<1M URLs)     |
| **Counter per Server** | ✅ Fast<br>✅ No coordination                    | ❌ Predictable<br>❌ Requires prefix       | Development/testing only |

#### Base62 Encoding Deep Dive

**Why Base62?**

- Uses characters: `A-Z, a-z, 0-9` (62 characters)
- URL-safe (no special encoding needed)
- Compact: 7 characters = $62^7 = 3.5$ trillion unique IDs

**Algorithm:**

The Base62 encoding algorithm uses an alphabet of 62 characters (0-9, A-Z, a-z) and performs repeated modulo-62 operations on the input number, appending each resulting character to build the encoded string in reverse order.

**Encoding Process:**
1. Take the numeric ID
2. Repeatedly divide by 62, taking the remainder
3. Map each remainder to the Base62 alphabet
4. Reverse the result to get the final encoded string

**Decoding Process:**
1. Iterate through each character in the encoded string
2. Multiply accumulator by 62 and add character's position in alphabet
3. Return the final numeric value

*For detailed implementation, see `pseudocode.md::base62_encode()` and `pseudocode.md::base62_decode()`*

**Example:**
```
ID 123456789 → encode → "8M0kX"

Capacity:
- 7-char Base62: 62^7 = 3,521,614,606,208 URLs
- 6-char Base62: 62^6 = 56,800,235,584 URLs
```

#### Write Path Flow (Detailed)

**URL Shortening Service Logic:**

The shortening service handles two paths:

**For Custom Aliases:**
1. Check cache for alias availability (fast path)
2. Attempt atomic database insert with unique constraint
3. If constraint violation occurs, return error
4. On success, cache the mapping and return short URL

**For Auto-Generated Aliases:**
1. Request unique ID from ID generator service
2. Encode the numeric ID using Base62 algorithm
3. Store mapping in database (no uniqueness check needed - ID is guaranteed unique)
4. Cache the mapping with 24-hour TTL
5. Return the short URL to client

The key design decisions are:
- **Cache-Aside pattern** on write (invalidate, don't update)
- **Atomic database constraints** prevent race conditions
- **Fire-and-forget** caching (failure doesn't block response)

*For detailed implementation, see `pseudocode.md::shorten_url()`*

**Write Path Sequence Diagram:**

```
Client                Shortening Service    ID Generator    Database          Cache
  │                          │                    │            │               │
  │──POST /shorten──────────>│                    │            │               │
  │  long_url                │                    │            │               │
  │                          │                    │            │               │
  │                          │──Get Next ID──────>│            │               │
  │                          │<───Returns 123456──│            │               │
  │                          │                    │            │               │
  │                          │ (Encode to Base62) │            │               │
  │                          │    123456 → 8M0kX  │            │               │
  │                          │                    │            │               │
  │                          │──INSERT INTO urls────────────>  │               │
  │                          │  (8M0kX, long_url) │            │               │
  │                          │<──────────OK───────────────────  │               │
  │                          │                    │            │               │
  │                          │──SET url:8M0kX ──────────────────────────────>  │
  │                          │  (TTL 24h)         │            │               │
  │                          │<───────────OK──────────────────────────────────  │
  │                          │                    │            │               │
  │<─ short_url: 8M0kX ──────│                    │            │               │
  │                          │                    │            │               │
```

### 3.3 Redirection Strategy (The Read Path)

The Read Path must prioritize speed and offload the database.

#### Design Decisions

| Choice               | Decision             | Rationale                                         | Trade-off                       |
|----------------------|----------------------|---------------------------------------------------|---------------------------------|
| **Database**         | Sharded PostgreSQL   | ACID for uniqueness, relations for analytics      | Write scaling harder than NoSQL |
| **Caching Strategy** | Cache-Aside with TTL | Read-heavy (100:1), optimize for reads            | Eventual consistency            |
| **Redirect Type**    | HTTP 301 (Permanent) | Browsers cache, reduces server load               | Can't track repeated visits     |
|                      | HTTP 302 (Temporary) | Every click goes through server, better analytics | Higher server load              |

#### HTTP 301 vs 302

| Aspect              | 301 (Permanent Redirect)                | 302 (Temporary Redirect)             | Recommendation        |
|---------------------|-----------------------------------------|--------------------------------------|-----------------------|
| **Browser Caching** | ✅ Yes (subsequent visits bypass server) | ❌ No (every visit hits server)       | -                     |
| **Analytics**       | ❌ Can't track repeat visits             | ✅ Can track every click              | -                     |
| **Server Load**     | ✅ Lower (cached by browser)             | ❌ Higher (every request hits server) | -                     |
| **Latency**         | ✅ Very fast (< 1ms after first visit)   | ⚠️ Consistent (~50ms every time)     | -                     |
| **Use Case**        | Permanent URL shorteners                | Click tracking, analytics required   | **302 for analytics** |

#### Read Path Implementation Logic

The redirection service implements a fast-path/slow-path pattern:

**Fast Path (Cache Hit - 90% of requests):**
1. Query Redis cache with key `url:{short_alias}`
2. If found, immediately issue HTTP 302 redirect
3. Fire asynchronous analytics event (non-blocking)
4. Total latency: < 5ms

**Slow Path (Cache Miss - 10% of requests):**
1. Query database for URL mapping
2. Validate expiration time and status
3. If invalid/expired, return HTTP 404 or 410
4. If valid, populate cache with 24-hour TTL
5. Fire asynchronous analytics event
6. Issue HTTP 302 redirect
7. Total latency: ~50-100ms

**Analytics Tracking (Async):**
- Publish click event to Kafka/Redis Stream (fire-and-forget)
- Increment real-time counter in Redis (optional)
- Batch process events for historical analytics
- **Critical:** Never block redirect on analytics

Key optimizations:
- **Cache-first strategy** for minimal latency
- **Asynchronous analytics** prevents blocking
- **Graceful degradation** if cache fails
- **TTL-based expiration** automatic cleanup

*For detailed implementation, see `pseudocode.md::handle_redirect()` and `pseudocode.md::async_track_click()`*

#### Read Path Sequence Diagram

```
Client              Redirection Service       Redis Cache         Database
  │                         │                      │                 │
  │──GET /abc123───────────>│                      │                 │
  │                         │                      │                 │
  │                         │──GET url:abc123─────>│                 │
  │                         │                      │                 │
  │                         │<─────MISS────────────│                 │
  │                         │                      │                 │
  │                         │──SELECT * FROM urls──────────────────> │
  │                         │  WHERE short_alias='abc123'            │
  │                         │<──────long_url─────────────────────────│
  │                         │                      │                 │
  │                         │──SET url:abc123──────>                 │
  │                         │  (long_url, TTL 24h) │                 │
  │                         │<─────OK──────────────│                 │
  │                         │                      │                 │
  │                         │ (Async: Track click) │                 │
  │                         │                      │                 │
  │<── HTTP 302 Redirect────│                      │                 │
  │    Location: long_url   │                      │                 │
  │                         │                      │                 │

--- Subsequent Request (Cache Hit) ---

  │──GET /abc123───────────>│                      │                 │
  │                         │──GET url:abc123─────>│                 │
  │                         │<───long_url──────────│                 │
  │                         │                      │                 │
  │<── HTTP 302 Redirect────│                      │                 │
  │    (< 5ms!)             │                      │                 │
```

#### Cache-Aside Pattern Benefits

| Aspect                 | Benefit                                            | Metric                 |
|------------------------|----------------------------------------------------|------------------------|
| **Cache Hit Latency**  | Redis GET: ~1ms                                    | < 5ms total            |
| **Cache Miss Latency** | Redis + DB: ~50ms (acceptable for 10% of requests) | < 100ms                |
| **Cache Hit Rate**     | 80-90% for popular URLs                            | Target: 85%+           |
| **Database Load**      | Reduced by 85-90%                                  | 10-15% of reads hit DB |

---

## 5. Availability and Fault Tolerance

### 5.1 Single Points of Failure (SPOFs)

**Potential SPOFs:**

| Component          | Impact if Failed                      | Mitigation Strategy                                      |
|--------------------|---------------------------------------|----------------------------------------------------------|
| **ID Generator**   | No new URLs can be created            | Use distributed Snowflake-style ID generation per server |
| **Redis Cache**    | High DB load, increased latency       | Redis Cluster with replication, circuit breaker pattern  |
| **Database Master**| No writes (creates/updates)           | PostgreSQL with automatic failover (Patroni + etcd)      |
| **API Servers**    | Reduced capacity                      | Auto-scaling group with health checks                    |

### 5.2 High Availability Strategies

**Database Layer:**

```
Primary-Replica Setup:
- 1 Primary (writes)
- 2+ Read Replicas (redirects, analytics)
- Automatic failover: < 30 seconds
- Replication lag: < 1 second

Failover Process:
1. Health check detects primary failure
2. Promote replica to primary
3. Update DNS/connection pool
4. Resume writes
```

**Cache Layer:**

```
Redis Cluster (6 nodes):
- 3 masters (sharded by alias hash)
- 3 replicas (one per master)
- Automatic failover with Redis Sentinel
- If cluster fails → Circuit breaker → Serve from DB
```

**Application Layer:**

```
Auto-Scaling Strategy:
- Min instances: 3 (across 3 AZs)
- Max instances: 20
- Scale up trigger: CPU > 70% for 2 minutes
- Scale down trigger: CPU < 30% for 5 minutes
- Health check: GET /health every 10 seconds
```

### 5.3 Disaster Recovery

**Backup Strategy:**

| Data Type        | Backup Frequency | Retention | Recovery Time Objective (RTO) |
|------------------|------------------|-----------|-------------------------------|
| **URL Mappings** | Continuous (WAL) | 30 days   | < 1 hour                      |
| **Analytics**    | Daily snapshots  | 90 days   | < 4 hours                     |
| **Redis State**  | RDB every 6 hours| 7 days    | < 15 minutes (rebuild cache)  |

**Multi-Region Strategy:**

For mission-critical deployments:

```
Active-Passive Setup:
- Primary Region: US-East (99% traffic)
- Backup Region: EU-West (1% canary traffic)
- Database replication: Async (lag < 5 seconds)
- DNS failover: Route53 health checks

Failover Triggers:
- Primary region error rate > 5%
- Primary region latency p99 > 1000ms
- Complete region outage

Recovery:
- Automatic DNS failover: 60 seconds
- Manual verification required
- Gradual traffic ramp-up to backup region
```

### 5.4 Circuit Breaker Pattern

**Protect Database from Cache Failures:**

```
Circuit Breaker States:
- CLOSED: Normal operation (cache miss → DB query)
- OPEN: DB overloaded (return cached error response)
- HALF-OPEN: Testing recovery (allow 10% traffic to DB)

Thresholds:
- Open circuit if: DB error rate > 50% over 10 seconds
- Retry after: 30 seconds (HALF-OPEN)
- Close circuit if: 5 consecutive successful DB queries

Benefits:
- Prevents cascading failures
- Fast failure response (no timeout waits)
- Automatic recovery testing
```

*See pseudocode.md::circuit_breaker_check() for implementation*

---

## 6. Bottlenecks and Future Scaling

1. **SPOF in Cache:** If the Redis Cluster fails, all traffic hits the $\text{DB}$, causing a
   potential $\text{Cache}$ $\text{Stampede}$ and $\text{DB}$ overload.
    - **Mitigation**: Implement a multi-master/active-active $\text{Redis}$ setup with automatic failover. Use
      a $\text{Circuit}$ $\text{Breaker}$ on the $\text{DB}$ calls in the $\text{Redirection}$ $\text{Service}$ to drop
      excess traffic gracefully.
2. **Write Scaling:** If the daily link creation rate grows beyond $\sim 100$ $\text{QPS}$, the $\text{RDBMS}$ will
   struggle with contention even with sharding.
    - **Future Scaling:** Decouple the creation process using a **Message Queue (Kafka).** The user gets a confirmation
      immediately, and the $\text{URL}$ $\text{Shortening}$ $\text{Service}$ processes the actual $\text{DB}$ write
      asynchronously.
3. **Rate Limiting:** Abusive users could $\text{DDoS}$ the $\text{Write}$ $\text{Path}$ by repeatedly submitting new
   URLs.
    - **Mitigation**: Enforce $\text{Rate}$ $\text{Limiting}$ at the $\text{API}$ $\text{Gateway}$ using the **Token
      Bucket Algorithm** (2.5.1), limited by $\text{User}$ $\text{ID}$ or $\text{IP}$ $\text{Address}$.

---

## 7. Common Anti-Patterns

### Anti-Pattern 1: Not Handling Race Conditions on Custom Aliases

**Problem:**

❌ **Check-Then-Act Race Condition**

Many developers check if an alias exists in cache, then insert into database. This creates a race condition window where two concurrent requests can both see the alias as available and both attempt to insert, potentially causing duplicates or errors.

**Timeline of Race Condition:**
```
Time 0ms:  Request A checks cache → alias available
Time 1ms:  Request B checks cache → alias available  
Time 2ms:  Request A inserts to DB → Success
Time 3ms:  Request B inserts to DB → Conflict! (but damage done)
```

**Better Approach:**

✅ **Let Database Enforce Atomicity**

Use database UNIQUE constraints and exception handling. The database guarantees atomicity - only one INSERT can succeed. This is the only safe way to handle concurrent custom alias requests.

**Benefits:**
- Atomic operation (no race window)
- Database handles locking internally
- Simple application logic
- Reliable under high concurrency

*See `pseudocode.md::create_custom_alias_good()` for implementation details vs `create_custom_alias_bad()` for the anti-pattern*

---

### Anti-Pattern 2: Synchronous Analytics Blocking Redirects

**Problem:**

❌ **Analytics in Critical Path**

Recording analytics data synchronously before redirecting adds 50-100ms latency to every redirect. Users perceive this as slowness. Worse, if the analytics service is slow or down, redirects become slow or fail entirely.

**Impact Analysis:**
- **User Experience:** 50ms baseline → 150ms with analytics (3x slower)
- **Availability:** Analytics DB down = Redirects fail (unnecessary coupling)
- **Scale:** Analytics writes don't scale with redirect traffic

**Better Approach:**

✅ **Fire-and-Forget Async Tracking**

Redirect the user immediately, then publish analytics event to a message queue for asynchronous processing. The redirect doesn't wait for analytics. If analytics fails, the redirect still succeeds.

**Benefits:**
- No latency impact (redirect in < 5ms)
- Decoupled services (analytics failure doesn't affect redirects)
- Better scalability (analytics processing scales independently)
- Batch processing possible (more efficient writes)

*See `pseudocode.md::redirect_with_async_analytics_good()` for implementation vs `redirect_with_blocking_analytics_bad()` for the anti-pattern*

---

### Anti-Pattern 3: Cache Stampede on Popular URLs

**Problem:**

❌ **Thundering Herd on Cache Expiry**

When a popular URL's cache entry expires, thousands of concurrent requests all experience a cache miss simultaneously. They all query the database at once, overwhelming it with identical queries.

**Impact:**
- Database receives 1,000+ identical queries in milliseconds
- Database CPU spikes to 100%
- Query latency increases from 10ms to 5,000ms
- Cascading failure possible if database can't handle load

**Solution 1: Distributed Locking**

✅ **Single-Flight Request Pattern**

Use a distributed lock so only the first request queries the database. Other concurrent requests wait for the first to complete and populate the cache, then read from cache.

**Solution 2: Probabilistic Early Expiration**

✅ **Proactive Cache Refresh**

For hot keys, probabilistically refresh the cache before it expires. This prevents the cache from ever going completely cold.

*See `pseudocode.md::get_url_with_lock()` and `pseudocode.md::get_url_with_probabilistic_refresh()` for implementations*

---

### Anti-Pattern 4: No Rate Limiting on URL Creation

**Problem:**

❌ **Unprotected Write Endpoint**

Without rate limiting, malicious users can abuse the URL shortening service by creating millions of URLs, exhausting:
- Database storage (billions of garbage URLs)
- ID space (waste sequential IDs)
- CPU resources (generate unnecessary aliases)

**Attack Scenarios:**
- **Storage exhaustion:** Create 1 billion URLs → Fill database
- **DDoS:** Send 100K requests/sec → Overload service
- **Spam:** Create millions of spam/phishing links

**Better Approach:**

✅ **Multi-Layered Rate Limiting**

Implement rate limits at multiple levels:

**By IP Address (Anonymous Users):**
- Limit: 10 URLs per hour
- Use case: Prevent basic abuse
- Storage: Redis counters with 1-hour TTL

**By User ID (Authenticated Users):**
- Limit: 100 URLs per hour (higher for legitimate users)
- Use case: Allow legitimate high-volume users
- Storage: Redis counters with 1-hour TTL

**By API Key (Enterprise):**
- Limit: Custom (negotiated)
- Use case: B2B integrations
- Billing/throttling based on plan

**Implementation Benefits:**
- **Redis INCR:** Atomic, fast counter increment
- **Automatic expiry:** TTL-based cleanup
- **Tiered limits:** Different rates for different user types
- **HTTP 429:** Standard rate limit response

*See `pseudocode.md::check_rate_limit()` and `pseudocode.md::shorten_with_validation()` for implementation*

---

### Anti-Pattern 5: Not Validating Input URLs

**Problem:**

❌ **Accepting Malicious URLs**

Without validation, attackers can submit malicious URLs that cause security vulnerabilities:
- **XSS:** `javascript:alert('xss')` - executes JavaScript
- **SSRF:** `http://localhost:6379` - access internal services
- **Phishing:** Shortened URLs hide malicious destinations
- **Invalid:** Broken URLs that waste storage

**Better Approach:**

✅ **Multi-Layer URL Validation**

Implement comprehensive validation before accepting URLs:

**1. Scheme Validation (XSS Prevention):**
- Allow only: `http://` and `https://`
- Block: `javascript:`, `data:`, `file:`, etc.
- Prevents code execution via URL

**2. Domain Validation (SSRF Prevention):**
- Block localhost: `127.0.0.1`, `localhost`, `0.0.0.0`
- Block private networks: `192.168.*`, `10.*`, `172.16-31.*`
- Prevents accessing internal infrastructure

**3. Length Validation:**
- Maximum length: 2048 characters (browser limit)
- Prevents database overflow attacks

**4. Optional Reachability Check:**
- HTTP HEAD request with 5-second timeout
- Verify URL actually exists
- Catch typos and dead links early

**5. Blacklist Check:**
- Query URL against known malware/phishing databases
- Google Safe Browsing API
- PhishTank integration

**Implementation Strategy:**
- Fail fast (reject invalid URLs immediately)
- Clear error messages (help users fix issues)
- Log suspicious attempts (detect patterns)
- Rate limit validation failures (prevent scanning)

*See `pseudocode.md::validate_url()` and `pseudocode.md::shorten_with_validation()` for detailed implementation*

---

### Anti-Pattern 6: Storing Everything in One Database

**Problem:**

```
❌ URL mappings and analytics in same table causes hot spots

CREATE TABLE urls (
  short_alias VARCHAR(8) PRIMARY KEY,
  long_url VARCHAR(2048),
  clicks BIGINT DEFAULT 0,  -- Updated on every redirect!
  last_clicked_at TIMESTAMP
);

-- Every redirect updates the row (write amplification)
UPDATE urls SET clicks = clicks + 1 WHERE short_alias = 'abc123';
```

**Better:**

```
✅ Separate hot and cold data

// URLs table (rarely updated) - PostgreSQL
CREATE TABLE urls (
  short_alias VARCHAR(8) PRIMARY KEY,
  long_url VARCHAR(2048),
  created_at TIMESTAMP
);

// Analytics (write-optimized, separate database) - Cassandra/ClickHouse
CREATE TABLE click_events (
  short_alias VARCHAR(8),
  clicked_at TIMESTAMP,
  user_agent VARCHAR(255),
  ip VARCHAR(45),
  PRIMARY KEY ((short_alias), clicked_at)
) WITH CLUSTERING ORDER BY (clicked_at DESC);

```

**Real-time counters:** Store in Redis, async write to analytics DB via message queue.

*See `pseudocode.md::track_click()` for implementation*

---

### Anti-Pattern 7: No Cache Fallback Strategy

**Problem:**

❌ **No fallback:** If Redis is down, cache.set() throws exception → entire service fails.

✅ **Solution:** Graceful degradation:
- Wrap all cache operations in try-catch
- If cache fails: Log error, fall back to database
- Continue serving requests (slower but available)
- Don't fail if cache.set() errors - continue without caching

*See `pseudocode.md::redirect()` for implementation with fallback*

---

### Anti-Pattern 8: Using Sequential IDs Directly as Aliases

**Problem:**

❌ **Sequential IDs** (`1`, `2`, `3`) are predictable → easy to enumerate → reveals usage statistics.

✅ **Solution:** Encode IDs using Base62:
- Generate sequential ID: `7234891234`
- Encode with Base62: `aB3xY9`
- Aliases appear random: `aB3xY9`, `kL9mP2`, `wX8qN5`
- Harder to enumerate, doesn't reveal total count

*See `pseudocode.md::base62_encode()` for implementation*

---

## 8. Alternative Approaches (Not Chosen)

### Approach A: NoSQL-First Design (DynamoDB/Cassandra)

**Architecture:**

- Use DynamoDB or Cassandra as primary database
- Leverage NoSQL's horizontal scalability
- Store mappings as simple key-value pairs

**Pros:**

- ✅ Excellent horizontal scalability
- ✅ Simple data model (key-value)
- ✅ Very high write throughput
- ✅ Built-in replication and high availability

**Cons:**

- ❌ More complex to enforce uniqueness constraints
- ❌ No ACID guarantees for custom alias conflicts
- ❌ Eventual consistency may cause issues during alias creation
- ❌ More expensive at small scale

**Why Not Chosen:**

- The requirement for **unique alias constraints** is better handled by RDBMS
- At 12 writes/sec, we don't need NoSQL-level write throughput yet
- ACID guarantees important for preventing duplicate aliases
- Better to start with proven relational model and scale later if needed

**When to Reconsider:**

- Write QPS exceeds 1,000 per second
- Data volume exceeds multiple terabytes
- Need multi-region active-active writes

---

### Approach B: Hash-Based Alias Generation (MD5/SHA)

**Architecture:**

- Use MD5 or SHA-256 hash of the long URL
- Take first 7 characters as the short alias
- Handle collisions with iteration (append counter)

**Pros:**

- ✅ Deterministic (same URL → same alias)
- ✅ No need for ID generator service
- ✅ Simpler architecture

**Cons:**

- ❌ **Collision handling is complex** (requires multiple DB checks)
- ❌ Aliases are not sequential or predictable
- ❌ Cannot support custom aliases easily
- ❌ Hash collisions increase with scale
- ❌ Same URL submitted twice creates duplicate entries

**Why Not Chosen:**

- Collision resolution adds latency and complexity
- Custom alias feature is harder to implement
- Base62 encoding of sequential ID is more elegant
- Deterministic hashing conflicts with custom aliases

**Example Collision Problem:**

```
URL1: https://example.com/page1 → MD5 → abc1234 (first 7 chars)
URL2: https://example.com/page2 → MD5 → abc1234 (collision!)
System must rehash with counter: hash(URL2 + "1") → xyz5678
```

---

### Approach C: Client-Side ID Generation

**Architecture:**

- Client generates UUID/GUID for each short URL
- No centralized ID generator needed

**Pros:**

- ✅ No single point of failure
- ✅ Infinite scalability
- ✅ No coordination needed

**Cons:**

- ❌ **UUIDs are 128-bit (too long for short URLs)**
- ❌ Not human-readable or memorable
- ❌ Base62 encoding of UUID still results in long strings (~22 chars)
- ❌ Cannot guarantee short length

**Why Not Chosen:**

- The entire point of URL shortener is **short** URLs
- UUID encoding defeats the purpose
- "tinyl.co/a5c9e1b2f8d4" is not short

---

## 9. Deep Dive: Handling Edge Cases

### Edge Case 1: Expired Links

**Problem:** Links with expiry need automatic cleanup

**Solution:**

```
// Background job (runs every hour)
Background job runs every hour:
1. Query database for expired links (`expires_at < NOW()`)
2. For each expired link:
   - Update status to 'EXPIRED'
   - Remove from cache
3. Log cleanup count

*See `pseudocode.md::cleanup_expired_links()` for implementation*

**Alternative: TTL in Database**

- Some databases (DynamoDB, Cassandra) support automatic TTL-based deletion
- More efficient than batch cleanup jobs

---

### Edge Case 2: Malicious/Inappropriate URLs

**Problem:** Users might create links to harmful content

**Solution:**

**Validation Steps:**
1. Validate URL format (scheme, domain, length)
2. Check against internal blacklist of banned domains
3. Optional: Scan with external safety service (Google Safe Browsing API)
4. If valid: Generate alias and save to database
5. If invalid: Return appropriate error

*See `pseudocode.md::create_short_url()` and `pseudocode.md::validate_url()` for implementation*

---

### Edge Case 3: Analytics and Click Tracking

**Problem:** Users want to know how many times their link was clicked

**Extended Schema:**

```sql
CREATE TABLE url_analytics (
    short_alias VARCHAR(8) PRIMARY KEY,
    total_clicks BIGINT DEFAULT 0,
    last_clicked_at TIMESTAMP,
    INDEX(short_alias)
);

CREATE TABLE click_events (
    event_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    short_alias VARCHAR(8),
    clicked_at TIMESTAMP,
    user_agent VARCHAR(255),
    ip_address VARCHAR(45),
    referer VARCHAR(2048),
    country VARCHAR(2),
    INDEX(short_alias, clicked_at)
);
```

**Architecture:**

```
User clicks short URL
    ↓
Redirection Service
    ↓
┌─────────────────────────────┐
│ 1. Increment counter (Redis)│
│ 2. Publish ClickEvent       │
│ 3. Redirect immediately     │
└─────────────────────────────┘
         ↓
    Kafka Queue
         ↓
Analytics Consumer
    ↓
Store in Cassandra/ClickHouse
(for long-term analytics)
```

**Key Decision:**

- **Asynchronous tracking** to avoid slowing down redirects
- Use **Redis counters** for real-time counts
- Use **batch inserts** to analytics DB for historical data

---

## 10. Monitoring and Observability

### Key Metrics to Track

| Metric                     | Type      | Threshold         | Alert Action                          |
|----------------------------|-----------|-------------------|---------------------------------------|
| **Redirect Latency (P99)** | Histogram | < 100ms           | If > 200ms, check cache hit rate      |
| **Cache Hit Rate**         | Gauge     | > 90%             | If < 80%, increase cache size or TTL  |
| **DB Connection Pool**     | Gauge     | < 80% utilization | If > 90%, scale DB connections        |
| **URL Creation QPS**       | Counter   | Monitor trends    | If spike, check for abuse             |
| **Error Rate (404s)**      | Counter   | < 1%              | If > 2%, investigate data consistency |
| **Redis Availability**     | Gauge     | 100%              | If down, trigger circuit breaker      |

### Distributed Tracing

**Trace Example:**

```
Request: GET /abc123
  ├─ Load Balancer [2ms]
  ├─ API Gateway [5ms]
  ├─ Redirection Service [45ms]
  │   ├─ Redis Cache Lookup [3ms] ← Cache hit!
  │   └─ HTTP 301 Redirect [1ms]
  └─ Total: 55ms
```

**Trace Example (Cache Miss):**

```
Request: GET /xyz789
  ├─ Load Balancer [2ms]
  ├─ API Gateway [5ms]
  ├─ Redirection Service [93ms]
  │   ├─ Redis Cache Lookup [3ms] ← Cache miss!
  │   ├─ Database Query [80ms] ← Slow!
  │   ├─ Cache Write [5ms]
  │   └─ HTTP 301 Redirect [2ms]
  └─ Total: 100ms
```

---

## 11. Interview Discussion Points

### Question 1: How would you handle 100× growth?

**Answer:**

- **Reads (Redirects):** Already horizontally scalable
    - Add more cache nodes (Redis Cluster)
    - Add more API servers behind load balancer
    - Implement CDN for popular links

- **Writes (Creation):** Bottleneck at ID generator
    - Use distributed ID generator (Snowflake-style)
    - Each server generates IDs independently
    - No single point of coordination

- **Database:** Shard by hash of short_alias
    - 10 shards → 100 shards
    - Use consistent hashing for shard assignment

---

### Question 2: What if custom aliases become 50% of traffic?

**Answer:**

- **Challenge:** Custom aliases can't use sequential ID generation
- **Solution:**
    1. Check Redis for alias availability (fast path)
    2. If available, atomically reserve in DB:
       ```sql
       INSERT INTO urls (short_alias, long_url, ...)
       VALUES (?, ?, ...)
       ON DUPLICATE KEY UPDATE short_alias = short_alias;
       -- If insert fails, alias is taken
       ```
    3. Use optimistic locking to handle race conditions

- **Performance Impact:**
    - Higher DB load (must check uniqueness)
    - May need stronger consistency (SERIALIZABLE isolation)
    - Consider pre-reserving popular aliases

---

### Question 3: How do you handle GDPR deletion requests?

**Answer:**

- **Requirements:**
    - Delete user's URLs within 30 days
    - Anonymize analytics data

- **Implementation:**
    1. Mark URLs as `status = 'DELETED'`
    2. Keep alias reserved (prevent reuse) but show 410 Gone
    3. Asynchronously purge from:
        - Primary database
        - Cache (immediate)
        - Analytics logs (anonymize user_id)
        - Backups (within retention window)

- **Compliance:**
  ```sql
  -- Soft delete
  UPDATE urls SET status = 'DELETED', long_url = NULL, user_id = NULL
  WHERE user_id = ? AND status = 'ACTIVE';
  
  -- Background job for hard delete
  DELETE FROM urls WHERE status = 'DELETED' AND deleted_at < NOW() - INTERVAL 30 DAY;
  ```

---

### Question 4: How would you prevent abuse (spam, phishing)?

**Answer:**

- **Rate Limiting:**
    - 10 URLs per hour for anonymous users
    - 100 URLs per hour for authenticated users
    - Use Token Bucket algorithm at API Gateway

- **Domain Blacklisting:**
    - Maintain list of known malicious domains
    - Check against Google Safe Browsing API

- **URL Validation:**
    - Verify URL is reachable (HTTP HEAD request)
    - Check SSL certificate validity
    - Scan content with VirusTotal API

- **User Reputation:**
    - Track abuse reports per user
    - Automatically ban users with high spam ratio
    - Require CAPTCHA for suspicious activity

---

### Question 5: How do you ensure 99.99% availability?

**Answer:**

- **Eliminate Single Points of Failure:**
    - Multi-AZ deployment (at least 3 zones)
    - Redundant load balancers
    - Redis Cluster with automatic failover
    - Database primary-replica setup with automatic promotion

- **Circuit Breakers:**
    - Protect against cascading failures
    - Fail fast when cache/DB is slow

- **Health Checks:**
    - Load balancer removes unhealthy nodes
    - Kubernetes auto-restarts failed pods

- **Graceful Degradation:**
    - If Redis fails, serve from DB (slower but functional)
    - If analytics fails, still serve redirects

- **Disaster Recovery:**
    - Multi-region deployment for critical services
    - Regular database backups
    - Tested failover procedures

---

## 12. Comparison with Real-World Systems

| Feature            | Our Design         | Bitly       | TinyURL   | Short.io  |
|--------------------|--------------------|-------------|-----------|-----------|
| **Alias Length**   | 7 chars (Base62)   | 7 chars     | 6-7 chars | 5-8 chars |
| **Custom Aliases** | Yes                | Yes (Pro)   | No        | Yes (Pro) |
| **Analytics**      | Basic (optional)   | Advanced    | Basic     | Advanced  |
| **Expiry**         | Yes                | Yes         | No        | Yes       |
| **Caching**        | Redis              | Redis + CDN | Unknown   | CDN       |
| **Database**       | Postgres (sharded) | Cassandra   | Unknown   | MongoDB   |
| **Scale**          | 1B links           | 25B links   | Unknown   | 1B+ links |

---

## 13. Trade-offs Summary

| Decision             | Choice                      | Alternative          | Why Chosen                            | Trade-off                          |
|----------------------|-----------------------------|----------------------|---------------------------------------|------------------------------------|
| **Alias Generation** | Base62 (sequential ID)      | MD5 Hash             | Guaranteed unique, no collisions      | Requires ID generator service      |
| **ID Generator**     | Snowflake (distributed)     | DB Auto-increment    | Horizontal scalability, no SPOF       | More complex infrastructure        |
| **Database**         | PostgreSQL (sharded)        | Cassandra/DynamoDB   | ACID, uniqueness constraints          | Write scaling harder               |
| **Caching**          | Cache-Aside with TTL        | Write-Through        | Optimize for read-heavy workload      | Eventual consistency               |
| **Redirect Type**    | HTTP 302 (Temporary)        | HTTP 301 (Permanent) | Better analytics, track every click   | Higher server load                 |
| **Analytics**        | Async (Kafka + separate DB) | Inline (same table)  | Don't slow down redirects             | Eventual consistency for stats     |
| **Sharding**         | By short_alias hash         | By user_id           | Lookups use short_alias (primary key) | User-based queries require scatter |

---

## Summary

A URL shortener system requires careful balance between:

**Key Design Choices:**

1. ✅ **Base62 Encoding** of sequential IDs for guaranteed uniqueness
2. ✅ **Distributed ID Generator** (Snowflake) for horizontal scalability
3. ✅ **Cache-Aside Pattern** with Redis for read-heavy workload (100:1)
4. ✅ **Sharded PostgreSQL** for ACID guarantees and uniqueness constraints
5. ✅ **Asynchronous Analytics** to avoid blocking redirects
6. ✅ **HTTP 302 Redirects** for better click tracking

**Performance Characteristics:**

- **Write Latency:** ~50-100ms (ID generation + DB write + cache write)
- **Read Latency (Cache Hit):** < 5ms (90% of requests)
- **Read Latency (Cache Miss):** ~50ms (10% of requests)
- **Throughput:** 100K+ redirects/sec with horizontal scaling

**Critical Components:**

- **Redis Cache:** 85-90% hit rate, reduces DB load by 9x
- **ID Generator:** Must be highly available (single point of failure for writes)
- **Database:** Sharding required for >1B URLs
- **Rate Limiting:** Essential to prevent abuse

**Scalability Path:**

1. **0-1M URLs:** Single DB + Redis, simple setup
2. **1M-100M URLs:** DB replication, Redis cluster, CDN for static content
3. **100M-1B URLs:** Database sharding by alias hash
4. **1B+ URLs:** Multi-region deployment, distributed ID generation

**Common Pitfalls to Avoid:**

1. ❌ Race conditions on custom aliases
2. ❌ Synchronous analytics blocking redirects
3. ❌ Cache stampede on popular URLs
4. ❌ No rate limiting (DDoS vulnerability)
5. ❌ Not validating URLs (XSS/SSRF risk)
6. ❌ Storing analytics in same table as URLs
7. ❌ No cache fallback strategy
8. ❌ Sequential IDs without encoding (enumeration risk)

**Recommended Stack:**

- **Load Balancer:** NGINX or AWS ALB
- **API Servers:** Go (performance) or Python/FastAPI (development speed)
- **Cache:** Redis Cluster (active-active, multi-AZ)
- **Database:** PostgreSQL with Citus or manual sharding
- **ID Generator:** Snowflake algorithm (embedded or separate service)
- **Analytics:** Kafka + ClickHouse or Cassandra
- **Monitoring:** Prometheus + Grafana + Distributed Tracing (Jaeger)

**Cost Efficiency:**

- Optimize with CDN (reduce data transfer by 80%)
- Use reserved instances (save 40% on EC2/RDS)
- Separate read replicas for analytics queries
- **Estimated cost:** ~$1,500-$3,000/month for 1M creations + 100M redirects/day

---

## 14. Advanced Features

### 14.1 QR Code Generation

**Automatic QR code generation for each short URL:**

```
Implementation:
- Generate QR code on URL creation (async)
- Store as PNG in S3
- CDN cache for fast delivery
- SVG option for scalable logos

API Endpoint:
GET /api/v1/qr/{shortCode}?size=300&format=png

Benefits:
- Mobile-first sharing
- Print materials (posters, business cards)
- No-type URL entry
```

### 14.2 Link Preview & Open Graph

**Rich link previews for social media:**

```
Features:
- Fetch Open Graph metadata from target URL
- Cache metadata (24-hour TTL)
- Generate preview card
- Custom OG tags for short URL page

Implementation:
- Async job on URL creation
- Parse og:title, og:description, og:image
- Store in metadata table
- Serve via og:tags on /{shortCode} page
```

### 14.3 Link Bundling & Campaigns

**Group related links for marketing campaigns:**

```
Use Cases:
- Social media campaigns (multiple product links)
- Event promotions (schedule, tickets, venue)
- Bio links (Instagram, TikTok)

Features:
- Create link bundle: /b/{bundleId}
- Landing page with all links
- Aggregate analytics across bundle
- Custom branding (logo, colors, CTAs)

Schema:
bundles: {bundle_id, user_id, title, theme, links[]}
bundle_links: {bundle_id, short_code, order, description}
```

### 14.4 Geographic Redirection

**Route users to different destinations based on location:**

```
Use Cases:
- E-commerce: Different stores for US/EU/Asia
- App downloads: App Store vs Google Play
- Localized content: Language-specific pages

Implementation:
- Detect country from IP (GeoIP2 database)
- Store geo_rules: {short_code, country_code, target_url}
- Priority: User country → Default
- Fallback to primary URL if no match

Performance:
- GeoIP lookup: < 1ms (in-memory database)
- Rule evaluation: < 1ms
- Total overhead: < 5ms
```

### 14.5 Link Retargeting & Pixels

**Marketing pixel injection:**

```
Features:
- Add Facebook Pixel, Google Analytics, etc.
- Inject into redirect page (302 with meta refresh)
- User sees redirect page for 100ms
- Pixels fire before redirect

Privacy Considerations:
- Comply with GDPR/CCPA
- Cookie consent banners
- User opt-out mechanism
```

---

## 15. Performance Optimization

### 15.1 Connection Pooling

**Optimize database connections:**

```
PostgreSQL Connection Pool:
- Pool size: 20 connections per API server
- Max connections: 500 (RDS limit)
- 20 servers × 20 = 400 connections (safe margin)

HikariCP Configuration (Java):
- maximumPoolSize: 20
- minimumIdle: 5
- connectionTimeout: 30000ms
- idleTimeout: 600000ms (10 minutes)
- maxLifetime: 1800000ms (30 minutes)

Benefits:
- Avoid connection overhead (10-50ms per connect)
- Reuse authenticated connections
- Handle traffic spikes gracefully
```

### 15.2 Read-Through Cache Pattern

**Optimize cache miss handling:**

```
Traditional Approach (2 queries on miss):
1. Check cache
2. If miss, query database
3. Write back to cache
Total: 1-2ms (cache) + 50ms (DB) = 51-52ms

Read-Through Cache (1 operation):
1. Check cache with automatic DB fallback
2. Cache library handles miss internally
3. Returns result + updates cache atomically

Implementation:
- Use Redis Cache-Aside with TTL
- Probability-based refresh (90% TTL)
- Prevents cache stampede

Benefits:
- Simpler application code
- Atomic cache update
- Reduced race conditions
```

### 15.3 Database Query Optimization

**Critical query optimizations:**

```
Indexes:
- UNIQUE INDEX on short_code (fast lookups)
- INDEX on user_id, created_at (user dashboards)
- PARTIAL INDEX on created_at WHERE active=true (active URLs)

Query Patterns:
✅ SELECT target_url, analytics_enabled FROM urls WHERE short_code = ?
   (Uses primary key, < 1ms)

❌ SELECT * FROM urls WHERE target_url LIKE '%example%'
   (Full table scan, > 1000ms on 1B rows)

✅ SELECT short_code FROM urls WHERE user_id = ? ORDER BY created_at DESC LIMIT 20
   (Uses composite index, < 10ms)

Connection Settings:
- statement_timeout: 5000ms (kill slow queries)
- idle_in_transaction_session_timeout: 10000ms
- work_mem: 16MB (sort optimization)
```

### 15.4 Compression & CDN

**Reduce bandwidth costs:**

```
Response Compression:
- Gzip for JSON responses
- Brotli for static assets
- Compression ratio: 70-80%
- Cost savings: $400/month on 1TB → $80/month on 200GB

CDN Strategy:
- CloudFront for redirect pages
- Cache static assets (QR codes, logos)
- Edge locations: 200+ globally
- Latency improvement: 200ms → 20ms (international)

Cache Headers:
Cache-Control: public, max-age=31536000, immutable (QR codes)
Cache-Control: public, max-age=300 (redirect pages)
```

### 15.5 Database Sharding Strategy

**Scale beyond 1 billion URLs:**

```
Shard Key: hash(short_code) % num_shards

Shard Distribution:
- 16 shards initially
- Each shard: 60M URLs (manageable)
- Add shards as needed (requires rehashing)

Shard Routing:
def get_shard(short_code):
    shard_id = hash(short_code) % NUM_SHARDS
    return shard_connections[shard_id]

Benefits:
- Horizontal scalability
- Isolated failures (1 shard down = 6% of URLs affected)
- Parallel queries

Challenges:
- Cross-shard analytics (use data warehouse)
- Shard rebalancing (pre-allocate shards)
```

---

## 16. References

### Related System Design Components

- **[2.1.7 PostgreSQL Deep Dive](../../02-components/2.1-databases/2.1.7-postgresql-deep-dive.md)** - Database design patterns
- **[2.2.1 Redis Deep Dive](../../02-components/2.2-caching/2.2.1-redis-deep-dive.md)** - Caching strategies
- **[2.5.1 Rate Limiting Algorithms](../../02-components/2.5-algorithms/2.5.1-rate-limiting-algorithms.md)** - Token bucket, sliding window
- **[1.1.2 Horizontal vs Vertical Scaling](../../01-principles/1.1.2-horizontal-vs-vertical-scaling.md)** - Scaling strategies
- **[1.2.2 CAP Theorem](../../01-principles/1.2.2-cap-theorem.md)** - Consistency trade-offs

### Related Design Challenges

- **[3.1.2 Distributed Cache](../3.1.2-distributed-cache/)** - Cache architecture
- **[3.1.3 Distributed ID Generator](../3.1.3-distributed-id-generator/)** - ID generation patterns
- **[3.4.2 News Feed](../3.4.2-news-feed/)** - Read-heavy system patterns
- **[3.2.4 Global Rate Limiter](../3.2.4-global-rate-limiter/)** - Rate limiting at scale

### External Resources

- **URL Shortening at Scale:** [bit.ly/bitly-architecture](https://bit.ly/bitly-architecture)
- **Base62 Encoding Explained:** [Wikipedia](https://en.wikipedia.org/wiki/Base62)
- **Redis Best Practices:** [Redis Documentation](https://redis.io/topics/memory-optimization)
- **PostgreSQL Performance Tuning:** [PostgreSQL Wiki](https://wiki.postgresql.org/wiki/Performance_Optimization)
- **Snowflake ID Generator:** [Twitter's Snowflake](https://github.com/twitter-archive/snowflake)

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Chapters on caching and partitioning
- *System Design Interview Vol 1* by Alex Xu - URL shortener design patterns
- *Database Internals* by Alex Petrov - B-tree indexing and query optimization

---

This design provides a **production-ready, scalable blueprint** for building a URL shortener that can handle billions of
URLs and millions of redirects per second! 🚀
