# High-Level Design Diagrams: Twitter/X Timeline

This document contains Mermaid diagrams illustrating the system architecture, component design, data flow, and scaling strategies for the Twitter Timeline system.

---

## 1. Complete System Architecture

```mermaid
graph TB
    Client[Client Applications<br/>Web, iOS, Android]
    LB[Load Balancer<br/>Multi-Region]
    AG[API Gateway<br/>Auth + Rate Limiting]
    
    PS[Post Service<br/>Stateless]
    TS[Timeline Service<br/>Stateless]
    
    Kafka[Kafka Message Queue<br/>Topic: new_tweets<br/>Partitions: 64]
    FW[Fanout Workers<br/>Consumer Groups]
    
    RC[Redis Cluster<br/>Timeline Cache]
    Cassandra[Cassandra<br/>UserTimeline Storage]
    PG[PostgreSQL Sharded<br/>Tweets Source of Truth]
    FG[Follower Graph<br/>Neo4j / PostgreSQL]
    
    Client --> LB
    LB --> AG
    
    AG -->|POST /tweet| PS
    AG -->|GET /timeline| TS
    
    PS -->|Save tweet| PG
    PS -->|Publish event| Kafka
    
    TS -->|Check cache| RC
    TS -->|Query timelines| Cassandra
    
    Kafka -->|Consume| FW
    FW -->|Get followers| FG
    FW -->|Insert tweets| Cassandra
    FW -->|Update cache| RC
    
    style PS fill:#e1f5ff
    style TS fill:#fff4e1
    style Kafka fill:#ffe1e1
    style RC fill:#e1ffe1
    style Cassandra fill:#f0e1ff
    style PG fill:#ffe1f0
```

**Flow Explanation:**

This diagram shows the complete Twitter Timeline system architecture with all major components.

**Key Components:**
1. **Load Balancer** → Routes traffic to nearest data center
2. **API Gateway** → Handles authentication, rate limiting, request routing
3. **Post Service (Write Path)** → Validates tweets, saves to PostgreSQL, publishes to Kafka
4. **Timeline Service (Read Path)** → Checks Redis cache, queries Cassandra for timelines
5. **Kafka** → Message queue that decouples write path from fanout operations
6. **Fanout Workers** → Consume tweet events, fetch followers, insert into timelines
7. **Redis Cluster** → Hot timeline cache for sub-10ms reads
8. **Cassandra** → Persistent timeline storage (denormalized)
9. **PostgreSQL** → Source of truth for tweets (ACID)
10. **Follower Graph** → Social graph for follower lookups

**Benefits:**
- **Separation of Concerns:** Write and read paths are independent
- **Async Processing:** Kafka enables non-blocking fanout
- **High Availability:** Redis + Cassandra both support multi-master replication
- **Horizontal Scaling:** All components scale independently

---

## 2. Write Path (Posting a Tweet)

```mermaid
flowchart TD
    Start([User Posts Tweet]) --> Validate[Post Service<br/>Validate Content<br/>Check Rate Limit]
    Validate --> GenID[Generate Snowflake ID]
    GenID --> SaveDB[Save to PostgreSQL<br/>Tweets Table]
    SaveDB --> PubKafka[Publish to Kafka<br/>Topic: new_tweets]
    PubKafka --> ReturnClient[Return Success to Client<br/>~100ms total]
    
    PubKafka -.Async.-> FWConsume[Fanout Workers<br/>Consume from Kafka]
    FWConsume --> GetFollowers[Query Follower Graph<br/>Get all followers]
    GetFollowers --> CheckCelebrity{Author is<br/>Celebrity?}
    
    CheckCelebrity -->|No <10K| FanoutPush[Fanout-on-Write<br/>Insert into each<br/>follower's timeline]
    CheckCelebrity -->|Yes >10K| SkipFanout[Skip Fanout<br/>Will be pulled on read]
    
    FanoutPush --> UpdateCache[Update Redis Cache<br/>For active followers]
    UpdateCache --> Done([Fanout Complete<br/>~10 seconds])
    SkipFanout --> Done
    
    style Start fill:#e1f5ff
    style ReturnClient fill:#e1ffe1
    style Done fill:#ffe1e1
    style CheckCelebrity fill:#fff4e1
```

**Flow Explanation:**

Shows the complete flow of posting a tweet from client to fanout completion.

**Steps:**
1. **Validation** (5ms): Check content length, spam detection, rate limits
2. **ID Generation** (1ms): Generate time-sortable Snowflake ID
3. **Save to DB** (20ms): Insert into PostgreSQL tweets table
4. **Publish to Kafka** (5ms): Async event, non-blocking
5. **Return to Client** (~100ms total): User doesn't wait for fanout
6. **Fanout Workers** (async, ~10 seconds): Background processing
   - Consume tweet event from Kafka
   - Fetch all followers from Follower Graph
   - **Decision point:** Celebrity check (follower count threshold)
   - If normal user: Insert tweet into each follower's UserTimeline (Cassandra)
   - If celebrity: Skip fanout (will use pull model)
   - Update Redis cache for active followers

**Benefits:**
- Fast user experience (client waits <100ms)
- Kafka absorbs write spikes
- Celebrity problem handled gracefully
- Async fanout scales independently

---

## 3. Read Path (Loading Timeline)

```mermaid
flowchart TD
    Start([User Requests Timeline]) --> CheckCache{Check Redis<br/>Timeline Cache}
    
    CheckCache -->|Cache Hit| ReturnFast[Return Cached Timeline<br/>~10ms]
    CheckCache -->|Cache Miss| QueryCassandra[Query Cassandra<br/>UserTimeline Table]
    
    QueryCassandra --> FetchTweets[Fetch Recent 50 Tweets]
    FetchTweets --> PopulateCache[Populate Redis Cache<br/>TTL: 1 hour]
    PopulateCache --> ReturnSlow[Return Timeline<br/>~50ms]
    
    ReturnFast --> HybridCheck{User follows<br/>celebrities?}
    ReturnSlow --> HybridCheck
    
    HybridCheck -->|No| FinalReturn([Return to Client])
    HybridCheck -->|Yes| FetchCeleb[Fetch Celebrity Tweets<br/>Pull Model]
    
    FetchCeleb --> Merge[Merge Pre-computed<br/>+ Celebrity Tweets]
    Merge --> Sort[Sort by Timestamp<br/>Return Top 50]
    Sort --> FinalReturn
    
    style Start fill:#e1f5ff
    style CheckCache fill:#fff4e1
    style ReturnFast fill:#e1ffe1
    style FinalReturn fill:#e1ffe1
```

**Flow Explanation:**

Shows how timelines are loaded for users, including the hybrid fanout model.

**Steps:**
1. **Cache Check** (1ms): Look for `user_id` in Redis
2. **Cache Hit Path** (~10ms total):
   - Return pre-computed timeline from Redis
   - Fastest path (90% of requests)
3. **Cache Miss Path** (~50ms total):
   - Query Cassandra UserTimeline table
   - Fetch most recent 50 tweets
   - Populate Redis cache with TTL
   - Return timeline
4. **Hybrid Enhancement** (for users following celebrities):
   - Check if user follows any celebrities
   - Fetch celebrity tweets on-demand (pull model)
   - Merge pre-computed timeline + celebrity tweets
   - Sort by timestamp, return top 50

**Benefits:**
- Sub-10ms latency for cache hits (90%+ requests)
- Hybrid model provides balance for celebrity content
- Cache miss path still fast (<50ms)
- Pagination support for infinite scroll

**Performance:**
- Cache Hit: <10ms (P99)
- Cache Miss: <50ms (P99)
- Hybrid Merge: +20ms overhead

---

## 4. Fanout Strategies Comparison

```mermaid
graph TB
    subgraph "Fanout-on-Write (Push Model)"
        P1[Tweet Posted] --> P2[Fetch All Followers]
        P2 --> P3[Insert into Each<br/>Follower's Timeline]
        P3 --> P4[Pre-computed Timeline]
        P4 --> P5[Read: Fast O1 Lookup]
        
        P6[Pros: Fast Reads<br/>Sub-10ms] -.-> P5
        P7[Cons: Expensive Writes<br/>Celebrity Problem] -.-> P3
    end
    
    subgraph "Fanout-on-Read (Pull Model)"
        R1[User Requests Timeline] --> R2[Fetch All Followees]
        R2 --> R3[Get Recent Tweets<br/>from Each]
        R3 --> R4[Merge & Sort]
        R4 --> R5[Return Timeline]
        
        R6[Pros: Cheap Writes<br/>No Celebrity Problem] -.-> R3
        R7[Cons: Slow Reads<br/>Complex Merge] -.-> R4
    end
    
    subgraph "Hybrid Model (Twitter's Approach)"
        H1[Tweet Posted] --> H2{Author<br/>Celebrity?}
        H2 -->|No| H3[Push to Followers]
        H2 -->|Yes| H4[Skip Fanout]
        
        H5[Timeline Request] --> H6[Fetch Pre-computed]
        H6 --> H7[Fetch Celebrity Tweets]
        H7 --> H8[Merge & Return]
        
        H9[Pros: Balanced<br/>Handles Both Cases] -.-> H8
    end
    
    style P5 fill:#e1ffe1
    style R4 fill:#ffe1e1
    style H8 fill:#fff4e1
```

**Flow Explanation:**

Compares the three fanout strategies side-by-side.

**Strategy 1: Fanout-on-Write (Push)**
- **Flow:** Tweet posted → Fetch followers → Insert into each timeline → Fast O(1) read
- **Pros:** Fast reads (<10ms), simple read logic
- **Cons:** Expensive writes (N writes per tweet), celebrity problem
- **Best For:** Normal users (<10K followers)

**Strategy 2: Fanout-on-Read (Pull)**
- **Flow:** Timeline request → Fetch followees → Get their tweets → Merge & sort → Return
- **Pros:** Cheap writes (1 write per tweet), no celebrity problem
- **Cons:** Slow reads (>100ms), complex merge logic O(N log N)
- **Best For:** Celebrity users (>10K followers)

**Strategy 3: Hybrid (Recommended)**
- **Flow:** Celebrity check at write time → Push for normal, skip for celebrities → Read merges both
- **Pros:** Balanced approach, handles both cases efficiently
- **Cons:** Increased complexity, requires careful threshold tuning
- **Best For:** Production systems with diverse user types

**Trade-offs:**
- Write Cost vs Read Latency
- System Complexity vs Performance
- Storage Cost vs User Experience

---

## 5. Data Storage Architecture

```mermaid
graph TB
    subgraph "PostgreSQL - Source of Truth"
        PG1[Tweets Table<br/>ACID Guarantees]
        PG1 --> PG2[Shard by tweet_id<br/>Time-based Partitioning]
        PG2 --> PG3[Primary-Replica<br/>Async Replication]
    end
    
    subgraph "Cassandra - Timeline Storage"
        C1[UserTimeline Table<br/>Denormalized]
        C1 --> C2[Partition by user_id<br/>Cluster by tweet_id DESC]
        C2 --> C3[Multi-Region<br/>Replication Factor: 3]
    end
    
    subgraph "Redis - Cache Layer"
        R1[Timeline Cache<br/>user_id → tweets]
        R1 --> R2[Consistent Hashing<br/>Virtual Nodes]
        R2 --> R3[Master-Replica<br/>Redis Sentinel]
    end
    
    subgraph "Neo4j - Follower Graph"
        N1[User Nodes<br/>FOLLOWS Relationships]
        N1 --> N2[Graph Partitioning<br/>by user_id]
        N2 --> N3[Fast Traversals<br/>Sub-ms Queries]
    end
    
    PG1 -->|Source| C1
    C1 -->|Hot Data| R1
    N1 -->|Fanout Queries| C1
    
    style PG1 fill:#ffe1f0
    style C1 fill:#f0e1ff
    style R1 fill:#e1ffe1
    style N1 fill:#e1f5ff
```

**Flow Explanation:**

Shows the four-tier storage architecture and data flow between storage systems.

**Storage Layers:**

1. **PostgreSQL (Source of Truth)**:
   - **Purpose:** Persistent, consistent storage for tweets
   - **Sharding:** By `tweet_id` (time-based for range queries)
   - **Replication:** Primary-Replica async replication
   - **Guarantees:** ACID, strong consistency

2. **Cassandra (Timeline Storage)**:
   - **Purpose:** Denormalized timeline entries (high-throughput writes)
   - **Partitioning:** By `user_id` (efficient timeline lookups)
   - **Clustering:** By `tweet_id DESC` (chronological order)
   - **Replication:** Multi-region, RF=3

3. **Redis (Cache Layer)**:
   - **Purpose:** Hot timeline cache (sub-ms reads)
   - **Distribution:** Consistent hashing for load balancing
   - **Availability:** Master-Replica with Redis Sentinel
   - **TTL:** 1 hour for automatic expiration

4. **Neo4j (Follower Graph)**:
   - **Purpose:** Social graph traversals (follower lookups)
   - **Partitioning:** Graph partitioning by `user_id`
   - **Performance:** Sub-millisecond graph queries
   - **Query:** `MATCH (u)-[:FOLLOWS]->(f) RETURN f`

**Data Flow:**
- PostgreSQL → Cassandra (fanout process)
- Cassandra → Redis (cache population)
- Neo4j → Cassandra (fanout uses follower list)

**Benefits:**
- Right tool for each job (polyglot persistence)
- PostgreSQL for consistency, Cassandra for scale, Redis for speed, Neo4j for graphs
- Each layer scales independently

---

## 6. Scaling Timeline Cache (Redis)

```mermaid
graph TB
    Client[Client Requests] --> CH[Consistent Hash Function]
    
    CH --> VN1[Virtual Node 1<br/>Hash: 100]
    CH --> VN2[Virtual Node 2<br/>Hash: 500]
    CH --> VN3[Virtual Node 3<br/>Hash: 900]
    
    VN1 --> P1[Physical Node 1<br/>Redis Master<br/>Users: A-G]
    VN2 --> P2[Physical Node 2<br/>Redis Master<br/>Users: H-N]
    VN3 --> P3[Physical Node 3<br/>Redis Master<br/>Users: O-Z]
    
    P1 --> R1[Replica 1A]
    P1 --> R2[Replica 1B]
    P2 --> R3[Replica 2A]
    P2 --> R4[Replica 2B]
    P3 --> R5[Replica 3A]
    P3 --> R6[Replica 3B]
    
    S[Redis Sentinel<br/>Monitors Health] -.-> P1
    S -.-> P2
    S -.-> P3
    
    style CH fill:#fff4e1
    style P1 fill:#e1ffe1
    style P2 fill:#e1ffe1
    style P3 fill:#e1ffe1
    style S fill:#ffe1e1
```

**Flow Explanation:**

Shows how Redis cache is distributed using consistent hashing and replicated for high availability.

**Architecture:**

1. **Consistent Hashing**:
   - Hash user_id to find responsible virtual node
   - Virtual nodes map to physical Redis nodes
   - 150 virtual nodes per physical node for even distribution

2. **Physical Nodes**:
   - 3 master nodes (example: can scale to 100+)
   - Each handles a portion of the key space
   - Hash range divided: Node1(0-333), Node2(334-666), Node3(667-999)

3. **Replication**:
   - Each master has 2 replicas
   - Async replication for low latency
   - Replicas in different availability zones

4. **Redis Sentinel**:
   - Monitors master health (heartbeat)
   - Automatic failover if master dies
   - Promotes replica to master in <30 seconds

**Benefits:**
- **Minimal Reshuffling:** Adding/removing nodes only affects ~1/N keys
- **High Availability:** Replicas provide automatic failover
- **Horizontal Scaling:** Add more nodes as traffic grows
- **Load Balancing:** Consistent hashing distributes load evenly

**Performance:**
- 99.99% cache hit rate for hot users
- Sub-1ms latency per operation
- 100K+ QPS per node

---

## 7. Multi-Region Deployment

```mermaid
graph TB
    subgraph "US-East-1 (Primary)"
        US_LB[Load Balancer]
        US_PS[Post Service]
        US_TS[Timeline Service]
        US_Kafka[Kafka Cluster]
        US_Redis[Redis Cluster]
        US_Cass[Cassandra DC1]
        US_PG[PostgreSQL Primary]
    end
    
    subgraph "EU-West-1 (Secondary)"
        EU_LB[Load Balancer]
        EU_PS[Post Service]
        EU_TS[Timeline Service]
        EU_Kafka[Kafka Cluster]
        EU_Redis[Redis Cluster]
        EU_Cass[Cassandra DC2]
        EU_PG[PostgreSQL Replica]
    end
    
    subgraph "AP-South-1 (Secondary)"
        AP_LB[Load Balancer]
        AP_PS[Post Service]
        AP_TS[Timeline Service]
        AP_Kafka[Kafka Cluster]
        AP_Redis[Redis Cluster]
        AP_Cass[Cassandra DC3]
        AP_PG[PostgreSQL Replica]
    end
    
    US_LB --> US_PS
    US_LB --> US_TS
    EU_LB --> EU_PS
    EU_LB --> EU_TS
    AP_LB --> AP_PS
    AP_LB --> AP_TS
    
    US_PG -.Async Replication.-> EU_PG
    US_PG -.Async Replication.-> AP_PG
    
    US_Cass -.Multi-Region Repl.-> EU_Cass
    US_Cass -.Multi-Region Repl.-> AP_Cass
    EU_Cass -.Multi-Region Repl.-> AP_Cass
    
    US_Kafka -.Mirror Maker.-> EU_Kafka
    US_Kafka -.Mirror Maker.-> AP_Kafka
    
    style US_LB fill:#e1ffe1
    style EU_LB fill:#e1f5ff
    style AP_LB fill:#f0e1ff
```

**Flow Explanation:**

Shows multi-region deployment for low-latency global access and disaster recovery.

**Architecture:**

1. **Regional Independence**:
   - Each region has complete stack (services, Kafka, Redis, Cassandra)
   - Users routed to nearest region (geo-DNS)
   - Read path served locally (<50ms latency)

2. **Write Path**:
   - Writes go to primary region (US-East-1)
   - PostgreSQL async replication to other regions
   - Kafka MirrorMaker replicates events cross-region
   - Cassandra multi-datacenter replication (tunable consistency)

3. **Read Path**:
   - Served from local region (lowest latency)
   - Redis cache populated per region
   - Cassandra reads use `LOCAL_QUORUM` (fast)

4. **Data Consistency**:
   - **PostgreSQL:** Async replication (eventual consistency across regions)
   - **Cassandra:** Multi-datacenter with NetworkTopologyStrategy
   - **Redis:** Independent per region (eventual consistency)
   - **Kafka:** MirrorMaker for event propagation

**Benefits:**
- **Low Latency:** Users always access nearest region (<100ms globally)
- **High Availability:** Region failure doesn't affect others
- **Disaster Recovery:** Full data replication across regions
- **Compliance:** Data residency requirements met

**Trade-offs:**
- **Cross-Region Lag:** Tweets might appear seconds later in other regions
- **Complexity:** More infrastructure to manage
- **Cost:** 3× infrastructure for full replication

---

## 8. Cassandra Data Model

```mermaid
graph LR
    subgraph "UserTimeline Table Schema"
        UT[CREATE TABLE user_timeline]
        UT --> PK[user_id BIGINT<br/>Partition Key]
        UT --> CK[tweet_id BIGINT<br/>Clustering Key DESC]
        UT --> TD[tweet_data BLOB<br/>Denormalized Content]
    end
    
    subgraph "Data Distribution"
        U1[user_id: 12345] --> P1[Partition 1<br/>Node 1, 2, 3]
        U2[user_id: 67890] --> P2[Partition 2<br/>Node 4, 5, 6]
        U3[user_id: 11111] --> P3[Partition 3<br/>Node 7, 8, 9]
    end
    
    subgraph "Query Pattern"
        Q[SELECT * FROM user_timeline<br/>WHERE user_id = ?<br/>LIMIT 50] --> Fast[O1 Partition Lookup<br/>Pre-sorted by tweet_id<br/>~5ms latency]
    end
    
    style PK fill:#e1ffe1
    style CK fill:#fff4e1
    style Fast fill:#e1f5ff
```

**Flow Explanation:**

Shows how Cassandra's data model is optimized for timeline queries.

**Schema Design:**

1. **Partition Key (`user_id`)**:
   - Determines which nodes store the data
   - All tweets for a user co-located on same partition
   - Enables fast lookups: O(1) to find partition

2. **Clustering Key (`tweet_id DESC`)**:
   - Data sorted within partition by tweet_id (descending)
   - Most recent tweets first (natural timeline order)
   - No sort needed on read

3. **Denormalized Data (`tweet_data BLOB`)**:
   - Full tweet content stored in timeline
   - Avoids joins (no foreign keys in Cassandra)
   - Trades storage for read performance

**Query Pattern:**
```sql
SELECT * FROM user_timeline 
WHERE user_id = ? 
LIMIT 50;
```

**Execution:**
1. Hash `user_id` to find partition
2. Read first 50 rows (already sorted)
3. Return results (~5ms total)

**Benefits:**
- **Fast Reads:** O(1) partition lookup + sequential read
- **Horizontal Scaling:** Add nodes, data redistributes automatically
- **High Throughput:** 50K+ writes/sec per node
- **No Joins:** Denormalization eliminates expensive joins

**Trade-offs:**
- **Storage Cost:** Denormalization duplicates data
- **Update Complexity:** Changes to tweet require updating all timelines
- **Eventual Consistency:** Cassandra uses BASE model (not ACID)

---

## 9. Kafka Partitioning Strategy

```mermaid
graph TB
    Producer[Post Service<br/>Produces Tweet Event] --> Partitioner{Partition by<br/>author_user_id}
    
    Partitioner --> P0[Partition 0<br/>Broker 1]
    Partitioner --> P1[Partition 1<br/>Broker 2]
    Partitioner --> P2[Partition 2<br/>Broker 3]
    Partitioner --> P3[Partition N...<br/>Broker N]
    
    P0 --> CG1[Consumer Group 1<br/>Worker 1]
    P1 --> CG2[Consumer Group 1<br/>Worker 2]
    P2 --> CG3[Consumer Group 1<br/>Worker 3]
    P3 --> CG4[Consumer Group 1<br/>Worker N]
    
    CG1 --> Fanout1[Fanout to Followers]
    CG2 --> Fanout2[Fanout to Followers]
    CG3 --> Fanout3[Fanout to Followers]
    CG4 --> Fanout4[Fanout to Followers]
    
    style Partitioner fill:#fff4e1
    style P0 fill:#e1ffe1
    style P1 fill:#e1ffe1
    style P2 fill:#e1ffe1
    style P3 fill:#e1ffe1
```

**Flow Explanation:**

Shows how Kafka partitions tweet events for parallel processing.

**Partitioning Strategy:**

1. **Partition Key:** `author_user_id`
   - All tweets from same user go to same partition
   - Ensures ordering (same user's tweets processed in order)
   - Load distribution across partitions

2. **Partition Assignment:**
   - Hash(user_id) % num_partitions
   - 64 partitions (example: can scale to 1000+)
   - Each partition assigned to one consumer in group

3. **Consumer Groups:**
   - Fanout Workers form consumer group
   - Each worker processes assigned partitions
   - Horizontal scaling: Add workers = more throughput

4. **Processing:**
   - Worker receives tweet event
   - Fetches followers from graph
   - Inserts into timelines (Cassandra)
   - Commits offset to Kafka

**Benefits:**
- **Parallel Processing:** N partitions = N workers processing simultaneously
- **Ordering Guarantee:** Same user's tweets processed in order
- **Fault Tolerance:** If worker dies, Kafka reassigns partition to another
- **Backpressure Handling:** Workers consume at their own pace

**Performance:**
- 64 partitions → 64 parallel workers
- Each worker handles ~36 tweets/sec (2,300 total / 64)
- Consumer lag monitored (alert if >10K messages behind)

**Scaling:**
- Add more partitions as write traffic grows
- Add more consumers to process faster
- Kafka brokers scale independently

---

## 10. Celebrity vs Normal User Flow

```mermaid
graph TB
    Start([Tweet Posted]) --> Check{Author Follower<br/>Count}
    
    Check -->|<10K Normal User| Push[Fanout-on-Write Push]
    Check -->|>10K Celebrity| Pull[Skip Fanout Pull]
    
    Push --> Fetch1[Fetch All Followers<br/>~5,000 users]
    Fetch1 --> Insert1[Insert into Each<br/>Timeline Cassandra<br/>5,000 writes]
    Insert1 --> Cache1[Update Redis Cache<br/>For Active Users]
    Cache1 --> Done1[Timeline Ready<br/>Instant Reads<br/>~5 seconds]
    
    Pull --> Skip[Skip Fanout<br/>No Timeline Writes]
    Skip --> Done2[Mark as Celebrity Tweet]
    
    Done2 -.Read Time.-> UserReq[User Requests Timeline]
    UserReq --> GetPrecomp[Get Pre-computed<br/>Timeline Cassandra]
    GetPrecomp --> GetCeleb[Fetch Celebrity<br/>Recent Tweets PostgreSQL]
    GetCeleb --> Merge[Merge + Sort]
    Merge --> Return[Return Timeline<br/>+20ms overhead]
    
    Done1 -.Read Time.-> FastRead[User Requests Timeline]
    FastRead --> CacheHit[Redis Cache Hit]
    CacheHit --> FastReturn[Return Immediately<br/>~10ms]
    
    style Check fill:#fff4e1
    style Done1 fill:#e1ffe1
    style Done2 fill:#ffe1e1
    style FastReturn fill:#e1ffe1
    style Return fill:#fff4e1
```

**Flow Explanation:**

Compares the processing flow for normal users vs celebrities.

**Normal User Flow (< 10K followers):**
1. **Write Time:** Fanout-on-write (push model)
   - Fetch all followers (~5,000)
   - Insert tweet into each follower's timeline (Cassandra)
   - Update Redis cache for active followers
   - **Duration:** ~5 seconds for fanout completion
2. **Read Time:** Fast cache hit
   - Timeline already pre-computed
   - Redis cache hit returns immediately
   - **Latency:** <10ms

**Celebrity User Flow (> 10K followers):**
1. **Write Time:** Skip fanout
   - Mark tweet as "celebrity tweet"
   - No timeline writes (saves millions of operations)
   - **Duration:** <1 second (just save tweet to PostgreSQL)
2. **Read Time:** Merge on-the-fly
   - Fetch user's pre-computed timeline (from normal users they follow)
   - Fetch celebrity tweets separately (pull model)
   - Merge both, sort by timestamp
   - **Latency:** ~30ms (+20ms overhead for merge)

**Thresholds:**
- **Normal → Celebrity:** When follower count crosses 10,000
- **Celebrity → Normal:** When follower count drops below 10,000
- **Monitoring:** Track distribution (95% normal, 5% celebrity)

**Benefits:**
- **Normal users:** Fast reads (<10ms), optimized for common case
- **Celebrities:** Avoids crushing system with millions of writes
- **Hybrid:** Best of both worlds

**Trade-offs:**
- **Complexity:** Two different code paths
- **Celebrity Latency:** Slightly slower reads for users following many celebrities
- **Tuning:** Threshold (10K) needs adjustment based on traffic patterns

---

## 11. Activity-Based Fanout Optimization

```mermaid
flowchart TD
    Start[Tweet Posted] --> GetFollowers[Fetch All Followers<br/>100K users]
    GetFollowers --> Filter{Check Last<br/>Login Time}
    
    Filter -->|Active <30 days| Active[Active Followers<br/>40K users]
    Filter -->|Inactive >30 days| Inactive[Inactive Followers<br/>60K users]
    
    Active --> Fanout[Insert into Timeline<br/>40K writes]
    Fanout --> UpdateCache[Update Redis Cache]
    UpdateCache --> Done[Fanout Complete<br/>60% reduction]
    
    Inactive --> Skip[Skip Fanout<br/>Save 60K writes]
    Skip --> LazyLoad[On Next Login<br/>Pull Recent Tweets]
    LazyLoad --> BuildTimeline[Build Timeline<br/>On-Demand]
    
    style Filter fill:#fff4e1
    style Active fill:#e1ffe1
    style Inactive fill:#ffe1e1
    style Done fill:#e1f5ff
```

**Flow Explanation:**

Shows optimization to reduce wasted fanout writes to inactive users.

**Problem:**
- User has 100K followers
- 60% haven't logged in for months
- Fanning out to them wastes 60K writes

**Solution: Activity-Based Fanout**

1. **Filter Followers by Activity**:
   - Query last login time for each follower
   - Active: Logged in last 30 days
   - Inactive: Not logged in for 30+ days

2. **Fanout to Active Only** (40K users):
   - Insert tweet into their timelines (Cassandra)
   - Update Redis cache
   - **Savings:** 60% reduction in writes

3. **Handle Inactive** (60K users):
   - Skip fanout (no timeline writes)
   - On next login: Pull recent tweets (fanout-on-read)
   - Build timeline on-demand

**Implementation:**
```sql
SELECT follower_id 
FROM followers 
WHERE followee_id = ? 
  AND last_login > NOW() - INTERVAL '30 days';
```

**Benefits:**
- **Cost Reduction:** 60% fewer writes (saves infrastructure cost)
- **Faster Fanout:** Smaller fanout set completes quicker
- **No User Impact:** Inactive users don't notice (they build timeline on login)

**Trade-offs:**
- **Complexity:** Need to track last login time
- **First Login Slow:** Inactive users experience slower first timeline load
- **Threshold Tuning:** 30 days is configurable based on user behavior

**Metrics:**
- Active user ratio: 40%
- Write savings: 60%
- Fanout time improvement: 50% faster

---

## 12. Monitoring Dashboard Layout

```mermaid
graph TB
    subgraph "Write Path Metrics"
        W1[Tweet Creation Rate<br/>2,300 QPS avg]
        W2[Kafka Publish Latency<br/>5ms p99]
        W3[PostgreSQL Write Latency<br/>20ms p99]
    end
    
    subgraph "Fanout Metrics"
        F1[Fanout Lag<br/>10 seconds p99]
        F2[Kafka Consumer Lag<br/>500 messages]
        F3[Cassandra Write Throughput<br/>50K writes/sec]
    end
    
    subgraph "Read Path Metrics"
        R1[Timeline Request Rate<br/>115K QPS avg]
        R2[Cache Hit Rate<br/>92%]
        R3[Read Latency<br/>10ms cache hit<br/>50ms cache miss]
    end
    
    subgraph "System Health"
        S1[Redis CPU: 60%<br/>Memory: 70%]
        S2[Cassandra CPU: 75%<br/>Disk: 50%]
        S3[PostgreSQL Replication Lag<br/>2 seconds]
    end
    
    subgraph "Alerts"
        A1[🔴 Fanout Lag >60s]
        A2[🟡 Cache Hit Rate <70%]
        A3[🔴 Kafka Consumer Lag >10K]
    end
    
    style W1 fill:#e1f5ff
    style R1 fill:#e1ffe1
    style F1 fill:#fff4e1
    style A1 fill:#ffe1e1
```

**Flow Explanation:**

Shows key metrics and alerts for monitoring the Twitter Timeline system.

**Dashboard Sections:**

1. **Write Path Metrics**:
   - **Tweet Creation Rate:** Current QPS (baseline 2,300, alert if >20K spike)
   - **Kafka Publish Latency:** Time to publish to Kafka (target <5ms p99)
   - **PostgreSQL Write Latency:** Time to insert tweet (target <20ms p99)

2. **Fanout Metrics**:
   - **Fanout Lag:** Time from tweet posted to timeline updated (target <10s p99)
   - **Kafka Consumer Lag:** How many messages behind (alert if >10K)
   - **Cassandra Write Throughput:** Timeline inserts/sec (capacity 50K/sec)

3. **Read Path Metrics**:
   - **Timeline Request Rate:** Current read QPS (baseline 115K)
   - **Cache Hit Rate:** Percentage served from Redis (target >90%)
   - **Read Latency:** P50, P99, P999 (target <10ms cache hit, <50ms miss)

4. **System Health**:
   - **Redis:** CPU, memory, connection count, eviction rate
   - **Cassandra:** CPU, disk I/O, compaction rate, repair status
   - **PostgreSQL:** Replication lag, query time, connection pool

5. **Alerts (PagerDuty/Slack)**:
   - 🔴 **Critical:** Fanout lag >60s, consumer lag >10K, cache hit <50%
   - 🟡 **Warning:** Cache hit <70%, read latency >200ms, CPU >80%
   - 🟢 **Info:** New deployments, config changes

**Monitoring Tools:**
- **Metrics:** Prometheus + Grafana
- **Logs:** ELK Stack (Elasticsearch, Logstash, Kibana)
- **Tracing:** Jaeger for distributed tracing
- **Alerts:** PagerDuty for on-call rotations

---

**Next:** See [sequence-diagrams.md](sequence-diagrams.md) for detailed interaction flows and failure scenarios.

