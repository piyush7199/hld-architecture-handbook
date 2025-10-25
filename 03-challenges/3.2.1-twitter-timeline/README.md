# 3.2.1 Design a Twitter/X Timeline (Microblogging Feed)

## Problem Statement

Design a scalable microblogging timeline system (like Twitter/X) that allows users to post short messages (tweets) and view a personalized feed containing tweets from people they follow. The system must handle 500 million monthly active users, process 200 million tweets per day, serve 10 billion timeline loads per day with sub-200ms latency, and efficiently solve the "Fanout Problem" — distributing one tweet to millions of followers' timelines in near-real-time.

---

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, fanout strategies, data flow, and scaling approaches
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows for posting tweets, timeline loading, fanout processes, and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices and trade-offs
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for fanout strategies, timeline merging, and activity-based fanout

---

## 1. Requirements and Scale Estimation

### Functional Requirements (FRs)

| Requirement | Description | Priority |
|------------|-------------|----------|
| **Post Tweet** | Users can publish new, short text messages (tweets) up to 280 characters | Must Have |
| **Timeline Retrieval** | Users can view a feed (timeline) containing tweets from people they follow | Must Have |
| **Real-Time Feed** | Timeline must be nearly real-time (eventual consistency acceptable within seconds) | Must Have |
| **Follow/Unfollow** | Users can follow and unfollow other users | Must Have |
| **Search & Mentions** | Users can search tweets and tag others (@mentions) | Nice to Have |

### Non-Functional Requirements (NFRs)

| Requirement | Target | Rationale |
|------------|--------|-----------|
| **High Availability** | 99.99% uptime (Read Path) | Timeline unavailability affects millions of users |
| **Low Read Latency** | < 200 ms (p99) | Users expect instant timeline loads |
| **Write Resilience** | Handle 20K+ QPS spikes | Viral tweets cause massive write spikes |
| **Scalability** | Handle 500M MAU, 100M DAU | Long-term growth |
| **Eventual Consistency** | Acceptable (BASE model) | A few seconds delay in timeline is tolerable |

### Scale Estimation

| Metric | Assumption | Calculation | Result |
|--------|-----------|-------------|--------|
| **Total Users** | 500 Million $\text{MAU}$ | - | - |
| **Active Users** | 100 Million $\text{DAU}$ | - | - |
| **Posts (Writes)** | 200 Million tweets per day | $\frac{200 \text{M}}{86400 \text{ s}}$ | $\sim 2,300$ Writes/sec |
| **Feed Views (Reads)** | 10 Billion timeline loads per day | $\frac{10 \text{B}}{86400 \text{ s}}$ | $\sim 115,740$ Reads/sec |
| **Read:Write Ratio** | Reads outweigh writes | $\frac{115,740}{2,300}$ | $\sim 50:1$ (read-heavy) |
| **Storage (Tweets)** | 200M tweets/day × 1 KB | $200 \text{M} \times 1 \text{ KB}$ | $\sim 200$ GB/day, $73$ TB/year |

**Key Challenge:** The **Fanout Problem** — efficiently distributing one tweet to millions of followers' timelines.

---

## 2. High-Level Architecture

> 📊 **See detailed architecture diagrams:** [HLD Diagrams](./hld-diagram.md)

### Key Components

| Component | Responsibility | Technology Options | Scalability |
|-----------|---------------|-------------------|-------------|
| **Load Balancer** | Distribute traffic, SSL termination, geo-routing | NGINX, HAProxy, AWS ALB | Horizontal (multi-region) |
| **API Gateway** | Authentication, rate limiting, routing | Kong, AWS API Gateway | Horizontal |
| **Post Service** | Tweet validation, ID generation, persistence | Go, Java, Kotlin (stateless) | Horizontal |
| **Timeline Service** | Timeline retrieval, merging, ranking | Go, Rust, Java (stateless) | Horizontal |
| **Kafka** | Message queue for async fanout | Apache Kafka | Horizontal (partitioned) |
| **Fanout Workers** | Distribute tweets to followers' timelines | Go, Java (consumer groups) | Horizontal (scale workers) |
| **Timeline Cache** | Hot timelines for active users | Redis Cluster, KeyDB | Horizontal (consistent hashing) |
| **UserTimeline DB** | Persistent timeline storage (denormalized) | Cassandra, ScyllaDB | Horizontal (sharding) |
| **Tweet Storage** | Source of truth for tweets | PostgreSQL (sharded) | Horizontal (sharding) |
| **Follower Graph** | Social graph (who follows whom) | Neo4j, PostgreSQL (sharded) | Horizontal |

---

## 3. Data Model

### Tweets Table (PostgreSQL)

| Field | Data Type | Notes |
|-------|-----------|-------|
| **tweet_id** | $\text{BIGINT}$ | Primary Key (Snowflake ID) |
| **user_id** | $\text{BIGINT}$ | Author, indexed |
| **content** | $\text{VARCHAR}(\text{280})$ | Tweet text |
| **created_at** | $\text{TIMESTAMP}$ | Creation timestamp |
| **like_count** | $\text{INT}$ | Denormalized counter |
| **retweet_count** | $\text{INT}$ | Denormalized counter |

**Sharding:** By `tweet_id` for time-based range queries.

### UserTimeline Table (Cassandra)

```sql
CREATE TABLE user_timeline (
    user_id BIGINT,          -- Partition key
    tweet_id BIGINT,         -- Clustering key (descending)
    tweet_data BLOB,         -- Denormalized
    PRIMARY KEY (user_id, tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);
```

**Why Denormalize?** Avoids joins. Timeline query becomes simple key lookup.

---

## 4. Fanout Strategies

> 📊 **See detailed sequence diagrams:** [Sequence Diagrams](./sequence-diagrams.md)

### Strategy 1: Fanout-on-Write (Push Model)

**How:** When tweet is posted, immediately push to all followers' timelines.

**Pros:** ✅ Fast reads (pre-computed), ✅ Low read latency
**Cons:** ❌ Expensive writes, ❌ Celebrity problem (millions of writes)

**Use For:** Normal users with $< 10,000$ followers.

*See [pseudocode.md::fanout_on_write()](pseudocode.md) for implementation.*

### Strategy 2: Fanout-on-Read (Pull Model)

**How:** When user requests timeline, fetch and merge tweets from all followees on-the-fly.

**Pros:** ✅ Cheap writes, ✅ No celebrity problem
**Cons:** ❌ Slow reads, ❌ Complex merge logic

**Use For:** Celebrity accounts with $> 10,000$ followers.

*See [pseudocode.md::fanout_on_read()](pseudocode.md) for implementation.*

### Strategy 3: Hybrid Fanout (Recommended) ✅

**How:**
- Push model for normal users
- Pull model for celebrities
- Timeline reads merge both: pre-computed + on-demand celebrity tweets

**Best of Both Worlds:** Balances write cost and read performance.

*See [pseudocode.md::hybrid_fanout()](pseudocode.md) for implementation.*

---

## 5. Write Path: Posting a Tweet

**Sequence:**

1. Client → POST `/tweet`
2. API Gateway → Authenticate, rate limit
3. Post Service:
   - Validate content
   - Generate Snowflake `tweet_id`
   - Save to PostgreSQL
   - Publish to Kafka (`new_tweets` topic)
   - Return success (immediately, without waiting)
4. Fanout Workers (async):
   - Consume from Kafka
   - Fetch followers from Follower Graph
   - Insert into UserTimeline (Cassandra)
   - Update Timeline Cache (Redis)

**Latency:** $< 100$ ms for API response, $< 10$ seconds for fanout completion.

**Why Kafka?** Decouples Post Service from Fanout Workers. Absorbs write spikes during viral events.

---

## 6. Read Path: Loading Timeline

**Sequence:**

1. Client → GET `/timeline`
2. Timeline Service:
   - Check Redis Cache
   - **Cache Hit** → Return immediately ($< 10$ ms)
   - **Cache Miss** → Query Cassandra
   - Load recent tweets, populate cache, return ($< 50$ ms)

**Hybrid Model Enhancement:**
- Fetch pre-computed timeline from cache/Cassandra
- For each celebrity followed, fetch their recent tweets
- Merge both, sort by timestamp, return top 50

---

## 7. Key Bottlenecks and Solutions

### Bottleneck 1: Celebrity Problem

**Problem:** Celebrity posts require millions of timeline writes.

**Solution:** Hybrid fanout - use pull model for celebrities, don't fanout.

### Bottleneck 2: Cache Hotspots

**Problem:** Popular users' timelines requested millions of times/sec.

**Solution:** Consistent hashing to distribute keys, multi-master Redis cluster.

### Bottleneck 3: Storage Growth

**Problem:** $73$ TB/year for tweets, more for timelines.

**Solution:** Archive old tweets to cold storage, prune timelines to 10,000 recent tweets.

### Bottleneck 4: Kafka Throughput

**Problem:** $50,000$ messages/sec during viral events.

**Solution:** Partition Kafka topic, batch writes, scale Kafka cluster.

---

## 8. Common Anti-Patterns

### ❌ Anti-Pattern 1: Fanout to Inactive Users

Wasting writes on users who haven't logged in for months.

**Solution:** Only fanout to active users (logged in last 30 days).

*See [pseudocode.md::activity_based_fanout()](pseudocode.md) for implementation.*

### ❌ Anti-Pattern 2: Synchronous Fanout

Blocking client while fanning out to thousands of followers.

**Solution:** Use Kafka for async fanout, return immediately to client.

### ❌ Anti-Pattern 3: No Cache Invalidation

Stale tweets remain in cache after deletion.

**Solution:** Set TTL on cache entries (1 hour), active invalidation on delete/edit.

### ❌ Anti-Pattern 4: Storing Entire Tweet in Timeline

Duplicating user profile data millions of times.

**Solution:** Store only `tweet_id` + minimal metadata. Batch-fetch full tweet details on read.

*See [pseudocode.md::load_timeline_with_details()](pseudocode.md) for implementation.*

---

## 9. Alternative Approaches (Not Chosen)

### Pure Fanout-on-Read (Pull Model)

**Pros:** Simple writes, no celebrity problem
**Cons:** Very slow reads ($> 1$ second)

**Why Not Chosen:** Read-heavy workload (50:1 ratio). Read latency is critical.

### NoSQL-Only (No RDBMS)

**Pros:** Easier horizontal scaling, simpler ops
**Cons:** Weaker consistency, complex queries

**Why Not Chosen:** Tweets require strong consistency (ACID). PostgreSQL + Cassandra is best of both worlds.

### Pure Fanout-on-Write (No Hybrid)

**Pros:** Simplest implementation, fastest reads
**Cons:** Crushes system when celebrities post

**Why Not Chosen:** Does not scale for celebrity users with millions of followers.

---

## 10. Technology Choices

| Component | Choice | Alternatives | Why Chosen? |
|-----------|--------|-------------|-------------|
| **Timeline Storage** | Cassandra | DynamoDB, HBase | High-throughput writes, cheap storage, natural horizontal scaling |
| **Timeline Cache** | Redis | Memcached | Rich data structures, persistence options, Lua scripting |
| **Message Queue** | Kafka | RabbitMQ, SQS | High throughput, durability, replay capability |
| **Tweet Storage** | PostgreSQL | MySQL, CockroachDB | ACID guarantees, mature ecosystem |
| **Follower Graph** | Neo4j / PostgreSQL | Graph DB optimal, but RDBMS acceptable with proper indexing |

*See [this-over-that.md](this-over-that.md) for detailed comparisons.*

---

## 11. Monitoring

**Critical Metrics:**
- Timeline Read Latency (P99): $< 200$ ms
- Tweet Write Latency (P99): $< 100$ ms
- Fanout Lag: $< 10$ seconds
- Cache Hit Rate: $> 90\%$
- Kafka Consumer Lag: $< 1,000$ messages

---

## 12. Trade-offs Summary

### What We Gained

✅ Low read latency ($< 10$ ms cache hits)
✅ High write throughput (Kafka absorbs spikes)
✅ Handles celebrity problem (hybrid fanout)
✅ Horizontal scaling (Cassandra, Kafka)

### What We Sacrificed

❌ Eventual consistency (timelines lag by seconds)
❌ Storage cost ($\sim 20$ TB denormalized)
❌ System complexity (hybrid fanout harder to debug)
❌ Operational overhead (multiple DB types)

---

## 13. Real-World Examples

**Twitter:** Hybrid fanout, Redis + MySQL + Manhattan DB, Kafka
**Facebook:** Mostly pull with ML ranking, TAO graph store
**Instagram:** Hybrid fanout optimized for media, aggressive CDN caching

---

## 14. References

- [Asynchronous Communication](../../02-components/2.3.1-asynchronous-communication.md)
- [Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)
- [NoSQL Deep Dive](../../02-components/2.1.2-no-sql-deep-dive.md)
- [Consistent Hashing](../../02-components/2.2.2-consistent-hashing.md)
- [CAP Theorem](../../01-principles/1.1.1-cap-theorem.md)

---

For complete details, algorithms, and trade-off analysis, see the **[Full Design Document](3.2.1-design-twitter-timeline.md)**.

