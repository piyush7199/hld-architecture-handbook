# URL Shortener - High-Level Design

## Table of Contents

1. [System Architecture Diagram](#system-architecture-diagram)
2. [Component Architecture](#component-architecture)
3. [Data Flow Diagrams](#data-flow-diagrams)
4. [Sharding Strategy](#sharding-strategy)
5. [Scaling Path](#scaling-path)

---

## System Architecture Diagram

**Flow Explanation:**
This diagram shows the complete system architecture for a globally distributed URL shortener. 

**Key Flows:**
1. **Write Path (Create Short URL):** Users → Load Balancer → Shortening Service → ID Generator (get unique ID) → Base62 encode → Save to Database → Cache result → Return short URL
2. **Read Path (Redirect):** Users → Load Balancer → Redirection Service → Check Redis Cache (90% hit rate) → If miss: Query Database → Return HTTP 301/302 redirect → Async track analytics
3. **Cache Layer:** Redis cluster stores short→long URL mappings with 24-hour TTL, dramatically reducing database load
4. **Database:** PostgreSQL sharded by alias hash (Shard 0: a-m, Shard 1: n-z) with primary-replica setup
5. **Analytics:** Async pipeline via Kafka to ClickHouse/Cassandra for click tracking without blocking redirects

**Traffic Distribution:** Read-heavy (1,157 QPS reads vs 12 QPS writes = 100:1 ratio)

```mermaid
graph TB
    subgraph "Users/Clients"
        Users[Users/Clients]
    end
    
    subgraph "Load Balancers"
        LB1[Load Balancer<br/>US-East-1]
        LB2[Load Balancer<br/>EU-West-1]
        LB3[Load Balancer<br/>AP-South-1]
    end
    
    Users --> LB1
    Users --> LB2
    Users --> LB3
    
    subgraph "Write Path - 12 QPS"
        ShorteningService[Shortening Service<br/>Stateless<br/>1. Get unique ID<br/>2. Base62 encode<br/>3. Save to DB<br/>4. Cache result]
    end
    
    subgraph "Read Path - 1,157 QPS"
        RedirectionService[Redirection Service<br/>Stateless<br/>1. Check cache<br/>2. Return 301/302<br/>3. Track analytics]
    end
    
    LB1 --> ShorteningService
    LB2 --> ShorteningService
    LB3 --> ShorteningService
    
    LB1 --> RedirectionService
    LB2 --> RedirectionService
    LB3 --> RedirectionService
    
    subgraph "Cache Layer"
        Redis[Redis Cache Cluster<br/>Cache-Aside Pattern<br/>short_url → long_url<br/>TTL: 24 hours<br/>Hit Rate: 90%+]
    end
    
    RedirectionService --> Redis
    ShorteningService --> Redis
    
    subgraph "Database Layer"
        DB[(PostgreSQL<br/>Sharded by alias hash<br/>Primary-Replica<br/>Async Replication)]
        Shard0[(Shard 0<br/>a-m)]
        Shard1[(Shard 1<br/>n-z)]
    end
    
    ShorteningService --> DB
    Redis --> DB
    DB --> Shard0
    DB --> Shard1
    
    subgraph "ID Generation"
        IDGen[ID Generator Service<br/>Snowflake<br/>Generates unique sequential IDs]
    end
    
    ShorteningService --> IDGen
    
    subgraph "Analytics Pipeline"
        Analytics[Analytics Service<br/>Kafka → ClickHouse/Cassandra]
    end
    
    RedirectionService -.->|Async| Analytics
    
    style ShorteningService fill:#e1f5ff
    style RedirectionService fill:#fff4e1
    style Redis fill:#ffe1e1
    style DB fill:#e1ffe1
    style IDGen fill:#f5e1ff
    style Analytics fill:#fffce1
```

## Component Architecture

**Flow Explanation:**
Shows the layered architecture and component interactions.

**Layers:**
1. **API Layer:** Stateless API servers behind load balancer, horizontally scalable
2. **Service Layer:** Shortening Service (write path) and Redirection Service (read path) as separate microservices
3. **Data Layer:** Redis Cache (hot data), PostgreSQL Shards (persistent storage), ID Generator (unique IDs)
4. **Analytics Layer:** Async pipeline via Kafka to ClickHouse for non-blocking click tracking

**Key Design:** Separation of concerns - each layer can scale independently based on traffic patterns.

```mermaid
graph LR
    subgraph "API Layer"
        API1[API Server 1]
        API2[API Server 2]
        API3[API Server N]
    end
    
    subgraph "Service Layer"
        SS[Shortening Service]
        RS[Redirection Service]
    end
    
    subgraph "Data Layer"
        Cache[Redis Cluster]
        DB[PostgreSQL Shards]
        IDG[ID Generator]
    end
    
    subgraph "Analytics Layer"
        Kafka[Kafka Queue]
        CH[ClickHouse]
    end
    
    API1 --> SS
    API2 --> RS
    API3 --> SS
    
    SS --> Cache
    SS --> DB
    SS --> IDG
    
    RS --> Cache
    RS --> DB
    RS -.->|Async| Kafka
    
    Kafka --> CH
    
    style SS fill:#e1f5ff
    style RS fill:#fff4e1
    style Cache fill:#ffe1e1
    style DB fill:#e1ffe1
```

## Data Flow Diagrams

### Write Path (URL Creation)

**Flow Explanation:**
Detailed flow for creating a shortened URL with two paths: custom alias vs auto-generated.

**Custom Alias Path:** Client provides desired alias → Check availability in cache → Atomic INSERT to database → If conflict: Return error → If success: Cache and return
**Auto-Generated Path:** Get unique ID from ID Generator → Base62 encode → Save to database → Cache → Return short URL

**Key Decision Point:** Custom alias check happens BEFORE database to fail fast, but database UNIQUE constraint is the source of truth to prevent race conditions.

```mermaid
flowchart TD
    Start([Client creates short URL]) --> Input[Receive long URL]
    Input --> CustomCheck{Custom alias<br/>provided?}
    
    CustomCheck -->|Yes| CheckAvail[Check alias availability<br/>in cache]
    CheckAvail --> IsAvail{Available?}
    IsAvail -->|No| Error1[Return: Alias taken]
    IsAvail -->|Yes| AtomicInsert[Atomic INSERT<br/>to database]
    
    CustomCheck -->|No| GetID[Get unique ID from<br/>ID Generator]
    GetID --> Encode[Base62 encode ID<br/>to short alias]
    Encode --> SaveDB[Save to database]
    
    AtomicInsert --> CheckConflict{Insert<br/>successful?}
    CheckConflict -->|No| Error2[Return: Conflict]
    CheckConflict -->|Yes| CacheWrite[Write to cache<br/>TTL: 24h]
    
    SaveDB --> CacheWrite
    CacheWrite --> Return[Return short URL]
    Return --> End([End])
    Error1 --> End
    Error2 --> End
    
    style GetID fill:#f5e1ff
    style Encode fill:#e1f5ff
    style CacheWrite fill:#ffe1e1
    style SaveDB fill:#e1ffe1
```

### Read Path (URL Redirection)

**Flow Explanation:**
Optimized flow for fast redirects with cache-first strategy.

**Fast Path (90% of requests):** Check Redis cache → Hit → Async track click → Return HTTP 302 redirect (~5ms total)
**Slow Path (10% cache miss):** Query database → Check if URL exists and not expired → Update cache → Async track click → Return HTTP 302 redirect (~50ms total)

**Optimization:** Analytics tracking is fully asynchronous to not block redirect response. Cache TTL of 24 hours balances freshness and performance.

```mermaid
flowchart TD
    Start([Client visits short URL]) --> CheckCache[Check Redis cache]
    CheckCache --> CacheHit{Cache<br/>hit?}
    
    CacheHit -->|Yes| TrackAsync1[Async: Track click]
    TrackAsync1 --> Redirect1[HTTP 302 Redirect<br/>< 5ms]
    Redirect1 --> End1([End])
    
    CacheHit -->|No| QueryDB[Query database<br/>for long URL]
    QueryDB --> Found{URL<br/>found?}
    
    Found -->|No| Error404[Return 404<br/>URL not found]
    Error404 --> End2([End])
    
    Found -->|Yes| CheckExpiry{URL<br/>expired?}
    CheckExpiry -->|Yes| Error410[Return 410<br/>URL expired]
    Error410 --> End2
    
    CheckExpiry -->|No| UpdateCache[Update cache<br/>TTL: 24h]
    UpdateCache --> TrackAsync2[Async: Track click]
    TrackAsync2 --> Redirect2[HTTP 302 Redirect<br/>~50ms]
    Redirect2 --> End3([End])
    
    style CheckCache fill:#ffe1e1
    style QueryDB fill:#e1ffe1
    style UpdateCache fill:#ffe1e1
    style TrackAsync1 fill:#fffce1
    style TrackAsync2 fill:#fffce1
```

## Sharding Strategy

**Flow Explanation:**
Database sharding strategy to distribute load and enable horizontal scaling.

**Sharding Logic:** Hash-based routing using `hash(short_alias) mod N` distributes URLs evenly across 4 shards (example: a-f, g-m, n-s, t-z)
**Per-Shard Setup:** Each shard has 1 Primary (read+write) and 2 Replicas (read-only) with async replication
**Benefits:** 
- Even distribution of load
- Independent scaling per shard
- Read scaling via replicas
- Fault tolerance (replica promotion on primary failure)

```mermaid
graph TB
    subgraph "Shard Distribution"
        Router[Hash Router<br/>hash short_alias mod N]
        
        Router --> Shard1[Shard 1<br/>short_alias: a-f]
        Router --> Shard2[Shard 2<br/>short_alias: g-m]
        Router --> Shard3[Shard 3<br/>short_alias: n-s]
        Router --> Shard4[Shard 4<br/>short_alias: t-z]
    end
    
    subgraph "Each Shard"
        Primary[Primary<br/>Read + Write]
        Replica1[Replica 1<br/>Read only]
        Replica2[Replica 2<br/>Read only]
        
        Primary -->|Async Replication| Replica1
        Primary -->|Async Replication| Replica2
    end
    
    style Router fill:#e1f5ff
    style Primary fill:#e1ffe1
    style Replica1 fill:#f0fff0
    style Replica2 fill:#f0fff0
```

## Scaling Path

**Flow Explanation:**
Progressive scaling strategy as system grows from thousands to billions of URLs.

**Phase 1 (0-1M URLs):** Single database + Redis → Simple monolithic setup → Good for MVP
**Phase 2 (1M-100M URLs):** Add DB replication + Redis cluster + CDN → Improved read performance
**Phase 3 (100M-1B URLs):** Database sharding + Multi-region cache → Horizontal scaling unlocked
**Phase 4 (1B+ URLs):** Multi-region deployment + Distributed ID Generator → Global scale with low latency

**Key Insight:** Start simple, scale incrementally based on actual traffic patterns and bottlenecks.

```mermaid
graph LR
    subgraph "Phase 1: 0-1M URLs"
        P1[Single DB + Redis<br/>Simple Setup]
    end
    
    subgraph "Phase 2: 1M-100M URLs"
        P2[DB Replication<br/>Redis Cluster<br/>CDN]
    end
    
    subgraph "Phase 3: 100M-1B URLs"
        P3[Database Sharding<br/>Multi-region Cache]
    end
    
    subgraph "Phase 4: 1B+ URLs"
        P4[Multi-region Deployment<br/>Distributed ID Gen]
    end
    
    P1 -->|Scale| P2
    P2 -->|Scale| P3
    P3 -->|Scale| P4
    
    style P1 fill:#e1ffe1
    style P2 fill:#e1f5ff
    style P3 fill:#ffe1e1
    style P4 fill:#f5e1ff
```

