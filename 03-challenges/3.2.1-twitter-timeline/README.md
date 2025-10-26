# 3.2.1 Design a Twitter/X Timeline (Microblogging Feed)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions. 
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, fanout strategies, data flow, and scaling approaches
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows for posting tweets, timeline loading, fanout processes, and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices and trade-offs
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for fanout strategies, timeline merging, and activity-based fanout

---

## Problem Statement

Design a scalable microblogging timeline system (like Twitter/X) that allows users to post short messages (tweets) and view a personalized feed containing tweets from people they follow. The system must handle 500 million monthly active users, process 200 million tweets per day, serve 10 billion timeline loads per day with sub-200ms latency, and efficiently solve the "Fanout Problem" — distributing one tweet to millions of followers' timelines in near-real-time.

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

> 📚 **Note:** For detailed pseudocode implementations of anti-patterns and their solutions, see **[3.2.1-design-twitter-timeline.md](./3.2.1-design-twitter-timeline.md#8-common-anti-patterns)** and **[pseudocode.md](./pseudocode.md)**.

Below are high-level descriptions of common mistakes and their solutions:

### Anti-Pattern 1: Fanout to Inactive Users

**Problem:**

❌ Wasting writes on users who haven't logged in for months

```
// BAD: Fanout to all followers blindly
function fanout_tweet(tweet_id, author_id):
  followers = get_all_followers(author_id)  // 5,000 followers
  
  for follower_id in followers:
    cassandra.INSERT_INTO("user_timeline", {
      user_id: follower_id,
      tweet_id: tweet_id,
      timestamp: now()
    })

Problem:
- User last logged in 6 months ago
- Still receiving 100+ tweets per day in timeline
- Wasting Cassandra writes (expensive!)
- 50% of fanout writes are to inactive users
```

**Solution:** ✅ Activity-based fanout - only fanout to active users

```
// GOOD: Fanout only to active users
function fanout_tweet_smart(tweet_id, author_id):
  followers = get_all_followers(author_id)
  active_followers = filter_active_users(followers, days=30)
  
  for follower_id in active_followers:
    cassandra.INSERT_INTO("user_timeline", {
      user_id: follower_id,
      tweet_id: tweet_id,
      timestamp: now()
    })
  
  // Inactive users use fanout-on-read when they return
  
function filter_active_users(user_ids, days):
  active_users = []
  for user_id in user_ids:
    last_active = redis.GET("last_active:" + user_id)
    if (now() - last_active) < (days * 86400):
      active_users.append(user_id)
  return active_users

Benefits:
- 50% reduction in Cassandra writes
- Save $5k/month on database costs
- Inactive users still see tweets when they return (via fanout-on-read)
```

*See `pseudocode.md::activity_based_fanout()` for implementation*

---

### Anti-Pattern 2: Synchronous Fanout (Blocking Client)

**Problem:** ❌ Blocking user while fanning out to thousands of followers

```
// BAD: Synchronous fanout
function post_tweet(user_id, content):
  // 1. Save tweet
  tweet_id = save_tweet_to_database(user_id, content)
  
  // 2. Fanout (BLOCKS for 10+ seconds!) ❌
  followers = get_followers(user_id)  // 5,000 followers
  for follower_id in followers:
    cassandra.INSERT(follower_id, tweet_id)
  
  // 3. Return to client after 10 seconds
  return {tweet_id: tweet_id, status: "posted"}

User Experience:
- Clicks "Post Tweet" button
- Waits 10-15 seconds staring at loading spinner 😞
- Finally sees success message
- Terrible UX!
```

**Solution:** ✅ Async fanout with Kafka

```
// GOOD: Async fanout
function post_tweet(user_id, content):
  // 1. Save tweet (fast: 20ms)
  tweet_id = save_tweet_to_database(user_id, content)
  
  // 2. Publish to Kafka (non-blocking: 10ms)
  kafka.publish("fanout_events", {
    tweet_id: tweet_id,
    author_id: user_id,
    timestamp: now()
  })
  
  // 3. Return immediately (<50ms total!)
  return {tweet_id: tweet_id, status: "posted"}

// Separate Fanout Worker (async)
fanout_worker:
  for event in kafka.consume("fanout_events"):
    followers = get_followers(event.author_id)
    for follower_id in followers:
      cassandra.INSERT(follower_id, event.tweet_id)
    // Fanout happens in background (10 seconds)

User Experience:
- Clicks "Post Tweet"
- Sees success message immediately (<50ms)
- Timeline updates in background (10 seconds)
- Happy user! 😊
```

**Benefits:**
- 200× faster user response (50ms vs 10s)
- Can handle traffic spikes (Kafka buffers)
- Fanout failures don't affect user experience

*See `pseudocode.md::async_fanout()` for implementation*

---

### Anti-Pattern 3: No Cache Invalidation (Stale Data)

**Problem:** ❌ Stale tweets remain in cache after deletion/edit

```
// Scenario: User deletes a tweet
1. User posts tweet (tweet_id=123)
2. Tweet cached in Redis: cache["timeline:user456"] = [123, 122, 121, ...]
3. User deletes tweet 123
4. Tweet deleted from Cassandra ✅
5. Cache still contains tweet 123 ❌
6. Other users still see deleted tweet for 1 hour (cache TTL)!

Problem:
- Deleted content visible
- Privacy violation
- Regulatory issues (GDPR right to erasure)
```

**Solution:** ✅ Active cache invalidation + TTL

```
// GOOD: Multi-layer invalidation
function delete_tweet(tweet_id):
  // 1. Delete from database
  cassandra.DELETE_FROM("tweets", where={tweet_id: tweet_id})
  
  // 2. Get all users who have this tweet in timeline
  followers = get_users_with_tweet_in_timeline(tweet_id)
  
  // 3. Invalidate cache for each affected user
  for follower_id in followers:
    redis.DEL("timeline:" + follower_id)
  
  // 4. Publish cache invalidation event
  kafka.publish("cache_invalidation", {
    tweet_id: tweet_id,
    action: "DELETE"
  })
  
  return SUCCESS

// Cache Invalidation Worker
cache_worker:
  for event in kafka.consume("cache_invalidation"):
    if event.action == "DELETE":
      // Invalidate all timelines containing this tweet
      invalidate_tweet_from_all_caches(event.tweet_id)

// Fallback: TTL
redis.SETEX("timeline:" + user_id, ttl=3600, timeline_data)
// Even if invalidation fails, cache expires in 1 hour
```

**Trade-off:**
- Adds complexity (invalidation logic)
- Increases cache misses after delete
- **Worth it:** Privacy compliance, correct behavior

*See `pseudocode.md::invalidate_cache()` for implementation*

---

### Anti-Pattern 4: Storing Entire Tweet in Timeline (Data Duplication)

> 📊 **See optimization diagram:** [Timeline Storage Optimization](./hld-diagram.md#timeline-storage-optimization)

**Problem:** ❌ Duplicating user profile data millions of times

```
// BAD: Store full tweet in every follower's timeline
Cassandra user_timeline:
user_id=follower1: [
  {
    tweet_id: 123,
    author_id: celebrity_user,
    author_name: "Celebrity Name",
    author_avatar: "https://cdn.example.com/avatars/celebrity.jpg",
    content: "Hello world!",
    like_count: 50000,
    retweet_count: 10000,
    created_at: "2024-01-01T12:00:00Z"
  },
  // ... 999 more tweets
]

Storage Calculation:
- Celebrity has 5M followers
- Each tweet = 500 bytes (full data)
- Storage per tweet: 5M × 500 bytes = 2.5 GB
- Celebrity posts 100 tweets/day
- Daily storage: 250 GB (just for one celebrity!)
- Annual cost: $10k/year per celebrity ❌
```

**Solution:** ✅ Store only tweet_id, fetch details on read

```
// GOOD: Store only tweet_id (denormalized pointers)
Cassandra user_timeline:
user_id=follower1: [
  {tweet_id: 123, timestamp: "2024-01-01T12:00:00Z"},
  {tweet_id: 122, timestamp: "2024-01-01T11:50:00Z"},
  // ... 998 more (only IDs)
]

Storage Calculation:
- Each entry = 16 bytes (tweet_id: 8 bytes + timestamp: 8 bytes)
- Storage per tweet: 5M × 16 bytes = 80 MB
- Celebrity posts 100 tweets/day
- Daily storage: 8 GB (96% reduction!)
- Annual cost: $400/year per celebrity ✅

// Read Path: Batch fetch tweet details
function load_timeline(user_id):
  // 1. Get timeline (tweet IDs)
  timeline_entries = cassandra.QUERY("user_timeline", user_id, limit=50)
  tweet_ids = [entry.tweet_id for entry in timeline_entries]
  
  // 2. Batch fetch full tweet details
  tweets = batch_fetch_tweets(tweet_ids)  // Single DB query
  
  // 3. Hydrate timeline with details
  timeline = []
  for tweet_id in tweet_ids:
    tweet = tweets[tweet_id]
    timeline.append({
      tweet_id: tweet_id,
      author_name: tweet.author_name,
      content: tweet.content,
      like_count: tweet.like_count,
      created_at: tweet.created_at
    })
  
  return timeline

function batch_fetch_tweets(tweet_ids):
  // Single query for 50 tweets
  return postgresql.SELECT_FROM("tweets", where={tweet_id IN tweet_ids})
```

**Benefits:**
- 96% storage reduction (500 bytes → 16 bytes per entry)
- Save $9.6k/year per celebrity
- Easy to update tweet data (like_count) without touching timelines

**Trade-off:**
- Adds 1 extra DB query on read (batch fetch)
- Acceptable: 1ms for batch query of 50 tweets

*See `pseudocode.md::load_timeline_with_details()` for implementation*

---

### Anti-Pattern 5: Not Handling Deleted Users

**Problem:** ❌ Timeline contains tweets from deleted users

```
// Scenario
1. User A follows User B
2. User B posts 100 tweets
3. User A's timeline contains all 100 tweets
4. User B deletes their account
5. User A's timeline still shows B's tweets ❌
6. Clicking on tweet → 404 error (user not found)

Problem:
- Broken user experience
- Privacy violation (deleted user data still visible)
- GDPR violation (right to be forgotten)
```

**Solution:** ✅ Cascade delete user content

```
// GOOD: Clean up when user is deleted
function delete_user(user_id):
  // 1. Delete user account
  postgresql.DELETE_FROM("users", where={user_id: user_id})
  
  // 2. Delete all user's tweets
  tweet_ids = get_all_tweets_by_user(user_id)
  for tweet_id in tweet_ids:
    delete_tweet(tweet_id)
  
  // 3. Remove user from all timelines
  kafka.publish("user_deleted", {
    user_id: user_id,
    tweet_ids: tweet_ids
  })
  
  return SUCCESS

// Timeline Cleanup Worker
cleanup_worker:
  for event in kafka.consume("user_deleted"):
    // Remove deleted user's tweets from all timelines
    for tweet_id in event.tweet_ids:
      remove_tweet_from_all_timelines(tweet_id)

function remove_tweet_from_all_timelines(tweet_id):
  // Find all users who have this tweet
  affected_users = cassandra.QUERY_USERS_WITH_TWEET(tweet_id)
  
  for user_id in affected_users:
    cassandra.DELETE_FROM("user_timeline", where={
      user_id: user_id,
      tweet_id: tweet_id
    })
    
    // Invalidate cache
    redis.DEL("timeline:" + user_id)
```

**Trade-off:**
- User deletion takes longer (background cleanup)
- Acceptable: User rarely deletes account
- **Worth it:** GDPR compliance, clean data

---

### Anti-Pattern 6: No Rate Limiting on Write Path

**Problem:** ❌ Spammers can DOS the system with rapid tweets

```
// BAD: No rate limiting
function post_tweet(user_id, content):
  save_tweet(user_id, content)
  kafka.publish("fanout_events", {author_id: user_id})
  return SUCCESS

Attack Scenario:
- Spammer posts 1,000 tweets per second
- Each tweet triggers fanout to 1,000 followers
- 1,000 tweets × 1,000 followers = 1M Cassandra writes/sec
- Database overwhelmed → cascading failure ❌
```

**Solution:** ✅ Multi-layer rate limiting

```
// GOOD: Rate limit at multiple levels
function post_tweet(user_id, content):
  // 1. Per-user rate limit
  if not check_user_rate_limit(user_id):
    return ERROR("Rate limit exceeded: 10 tweets per minute")
  
  // 2. Global rate limit (protect system)
  if not check_global_rate_limit():
    return ERROR("System under heavy load, try again later")
  
  // 3. Save tweet
  tweet_id = save_tweet(user_id, content)
  
  // 4. Publish to Kafka
  kafka.publish("fanout_events", {author_id: user_id, tweet_id: tweet_id})
  
  return SUCCESS

function check_user_rate_limit(user_id):
  key = "rate_limit:tweets:" + user_id
  count = redis.INCR(key)
  redis.EXPIRE(key, 60)  // 1-minute window
  
  return count <= 10  // Max 10 tweets per minute

function check_global_rate_limit():
  key = "rate_limit:global:tweets"
  count = redis.INCR(key)
  redis.EXPIRE(key, 1)  // 1-second window
  
  return count <= 5000  // Max 5,000 tweets/sec globally
```

**Rate Limits:**
```
Per-User Limits:
- Regular user: 10 tweets/minute, 100 tweets/hour
- Verified user: 50 tweets/minute, 500 tweets/hour
- API client: 300 tweets/minute (with API key)

Global Limits:
- Max 5,000 tweets/sec (protects backend)
- If exceeded: Return 429 (Too Many Requests)
```

*See `pseudocode.md::check_rate_limit()` for implementation*

---

### Anti-Pattern 7: Unbounded Timeline Growth

**Problem:** ❌ Timeline grows infinitely, slowing down queries

```
// BAD: Store all tweets forever
Cassandra user_timeline:
user_id=active_user: [
  {tweet_id: 999999, timestamp: "2024-01-01"},
  {tweet_id: 999998, timestamp: "2024-01-01"},
  {tweet_id: 999997, timestamp: "2024-01-01"},
  // ... 50,000 tweets (5 years of following celebrities)
]

Problem:
- Timeline query: SELECT * FROM user_timeline WHERE user_id=? LIMIT 50
- Cassandra must scan 50,000 rows to find latest 50
- Query latency: 500ms (too slow!) ❌
- User only sees first 50 tweets anyway
```

**Solution:** ✅ Limit timeline size, prune old entries

```
// GOOD: Cap timeline at 10,000 most recent tweets
function fanout_tweet_with_pruning(follower_id, tweet_id):
  // 1. Insert new tweet
  cassandra.INSERT_INTO("user_timeline", {
    user_id: follower_id,
    tweet_id: tweet_id,
    timestamp: now()
  })
  
  // 2. Count timeline size
  count = cassandra.COUNT("user_timeline", where={user_id: follower_id})
  
  // 3. Prune if over limit
  if count > 10000:
    // Delete oldest 1,000 tweets
    oldest_tweets = cassandra.QUERY("user_timeline",
      where={user_id: follower_id},
      order_by="timestamp ASC",
      limit=1000
    )
    
    for tweet in oldest_tweets:
      cassandra.DELETE_FROM("user_timeline", where={
        user_id: follower_id,
        tweet_id: tweet.tweet_id
      })

// Alternative: Background pruning job (less intrusive)
pruning_job (runs daily):
  for each user in all_users:
    count = cassandra.COUNT("user_timeline", user_id=user)
    if count > 10000:
      prune_old_tweets(user, keep_latest=10000)
```

**Benefits:**
- Query latency: 500ms → 50ms (10× faster)
- Storage: 50k tweets → 10k tweets per user (80% reduction)
- User rarely scrolls past 1,000 tweets anyway

**Trade-off:**
- Very old tweets not in timeline (acceptable)
- User can still search for old tweets via search API

---

### Anti-Pattern 8: Not Optimizing Celebrity Fanout

**Problem:** ❌ Celebrities with millions of followers cause fanout lag

```
// BAD: Treat all users the same
function fanout_tweet(tweet_id, author_id):
  followers = get_all_followers(author_id)
  
  for follower_id in followers:
    cassandra.INSERT(follower_id, tweet_id)

Celebrity (5M followers):
- Fanout time: 5M writes × 10ms = 50,000 seconds = 14 hours! ❌
- Timeline lag: Followers don't see tweet for hours
- Kafka consumer lag grows to millions of messages
```

**Solution:** ✅ Hybrid fanout (push for normal users, pull for celebrities)

```
// GOOD: Adaptive fanout strategy
function fanout_tweet_adaptive(tweet_id, author_id):
  follower_count = get_follower_count(author_id)
  
  if follower_count < 10000:
    // Push model: Fanout to all followers (fast)
    fanout_push(tweet_id, author_id)
  
  elif follower_count < 1000000:
    // Hybrid: Fanout to active followers only
    active_followers = get_active_followers(author_id, days=7)
    fanout_push(tweet_id, active_followers)
  
  else:
    // Pull model: Don't fanout, mark as celebrity
    mark_as_celebrity_tweet(tweet_id, author_id)
    // Followers will fetch on-demand when loading timeline

// Timeline Read (Hybrid)
function load_timeline_hybrid(user_id):
  // 1. Get pre-computed timeline (push model)
  timeline = cassandra.QUERY("user_timeline", user_id, limit=50)
  
  // 2. Get list of celebrities user follows
  celebrities = get_followed_celebrities(user_id)
  
  // 3. Fetch latest tweets from celebrities (pull model)
  for celebrity_id in celebrities:
    recent_tweets = get_recent_tweets(celebrity_id, limit=10)
    timeline.extend(recent_tweets)
  
  // 4. Merge and sort by timestamp
  timeline = sort_by_timestamp(timeline)
  
  return timeline[:50]  // Return top 50
```

**Results:**
```
Normal User (1,000 followers):
- Fanout time: 10 seconds
- Timeline load: <10ms (cache hit)

Celebrity (5M followers):
- Fanout time: 0 seconds (skip fanout)
- Timeline load: ~50ms (fetch + merge)
- Acceptable: Followers willing to wait 50ms for celebrity content
```

*See `pseudocode.md::hybrid_fanout()` for implementation*

---

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

## 11.5. Cost Analysis (AWS Example)

**Assumptions:**
- 200M tweets/day (2,300 writes/sec)
- 10B timeline loads/day (115,740 reads/sec)
- 100M daily active users
- 73 TB/year tweet storage

| Component                      | Specification                        | Monthly Cost        |
|--------------------------------|--------------------------------------|---------------------|
| **Post Service (EC2)**         | 10× t3.large (2 vCPU, 8 GB)         | $680                |
| **Timeline Service (EC2)**     | 50× t3.xlarge (4 vCPU, 16 GB)       | $6,800              |
| **Kafka Cluster**              | 6× kafka.m5.2xlarge                  | $4,368              |
| **Fanout Workers (EC2)**       | 30× t3.large (2 vCPU, 8 GB)         | $2,040              |
| **Redis Cluster (Timelines)**  | cache.r5.12xlarge (360 GB RAM)       | $2,737              |
| **Cassandra Cluster (Timelines)** | 12× i3.2xlarge (8 vCPU, 61 GB)   | $9,576              |
| **PostgreSQL (Tweets)**        | db.r5.4xlarge (16 vCPU, 128 GB)      | $1,456              |
| **Application Load Balancer**  | Standard ALB                         | $20                 |
| **CloudWatch Monitoring**      | Logs + Metrics + Dashboards          | $300                |
| **VPC, NAT Gateway**           | Multi-AZ setup                       | $90                 |
| **Data Transfer**              | 20 TB/month egress                   | $1,800              |
| **S3 (Media Storage)**         | 100 TB storage + requests            | $2,300              |
| **Total**                      |                                      | **~$32,167/month**  |

**Cost Breakdown:**
```
Per Tweet Cost:
- Write (with fanout): $0.0016 per tweet
- Read (timeline load): $0.0000028 per load

Daily Costs:
- Tweets: 200M × $0.0016 = $320/day
- Timeline loads: 10B × $0.0000028 = $28/day
```

**Cost Optimization:**

| Optimization                     | Savings       | Notes                                      |
|----------------------------------|---------------|--------------------------------------------|
| **Reserved Instances (1-year)**  | -40% EC2      | Save $3,800/month on EC2                   |
| **Spot Instances for Workers**   | -70% workers  | Use Spot for fanout workers → Save $1,430/month |
| **Activity-Based Fanout**        | -50% Cassandra writes | Save $4,800/month           |
| **Cassandra on Local SSD**       | -30% storage  | Use i3 instances vs EBS → Save $2,900/month|
| **CDN for Media**                | -80% data transfer | CloudFront → Save $1,440/month       |
| **Aggressive Cache TTL**         | +20% cache hits | Reduce DB load → Save $1,500/month    |

**Optimized Cost: ~$16,300/month** (~49% savings)

**Cost per User:**
- Current: $32,167 / 100M users = **$0.00032 per user/month**
- Optimized: $16,300 / 100M users = **$0.00016 per user/month**
- **Industry Benchmark:** $0.0002 per user/month (we're 20% cheaper!)

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

## Summary

A Twitter/X timeline system requires careful balance between **read performance**, **write scalability**, and **handling the celebrity problem**:

**Key Design Choices:**

1. ✅ **Hybrid Fanout Strategy** - Push for normal users (<10k followers), pull for celebrities (>10k followers)
2. ✅ **Kafka for Async Fanout** to handle 50k message/sec spikes and decouple write path
3. ✅ **Cassandra for Timeline Storage** to handle high-throughput writes (115k writes/sec)
4. ✅ **Redis Cache for Hot Timelines** to achieve <10ms read latency
5. ✅ **PostgreSQL for Tweets** (ACID guarantees, source of truth)
6. ✅ **Activity-Based Fanout** to avoid wasting writes on inactive users (50% cost savings)
7. ✅ **Store Tweet IDs Only** in timelines (96% storage savings)
8. ✅ **Rate Limiting** on write path to prevent abuse

**Performance Characteristics:**

- **Read Latency:** <10ms (cache hit), ~50ms (cache miss)
- **Write Latency:** <50ms (async fanout, user doesn't wait)
- **Fanout Time:** ~10 seconds for 5,000 followers
- **Throughput:** 2,300 writes/sec, 115,740 reads/sec
- **Cache Hit Rate:** 90% (reduces DB load by 9×)
- **Fanout Lag:** <10 seconds (acceptable for social media)

**Critical Components:**

- **Kafka:** Must handle 50k messages/sec during viral events, 64+ partitions
- **Cassandra:** Write-optimized, 12+ nodes for timeline storage
- **Redis:** Hot timeline cache, 90%+ hit rate target
- **Fanout Workers:** 30+ workers for async fanout processing
- **Hybrid Logic:** Classify users (normal vs celebrity) dynamically

**Scalability Path:**

1. **0-1M users:** Single DB, synchronous fanout, simple cache
2. **1M-10M users:** Add Kafka, Redis cache, async fanout
3. **10M-100M users:** Cassandra for timelines, 10 fanout workers
4. **100M-500M users:** Hybrid fanout, 30 workers, sharded PostgreSQL, 12-node Cassandra
5. **500M+ users:** Multi-region deployment, 100+ workers, geo-partitioned data

**Common Pitfalls to Avoid:**

1. ❌ Fanout to inactive users (wastes 50% of writes)
2. ❌ Synchronous fanout (blocks user for 10+ seconds)
3. ❌ No cache invalidation (stale data, privacy violations)
4. ❌ Storing entire tweet in timeline (96% wasted storage)
5. ❌ Not handling deleted users (broken UX, GDPR violation)
6. ❌ No rate limiting (DDoS vulnerability)
7. ❌ Unbounded timeline growth (slow queries)
8. ❌ Not optimizing celebrity fanout (14-hour fanout time!)

**Recommended Stack:**

- **Load Balancer:** NGINX or AWS ALB (geo-routing, SSL termination)
- **API Gateway:** Kong or AWS API Gateway (auth, rate limiting)
- **Post/Timeline Service:** Go or Java (stateless, horizontally scalable)
- **Message Queue:** Apache Kafka (fault-tolerant, high throughput, replay capability)
- **Timeline Cache:** Redis Cluster (consistent hashing, multi-master)
- **Timeline Storage:** Cassandra or ScyllaDB (write-optimized, horizontal scaling)
- **Tweet Storage:** PostgreSQL (ACID, sharded by tweet_id)
- **Follower Graph:** Neo4j or PostgreSQL (efficient follow/unfollow operations)
- **Monitoring:** Prometheus + Grafana + PagerDuty + Distributed Tracing (Jaeger)

**Cost Efficiency:**

- Optimize with Reserved Instances (save 40% on EC2)
- Use Spot Instances for fanout workers (save 70%)
- Activity-based fanout (save 50% on Cassandra writes)
- Store tweet IDs only (save 96% on storage)
- CDN for media (save 80% on data transfer)
- **Optimized cost:** ~$16,300/month for 100M users ($0.00016 per user/month)

**Real-World Comparisons:**

| Feature                  | Our Design               | Twitter/X             | Facebook Feed        | Instagram       |
|--------------------------|--------------------------|----------------------|----------------------|-----------------|
| **Fanout Strategy**      | Hybrid (push+pull)       | Hybrid                | Pull with ML ranking | Hybrid (media-optimized) |
| **Scale**                | 100M DAU, 200M tweets/day | 237M DAU, 500M tweets/day | 2B DAU               | 1.4B DAU        |
| **Read Latency**         | <10ms (cache hit)        | ~20ms                 | ~50ms                | ~30ms           |
| **Timeline Storage**     | Cassandra                | Manhattan DB          | TAO (graph store)    | Cassandra       |
| **Message Queue**        | Kafka                    | Kafka                 | Scribe               | Kafka           |
| **Cache**                | Redis                    | Redis + Memcached     | Memcached            | Redis           |
| **Celebrity Problem**    | Pull model for celebrities | Hybrid fanout         | ML-based ranking     | Aggressive caching |

**Trade-Off Analysis:**

| Decision                      | Gain                               | Sacrifice                      |
|-------------------------------|------------------------------------|---------------------------------|
| Hybrid Fanout                 | Handles celebrity problem          | Complex implementation         |
| Async Fanout (Kafka)          | Fast user response (<50ms)         | Eventual consistency (~10s lag)|
| Cassandra for Timelines       | High write throughput              | No joins, eventual consistency |
| Redis Cache                   | <10ms read latency                 | Stale data (1-hour TTL)        |
| Store Tweet IDs Only          | 96% storage savings                | Extra DB query on read         |
| Activity-Based Fanout         | 50% cost savings                   | Inactive users see tweets late |
| Rate Limiting                 | Prevent abuse                      | Legitimate users may be blocked|
| Timeline Pruning (10k limit)  | Fast queries, 80% storage savings  | Very old tweets not in timeline|

This design provides a **production-ready, scalable blueprint** for building a Twitter-style timeline system that can handle hundreds of millions of users and billions of timeline loads per day while solving the celebrity fanout problem efficiently! 🚀

---

For complete details, algorithms, and trade-off analysis, see the **[Full Design Document](3.2.1-design-twitter-timeline.md)**.

