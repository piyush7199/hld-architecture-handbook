# Instagram/Pinterest Feed - Quick Overview

## Core Concept

A social media feed system that displays personalized posts (images/videos) to users based on their social graph (followers) and ML recommendations. It uses **Hybrid Fanout** (fanout-on-write for most users, fanout-on-read for celebrities) with multi-layer caching and content discovery.

**Key Challenge**: Serve 100M+ MAU with <200ms feed load, handle 10M posts/day, and provide real-time updates with ML-based content discovery.

---

## Requirements

### Functional
- **Feed Generation**: Personalized feed (follower posts + recommendations)
- **Media Upload**: Images/videos with multiple sizes (thumbnail, medium, large)
- **Fanout**: Push posts to followers' timelines
- **Content Discovery**: ML-based recommendations (similar posts, trending)
- **Engagement**: Likes, comments, shares
- **Hashtags**: Search and discovery by hashtags

### Non-Functional
- **Throughput**: 10M posts/day, 1B feed loads/day
- **Latency**: <200ms for feed load (P99)
- **Availability**: 99.9% uptime
- **Scalability**: Handle 100M+ MAU, 1B+ posts
- **Media Storage**: Petabytes of images/videos

---

## Components

### 1. **Media Upload Pipeline**
- **Pre-signed URLs**: Client uploads directly to S3 (bypass server)
- **Async Queue**: Kafka (`media_processing` topic)
- **Media Workers**: Generate multiple sizes (thumbnail, medium, large), video transcoding
- **Object Store**: S3 (images/videos)

### 2. **Post Service**
- **Post Creation**: Store metadata in PostgreSQL
- **Fanout Service**: Push post_id to followers' timeline caches (Redis)
- **Celebrity Handling**: Skip fanout for users with >10K followers (fanout-on-read)

### 3. **Feed Service**
- **Timeline Generation**: Merge follower posts + recommended posts + ads
- **Ranking**: ML-based ranking (engagement, recency, relevance)
- **Caching**: Multi-layer cache (Redis timeline, Redis post cache, CDN)

### 4. **Recommendation Engine**
- **ML Models**: Collaborative filtering, content-based, deep learning embeddings
- **Real-Time**: Serve from Redis cache
- **Batch**: Train models nightly (Spark)

### 5. **Social Graph (PostgreSQL)**
- **Follows Table**: Sharded by follower_id
- **Queries**: Get followers (fanout), get following (feed generation)

### 6. **Engagement Storage (Cassandra)**
- **Likes Table**: Time-series (post_id → user_id, timestamp)
- **Comments Table**: Time-series (post_id → comment_id, timestamp)
- **Optimized for**: Recent engagement queries

### 7. **Search Service (Elasticsearch)**
- **Hashtag Search**: Inverted index for hashtags
- **User Search**: Full-text search on usernames, bios
- **Visual Search**: ML-based image similarity (ResNet embeddings)

### 8. **CDN (Cloudflare)**
- **Media Delivery**: Images/videos cached at edge (200+ PoPs)
- **Cache Headers**: `Cache-Control: public, max-age=604800, immutable`
- **Dynamic Resizing**: On-the-fly image resizing (Cloudflare Workers)

---

## Architecture Flow

### Post Creation Flow

```
1. Client → Upload Service: Request pre-signed URL
2. Upload Service → Client: Return pre-signed URL
3. Client → S3: Upload image/video directly (bypass server)
4. Client → Post Service: POST /posts {caption, media_urls, hashtags}
5. Post Service → PostgreSQL: Insert post metadata
6. Post Service → Kafka: Publish post.created event
7. Fanout Service → Kafka: Consume post.created
8. Fanout Service → Social Graph: Get followers (SELECT followee_id FROM follows WHERE follower_id = user_id)
9. Fanout Service → Redis: ZADD timeline:{follower_id} timestamp post_id (for users with <10K followers)
10. Fanout Service → Skip fanout (for celebrities, fetch on-read)
11. Media Worker → Kafka: Consume media_processing job
12. Media Worker → S3: Generate thumbnail, medium, large sizes
13. Media Worker → PostgreSQL: Update post with media URLs
```

### Feed Generation Flow

```
1. Client → Feed Service: GET /feed?limit=50
2. Feed Service → Redis: ZREVRANGE timeline:{user_id} 0 49 (follower posts)
3. Feed Service → Recommendation Engine: Get recommended posts (parallel)
4. Feed Service → Redis: HGETALL post:{post_id} (post metadata, cache)
5. Feed Service → Merge & Rank: Combine follower + recommended posts, ML ranking
6. Feed Service → Client: Return feed (50 posts)
```

### Like/Comment Flow

```
1. Client → Engagement Service: POST /posts/{id}/like
2. Engagement Service → Cassandra: Insert like (post_id, user_id, timestamp)
3. Engagement Service → Redis: INCR post:likes:{post_id} (counter)
4. Engagement Service → Kafka: Publish post.liked event
5. Engagement Service → Fanout: Update like count in timeline cache (async)
```

---

## Key Design Decisions

### 1. **Hybrid Fanout Model**

**Fanout-on-Write (for most users)**:
- When user posts, push post_id to all followers' timeline caches (Redis)
- Suitable for users with <10K followers
- Guarantees instant feed updates

**Fanout-on-Read (for celebrities)**:
- Don't pre-compute feed for users following celebrities
- Fetch celebrity posts at read time
- Prevents celebrity write amplification (1M followers × 1 post = 1M writes)

**Trade-off**: Complexity (two strategies) vs write amplification (fanout-on-write for all)

### 2. **PostgreSQL for Social Graph** (vs Cassandra/Neo4j)

**Why PostgreSQL?**
- **ACID Guarantees**: Follow/unfollow operations (consistency)
- **Rich Queries**: JOINs for complex queries (mutual follows, recommendations)
- **Mature Ecosystem**: Widely used, well-documented

**Trade-off**: Harder to scale (need sharding) vs eventual consistency

### 3. **Cassandra for Engagement** (vs PostgreSQL)

**Why Cassandra?**
- **Time-Series Optimized**: Recent likes/comments queries (clustering by timestamp)
- **High Write Throughput**: 100k+ writes/sec per node
- **Horizontal Scaling**: Linear scaling with nodes

**Trade-off**: Eventual consistency vs strong consistency

### 4. **Multi-Layer Caching**

**L1: Client Cache** (Mobile app): Feed data, images (1 hour TTL, 30% hit rate)
**L2: CDN Cache** (Cloudflare): Images, videos (7 days TTL, 90% hit rate)
**L3: Redis Timeline** (Sorted Set): Feed post_ids (24 hours TTL, 85% hit rate)
**L4: Redis Post Cache** (Hash): Post metadata (1 hour TTL, 70% hit rate)
**L5: Database** (PostgreSQL): Source of truth

**Trade-off**: Memory cost vs query latency

### 5. **Pre-signed URLs for Media Upload**

**Why?**
- **Reduces Server Load**: Media bypasses application servers
- **Faster Upload**: Client uploads directly to S3's edge location
- **Cost Savings**: No egress costs from app servers

**Trade-off**: Additional complexity (pre-signed URL generation) vs simpler (direct upload)

### 6. **ML-Based Ranking** (vs Chronological)

**Why?**
- **Engagement**: Higher engagement (likes, comments, shares) → higher rank
- **Recency**: Recent posts ranked higher
- **Relevance**: User interests, past behavior
- **Diversity**: Avoid showing too many posts from same user

**Trade-off**: Computational cost vs user engagement

---

## Data Models

### PostgreSQL Schemas

**Users Table**:
```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    profile_image_url TEXT,
    bio TEXT,
    follower_count INT DEFAULT 0,
    following_count INT DEFAULT 0
);
```

**Posts Table** (Sharded by user_id):
```sql
CREATE TABLE posts (
    post_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    caption TEXT,
    media_urls JSONB NOT NULL,  -- Array of {url, type, width, height}
    hashtags TEXT[],
    like_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Follows Table** (Sharded by follower_id):
```sql
CREATE TABLE follows (
    follower_id BIGINT NOT NULL,
    followee_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);
```

### Cassandra Schema

**Likes Table**:
```sql
CREATE TABLE likes (
    post_id BIGINT,
    user_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY ((post_id), created_at, user_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

### Redis Data Structures

**Timeline Cache** (Sorted Set):
```
Key: timeline:{user_id}
Members: post_id
Scores: timestamp (or ML_score)
TTL: 24 hours
```

**Post Cache** (Hash):
```
Key: post:{post_id}
Fields: user_id, caption, media_urls, like_count, comment_count, created_at
TTL: 1 hour
```

---

## Bottlenecks & Solutions

### 1. **Feed Stitching Latency**

**Problem**: Combining follower feed + recommended feed adds 100ms+ latency

**Solution**: Parallel Fetching
- Fetch follower posts from Redis: 15ms
- Fetch recommended posts from Redis: 20ms
- Fetch ads from cache: 10ms
- **Total**: max(15, 20, 10) + 10ms merge = 30ms

### 2. **Hot Key Problem (Viral Posts)**

**Problem**: Celebrity posts receive 1M+ likes in minutes, overwhelming Redis

**Solution**: Distributed Counters
- Shard counter across 100 Redis keys: `post:likes:{post_id}:shard0`, `post:likes:{post_id}:shard1`, ...
- Aggregate on read: `SUM(post:likes:{post_id}:shard*)`
- Reduces hot key contention by 100x

### 3. **Celebrity Fanout Write Amplification**

**Problem**: User with 10M followers posts → 10M writes to Redis

**Solution**: Fanout-on-Read for Celebrities
- Skip fanout for users with >10K followers
- Fetch celebrity posts at read time (merge with follower feed)
- Reduces writes by 1000x for celebrities

### 4. **Media Storage Cost**

**Problem**: Petabytes of images/videos (expensive storage)

**Solution**:
- **CDN Caching**: 90% hit rate (reduces origin load)
- **Lifecycle Policies**: Move old media to S3 Glacier (cheaper)
- **Compression**: WebP for images (30% smaller), H.265 for videos (50% smaller)

---

## Common Anti-Patterns

### ❌ **1. Fanout-on-Write for All Users**

**Problem**: Celebrity with 10M followers posts → 10M writes (write amplification)

**Bad**:
```
User posts → Fanout to all 10M followers → 10M Redis writes
```

**Good**:
```
User posts → If followers < 10K: Fanout-on-write
           → If followers >= 10K: Skip fanout, fetch on-read
```

### ❌ **2. No Caching**

**Problem**: Every feed load queries database (slow, expensive)

**Bad**:
```
GET /feed → Query PostgreSQL for posts → 500ms latency
```

**Good**:
```
GET /feed → Redis timeline cache → 15ms latency (85% hit rate)
```

### ❌ **3. Synchronous Media Processing**

**Problem**: Client waits for image resizing (slow upload)

**Bad**:
```
POST /posts → Resize images (5s) → Return response
```

**Good**:
```
POST /posts → Return immediately → Async: Resize images in background
```

---

## Monitoring & Observability

### Key Metrics

- **Feed Load Latency P99**: <200ms
- **Post Creation Latency**: <100ms
- **Cache Hit Rates**: Timeline 85%, Post 70%, CDN 90%
- **Fanout Latency**: <500ms (for users with <10K followers)
- **Media Processing Time**: <30s (for images), <5min (for videos)
- **Engagement Rate**: Likes/comments per post

### Alerts

- **Feed Load Latency P99** >300ms
- **Cache Hit Rate** <70% (timeline)
- **Fanout Latency** >1s
- **Media Processing Queue** >10k jobs
- **Database Replication Lag** >500ms

---

## Trade-offs Summary

| Decision | What We Gain | What We Sacrifice |
|----------|--------------|-------------------|
| **Hybrid Fanout** | ✅ Optimal write/read balance | ❌ Complexity (two strategies) |
| **PostgreSQL Social Graph** | ✅ ACID guarantees | ❌ Harder to scale (sharding) |
| **Cassandra Engagement** | ✅ High write throughput | ❌ Eventual consistency |
| **Multi-Layer Caching** | ✅ Fast queries (<200ms) | ❌ Memory cost |
| **Pre-signed URLs** | ✅ Reduced server load | ❌ Additional complexity |
| **ML Ranking** | ✅ Higher engagement | ❌ Computational cost |

---

## Real-World Examples

### Instagram (Meta)
- **Architecture**: TAO (custom graph database), Cassandra (feed storage), Memcache (massive cache), Haystack (photo storage)
- **Innovation**: ML embeddings for image content recommendations
- **Scale**: 2B+ users, 100M+ photos/day, 500B+ CDN requests/day

### Pinterest
- **Architecture**: HBase (key-value storage), Pinball (workflow management), PinSearch (visual search), PinLater (async job queue)
- **Innovation**: Visual search using deep learning (ResNet embeddings)
- **Scale**: 450M+ users, 200B+ pins, visual search on billions of images

### TikTok
- **Architecture**: Sophisticated ML recommendation engine, For You Page (100% algorithmic feed)
- **Innovation**: Best-in-class recommendation engine (personalizes within 5-10 videos)
- **Scale**: 1B+ users, 50M+ videos/day, 100B+ video views/day

---

## Key Takeaways

1. **Hybrid Fanout**: Fanout-on-write (most users) + fanout-on-read (celebrities)
2. **Multi-Layer Caching**: Client → CDN → Redis Timeline → Redis Post → Database
3. **Pre-signed URLs**: Direct S3 upload (bypass server, faster, cheaper)
4. **ML-Based Ranking**: Engagement + recency + relevance + diversity
5. **Distributed Counters**: Shard hot keys (viral posts) across 100 Redis keys
6. **Time-Series Storage**: Cassandra for engagement (likes, comments)
7. **Visual Search**: ML-based image similarity (ResNet embeddings)
8. **CDN Strategy**: Aggressive caching (7 days, immutable) for 90% hit rate

---

## Recommended Stack

- **API Gateway**: NGINX / AWS API Gateway
- **Post Service**: Go/Java (high throughput)
- **Feed Service**: Go/Java (low latency)
- **Social Graph**: PostgreSQL (sharded by follower_id)
- **Engagement Storage**: Cassandra (time-series)
- **Timeline Cache**: Redis Cluster (Sorted Sets)
- **Post Cache**: Redis Cluster (Hash)
- **Media Storage**: S3 (images/videos)
- **CDN**: Cloudflare (200+ PoPs)
- **Search**: Elasticsearch (hashtags, users)
- **Recommendation Engine**: Python (ML models), TensorFlow
- **Message Queue**: Kafka (fanout events, media processing)
- **Monitoring**: Prometheus, Grafana, ELK Stack

