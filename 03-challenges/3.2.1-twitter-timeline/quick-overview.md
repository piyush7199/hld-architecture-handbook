# Twitter Timeline - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A scalable microblogging timeline system that allows users to post short messages (tweets) and view a personalized feed
containing tweets from people they follow. The system efficiently solves the **Fanout Problem** — distributing one tweet
to millions of followers' timelines in near-real-time.

**Key Characteristics:**

- **Read-heavy** - 50:1 read-to-write ratio (10B timeline loads/day vs 200M tweets/day)
- **Fanout Challenge** - One tweet must reach millions of followers' timelines
- **Sub-200ms latency** - Users expect instant timeline loads
- **Eventual consistency** - A few seconds delay in timeline is acceptable (BASE model)
- **High availability** - 99.99% uptime for read path
- **Viral spike handling** - Must handle 20K+ QPS spikes during viral events

---

## Requirements & Scale

### Functional Requirements

| Requirement            | Description                                                                        | Priority     |
|------------------------|------------------------------------------------------------------------------------|--------------|
| **Post Tweet**         | Users can publish new, short text messages (tweets) up to 280 characters           | Must Have    |
| **Timeline Retrieval** | Users can view a feed (timeline) containing tweets from people they follow         | Must Have    |
| **Real-Time Feed**     | Timeline must be nearly real-time (eventual consistency acceptable within seconds) | Must Have    |
| **Follow/Unfollow**    | Users can follow and unfollow other users                                          | Must Have    |
| **Search & Mentions**  | Users can search tweets and tag others (@mentions)                                 | Nice to Have |

### Scale Estimation

| Metric                  | Assumption                                  | Calculation        | Result                     |
|-------------------------|---------------------------------------------|--------------------|----------------------------|
| **Total Users**         | 500 Million MAU (Monthly Active Users)      | -                  | -                          |
| **Active Users**        | 100 Million DAU (Daily Active Users)        | -                  | -                          |
| **Posts (Writes)**      | 200 Million tweets per day                  | 200M / 86400 s/day | ~2,300 Writes/sec (QPS)    |
| **Feed Views (Reads)**  | 10 Billion timeline loads per day           | 10B / 86400 s/day  | ~115,740 Reads/sec (QPS)   |
| **Read:Write Ratio**    | Reads outweigh writes                       | 115,740 / 2,300    | ~50:1 (read-heavy)         |
| **Avg Followers**       | 200 followers per user (median)             | -                  | -                          |
| **Fanout Operations**   | Per tweet, deliver to 200 followers         | 200M tweets × 200  | ~40 billion fanout ops/day |
| **Storage (Tweets)**    | 200M tweets/day × 1 KB average              | 200M × 1 KB        | ~200 GB/day, ~73 TB/year   |
| **Storage (Timelines)** | 100M users × 2000 cached tweets × 100 bytes | 100M × 2000 × 100B | ~20 TB in cache            |

**Key Insight:** The system is **read-heavy** (50:1 ratio). The critical challenge is the **Fanout Problem** —
efficiently distributing one tweet to millions of followers' timelines.

---

## Key Components

| Component            | Responsibility                                   | Technology Options                | Scalability                     |
|----------------------|--------------------------------------------------|-----------------------------------|---------------------------------|
| **Load Balancer**    | Distribute traffic, SSL termination, geo-routing | NGINX, HAProxy, AWS ALB           | Horizontal (multi-region)       |
| **API Gateway**      | Authentication, rate limiting, routing           | Kong, AWS API Gateway             | Horizontal                      |
| **Post Service**     | Tweet validation, ID generation, persistence     | Go, Java, Kotlin (stateless)      | Horizontal                      |
| **Timeline Service** | Timeline retrieval, merging, ranking             | Go, Rust, Java (stateless)        | Horizontal                      |
| **Kafka**            | Message queue for async fanout                   | Apache Kafka                      | Horizontal (partitioned)        |
| **Fanout Workers**   | Distribute tweets to followers' timelines        | Go, Java (consumer groups)        | Horizontal (scale workers)      |
| **Timeline Cache**   | Hot timelines for active users                   | Redis Cluster, KeyDB              | Horizontal (consistent hashing) |
| **UserTimeline DB**  | Persistent timeline storage (denormalized)       | Cassandra, ScyllaDB               | Horizontal (sharding)           |
| **Tweet Storage**    | Source of truth for tweets                       | PostgreSQL (sharded), CockroachDB | Horizontal (sharding)           |
| **Follower Graph**   | Social graph (who follows whom)                  | Neo4j, PostgreSQL (sharded)       | Horizontal (graph partitioning) |

---

## Data Model

### Tweets Table (PostgreSQL - Source of Truth)

| Field                   | Data Type    | Notes                                     |
|-------------------------|--------------|-------------------------------------------|
| **tweet_id**            | BIGINT       | Primary Key (Snowflake ID, time-sortable) |
| **user_id**             | BIGINT       | Author of the tweet, indexed              |
| **content**             | VARCHAR(280) | Tweet text                                |
| **created_at**          | TIMESTAMP    | Creation timestamp                        |
| **reply_to_tweet_id**   | BIGINT       | Nullable, for reply threads               |
| **retweet_of_tweet_id** | BIGINT       | Nullable, for retweets                    |
| **like_count**          | INT          | Denormalized counter (updated async)      |
| **retweet_count**       | INT          | Denormalized counter (updated async)      |
| **status**              | ENUM         | ACTIVE, DELETED, HIDDEN                   |

**Sharding Strategy:** Shard by `tweet_id` (time-based, enables range queries for recent tweets).

**Why PostgreSQL?** Strong consistency for source of truth. Tweets must never be lost or duplicated. ACID guarantees are
critical.

### UserTimeline Table (Cassandra - Denormalized Timeline)

| Field          | Data Type | Notes                              |
|----------------|-----------|------------------------------------|
| **user_id**    | BIGINT    | Partition Key                      |
| **tweet_id**   | BIGINT    | Clustering Key (sorted descending) |
| **tweet_data** | BLOB      | Denormalized tweet content         |
| **created_at** | TIMESTAMP | For sorting and filtering          |

**Why Denormalize?** Avoids expensive joins on read. Timeline query becomes simple key lookup:

```sql
SELECT * FROM user_timeline WHERE user_id = ? LIMIT 50;
```

**Why Cassandra?**

- Optimized for high-throughput writes (fanout writes billions of timeline entries/day)
- Cheap storage compared to Redis (~$50/TB vs ~$500/TB)
- Naturally scales horizontally via consistent hashing
- Query pattern is simple: `user_id` → list of `tweet_ids`

### Follower Graph

**Option 1: Graph Database (Neo4j)**

- Optimized for graph traversals
- Sub-ms queries for social graph operations

**Option 2: RDBMS (Sharded PostgreSQL)**

- Simpler ops if team lacks graph DB expertise
- Works well with proper indexing
- Shard by `follower_id` for write path, replicate for read path fanout queries

---

## Write Path: Posting a Tweet

### Sequence

1. **Client** → POST `/tweet` (content, media URLs)
2. **API Gateway** → Authenticate user, check rate limit
3. **Post Service**:
    - Validate content (length, spam detection)
    - Generate Snowflake `tweet_id`
    - Save tweet to **PostgreSQL** (source of truth)
    - Publish event to **Kafka** (`new_tweets` topic)
    - Return success to client (immediately, without waiting for fanout)
4. **Fanout Workers** (async):
    - Consume tweet event from Kafka
    - Fetch author's followers from **Follower Graph`
    - For each follower: Insert tweet into **UserTimeline** (Cassandra)
    - Update **Timeline Cache** (Redis) for active followers

### Latency

- Post API response: < 100 ms (write to PostgreSQL + Kafka publish)
- Fanout completion: < 10 seconds (async, user doesn't wait)

**Why Kafka?** Decouples Post Service from Fanout Workers. During viral events (millions of tweets/minute), Kafka
absorbs the spike. Fanout workers process at their own pace (backpressure handling).

---

## Read Path: Loading Timeline

### Sequence

1. **Client** → GET `/timeline`
2. **Timeline Service**:
    - Check **Redis Cache** for `user_id` → timeline
    - **Cache Hit** → Return immediately (< 10 ms)
    - **Cache Miss** → Query **Cassandra** UserTimeline table
    - Load most recent N tweets (e.g., 50)
    - Populate Redis cache (TTL: 1 hour)
    - Return to client
3. **Client** → Display timeline

### Performance

- Cache hit: < 10 ms (P99)
- Cache miss (Cassandra): < 50 ms (P99)
- Target: < 200 ms end-to-end (including network latency)

**Optimization: Pagination**

Load timelines in chunks (50 tweets at a time). As user scrolls, fetch next page:

```
GET /timeline?cursor=<last_tweet_id>&limit=50
```

---

## Fanout Strategies: The Core Challenge

The **Fanout Problem** is the central challenge. When @Cristiano (500M followers) posts, how do we efficiently deliver
it to all followers?

### Strategy 1: Fanout-on-Write (Push Model)

**How It Works:**
When a tweet is posted, immediately push it to all followers' timelines (pre-compute).

**Implementation:**

1. Save tweet to database
2. Fetch all followers from Follower Graph
3. For each follower: INSERT into their UserTimeline (Cassandra)
4. Update Timeline Cache for active followers

**Advantages:**

- ✅ **Fast Reads:** Timeline is pre-built, reading is just a key lookup (O(1))
- ✅ **Low Read Latency:** Users see updates instantly when they refresh
- ✅ **Simple Read Logic:** No complex merge/sort on read path

**Disadvantages:**

- ❌ **Expensive Writes:** One tweet = N writes (where N = number of followers)
- ❌ **Celebrity Problem:** When celebrity posts, system must perform millions of writes
- ❌ **Wasted Work:** Many followers might be inactive (never read the tweet)

**When to Use:** For normal users with < 10,000 followers.

### Strategy 2: Fanout-on-Read (Pull Model)

**How It Works:**
When a user requests their timeline, fetch tweets from all accounts they follow and merge on-the-fly.

**Implementation:**

1. Fetch all followees from Follower Graph
2. For each followee: Fetch their N most recent tweets
3. Merge all tweets, sort by timestamp, return top 50

**Advantages:**

- ✅ **Cheap Writes:** One tweet = 1 write (just save the tweet)
- ✅ **No Celebrity Problem:** Doesn't matter how many followers someone has
- ✅ **No Wasted Work:** Only compute timelines for active users

**Disadvantages:**

- ❌ **Slow Reads:** Must query N accounts (where N = number of followees), merge, sort
- ❌ **Complex Read Logic:** Merging N sorted lists is O(N log N)
- ❌ **High Read Latency:** Can take seconds for users following thousands

**When to Use:** For celebrity accounts with millions of followers.

### Strategy 3: Hybrid Fanout (Twitter's Approach) ✅ Recommended

**How It Works:**

- **Normal users** (< 10,000 followers): Use Fanout-on-Write (push)
- **Celebrity users** (> 10,000 followers): Use Fanout-on-Read (pull)
- **Timeline Reads:** Merge pre-computed timeline (push) with on-demand celebrity tweets (pull)

**Write Path:**

```
If author has < 10,000 followers:
    Fanout to all followers (push model)
Else:
    Mark as celebrity, skip fanout
```

**Read Path:**

```
1. Fetch pre-computed timeline from cache/Cassandra (covers ~90% of feed)
2. For each celebrity the user follows:
    Fetch their N most recent tweets
3. Merge both result sets, sort by timestamp
4. Return top 50 tweets
```

**Advantages:**

- ✅ Balances write cost and read latency
- ✅ Handles celebrity problem gracefully
- ✅ Optimizes for the common case (95% of users)

**Disadvantages:**

- ❌ Increased system complexity
- ❌ More challenging to debug and monitor
- ❌ Celebrity tweets might appear slightly slower (pull on read)

---

## Handling Celebrity Problem: Optimizations

**Problem:** When @Cristiano (500M followers) posts, hybrid model still requires fetching his tweet for every timeline
load.

### Optimization 1: Celebrity Tweet Cache

Cache celebrity tweets separately in Redis:

```
celebrity:<user_id>:recent_tweets → [tweet_id1, tweet_id2, ...]
TTL: 1 hour
```

When loading timeline, batch-fetch celebrity tweets from this cache.

### Optimization 2: Tiered Fanout

Even for celebrities, fanout to "super fans" (highly active followers):

```
If follower logged in last 24 hours:
    Fanout tweet to their timeline (push)
Else:
    Skip (they'll pull on next login)
```

### Optimization 3: Dedicated Celebrity Pipeline

- Separate Kafka topic: `celebrity_tweets`
- Higher-priority fanout workers
- Separate monitoring and alerting

---

## Bottlenecks & Solutions

### Bottleneck 1: Celebrity Problem (High Fanout)

**Problem:** When @BarackObama (130M followers) posts, system must handle 130M timeline inserts.

**Current Mitigation:**

- Hybrid fanout: Use pull model for celebrities
- Tiered fanout: Only push to active followers

**Future Enhancements:**

- **Sampling:** Only deliver to subset of followers, others fetch on-demand
- **Bloom Filter:** Skip followers who recently saw a tweet from this celebrity
- **Geographically Tiered:** Fanout to users in same region first (locality)

### Bottleneck 2: Cache Hotspots

**Problem:** Timelines of popular users (followed by millions) requested millions of times/sec.

**Mitigation:**

- **Consistent Hashing:** Distribute popular `user_id` keys across Redis cluster
- **Multi-Master Redis:** High availability, automatic failover
- **CDN Caching:** For public profiles/timelines (read-only)
- **Connection Pooling:** Reuse connections to Redis (avoid TCP overhead)

### Bottleneck 3: Storage Growth

**Problem:** 200 million tweets/day = 73 TB/year. Timeline storage grows even faster (denormalized).

**Mitigation:**

- **Archive Old Tweets:** Move tweets > 6 months to cold storage (S3, Glacier)
- **Timeline Pruning:** Limit timeline history to 10,000 most recent tweets per user
- **Compression:** Use columnar storage and compression in Cassandra
- **Sharding:** Partition Cassandra by `user_id` across hundreds of nodes

### Bottleneck 4: Kafka Throughput

**Problem:** At peak (viral event), Kafka might receive 50,000 messages/sec.

**Mitigation:**

- **Partition Kafka Topic:** Use `user_id` as partition key → spread load across brokers
- **Batch Writes:** Fanout workers batch timeline inserts (100 writes/batch) to reduce overhead
- **Scale Kafka Cluster:** Add more brokers as traffic grows
- **Monitor Consumer Lag:** Alert when consumer lag exceeds threshold (slow fanout)

---

## Common Anti-Patterns to Avoid

### 1. Fanout to All Followers (Including Inactive)

❌ **Bad:** When a celebrity posts, fanning out to all 500M followers wastes writes. 90% of those followers might be
inactive (haven't logged in for months).

**Why It's Bad:**

- Wastes database writes and storage
- Increases fanout lag
- Higher infrastructure cost

✅ **Good:** Activity-Based Fanout

- Only fanout to followers who logged in recently (last 30 days)
- Skip inactive followers (they'll fetch on next login using pull model)

### 2. Synchronous Fanout (Blocking Write)

❌ **Bad:** If Post Service waits for fanout to complete before returning to client, users with many followers experience
slow post times (several seconds).

**Why It's Bad:**

- Poor user experience (slow post confirmation)
- Blocks write path during viral events
- Single point of failure

✅ **Good:** Asynchronous Fanout

- Post Service returns immediately after saving to DB and publishing to Kafka
- Fanout workers process asynchronously
- User doesn't wait for fanout completion

### 3. No Caching for Timeline Reads

❌ **Bad:** Every timeline request queries Cassandra, causing high latency and database load.

✅ **Good:** Multi-Layer Caching

- Redis cache for hot timelines (90%+ hit rate)
- CDN for public profiles
- Cache warming for active users

### 4. Fanout-on-Read for All Users

❌ **Bad:** Using pull model for all users causes slow reads (must query N accounts, merge, sort).

✅ **Good:** Hybrid Approach

- Push model for normal users (< 10K followers)
- Pull model only for celebrities (> 10K followers)
- Merge both on read path

---

## Monitoring & Observability

### Key Metrics

| Metric                          | Target       | Alert Threshold | Why Important        |
|---------------------------------|--------------|-----------------|----------------------|
| **Timeline Read Latency (p99)** | < 200 ms     | > 500 ms        | User experience      |
| **Post Write Latency (p99)**    | < 100 ms     | > 500 ms        | User experience      |
| **Fanout Lag**                  | < 10 seconds | > 60 seconds    | Timeline freshness   |
| **Cache Hit Rate**              | > 90%        | < 80%           | Database load        |
| **Kafka Consumer Lag**          | < 1 minute   | > 5 minutes     | Fanout delay         |
| **Error Rate**                  | < 0.1%       | > 1%            | Service health       |
| **Celebrity Fanout Time**       | < 30 seconds | > 2 minutes     | Viral tweet handling |

---

## Trade-offs Summary

| Decision              | Choice                   | Alternative             | Why Chosen                  | Trade-off               |
|-----------------------|--------------------------|-------------------------|-----------------------------|-------------------------|
| **Fanout Strategy**   | Hybrid (Push + Pull)     | Pure Push or Pure Pull  | Balances cost and latency   | Increased complexity    |
| **Timeline Storage**  | Denormalized (Cassandra) | Normalized (PostgreSQL) | Fast reads, cheap writes    | Storage overhead        |
| **Tweet Storage**     | PostgreSQL (ACID)        | Cassandra               | Strong consistency          | Write bottleneck        |
| **Consistency Model** | Eventual (BASE)          | Strong (ACID)           | Performance and scalability | Temporary inconsistency |
| **Caching Strategy**  | Redis (in-memory)        | Database-only           | Sub-10ms reads              | Memory cost             |
| **Message Queue**     | Kafka                    | RabbitMQ, SQS           | High throughput, durability | Operational complexity  |

---

## Key Takeaways

1. **Hybrid Fanout** balances write cost and read latency (push for normal users, pull for celebrities)
2. **Denormalized timelines** in Cassandra enable fast reads (O(1) key lookup)
3. **Asynchronous fanout** via Kafka decouples write path from fanout processing
4. **Multi-layer caching** (Redis + CDN) achieves < 10ms read latency for 90%+ requests
5. **Activity-based fanout** reduces wasted writes (only fanout to active followers)
6. **Eventual consistency** is acceptable for timelines (BASE model)
7. **Celebrity problem** handled via pull model + dedicated caching
8. **Kafka partitions** distribute fanout load across workers
9. **Sharding strategies** enable horizontal scaling (by tweet_id, user_id)
10. **Monitoring fanout lag** is critical for timeline freshness

---

## Recommended Stack

- **Post Service:** Go or Java (stateless, horizontal scaling)
- **Timeline Service:** Go or Rust (low latency, high throughput)
- **Message Queue:** Apache Kafka (high throughput, durability)
- **Timeline Cache:** Redis Cluster (consistent hashing, high availability)
- **Timeline Storage:** Cassandra (denormalized, high write throughput)
- **Tweet Storage:** PostgreSQL (ACID, source of truth)
- **Follower Graph:** Neo4j or PostgreSQL (graph queries or simple joins)
- **Load Balancer:** NGINX or AWS ALB (geo-routing, SSL termination)
- **Monitoring:** Prometheus + Grafana (metrics, alerting)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, fanout strategies, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (posting, timeline loading, fanout)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (fanout strategies, timeline merging)

