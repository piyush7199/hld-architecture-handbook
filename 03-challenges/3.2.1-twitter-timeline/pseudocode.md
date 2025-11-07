# Twitter Timeline - Pseudocode Implementations

This document contains detailed algorithm implementations for the Twitter Timeline system. The main challenge document
references these functions.

---

## Table of Contents

1. [Fanout Strategies](#fanout-strategies)
    - `fanout_on_write()` - Push model implementation
    - `fanout_on_read()` - Pull model implementation
    - `hybrid_fanout()` - Hybrid strategy
    - `activity_based_fanout()` - Optimized with activity filtering
2. [Timeline Operations](#timeline-operations)
    - `load_timeline()` - Load user timeline with cache
    - `load_timeline_with_details()` - Fetch full tweet data
    - `merge_timelines()` - Merge multiple sorted lists
3. [Write Path](#write-path)
    - `post_tweet()` - Complete tweet posting flow
    - `publish_to_kafka()` - Kafka event publishing
4. [Read Path](#read-path)
    - `get_timeline_cached()` - Cache-first timeline retrieval
    - `get_timeline_cassandra()` - Database fallback
5. [Celebrity Handling](#celebrity-handling)
    - `is_celebrity()` - Celebrity detection
    - `fetch_celebrity_tweets()` - Pull celebrity content
6. [Cache Operations](#cache-operations)
    - `populate_timeline_cache()` - Cache warming
    - `invalidate_timeline_cache()` - Cache invalidation
7. [Helper Functions](#helper-functions)
    - `get_followers()` - Fetch follower list
    - `batch_insert_timelines()` - Optimized batch writes

---

## Fanout Strategies

### fanout_on_write()

**Purpose:** Implement the push model - insert tweet into all followers' timelines immediately.

**Parameters:**

- `tweet_id`: Unique identifier for the tweet
- `author_user_id`: ID of the user who posted the tweet

**Returns:** None (async operation)

**Algorithm:**

```
function fanout_on_write(tweet_id, author_user_id):
    // Step 1: Fetch all followers of the author
    followers = get_followers(author_user_id)
    
    // Step 2: Fetch full tweet data
    tweet_data = get_tweet_from_db(tweet_id)
    
    // Step 3: Prepare timeline entry
    timeline_entry = {
        tweet_id: tweet_id,
        user_id: author_user_id,
        created_at: tweet_data.created_at,
        tweet_data: serialize(tweet_data)  // Denormalized
    }
    
    // Step 4: Insert into each follower's timeline (batch operation)
    batch_size = 1000
    for i = 0 to followers.length step batch_size:
        batch = followers[i:i+batch_size]
        
        // Prepare batch insert for Cassandra
        statements = []
        for follower_id in batch:
            statement = {
                query: "INSERT INTO user_timeline (user_id, tweet_id, tweet_data, created_at) VALUES (?, ?, ?, ?)",
                params: [follower_id, tweet_id, timeline_entry.tweet_data, timeline_entry.created_at]
            }
            statements.append(statement)
        
        // Execute batch insert
        cassandra.batch_execute(statements)
    
    // Step 5: Update Redis cache for active followers
    active_followers = filter_active_followers(followers, days=30)
    for follower_id in active_followers:
        redis.lpush("user:#{follower_id}:timeline", tweet_id)
        redis.ltrim("user:#{follower_id}:timeline", 0, 2000)  // Keep only recent 2000
        redis.expire("user:#{follower_id}:timeline", 3600)  // 1 hour TTL
    
    // Step 6: Log metrics
    log_metric("fanout.write.followers", followers.length)
    log_metric("fanout.write.duration", elapsed_time())
```

**Time Complexity:** $\text{O}(N)$ where $N$ = number of followers

**Example Usage:**

```
tweet_id = 123456789
author_user_id = 42

fanout_on_write(tweet_id, author_user_id)
// Result: Tweet inserted into 5,000 follower timelines (~5 seconds)
```

---

### fanout_on_read()

**Purpose:** Implement the pull model - fetch and merge tweets from all followees on-demand.

**Parameters:**

- `user_id`: ID of the user requesting their timeline
- `limit`: Number of tweets to return (default: 50)

**Returns:** Array of tweet objects sorted by created_at descending

**Algorithm:**

```
function fanout_on_read(user_id, limit=50):
    // Step 1: Fetch all users that this user follows
    followees = get_followees(user_id)
    
    if followees.length == 0:
        return []
    
    // Step 2: Fetch recent tweets from each followee (parallel)
    tweets_per_followee = 100  // Over-fetch to ensure we have enough after merging
    all_tweets = []
    
    // Parallel fetch (using thread pool or async I/O)
    futures = []
    for followee_id in followees:
        future = async_fetch_user_tweets(followee_id, limit=tweets_per_followee)
        futures.append(future)
    
    // Wait for all fetches to complete
    for future in futures:
        tweets = future.result()
        all_tweets.extend(tweets)
    
    // Step 3: Merge and sort all tweets by created_at
    sorted_tweets = merge_timelines(all_tweets, limit)
    
    return sorted_tweets


function async_fetch_user_tweets(user_id, limit):
    // Query PostgreSQL for user's recent tweets
    query = "SELECT * FROM tweets WHERE user_id = ? ORDER BY created_at DESC LIMIT ?"
    return db.execute(query, [user_id, limit])
```

**Time Complexity:** $\text{O}(N \log N)$ where $N$ = total tweets fetched

**Example Usage:**

```
user_id = 789
followees_count = 500

timeline = fanout_on_read(user_id, limit=50)
// Result: 500 parallel DB queries, merge 50K tweets, return top 50
// Duration: ~1-2 seconds (slow but no pre-computation needed)
```

---

### hybrid_fanout()

**Purpose:** Implement the hybrid strategy - push for normal users, pull for celebrities.

**Parameters:**

- `tweet_id`: Unique identifier for the tweet
- `author_user_id`: ID of the user who posted the tweet

**Returns:** None (async operation)

**Algorithm:**

```
function hybrid_fanout(tweet_id, author_user_id):
    // Step 1: Check if author is a celebrity
    follower_count = get_follower_count(author_user_id)
    
    if is_celebrity(follower_count):
        // Celebrity: Skip fanout, mark tweet for pull-based retrieval
        log_info("Celebrity tweet detected, skipping fanout", {
            tweet_id: tweet_id,
            author_id: author_user_id,
            follower_count: follower_count
        })
        
        // Optional: Cache celebrity's recent tweets for faster pull
        redis.lpush("celebrity:#{author_user_id}:tweets", tweet_id)
        redis.ltrim("celebrity:#{author_user_id}:tweets", 0, 100)
        redis.expire("celebrity:#{author_user_id}:tweets", 600)  // 10 min TTL
        
        return  // Skip fanout
    
    // Step 2: Normal user: Use fanout-on-write
    fanout_on_write(tweet_id, author_user_id)


function is_celebrity(follower_count):
    CELEBRITY_THRESHOLD = 10000  // Configurable via feature flag
    return follower_count > CELEBRITY_THRESHOLD


function get_follower_count(user_id):
    // Query follower graph for count
    count = follower_graph.query("MATCH (u:User {id: $user_id})<-[:FOLLOWS]-(f) RETURN count(f)", {user_id: user_id})
    return count
```

**Hybrid Timeline Read:**

```
function load_timeline_hybrid(user_id, limit=50):
    // Step 1: Load pre-computed timeline (from fanout-on-write)
    precomputed_timeline = get_timeline_cached(user_id, limit=limit * 2)  // Over-fetch
    
    // Step 2: Check if user follows any celebrities
    followees = get_followees(user_id)
    celebrities = filter(followees, is_celebrity)
    
    if celebrities.length == 0:
        // No celebrities followed, return pre-computed timeline
        return precomputed_timeline[:limit]
    
    // Step 3: Fetch recent tweets from celebrities (pull model)
    celebrity_tweets = []
    for celebrity_id in celebrities:
        // Check celebrity tweet cache first
        cached_tweets = redis.lrange("celebrity:#{celebrity_id}:tweets", 0, 10)
        
        if cached_tweets.length > 0:
            tweet_ids = cached_tweets
        else:
            // Fallback to database
            tweet_ids = db.query("SELECT tweet_id FROM tweets WHERE user_id = ? ORDER BY created_at DESC LIMIT 10", [celebrity_id])
        
        celebrity_tweets.extend(tweet_ids)
    
    // Step 4: Merge pre-computed timeline + celebrity tweets
    merged_timeline = merge_timelines(precomputed_timeline, celebrity_tweets, limit)
    
    return merged_timeline
```

**Benefits:**

- Normal users: Fast reads (<10ms cache hits)
- Celebrities: Avoids fanout explosion
- Hybrid reads: Acceptable latency (~30ms)

**Example Usage:**

```
// Post tweet from normal user
hybrid_fanout(tweet_id=123, author_user_id=42)  // Follower count: 5,000
→ Fanout to 5,000 timelines

// Post tweet from celebrity
hybrid_fanout(tweet_id=456, author_user_id=99)  // Follower count: 10M
→ Skip fanout, cache celebrity tweet

// Load timeline for user following both
timeline = load_timeline_hybrid(user_id=789, limit=50)
→ Merge pre-computed (normal users) + celebrity tweets
→ Return top 50
```

---

### activity_based_fanout()

**Purpose:** Optimize fanout by only pushing to active followers (logged in recently).

**Parameters:**

- `tweet_id`: Unique identifier for the tweet
- `author_user_id`: ID of the user who posted the tweet
- `activity_threshold_days`: Number of days to consider a follower active (default: 30)

**Returns:** Metrics about fanout operation

**Algorithm:**

```
function activity_based_fanout(tweet_id, author_user_id, activity_threshold_days=30):
    // Step 1: Fetch all followers
    all_followers = get_followers(author_user_id)
    
    // Step 2: Filter followers by activity
    // Query: Get followers who logged in within threshold
    cutoff_date = current_date() - activity_threshold_days
    
    active_followers = []
    inactive_followers = []
    
    // Batch query for efficiency
    batch_size = 10000
    for i = 0 to all_followers.length step batch_size:
        batch = all_followers[i:i+batch_size]
        
        query = "SELECT user_id FROM users WHERE user_id IN (?) AND last_login > ?"
        active_batch = db.execute(query, [batch, cutoff_date])
        
        active_followers.extend(active_batch)
        
        // Inactive followers = all - active
        inactive_batch = batch - active_batch
        inactive_followers.extend(inactive_batch)
    
    // Step 3: Fanout only to active followers
    fanout_on_write_to_users(tweet_id, author_user_id, active_followers)
    
    // Step 4: Log metrics
    metrics = {
        total_followers: all_followers.length,
        active_followers: active_followers.length,
        inactive_followers: inactive_followers.length,
        savings_percent: (inactive_followers.length / all_followers.length) * 100,
        duration: elapsed_time()
    }
    
    log_metric("fanout.activity_based", metrics)
    
    return metrics


function fanout_on_write_to_users(tweet_id, author_user_id, target_users):
    // Same as fanout_on_write but only for specified users
    tweet_data = get_tweet_from_db(tweet_id)
    
    batch_size = 1000
    for i = 0 to target_users.length step batch_size:
        batch = target_users[i:i+batch_size]
        
        statements = []
        for user_id in batch:
            statement = {
                query: "INSERT INTO user_timeline (user_id, tweet_id, tweet_data, created_at) VALUES (?, ?, ?, ?)",
                params: [user_id, tweet_id, serialize(tweet_data), tweet_data.created_at]
            }
            statements.append(statement)
        
        cassandra.batch_execute(statements)
```

**Handling Inactive Users:**

```
function load_timeline_for_inactive_user(user_id, limit=50):
    // When inactive user logs in, build their timeline on-demand
    
    // Step 1: Get last login time
    last_login = get_last_login_time(user_id)
    
    // Step 2: Get all followees
    followees = get_followees(user_id)
    
    // Step 3: Fetch tweets posted since last login
    all_tweets = []
    for followee_id in followees:
        query = "SELECT * FROM tweets WHERE user_id = ? AND created_at > ? ORDER BY created_at DESC LIMIT 100"
        tweets = db.execute(query, [followee_id, last_login])
        all_tweets.extend(tweets)
    
    // Step 4: Merge and return top N
    sorted_tweets = merge_timelines(all_tweets, limit)
    
    // Step 5: Populate cache for future fast loads
    populate_timeline_cache(user_id, sorted_tweets)
    
    return sorted_tweets
```

**Example Usage:**

```
// User with 100K followers posts
metrics = activity_based_fanout(tweet_id=123, author_user_id=42, activity_threshold_days=30)

// Result:
{
    total_followers: 100000,
    active_followers: 40000,  // 40% active
    inactive_followers: 60000,  // 60% inactive
    savings_percent: 60,
    duration: 8 seconds  // vs 20 seconds for full fanout
}

// Savings: 60K writes avoided, 60% faster fanout
```

---

## Timeline Operations

### load_timeline()

**Purpose:** Load user's timeline with cache-first strategy.

**Parameters:**

- `user_id`: ID of the user requesting timeline
- `limit`: Number of tweets to return
- `cursor`: Pagination cursor (last tweet_id seen)

**Returns:** Timeline object with tweets and pagination cursor

**Algorithm:**

```
function load_timeline(user_id, limit=50, cursor=null):
    // Step 1: Try Redis cache first
    cache_key = "user:#{user_id}:timeline"
    
    cached_tweet_ids = redis.lrange(cache_key, 0, limit - 1)
    
    if cached_tweet_ids.length >= limit:
        // Cache hit: Fetch full tweet data
        tweets = load_timeline_with_details(cached_tweet_ids)
        
        log_metric("timeline.cache_hit", 1)
        return {
            tweets: tweets,
            cursor: tweets[-1].tweet_id,
            source: "cache"
        }
    
    // Step 2: Cache miss: Query Cassandra
    log_metric("timeline.cache_miss", 1)
    
    query = "SELECT * FROM user_timeline WHERE user_id = ? LIMIT ?"
    params = [user_id, limit]
    
    if cursor != null:
        // Pagination: Fetch tweets after cursor
        query = "SELECT * FROM user_timeline WHERE user_id = ? AND tweet_id < ? LIMIT ?"
        params = [user_id, cursor, limit]
    
    timeline_entries = cassandra.execute(query, params)
    
    // Step 3: Populate cache (async)
    async_populate_cache(user_id, timeline_entries)
    
    // Step 4: Return timeline
    return {
        tweets: timeline_entries,
        cursor: timeline_entries[-1].tweet_id if timeline_entries.length > 0 else null,
        source: "cassandra"
    }


function async_populate_cache(user_id, timeline_entries):
    // Run in background thread
    cache_key = "user:#{user_id}:timeline"
    
    for entry in timeline_entries:
        redis.rpush(cache_key, entry.tweet_id)
    
    redis.expire(cache_key, 3600)  // 1 hour TTL
```

**Time Complexity:**

- Cache hit: $\text{O}(N)$ where $N$ = limit
- Cache miss: $\text{O}(N)$ + cache population overhead

---

### load_timeline_with_details()

**Purpose:** Fetch full tweet details for a list of tweet IDs (batch operation).

**Parameters:**

- `tweet_ids`: Array of tweet IDs

**Returns:** Array of full tweet objects

**Algorithm:**

```
function load_timeline_with_details(tweet_ids):
    // Step 1: Try to fetch from cache (batch)
    cache_keys = []
    for tweet_id in tweet_ids:
        cache_keys.append("tweet:#{tweet_id}")
    
    cached_tweets = redis.mget(cache_keys)
    
    // Step 2: Identify cache misses
    missing_tweet_ids = []
    result = []
    
    for i = 0 to tweet_ids.length:
        if cached_tweets[i] != null:
            result.append(deserialize(cached_tweets[i]))
        else:
            missing_tweet_ids.append(tweet_ids[i])
    
    // Step 3: Fetch missing tweets from database (batch)
    if missing_tweet_ids.length > 0:
        query = "SELECT * FROM tweets WHERE tweet_id IN (?)"
        db_tweets = db.execute(query, [missing_tweet_ids])
        
        // Cache the fetched tweets
        for tweet in db_tweets:
            redis.setex("tweet:#{tweet.tweet_id}", 3600, serialize(tweet))
            result.append(tweet)
    
    // Step 4: Sort by created_at (maintain timeline order)
    result.sort(key=lambda t: t.created_at, reverse=true)
    
    return result
```

**Optimization - Minimize Denormalization:**

Instead of storing entire tweet in timeline, store only tweet_id + minimal metadata:

```
// Lightweight timeline entry (100 bytes)
{
    tweet_id: 123456789,
    author_user_id: 42,
    created_at: "2024-01-15T10:30:00Z"
}

// Full tweet fetched on-demand (1 KB)
{
    tweet_id: 123456789,
    author_user_id: 42,
    author_username: "@user",
    author_profile_pic: "https://...",
    content: "Hello world!",
    created_at: "2024-01-15T10:30:00Z",
    like_count: 150,
    retweet_count: 25,
    media_urls: [...]
}

// Benefit: 10× storage savings in Cassandra
```

---

### merge_timelines()

**Purpose:** Merge multiple sorted lists of tweets into a single sorted list.

**Parameters:**

- `timeline_lists`: Array of sorted tweet arrays
- `limit`: Number of tweets to return

**Returns:** Merged sorted array of tweets

**Algorithm:**

```
function merge_timelines(timeline_lists, limit):
    // Use min-heap for efficient merging (N-way merge)
    // Each heap entry: (tweet, list_index, position_in_list)
    
    heap = MinHeap()
    
    // Step 1: Initialize heap with first tweet from each list
    for i = 0 to timeline_lists.length:
        if timeline_lists[i].length > 0:
            tweet = timeline_lists[i][0]
            heap.insert({
                tweet: tweet,
                list_index: i,
                position: 0,
                priority: tweet.created_at  // Sort by timestamp
            })
    
    // Step 2: Extract min (latest tweet) repeatedly
    result = []
    
    while result.length < limit and heap.size() > 0:
        // Extract tweet with latest timestamp
        entry = heap.extract_min()
        result.append(entry.tweet)
        
        // Add next tweet from same list
        next_position = entry.position + 1
        if next_position < timeline_lists[entry.list_index].length:
            next_tweet = timeline_lists[entry.list_index][next_position]
            heap.insert({
                tweet: next_tweet,
                list_index: entry.list_index,
                position: next_position,
                priority: next_tweet.created_at
            })
    
    return result
```

**Time Complexity:** $\text{O}(K \log N)$ where:

- $K$ = limit (number of tweets to return)
- $N$ = number of lists to merge

**Alternative (Simpler but less efficient):**

```
function merge_timelines_simple(timeline_lists, limit):
    // Combine all tweets
    all_tweets = []
    for list in timeline_lists:
        all_tweets.extend(list)
    
    // Sort by created_at descending
    all_tweets.sort(key=lambda t: t.created_at, reverse=true)
    
    // Return top N
    return all_tweets[:limit]
```

**Time Complexity:** $\text{O}(M \log M)$ where $M$ = total tweets

---

## Write Path

### post_tweet()

**Purpose:** Complete flow for posting a tweet.

**Parameters:**

- `user_id`: ID of the user posting
- `content`: Tweet text content
- `media_urls`: Optional array of media URLs

**Returns:** Tweet object with generated tweet_id

**Algorithm:**

```
function post_tweet(user_id, content, media_urls=[]):
    // Step 1: Validate input
    if content.length > 280:
        throw ValidationError("Tweet exceeds 280 characters")
    
    if is_spam(content):
        throw ValidationError("Spam detected")
    
    // Step 2: Check rate limit
    if not check_rate_limit(user_id):
        throw RateLimitError("Rate limit exceeded")
    
    // Step 3: Generate unique tweet ID (Snowflake)
    tweet_id = generate_snowflake_id()
    
    // Step 4: Create tweet object
    tweet = {
        tweet_id: tweet_id,
        user_id: user_id,
        content: content,
        media_urls: media_urls,
        created_at: current_timestamp(),
        status: "ACTIVE"
    }
    
    // Step 5: Save to PostgreSQL (source of truth)
    query = "INSERT INTO tweets (tweet_id, user_id, content, created_at, status) VALUES (?, ?, ?, ?, ?)"
    db.execute(query, [tweet.tweet_id, tweet.user_id, tweet.content, tweet.created_at, tweet.status])
    
    // Step 6: Publish to Kafka (async fanout)
    publish_to_kafka("new_tweets", {
        event_type: "tweet_posted",
        tweet_id: tweet.tweet_id,
        user_id: tweet.user_id,
        timestamp: tweet.created_at
    })
    
    // Step 7: Return immediately (don't wait for fanout)
    return tweet


function check_rate_limit(user_id):
    // Sliding window rate limit using Redis
    key = "rate_limit:#{user_id}:tweets:hour"
    
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, 3600)  // Set TTL on first increment
    
    limit = get_user_rate_limit(user_id)  // e.g., 300 for free tier
    
    return count <= limit


function is_spam(content):
    // Simple spam detection (production would use ML model)
    spam_keywords = ["buy now", "click here", "free money"]
    
    content_lower = content.lower()
    for keyword in spam_keywords:
        if keyword in content_lower:
            return true
    
    return false
```

**Example Usage:**

```
user_id = 42
content = "Hello, Twitter! This is my first tweet."

tweet = post_tweet(user_id, content)
// Returns immediately (~100ms)

// Background: Kafka consumer triggers fanout (~10 seconds)
```

---

### publish_to_kafka()

**Purpose:** Publish event to Kafka topic.

**Parameters:**

- `topic`: Kafka topic name
- `event`: Event data (will be JSON serialized)

**Returns:** Kafka offset

**Algorithm:**

```
function publish_to_kafka(topic, event):
    // Step 1: Serialize event
    message = json_serialize(event)
    
    // Step 2: Determine partition key (for ordering)
    partition_key = event.user_id.to_string()
    
    // Step 3: Publish to Kafka
    try:
        result = kafka_producer.send(
            topic=topic,
            key=partition_key,
            value=message,
            timeout_ms=5000
        )
        
        // Wait for acknowledgment (synchronous for reliability)
        offset = result.get()
        
        log_info("Published to Kafka", {
            topic: topic,
            partition: result.partition,
            offset: offset
        })
        
        return offset
        
    catch KafkaError as e:
        log_error("Kafka publish failed", e)
        throw e
```

---

## Celebrity Handling

### fetch_celebrity_tweets()

**Purpose:** Fetch recent tweets from celebrity users (for hybrid timeline).

**Parameters:**

- `celebrity_ids`: Array of celebrity user IDs
- `tweets_per_celebrity`: Number of tweets to fetch per celebrity

**Returns:** Array of tweets from all celebrities

**Algorithm:**

```
function fetch_celebrity_tweets(celebrity_ids, tweets_per_celebrity=10):
    all_celebrity_tweets = []
    
    for celebrity_id in celebrity_ids:
        // Step 1: Try cache first
        cache_key = "celebrity:#{celebrity_id}:tweets"
        cached_tweet_ids = redis.lrange(cache_key, 0, tweets_per_celebrity - 1)
        
        if cached_tweet_ids.length >= tweets_per_celebrity:
            // Cache hit
            tweets = load_tweets_by_ids(cached_tweet_ids)
            all_celebrity_tweets.extend(tweets)
        else:
            // Cache miss: Query database
            query = "SELECT * FROM tweets WHERE user_id = ? ORDER BY created_at DESC LIMIT ?"
            tweets = db.execute(query, [celebrity_id, tweets_per_celebrity])
            all_celebrity_tweets.extend(tweets)
            
            // Populate cache
            for tweet in tweets:
                redis.rpush(cache_key, tweet.tweet_id)
            redis.expire(cache_key, 600)  // 10 min TTL
    
    return all_celebrity_tweets
```

---

## Cache Operations

### populate_timeline_cache()

**Purpose:** Warm cache with user's timeline.

**Parameters:**

- `user_id`: User whose timeline to cache
- `tweets`: Array of tweets to cache

**Returns:** None

**Algorithm:**

```
function populate_timeline_cache(user_id, tweets):
    cache_key = "user:#{user_id}:timeline"
    
    // Delete existing cache
    redis.del(cache_key)
    
    // Push tweet IDs (most recent first)
    for tweet in tweets:
        redis.rpush(cache_key, tweet.tweet_id)
    
    // Set TTL
    redis.expire(cache_key, 3600)  // 1 hour
    
    // Also cache individual tweets
    for tweet in tweets:
        tweet_key = "tweet:#{tweet.tweet_id}"
        redis.setex(tweet_key, 3600, serialize(tweet))
```

---

### invalidate_timeline_cache()

**Purpose:** Invalidate cache when tweet is deleted.

**Parameters:**

- `tweet_id`: ID of deleted tweet
- `author_user_id`: ID of tweet author

**Returns:** Number of cache entries invalidated

**Algorithm:**

```
function invalidate_timeline_cache(tweet_id, author_user_id):
    // Step 1: Get all followers (who might have this tweet in cache)
    followers = get_followers(author_user_id)
    
    invalidated_count = 0
    
    // Step 2: Remove tweet from each follower's cache
    for follower_id in followers:
        cache_key = "user:#{follower_id}:timeline"
        
        // Remove tweet from list
        removed = redis.lrem(cache_key, 0, tweet_id)
        if removed > 0:
            invalidated_count++
    
    // Step 3: Delete tweet cache
    redis.del("tweet:#{tweet_id}")
    
    return invalidated_count
```

---

## Helper Functions

### get_followers()

**Purpose:** Fetch all followers of a user from the follower graph.

**Parameters:**

- `user_id`: User whose followers to fetch

**Returns:** Array of follower user IDs

**Algorithm:**

```
function get_followers(user_id):
    // Neo4j query
    query = "MATCH (u:User {id: $user_id})<-[:FOLLOWS]-(follower) RETURN follower.id"
    result = neo4j.execute(query, {user_id: user_id})
    
    follower_ids = []
    for record in result:
        follower_ids.append(record.follower_id)
    
    return follower_ids


// PostgreSQL alternative
function get_followers_postgres(user_id):
    query = "SELECT follower_id FROM followers WHERE followee_id = ?"
    result = db.execute(query, [user_id])
    
    follower_ids = []
    for row in result:
        follower_ids.append(row.follower_id)
    
    return follower_ids
```

---

### batch_insert_timelines()

**Purpose:** Optimized batch insert for fanout operations.

**Parameters:**

- `timeline_entries`: Array of {user_id, tweet_id, tweet_data} objects

**Returns:** Number of entries inserted

**Algorithm:**

```
function batch_insert_timelines(timeline_entries):
    batch_size = 1000
    total_inserted = 0
    
    for i = 0 to timeline_entries.length step batch_size:
        batch = timeline_entries[i:i+batch_size]
        
        // Prepare Cassandra batch
        statements = []
        for entry in batch:
            statement = {
                query: "INSERT INTO user_timeline (user_id, tweet_id, tweet_data, created_at) VALUES (?, ?, ?, ?)",
                params: [entry.user_id, entry.tweet_id, entry.tweet_data, entry.created_at]
            }
            statements.append(statement)
        
        // Execute batch with retry
        retry_count = 0
        max_retries = 3
        
        while retry_count < max_retries:
            try:
                cassandra.batch_execute(statements, consistency_level="QUORUM")
                total_inserted += batch.length
                break
            catch CassandraTimeout as e:
                retry_count++
                if retry_count >= max_retries:
                    log_error("Batch insert failed after retries", e)
                    throw e
                sleep(retry_count * 100)  // Exponential backoff
    
    return total_inserted
```