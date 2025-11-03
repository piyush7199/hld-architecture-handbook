# Instagram/Pinterest Feed - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made in designing the Instagram/Pinterest
Feed system, including trade-offs, alternatives considered, and rationale behind each choice.

---

## Table of Contents

1. [Fanout Strategy: Hybrid (Fanout-on-Write + Fanout-on-Read) Over Pure Fanout-on-Write](#1-fanout-strategy-hybrid-fanout-on-write--fanout-on-read-over-pure-fanout-on-write)
2. [Primary Database: PostgreSQL (Sharded) Over MongoDB](#2-primary-database-postgresql-sharded-over-mongodb)
3. [Timeline Cache: Redis Sorted Sets Over Memcached](#3-timeline-cache-redis-sorted-sets-over-memcached)
4. [Engagement Storage: Cassandra Over DynamoDB](#4-engagement-storage-cassandra-over-dynamodb)
5. [Media Upload: Pre-Signed URLs Over Direct Server Upload](#5-media-upload-pre-signed-urls-over-direct-server-upload)
6. [Image Processing: Async Queue Over Synchronous Processing](#6-image-processing-async-queue-over-synchronous-processing)
7. [Recommendation Engine: Offline Training + Online Serving Over Real-Time Training](#7-recommendation-engine-offline-training--online-serving-over-real-time-training)
8. [CDN Strategy: Multi-Tier CDN Over Direct Origin Serving](#8-cdn-strategy-multi-tier-cdn-over-direct-origin-serving)
9. [Feed Stitching: Server-Side Merging Over Client-Side Merging](#9-feed-stitching-server-side-merging-over-client-side-merging)
10. [Hot Key Mitigation: Distributed Counters Over Single Counter](#10-hot-key-mitigation-distributed-counters-over-single-counter)

---

## 1. Fanout Strategy: Hybrid (Fanout-on-Write + Fanout-on-Read) Over Pure Fanout-on-Write

### The Problem

When a user posts content, the system must decide how to update followers' timelines. Two extremes exist:

- **Fanout-on-Write**: Pre-compute and push posts to all followers' timelines immediately
- **Fanout-on-Read**: Compute timeline on-demand when user requests feed

For Instagram/Pinterest with billions of users and varying follower counts (1 to 500M), a pure strategy doesn't work:

- **Pure Fanout-on-Write**: Celebrities with 100M followers → 1 post generates 100M writes (write storm)
- **Pure Fanout-on-Read**: Every feed request queries database → 347k QPS overwhelms database

### Options Considered

| Approach                    | Pros                                                                | Cons                                                                         | Performance                                    | Cost                                  |
|-----------------------------|---------------------------------------------------------------------|------------------------------------------------------------------------------|------------------------------------------------|---------------------------------------|
| **Pure Fanout-on-Write**    | • Ultra-fast feed reads<br/>• Simple logic                          | • Unbounded writes for celebrities<br/>• Wasted effort if followers inactive | • Read: 50ms<br/>• Write: 1 post → 100M writes | • Extremely expensive for celebrities |
| **Pure Fanout-on-Read**     | • No write amplification<br/>• Always fresh                         | • Slow feed generation<br/>• High database load                              | • Read: 500ms+<br/>• Write: 1 post → 1 write   | • Expensive database capacity         |
| **Hybrid (This Over That)** | • Fast reads for most users<br/>• Controlled writes for celebrities | • Complex dual logic<br/>• Threshold tuning needed                           | • Read: 80-150ms<br/>• Write: Bounded          | • Balanced cost                       |

### Decision Made

**Hybrid Fanout Strategy** with 10K follower threshold:

- **< 10K followers**: Fanout-on-write (pre-compute timelines)
- **> 10K followers**: Fanout-on-read (query at read time)

### Rationale

1. **Write Scalability**: Prevents celebrity write storms
    - Example: Taylor Swift (500M followers) posts → 1 write instead of 500M writes
    - Saves $\sim 99.9$% of write capacity for high-follower accounts

2. **Read Performance**: 95% of users have < 10K followers
    - These users get pre-computed timelines (50ms read latency)
    - Only 5% of users (celebrities) add +30ms for on-demand query

3. **Cost Efficiency**: Reduces infrastructure costs by 80%
    - Pure fanout-on-write would require 10x Redis capacity
    - Pure fanout-on-read would require 20x PostgreSQL capacity

4. **Real-World Validation**: Used by Twitter, Instagram, Facebook
    - Twitter threshold: 5K followers
    - Instagram threshold: 10K followers
    - Facebook threshold: 2K friends

### Implementation Details

```
function post_content(user_id, content):
  // Write post to database
  post_id = postgres.insert("posts", user_id, content)
  
  // Check follower count
  follower_count = get_follower_count(user_id)
  
  if follower_count < 10000:
    // Fanout-on-write: Push to followers' timelines
    kafka.publish("post.created.fanout", post_id, user_id)
  else:
    // Fanout-on-read: Skip pre-computation
    kafka.publish("post.created.celebrity", post_id, user_id)
  
  return post_id
```

### Trade-offs Accepted

| What We Gain                                                      | What We Sacrifice                                                     |
|-------------------------------------------------------------------|-----------------------------------------------------------------------|
| ✅ **Write Efficiency**: 99.9% reduction in writes for celebrities | ❌ **Read Latency**: +30ms for feeds of users following celebrities    |
| ✅ **Cost Savings**: $50M/month saved on Redis capacity            | ❌ **Complexity**: Two code paths (fanout-on-write + fanout-on-read)   |
| ✅ **Scalability**: Handles unlimited follower counts              | ❌ **Threshold Tuning**: Need to adjust 10K threshold based on traffic |

### When to Reconsider

- **If 50%+ users follow celebrities**: Increase fanout-on-read, raise threshold to 5K
- **If feed latency exceeds 200ms**: Lower threshold to 20K (more fanout-on-write)
- **If celebrity write storms occur**: Lower threshold to 5K
- **If Redis costs are acceptable**: Use pure fanout-on-write (simpler logic)

---

## 2. Primary Database: PostgreSQL (Sharded) Over MongoDB

### The Problem

The system requires storing structured relational data (users, posts, follows) with ACID guarantees for critical
operations (posting, following). Need to choose between:

- **SQL databases** (PostgreSQL, MySQL): Relational, ACID, schemas
- **NoSQL databases** (MongoDB, DynamoDB): Flexible schemas, eventual consistency

### Options Considered

| Database                 | Pros                                                                                                        | Cons                                                                          | Use Case Fit                                                            |
|--------------------------|-------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| **PostgreSQL (Sharded)** | • ACID transactions<br/>• Strong consistency<br/>• Rich query support (JOINs)<br/>• Mature sharding (Citus) | • Vertical scaling limits<br/>• Complex sharding setup                        | • ✅ Structured social graph<br/>• ✅ Post metadata<br/>• ✅ User profiles |
| **MongoDB**              | • Flexible schema<br/>• Horizontal scaling<br/>• Native sharding                                            | • No ACID for multi-doc<br/>• Limited JOIN support<br/>• Eventual consistency | • ❌ Social graph (needs JOINs)<br/>• ⚠️ Post metadata (acceptable)      |
| **MySQL (Sharded)**      | • Similar to PostgreSQL<br/>• Proven at scale (Facebook)                                                    | • Weaker JSON support<br/>• Less modern features                              | • ✅ Similar to PostgreSQL                                               |
| **DynamoDB**             | • Fully managed<br/>• Unlimited scale                                                                       | • No complex queries<br/>• Expensive for small workloads                      | • ❌ Social graph (needs JOINs)<br/>• ⚠️ Timeline data (acceptable)      |

### Decision Made

**PostgreSQL (Sharded)** with 16 shards, partitioned by `user_id`.

### Rationale

1. **ACID Guarantees**: Critical for post creation and follow actions
    - Example: User posts → must be durable immediately (no eventual consistency)
    - Example: User follows → must prevent duplicate follows (unique constraints)

2. **Social Graph Queries**: Requires JOIN operations
   ```sql
   -- Get posts from users you follow (complex query)
   SELECT p.* FROM posts p
   JOIN follows f ON p.user_id = f.followee_id
   WHERE f.follower_id = 123
   ORDER BY p.created_at DESC
   LIMIT 50;
   ```
    - MongoDB would require application-level JOINs (slow)

3. **Mature Ecosystem**: Proven at massive scale
    - Instagram: 2B users on PostgreSQL (via Citus sharding)
    - Discord: 1B messages/day on PostgreSQL
    - Uber: Billions of trips on PostgreSQL

4. **JSON Support**: PostgreSQL JSONB handles flexible post metadata
   ```sql
   CREATE TABLE posts (
     post_id BIGINT PRIMARY KEY,
     media_urls JSONB NOT NULL  -- Array of {url, type, width, height}
   );
   ```
    - Combines SQL structure with NoSQL flexibility

### Implementation Details

**Sharding Strategy**:

- **Shard Key**: `user_id`
- **Shard Count**: 16 shards (each shard: 125M users)
- **Shard Function**: `shard_id = hash(user_id) % 16`
- **Read Replicas**: 2 replicas per shard (total: 48 instances)

**Data Co-location**:

- `users` table: Sharded by `user_id`
- `posts` table: Sharded by `user_id` (co-located with user)
- `follows` table: Sharded by `follower_id`

### Trade-offs Accepted

| What We Gain                                                        | What We Sacrifice                                           |
|---------------------------------------------------------------------|-------------------------------------------------------------|
| ✅ **ACID Transactions**: Strong consistency for critical operations | ❌ **Scaling Complexity**: Manual sharding and rebalancing   |
| ✅ **Rich Queries**: JOINs, aggregations, indexes                    | ❌ **Schema Rigidity**: Schema migrations require planning   |
| ✅ **Mature Tooling**: Battle-tested at scale                        | ❌ **Vertical Limits**: Each shard has storage limits (~5TB) |

### When to Reconsider

- **If post schema becomes extremely varied**: Consider MongoDB for flexible schemas
- **If write throughput exceeds 100k/sec per shard**: Add more shards (16 → 32 → 64)
- **If cross-shard queries become dominant**: Consider denormalization or separate query database
- **If ACID not needed**: DynamoDB might be simpler (fully managed)

---

## 3. Timeline Cache: Redis Sorted Sets Over Memcached

### The Problem

Need to cache pre-computed timelines (list of post IDs) for ultra-low-latency feed retrieval. Requirements:

- **Sorted by timestamp** (most recent first)
- **Range queries** (fetch top 50 posts)
- **Fast inserts** (fanout-on-write adds posts)
- **Atomic updates** (multiple writers adding posts)

### Options Considered

| Cache                        | Pros                                                                                                           | Cons                                                                             | Timeline Use Case                                                    |
|------------------------------|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------|----------------------------------------------------------------------|
| **Redis Sorted Sets (ZSET)** | • Native sorted data structure<br/>• Range queries (ZREVRANGE)<br/>• Atomic operations<br/>• Persistence (AOF) | • Higher memory cost vs Memcached<br/>• Single-threaded (per instance)           | • ✅ Perfect for sorted timelines<br/>• ✅ Native ZADD/ZREVRANGE       |
| **Memcached**                | • Faster (multi-threaded)<br/>• Lower memory cost                                                              | • No sorted data structures<br/>• No persistence<br/>• Application-level sorting | • ❌ Would need serialized arrays<br/>• ❌ Manual sorting in app layer |
| **DynamoDB**                 | • Fully managed<br/>• Persistent                                                                               | • Higher latency (10ms vs 1ms)<br/>• Expensive for cache workload                | • ⚠️ Acceptable but slower                                           |

### Decision Made

**Redis Sorted Sets (ZSET)** with 1000-node cluster, sharded by `user_id`.

### Rationale

1. **Native Sorted Data Structure**: Redis ZSET is purpose-built for this use case
   ```
   Key: timeline:{user_id}
   Type: Sorted Set
   Members: post_id
   Score: timestamp
   
   ZADD timeline:12345 1699123456 "post_789"  // Add post
   ZREVRANGE timeline:12345 0 49              // Get top 50 posts
   ```
    - With Memcached, would need to store serialized array and sort in application layer (slower)

2. **Atomic Operations**: Multiple workers can add posts concurrently without race conditions
   ```
   // Worker 1 and Worker 2 both add posts to same timeline
   Worker1: ZADD timeline:12345 timestamp1 post_id1
   Worker2: ZADD timeline:12345 timestamp2 post_id2
   // Redis handles concurrency automatically
   ```
    - Memcached would require application-level locking (complex)

3. **Range Queries**: Efficient pagination
   ```
   ZREVRANGE timeline:12345 0 49        // Page 1 (top 50)
   ZREVRANGE timeline:12345 50 99       // Page 2 (next 50)
   ```
    - With Memcached, would need to fetch entire array and slice in app (wasteful)

4. **Persistence**: Redis AOF (Append-Only File) provides crash recovery
    - If Redis crashes, timelines are not lost
    - Memcached loses all data on restart (acceptable for true cache, but timelines are "source of truth" for 24 hours)

### Implementation Details

**Redis Configuration**:

- **Cluster Size**: 1000 nodes (sharded by `user_id`)
- **Replication**: 3x (master + 2 replicas per shard)
- **Memory**: 256 GB per node (total: 256 TB)
- **TTL**: 24 hours per timeline
- **Eviction Policy**: `volatile-lru` (evict least recently used with TTL)

**Data Structure**:

```
Key: timeline:{user_id}
Type: Sorted Set
Score: Unix timestamp (or ML score)
Members: post_id

Commands:
- ZADD timeline:123 1699123456 post_789        // Add post
- ZREVRANGE timeline:123 0 49                  // Get top 50
- ZREM timeline:123 post_789                   // Remove post (deletion)
- ZCARD timeline:123                           // Count total posts
```

### Trade-offs Accepted

| What We Gain                                    | What We Sacrifice                                          |
|-------------------------------------------------|------------------------------------------------------------|
| ✅ **Native Sorting**: O(log N) sorted inserts   | ❌ **Memory Cost**: 2x more expensive than Memcached        |
| ✅ **Atomic Operations**: Concurrent writes safe | ❌ **Single-Threaded**: Each Redis instance single-threaded |
| ✅ **Persistence**: AOF crash recovery           | ❌ **Operational Complexity**: More complex than Memcached  |

### When to Reconsider

- **If memory costs exceed $10M/month**: Consider Memcached + application-level sorting
- **If workload becomes write-heavy (> 1M writes/sec)**: Redis may bottleneck (add more shards)
- **If timelines don't need sorting**: Memcached would be sufficient (e.g., if sorting happens client-side)
- **If need serverless**: Consider DynamoDB (fully managed, no ops)

---

## 4. Engagement Storage: Cassandra Over DynamoDB

### The Problem

Need to store high-volume engagement events (likes, comments, saves) with requirements:

- **Massive write throughput**: 578k writes/sec sustained
- **Time-series queries**: Fetch recent likes for a post
- **Durable storage**: Don't lose engagement data
- **Cost-effective**: Store billions of events

### Options Considered

| Database       | Pros                                                                                                             | Cons                                                                          | Engagement Use Case                                                               |
|----------------|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
| **Cassandra**  | • Excellent write performance<br/>• Time-series optimized<br/>• Open-source (cheaper)<br/>• Fine-grained control | • Operational overhead<br/>• Manual tuning needed                             | • ✅ Perfect for time-series likes/comments<br/>• ✅ 578k writes/sec easily handled |
| **DynamoDB**   | • Fully managed<br/>• Unlimited scale<br/>• No ops burden                                                        | • Expensive at high volume<br/>• Less flexible                                | • ✅ Can handle workload<br/>• ❌ 10x more expensive than Cassandra                 |
| **PostgreSQL** | • ACID guarantees<br/>• Rich queries                                                                             | • Poor write performance for time-series<br/>• Vertical scaling limits        | • ❌ Can't handle 578k writes/sec                                                  |
| **MongoDB**    | • Flexible schema<br/>• Time-series collections                                                                  | • Less mature for time-series than Cassandra<br/>• Higher cost than Cassandra | • ⚠️ Acceptable but not optimal                                                   |

### Decision Made

**Cassandra** with time-series data model, partitioned by `post_id`.

### Rationale

1. **Write-Optimized**: Cassandra's LSM tree architecture excels at writes
    - **Cassandra**: 10k writes/sec per node (578k total with 60 nodes)
    - **PostgreSQL**: 1k writes/sec per node (would need 600 nodes)

2. **Time-Series Data Model**: Native support for time-ordered queries
   ```sql
   CREATE TABLE likes (
     post_id BIGINT,
     created_at TIMESTAMP,
     user_id BIGINT,
     PRIMARY KEY ((post_id), created_at, user_id)
   ) WITH CLUSTERING ORDER BY (created_at DESC);
   ```
    - Query recent likes: `SELECT * FROM likes WHERE post_id = 789 LIMIT 100;` (fast)

3. **Cost Efficiency**: Open-source vs. DynamoDB
    - **Cassandra**: 60 nodes × $500/month = $30k/month
    - **DynamoDB**: 578k writes/sec × $1.25/million writes × 86400 sec/day × 30 days = $1.8M/month
    - **Savings**: $1.77M/month (60x cheaper)

4. **Battle-Tested**: Proven at scale
    - Instagram: 500M+ likes/day on Cassandra
    - Netflix: Billions of events/day on Cassandra
    - Apple: 400k writes/sec on Cassandra

### Implementation Details

**Cluster Configuration**:

- **Nodes**: 60 nodes (each node: 10k writes/sec)
- **Replication**: RF=3 (3 copies of each partition)
- **Partitioning**: By `post_id` (ensures all likes for a post are co-located)
- **Compaction**: Time-Window Compaction (optimized for time-series)

**Schema Design**:

```sql
-- Likes table
CREATE TABLE likes (
  post_id BIGINT,            -- Partition key
  created_at TIMESTAMP,       -- Clustering column 1
  user_id BIGINT,             -- Clustering column 2
  PRIMARY KEY ((post_id), created_at, user_id)
) WITH CLUSTERING ORDER BY (created_at DESC)
  AND compaction = {'class': 'TimeWindowCompactionStrategy'};

-- Comments table
CREATE TABLE comments (
  post_id BIGINT,
  created_at TIMESTAMP,
  comment_id BIGINT,
  user_id BIGINT,
  text TEXT,
  PRIMARY KEY ((post_id), created_at, comment_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

### Trade-offs Accepted

| What We Gain                                             | What We Sacrifice                                                           |
|----------------------------------------------------------|-----------------------------------------------------------------------------|
| ✅ **Cost Savings**: $1.77M/month vs DynamoDB             | ❌ **Operational Overhead**: Need Cassandra expertise                        |
| ✅ **Write Performance**: 578k writes/sec with 60 nodes   | ❌ **Manual Scaling**: Need to add nodes manually (vs DynamoDB auto-scaling) |
| ✅ **Fine-Grained Control**: Tune compaction, replication | ❌ **Complexity**: More complex than managed DynamoDB                        |

### When to Reconsider

- **If operational burden too high**: Consider DynamoDB (fully managed)
- **If write throughput exceeds 1M writes/sec**: Add more Cassandra nodes (scales linearly)
- **If need ACID transactions**: Use PostgreSQL (Cassandra is eventually consistent)
- **If team lacks Cassandra expertise**: DynamoDB might be safer (managed service)

---

## 5. Media Upload: Pre-Signed URLs Over Direct Server Upload

### The Problem

Users need to upload high-resolution images (up to 20MB) and videos (up to 1GB). Two approaches:

- **Direct Server Upload**: Client uploads to application server → server uploads to S3
- **Pre-Signed URLs**: Client requests URL → uploads directly to S3 (bypasses server)

### Options Considered

| Approach                             | Pros                                                                                    | Cons                                                                           | Performance                                       |
|--------------------------------------|-----------------------------------------------------------------------------------------|--------------------------------------------------------------------------------|---------------------------------------------------|
| **Pre-Signed URLs (This Over That)** | • Bypasses application servers<br/>• Faster (client → S3 edge)<br/>• Lower server costs | • Client needs S3 SDK<br/>• URL expiry management                              | • Upload: 5 sec for 10MB<br/>• Server load: 0%    |
| **Direct Server Upload**             | • Simpler client<br/>• Server-side validation                                           | • Server bottleneck<br/>• Expensive egress<br/>• Slower (client → server → S3) | • Upload: 10 sec for 10MB<br/>• Server load: 100% |

### Decision Made

**Pre-Signed URLs** with 15-minute expiry.

### Rationale

1. **Server Offload**: Eliminates upload traffic from application servers
    - **Without Pre-Signed**: 17k uploads/sec × 10MB avg = 170 GB/sec bandwidth on app servers
    - **With Pre-Signed**: 0 GB/sec on app servers (clients upload directly to S3)
    - **Cost Savings**: $50M/year in server bandwidth

2. **Faster Uploads**: Client uploads to nearest S3 edge location
    - **Direct Server**: Client (Tokyo) → Server (US-East) → S3 (US-East) = 200ms latency
    - **Pre-Signed**: Client (Tokyo) → S3 Edge (Tokyo) → S3 (US-East) = 50ms latency
    - **Improvement**: 4x faster

3. **Scalability**: S3 handles unlimited throughput
    - **Direct Server**: Need to scale application servers with upload traffic (expensive)
    - **Pre-Signed**: S3 auto-scales (no manual intervention)

4. **Industry Standard**: Used by Dropbox, Google Photos, Instagram
    - Proven pattern for large file uploads

### Implementation Details

**Upload Flow**:

1. **Client Request**: `POST /upload/request`
   ```json
   {
     "filename": "image.jpg",
     "file_size": 10485760,  // 10 MB
     "content_type": "image/jpeg"
   }
   ```

2. **Server Generates Pre-Signed URL**:
   ```python
   s3_client = boto3.client('s3')
   presigned_url = s3_client.generate_presigned_url(
     ClientMethod='put_object',
     Params={
       'Bucket': 'instagram-media',
       'Key': f'uploads/{user_id}/{uuid}.jpg',
       'ContentType': 'image/jpeg'
     },
     ExpiresIn=900  // 15 minutes
   )
   ```

3. **Client Uploads Directly**:
   ```
   PUT https://instagram-media.s3.amazonaws.com/uploads/123/abc.jpg
   ```

4. **S3 Triggers Event**: After upload completes, S3 triggers Lambda or Kafka event for processing

**Security**:

- **Expiry**: URL valid for 15 minutes (prevents abuse)
- **Validation**: Server validates file size and type before generating URL
- **Rate Limiting**: Limit to 10 upload requests per user per minute

### Trade-offs Accepted

| What We Gain                                       | What We Sacrifice                                          |
|----------------------------------------------------|------------------------------------------------------------|
| ✅ **Cost Savings**: $50M/year in server bandwidth  | ❌ **Client Complexity**: Need S3 SDK in mobile/web apps    |
| ✅ **Faster Uploads**: 4x faster via S3 edge        | ❌ **URL Management**: Need to handle expiry and retries    |
| ✅ **Server Offload**: 0% upload traffic on servers | ❌ **Debugging**: Harder to debug client-side upload issues |

### When to Reconsider

- **If client apps lack S3 SDK support**: Use direct server upload (simpler)
- **If need real-time progress tracking**: Server upload allows server-side progress reporting
- **If security requirements prohibit client-side upload**: Use server proxy
- **If file sizes are small (< 100KB)**: Direct server upload acceptable (less overhead)

---

## 6. Image Processing: Async Queue Over Synchronous Processing

### The Problem

After upload, images require processing (resize, compress, filter, feature extraction). Two approaches:

- **Synchronous**: User waits while server processes image before receiving success
- **Asynchronous**: User receives immediate success, processing happens in background

### Options Considered

| Approach                         | Pros                                                                                | Cons                                                       | User Experience                                         |
|----------------------------------|-------------------------------------------------------------------------------------|------------------------------------------------------------|---------------------------------------------------------|
| **Async Queue (This Over That)** | • Immediate user feedback<br/>• Scalable worker pool<br/>• Fault-tolerant (retries) | • Eventual consistency<br/>• More complex                  | • Response: 2 sec<br/>• Processing: 5-10 sec background |
| **Synchronous Processing**       | • Simpler logic<br/>• Immediate consistency                                         | • User waits 10+ sec<br/>• Poor UX<br/>• Server bottleneck | • Response: 10+ sec<br/>• User may close app            |

### Decision Made

**Async Queue (Kafka)** with worker pool processing in background.

### Rationale

1. **User Experience**: User sees success in 2 seconds (after upload to S3)
    - **Sync**: User waits 10+ seconds (upload + processing) → 30% abandon rate
    - **Async**: User waits 2 seconds → 5% abandon rate
    - **Improvement**: 6x better retention

2. **Scalability**: Worker pool scales independently of API servers
    - **Sync**: Processing on API servers → need 10x more API capacity
    - **Async**: Dedicated worker pool (100 instances) scales separately

3. **Fault Tolerance**: Failed jobs can be retried without user seeing error
    - **Sync**: If processing fails, user sees error and must retry upload
    - **Async**: Job re-queued automatically, user unaware of transient failures

4. **Cost Efficiency**: Spot instances for workers
    - **Sync**: API servers must be always-on (expensive on-demand instances)
    - **Async**: Workers can use spot instances (70% cheaper)

### Implementation Details

**Async Flow**:

1. **Upload Complete**: S3 triggers event → Kafka topic `media.uploaded`
2. **Worker Consumes Job**: Worker picks up job from Kafka
3. **Processing Steps**:
    - Download original from S3
    - Generate 8 versions (thumbnails, WebP, AVIF, etc.)
    - Apply filters (if requested)
    - Extract visual features (CNN model)
    - Upload all versions to S3
    - Save metadata to PostgreSQL
4. **Trigger Fanout**: Publish `post.created` event → fanout to followers

**Worker Configuration**:

- **Pool Size**: 100 workers
- **Autoscaling**: Scale to 500 workers during peak
- **Instance Type**: c6i.4xlarge (16 vCPUs, 32 GB RAM)
- **Spot Instances**: 70% cost savings
- **Processing Time**: 3-5 seconds per image

### Trade-offs Accepted

| What We Gain                                     | What We Sacrifice                                                |
|--------------------------------------------------|------------------------------------------------------------------|
| ✅ **User Experience**: 2 sec response vs 10+ sec | ❌ **Eventual Consistency**: Post visible before thumbnails ready |
| ✅ **Scalability**: Independent worker scaling    | ❌ **Complexity**: Queue management, retries, monitoring          |
| ✅ **Fault Tolerance**: Automatic retries         | ❌ **Debugging**: Harder to trace async failures                  |

### When to Reconsider

- **If post visibility must be instant**: Use synchronous processing (accept 10+ sec latency)
- **If processing is very fast (< 500ms)**: Sync processing acceptable
- **If team lacks queue expertise**: Sync processing simpler (but poor UX)
- **If retry logic too complex**: Sync processing has immediate error feedback

---

## 7. Recommendation Engine: Offline Training + Online Serving Over Real-Time Training

### The Problem

Need to provide personalized content recommendations. Two approaches:

- **Offline Training**: Train ML model on batch data (overnight), serve pre-trained model
- **Real-Time Training**: Continuously update model with every user interaction

### Options Considered

| Approach                                               | Pros                                                                                | Cons                                                                          | Performance                                                           |
|--------------------------------------------------------|-------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| **Offline Training + Online Serving (This Over That)** | • Scalable training (batch)<br/>• Fast inference (pre-trained)<br/>• Cost-effective | • Model staleness (24h old)<br/>• Can't adapt immediately                     | • Training: 8 hours (daily)<br/>• Inference: <50ms                    |
| **Real-Time Training**                                 | • Always up-to-date<br/>• Instant adaptation                                        | • Extremely expensive (GPU)<br/>• Slow inference<br/>• Complex infrastructure | • Training: Continuous<br/>• Inference: 500ms+<br/>• Cost: 10x higher |
| **Hybrid (Online Learning)**                           | • Balance freshness and cost<br/>• Incremental updates                              | • Complex implementation<br/>• Still requires batch retraining                | • Training: Hourly updates<br/>• Inference: 100ms                     |

### Decision Made

**Offline Training + Online Serving** with daily model retraining.

### Rationale

1. **Cost Efficiency**: Batch training on 100 GPUs for 8 hours vs continuous training
    - **Offline**: 100 GPUs × 8 hours/day = 800 GPU-hours/day = $8k/day
    - **Real-Time**: 1000 GPUs × 24 hours/day = 24,000 GPU-hours/day = $240k/day
    - **Savings**: $232k/day = $7M/month

2. **Inference Latency**: Pre-trained model serves predictions in <50ms
    - **Offline**: Model loaded in memory → fast inference
    - **Real-Time**: Model training and inference interleaved → slow (500ms+)

3. **Scalability**: Can train on massive datasets (30 days of data)
    - **Offline**: Batch process on Spark cluster (10B events)
    - **Real-Time**: Limited to recent data (streaming window)

4. **Acceptable Staleness**: Recommendations based on yesterday's data are "good enough"
    - User behavior doesn't change dramatically overnight
    - Example: If user liked cats yesterday, still relevant today

### Implementation Details

**Offline Training Pipeline**:

1. **Data Collection** (Daily, 2am):
    - Extract past 30 days of interactions from data warehouse (Hive/BigQuery)
    - Features: user_id, post_id, action (like/save/share), dwell_time, timestamp

2. **Feature Engineering** (4am):
    - User features: past_likes, demographics, activity_level
    - Post features: engagement_rate, visual_features (CNN), author_popularity
    - Context features: time_of_day, day_of_week

3. **Model Training** (6am):
    - Train two-tower model (user tower + post tower)
    - Objective: Predict P(user will engage with post)
    - Framework: TensorFlow or PyTorch on 100 GPUs
    - Training time: 8 hours

4. **Model Validation** (2pm):
    - Evaluate on holdout set (last 3 days)
    - Metrics: AUC, precision@50, recall@50
    - A/B test on 1% traffic

5. **Model Deployment** (6pm):
    - Deploy to TensorFlow Serving (100 instances)
    - Gradual rollout (1% → 10% → 100%)

**Online Serving**:

- **Request**: User opens app → request top 50 recommendations
- **Candidate Generation**: Fetch 1000 candidates from various sources
- **Batch Inference**: Score all 1000 candidates with pre-trained model (<50ms)
- **Re-ranking**: Apply business rules (diversity, freshness)
- **Response**: Return top 50

### Trade-offs Accepted

| What We Gain                                        | What We Sacrifice                                                    |
|-----------------------------------------------------|----------------------------------------------------------------------|
| ✅ **Cost Savings**: $7M/month vs real-time training | ❌ **Model Staleness**: Recommendations based on yesterday's data     |
| ✅ **Fast Inference**: <50ms for 1000 candidates     | ❌ **Delayed Adaptation**: Can't react to viral content instantly     |
| ✅ **Scalable Training**: Can process 10B events     | ❌ **Cold Start**: New users get generic recommendations for 24 hours |

### When to Reconsider

- **If staleness hurts engagement**: Consider hourly retraining (hybrid approach)
- **If need real-time personalization**: Use online learning (more complex)
- **If training time exceeds 12 hours**: Add more GPUs or optimize model
- **If cost is not a constraint**: Real-time training provides best personalization

---

## 8. CDN Strategy: Multi-Tier CDN Over Direct Origin Serving

### The Problem

Need to serve 500B media requests/day globally with low latency. Two approaches:

- **Direct Origin**: Serve all requests directly from S3 buckets
- **Multi-Tier CDN**: Use CDN with edge locations (Cloudflare/Akamai)

### Options Considered

| Approach                            | Pros                                                                    | Cons                                                                       | Performance                              | Cost                                               |
|-------------------------------------|-------------------------------------------------------------------------|----------------------------------------------------------------------------|------------------------------------------|----------------------------------------------------|
| **Multi-Tier CDN (This Over That)** | • Low latency (edge)<br/>• 90% cache hit rate<br/>• Reduced origin load | • CDN costs<br/>• Cache invalidation complexity                            | • Latency: 20ms<br/>• Origin load: 10%   | • $150M/month<br/>(CDN cheaper than origin egress) |
| **Direct Origin Serving**           | • Simpler<br/>• No CDN vendor dependency                                | • High latency (cross-region)<br/>• Origin overload<br/>• Expensive egress | • Latency: 200ms<br/>• Origin load: 100% | • $300M/month<br/>(S3 egress: $0.09/GB)            |
| **Single-Tier CDN**                 | • Simpler than multi-tier<br/>• Still low latency                       | • Higher origin load (85% hit rate)<br/>• Less efficient                   | • Latency: 30ms<br/>• Origin load: 15%   | • $180M/month                                      |

### Decision Made

**Multi-Tier CDN** (Edge + Shield + Origin) with Cloudflare.

### Rationale

1. **Cost Savings**: CDN bandwidth cheaper than S3 egress
    - **S3 Egress**: $0.09/GB × 250 PB/month = $22.5M/month (origin only)
    - **CDN**: $0.02/GB × 250 PB/month = $5M/month
    - **With 90% hit rate**: Only 10% hits origin → $2.25M S3 egress + $5M CDN = $7.25M/month
    - **Savings**: $15.25M/month vs direct origin

2. **Latency Reduction**: Edge locations near all users
    - **Direct Origin**: User (Australia) → S3 (US) = 200ms
    - **CDN Edge**: User (Australia) → Edge (Sydney) = 20ms
    - **Improvement**: 10x faster

3. **Origin Protection**: Shield reduces load on S3
    - **Without Shield**: 50B requests/day hit origin (5% cache miss)
    - **With Shield**: 5B requests/day hit origin (0.5% cache miss)
    - **Reduction**: 10x fewer origin requests

4. **Cache Efficiency**: Multi-tier improves hit rate
    - **Single-Tier**: 85% hit rate at edge
    - **Multi-Tier**: 90% hit rate at edge + 95% at shield = 99.5% effective hit rate

### Implementation Details

**Three-Tier Architecture**:

**Tier 1 - Origin (S3)**:

- S3 buckets in 3 regions (US, EU, Asia)
- Cross-region replication (CRR)
- Lifecycle policies (move old media to Glacier)

**Tier 2 - Shield (Regional PoPs)**:

- 10 regional shield locations (Cloudflare)
- Cache-Control: `max-age=604800` (7 days)
- 95% hit rate

**Tier 3 - Edge (200+ Cities)**:

- 200+ edge locations worldwide
- Cache-Control: `max-age=604800` (7 days)
- 90% hit rate

**Cache Headers**:

```
Cache-Control: public, max-age=604800, immutable
```

**Rationale**:

- `public`: Cacheable by CDN and browsers
- `max-age=604800`: 7 days TTL (images never change after upload)
- `immutable`: Tells browser not to revalidate (use new URL if editing)

**Cache Invalidation**:

- Purge API: `POST /purge` with URL pattern
- Propagation time: ~1 minute to all edge locations
- Use for deleted posts or edited images

### Trade-offs Accepted

| What We Gain                                    | What We Sacrifice                                     |
|-------------------------------------------------|-------------------------------------------------------|
| ✅ **Cost Savings**: $15M/month vs direct origin | ❌ **Vendor Lock-in**: Dependent on Cloudflare         |
| ✅ **Low Latency**: 20ms vs 200ms                | ❌ **Cache Invalidation Delay**: ~1 minute propagation |
| ✅ **Origin Protection**: 99.5% requests cached  | ❌ **Complexity**: Managing multi-tier cache           |

### When to Reconsider

- **If CDN costs exceed $200M/month**: Consider building custom PoP network (Facebook style)
- **If invalidation latency unacceptable**: Use shorter TTLs or versioned URLs
- **If need more control**: Build custom CDN (requires massive investment)
- **If traffic < 1PB/month**: Single-tier CDN sufficient (simpler)

---

## 9. Feed Stitching: Server-Side Merging Over Client-Side Merging

### The Problem

Need to combine follower posts, recommended posts, and ads into a unified feed. Two approaches:

- **Server-Side**: Merge on backend, send unified feed to client
- **Client-Side**: Send separate lists to client, merge in app

### Options Considered

| Approach                                 | Pros                                                              | Cons                                                                 | Performance                                   |
|------------------------------------------|-------------------------------------------------------------------|----------------------------------------------------------------------|-----------------------------------------------|
| **Server-Side Merging (This Over That)** | • Single API call<br/>• Consistent logic<br/>• Easier A/B testing | • Server overhead<br/>• Rigid mix ratio                              | • Client: 1 API call<br/>• Server: 50ms merge |
| **Client-Side Merging**                  | • Flexible client control<br/>• Reduced server load               | • Multiple API calls<br/>• Inconsistent logic<br/>• Harder debugging | • Client: 3 API calls<br/>• Server: 0ms merge |

### Decision Made

**Server-Side Merging** with configurable mix ratios.

### Rationale

1. **Consistent User Experience**: All clients see same feed ordering
    - **Server-Side**: Backend controls mix ratio (60% follower, 30% reco, 10% ads)
    - **Client-Side**: iOS, Android, Web may have different merging logic → inconsistent UX

2. **A/B Testing**: Easy to experiment with mix ratios
    - **Server-Side**: Change ratio in config → instantly affects all clients
    - **Client-Side**: Need to deploy app updates → slow iteration

3. **Single API Call**: Lower client latency
    - **Server-Side**: 1 API call → 80ms total
    - **Client-Side**: 3 API calls (follower, reco, ads) → 150ms total (sequential)

4. **Business Logic Centralized**: Easier to update ad placement rules
    - **Server-Side**: Change ad frequency in backend → all clients updated
    - **Client-Side**: Need app updates → slow rollout

### Implementation Details

**Server-Side Stitching Algorithm**:

```
function stitch_feed(user_id, limit=50):
  // Step 1: Parallel fetch
  [follower_posts, recommended_posts, ads] = parallel_fetch(user_id)
  
  // Step 2: Mix with ratio (60/30/10)
  merged = []
  for i in 0..limit:
    if i % 10 < 6:
      merged.append(follower_posts.pop())  // 60%
    elif i % 10 < 9:
      merged.append(recommended_posts.pop())  // 30%
    else:
      merged.append(ads.pop())  // 10%
  
  // Step 3: Apply final ranking model
  ranked = ranking_model.score(merged, user_context)
  
  // Step 4: Hydrate post details
  hydrated = hydrate_posts(ranked[0:limit])
  
  return hydrated
```

**Config-Driven Mix Ratios**:

```json
{
  "feed_config": {
    "default": {
      "follower": 0.6,
      "recommended": 0.3,
      "ads": 0.1
    },
    "premium_users": {
      "follower": 0.7,
      "recommended": 0.3,
      "ads": 0.0
    }
  }
}
```

### Trade-offs Accepted

| What We Gain                                                   | What We Sacrifice                                     |
|----------------------------------------------------------------|-------------------------------------------------------|
| ✅ **Consistent UX**: Same feed logic for all clients           | ❌ **Server Overhead**: Merge computation on backend   |
| ✅ **Fast Iteration**: Change mix ratios without app deployment | ❌ **Rigid Control**: Client can't customize mix ratio |
| ✅ **Single API Call**: Lower client latency                    | ❌ **Server Load**: More computation per request       |

### When to Reconsider

- **If server load exceeds capacity**: Push merge logic to client (requires app updates)
- **If need per-user customization**: Client-side merge allows user preferences
- **If clients want control**: Provide separate APIs + client-side merge option
- **If latency critical**: Client-side merge can be parallelized better

---

## 10. Hot Key Mitigation: Distributed Counters Over Single Counter

### The Problem

Viral posts receive millions of likes in minutes, overwhelming a single Redis counter. Need to handle 100k+ likes/second
for a single post without Redis bottleneck.

### Options Considered

| Approach                                  | Pros                                                                   | Cons                                                   | Performance                                                   |
|-------------------------------------------|------------------------------------------------------------------------|--------------------------------------------------------|---------------------------------------------------------------|
| **Distributed Counters (This Over That)** | • Scalable (100+ shards)<br/>• No hotspot<br/>• Handles 10M writes/sec | • Read overhead (aggregate shards)<br/>• Complex logic | • Write: 1ms per shard<br/>• Read: 10ms (SUM 100 shards)      |
| **Single Counter**                        | • Simple logic<br/>• Fast reads (1ms)                                  | • Hotspot bottleneck<br/>• Max 10k writes/sec          | • Write: 1ms (until bottleneck)<br/>• Fails at 10k writes/sec |
| **Local Buffering**                       | • Reduces Redis load<br/>• Fast writes                                 | • Data loss risk<br/>• Complex flush logic             | • Write: 0.1ms (in-memory)<br/>• Flush: Every 100ms           |

### Decision Made

**Distributed Counters** with 100 shards + local buffering.

### Rationale

1. **Write Scalability**: Distributes load across 100 Redis keys
    - **Single Counter**: 1 Redis key handles max 10k writes/sec → bottleneck
    - **Distributed**: 100 shards × 10k/sec = 1M writes/sec capacity
    - **Example**: Viral post with 100k likes/sec → 1k writes/sec per shard (no bottleneck)

2. **Fault Isolation**: One shard failure doesn't bring down entire system
    - **Single Counter**: If Redis key fails, all likes fail
    - **Distributed**: If 1 shard fails, only 1% of likes affected (99% still work)

3. **Real-World Proven**: Used by Twitter, Instagram for viral content
    - Twitter's celebrity tweet likes use similar sharding

### Implementation Details

**Shard Allocation**:

```
// Hash user_id to pick shard (ensures same user always hits same shard)
shard_id = hash(user_id) % 100

// Like action
function like_post(user_id, post_id):
  shard_id = hash(user_id) % 100
  redis.INCR(f"post:likes:{post_id}:{shard_id}")
  return "Liked"

// Read total likes
function get_like_count(post_id):
  total = 0
  for shard_id in 0..99:
    count = redis.GET(f"post:likes:{post_id}:{shard_id}")
    total += count
  return total
```

**Local Buffering (Optional Optimization)**:

```
// Buffer likes in application memory, flush every 100ms
local_buffer = {}

function like_post_buffered(user_id, post_id):
  shard_id = hash(user_id) % 100
  key = f"post:likes:{post_id}:{shard_id}"
  
  // Increment local counter
  local_buffer[key] = local_buffer.get(key, 0) + 1
  
  return "Liked"

// Background thread flushes every 100ms
every 100ms:
  for key, count in local_buffer.items():
    redis.INCRBY(key, count)
  local_buffer.clear()
```

### Trade-offs Accepted

| What We Gain                                                        | What We Sacrifice                                                     |
|---------------------------------------------------------------------|-----------------------------------------------------------------------|
| ✅ **Write Scalability**: 1M writes/sec (vs 10k with single counter) | ❌ **Read Overhead**: 10ms to aggregate 100 shards (vs 1ms single key) |
| ✅ **Fault Isolation**: 1 shard failure = 1% impact                  | ❌ **Complexity**: More complex counter logic                          |
| ✅ **No Hotspots**: Even load distribution                           | ❌ **Storage Overhead**: 100x more Redis keys                          |

### When to Reconsider

- **If read latency unacceptable**: Use single counter + periodic aggregation to cache
- **If viral posts rare**: Single counter sufficient (simpler)
- **If Redis cluster large enough**: Increase Redis cluster capacity instead of sharding
- **If need exact real-time count**: Distributed counters have slight eventual consistency

---

## Summary Comparison Table

| Decision                  | Chosen Approach      | Alternative          | Key Reason                          | Trade-off                              |
|---------------------------|----------------------|----------------------|-------------------------------------|----------------------------------------|
| **Fanout Strategy**       | Hybrid               | Pure Fanout-on-Write | Write efficiency for celebrities    | +30ms read latency for celebrity posts |
| **Primary Database**      | PostgreSQL (Sharded) | MongoDB              | ACID + social graph JOINs           | Manual sharding complexity             |
| **Timeline Cache**        | Redis Sorted Sets    | Memcached            | Native sorted data structure        | 2x memory cost                         |
| **Engagement Storage**    | Cassandra            | DynamoDB             | Cost ($1.77M/month savings)         | Operational overhead                   |
| **Media Upload**          | Pre-Signed URLs      | Direct Server Upload | Server offload + speed              | Client complexity                      |
| **Image Processing**      | Async Queue          | Synchronous          | User experience (2 sec vs 10 sec)   | Eventual consistency                   |
| **Recommendation Engine** | Offline Training     | Real-Time Training   | Cost ($7M/month savings)            | 24-hour staleness                      |
| **CDN Strategy**          | Multi-Tier CDN       | Direct Origin        | Cost + latency ($15M/month savings) | Cache invalidation delay               |
| **Feed Stitching**        | Server-Side          | Client-Side          | Consistent UX + A/B testing         | Server overhead                        |
| **Hot Key Mitigation**    | Distributed Counters | Single Counter       | Scalability (1M writes/sec)         | Read aggregation overhead              |
