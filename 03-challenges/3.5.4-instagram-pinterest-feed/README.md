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

## Problem Statement

Design a highly scalable, media-heavy social platform like **Instagram** or **Pinterest** that serves personalized feeds
to billions of users. The system must handle massive volumes of image and video uploads, deliver ultra-low-latency
feeds ($\sim 150$ ms), support real-time engagement (likes, comments, saves), and provide intelligent content discovery
through ML-based recommendations.

**Core Challenges:**

1. **Media at Scale**: Storing and serving petabytes of high-resolution images and videos globally
2. **Personalized Feed**: Combining followed users' content with ML-recommended content in real-time
3. **Read-Heavy Workload**: 10+ billion timeline views per day with sub-200ms latency
4. **Real-Time Engagement**: Millions of likes/comments/shares per second without hotspots
5. **Content Discovery**: Running sophisticated visual search and recommendation algorithms
6. **Multi-Device Optimization**: Serving appropriately sized media for different devices

Unlike text-based systems, Instagram/Pinterest requires:

- **Complex Media Pipeline**: Image filtering, compression, format conversion, thumbnail generation
- **Visual Search**: ML models analyzing image content for recommendations
- **Higher Storage Costs**: $20+$ petabytes vs. $\sim 2$ petabytes for text
- **CDN Dependency**: 90%+ of traffic is media delivery through CDN networks

---

## Requirements and Scale Estimation

### Functional Requirements

| Requirement              | Description                                                       | Priority |
|--------------------------|-------------------------------------------------------------------|----------|
| **Media Upload**         | Upload high-resolution images (up to 20MB) and videos (up to 1GB) | Critical |
| **Timeline Generation**  | Serve personalized feed mixing followed users + recommendations   | Critical |
| **Real-Time Engagement** | Support likes, comments, shares, saves with immediate feedback    | Critical |
| **Content Discovery**    | Explore page with ML-recommended content based on visual features | High     |
| **Stories**              | 24-hour temporary content                                         | Medium   |
| **Search**               | Search users, hashtags, and visual content                        | High     |
| **Notifications**        | Real-time push notifications for engagement                       | High     |

### Non-Functional Requirements

| Requirement           | Target                               | Justification                                      |
|-----------------------|--------------------------------------|----------------------------------------------------|
| **Feed Load Latency** | $< 150$ ms (p95)                     | Users expect instant feed on app open              |
| **Media Upload**      | $< 5$ seconds for 10MB image         | Acceptable upload time                             |
| **Availability**      | 99.99% uptime                        | Social platforms are critical daily services       |
| **Data Consistency**  | Eventual for feeds, strong for posts | Feeds can be slightly stale; posts must be durable |
| **Scalability**       | Handle 10x traffic spikes            | Content can go viral unpredictably                 |
| **Cost Efficiency**   | Optimize CDN bandwidth               | Media delivery is most expensive component         |

### Scale Estimation

#### Traffic Metrics

| Metric                 | Assumption              | Calculation        | Result                    |
|------------------------|-------------------------|--------------------|---------------------------|
| **Daily Active Users** | 2 billion users         | -                  | 2B DAU                    |
| **Posts Uploaded**     | 500M posts/day          | $500M \div 86400$  | $\sim 5,780$ posts/sec    |
| **Peak Post Rate**     | 3x average              | $5,780 \times 3$   | $\sim 17,340$ posts/sec   |
| **Timeline Views**     | 10B views/day           | $10B \div 86400$   | $\sim 115,740$ reads/sec  |
| **Peak Read Rate**     | 3x average              | $115,740 \times 3$ | $\sim 347,220$ reads/sec  |
| **Likes/Comments**     | 50B engagements/day     | $50B \div 86400$   | $\sim 578,700$ writes/sec |
| **Media Requests**     | 50 images per feed view | $10B \times 50$    | $500B$ CDN requests/day   |

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

**Key Insight**: CDN bandwidth is the dominant infrastructure expense, making image optimization critical.

#### Compute Estimation

| Component                 | QPS              | CPU per Request | Total Cores         |
|---------------------------|------------------|-----------------|---------------------|
| **Feed Service**          | 347k peak reads  | 50ms CPU        | $\sim 17,350$ cores |
| **Media Upload**          | 17k peak writes  | 200ms CPU       | $\sim 3,400$ cores  |
| **Recommendation Engine** | 347k peak        | 100ms CPU       | $\sim 34,700$ cores |
| **Engagement Service**    | 578k peak writes | 10ms CPU        | $\sim 5,780$ cores  |

**Total**: $\sim 61,230$ cores for application tier.

---

## High-Level Architecture

The system combines **Twitter's Fanout Logic** with a **Complex Media Pipeline** and **Real-Time Ranking Service**.

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

### Core Components

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

## Data Models

### PostgreSQL Schemas

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
    media_urls JSONB NOT NULL,
    media_type VARCHAR(10),
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

### Cassandra Schema

#### Likes Table

```sql
CREATE TABLE likes (
    post_id BIGINT,
    user_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY ((post_id), created_at, user_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

### Redis Data Structures

#### Timeline Cache (Sorted Set)

Instagram uses Redis Sorted Sets (ZSET) for storing pre-computed timelines. This data structure is perfect for timelines
because it maintains sorted order and supports efficient range queries.

```
Key: timeline:{user_id}
Type: Sorted Set
Members: post_id
Score: timestamp (or ML_score)
TTL: 24 hours

Operations:
ZADD timeline:12345 1699123456 "post_789"  // Add post to timeline
ZREVRANGE timeline:12345 0 49              // Get top 50 posts
ZREM timeline:12345 "post_789"             // Remove post (deletion)
ZCARD timeline:12345                       // Count total posts
ZREMRANGEBYRANK timeline:12345 0 -1001     // Trim to keep top 1000
```

**Why Sorted Sets?**

- **O(log N) inserts**: Efficient for fanout-on-write
- **O(log N + M) range queries**: Fast feed retrieval
- **Atomic operations**: No race conditions during concurrent writes
- **Native sorting**: No application-level sorting needed

**Cache Management:**

- **TTL**: 24 hours (timelines expire if user inactive)
- **Max Size**: 1000 posts per timeline (trim older posts)
- **Eviction**: `volatile-lru` policy (evict least recently used)

#### Like Counters

To handle high-volume engagement, Instagram uses multiple Redis data structures working together:

```
Key: post:likes:{post_id}
Type: String (atomic counter)

Key: post:likers:{post_id}
Type: HyperLogLog (unique user count)

Key: post:{post_id}:likers
Type: Set (for duplicate detection)

Operations:
INCR post:likes:789                    // Increment like count
PFADD post:likers:789 user_123         // Track unique likers
SADD post:789:likers user_123          // Duplicate check
```

**Why Three Structures?**

1. **Counter (String)**: Fast atomic increments, can handle 10k ops/sec per key
2. **HyperLogLog**: Space-efficient unique count (0.81% error rate, only 12 KB per key)
3. **Set**: Exact duplicate detection (prevents users from liking twice)

**Hot Key Mitigation:**
For viral posts, counters are sharded across 100 Redis keys:

```
post:likes:789:0
post:likes:789:1
...
post:likes:789:99
```

This distributes 100k likes/sec across 100 shards = 1k likes/sec per shard.

### Object Storage Structure

```
s3://instagram-media/
├── images/
│   ├── original/{year}/{month}/{day}/{post_id}.jpg
│   ├── thumbnails/{post_id}_150x150.jpg
│   ├── medium/{post_id}_640x640.jpg
│   └── large/{post_id}_1080x1080.jpg
└── videos/
    ├── original/{year}/{month}/{day}/{post_id}.mp4
    ├── transcoded/{post_id}_720p.mp4
    └── thumbnails/{post_id}_thumb.jpg
```

---

## Detailed Component Design

### Media Upload Pipeline

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

### Feed Generation Strategy (Hybrid Fanout)

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
  // Get pre-computed fanout posts
  fanout_posts = redis.ZREVRANGE("timeline:{user_id}", 0, 49)
  
  // Get celebrity posts
  celebrity_posts = fetch_celebrity_posts(user_id)
  
  // Get ML-recommended posts
  recommended_posts = recommendation_engine.get_recommendations(user_id, 50)
  
  // Merge and rank
  all_posts = merge_and_rank(fanout_posts, celebrity_posts, recommended_posts)
  
  // Hydrate metadata
  return hydrate_posts(all_posts[0:49])
```

*See pseudocode.md::generate_feed() for detailed implementation.*

### Recommendation Engine Architecture

**Offline Training Pipeline** (runs daily):

1. **Data Collection**: Collect past 30 days of user interactions
2. **Feature Engineering**: Extract visual features (CNN embeddings), user features
3. **Model Training**: Train collaborative filtering + content-based model
4. **Model Deployment**: Deploy updated model to serving layer

**Online Serving** (real-time):

1. **Candidate Generation**: Fetch 1000 potential posts
2. **Feature Extraction**: Extract real-time features
3. **Ranking**: Run ML model to score each post
4. **Re-Ranking**: Apply business rules (diversity, freshness)

**Model Architecture**:

- **Two-Tower Model**: User tower + Post tower
- **User Tower**: Embeds user_id, recent_posts_liked, time_features
- **Post Tower**: Embeds post_id, visual_features (ResNet), text_features
- **Score**: Cosine similarity between embeddings

*See pseudocode.md::RecommendationEngine for implementation.*

### Real-Time Engagement System

**Challenge**: Handling 578k likes/comments per second with hot key problem.

**Solution: Multi-Layer Write Path**

1. **Layer 1: Redis Atomic Counters** (immediate response)
    - `INCR post:likes:{post_id}`
    - Returns in $< 1$ ms

2. **Layer 2: Cassandra Write** (durable persistence)
    - Async write to Cassandra
    - Partition key: `post_id`

3. **Layer 3: PostgreSQL Update** (periodic batch)
    - Every 5 minutes, aggregate Redis counters
    - Update `posts.like_count` in PostgreSQL

**Hot Key Mitigation**:

- **Sharding**: Shard Redis cluster by `post_id` hash
- **Local Aggregation**: Buffer likes in application memory, flush every 100ms
- **Rate Limiting**: Limit likes to 1 per user per second per post

*See pseudocode.md::like_post() for implementation.*

---

## Image Processing and Optimization

### Multi-Resolution Strategy

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

- Detect device type and screen DPI
- Serve AVIF to modern browsers, WebP to Chrome, JPEG to legacy
- Use BlurHash placeholders while loading

### Image Processing Pipeline

**Technologies**:

- **FFmpeg**: Video transcoding (H.264, H.265, VP9)
- **ImageMagick**: Image manipulation and format conversion
- **libvips**: Fast image processing (2-8x faster than ImageMagick)
- **TensorFlow**: Visual feature extraction (ResNet-50 CNN)

**Processing Steps**:

1. **Upload Validation** (100ms):
    - Check file signature (magic bytes) to prevent format spoofing
    - Validate dimensions: max 4096×4096 pixels
    - Validate size: max 20MB for images, 1GB for videos
    - Scan for malware using ClamAV

2. **EXIF Stripping** (50ms):
    - Remove all metadata (GPS coordinates, camera model, timestamps)
    - Protects user privacy (prevents location tracking)
    - Reduces file size by 5-10%

3. **Image Normalization** (200ms):
    - Auto-rotate based on EXIF orientation (if not stripped)
    - Color space conversion (sRGB normalization)
    - Aspect ratio preservation or cropping

4. **Multi-Resolution Generation** (parallel, 2 seconds):
   Generate 8 versions simultaneously on worker pool:
    - **Original** (backup): Stored as-is
    - **Large** (1080×1080): For web feed, JPEG quality 85
    - **Medium** (640×640): For mobile feed, JPEG quality 85
    - **Thumbnail** (150×150): For grid view, JPEG quality 80
    - **AVIF** (640×640): Modern format, quality 75 (50% smaller than JPEG)
    - **WebP** (640×640): Chrome/Android, quality 80 (30% smaller)
    - **BlurHash** (32×32): Placeholder while loading (1 KB)
    - **Story Format** (1080×1920): Vertical for Instagram Stories

5. **Compression Optimization** (500ms):
    - **JPEG**: Use progressive encoding, optimize Huffman tables
    - **WebP**: Lossy compression with quality 80
    - **AVIF**: Advanced compression (AV1-based), quality 75

6. **Filter Application** (300ms, optional):
    - Apply Instagram-style filters (Valencia, Clarendon, etc.)
    - Color grading, contrast adjustment, vignette
    - Implemented as GPU shaders for performance

7. **Visual Feature Extraction** (500ms):
    - Run ResNet-50 CNN model (pre-trained on ImageNet)
    - Extract 2048-dim embedding vector
    - Store in FAISS vector database for visual search

8. **CDN Distribution** (100ms):
    - Upload all 8 versions to S3
    - Trigger CDN cache pre-warming for popular accounts
    - Generate signed URLs for client access

**Performance Metrics**:

- **Total Time**: 3-5 seconds per 10MB image
- **Parallelization**: 8 versions processed in parallel
- **Throughput**: 100 workers × 20 images/min = 2,000 images/min
- **Peak Capacity**: Can scale to 500 workers = 10,000 images/min

**Error Handling**:

- If processing fails, retry up to 3 times with exponential backoff
- If all retries fail, post is marked as "processing failed" (user notified)
- Partial success: If 6/8 versions succeed, post is still published (missing versions generated later)

**Cost Optimization**:

- Use spot instances for workers (70% cost savings)
- Batch processing during off-peak hours for non-urgent posts
- Compress old images (> 1 year) to reduce storage costs

*See pseudocode.md::ImageProcessor for implementation.*

---

## Content Discovery and Visual Search

### Visual Similarity Search

Pinterest uses **Visual Embeddings** to find similar images.

**Architecture**:

1. **Image Encoder**: ResNet-50 extracts 2048-dim feature vector
2. **Vector Database**: Store embeddings in FAISS or Milvus
3. **Nearest Neighbor Search**: Find top-K similar images using cosine similarity

**Query Flow**:

```
User clicks "Find Similar"
→ Fetch embedding from vector DB
→ Run ANN search (Approximate Nearest Neighbor)
→ Retrieve top 100 similar post_ids
→ Apply ranking and personalization
→ Return top 50
```

**Performance**:

- Index size: 10B images × 2048 dims = 80 TB
- Query latency: $< 50$ ms using HNSW algorithm
- Index updates: Batch every hour

*See pseudocode.md::visual_search() for implementation.*

### Hashtag and Text Search

**Elasticsearch Index**:

```json
{
  "post_id": 12345,
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

**Ranking**:

- **Relevance**: BM25 score from Elasticsearch
- **Engagement**: Boost by like_count, comment_count
- **Freshness**: Boost recent posts
- **Personalization**: Boost posts from users you follow

---

## Feed Stitching and Ranking

### Feed Stitcher Logic

The **Feed Stitcher** combines:

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
  
  // Apply final ranking
  feed = ranking_model.score(feed, user_context)
  
  return feed.sort_by_score()
```

**Why This Mix?**

- **60% Follower**: Users expect content from people they follow
- **30% Recommended**: Discovery keeps users engaged
- **10% Ads**: Monetization

### Ranking Model

**Two-Stage Ranking**:

**Stage 1: Candidate Generation**:

- Fetch 1000 candidates from multiple sources
- Filter posts user already seen (bloom filter)
- Basic scoring: engagement_rate × freshness

**Stage 2: Ranking Model**:

- Input: user features, post features, context features
- Model: LightGBM or neural network
- Features:
    - **User**: past_likes, past_saves, follower_count
    - **Post**: like_count, comment_count, save_rate
    - **Cross**: user_post_similarity
    - **Context**: time_of_day, device_type
- Output: Probability user will engage

*See pseudocode.md::RankingModel for implementation.*

---

## Caching Strategy

### Multi-Layer Cache Architecture

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
4. **CDN Purge**: Send purge request to CDN API
5. **Timeline Update**: Remove post_id from affected timelines

*See pseudocode.md::invalidate_cache() for implementation.*

### CDN Configuration

**Multi-Tier CDN**:

1. **Origin**: S3 bucket with media files
2. **Regional PoP**: Cloudflare edge in 200+ cities
3. **Shield**: Mid-tier cache reduces origin load

**Cache Headers**:

```
Cache-Control: public, max-age=604800, immutable
```

**Why immutable?**

- Images never change after upload
- Allows aggressive caching
- Reduces CDN cost by 60%

---

## Availability and Fault Tolerance

### Multi-Region Architecture

**Active-Active Deployment**:

```
Region US-EAST                 Region EU-WEST
┌─────────────┐                ┌─────────────┐
│ API Gateway │                │ API Gateway │
└──────┬──────┘                └──────┬──────┘
       │                              │
┌──────▼──────┐                ┌──────▼──────┐
│  Feed Svc   │                │  Feed Svc   │
└──────┬──────┘                └──────┬──────┘
       │                              │
┌──────▼──────┐                ┌──────▼──────┐
│Redis Replica│◄───Async────►  │Redis Replica│
└──────┬──────┘  Replication   └──────┬──────┘
       │                              │
┌──────▼──────┐                ┌──────▼──────┐
│PostgreSQL   │◄──── Sync ────►│PostgreSQL   │
└─────────────┘   Replication  └─────────────┘
```

**Write Strategy**:

- **Posts**: Route to user's home region (by user_id hash)
- **Reads**: Serve from nearest region
- **Consistency**: Eventual consistency for timeline, strong for posts

### Failure Scenarios

| Failure                   | Impact                  | Mitigation                | Recovery Time |
|---------------------------|-------------------------|---------------------------|---------------|
| **API Gateway Down**      | Can't access app        | Load balancer fails over  | $< 1$ sec     |
| **Redis Cache Down**      | Slow feed load          | Fallback to PostgreSQL    | $< 5$ sec     |
| **PostgreSQL Shard Down** | 1/16th users can't post | Promote replica           | $< 30$ sec    |
| **CDN Down**              | Images don't load       | Fallback to origin S3     | Immediate     |
| **Recommendation Down**   | No discovery            | Serve follower posts only | Immediate     |

**Circuit Breaker Pattern**:

- If Recommendation Engine fails 50% of requests
- Open circuit breaker → stop calling it
- Serve cached recommendations

*See pseudocode.md::CircuitBreaker for implementation.*

---

## Bottlenecks and Optimizations

### 1. Feed Stitching Latency

**Problem**:
Combining follower posts, celebrity posts, recommended posts, and ads can add 100-150ms of latency if done sequentially.
With 347k feed requests/sec at peak, even small latency increases significantly impact user experience.

**Root Cause**:

- Fetching follower posts from Redis: 15ms
- Fetching celebrity posts from PostgreSQL: 30ms
- Fetching recommendations from ML engine: 50ms
- Fetching ads from ad service: 20ms
- **Sequential Total**: 115ms (too slow)

**Solution: Parallel Fetching**

```
parallel_start:
  task1 = async fetch_follower_posts(user_id)
  task2 = async fetch_celebrity_posts(user_id)
  task3 = async fetch_recommended_posts(user_id)
  task4 = async fetch_ads(user_id)
parallel_wait_all

total = max(task1, task2, task3, task4) + merge_time
```

**Result**:

- **Parallel Total**: max(15, 30, 50, 20) + 10ms merge = 60ms
- **Improvement**: 115ms → 60ms (48% faster)
- **Implementation**: Use async/await or Promise.all() pattern

**Additional Optimizations**:

- **Connection Pooling**: Reuse database connections (saves 5ms)
- **Request Coalescing**: Batch multiple feed requests together
- **Edge Caching**: Cache feed for 30 seconds for inactive users

---

### 2. Hot Key Problem (Viral Posts)

**Problem**:
Celebrity posts receive 1M+ likes in minutes, creating a Redis hot key that overwhelms a single Redis node.

**Example Scenario**:

- Taylor Swift posts a photo
- 1M likes in 10 minutes = 100k likes/second
- Single Redis key `post:likes:12345` receives 100k writes/sec
- Redis node maxes out at 10k writes/sec → **10x overload**

**Solution: Distributed Counters**

**Approach 1: Shard by User ID**

```
// Shard counter across 100 Redis keys
shard_id = hash(user_id) % 100
redis.INCR(f"post:likes:12345:{shard_id}")

// Read: Aggregate all shards
total = sum(redis.GET(f"post:likes:12345:{i}") for i in 0..99)
```

**Result**:

- Distributes 100k writes/sec across 100 shards
- Each shard handles 1k writes/sec (well within capacity)
- Read latency increases from 1ms to 10ms (acceptable)

**Approach 2: Local Buffering**

```
// Buffer likes in application memory
local_buffer[post_id] += 1

// Flush every 100ms
every 100ms:
  redis.INCRBY(f"post:likes:{post_id}", local_buffer[post_id])
  local_buffer.clear()
```

**Result**:

- Reduces Redis calls by 100x (100 likes → 1 Redis call)
- Risk: Up to 100ms of likes could be lost if app crashes (acceptable)

*See pseudocode.md::distributed_counter() for implementation.*

---

### 3. Media I/O Bottleneck

**Problem**:
Object storage handles 500B requests/day (5.7M requests/sec), which is expensive and can hit throughput limits.

**Cost Analysis**:

- **Without CDN**: 500B requests × $0.0004/request = $200M/month
- **With CDN (90% hit)**: 50B origin requests × $0.0004 = $20M/month + $150M CDN = $170M/month
- **Savings**: $30M/month

**Mitigation 1: Aggressive CDN Caching**

- **Cache Headers**: `Cache-Control: public, max-age=604800, immutable`
- **TTL**: 7 days (images never change after upload)
- **Hit Rate**: 90% at edge, 95% at shield
- **Result**: Only 5% of requests hit origin

**Mitigation 2: Pre-compute All Image Sizes**

- Generate all 8 versions during upload (not on-demand)
- Avoid on-the-fly resizing (CPU-intensive)
- Trade-off: 8x storage cost vs. infinite resize cost

**Mitigation 3: S3 Transfer Acceleration**

- Routes uploads to nearest AWS edge location
- Uses AWS backbone network (faster than internet)
- **Result**: 50% faster uploads from distant regions
- **Cost**: +$0.04/GB (worth it for user experience)

**Mitigation 4: Intelligent Pre-fetching**

- CDN pre-warms cache for popular accounts when they post
- Reduces cache miss latency for viral content
- **Example**: When celebrity posts, push to CDN immediately

---

### 4. Database Hotspots

**Problem**:
User profile reads (username, profile picture) hit PostgreSQL hard during feed generation. Each feed request needs 50
user profiles (one per post).

**Load Analysis**:

- 347k feed requests/sec × 50 users/feed = 17.3M user profile reads/sec
- PostgreSQL can handle ~100k reads/sec per shard
- Would need 173 PostgreSQL shards (expensive)

**Solution: Redis User Cache**

```
Key: user:{user_id}
Type: Hash
Fields: username, profile_image_url, bio, follower_count, verified
TTL: 1 hour

// Write-through cache
function get_user_profile(user_id):
  profile = redis.HGETALL(f"user:{user_id}")
  if profile is None:
    profile = postgres.SELECT("users", user_id)
    redis.HMSET(f"user:{user_id}", profile)
    redis.EXPIRE(f"user:{user_id}", 3600)
  return profile
```

**Result**:

- **Cache Hit Rate**: 95% (users are read frequently)
- **PostgreSQL Load**: 17.3M × 5% = 865k reads/sec → 9 shards (vs 173 shards)
- **Cost Savings**: $10M/year in database capacity

**Cache Invalidation**:

- When user updates profile → invalidate cache immediately
- When user deletes account → purge from cache
- TTL handles stale data for inactive changes

---

### 5. Recommendation Engine Latency

**Problem**:
ML inference for 1000 candidates takes 50ms, which is acceptable but could be faster.

**Optimization 1: Model Quantization**

- Reduce model precision from FP32 to INT8
- **Size**: 500 MB → 125 MB (4x smaller)
- **Latency**: 50ms → 30ms (40% faster)
- **Accuracy**: -0.2% (acceptable trade-off)

**Optimization 2: Batch Inference**

- Process multiple user requests in a single batch
- **Batch Size**: 16 users per batch
- **Latency**: 30ms per user (same as before)
- **Throughput**: 16x higher (GPU utilization 90%)

**Optimization 3: Caching**

- Cache recommended posts for 10 minutes
- Works well for users who refresh frequently
- **Hit Rate**: 30% (modest but helpful)

**Result**:

- **Total Latency**: 50ms → 20ms (60% faster)
- **Cost**: Same GPU capacity handles 16x more users

---

## Common Anti-Patterns

### ❌ **1. Storing Images in Database**

**Problem**: Storing media BLOBs in PostgreSQL/MySQL.

**Why Bad**:

- Database bloats to petabytes
- Slow queries
- Can't leverage CDN

**Solution**: Use Object Storage (S3) and store URLs.

```sql
-- ❌ Bad
CREATE TABLE posts (image_data BYTEA);

-- ✅ Good
CREATE TABLE posts (image_url TEXT);
```

### ❌ **2. Synchronous Media Processing**

**Problem**: Client waits for image processing.

**Why Bad**: Upload takes 30+ seconds, poor UX

**Solution**: Async processing with Kafka queue.

```
// ❌ Bad: Synchronous
Client → Upload → [Process 30s] → Success

// ✅ Good: Asynchronous
Client → Upload → Queue → Immediate Success
          ↓
       Background Worker
```

### ❌ **3. No Rate Limiting on Engagement**

**Problem**: Bots spam likes/comments.

**Solution**: Multi-layer rate limiting.

| Layer        | Limit         | Purpose              |
|--------------|---------------|----------------------|
| **Per User** | 100 likes/min | Prevent bot spamming |
| **Per IP**   | 500 likes/min | Prevent DDoS         |
| **Per Post** | 10k likes/sec | Prevent hot key      |

### ❌ **4. Not Handling Deleted Posts in Feed**

**Problem**: Deleted posts remain in feed caches.

**Solution**: Publish `post.deleted` event, invalidate timelines.

*See pseudocode.md::delete_post() for implementation.*

---

## Alternative Approaches

### Alternative 1: Full Fanout-on-Read

**Approach**: Generate feed on-demand.

**Pros**: No write amplification, always fresh

**Cons**: 500ms+ feed load, high DB load

**When to Use**: Small apps with $< 1$M users

### Alternative 2: Kafka Streams for Feed

**Approach**: Use Kafka Streams for real-time feed building.

**Pros**: Real-time updates, exactly-once semantics

**Cons**: Complex infrastructure

**When to Use**: If you have Kafka expertise

### Alternative 3: GraphQL Federation

**Approach**: Use GraphQL instead of REST.

**Pros**: Flexible queries, reduce over-fetching

**Cons**: Harder to cache, steeper learning curve

**When to Use**: If you need flexible client-driven queries

---

## Monitoring and Observability

### Key Metrics

| Metric                      | Target     | Alert Threshold | Impact                      |
|-----------------------------|------------|-----------------|-----------------------------|
| **Feed Load Latency (p95)** | $< 150$ ms | $> 300$ ms      | User experience degradation |
| **Upload Success Rate**     | $> 99.9$%  | $< 99.5$%       | Users can't post            |
| **CDN Hit Rate**            | $> 90$%    | $< 85$%         | Cost spike                  |
| **Recommendation Latency**  | $< 50$ ms  | $> 100$ ms      | Feed timeout                |
| **Redis Cache Hit**         | $> 85$%    | $< 75$%         | DB overload                 |
| **Kafka Consumer Lag**      | $< 1$ min  | $> 5$ min       | Delayed fanout              |

### Distributed Tracing

**OpenTelemetry Trace Example**:

```
Span: GET /feed (user_id=12345)
├─ Span: fetch_follower_posts [15ms]
├─ Span: fetch_recommended_posts [22ms]
├─ Span: stitch_and_rank [8ms]
└─ Span: hydrate_posts [12ms]

Total: 57ms
```

**Tools**: Jaeger, Zipkin, AWS X-Ray

### Logging Strategy

**Structured Logs**:

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "service": "feed-service",
  "user_id": 12345,
  "endpoint": "/feed",
  "latency_ms": 57,
  "cache_hit": true
}
```

**Log Aggregation**: ELK stack or Datadog

---

## Performance Optimization

### Database Query Optimization

**Slow Query**:

```sql
-- ❌ Bad: Full table scan
SELECT * FROM posts WHERE user_id = 12345 ORDER BY created_at DESC;
```

**Optimized**:

```sql
-- ✅ Good: Uses composite index
CREATE INDEX idx_user_created ON posts(user_id, created_at DESC);
SELECT post_id, caption FROM posts WHERE user_id = 12345 ORDER BY created_at DESC LIMIT 50;
```

### Redis Pipeline

**Without Pipeline**: 50 round trips = 50ms

**With Pipeline**: 1 round trip = 1ms

**Result**: 50x faster

### Compression

**Response Payload**: gzip compression (500KB → 50KB)

**Media Compression**:

- JPEG → WebP: 30% smaller
- WebP → AVIF: 50% smaller

---

## Trade-offs Summary

| What You Gain                             | What You Sacrifice                     |
|-------------------------------------------|----------------------------------------|
| ✅ **Ultra-low feed latency** ($< 150$ ms) | ❌ Eventual consistency                 |
| ✅ **Petabyte-scale media storage**        | ❌ High CDN costs ($\sim \$150$M/month) |
| ✅ **Highly personalized recommendations** | ❌ Expensive ML training                |
| ✅ **Real-time engagement**                | ❌ Hot key problem                      |
| ✅ **Global availability**                 | ❌ Complex replication                  |
| ✅ **Hybrid fanout**                       | ❌ Complex stitching logic              |
| ✅ **Visual search**                       | ❌ Large vector database (80TB)         |
| ✅ **Adaptive image delivery**             | ❌ Storage overhead (8+ versions)       |

---

## Real-World Examples

### Instagram (Meta)

**Architecture Highlights**:

- **TAO**: Custom graph database
- **Cassandra**: Feed storage
- **Memcache**: Massive cache layer
- **Haystack**: Custom photo storage

**Key Innovation**: ML embeddings for image content understanding

**Scale**: 2B+ users, 100M+ photos/day

### Pinterest

**Architecture Highlights**:

- **HBase**: Key-value storage
- **PinSearch**: Custom visual search
- **Pinball**: Custom workflow management

**Key Innovation**: Visual search using ResNet embeddings

**Scale**: 450M+ users, 200B+ pins

### TikTok

**Architecture Highlights**:

- **Best-in-class Recommendation Engine**
- **For You Page**: 100% algorithmic
- **Optimized Video Pipeline**

**Key Innovation**: Personalization within 5-10 videos for new users

**Scale**: 1B+ users, 100B+ video views/day

---

## References

### Related Chapters

- **[3.2.1 Twitter Timeline](../3.2.1-twitter-timeline/README.md)** - Fanout strategies
- **[3.4.2 News Feed](../3.4.2-news-feed/README.md)** - Feed ranking
- **[3.4.4 Recommendation System](../3.4.4-recommendation-system/README.md)** - ML recommendations
- **[2.1.11 Redis Deep Dive](../../02-components/2.1-databases/2.1.11-redis-deep-dive.md)** - Caching
- **[2.1.9 Cassandra Deep Dive](../../02-components/2.1-databases/2.1.9-cassandra-deep-dive.md)** - Time-series data
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Event streaming

### External Resources

- **Instagram Engineering Blog**: https://instagram-engineering.com/
- **Pinterest Engineering Blog**: https://medium.com/pinterest-engineering
- **"Scaling Instagram" by Mike Krieger**: YouTube
- **"TAO: Facebook's Distributed Data Store" Paper**: USENIX ATC 2013
- **"Haystack: Facebook's Photo Storage" Paper**: OSDI 2010
