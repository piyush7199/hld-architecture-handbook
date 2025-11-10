# 3.5.4 Design Instagram/Pinterest Feed (Media-Heavy, Personalized)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## 1. Problem Statement

Design a highly scalable, media-heavy social platform like **Instagram** or **Pinterest** that serves personalized feeds
to billions of users. The system must handle massive volumes of image and video uploads, deliver ultra-low-latency
feeds ($\sim 150$ ms), support real-time engagement (likes, comments, saves), and provide intelligent content discovery
through machine learning-based recommendations.

**Key Challenges:**

1. **Media at Scale**: Storing and serving petabytes of high-resolution images and videos with global distribution
2. **Personalized Feed**: Combining content from followed users with ML-recommended content in real-time
3. **Read-Heavy Workload**: Handling 10+ billion timeline views per day with sub-200ms latency
4. **Real-Time Engagement**: Processing millions of likes/comments/shares per second without hotspots
5. **Content Discovery**: Running sophisticated visual search and recommendation algorithms on massive datasets
6. **Multi-Device Optimization**: Serving appropriately sized media for web, iOS, Android with minimal bandwidth

Unlike text-based systems like Twitter, Instagram/Pinterest requires:

- **Complex Media Pipeline**: Image filtering, compression, format conversion, thumbnail generation
- **Visual Search**: ML models analyzing image content for recommendations (not just social graph)
- **Higher Storage Costs**: $20+$ petabytes vs. $\sim 2$ petabytes for text
- **CDN Dependency**: 90%+ of traffic is media delivery through CDN networks

---

## 2. Requirements and Scale Estimation

### 2.1 Functional Requirements

| Requirement                   | Description                                                             | Priority |
|-------------------------------|-------------------------------------------------------------------------|----------|
| **Media Upload**              | Users upload high-resolution images (up to 20MB) and videos (up to 1GB) | Critical |
| **Timeline Generation**       | Serve personalized feed mixing followed users + recommendations         | Critical |
| **Real-Time Engagement**      | Support likes, comments, shares, saves with immediate feedback          | Critical |
| **Content Discovery**         | Explore page with ML-recommended content based on visual features       | High     |
| **Stories/Ephemeral Content** | 24-hour temporary content (Instagram Stories)                           | Medium   |
| **Direct Messaging**          | Private messaging with media sharing                                    | Medium   |
| **Search**                    | Search users, hashtags, and visual content                              | High     |
| **Notifications**             | Real-time push notifications for engagement                             | High     |

### 2.2 Non-Functional Requirements

| Requirement           | Target                                           | Justification                                      |
|-----------------------|--------------------------------------------------|----------------------------------------------------|
| **Feed Load Latency** | $< 150$ ms (p95)                                 | Users expect instant feed on app open              |
| **Media Upload**      | $< 5$ seconds for 10MB image                     | Acceptable upload time                             |
| **Availability**      | 99.99% uptime                                    | Social platforms are critical daily services       |
| **Data Consistency**  | Eventual consistency for feeds, strong for posts | Feeds can be slightly stale; posts must be durable |
| **Scalability**       | Handle 10x traffic spikes (viral posts)          | Content can go viral unpredictably                 |
| **Cost Efficiency**   | Optimize CDN bandwidth and storage               | Media delivery is most expensive component         |

### 2.3 Scale Estimation

#### Traffic Metrics

| Metric                       | Assumption              | Calculation        | Result                    |
|------------------------------|-------------------------|--------------------|---------------------------|
| **Daily Active Users (DAU)** | 2 billion users         | -                  | 2B DAU                    |
| **Posts Uploaded**           | 500M posts/day          | $500M \div 86400$  | $\sim 5,780$ posts/sec    |
| **Peak Post Rate**           | 3x average              | $5,780 \times 3$   | $\sim 17,340$ posts/sec   |
| **Timeline Views**           | 10B views/day           | $10B \div 86400$   | $\sim 115,740$ reads/sec  |
| **Peak Read Rate**           | 3x average              | $115,740 \times 3$ | $\sim 347,220$ reads/sec  |
| **Likes/Comments**           | 50B engagements/day     | $50B \div 86400$   | $\sim 578,700$ writes/sec |
| **Media Requests**           | 50 images per feed view | $10B \times 50$    | $500B$ CDN requests/day   |

#### Storage Estimation

| Component              | Calculation                                                 | Result               |
|------------------------|-------------------------------------------------------------|----------------------|
| **Average Image Size** | 3MB (original) + 500KB (thumbnails)                         | $3.5$ MB per image   |
| **Average Video Size** | 50MB (mobile), 200MB (HD)                                   | $100$ MB avg         |
| **Posts per Day**      | 400M images + 100M videos                                   | 500M posts           |
| **Daily Storage**      | $(400M \times 3.5 \text{MB}) + (100M \times 100 \text{MB})$ | $\sim 11.4$ TB/day   |
| **Annual Storage**     | $11.4 \text{TB} \times 365$                                 | $\sim 4.16$ PB/year  |
| **5-Year Storage**     | $4.16 \text{PB} \times 5$                                   | $\sim 20.8$ PB       |
| **Replication (3x)**   | $20.8 \text{PB} \times 3$                                   | $\sim 62.4$ PB total |

#### Bandwidth Estimation

| Metric                  | Calculation                                       | Result              |
|-------------------------|---------------------------------------------------|---------------------|
| **Average Media Size**  | 500KB (optimized thumbnails)                      | 500 KB              |
| **Media per Feed View** | 50 images                                         | 50 images           |
| **Bandwidth per View**  | $50 \times 500 \text{KB}$                         | $25$ MB             |
| **Daily Bandwidth**     | $10B \times 25 \text{MB}$                         | $250$ PB/day        |
| **Monthly CDN Cost**    | $250 \text{PB} \times 30 \times \$0.02/\text{GB}$ | $\sim \$150M/month$ |

**Key Insight**: CDN bandwidth cost is the dominant infrastructure expense, making image optimization critical.

#### Compute Estimation

| Component                 | QPS              | CPU per Request | Total Cores         |
|---------------------------|------------------|-----------------|---------------------|
| **Feed Service**          | 347k peak reads  | 50ms CPU        | $\sim 17,350$ cores |
| **Media Upload**          | 17k peak writes  | 200ms CPU       | $\sim 3,400$ cores  |
| **Recommendation Engine** | 347k peak        | 100ms CPU       | $\sim 34,700$ cores |
| **Engagement Service**    | 578k peak writes | 10ms CPU        | $\sim 5,780$ cores  |

**Total**: $\sim 61,230$ cores for application tier (not including ML training clusters).

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

The system combines **Twitter's Fanout Logic** (**3.2.1**) with a **Complex Media Pipeline** and a **Real-Time Ranking
Service**.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER (Mobile/Web)                        │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                ┌────────────┴──────────┐
                │                       │
        ┌───────▼────────┐      ┌──────▼────────┐
        │   CDN Network  │      │  API Gateway  │
        │ (Cloudflare)   │      │  (Rate Limit) │
        └────────────────┘      └───────┬───────┘
                                        │
                        ┌───────────────┼───────────────┐
                        │               │               │
                ┌───────▼─────┐  ┌─────▼──────┐  ┌────▼────────┐
                │Upload Service│  │Feed Service│  │Engage Service│
                └───────┬──────┘  └─────┬──────┘  └────┬────────┘
                        │               │               │
                ┌───────▼──────┐  ┌─────▼──────┐  ┌────▼────────┐
                │ Media Pipeline│  │Feed Builder│  │Redis Counters│
                │ (Transcode)   │  │ (Stitcher) │  │ (Hot Keys)  │
                └───────┬───────┘  └─────┬──────┘  └─────────────┘
                        │                │
                        │          ┌─────▼─────────┐
                        │          │Recommendation │
                        │          │   Engine      │
                        │          │ (ML Models)   │
                        │          └───────┬───────┘
                        │                  │
                ┌───────▼──────┐  ┌────────▼───────────┐
                │ Object Store │  │ Timeline Cache     │
                │ (S3/GCS)     │  │ (Redis Sorted Set) │
                └──────────────┘  └────────────────────┘
                                           │
                                  ┌────────▼────────┐
                                  │   PostgreSQL    │
                                  │ (Posts Metadata)│
                                  └─────────────────┘
```

### 3.1 Core Components

| Component                 | Technology         | Purpose                         | Scalability                |
|---------------------------|--------------------|---------------------------------|----------------------------|
| **API Gateway**           | Kong/NGINX         | Rate limiting, auth, routing    | Horizontal (stateless)     |
| **Upload Service**        | Go/Java            | Handles media ingestion         | Horizontal (async queues)  |
| **Media Pipeline**        | FFmpeg/ImageMagick | Transcode, compress, filter     | Horizontal (workers)       |
| **Object Store**          | S3/GCS             | Stores original + derived media | Petabyte-scale             |
| **CDN**                   | Cloudflare/Akamai  | Global media delivery           | Global edge network        |
| **Feed Service**          | Java/Go            | Serves personalized timelines   | Horizontal (cache-heavy)   |
| **Feed Builder**          | Kafka Consumers    | Fanout-on-write to Redis        | Horizontal (partitioned)   |
| **Recommendation Engine** | Python/TensorFlow  | ML-based content ranking        | Horizontal (model serving) |
| **Timeline Cache**        | Redis Sorted Sets  | Stores pre-computed feeds       | Sharded by user_id         |
| **Engagement Service**    | Go                 | Handles likes/comments          | Horizontal (Redis cluster) |
| **PostgreSQL**            | Sharded PostgreSQL | Post metadata, users, graph     | Sharded by user_id         |
| **Cassandra**             | Cassandra          | Engagement events (time-series) | Partitioned by post_id     |

---

## 4. Data Models

### 4.1 PostgreSQL Schemas (Relational Data)

#### Users Table

```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    profile_image_url TEXT,
    bio TEXT,
    follower_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_username (username)
);
```

#### Posts Table (Sharded by user_id)

```sql
CREATE TABLE posts (
    post_id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    caption TEXT,
    media_urls JSONB NOT NULL, -- Array of {url, type, width, height}
    media_type VARCHAR(10), -- 'image' or 'video'
    location_id BIGINT,
    hashtags TEXT[],
    like_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_created (user_id, created_at DESC),
    INDEX idx_hashtags USING GIN(hashtags)
);
```

#### Social Graph (Sharded by follower_id)

```sql
CREATE TABLE follows (
    follower_id BIGINT NOT NULL,
    followee_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id),
    INDEX idx_followee (followee_id)
);
```

### 4.2 Cassandra Schema (Time-Series Engagement)

#### Likes Table

```sql
CREATE TABLE likes (
    post_id BIGINT,
    user_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY ((post_id), created_at, user_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

#### Comments Table

```sql
CREATE TABLE comments (
    comment_id BIGINT,
    post_id BIGINT,
    user_id BIGINT,
    text TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY ((post_id), created_at, comment_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

### 4.3 Redis Data Structures

#### Timeline Cache (Sorted Set)

```
Key: timeline:{user_id}
Type: Sorted Set
Members: post_id
Score: timestamp (or ML_score)
TTL: 24 hours

Example:
ZADD timeline:12345 1699123456 "post_789" 1699123400 "post_456"
ZREVRANGE timeline:12345 0 49 -- Get top 50 posts
```

#### Like Counters (String + HyperLogLog)

```
Key: post:likes:{post_id}
Type: String (atomic counter)

Key: post:likers:{post_id}
Type: HyperLogLog (unique user count)

Example:
INCR post:likes:789
PFADD post:likers:789 user_123
```

#### Hot Post Cache (Hash)

```
Key: post:{post_id}
Type: Hash
Fields: user_id, caption, media_urls, like_count, comment_count, created_at
TTL: 1 hour

Example:
HGETALL post:789
```

### 4.4 Object Storage Structure

```
s3://instagram-media/
├── images/
│   ├── original/
│   │   └── {year}/{month}/{day}/{post_id}.jpg
│   ├── thumbnails/
│   │   └── {post_id}_150x150.jpg
│   ├── medium/
│   │   └── {post_id}_640x640.jpg
│   └── large/
│       └── {post_id}_1080x1080.jpg
└── videos/
    ├── original/
    │   └── {year}/{month}/{day}/{post_id}.mp4
    ├── transcoded/
    │   ├── {post_id}_720p.mp4
    │   └── {post_id}_1080p.mp4
    └── thumbnails/
        └── {post_id}_thumb.jpg
```

---

## 5. Detailed Component Design

### 5.1 Media Upload Pipeline

**Flow**: Client → Upload Service → Async Queue → Media Workers → Object Store

The upload pipeline handles:

1. **Pre-signed URL Generation**: Client requests upload permission
2. **Direct Upload**: Client uploads to S3 via pre-signed URL (bypass server)
3. **Async Processing**: Worker picks up job from Kafka queue
4. **Image Processing**: Generate multiple sizes (thumbnail, medium, large)
5. **Video Transcoding**: Convert to multiple bitrates (360p, 720p, 1080p)
6. **Filter Application**: Apply Instagram-style filters if requested
7. **Metadata Extraction**: Extract dimensions, EXIF data, duration
8. **Database Write**: Store post metadata in PostgreSQL

**Why Pre-Signed URLs?**

- **Reduces Server Load**: Media bypasses application servers
- **Faster Upload**: Client uploads directly to S3's edge location
- **Cost Savings**: No egress costs from app servers

*See pseudocode.md::generate_presigned_url() and pseudocode.md::process_media_job() for implementation.*

### 5.2 Feed Generation Strategy (Hybrid Fanout)

Instagram uses a **Hybrid Fanout Model**:

**Fanout-on-Write (for most users)**:

- When user posts, push post_id to all followers' timeline caches (Redis)
- Suitable for users with $< 10$K followers
- Guarantees instant feed updates

**Fanout-on-Read (for celebrities)**:

- Don't pre-compute feed for users following celebrities
- Fetch celebrity posts at read time
- Prevents celebrity write amplification

**Implementation**:

```
function generate_feed(user_id):
  // Step 1: Get pre-computed fanout posts
  fanout_posts = redis.ZREVRANGE("timeline:{user_id}", 0, 49)
  
  // Step 2: Get celebrity posts (users with > 10K followers)
  celebrity_posts = fetch_celebrity_posts(user_id)
  
  // Step 3: Get ML-recommended posts
  recommended_posts = recommendation_engine.get_recommendations(user_id, limit=50)
  
  // Step 4: Merge and rank
  all_posts = merge_and_rank(fanout_posts, celebrity_posts, recommended_posts)
  
  // Step 5: Hydrate metadata
  return hydrate_posts(all_posts[0:49])
```

*See pseudocode.md::generate_feed() for detailed implementation.*

### 5.3 Recommendation Engine Architecture

**Offline Training Pipeline** (runs daily):

1. **Data Collection**: Collect past 30 days of user interactions (likes, saves, shares, dwell time)
2. **Feature Engineering**: Extract visual features (CNN embeddings), user features, graph features
3. **Model Training**: Train collaborative filtering + content-based model on Spark cluster
4. **Model Deployment**: Deploy updated model to serving layer

**Online Serving** (real-time):

1. **Candidate Generation**: Fetch 1000 potential posts from various sources (popular, similar users, visual similarity)
2. **Feature Extraction**: Extract real-time features (user's recent activity, time of day)
3. **Ranking**: Run ML model to score each post
4. **Re-Ranking**: Apply business rules (diversity, freshness, novelty)

**Model Architecture**:

- **Two-Tower Model**: User tower + Post tower
- **User Tower**: Embeds user_id, recent_posts_liked, time_features
- **Post Tower**: Embeds post_id, visual_features (from ResNet), text_features (from caption)
- **Score**: Cosine similarity between user and post embeddings

*See pseudocode.md::RecommendationEngine for implementation.*

### 5.4 Real-Time Engagement System

**Challenge**: Handling 578k likes/comments per second with **hot key problem**.

**Solution: Multi-Layer Write Path**

1. **Layer 1: Redis Atomic Counters** (immediate response)
    - `INCR post:likes:{post_id}`
    - Returns in $< 1$ ms
    - Risk: Counter could be lost if Redis fails

2. **Layer 2: Cassandra Write** (durable persistence)
    - Async write to Cassandra after Redis
    - Partition key: `post_id`
    - Ensures durability

3. **Layer 3: PostgreSQL Update** (periodic batch)
    - Every 5 minutes, aggregate Redis counters
    - Update `posts.like_count` in PostgreSQL
    - Used for analytics and long-term storage

**Hot Key Mitigation**:

- **Sharding**: Shard Redis cluster by `post_id` hash
- **Local Aggregation**: Buffer likes in application memory, flush every 100ms
- **Rate Limiting**: Limit likes to 1 per user per second per post

*See pseudocode.md::like_post() for implementation.*

---

## 6. Image Processing and Optimization

### 6.1 Multi-Resolution Strategy

Instagram generates **8+ versions** of each image:

| Version         | Dimensions           | Use Case        | File Size  |
|-----------------|----------------------|-----------------|------------|
| **Original**    | Variable (up to 4K)  | Backup/archive  | 3-20 MB    |
| **Feed Large**  | 1080x1080            | Web feed        | 150-300 KB |
| **Feed Medium** | 640x640              | Mobile feed     | 80-150 KB  |
| **Thumbnail**   | 150x150              | Grid view       | 5-10 KB    |
| **Story**       | 1080x1920 (vertical) | Stories         | 200 KB     |
| **AVIF**        | 640x640              | Modern browsers | 40-80 KB   |
| **WebP**        | 640x640              | Chrome/Android  | 60-100 KB  |
| **BlurHash**    | 32x32                | Placeholder     | 1 KB       |

**Adaptive Delivery**:

- Detect device type (mobile/desktop) and screen DPI
- Serve AVIF to modern browsers, WebP to Chrome, JPEG to legacy
- Use BlurHash placeholders while loading

### 6.2 Image Processing Pipeline

**Technologies**:

- **FFmpeg**: Video transcoding
- **ImageMagick**: Image manipulation
- **libvips**: Fast image processing
- **TensorFlow**: Visual feature extraction

**Steps**:

1. **Upload**: Client uploads original image
2. **Validation**: Check file type, dimensions, size
3. **EXIF Stripping**: Remove GPS and metadata for privacy
4. **Resize**: Generate 8 versions using libvips
5. **Compression**: Optimize JPEG quality (85%), WebP, AVIF
6. **Filter Application**: Apply user-selected filter
7. **Visual Feature Extraction**: Run CNN model for recommendations
8. **CDN Upload**: Push all versions to CDN origin

**Performance**:

- Process 10MB image in $< 3$ seconds
- Parallel processing of all 8 versions
- Queue depth: 1000 jobs per worker

*See pseudocode.md::ImageProcessor for implementation.*

---

## 7. Content Discovery and Visual Search

### 7.1 Visual Similarity Search

Pinterest uses **Visual Embeddings** to find similar images.

**Architecture**:

1. **Image Encoder**: ResNet-50 model extracts 2048-dim feature vector
2. **Vector Database**: Store embeddings in FAISS or Milvus
3. **Nearest Neighbor Search**: Find top-K similar images using cosine similarity

**Query Flow**:

```
User clicks "Find Similar" on image
→ Fetch embedding from vector DB
→ Run ANN search (Approximate Nearest Neighbor)
→ Retrieve top 100 similar post_ids
→ Apply ranking and personalization
→ Return top 50 to user
```

**Performance**:

- Index size: 10B images × 2048 dims = 80 TB
- Query latency: $< 50$ ms using HNSW algorithm
- Index updates: Batch every hour (eventual consistency)

*See pseudocode.md::visual_search() for implementation.*

### 7.2 Hashtag and Text Search

**Elasticsearch Index**:

```json
{
  "post_id": 12345,
  "user_id": 789,
  "username": "john_doe",
  "caption": "Beautiful sunset at the beach",
  "hashtags": [
    "sunset",
    "beach",
    "nature"
  ],
  "location": "Malibu, CA",
  "created_at": "2024-01-15T10:30:00Z",
  "engagement_score": 1250
}
```

**Search Queries**:

- Hashtag search: `GET /posts/_search?q=hashtags:sunset`
- Caption search: `GET /posts/_search?q=caption:beach`
- User search: `GET /users/_search?q=username:john*`

**Ranking**:

- **Relevance**: BM25 score from Elasticsearch
- **Engagement**: Boost by like_count, comment_count
- **Freshness**: Boost recent posts (decay function)
- **Personalization**: Boost posts from users you follow

---

## 8. Feed Stitching and Ranking

### 8.1 Feed Stitcher Logic

The **Feed Stitcher** is the most critical component. It combines:

1. **Follower Posts** (social signal)
2. **Recommended Posts** (discovery)
3. **Ads** (monetization)

**Algorithm**:

```
function stitch_feed(user_id):
  // Parallel fetch
  [follower_posts, recommended_posts, ads] = parallel_fetch()
  
  // Mix ratio: 60% follower, 30% recommended, 10% ads
  feed = []
  for i in range(50):
    if i % 10 < 6:
      feed.append(follower_posts.pop())
    elif i % 10 < 9:
      feed.append(recommended_posts.pop())
    else:
      feed.append(ads.pop())
  
  // Apply final ranking model
  feed = ranking_model.score(feed, user_context)
  
  return feed.sort_by_score()
```

**Why This Mix?**

- **60% Follower**: Users expect to see content from people they follow
- **30% Recommended**: Discovery keeps users engaged longer
- **10% Ads**: Monetization without overwhelming user

### 8.2 Ranking Model

**Two-Stage Ranking**:

**Stage 1: Candidate Generation** (cheap, fast):

- Fetch 1000 candidates from multiple sources
- Filter out posts user already seen (bloom filter)
- Basic scoring: engagement_rate × freshness

**Stage 2: Ranking Model** (expensive, accurate):

- Input: user features, post features, context features
- Model: LightGBM or neural network
- Features:
    - **User**: past_likes, past_saves, follower_count, account_age
    - **Post**: like_count, comment_count, save_rate, age
    - **Cross**: user_post_similarity (visual + text), author_follower_overlap
    - **Context**: time_of_day, day_of_week, device_type
- Output: Probability user will engage (click, like, save)

**Model Training**:

- Objective: Maximize engagement time (dwell time + interactions)
- Positive examples: Posts user liked/saved
- Negative examples: Posts user scrolled past without engagement
- Retrain daily with last 30 days of data

*See pseudocode.md::RankingModel for implementation.*

---

## 9. Caching Strategy

### 9.1 Multi-Layer Cache Architecture

| Layer                    | Technology       | Data              | TTL       | Hit Rate |
|--------------------------|------------------|-------------------|-----------|----------|
| **L1: Client Cache**     | Mobile app cache | Feed data, images | 1 hour    | 30%      |
| **L2: CDN Cache**        | Cloudflare       | Images, videos    | 7 days    | 90%      |
| **L3: Redis Timeline**   | Redis Sorted Set | Feed post_ids     | 24 hours  | 85%      |
| **L4: Redis Post Cache** | Redis Hash       | Post metadata     | 1 hour    | 70%      |
| **L5: Database**         | PostgreSQL       | Source of truth   | Permanent | N/A      |

**Cache Invalidation**:

**Problem**: User edits caption or deletes post – caches must be invalidated.

**Solution**:

1. **Write-Through**: Update PostgreSQL first
2. **Async Invalidation**: Publish event to Kafka topic `post.updated`
3. **Cache Listener**: Consumer invalidates Redis keys
4. **CDN Purge**: Send purge request to CDN API (Cloudflare)
5. **Timeline Update**: Remove post_id from affected user timelines

**Edge Case – Viral Post**:

- Post goes viral, receives 100k likes/second
- Redis `post:likes:{post_id}` becomes a hot key
- **Solution**: Use Redis Cluster with hash slot sharding + local buffering

*See pseudocode.md::invalidate_cache() for implementation.*

### 9.2 CDN Configuration

**Multi-Tier CDN**:

1. **Origin**: S3 bucket with media files
2. **Regional PoP**: Cloudflare edge in 200+ cities
3. **Shield**: Mid-tier cache reduces origin load

**Cache Headers**:

```
Cache-Control: public, max-age=604800, immutable
```

**Why immutable?**

- Images never change after upload (use new URL if editing)
- Allows aggressive caching
- Reduces CDN cost by 60%

**Dynamic Image Sizing**:

- URL: `https://cdn.instagram.com/image/12345?w=640&h=640&q=85`
- Cloudflare Worker resizes on-the-fly
- Caches resized version at edge

---

## 10. Availability and Fault Tolerance

### 10.1 Multi-Region Architecture

**Active-Active Deployment**:

```
Region US-EAST                 Region EU-WEST
┌─────────────┐                ┌─────────────┐
│ API Gateway │                │ API Gateway │
│   (Primary) │                │  (Primary)  │
└──────┬──────┘                └──────┬──────┘
       │                              │
┌──────▼──────┐                ┌──────▼──────┐
│  Feed Svc   │                │  Feed Svc   │
│   (Read)    │                │   (Read)    │
└──────┬──────┘                └──────┬──────┘
       │                              │
┌──────▼──────┐                ┌──────▼──────┐
│Redis Replica│◄───Async────►  │Redis Replica│
└──────┬──────┘  Replication   └──────┬──────┘
       │                              │
┌──────▼──────┐                ┌──────▼──────┐
│PostgreSQL   │◄──── Sync ────►│PostgreSQL   │
│  (Sharded)  │   Replication  │  (Sharded)  │
└─────────────┘                └─────────────┘
```

**Write Strategy**:

- **Posts**: Route to user's home region (by user_id hash)
- **Reads**: Serve from nearest region (geo-routing)
- **Consistency**: Eventual consistency for timeline cache, strong for posts

### 10.2 Failure Scenarios

| Failure                        | Impact                     | Mitigation                                | Recovery Time  |
|--------------------------------|----------------------------|-------------------------------------------|----------------|
| **API Gateway Down**           | Users can't access app     | Load balancer fails over to backup        | $< 1$ second   |
| **Redis Timeline Cache Down**  | Slow feed load             | Fallback to PostgreSQL + rebuild cache    | $< 5$ seconds  |
| **PostgreSQL Shard Down**      | 1/16th of users can't post | Promote read replica to primary           | $< 30$ seconds |
| **CDN Down**                   | Images don't load          | Fallback to origin S3 (slower)            | Immediate      |
| **Recommendation Engine Down** | No discovery feed          | Serve only follower posts                 | Immediate      |
| **Kafka Queue Full**           | Fanout delayed             | Drop low-priority fanouts (non-followers) | Self-heals     |

**Circuit Breaker Pattern**:

- If Recommendation Engine fails 50% of requests
- Open circuit breaker → stop calling it
- Serve cached recommendations for 5 minutes
- Retry with exponential backoff

*See pseudocode.md::CircuitBreaker for implementation.*

---

## 11. Bottlenecks and Optimizations

### 11.1 Feed Stitching Latency

**Problem**: Combining follower feed + recommended feed adds 100ms+ latency.

**Solution: Parallel Fetching**

```
parallel_start:
  task1 = async fetch_follower_posts(user_id)
  task2 = async fetch_recommended_posts(user_id)
  task3 = async fetch_ads(user_id)
parallel_wait_all

merge_time = 10ms
total = max(task1, task2, task3) + 10ms
```

**Optimization**:

- Fetch follower posts from Redis: $15$ ms
- Fetch recommended posts from Redis: $20$ ms
- Fetch ads from cache: $10$ ms
- **Total**: $20$ ms + $10$ ms = $30$ ms

### 11.2 Hot Key Problem (Viral Posts)

**Problem**: Celebrity posts receive 1M+ likes in minutes, overwhelming Redis.

**Solution: Distributed Counters**

1. **Shard Counter Across 100 Redis Keys**:
   ```
   post:likes:12345:0
   post:likes:12345:1
   ...
   post:likes:12345:99
   ```
2. **Hash user_id to Pick Shard**:
   ```
   shard = hash(user_id) % 100
   INCR post:likes:12345:{shard}
   ```
3. **Aggregate on Read**:
   ```
   total_likes = SUM(post:likes:12345:*)
   ```

**Result**: Distributes 1M writes/sec across 100 shards = 10k writes/sec per shard.

*See pseudocode.md::distributed_counter() for implementation.*

### 11.3 Media I/O Bottleneck

**Problem**: Object storage cluster handles 500B requests/day.

**Mitigation 1: Aggressive CDN Caching**

- Set Cache-Control: `max-age=604800` (7 days)
- 90% CDN hit rate → only 50B origin requests/day

**Mitigation 2: Cache Derived Sizes**

- Pre-compute all 8 image sizes during upload
- Never resize on-the-fly (except for rare edge cases)

**Mitigation 3: S3 Transfer Acceleration**

- Use S3 Transfer Acceleration for uploads
- Routes uploads to nearest AWS edge location
- 50% faster uploads from distant regions

### 11.4 Database Hotspots

**Problem**: User profile reads (for username, profile pic) hit PostgreSQL hard.

**Solution: Redis User Cache**

```
Key: user:{user_id}
Type: Hash
Fields: username, profile_image_url, bio, follower_count
TTL: 1 hour
```

**Cache-Aside Pattern**:

```
function get_user_profile(user_id):
  profile = redis.HGETALL("user:{user_id}")
  if profile is None:
    profile = postgres.query("SELECT * FROM users WHERE user_id = ?", user_id)
    redis.HMSET("user:{user_id}", profile)
    redis.EXPIRE("user:{user_id}", 3600)
  return profile
```

**Result**: 95% cache hit rate, PostgreSQL load reduced by 20x.

---

## 12. Common Anti-Patterns

### ❌ **1. Storing Images in Database**

**Problem**: Storing media BLOBs in PostgreSQL/MySQL.

**Why It's Bad**:

- Database bloats to petabytes
- Slow queries (scanning large BLOBs)
- Expensive backups
- Can't leverage CDN

**Solution**: Use Object Storage (S3/GCS) and store URLs in database.

```sql
-- ❌ Bad
CREATE TABLE posts (
    post_id BIGINT,
    image_data BYTEA -- 10MB per row!
);

-- ✅ Good
CREATE TABLE posts (
    post_id BIGINT,
    image_url TEXT -- "https://cdn.instagram.com/image/12345.jpg"
);
```

### ❌ **2. Synchronous Media Processing**

**Problem**: Client waits for image processing before seeing success.

**Why It's Bad**:

- Upload takes 30+ seconds
- User closes app before completion
- Poor user experience

**Solution**: Async processing with Kafka queue.

```
// ❌ Bad: Synchronous
Client → Upload Service → [Process 30 sec] → Success

// ✅ Good: Asynchronous
Client → Upload Service → [Queue Job] → Immediate Success
              ↓
          Background Worker → Process → Update DB
```

### ❌ **3. No Rate Limiting on Engagement**

**Problem**: Bots can spam likes/comments, overwhelming the system.

**Why It's Bad**:

- Costs spike due to fake engagement
- Redis hot keys
- Skews recommendation algorithms

**Solution**: Implement multi-layer rate limiting.

| Layer        | Limit         | Purpose                  |
|--------------|---------------|--------------------------|
| **Per User** | 100 likes/min | Prevent bot spamming     |
| **Per IP**   | 500 likes/min | Prevent DDoS             |
| **Per Post** | 10k likes/sec | Prevent hot key overload |

*See pseudocode.md::rate_limit_engagement() for implementation.*

### ❌ **4. Not Handling Deleted Posts in Feed**

**Problem**: User deletes post, but it remains in followers' feed caches.

**Why It's Bad**:

- Broken links in feed (404 errors)
- User confusion
- Cache inconsistency

**Solution**: Publish `post.deleted` event to Kafka, invalidate all timelines.

```
function delete_post(post_id):
  // 1. Soft delete in database
  postgres.UPDATE("posts SET deleted_at = NOW() WHERE post_id = ?", post_id)
  
  // 2. Publish event
  kafka.publish("post.deleted", {post_id: post_id})
  
  // 3. Consumer invalidates caches
  followers = get_followers(user_id)
  for follower in followers:
    redis.ZREM("timeline:{follower}", post_id)
```

*See pseudocode.md::delete_post() for implementation.*

### ❌ **5. Single Global Ranking Model**

**Problem**: Using same recommendation model for all users.

**Why It's Bad**:

- New users (cold start) get poor recommendations
- Different user segments have different preferences
- One-size-fits-all doesn't maximize engagement

**Solution**: Segment-specific models.

| Segment          | Model                   | Objective                       |
|------------------|-------------------------|---------------------------------|
| **New Users**    | Content-based           | Show popular posts (no history) |
| **Active Users** | Collaborative filtering | Personalized based on history   |
| **Creators**     | Creator-focused         | Show trending posts, analytics  |
| **Shoppers**     | E-commerce optimized    | Product-heavy recommendations   |

---

## 13. Alternative Approaches

### 13.1 Alternative: Full Fanout-on-Read

**Approach**: Skip pre-computed timelines, generate feed on-demand.

**Pros**:

- No fanout write amplification
- Always fresh content
- Simpler architecture

**Cons**:

- Feed load takes 500ms+ (too slow)
- High database load (100k+ QPS per shard)
- Expensive ranking computation per request

**When to Use**: Small-scale apps with $< 1$M users.

### 13.2 Alternative: Kafka Streams for Feed

**Approach**: Use Kafka Streams to build timeline as stream processing.

**Pros**:

- Real-time feed updates
- Scalable stream processing
- Exactly-once semantics

**Cons**:

- Complex infrastructure
- Higher operational overhead
- Still needs Redis for low-latency reads

**When to Use**: If you already have Kafka expertise and want real-time guarantees.

### 13.3 Alternative: GraphQL Federation

**Approach**: Use GraphQL with federated services instead of REST.

**Pros**:

- Client specifies exact fields needed
- Reduces over-fetching
- Better mobile bandwidth usage

**Cons**:

- Adds complexity (federated gateway)
- Harder to cache (dynamic queries)
- Steeper learning curve

**When to Use**: If you need flexible client-driven queries and have GraphQL expertise.

---

## 14. Monitoring and Observability

### 14.1 Key Metrics

| Metric                        | Target     | Alert Threshold | Impact                      |
|-------------------------------|------------|-----------------|-----------------------------|
| **Feed Load Latency (p95)**   | $< 150$ ms | $> 300$ ms      | User experience degradation |
| **Image Upload Success Rate** | $> 99.9$%  | $< 99.5$%       | Users can't post            |
| **CDN Hit Rate**              | $> 90$%    | $< 85$%         | Cost spike + slow loads     |
| **Recommendation Latency**    | $< 50$ ms  | $> 100$ ms      | Feed load timeout           |
| **Redis Timeline Cache Hit**  | $> 85$%    | $< 75$%         | Database overload           |
| **Kafka Consumer Lag**        | $< 1$ min  | $> 5$ min       | Delayed fanout              |
| **Engagement Write QPS**      | Baseline   | $> 2$x baseline | Potential bot attack        |

### 14.2 Distributed Tracing

**OpenTelemetry Trace Example**:

```
Span: GET /feed (user_id=12345)
├─ Span: fetch_follower_posts [15ms]
│  └─ Span: redis.ZREVRANGE [2ms]
├─ Span: fetch_recommended_posts [22ms]
│  ├─ Span: recommendation_engine.get [18ms]
│  └─ Span: redis.MGET [4ms]
├─ Span: stitch_and_rank [8ms]
└─ Span: hydrate_posts [12ms]
   └─ Span: redis.HMGET [3ms]

Total: 57ms
```

**Tracing Tools**: Jaeger, Zipkin, AWS X-Ray

### 14.3 Logging Strategy

**Structured Logs**:

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "INFO",
  "service": "feed-service",
  "trace_id": "abc123",
  "user_id": 12345,
  "endpoint": "/feed",
  "latency_ms": 57,
  "cache_hit": true,
  "posts_returned": 50
}
```

**Log Aggregation**: Use ELK stack (Elasticsearch, Logstash, Kibana) or Datadog.

**Alert Examples**:

- Feed latency p95 > 300ms for 5 minutes → Page on-call engineer
- CDN hit rate < 85% → Investigate cache configuration
- Kafka consumer lag > 5 minutes → Scale up consumers

---

## 15. Deployment and Infrastructure

### 15.1 Kubernetes Deployment

**Service Mesh**: Istio for traffic management, retries, circuit breaking.

**Horizontal Pod Autoscaling**:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: feed-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: feed-service
  minReplicas: 50
  maxReplicas: 500
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

**Pod Resource Limits**:

```yaml
resources:
  requests:
    cpu: 2000m
    memory: 4Gi
  limits:
    cpu: 4000m
    memory: 8Gi
```

### 15.2 Database Sharding Strategy

**PostgreSQL Sharding**:

- **Shard Key**: `user_id`
- **Shard Count**: 16 shards
- **Shard Function**: `shard_id = hash(user_id) % 16`

**Example**:

```
user_id = 12345
shard_id = hash(12345) % 16 = 7
→ Route to PostgreSQL shard-7
```

**Cross-Shard Queries**: Avoid if possible; use denormalization or read replicas.

---

## 16. Performance Optimization

### 16.1 Database Query Optimization

**Slow Query Example**:

```sql
-- ❌ Bad: Full table scan
SELECT * FROM posts WHERE user_id = 12345 ORDER BY created_at DESC LIMIT 50;
```

**Optimized**:

```sql
-- ✅ Good: Uses composite index
CREATE INDEX idx_user_created ON posts(user_id, created_at DESC);
SELECT post_id, caption, media_urls FROM posts WHERE user_id = 12345 ORDER BY created_at DESC LIMIT 50;
```

**Result**: Query time: 200ms → 5ms

### 16.2 Redis Pipeline

**Without Pipeline**:

```
for post_id in post_ids:
  redis.HGETALL(f"post:{post_id}")  // 50 round trips
  
Total: 50 × 1ms = 50ms
```

**With Pipeline**:

```
pipeline = redis.pipeline()
for post_id in post_ids:
  pipeline.HGETALL(f"post:{post_id}")
pipeline.execute()  // 1 round trip

Total: 1ms
```

**Result**: 50ms → 1ms (50x faster)

### 16.3 Compression

**Response Payload**:

- Enable gzip compression on API responses
- Feed payload: 500KB uncompressed → 50KB compressed (10x smaller)
- Reduces mobile bandwidth by 90%

**Media Compression**:

- JPEG → WebP: 30% smaller
- WebP → AVIF: 50% smaller
- Use AVIF for modern browsers, fallback to WebP, then JPEG

---

## 17. Trade-offs Summary

| What You Gain                                   | What You Sacrifice                                    |
|-------------------------------------------------|-------------------------------------------------------|
| ✅ **Ultra-low feed latency** ($< 150$ ms)       | ❌ Eventual consistency (feed may be stale by seconds) |
| ✅ **Petabyte-scale media storage**              | ❌ High CDN costs ($\sim \$150$M/month)                |
| ✅ **Highly personalized recommendations**       | ❌ Expensive ML training ($\sim 10$K GPU hours/day)    |
| ✅ **Real-time engagement** (likes, comments)    | ❌ Hot key problem (requires distributed counters)     |
| ✅ **Global availability** (multi-region)        | ❌ Complex cross-region replication                    |
| ✅ **Hybrid fanout** (fast + fresh)              | ❌ Complex feed stitching logic                        |
| ✅ **Visual search** (find similar images)       | ❌ Large vector database (80TB+ for embeddings)        |
| ✅ **Adaptive image delivery** (device-specific) | ❌ Generates 8+ versions per image (storage overhead)  |

---

## 18. Real-World Examples

### 18.1 Instagram (Meta)

**Architecture Highlights**:

- **TAO (The Associations and Objects)**: Custom graph database for social graph
- **Cassandra**: Used for feed storage (replaced by custom solution)
- **Memcache**: Massive cache layer (millions of QPS)
- **Haystack**: Custom photo storage system (optimized for small files)

**Key Innovation**: Instagram uses **machine learning embeddings** to understand image content and recommend similar
posts, not just social graph-based recommendations.

**Scale**: 2B+ users, 100M+ photos/day, 500B+ CDN requests/day

### 18.2 Pinterest

**Architecture Highlights**:

- **HBase**: Used for large-scale key-value storage
- **Pinball**: Custom workflow management (like Airflow)
- **PinSearch**: Custom Elasticsearch-based visual search
- **PinLater**: Custom async job queue

**Key Innovation**: Pinterest pioneered **visual search** using deep learning (ResNet embeddings) to find similar pins
based on image content, not just text.

**Scale**: 450M+ users, 200B+ pins, visual search on billions of images

### 18.3 TikTok

**Architecture Highlights**:

- **Recommendation Engine**: Extremely sophisticated ML model (best in class)
- **For You Page (FYP)**: 100% algorithmic feed (no follower posts)
- **Short Video Storage**: Highly optimized video transcoding pipeline
- **Real-Time Analytics**: Track video performance within seconds

**Key Innovation**: TikTok's recommendation engine is unparalleled – it can personalize content even for new users
within 5-10 videos watched.

**Scale**: 1B+ users, 50M+ videos/day, 100B+ video views/day

---

## 19. References

### Related System Design Components

- **[2.1.11 Redis Deep Dive](../../02-components/2.1-databases/2.1.11-redis-deep-dive.md)** - Caching and counters
- **[2.1.9 Cassandra Deep Dive](../../02-components/2.1-databases/2.1.9-cassandra-deep-dive.md)** - Time-series engagement data
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Async fanout and event streaming
- **[2.2.1 Caching Deep Dive](../../02-components/2.2-caching/2.2.1-caching-deep-dive.md)** - Multi-layer caching strategies
- **[2.1.7 PostgreSQL Deep Dive](../../02-components/2.1-databases/2.1.7-postgresql-deep-dive.md)** - Post metadata storage
- **[2.1.13 Elasticsearch Deep Dive](../../02-components/2.1-databases/2.1.13-elasticsearch-deep-dive.md)** - Visual search, image indexing
- **[2.1.4 Database Scaling](../../02-components/2.1.4-database-scaling.md)** - Database sharding strategies

### Related Design Challenges

- **[3.2.1 Twitter Timeline](../3.2.1-twitter-timeline/)** - Fanout strategies and timeline generation
- **[3.4.2 News Feed](../3.4.2-news-feed/)** - Feed ranking and personalization
- **[3.4.4 Recommendation System](../3.4.4-recommendation-system/)** - ML-based recommendations
- **[3.4.8 Video Streaming System](../3.4.8-video-streaming-system/)** - Media processing pipeline

### External Resources

- **Instagram Engineering Blog:** [Instagram Engineering](https://instagram-engineering.com/) - Instagram architecture
- **Pinterest Engineering Blog:** [Pinterest Engineering](https://medium.com/pinterest-engineering) - Pinterest system design
- **"Scaling Instagram" Talk:** [Mike Krieger YouTube](https://www.youtube.com/) - Scaling strategies
- **"Pinterest Visual Search at Scale" Paper:** [arXiv](https://arxiv.org/) - Visual search algorithms
- **"TikTok Recommendation Algorithm" Analysis:** [ByteDance Research](https://www.bytedance.com/) - Recommendation algorithms
- **"TAO: Facebook's Distributed Data Store" Paper:** [USENIX ATC 2013](https://www.usenix.org/) - Social graph storage
- **"Haystack: Facebook's Photo Storage" Paper:** [OSDI 2010](https://www.usenix.org/) - Photo storage architecture

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Fanout patterns, caching strategies
- *Building Microservices* by Sam Newman - Service architecture

---

## 20. Interview Discussion Points

### 20.1 Common Interview Questions

**Q1: How would you handle a celebrity posting and reaching 10M users?**

**Answer**: Use **hybrid fanout**:

- Pre-compute feeds for users with $< 10$K followers (fanout-on-write)
- For celebrities ($> 10$K followers), don't pre-compute – fetch posts at read time (fanout-on-read)
- This prevents write amplification (1 post → 10M writes) while maintaining low read latency

**Q2: How do you prevent hot keys in Redis when a post goes viral?**

**Answer**: Use **distributed counters**:

- Shard counter across 100 Redis keys: `post:likes:12345:0` to `post:likes:12345:99`
- Hash user_id to pick shard: `shard = hash(user_id) % 100`
- Aggregate on read: `SUM(post:likes:12345:*)`
- This distributes 1M writes/sec across 100 shards = 10k writes/sec per shard

**Q3: Why use Redis instead of Memcached for timeline cache?**

**Answer**: Redis provides **Sorted Sets** (ZSET):

- Timeline is naturally a sorted list (by timestamp or ML score)
- Redis `ZADD` and `ZREVRANGE` operations are $O(\log n)$
- Memcached only supports key-value (would need custom sorting in app layer)
- Redis also provides atomic counters (INCR) for engagement

**Q4: How do you ensure feed freshness with caching?**

**Answer**: Use **TTL + cache invalidation**:

- Timeline cache TTL: 24 hours
- On new post: publish event to Kafka → consumer updates followers' timelines
- On post delete: publish event → consumer removes post from timelines
- On feed read: if cache miss, rebuild from database + recommendation engine

**Q5: How would you optimize bandwidth for mobile users?**

**Answer**: Multi-resolution + adaptive delivery:

- Generate 8+ versions per image (150x150 thumbnail to 1080x1080 feed)
- Detect device type and screen DPI
- Serve AVIF (50% smaller) to modern browsers, WebP to Chrome, JPEG to legacy
- Use BlurHash placeholders while loading
- This reduces bandwidth by 80%+

### 20.2 Scalability Discussion

**"How would you scale from 10M to 1B users?"**

| Component          | 10M Users      | 1B Users (100x)         | Scaling Strategy     |
|--------------------|----------------|-------------------------|----------------------|
| **API Servers**    | 100 servers    | 10,000 servers          | Horizontal scaling   |
| **Redis Timeline** | 10 nodes       | 1,000 nodes             | Shard by user_id     |
| **PostgreSQL**     | 4 shards       | 64 shards               | Horizontal sharding  |
| **Kafka**          | 10 partitions  | 1,000 partitions        | Partition by user_id |
| **Recommendation** | 50 GPU servers | 5,000 GPU servers       | Model parallelism    |
| **CDN**            | 1 provider     | 3 providers (multi-CDN) | Redundancy + cost    |

**Key Insight**: Most components scale horizontally via sharding or partitioning. The hardest parts are:

1. **Recommendation Engine**: GPU costs scale linearly with users
2. **CDN Bandwidth**: Costs grow with traffic (can't optimize away)

### 20.3 Trade-Off Discussions

**"Would you choose eventual consistency or strong consistency for the feed?"**

**Answer**: **Eventual consistency** is acceptable because:

- Users don't notice if a post appears 5 seconds late in their feed
- Strong consistency would require synchronous replication → 100ms+ latency
- Feed is read-heavy (115k QPS) – can't afford distributed transactions

**However, use strong consistency for:**

- Post creation (user must see their own post immediately)
- Engagement actions (like button must respond immediately)
- Account actions (follow/unfollow must be reflected)

---

## 21. Security and Privacy Considerations

### 21.1 Content Moderation

**Challenge**: Detect and remove inappropriate content (violence, spam, NSFW).

**Solution: Multi-Layer Moderation**

1. **Client-Side Filtering**: Block known bad words in captions
2. **ML-Based Detection**: Run image classification model (NSFW detector)
3. **Hash-Based Matching**: Compare hash against known bad content database
4. **Human Review**: Flagged content goes to moderation queue
5. **User Reporting**: Allow users to report content

**Performance**:

- ML model runs in $< 100$ ms during upload
- False positive rate: $< 1$%
- Human review: 24-hour SLA

### 21.2 Privacy Controls

**Data Privacy**:

- **EXIF Stripping**: Remove GPS coordinates from photos
- **Private Accounts**: Only followers can see posts
- **Story Deletion**: Auto-delete stories after 24 hours
- **Data Export**: GDPR compliance (user can download all data)
- **Right to Deletion**: User can delete account + all posts

**Access Control**:

```sql
-- Check if user can view post
function can_view_post(viewer_id, post):
  if post.user.is_private:
    return is_follower(viewer_id, post.user_id)
  return true
```

---

## 22. Cost Analysis

### 22.1 Infrastructure Cost Breakdown

| Component                 | Monthly Cost | Percentage | Notes                        |
|---------------------------|--------------|------------|------------------------------|
| **CDN Bandwidth**         | $150M        | 60%        | 250 PB/month @ $0.02/GB      |
| **Object Storage**        | $40M         | 16%        | 60 PB @ $0.023/GB/month      |
| **Compute (EC2/K8s)**     | $30M         | 12%        | 60k cores @ $0.03/core-hour  |
| **Database (RDS/Aurora)** | $15M         | 6%         | 64 shards @ $250k/shard      |
| **Redis Cache**           | $8M          | 3%         | 1000 nodes @ $8k/node        |
| **ML Training (GPUs)**    | $5M          | 2%         | 5000 GPUs @ $1k/GPU/month    |
| **Kafka/Streaming**       | $2M          | 1%         | 1000 brokers @ $2k/broker    |
| **Total**                 | **$250M**    | **100%**   | Per month at Instagram scale |

**Cost Optimization Strategies**:

1. **Increase CDN hit rate from 90% to 95%** → Save $25M/month
2. **Use AVIF format (50% smaller)** → Save $75M/month in bandwidth
3. **Compress older images** → Save $10M/month in storage
4. **Use spot instances for ML training** → Save $2.5M/month

**Revenue**: $\sim \$50$B/year (ads) → Infrastructure is $\sim 6$% of revenue

---

## 23. Advanced Features

### 23.1 Instagram Stories (Ephemeral Content)

**Requirements**:

- Auto-delete after 24 hours
- Circular profile indicator shows who has new stories
- Swipe-through UI (not scroll feed)

**Architecture**:

```
Stories Timeline (Separate from Feed):
Key: stories:viewed:{user_id}
Type: Hash
Fields: {story_id: viewed_timestamp}
TTL: 24 hours

Stories Content:
Key: story:{story_id}
Type: Hash
Fields: {user_id, media_url, created_at, expires_at}
TTL: 24 hours (auto-delete)
```

**Viewed Tracking**:

- Mark story as viewed: `HSET stories:viewed:{user_id} {story_id} {timestamp}`
- Query unviewed stories: Compare all stories from followers with viewed hash

### 23.2 Direct Messaging

**Requirements**:

- Real-time message delivery
- End-to-end encryption (optional)
- Media sharing in DMs

**Architecture**:

- **WebSocket** for real-time delivery
- **Cassandra** for message storage (partition by conversation_id)
- **Redis Pub/Sub** for presence (online/offline status)

*See [3.3.1 Live Chat System](../3.3.1-live-chat-system/README.md) for detailed messaging architecture.*

### 23.3 Shopping and E-Commerce

**Features**:

- Product tags in posts
- In-app checkout
- Creator shops

**Database Schema**:

```sql
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,
    merchant_id BIGINT,
    name VARCHAR(255),
    price DECIMAL(10,2),
    image_url TEXT,
    inventory_count INT
);

CREATE TABLE post_product_tags (
    post_id BIGINT,
    product_id BIGINT,
    tag_position_x FLOAT,
    tag_position_y FLOAT,
    PRIMARY KEY (post_id, product_id)
);
```

**Checkout Flow**:

1. User clicks product tag → Opens product page
2. Add to cart → Stored in Redis session
3. Checkout → Payment gateway integration (Stripe)
4. Order confirmation → Merchant notification

---

## 24. Future Scaling and Evolution

### 24.1 When to Reconsider Architecture

**Scenario 1: Users Reach 5B+**

- Current sharding (64 shards) may not be enough
- **Solution**: Increase shards to 256, use consistent hashing

**Scenario 2: CDN Costs Exceed Revenue**

- Bandwidth costs become unsustainable
- **Solution**: Build custom CDN (like Facebook's PoP network)

**Scenario 3: ML Training Takes > 24 Hours**

- Can't retrain recommendation model daily
- **Solution**: Incremental learning or online learning

### 24.2 Emerging Technologies

**AR Filters** (Augmented Reality):

- Real-time face filters using device GPU
- Store filter assets in CDN
- Run filter processing client-side (not server)

**Video-First Platform** (TikTok competition):

- Shift from images to short-form video
- Requires massive transcoding capacity
- Video recommendations more compute-intensive

**Web3 / NFT Integration**:

- Allow users to mint posts as NFTs
- Blockchain integration for ownership verification
- Decentralized storage (IPFS) for permanence
