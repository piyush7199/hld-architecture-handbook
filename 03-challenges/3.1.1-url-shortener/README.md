# 3.1.1 Design URL Shortener (TinyURL/Bitly)

## Problem Statement

Design a highly available, scalable URL shortening service like TinyURL or Bitly that can convert long URLs into short,
memorable links. The system must handle billions of URLs, support extremely high read throughput (100:1 read-to-write
ratio),
provide sub-100ms redirect latency, and support custom aliases, expiration, and basic analytics.

---

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow, and scaling strategy
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows for URL creation, redirection, analytics, and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices and trade-offs
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for all core functions

---

## 1. Requirements and Scale Estimation

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

## 2. High-Level Architecture

> 📊 **See detailed architecture diagrams:** [HLD Diagrams](./hld-diagram.md)

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

## 3. Detailed Component Design

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

```
Base62Encoder:
  ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
  BASE = 62
  
  function encode(num):
    if num == 0:
      return ALPHABET[0]
    
    result = empty_array
    while num > 0:
      result.append(ALPHABET[num % BASE])
      num = num / BASE  // integer division
    
    return reverse(result)
  
  function decode(encoded):
    num = 0
    for each char in encoded:
      num = num * BASE + index_of(char in ALPHABET)
    return num
```

**Example:**
```
ID 123456789 → encode → "8M0kX"

Capacity:
- 7-char Base62: 62^7 = 3,521,614,606,208 URLs
- 6-char Base62: 62^6 = 56,800,235,584 URLs
```

#### Write Path Flow (Detailed)

> 📊 **See visual flow:** [Write Path Sequence Diagram](./sequence-diagrams.md#write-path-url-shortening-creation)

**URL Shortening Service Logic:**

**Two Paths:**

1. **Custom Alias Path:**
   - User provides desired alias (e.g., "mylink")
   - Check availability in cache first (fast check)
   - Attempt atomic insert in database with alias as primary key
   - If UniqueConstraintViolation: Return "Alias already taken" error
   - If success: Cache the mapping

2. **Auto-Generated Alias Path:**
   - Request unique ID from ID Generator service
   - Encode ID using Base62 algorithm (converts number to short string)
   - Example: ID 123456789 → "8M0kX"
   - Store mapping in database
   - Cache the mapping

**Both paths end with:**
- Store mapping in cache with 24-hour TTL (Cache-Aside pattern)
- Return shortened URL to user

*See `pseudocode.md::shorten_url()` for detailed implementation*

### 3.3 Redirection Strategy (The Read Path)

> 📊 **See visual flow:** [Read Path Sequence Diagram](./sequence-diagrams.md#read-path-url-redirection)

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

**Redirect Flow (Fast Path):**

1. **Check Cache First (Redis):**
   - Try to get long_url from cache using key `"url:" + short_alias`
   - **Cache Hit (85-90%):** Return long_url immediately (~1ms)

2. **Cache Miss (10-15%):**
   - Query database for URL record
   - Validate: Check if URL exists, not expired, status is ACTIVE
   - If invalid: Return 404 error
   - If valid: Store in cache with 24-hour TTL (Cache-Aside pattern)

3. **Track Analytics (Asynchronous, Non-Blocking):**
   - Publish click event to message queue (Kafka/Redis Stream)
   - Increment real-time counter in cache
   - Does not block redirect response

4. **Redirect to Long URL:**
   - Return HTTP 302 redirect to long_url
   - Browser follows redirect automatically

*See `pseudocode.md::handle_redirect()` and `pseudocode.md::async_track_click()` for detailed implementation*

#### Cache-Aside Pattern Benefits

| Aspect                 | Benefit                                            | Metric                 |
|------------------------|----------------------------------------------------|------------------------|
| **Cache Hit Latency**  | Redis GET: ~1ms                                    | < 5ms total            |
| **Cache Miss Latency** | Redis + DB: ~50ms (acceptable for 10% of requests) | < 100ms                |
| **Cache Hit Rate**     | 80-90% for popular URLs                            | Target: 85%+           |
| **Database Load**      | Reduced by 85-90%                                  | 10-15% of reads hit DB |

---

## 4. Bottlenecks and Future Scaling

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

## 5. Common Anti-Patterns

> 📚 **Note:** For detailed pseudocode implementations of anti-patterns and their solutions, see **[3.1.1-design-url-shortener.md](./3.1.1-design-url-shortener.md#5-common-anti-patterns)** and **[pseudocode.md](./pseudocode.md)**.

Below are high-level descriptions of common mistakes and their solutions:

### Anti-Pattern 1: Not Handling Race Conditions on Custom Aliases

**Problem:**

❌ **Race condition:** Two users request same custom alias simultaneously. Both check availability, both see it's free, both try to insert → conflict!

**Solution:** ✅ Use database UNIQUE constraint on `short_alias` column. Let database enforce uniqueness atomically. Catch `UniqueConstraintViolation` exception and return "Alias already taken" error.

*See `pseudocode.md::create_custom_alias()` for implementation*

---

### Anti-Pattern 2: Synchronous Analytics Blocking Redirects

**Problem:** ❌ Redirect endpoint waits for analytics tracking to complete before responding. User experiences 50ms+ extra latency just to track a click!

**Solution:** ✅ Use asynchronous fire-and-forget tracking. Publish click event to message queue (Kafka/Redis Stream) without waiting. Return redirect immediately. Background workers process analytics asynchronously.

*See `pseudocode.md::redirect()` and `pseudocode.md::track_click_async()` for implementation*

---

### Anti-Pattern 3: Cache Stampede on Popular URLs

> 📊 **See solution diagram:** [Cache Stampede Prevention](./sequence-diagrams.md#cache-stampede-prevention)

**Problem:** ❌ When popular URL cache expires, 1,000 concurrent requests all experience cache miss simultaneously. All 1,000 hit database → overwhelms it → cascading failure.

**Solution 1:** ✅ **Distributed Locking** - Only first request queries database while acquiring lock. Other 999 requests wait, then read from cache.

**Solution 2:** ✅ **Probabilistic Early Expiration** - For hot keys, refresh cache proactively before TTL expires. If TTL < 5 minutes remaining, 10% chance to async refresh in background. Cache never goes cold.

*See `pseudocode.md::get_url_with_lock()` and `pseudocode.md::get_url_with_probabilistic_refresh()` for implementations*

---

### Anti-Pattern 4: No Rate Limiting on URL Creation

**Problem:** ❌ Without rate limiting, attackers can create millions of URLs → exhaust database storage, waste ID space, enable spam/phishing.

**Solution:** ✅ Multi-layered rate limiting:
- **By IP:** 10 URLs/hour for anonymous users
- **By User ID:** 100 URLs/hour for authenticated users  
- **By API Key:** Custom limits for enterprise
- Use Redis INCR with TTL for counters

*See `pseudocode.md::check_rate_limit()` for implementation*

---

### Anti-Pattern 5: Not Validating Input URLs

**Problem:** ❌ Accepting any string as URL allows XSS (`javascript:alert()`), SSRF (`http://localhost:6379`), and phishing attacks.

**Solution:** ✅ Multi-layer validation:
- **Scheme:** Only allow `http://` and `https://`
- **Domain:** Block localhost, 127.0.0.1, private networks (192.168.*, 10.*, 172.16-31.*)
- **Length:** Max 2048 characters
- **Reachability:** Optional HTTP HEAD request with timeout
- **Blacklist:** Check against malware/phishing databases

*See `pseudocode.md::validate_url()` for implementation*

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

**Problem:** ❌ If Redis is down, entire service fails. Cache operations throw exceptions → service unavailable.

**Solution:** ✅ Graceful degradation with try-catch:
- Wrap all cache operations in try-catch
- If cache fails: Log error, fall back to database
- Continue serving requests (slower but available)
- Don't fail cache.set() - continue without caching

*See `pseudocode.md::redirect()` for implementation with fallback*

---

### Anti-Pattern 8: Using Sequential IDs Directly as Aliases

**Problem:** ❌ Sequential IDs (`1`, `2`, `3`) are predictable → easy to enumerate → reveals usage statistics ("You have 1,234,567 URLs").

**Solution:** ✅ Encode IDs using Base62:
- Generate sequential ID: `7234891234`
- Encode with Base62: `aB3xY9`
- Aliases appear random: `aB3xY9`, `kL9mP2`, `wX8qN5`
- Harder to enumerate, doesn't reveal total count

*See `pseudocode.md::base62_encode()` for implementation*

---

## 6. Alternative Approaches (Not Chosen)

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

## 7. Deep Dive: Handling Edge Cases

### Edge Case 1: Expired Links

**Problem:** Links with expiry need automatic cleanup

**Solution:**

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

> 📊 **See detailed flow:** [Analytics Flow Diagram](./sequence-diagrams.md#analytics-flow)

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

**Key Decision:**

- **Asynchronous tracking** to avoid slowing down redirects
- Use **Redis counters** for real-time counts
- Use **batch inserts** to analytics DB for historical data

---

## 8. Monitoring and Observability

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

## 9. Interview Discussion Points

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

> 📊 **See detailed scenario:** [Redis Failover Diagram](./sequence-diagrams.md#redis-failover)

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

## 10. Comparison with Real-World Systems

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

## 11. Cost Analysis (AWS Example)

**Assumptions:**

- 1M URL creations/day = 12 QPS write
- 100M redirects/day = 1,157 QPS read
- 1.8 TB storage (5 years)

| Component                     | Specification           | Monthly Cost      |
|-------------------------------|-------------------------|-------------------|
| **EC2 (API Servers)**         | 10× t3.medium           | $400              |
| **RDS PostgreSQL**            | db.r5.2xlarge           | $800              |
| **ElastiCache Redis**         | cache.r5.xlarge cluster | $300              |
| **Application Load Balancer** | Standard                | $20               |
| **Data Transfer**             | 10 TB/month egress      | $900              |
| **CloudWatch Monitoring**     | Logs + Metrics          | $50               |
| **Route 53 (DNS)**            | 1B queries/month        | $400              |
| **S3 (Backups)**              | 2 TB                    | $50               |
| **Total**                     |                         | **~$2,920/month** |

**Cost Optimization:**

- Use CDN (CloudFront) to cache redirects → Reduce data transfer by 80%
- Reserved instances → Save 40% on EC2/RDS
- Spot instances for non-critical services
- **Optimized Cost: ~$1,500/month**

---

## 12. Trade-offs Summary

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

This design provides a **production-ready, scalable blueprint** for building a URL shortener that can handle billions of
URLs and millions of redirects per second! 🚀

