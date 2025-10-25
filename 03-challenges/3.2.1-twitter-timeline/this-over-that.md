# Design Decisions: This Over That - Twitter Timeline

This document provides in-depth analysis of the major architectural choices made in the Twitter Timeline system, including alternatives considered, trade-offs, and conditions that would change these decisions.

---

## 1. Fanout Strategy: Hybrid vs Pure Push vs Pure Pull

### The Problem

When a user posts a tweet, how do we efficiently deliver it to their followers' timelines?

**Constraints:**
- 50:1 read-to-write ratio (read-heavy workload)
- Users have varying follower counts (0 to 500M)
- Sub-200ms timeline load latency requirement
- System must handle viral events (20K+ QPS spikes)

### Options Considered

| Strategy | Write Cost | Read Speed | Celebrity Problem | Complexity |
|----------|-----------|------------|-------------------|------------|
| **Pure Fanout-on-Write (Push)** | High ($N$ writes per tweet) | Very Fast (O(1)) | ❌ Crushes system | Low |
| **Pure Fanout-on-Read (Pull)** | Low (1 write per tweet) | Slow (O(N log N)) | ✅ Handles well | Medium |
| **Hybrid (Push + Pull)** | Medium (conditional) | Fast (O(1) + overhead) | ✅ Handles well | High |

#### Option A: Pure Fanout-on-Write (Push Model)

**How It Works:**
- When tweet posted, immediately insert into all followers' timelines
- Timelines are pre-computed and ready to read

**Pros:**
- ✅ Fastest reads possible (<10ms cache hits)
- ✅ Simple read logic (just key lookup)
- ✅ Optimal for read-heavy workloads

**Cons:**
- ❌ **Celebrity Problem:** When @Cristiano (500M followers) posts, system must perform 500M writes
- ❌ **Wasted Work:** 90% of followers might be inactive (never read the tweet)
- ❌ **Infrastructure Cost:** Requires massive write capacity ($\sim 10\times$ larger DB cluster)
- ❌ **Fanout Lag:** Takes minutes to complete for celebrities (poor UX)

**Example Scenario:**
```
User with 100K followers posts tweet
→ 100K Cassandra writes required
→ At 1ms per write = 100 seconds to complete
→ Celebrity with 10M followers = 2.7 hours!
```

#### Option B: Pure Fanout-on-Read (Pull Model)

**How It Works:**
- Don't pre-compute timelines
- When user requests timeline, fetch tweets from all followees and merge

**Pros:**
- ✅ Cheap writes (1 write per tweet)
- ✅ No celebrity problem
- ✅ No wasted work (only compute for active users)
- ✅ Always up-to-date (no cache consistency issues)

**Cons:**
- ❌ **Slow Reads:** Must query $N$ accounts, merge, and sort (where $N$ = followee count)
- ❌ **Complex Merge Logic:** Merging $N$ sorted lists is $\text{O}(N \log N)$
- ❌ **High CPU Cost:** Every timeline load requires computation
- ❌ **Poor Scalability:** Users following 1000+ accounts experience >1 second latency

**Example Scenario:**
```
User follows 1000 accounts, requests timeline
→ Fetch last 50 tweets from each: 1000 DB queries
→ Merge 50K tweets, sort, return top 50
→ Total: 1-2 seconds (unacceptable)
```

#### Option C: Hybrid Fanout (Twitter's Approach) ✅

**How It Works:**
- Normal users (<10K followers): Use push model
- Celebrities (>10K followers): Use pull model
- Timeline reads: Merge pre-computed + celebrity tweets

**Pros:**
- ✅ Balances write cost and read latency
- ✅ Handles celebrity problem gracefully
- ✅ Optimizes for common case (95% of users)
- ✅ Read latency acceptable (<30ms with celebrity overhead)

**Cons:**
- ❌ **Increased Complexity:** Two different code paths
- ❌ **Harder to Debug:** More edge cases and failure modes
- ❌ **Threshold Tuning:** 10K threshold needs adjustment based on traffic
- ❌ **Celebrity Timeline Lag:** Users following many celebrities see slightly slower loads

**Example Scenario:**
```
Normal user (5K followers) posts:
→ Fanout to 5K timelines (push) = 5 seconds
→ Followers see tweet instantly (<10ms cache hit)

Celebrity (10M followers) posts:
→ Skip fanout (pull model) = 1 second
→ Followers fetch on-demand (+20ms merge overhead)

Result: Balanced system, handles both cases
```

### Decision Made

**✅ Hybrid Fanout (Push for normal, Pull for celebrities)**

**Rationale:**
1. **Optimizes for Common Case:** 95% of users have <10K followers → push model gives them <10ms timeline loads
2. **Handles Celebrity Problem:** Avoids crushing system when celebrities post
3. **Production-Proven:** Twitter, Instagram use similar approach
4. **Read-Heavy Workload:** Pre-computing timelines is worth it for 50:1 read ratio

**Implementation Details:**
- Threshold: 10,000 followers (configurable via feature flag)
- Push model: Fanout to all followers, insert into Cassandra
- Pull model: Skip fanout, fetch on timeline load
- Merge overhead: ~20ms to merge celebrity tweets

### Trade-offs Accepted

| Trade-off | Impact | Mitigation |
|-----------|--------|------------|
| **Complexity** | Two code paths to maintain | Comprehensive testing, feature flags |
| **Celebrity Latency** | +20ms for users following many celebrities | Cache celebrity tweets separately |
| **Threshold Tuning** | Requires monitoring and adjustment | A/B testing, gradual rollout |

### When to Reconsider

**Switch to Pure Pull if:**
- Read-to-write ratio drops below 10:1 (more write-heavy)
- Infrastructure cost becomes prohibitive
- User behavior shifts to following thousands of accounts

**Switch to Pure Push if:**
- Celebrity problem solved differently (e.g., sampling)
- Storage becomes extremely cheap
- Read latency requirements become even stricter (<5ms)

**Adjust Threshold if:**
- Fanout lag exceeds target (increase threshold to reduce push fanout)
- Cache hit rate drops (decrease threshold to pre-compute more)
- Celebrity users complain about slow timeline propagation

---

## 2. Timeline Storage: Cassandra vs DynamoDB vs PostgreSQL

### The Problem

Where should we store denormalized user timelines for fast retrieval?

**Requirements:**
- High-throughput writes (40B timeline inserts/day)
- Fast key-based reads (user_id → timeline)
- Horizontal scalability (petabyte-scale)
- Reasonable cost (<$100/TB/month)

### Options Considered

| Database | Write Throughput | Read Latency | Cost | Horizontal Scaling | Ops Complexity |
|----------|-----------------|--------------|------|-------------------|----------------|
| **Cassandra** | 50K/node | 5-10ms | $50/TB | Excellent | High |
| **DynamoDB** | Configurable | 1-5ms | $250/TB | Excellent | Low (managed) |
| **PostgreSQL** | 5K/node | 1-3ms | $100/TB | Medium (sharding) | Medium |
| **Redis** | 100K/node | <1ms | $500/TB | Good | Medium |

#### Option A: Cassandra

**How It Works:**
- Wide-column store optimized for write-heavy workloads
- Data partitioned by user_id, clustered by tweet_id
- Multi-datacenter replication built-in

**Pros:**
- ✅ **Excellent Write Throughput:** 50K+ writes/sec per node
- ✅ **Cheap Storage:** ~$50/TB/month (commodity hardware)
- ✅ **Natural Scaling:** Add nodes, data redistributes automatically
- ✅ **Multi-DC Replication:** Built-in cross-region support
- ✅ **No Single Point of Failure:** Peer-to-peer architecture

**Cons:**
- ❌ **Eventual Consistency:** No ACID guarantees (BASE model)
- ❌ **Operational Complexity:** Requires expertise (compaction, repair, tuning)
- ❌ **No Joins:** Must denormalize all data
- ❌ **Query Limitations:** Only efficient for partition key lookups

**Cost Analysis:**
```
20 TB timeline storage × $50/TB = $1,000/month
9 nodes (RF=3) × $200/node = $1,800/month
Total: ~$2,800/month
```

#### Option B: Amazon DynamoDB

**How It Works:**
- Managed NoSQL database (AWS)
- Automatic scaling, multi-region replication
- Pay-per-request or provisioned capacity

**Pros:**
- ✅ **Fully Managed:** No ops burden (backups, scaling, patching)
- ✅ **Fast Reads:** 1-5ms latency with DAX cache
- ✅ **Auto-Scaling:** Handles traffic spikes automatically
- ✅ **Global Tables:** Multi-region replication built-in

**Cons:**
- ❌ **Expensive:** ~$250/TB/month (5× Cassandra)
- ❌ **Vendor Lock-In:** Tied to AWS ecosystem
- ❌ **Cost Unpredictability:** Can spike with traffic
- ❌ **Limited Control:** Can't tune internal parameters

**Cost Analysis:**
```
20 TB timeline storage × $250/TB = $5,000/month
Provisioned capacity (50K writes/sec) = $3,000/month
Total: ~$8,000/month (3× Cassandra)
```

#### Option C: PostgreSQL (Sharded)

**How It Works:**
- Traditional RDBMS sharded by user_id
- Each shard handles subset of users
- Application-level routing

**Pros:**
- ✅ **ACID Guarantees:** Strong consistency for critical data
- ✅ **Mature Ecosystem:** Well-understood, extensive tooling
- ✅ **Complex Queries:** Supports joins, indexes, aggregations
- ✅ **Team Familiarity:** Most teams already know SQL

**Cons:**
- ❌ **Lower Write Throughput:** ~5K writes/sec per node (10× less than Cassandra)
- ❌ **Sharding Complexity:** Must implement in application layer
- ❌ **Vertical Scaling Limits:** Each shard eventually hits ceiling
- ❌ **Cross-Shard Queries:** Difficult and slow

**Cost Analysis:**
```
20 TB timeline storage ÷ 16 shards = 1.25 TB/shard
16 shards × $300/month = $4,800/month
```

#### Option D: Redis (In-Memory)

**How It Works:**
- Pure in-memory cache
- Persistence via AOF/RDB snapshots
- Cluster mode for horizontal scaling

**Pros:**
- ✅ **Fastest Reads:** <1ms latency
- ✅ **High Throughput:** 100K+ ops/sec per node
- ✅ **Rich Data Structures:** Lists, sets, sorted sets

**Cons:**
- ❌ **Very Expensive:** ~$500/TB/month (RAM cost)
- ❌ **Limited Capacity:** 20 TB in RAM = huge cluster
- ❌ **Durability Concerns:** Persistence is secondary feature
- ❌ **Eviction Risk:** LRU eviction can lose data

**Cost Analysis:**
```
20 TB RAM storage × $500/TB = $10,000/month
Massive cluster required = additional $5,000/month
Total: ~$15,000/month (5× Cassandra)
```

### Decision Made

**✅ Cassandra for timeline storage + Redis for hot cache**

**Rationale:**
1. **Cost-Effective:** $50/TB is 5× cheaper than DynamoDB, 10× cheaper than Redis
2. **Write Performance:** 50K writes/sec per node handles fanout load
3. **Horizontal Scaling:** Add nodes as data grows (no sharding complexity)
4. **Multi-DC Replication:** Built-in support for global deployment
5. **Query Pattern Match:** Our access pattern (partition key lookup) is Cassandra's strength

**Implementation:**
- **Cassandra:** Full timeline history (all 10K tweets per user)
- **Redis:** Hot cache (most recent 2K tweets per user, 1-hour TTL)
- **Reads:** Check Redis first → Fallback to Cassandra on miss

### Trade-offs Accepted

| Trade-off | Impact | Mitigation |
|-----------|--------|------------|
| **Eventual Consistency** | Timelines might lag by seconds | Acceptable for social media |
| **Ops Complexity** | Requires Cassandra expertise | Hire specialists, managed services (ScyllaDB Cloud) |
| **No ACID** | Can't do transactions | Acceptable, tweets are independent |

### When to Reconsider

**Switch to DynamoDB if:**
- Team lacks Cassandra expertise
- Willing to pay 3× cost for managed service
- Prefer vendor lock-in to ops burden

**Switch to PostgreSQL if:**
- Write throughput drops significantly (<10K writes/sec)
- Need complex queries and joins
- Strong consistency becomes critical

**Switch to Pure Redis if:**
- Storage needs drop dramatically (<1 TB)
- Budget increases significantly
- Sub-millisecond latency becomes mandatory

---

## 3. Message Queue: Kafka vs RabbitMQ vs AWS SQS

### The Problem

How do we decouple the Post Service from Fanout Workers to handle write spikes?

**Requirements:**
- Handle 20K+ QPS write spikes (viral events)
- Durability (no message loss)
- Ordered processing (same user's tweets in order)
- Replay capability (reprocess on failure)

### Options Considered

| Queue | Throughput | Durability | Ordering | Replay | Ops Complexity |
|-------|-----------|-----------|----------|--------|----------------|
| **Kafka** | Very High (100K+/sec) | Excellent (replicated log) | Per-partition | ✅ Yes | High |
| **RabbitMQ** | Medium (10K-20K/sec) | Good (quorum queues) | Per-queue | Limited | Medium |
| **AWS SQS** | High (managed) | Excellent | Per-group (FIFO) | ❌ No | Low (managed) |

#### Option A: Apache Kafka ✅

**How It Works:**
- Distributed commit log (not traditional queue)
- Topics divided into partitions
- Consumers track offset (position in log)

**Pros:**
- ✅ **High Throughput:** 100K+ messages/sec per broker
- ✅ **Durability:** Replicated across brokers (RF=3)
- ✅ **Replay:** Consumers can rewind to any offset
- ✅ **Ordering:** Per-partition ordering guaranteed
- ✅ **Retention:** Keep messages for days/weeks (configurable)
- ✅ **Scalability:** Add brokers/partitions as traffic grows

**Cons:**
- ❌ **Operational Complexity:** Requires ZooKeeper, monitoring, tuning
- ❌ **Learning Curve:** Different paradigm from traditional queues
- ❌ **Resource Heavy:** Needs dedicated cluster

**Use Case Fit:**
```
Viral event: 50K tweets/sec spike
→ Kafka absorbs spike (no backpressure on Post Service)
→ Fanout workers consume at steady pace (10K/sec)
→ Consumer lag increases temporarily, recovers within minutes
```

#### Option B: RabbitMQ

**How It Works:**
- Traditional message broker
- Exchanges route to queues
- Consumers acknowledge messages

**Pros:**
- ✅ **Flexible Routing:** Multiple exchange types (topic, fanout, direct)
- ✅ **Mature:** Well-understood, extensive plugins
- ✅ **Lower Ops Complexity:** Easier than Kafka to operate
- ✅ **Message Priority:** Support for priority queues

**Cons:**
- ❌ **Lower Throughput:** 10K-20K messages/sec (5× less than Kafka)
- ❌ **Limited Replay:** Messages deleted after ACK
- ❌ **Vertical Scaling:** Harder to scale horizontally
- ❌ **Memory Pressure:** Can struggle with large backlogs

**Use Case Fit:**
```
Normal load: 2K tweets/sec → RabbitMQ handles fine
Viral event: 50K tweets/sec spike → RabbitMQ struggles
→ Queue backs up, memory pressure, potential message loss
→ Not ideal for unpredictable spikes
```

#### Option C: AWS SQS

**How It Works:**
- Fully managed queue service (AWS)
- Standard (high throughput) or FIFO (ordered)
- Automatic scaling

**Pros:**
- ✅ **Fully Managed:** No ops burden
- ✅ **Auto-Scaling:** Handles any throughput
- ✅ **Dead Letter Queues:** Built-in failure handling
- ✅ **Cheap:** Pay only for usage

**Cons:**
- ❌ **No Replay:** Messages deleted after processing
- ❌ **FIFO Limitations:** Only 300 TPS per message group
- ❌ **Vendor Lock-In:** Tied to AWS
- ❌ **Visibility Timeout:** Complex failure handling

**Use Case Fit:**
```
FIFO requirement for same user's tweets
→ SQS FIFO: 300 TPS per message group
→ If partitioned by user_id: bottleneck for active users
→ Standard queue: no ordering guarantee

Not ideal for our ordering requirements
```

### Decision Made

**✅ Apache Kafka**

**Rationale:**
1. **Throughput:** Handles 50K+ tweets/sec spikes easily
2. **Durability:** Replicated log ensures no message loss
3. **Replay:** Can reprocess failed fanouts from any offset
4. **Ordering:** Partition by user_id ensures same user's tweets processed in order
5. **Industry Standard:** Twitter, LinkedIn, Uber all use Kafka for similar use cases

**Implementation:**
- **Topic:** `new_tweets` (64 partitions)
- **Partition Key:** `author_user_id` (ensures ordering per user)
- **Retention:** 7 days (allows delayed processing)
- **Replication Factor:** 3 (durability)

### Trade-offs Accepted

| Trade-off | Impact | Mitigation |
|-----------|--------|------------|
| **Ops Complexity** | Requires dedicated Kafka team | Managed services (Confluent Cloud, AWS MSK) |
| **Resource Cost** | Dedicated cluster needed | Amortize across multiple use cases |
| **Learning Curve** | Team must learn Kafka concepts | Training, documentation, runbooks |

### When to Reconsider

**Switch to RabbitMQ if:**
- Throughput drops below 10K messages/sec
- Team lacks Kafka expertise
- Prefer simpler operational model

**Switch to AWS SQS if:**
- Willing to sacrifice replay capability
- Prefer fully managed service
- Cost-sensitive (SQS is cheaper for low volumes)

**Stick with Kafka if:**
- Throughput continues to grow
- Replay capability proves valuable
- Planning to use Kafka for other use cases (analytics, CDC, etc.)

---

## 4. Tweet Storage: PostgreSQL vs MySQL vs Cassandra

### The Problem

Where should we store the source of truth for tweets?

**Requirements:**
- Strong consistency (ACID)
- Fast writes (2,300 tweets/sec)
- Time-based range queries (recent tweets)
- Support for relational features (replies, likes, retweets)

### Decision Made

**✅ PostgreSQL (Sharded by tweet_id)**

**Rationale:**
1. **ACID Guarantees:** Tweets must never be lost or duplicated
2. **Relational Model:** Natural fit for replies, likes, retweets
3. **Mature Ecosystem:** Extensive tooling, monitoring, backup solutions
4. **Time-Sortable IDs:** Snowflake IDs enable efficient sharding
5. **Secondary Indexes:** Support for user_id, hashtag queries

**Alternatives Considered:**

| Database | Consistency | Relations | Throughput | Scalability |
|----------|------------|-----------|------------|-------------|
| **PostgreSQL** | ✅ ACID | ✅ Excellent | Medium (5K/sec) | Medium (sharding) |
| **MySQL** | ✅ ACID | ✅ Excellent | Medium (5K/sec) | Medium (sharding) |
| **Cassandra** | ❌ BASE | ❌ None | High (50K/sec) | Excellent |

**Why Not Cassandra for Tweets:**
- No ACID guarantees → Risk of duplicate tweets
- No joins → Hard to model replies, quote tweets
- Eventual consistency → User might not see their own tweet immediately

**Why PostgreSQL over MySQL:**
- Better JSON support (for tweet metadata)
- More advanced indexing (GIN, GiST)
- Team familiarity

---

## 5. Follower Graph: Neo4j vs PostgreSQL vs Redis

### The Problem

How do we efficiently fetch all followers for fanout operations?

**Requirements:**
- Fast follower queries (<50ms for 100K followers)
- Support for graph traversals (mutual friends, suggestions)
- Horizontal scalability

### Options Considered

#### Option A: Neo4j (Graph Database) ✅

**Pros:**
- ✅ Optimized for graph traversals
- ✅ Sub-millisecond queries for social graphs
- ✅ Natural data model for relationships

**Cons:**
- ❌ Less common (team learning curve)
- ❌ Sharding complexity for large graphs

#### Option B: PostgreSQL (Sharded)

**Pros:**
- ✅ Team familiarity
- ✅ Works fine with proper indexing

**Cons:**
- ❌ Not optimized for graph queries
- ❌ Complex queries require multiple joins

### Decision Made

**✅ Neo4j for production, PostgreSQL acceptable alternative**

**Rationale:**
- Neo4j optimal for "followers of followers" (mutual connections)
- PostgreSQL fine if team lacks graph DB expertise
- Query is simple: `SELECT follower_id FROM followers WHERE followee_id = ?`

---

## 6. Caching Strategy: Redis vs Memcached

### Decision Made

**✅ Redis**

**Rationale:**
1. **Data Structures:** Lists for timelines (LPUSH, LRANGE)
2. **Persistence:** Optional durability (AOF)
3. **Atomic Operations:** INCR for counters
4. **Lua Scripting:** Complex operations in one round-trip

**Why Not Memcached:**
- Only supports key-value (no lists)
- No persistence
- Less flexible

---

## 7. Consistency Model: ACID vs BASE

### Decision Made

**✅ Hybrid: ACID for tweets, BASE for timelines**

**Rationale:**
- **Tweets (PostgreSQL):** Strong consistency critical (source of truth)
- **Timelines (Cassandra + Redis):** Eventual consistency acceptable
- **User Expectation:** Few seconds delay in timeline is tolerable
- **Real-World Example:** Twitter accepts eventual consistency

**Trade-off:** Users might not see tweet immediately in followers' timelines (2-10 seconds delay).

---

## Summary Table: All Decisions

| Component | Choice | Alternative | Key Reason |
|-----------|--------|-------------|------------|
| **Fanout Strategy** | Hybrid | Pure Push/Pull | Balances write cost & read latency |
| **Timeline Storage** | Cassandra | DynamoDB | 5× cheaper, same performance |
| **Timeline Cache** | Redis | Memcached | Rich data structures, persistence |
| **Message Queue** | Kafka | RabbitMQ, SQS | High throughput, replay capability |
| **Tweet Storage** | PostgreSQL | MySQL, Cassandra | ACID, relational model |
| **Follower Graph** | Neo4j | PostgreSQL | Graph traversals, optimal queries |
| **Consistency Model** | Hybrid | Pure ACID/BASE | Right consistency for each component |

