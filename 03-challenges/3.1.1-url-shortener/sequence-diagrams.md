# URL Shortener - Sequence Diagrams

## Write Path: URL Shortening (Creation)

### Standard Flow (Auto-generated Alias)

**Flow Explanation:**
This shows the standard URL shortening flow when the system auto-generates the short alias.

**Steps:**
1. Client sends POST request with long URL
2. Service requests unique ID from ID Generator (e.g., 123456)
3. Service encodes ID using Base62 algorithm (123456 → "8M0kX")
4. Service saves mapping to database (short_alias as primary key)
5. Service caches mapping in Redis with 24-hour TTL
6. Service returns short URL to client

**Latency:** ~30-50ms total (ID generation: 1ms, DB write: 20ms, Cache write: 5ms)

```mermaid
sequenceDiagram
    participant Client
    participant ShorteningService as Shortening Service
    participant IDGen as ID Generator
    participant DB as Database
    participant Cache as Redis Cache

    Client->>ShorteningService: POST /shorten<br/>long_url
    activate ShorteningService
    
    ShorteningService->>IDGen: Get Next ID
    activate IDGen
    IDGen-->>ShorteningService: Returns 123456
    deactivate IDGen
    
    Note over ShorteningService: Encode to Base62<br/>123456 → 8M0kX
    
    ShorteningService->>DB: INSERT INTO urls<br/>(8M0kX, long_url)
    activate DB
    DB-->>ShorteningService: OK
    deactivate DB
    
    ShorteningService->>Cache: SET url:8M0kX<br/>(long_url, TTL 24h)
    activate Cache
    Cache-->>ShorteningService: OK
    deactivate Cache
    
    ShorteningService-->>Client: short_url: 8M0kX
    deactivate ShorteningService
```

### Custom Alias Flow

**Flow Explanation:**
Handles user-provided custom aliases with race condition protection.

**Steps:**
1. Client provides desired custom alias (e.g., "mycustom")
2. Service checks Redis cache for availability (fast pre-check)
3. Service attempts atomic INSERT to database with UNIQUE constraint
4. **Success path:** Insert succeeds → Cache the mapping → Return success
5. **Conflict path:** UNIQUE constraint violation → Return "Alias already taken" error

**Race Condition Protection:** Database UNIQUE constraint is the source of truth. Cache check is just an optimization to fail fast.

```mermaid
sequenceDiagram
    participant Client
    participant ShorteningService as Shortening Service
    participant Cache as Redis Cache
    participant DB as Database

    Client->>ShorteningService: POST /shorten<br/>long_url<br/>custom_alias=mycustom
    activate ShorteningService
    
    ShorteningService->>Cache: EXISTS url:mycustom
    activate Cache
    Cache-->>ShorteningService: false (available)
    deactivate Cache
    
    ShorteningService->>DB: INSERT INTO urls<br/>(mycustom, long_url)<br/>[Atomic with constraint]
    activate DB
    
    alt Success
        DB-->>ShorteningService: OK
        
        ShorteningService->>Cache: SET url:mycustom<br/>(long_url, TTL 24h)
        activate Cache
        Cache-->>ShorteningService: OK
        deactivate Cache
        
        ShorteningService-->>Client: short_url: mycustom
    else Unique Constraint Violation
        DB-->>ShorteningService: Error: Duplicate key
        ShorteningService-->>Client: Error: Alias already taken
    end
    
    deactivate DB
    
    deactivate ShorteningService
```

## Read Path: URL Redirection

### Cache Hit (Fast Path - 90% of requests)

**Flow Explanation:**
The optimized path when URL is cached (90% of requests).

**Steps:**
1. Client requests short URL (GET /abc123)
2. Service checks Redis cache
3. **Cache HIT** - long URL found in cache
4. Service publishes click event to analytics queue (async, non-blocking)
5. Service immediately returns HTTP 302 redirect

**Performance:** <5ms total latency (cache lookup ~1ms, redirect response ~3ms). Analytics is async so doesn't block.

```mermaid
sequenceDiagram
    participant Client
    participant RedirectionService as Redirection Service
    participant Cache as Redis Cache
    participant Analytics as Analytics Queue

    Client->>RedirectionService: GET /abc123
    activate RedirectionService
    
    RedirectionService->>Cache: GET url:abc123
    activate Cache
    Cache-->>RedirectionService: long_url (HIT)
    deactivate Cache
    
    Note over RedirectionService: < 5ms total latency
    
    RedirectionService--)Analytics: Publish click event<br/>(async, non-blocking)
    
    RedirectionService-->>Client: HTTP 302 Redirect<br/>Location: long_url
    deactivate RedirectionService
```

### Cache Miss (Slow Path - 10% of requests)

**Flow Explanation:**
Slower path when URL is not cached (10% of requests).

**Steps:**
1. Client requests short URL
2. Service checks Redis cache → **MISS**
3. Service queries database for URL mapping
4. **If found:** Update cache with 24h TTL → Track analytics async → Return redirect (~50ms)
5. **If not found:** Return HTTP 404 error

**Performance:** ~50ms total latency (DB query ~30ms, cache write ~5ms, redirect ~10ms). Still acceptable for 10% of traffic.

```mermaid
sequenceDiagram
    participant Client
    participant RedirectionService as Redirection Service
    participant Cache as Redis Cache
    participant DB as Database
    participant Analytics as Analytics Queue

    Client->>RedirectionService: GET /abc123
    activate RedirectionService
    
    RedirectionService->>Cache: GET url:abc123
    activate Cache
    Cache-->>RedirectionService: null (MISS)
    deactivate Cache
    
    RedirectionService->>DB: SELECT * FROM urls<br/>WHERE short_alias='abc123'
    activate DB
    DB-->>RedirectionService: long_url, expires_at, status
    deactivate DB
    
    alt URL Found and Active
        Note over RedirectionService: Check expiration
        
        RedirectionService->>Cache: SET url:abc123<br/>(long_url, TTL 24h)
        activate Cache
        Cache-->>RedirectionService: OK
        deactivate Cache
        
        RedirectionService--)Analytics: Publish click event<br/>(async)
        
        RedirectionService-->>Client: HTTP 302 Redirect<br/>Location: long_url<br/>(~50-100ms total)
    else URL Not Found
        RedirectionService-->>Client: HTTP 404<br/>URL not found
    else URL Expired
        RedirectionService-->>Client: HTTP 410<br/>URL expired
    end
    
    deactivate RedirectionService
```

## Analytics Flow

**Flow Explanation:**
Asynchronous analytics pipeline to track clicks without blocking redirects.

**Steps:**
1. Redirection Service publishes click event to Kafka (non-blocking, <1ms)
2. Kafka stores event in durable log
3. Consumer reads events in batches
4. Consumer writes to ClickHouse (columnar analytics DB)
5. Analytics dashboard queries ClickHouse for reports

**Key Design:** Fully asynchronous to not impact redirect latency. Event loss is acceptable (monitoring/analytics, not critical path).

```mermaid
sequenceDiagram
    participant RedirectionService as Redirection Service
    participant Kafka as Kafka Queue
    participant Consumer as Analytics Consumer
    participant ClickHouse as ClickHouse DB
    participant Redis as Redis (Counters)

    RedirectionService->>Kafka: Publish ClickEvent<br/>{alias, timestamp, ip, user_agent}
    activate Kafka
    
    Note over Kafka: Non-blocking,<br/>fire-and-forget
    
    RedirectionService->>Redis: INCR clicks:abc123
    activate Redis
    Redis-->>RedirectionService: New count
    deactivate Redis
    
    Consumer->>Kafka: Consume events<br/>(batch of 100)
    deactivate Kafka
    activate Consumer
    
    Consumer->>ClickHouse: Batch INSERT<br/>click_events
    activate ClickHouse
    ClickHouse-->>Consumer: OK
    deactivate ClickHouse
    
    deactivate Consumer
```

## ID Generation Flow (Snowflake)

**Flow Explanation:**
Shows how the ID Generator creates unique, sequential 64-bit IDs using Snowflake algorithm.

**Steps:**
1. Shortening Service requests new ID
2. ID Generator checks current timestamp
3. If same millisecond: increment sequence counter
4. If sequence overflow (>4095): wait for next millisecond
5. Compose ID: [timestamp(41 bits) | worker_id(10 bits) | sequence(12 bits)]
6. Return 64-bit ID

**Capacity:** 4,096 IDs per millisecond per worker = ~4.1 million IDs/second per worker. Multiple workers for higher throughput.

```mermaid
sequenceDiagram
    participant ShorteningService as Shortening Service
    participant IDGen as ID Generator
    participant ZooKeeper as ZooKeeper/etcd

    Note over IDGen: Startup: Acquire Worker ID
    
    IDGen->>ZooKeeper: Request worker ID
    activate ZooKeeper
    ZooKeeper-->>IDGen: Assigned Worker ID: 5<br/>(with TTL lease)
    deactivate ZooKeeper
    
    loop Heartbeat (every 10s)
        IDGen->>ZooKeeper: Refresh lease
        ZooKeeper-->>IDGen: OK
    end
    
    Note over IDGen: Ready to generate IDs
    
    ShorteningService->>IDGen: Generate ID
    activate IDGen
    
    Note over IDGen: Snowflake Algorithm:<br/>timestamp | worker_id | sequence
    
    IDGen-->>ShorteningService: 7234891234567890
    deactivate IDGen
    
    Note over ShorteningService: Base62 encode<br/>7234891234567890 → aB3xY9
```

## Cache Stampede Prevention

**Flow Explanation:**
Prevents thundering herd when popular URL's cache expires and 1000+ requests hit simultaneously.

**Without Protection:** All requests → cache miss → all query DB → DB overwhelmed
**With Protection (Shown):**
1. First request acquires distributed lock
2. Other requests wait at lock
3. First request queries DB and populates cache
4. First request releases lock
5. Waiting requests now hit cache (no DB queries)

**Result:** 1,000 concurrent requests → Only 1 DB query instead of 1,000. Critical for high-traffic URLs.

```mermaid
sequenceDiagram
    participant R1 as Request 1
    participant R2 as Request 2-1000
    participant RedirectionService as Service (with lock)
    participant Cache as Redis Cache
    participant DB as Database

    Note over Cache: Hot key expires!
    
    R1->>RedirectionService: GET /popular-url
    R2->>RedirectionService: GET /popular-url<br/>(999 more requests)
    
    activate RedirectionService
    
    RedirectionService->>Cache: GET url:popular-url
    Cache-->>RedirectionService: null (MISS)
    
    Note over RedirectionService: Acquire distributed lock<br/>lock:url:popular-url
    
    RedirectionService->>Cache: Double-check cache
    Cache-->>RedirectionService: Still null
    
    RedirectionService->>DB: SELECT * FROM urls
    activate DB
    DB-->>RedirectionService: long_url
    deactivate DB
    
    RedirectionService->>Cache: SET url:popular-url
    Cache-->>RedirectionService: OK
    
    Note over RedirectionService: Release lock
    
    RedirectionService-->>R1: HTTP 302 Redirect
    deactivate RedirectionService
    
    Note over R2: Other 999 requests<br/>now get cache hit
    
    R2->>RedirectionService: GET /popular-url
    activate RedirectionService
    RedirectionService->>Cache: GET url:popular-url
    Cache-->>RedirectionService: long_url (HIT)
    RedirectionService-->>R2: HTTP 302 Redirect
    deactivate RedirectionService
```

## Multi-Region Deployment

**Flow Explanation:**
Shows global deployment with region-specific routing for low latency.

**Steps:**
1. User in Asia hits nearest region (AP-South-1)
2. Request routed to regional load balancer
3. Regional cache checked first (Redis cluster in same region)
4. If cache miss: Query regional database replica
5. If data not in regional replica: Query primary in US
6. Return result and cache regionally

**Benefits:** <50ms latency for users worldwide. Cache and read replicas in each region. Writes go to primary but reads are local.

```mermaid
sequenceDiagram
    participant US as Client (US)
    participant EU as Client (EU)
    participant DNS as GeoDNS
    participant LB_US as Load Balancer (US-East)
    participant LB_EU as Load Balancer (EU-West)
    participant Service_US as Service (US)
    participant Service_EU as Service (EU)
    participant Cache_US as Redis (US)
    participant Cache_EU as Redis (EU)
    participant DB as Database (Multi-region)

    US->>DNS: Resolve short.ly
    DNS-->>US: IP: US-East LB
    
    EU->>DNS: Resolve short.ly
    DNS-->>EU: IP: EU-West LB
    
    US->>LB_US: GET /abc123
    LB_US->>Service_US: Forward request
    activate Service_US
    
    Service_US->>Cache_US: GET url:abc123
    activate Cache_US
    Cache_US-->>Service_US: long_url (HIT)
    deactivate Cache_US
    
    Service_US-->>US: HTTP 302 Redirect
    deactivate Service_US
    
    EU->>LB_EU: GET /abc123
    LB_EU->>Service_EU: Forward request
    activate Service_EU
    
    Service_EU->>Cache_EU: GET url:abc123
    activate Cache_EU
    
    alt Cache Hit
        Cache_EU-->>Service_EU: long_url (HIT)
    else Cache Miss
        Service_EU->>DB: Query database<br/>(nearest replica)
        DB-->>Service_EU: long_url
        Service_EU->>Cache_EU: SET url:abc123
    end
    deactivate Cache_EU
    
    Service_EU-->>EU: HTTP 302 Redirect
    deactivate Service_EU
```

## Failure Scenarios

### Redis Failover

**Flow Explanation:**
Shows graceful degradation when Redis cache fails.

**Steps:**
1. Client requests redirect
2. Service tries Redis → **Connection timeout/error**
3. Service catches exception and logs error
4. Service falls back to database (slower but works)
5. Service queries database for URL
6. Service returns redirect (degraded mode: ~100ms instead of ~5ms)
7. Meanwhile, operations team is alerted
8. Sentinel promotes replica to new master
9. Service reconnects to new Redis master

**Key Design:** Service continues operating (slower) even when cache is down. No data loss, just higher latency.

```mermaid
sequenceDiagram
    participant Client
    participant Service as Redirection Service
    participant Cache_M as Redis Master
    participant Cache_R as Redis Replica
    participant Sentinel as Redis Sentinel
    participant DB as Database

    Client->>Service: GET /abc123
    activate Service
    
    Service->>Cache_M: GET url:abc123
    
    Note over Cache_M: Master FAILS!
    
    Cache_M--XService: Timeout / Connection error
    
    Service->>Sentinel: Detect failure
    activate Sentinel
    
    Note over Sentinel: Quorum vote:<br/>Master is down
    
    Sentinel->>Cache_R: Promote to Master
    activate Cache_R
    Cache_R-->>Sentinel: Promotion successful
    deactivate Sentinel
    
    Note over Service: Retry with<br/>circuit breaker
    
    Service->>DB: Fallback to database
    activate DB
    DB-->>Service: long_url
    deactivate DB
    
    Service-->>Client: HTTP 302 Redirect<br/>(Degraded performance)
    deactivate Service
    deactivate Cache_R
    
    Note over Service,Cache_R: Service discovers<br/>new master<br/>(~10-15 seconds total)
```

