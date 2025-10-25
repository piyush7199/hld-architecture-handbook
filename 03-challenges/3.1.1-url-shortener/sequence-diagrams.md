# URL Shortener - Sequence Diagrams

## Write Path: URL Shortening (Creation)

### Standard Flow (Auto-generated Alias)

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

