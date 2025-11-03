# Instagram/Pinterest Feed - High-Level Design

## Table of Contents

1. [Overall System Architecture](#1-overall-system-architecture)
2. [Media Upload Pipeline](#2-media-upload-pipeline)
3. [Feed Generation Architecture](#3-feed-generation-architecture)
4. [Hybrid Fanout Strategy](#4-hybrid-fanout-strategy)
5. [Recommendation Engine Architecture](#5-recommendation-engine-architecture)
6. [Engagement System Architecture](#6-engagement-system-architecture)
7. [Multi-Layer Caching Architecture](#7-multi-layer-caching-architecture)
8. [Image Processing Pipeline](#8-image-processing-pipeline)
9. [Visual Search Architecture](#9-visual-search-architecture)
10. [Multi-Region Deployment](#10-multi-region-deployment)
11. [Database Sharding Strategy](#11-database-sharding-strategy)
12. [CDN Distribution Architecture](#12-cdn-distribution-architecture)
13. [Feed Stitcher Component](#13-feed-stitcher-component)
14. [Monitoring and Observability](#14-monitoring-and-observability)

---

## 1. Overall System Architecture

**Flow Explanation:**

This diagram shows the complete end-to-end architecture of the Instagram/Pinterest feed system, from client request to
data storage layers.

**Components:**

1. **Client Layer**: Mobile and web applications
2. **API Gateway**: Rate limiting, authentication, and routing
3. **Service Layer**: Upload, Feed, and Engagement microservices
4. **Processing Layer**: Media pipeline and recommendation engine
5. **Storage Layer**: Object store (S3), Redis cache, and PostgreSQL
6. **CDN**: Global content delivery network

**Benefits:**

- **Scalability**: Each layer scales independently
- **Separation of Concerns**: Clear responsibility boundaries
- **High Availability**: Multiple redundant components

**Trade-offs:**

- **Complexity**: Many moving parts require coordination
- **Latency**: Multiple hops between layers
- **Cost**: Maintaining multiple infrastructure layers

```mermaid
graph TB
    subgraph "Client Layer"
        MobileApp[Mobile App]
        WebApp[Web App]
    end

    subgraph "Edge Layer"
        CDN[CDN Network<br/>Cloudflare]
        APIGateway[API Gateway<br/>Rate Limiting + Auth]
    end

    subgraph "Application Services"
        UploadSvc[Upload Service]
        FeedSvc[Feed Service]
        EngageSvc[Engagement Service]
        SearchSvc[Search Service]
    end

    subgraph "Processing Layer"
        MediaPipeline[Media Pipeline<br/>FFmpeg + ImageMagick]
        FeedBuilder[Feed Builder<br/>Kafka Consumer]
        RecoEngine[Recommendation Engine<br/>ML Models]
    end

    subgraph "Cache Layer"
        RedisTimeline[Redis Timeline Cache<br/>Sorted Sets]
        RedisPost[Redis Post Cache<br/>Hashes]
        RedisCounter[Redis Counters<br/>Atomic INCR]
    end

    subgraph "Storage Layer"
        S3[Object Store<br/>S3/GCS]
        Postgres[(PostgreSQL<br/>Sharded)]
        Cassandra[(Cassandra<br/>Time-Series)]
        VectorDB[(Vector DB<br/>FAISS)]
    end

    subgraph "Message Queue"
        Kafka[Kafka<br/>Event Streaming]
    end

    MobileApp --> CDN
    WebApp --> CDN
    MobileApp --> APIGateway
    WebApp --> APIGateway
    APIGateway --> UploadSvc
    APIGateway --> FeedSvc
    APIGateway --> EngageSvc
    APIGateway --> SearchSvc
    UploadSvc --> Kafka
    Kafka --> MediaPipeline
    MediaPipeline --> S3
    MediaPipeline --> Postgres
    FeedSvc --> RedisTimeline
    FeedSvc --> RedisPost
    FeedSvc --> RecoEngine
    RedisTimeline --> Postgres
    EngageSvc --> RedisCounter
    RedisCounter --> Cassandra
    Kafka --> FeedBuilder
    FeedBuilder --> RedisTimeline
    RecoEngine --> VectorDB
    RecoEngine --> Postgres
    CDN --> S3
    SearchSvc --> VectorDB
```

---

## 2. Media Upload Pipeline

**Flow Explanation:**

This diagram illustrates the asynchronous media upload and processing pipeline, which separates upload acknowledgment
from heavy processing tasks.

**Steps:**

1. **Client Request**: User uploads media file (image or video)
2. **Pre-signed URL**: Upload service generates S3 pre-signed URL
3. **Direct Upload**: Client uploads directly to S3 (bypasses server)
4. **Event Trigger**: S3 triggers Lambda or publishes to Kafka
5. **Async Processing**: Worker pool processes media (resize, transcode)
6. **Multi-Resolution Generation**: Create 8+ versions of each image
7. **Metadata Storage**: Save post metadata and URLs to PostgreSQL
8. **Timeline Fanout**: Trigger fanout to followers' timelines

**Performance:**

- **Upload Latency**: Client receives success in <2 seconds
- **Processing Time**: Background processing takes 3-10 seconds
- **Throughput**: 17k uploads/sec peak capacity

**Benefits:**

- **Fast Response**: User doesn't wait for processing
- **Scalability**: Worker pool scales independently
- **Fault Tolerance**: Failed jobs can be retried

**Trade-offs:**

- **Eventual Consistency**: Post visible before all thumbnails ready
- **Complexity**: Need job queue management
- **Cost**: More infrastructure components

```mermaid
graph LR
    subgraph "Client"
        User[User]
    end

    subgraph "Upload Service"
        UploadAPI[Upload API]
        PreSigned[Pre-signed URL<br/>Generator]
    end

    subgraph "Object Storage"
        S3[S3 Bucket<br/>Original Media]
    end

    subgraph "Message Queue"
        Kafka[Kafka Queue<br/>media.uploaded]
    end

    subgraph "Worker Pool"
        Worker1[Worker 1]
        Worker2[Worker 2]
        Worker3[Worker N]
    end

    subgraph "Image Processing"
        ImageMagick[ImageMagick<br/>libvips]
        FFmpeg[FFmpeg<br/>Video Transcode]
        MLModel[CNN Model<br/>Feature Extract]
    end

    subgraph "Output Storage"
        S3Processed[S3<br/>Processed Media]
        Postgres[(PostgreSQL<br/>Metadata)]
        VectorDB[(Vector DB<br/>Embeddings)]
    end

    User -->|1 . Upload Request| UploadAPI
    UploadAPI -->|2 . Generate URL| PreSigned
    PreSigned -->|3 . Pre - signed URL| User
    User -->|4 . Direct Upload| S3
    S3 -->|5 . Trigger Event| Kafka
    Kafka --> Worker1
    Kafka --> Worker2
    Kafka --> Worker3
    Worker1 --> ImageMagick
    Worker2 --> FFmpeg
    Worker3 --> MLModel
    ImageMagick -->|8 Versions| S3Processed
    FFmpeg -->|Multiple Bitrates| S3Processed
    MLModel --> VectorDB
    Worker1 -->|7 . Save Metadata| Postgres
    Postgres -->|8 . Trigger Fanout| Kafka
```

---

## 3. Feed Generation Architecture

**Flow Explanation:**

This diagram shows how the personalized feed is generated by combining multiple data sources and applying ML-based
ranking.

**Steps:**

1. **User Request**: User opens app and requests feed
2. **Feed Service**: Routes to appropriate feed generation logic
3. **Parallel Fetch**: Simultaneously fetch from 3 sources
    - Follower posts from Redis timeline cache
    - Celebrity posts from database (fanout-on-read)
    - Recommended posts from ML recommendation engine
4. **Feed Stitcher**: Merges all sources with configurable ratio (60/30/10)
5. **Ranking Model**: Applies final ML model for personalized ordering
6. **Hydration**: Fetches full post details from cache or database
7. **Response**: Returns top 50 posts to client

**Performance:**

- **Total Latency**: <150ms end-to-end
- **Cache Hit Rate**: 85% for timeline, 70% for posts
- **Throughput**: 347k reads/sec peak

**Benefits:**

- **Low Latency**: Pre-computed timelines + caching
- **Personalized**: ML-driven recommendations
- **Fresh Content**: Mix of social graph and discovery

**Trade-offs:**

- **Eventual Consistency**: Timeline may be seconds stale
- **Complexity**: Multiple data sources to merge
- **Cost**: Expensive ML inference

```mermaid
graph TB
    subgraph "Client Request"
        Client[Mobile/Web Client]
    end

    subgraph "Feed Service"
        FeedAPI[Feed API]
        Stitcher[Feed Stitcher]
        Ranker[Ranking Model<br/>ML Inference]
        Hydrator[Post Hydrator]
    end

    subgraph "Data Sources"
        RedisTimeline[(Redis Timeline<br/>Follower Posts)]
        DBCelebrity[(PostgreSQL<br/>Celebrity Posts)]
        RecoEngine[Recommendation<br/>Engine]
        AdService[Ad Service]
    end

    subgraph "Post Details"
        RedisPostCache[(Redis Post Cache)]
        PostgresPost[(PostgreSQL<br/>Post Metadata)]
    end

    Client -->|GET /feed| FeedAPI
    FeedAPI -->|1 . Parallel Fetch| RedisTimeline
    FeedAPI -->|2 . Parallel Fetch| DBCelebrity
    FeedAPI -->|3 . Parallel Fetch| RecoEngine
    FeedAPI -->|4 . Parallel Fetch| AdService
    RedisTimeline -->|50 post_ids| Stitcher
    DBCelebrity -->|20 post_ids| Stitcher
    RecoEngine -->|50 post_ids| Stitcher
    AdService -->|10 ad_ids| Stitcher
    Stitcher -->|Merged List| Ranker
    Ranker -->|Ranked List| Hydrator
    Hydrator -->|Get Details| RedisPostCache
    RedisPostCache -->|Cache Miss| PostgresPost
    Hydrator -->|Top 50 Posts| Client
    style Stitcher fill: #ffeb3b
    style Ranker fill: #ff9800
```

---

## 4. Hybrid Fanout Strategy

**Flow Explanation:**

This diagram illustrates the hybrid fanout strategy that combines fanout-on-write (for regular users) and
fanout-on-read (for celebrities) to optimize write amplification and read latency.

**Decision Logic:**

- **Regular User** (< 10K followers): Fanout-on-write
    - Pre-compute timelines for all followers
    - Write post_id to Redis timeline cache
    - Guarantees instant feed updates

- **Celebrity** (> 10K followers): Fanout-on-read
    - Don't pre-compute timelines
    - Fetch posts at read time from database
    - Prevents write amplification (1 post → 10M writes)

**Steps:**

1. User posts new content
2. Check follower count
3. If < 10K: Fanout-on-write path
4. If > 10K: Skip fanout, rely on fanout-on-read

**Benefits:**

- **Write Efficiency**: Prevents celebrity write storms
- **Read Performance**: Pre-computed timelines for most users
- **Scalability**: Handles both high-follower and low-follower users

**Trade-offs:**

- **Complexity**: Two different code paths
- **Consistency**: Celebrity posts may appear slightly delayed
- **Threshold Management**: 10K threshold needs tuning

```mermaid
graph TD
    UserPost[User Creates Post]
    CheckFollowers{Follower Count<br/>< 10K?}

    subgraph "Fanout-on-Write Path"
        FanoutWrite[Fanout Service]
        GetFollowers[Get Follower List<br/>from PostgreSQL]
        KafkaFanout[Kafka: post.created]
        ConsumerPool[Consumer Pool<br/>100 Workers]
        UpdateTimelines[Update Follower<br/>Timelines in Redis]
    end

    subgraph "Fanout-on-Read Path"
        SkipFanout[Skip Pre-computation]
        StoreMetadata[Store in PostgreSQL<br/>Only]
        ReadTimeQuery[Query at Read Time<br/>by Followers]
    end

    subgraph "Storage"
        RedisTimeline[(Redis Timeline Cache<br/>ZADD timeline:user_id)]
        PostgresPost[(PostgreSQL<br/>posts table)]
    end

    UserPost --> CheckFollowers
    CheckFollowers -->|Yes: Regular User| FanoutWrite
    FanoutWrite --> GetFollowers
    GetFollowers --> KafkaFanout
    KafkaFanout --> ConsumerPool
    ConsumerPool --> UpdateTimelines
    UpdateTimelines --> RedisTimeline
    CheckFollowers -->|No: Celebrity| SkipFanout
    SkipFanout --> StoreMetadata
    StoreMetadata --> PostgresPost
    PostgresPost --> ReadTimeQuery
    style FanoutWrite fill: #4caf50
    style SkipFanout fill: #ff5722
```

---

## 5. Recommendation Engine Architecture

**Flow Explanation:**

This diagram shows the two-stage recommendation architecture: offline training pipeline and online serving
infrastructure.

**Offline Training (Batch):**

1. **Data Collection**: Collect 30 days of user interactions from data warehouse
2. **Feature Engineering**: Extract user features, post features, visual features
3. **Model Training**: Train two-tower model on Spark/GPU cluster
4. **Model Validation**: Evaluate on holdout set
5. **Model Deployment**: Deploy to TensorFlow Serving

**Online Serving (Real-time):**

1. **Candidate Generation**: Fetch 1000 candidate posts from multiple sources
2. **Feature Extraction**: Extract real-time user context
3. **Batch Inference**: Score all candidates with ML model
4. **Re-ranking**: Apply business rules (diversity, freshness)
5. **Return Top-K**: Return top 50 posts

**Performance:**

- **Training Time**: 8-12 hours on 100 GPUs
- **Inference Latency**: <50ms for 1000 candidates
- **Model Update Frequency**: Daily

**Benefits:**

- **Personalization**: Tailored to individual user preferences
- **Discovery**: Helps users find new content beyond social graph
- **Engagement**: Increases time spent on platform

**Trade-offs:**

- **Staleness**: Model trained on yesterday's data
- **Cost**: Expensive GPU training and inference
- **Complexity**: ML pipeline requires data science expertise

```mermaid
graph TB
    subgraph "Offline Training Pipeline"
        DataWarehouse[(Data Warehouse<br/>Hive/BigQuery)]
        FeatureEng[Feature Engineering<br/>Spark Jobs]
        ModelTrain[Model Training<br/>TensorFlow/PyTorch]
        ModelVal[Model Validation<br/>A/B Testing]
        ModelDeploy[Model Deployment<br/>TF Serving]
    end

    subgraph "Online Serving Infrastructure"
        FeedRequest[Feed Request]
        CandidateGen[Candidate Generation<br/>1000 Posts]
        FeatureExtract[Real-time Feature<br/>Extraction]
        MLInference[ML Model Inference<br/>Two-Tower Model]
        Reranker[Re-ranking<br/>Business Rules]
        TopK[Return Top 50]
    end

    subgraph "Candidate Sources"
        PopularPosts[(Popular Posts<br/>Last 24h)]
        SimilarUsers[(Similar Users<br/>Collaborative)]
        VisualSimilar[(Visual Similarity<br/>Vector DB)]
        TrendingHashtags[(Trending<br/>Hashtags)]
    end

    subgraph "Feature Stores"
        UserFeatures[(User Features<br/>Redis)]
        PostFeatures[(Post Features<br/>Redis)]
        ContextFeatures[Context<br/>time, device]
    end

    DataWarehouse --> FeatureEng
    FeatureEng --> ModelTrain
    ModelTrain --> ModelVal
    ModelVal --> ModelDeploy
    FeedRequest --> CandidateGen
    CandidateGen --> PopularPosts
    CandidateGen --> SimilarUsers
    CandidateGen --> VisualSimilar
    CandidateGen --> TrendingHashtags
    PopularPosts --> FeatureExtract
    SimilarUsers --> FeatureExtract
    VisualSimilar --> FeatureExtract
    TrendingHashtags --> FeatureExtract
    FeatureExtract --> UserFeatures
    FeatureExtract --> PostFeatures
    FeatureExtract --> ContextFeatures
    UserFeatures --> MLInference
    PostFeatures --> MLInference
    ContextFeatures --> MLInference
    ModelDeploy --> MLInference
    MLInference --> Reranker
    Reranker --> TopK
    style ModelTrain fill: #9c27b0
    style MLInference fill: #ff9800
```

---

## 6. Engagement System Architecture

**Flow Explanation:**

This diagram shows the multi-layer engagement system that handles likes, comments, and shares with high throughput and
low latency.

**Write Path (Like Action):**

1. **User Clicks Like**: Client sends POST /like request
2. **API Gateway**: Rate limiting and authentication
3. **Engagement Service**: Validates request (not already liked)
4. **Layer 1 - Redis**: Atomic counter increment (immediate response <1ms)
5. **Layer 2 - Cassandra**: Durable write (async, within 10ms)
6. **Layer 3 - PostgreSQL**: Batch update every 5 minutes (aggregated)

**Read Path:**

1. **Get Like Count**: Fetch from Redis counter
2. **Cache Miss**: Fall back to PostgreSQL
3. **Hydrate User List**: Fetch recent likers from Cassandra

**Performance:**

- **Write Latency**: <5ms (Redis response)
- **Write Throughput**: 578k likes/sec peak
- **Cache Hit Rate**: 90% for counters

**Benefits:**

- **Immediate Feedback**: User sees like instantly
- **Durability**: Multiple layers ensure no data loss
- **Scalability**: Redis cluster handles high write load

**Trade-offs:**

- **Eventual Consistency**: PostgreSQL count may be minutes stale
- **Hot Key Problem**: Viral posts need distributed counters
- **Cost**: Three storage layers to maintain

```mermaid
graph TB
    subgraph "Client Action"
        User[User Clicks Like]
    end

    subgraph "API Layer"
        APIGateway[API Gateway<br/>Rate Limit Check]
        EngageAPI[Engagement API]
    end

    subgraph "Write Path - Multi-Layer"
        ValidationSvc[Validation Service<br/>Check Duplicate]

        subgraph "Layer 1: Immediate Response"
            RedisCounter[Redis Counter<br/>INCR post:likes:123]
            RedisHLL[Redis HyperLogLog<br/>PFADD post:likers:123]
        end

        subgraph "Layer 2: Durable Write"
            CassandraWrite[Cassandra Write<br/>likes table]
        end

        subgraph "Layer 3: Batch Aggregation"
            KafkaEvent[Kafka Event<br/>like.created]
            BatchWorker[Batch Worker<br/>5 min window]
            PostgresUpdate[(PostgreSQL<br/>UPDATE like_count)]
        end
    end

    subgraph "Read Path"
        GetLikeCount[Get Like Count]
        RedisRead[Redis GET<br/>post:likes:123]
        CacheMiss{Cache Miss?}
        PostgresRead[(PostgreSQL<br/>SELECT like_count)]
    end

    User --> APIGateway
    APIGateway --> EngageAPI
    EngageAPI --> ValidationSvc
    ValidationSvc --> RedisCounter
    ValidationSvc --> RedisHLL
    RedisCounter -->|Async| CassandraWrite
    CassandraWrite -->|Event| KafkaEvent
    KafkaEvent --> BatchWorker
    BatchWorker --> PostgresUpdate
    GetLikeCount --> RedisRead
    RedisRead --> CacheMiss
    CacheMiss -->|Yes| PostgresRead
    CacheMiss -->|No| User
    PostgresRead --> User
    style RedisCounter fill: #4caf50
    style CassandraWrite fill: #ff9800
    style PostgresUpdate fill: #2196f3
```

---

## 7. Multi-Layer Caching Architecture

**Flow Explanation:**

This diagram illustrates the multi-tier caching strategy that reduces latency and backend load through progressive
caching layers.

**Layers:**

1. **L1 - Client Cache**: Mobile app cache (30% hit rate, 1 hour TTL)
2. **L2 - CDN Cache**: Cloudflare edge (90% hit rate, 7 days TTL)
3. **L3 - Redis Timeline Cache**: Feed post IDs (85% hit rate, 24 hours TTL)
4. **L4 - Redis Post Cache**: Post metadata (70% hit rate, 1 hour TTL)
5. **L5 - Database**: PostgreSQL source of truth

**Request Flow:**

1. Client checks local cache → hit or miss
2. If miss, request to CDN → hit or miss
3. If miss, request to Redis → hit or miss
4. If miss, query PostgreSQL → cache result

**Cache Invalidation:**

- **Post Update**: Invalidate L4 + publish event
- **Post Delete**: Invalidate L3, L4 + purge CDN
- **User Update**: Invalidate user cache

**Performance:**

- **Effective Hit Rate**: 95%+ (combined layers)
- **Average Latency**: 50ms (mostly served from cache)
- **Database Load**: Reduced by 20x

**Benefits:**

- **Ultra-Low Latency**: Most requests served from edge
- **Reduced Backend Load**: 95% traffic never hits database
- **Cost Efficiency**: Fewer database queries

**Trade-offs:**

- **Cache Consistency**: Invalidation complexity
- **Memory Cost**: Large Redis clusters
- **Debugging**: Harder to trace cache bugs

```mermaid
graph TB
    subgraph "Client Layer - L1"
        MobileCache[Mobile App Cache<br/>TTL: 1 hour<br/>Hit Rate: 30%]
    end

    subgraph "Edge Layer - L2"
        CDNCache[CDN Cache<br/>Cloudflare<br/>TTL: 7 days<br/>Hit Rate: 90%]
    end

    subgraph "Application Cache - L3 & L4"
        RedisTimeline[Redis Timeline Cache<br/>Sorted Sets<br/>TTL: 24 hours<br/>Hit Rate: 85%]
        RedisPost[Redis Post Cache<br/>Hashes<br/>TTL: 1 hour<br/>Hit Rate: 70%]
    end

    subgraph "Database Layer - L5"
        PostgreSQL[(PostgreSQL<br/>Source of Truth<br/>Permanent Storage)]
    end

    subgraph "Cache Invalidation"
        KafkaInvalidate[Kafka Topic<br/>cache.invalidate]
        InvalidateWorker[Invalidation Worker]
        CDNPurge[CDN Purge API]
    end

    User[User Request]
    User -->|1 . Check Cache| MobileCache
    MobileCache -->|Miss| CDNCache
    CDNCache -->|Miss| RedisTimeline
    RedisTimeline -->|Miss| RedisPost
    RedisPost -->|Miss| PostgreSQL
    PostgreSQL -->|Write - Through| RedisPost
    PostgreSQL -->|Publish Event| KafkaInvalidate
    KafkaInvalidate --> InvalidateWorker
    InvalidateWorker --> RedisTimeline
    InvalidateWorker --> RedisPost
    InvalidateWorker --> CDNPurge
    CDNPurge --> CDNCache
    style MobileCache fill: #e3f2fd
    style CDNCache fill: #bbdefb
    style RedisTimeline fill: #90caf9
    style RedisPost fill: #64b5f6
    style PostgreSQL fill: #2196f3
```

---

## 8. Image Processing Pipeline

**Flow Explanation:**

This diagram shows the complete image processing workflow from upload to multi-resolution storage and CDN distribution.

**Processing Steps:**

1. **Upload**: Client uploads original high-res image
2. **Validation**: Check file type, size, dimensions
3. **EXIF Stripping**: Remove GPS and privacy metadata
4. **Parallel Processing**: Generate 8 versions simultaneously
    - Original (backup)
    - Feed Large (1080x1080)
    - Feed Medium (640x640)
    - Thumbnail (150x150)
    - AVIF format (modern browsers)
    - WebP format (Chrome)
    - BlurHash (placeholder)
5. **Filter Application**: Apply Instagram-style filters
6. **Visual Feature Extraction**: Run CNN model for recommendations
7. **Upload to S3**: Store all versions
8. **CDN Distribution**: Push to CDN origin

**Performance:**

- **Processing Time**: 3-5 seconds per image
- **Parallelization**: 8 workers per image
- **Throughput**: 5,780 images/sec

**Benefits:**

- **Multi-Device Support**: Optimized sizes for each device
- **Bandwidth Savings**: Serve smallest appropriate size
- **Fast Load Times**: Progressive loading with BlurHash

**Trade-offs:**

- **Storage Cost**: 8x more storage per image
- **Processing Cost**: CPU-intensive transformations
- **Complexity**: Managing multiple versions

```mermaid
graph TB
    subgraph "Upload"
        Client[Client Upload]
        S3Original[S3 Original Bucket]
    end

    subgraph "Processing Orchestrator"
        JobQueue[Kafka Job Queue]
        Worker[Worker Pool<br/>100 Instances]
    end

    subgraph "Image Processing Tasks"
        Validate[Validation<br/>Type/Size/Dims]
        EXIFStrip[EXIF Stripping<br/>Privacy]

        subgraph "Parallel Resize"
            GenLarge[Large<br/>1080x1080]
            GenMedium[Medium<br/>640x640]
            GenThumb[Thumbnail<br/>150x150]
            GenAVIF[AVIF<br/>Modern]
            GenWebP[WebP<br/>Chrome]
            GenBlur[BlurHash<br/>Placeholder]
        end

        FilterApply[Apply Filters<br/>Instagram Style]
        FeatureExtract[CNN Model<br/>Visual Features]
    end

    subgraph "Output Storage"
        S3Processed[S3 Processed Bucket<br/>8 Versions]
        VectorDB[(Vector DB<br/>Embeddings)]
        PostgresMetadata[(PostgreSQL<br/>Metadata)]
    end

    subgraph "Distribution"
        CDN[CDN Origin<br/>Cloudflare]
    end

    Client --> S3Original
    S3Original --> JobQueue
    JobQueue --> Worker
    Worker --> Validate
    Validate --> EXIFStrip
    EXIFStrip --> GenLarge
    EXIFStrip --> GenMedium
    EXIFStrip --> GenThumb
    EXIFStrip --> GenAVIF
    EXIFStrip --> GenWebP
    EXIFStrip --> GenBlur
    GenLarge --> FilterApply
    GenMedium --> FilterApply
    FilterApply --> FeatureExtract
    GenLarge --> S3Processed
    GenMedium --> S3Processed
    GenThumb --> S3Processed
    GenAVIF --> S3Processed
    GenWebP --> S3Processed
    GenBlur --> S3Processed
    FeatureExtract --> VectorDB
    Worker --> PostgresMetadata
    S3Processed --> CDN
    style Worker fill: #ff9800
```

---

## 9. Visual Search Architecture

**Flow Explanation:**

This diagram shows the visual similarity search system that enables users to find images similar to a given image using
deep learning embeddings.

**Indexing Flow (Offline):**

1. **Image Upload**: User uploads new image
2. **Feature Extraction**: ResNet-50 CNN extracts 2048-dim embedding
3. **Vector Storage**: Store embedding in FAISS vector database
4. **Index Update**: Update nearest neighbor index

**Search Flow (Online):**

1. **User Query**: User clicks "Find Similar" on an image
2. **Fetch Embedding**: Retrieve pre-computed embedding from vector DB
3. **ANN Search**: Run approximate nearest neighbor search using HNSW algorithm
4. **Retrieve Top-K**: Get top 100 similar post IDs
5. **Apply Filters**: Remove seen posts, apply personalization
6. **Rank Results**: Apply engagement and freshness ranking
7. **Return**: Send top 50 similar posts to user

**Performance:**

- **Index Size**: 10B images × 2048 dims = 80 TB
- **Search Latency**: <50ms for 100 nearest neighbors
- **Accuracy**: 95% recall@100

**Benefits:**

- **Visual Discovery**: Find similar content without text
- **User Engagement**: Increases time spent browsing
- **Content Understanding**: Understands image semantics

**Trade-offs:**

- **Storage Cost**: 80TB vector index
- **Index Update Lag**: Eventual consistency (1-hour delay)
- **Compute Cost**: CNN inference on all images

```mermaid
graph TB
    subgraph "Indexing Pipeline - Offline"
        ImageUpload[New Image Upload]
        CNNModel[ResNet-50 CNN<br/>Feature Extractor]
        Embedding[2048-dim Embedding<br/>Vector]
        IndexUpdate[FAISS Index<br/>Update]
    end

    subgraph "Search Pipeline - Online"
        UserQuery[User: Find Similar]
        FetchEmbed[Fetch Embedding<br/>from Vector DB]
        ANNSearch[ANN Search<br/>HNSW Algorithm]
        RetrieveTopK[Retrieve Top 100<br/>Similar Posts]
        ApplyFilters[Apply Filters<br/>Remove Seen]
        Personalize[Personalization<br/>Ranking Model]
        ReturnResults[Return Top 50]
    end

    subgraph "Storage"
        VectorDB[(Vector Database<br/>FAISS/Milvus<br/>10B vectors)]
        PostgreSQL[(PostgreSQL<br/>Post Metadata)]
    end

    subgraph "Serving Infrastructure"
        VectorSearchAPI[Vector Search API]
        LoadBalancer[Load Balancer]
        SearchReplicas[Search Replicas<br/>10 Instances]
    end

    ImageUpload --> CNNModel
    CNNModel --> Embedding
    Embedding --> VectorDB
    Embedding --> IndexUpdate
    UserQuery --> VectorSearchAPI
    VectorSearchAPI --> LoadBalancer
    LoadBalancer --> SearchReplicas
    SearchReplicas --> FetchEmbed
    FetchEmbed --> VectorDB
    FetchEmbed --> ANNSearch
    ANNSearch --> RetrieveTopK
    RetrieveTopK --> ApplyFilters
    ApplyFilters --> Personalize
    Personalize --> PostgreSQL
    Personalize --> ReturnResults
    style CNNModel fill: #9c27b0
    style ANNSearch fill: #ff9800
```

---

## 10. Multi-Region Deployment

**Flow Explanation:**

This diagram shows the global multi-region active-active deployment strategy for low latency and high availability.

**Architecture:**

- **3 Regions**: US-East, EU-West, Asia-Pacific
- **Geo-Routing**: Route users to nearest region via DNS
- **Data Replication**:
    - PostgreSQL: Synchronous replication (strong consistency)
    - Redis: Asynchronous replication (eventual consistency)
    - S3: Cross-region replication
- **Write Strategy**: Route writes to user's home region (by user_id hash)
- **Read Strategy**: Serve from nearest region

**Failover:**

1. **Health Checks**: Monitor region health every 10 seconds
2. **Auto-Failover**: Route53 detects failure and redirects traffic
3. **Replica Promotion**: Promote read replica to primary
4. **Recovery Time**: <30 seconds

**Performance:**

- **Latency Reduction**: 50-100ms faster for distant users
- **Availability**: 99.99% (survives single region failure)
- **Data Loss**: RPO < 1 second (synchronous replication)

**Benefits:**

- **Low Latency**: Users served from nearby region
- **High Availability**: Survives regional outages
- **Data Sovereignty**: Store data in local regions

**Trade-offs:**

- **Complexity**: Multi-region coordination
- **Cost**: 3x infrastructure in each region
- **Replication Lag**: Cross-region eventual consistency

```mermaid
graph TB
    subgraph "Global DNS"
        Route53[Route53<br/>Geo-Routing]
    end

    subgraph "Region: US-EAST"
        USLB[Load Balancer]
        USApp[App Servers<br/>Feed/Upload/Engage]
        USRedis[(Redis Cluster<br/>Primary)]
        USPostgres[(PostgreSQL<br/>Primary)]
        USS3[S3 Bucket<br/>Media]
    end

    subgraph "Region: EU-WEST"
        EULB[Load Balancer]
        EUApp[App Servers<br/>Feed/Upload/Engage]
        EURedis[(Redis Cluster<br/>Replica)]
        EUPostgres[(PostgreSQL<br/>Replica)]
        EUS3[S3 Bucket<br/>Media]
    end

    subgraph "Region: ASIA-PACIFIC"
        APLB[Load Balancer]
        APApp[App Servers<br/>Feed/Upload/Engage]
        APRedis[(Redis Cluster<br/>Replica)]
        APPostgres[(PostgreSQL<br/>Replica)]
        APS3[S3 Bucket<br/>Media]
    end

    subgraph "Users"
        USUser[US User]
        EUUser[EU User]
        APUser[Asia User]
    end

    USUser --> Route53
    EUUser --> Route53
    APUser --> Route53
    Route53 -->|Nearest| USLB
    Route53 -->|Nearest| EULB
    Route53 -->|Nearest| APLB
    USLB --> USApp
    USApp --> USRedis
    USApp --> USPostgres
    USApp --> USS3
    EULB --> EUApp
    EUApp --> EURedis
    EUApp --> EUPostgres
    EUApp --> EUS3
    APLB --> APApp
    APApp --> APRedis
    APApp --> APPostgres
    APApp --> APS3
    USPostgres <-->|Sync Replication| EUPostgres
    EUPostgres <-->|Sync Replication| APPostgres
    USRedis -.->|Async Replication| EURedis
    EURedis -.->|Async Replication| APRedis
    USS3 -.->|Cross - Region Copy| EUS3
    EUS3 -.->|Cross - Region Copy| APS3
    style Route53 fill: #ff9800
```

---

## 11. Database Sharding Strategy

**Flow Explanation:**

This diagram shows the database sharding strategy using consistent hashing to distribute data across multiple PostgreSQL
shards.

**Sharding Logic:**

1. **Shard Key**: user_id (ensures user data co-located)
2. **Shard Function**: `shard_id = hash(user_id) % 16`
3. **16 Shards**: Each shard handles 1/16th of users
4. **Read Replicas**: Each shard has 2 read replicas

**Data Distribution:**

- **Users Table**: Sharded by user_id
- **Posts Table**: Sharded by user_id (co-located with user)
- **Follows Table**: Sharded by follower_id
- **Global Tables**: Replicated to all shards (e.g., hashtags)

**Query Routing:**

1. Client sends query with user_id
2. API Gateway computes shard_id
3. Routes query to appropriate shard
4. Shard processes query and returns result

**Scalability:**

- **Horizontal Scaling**: Add more shards (16 → 32 → 64)
- **Resharding**: Use consistent hashing for minimal data movement
- **Capacity**: Each shard handles 125M users (2B total / 16)

**Benefits:**

- **Horizontal Scaling**: Add shards to scale
- **Isolation**: Failure of one shard doesn't affect others
- **Performance**: Parallel query execution

**Trade-offs:**

- **Cross-Shard Queries**: Complex and slow
- **Rebalancing**: Resharding is expensive
- **Transactions**: No cross-shard ACID

```mermaid
graph TB
    subgraph "Application Layer"
        AppServer1[App Server 1]
        AppServer2[App Server 2]
        AppServerN[App Server N]
        ShardRouter[Shard Router<br/>hash user_id mod 16]
    end

    subgraph "Shard 0 - user_id % 16 = 0"
        Shard0Primary[(PostgreSQL Primary<br/>Users: 0, 16, 32...)]
        Shard0Replica1[(Read Replica 1)]
        Shard0Replica2[(Read Replica 2)]
    end

    subgraph "Shard 1 - user_id % 16 = 1"
        Shard1Primary[(PostgreSQL Primary<br/>Users: 1, 17, 33...)]
        Shard1Replica1[(Read Replica 1)]
        Shard1Replica2[(Read Replica 2)]
    end

    subgraph "Shard 15 - user_id % 16 = 15"
        Shard15Primary[(PostgreSQL Primary<br/>Users: 15, 31, 47...)]
        Shard15Replica1[(Read Replica 1)]
        Shard15Replica2[(Read Replica 2)]
    end

    subgraph "Query Examples"
        Query1[Query: user_id=34<br/>shard = 34 mod 16 = 2]
        Query2[Query: user_id=129<br/>shard = 129 mod 16 = 1]
    end

    AppServer1 --> ShardRouter
    AppServer2 --> ShardRouter
    AppServerN --> ShardRouter
    ShardRouter -->|Route| Shard0Primary
    ShardRouter -->|Route| Shard1Primary
    ShardRouter -->|Route| Shard15Primary
    Shard0Primary --> Shard0Replica1
    Shard0Primary --> Shard0Replica2
    Shard1Primary --> Shard1Replica1
    Shard1Primary --> Shard1Replica2
    Shard15Primary --> Shard15Replica1
    Shard15Primary --> Shard15Replica2
    Query1 -.->|Routes to Shard 2| ShardRouter
    Query2 -.->|Routes to Shard 1| Shard1Primary
    style ShardRouter fill: #ff9800
```

---

## 12. CDN Distribution Architecture

**Flow Explanation:**

This diagram shows the three-tier CDN architecture that serves media files globally with high cache hit rates and low
latency.

**Architecture:**

1. **Tier 1 - Origin**: S3 buckets in 3 regions (US, EU, Asia)
2. **Tier 2 - Shield**: Regional PoPs reduce origin load
3. **Tier 3 - Edge**: 200+ edge locations worldwide

**Request Flow:**

1. User requests image: `https://cdn.instagram.com/image/12345.jpg`
2. DNS routes to nearest edge location
3. Edge checks cache:
    - **Hit**: Return immediately (90% of requests)
    - **Miss**: Request from shield
4. Shield checks cache:
    - **Hit**: Return to edge
    - **Miss**: Request from origin (S3)
5. Origin returns image
6. Shield caches (7 days TTL)
7. Edge caches (7 days TTL)

**Performance:**

- **Edge Latency**: <20ms (served from nearest location)
- **Cache Hit Rate**: 90% at edge, 95% at shield
- **Origin Load**: Only 5% of requests reach S3
- **Bandwidth Savings**: $75M/month vs direct serving

**Benefits:**

- **Low Latency**: Edge locations near all users
- **Reduced Origin Load**: 95% of traffic cached
- **Cost Savings**: CDN bandwidth cheaper than origin egress

**Trade-offs:**

- **Cache Consistency**: Invalidation takes minutes
- **Cost**: $150M/month CDN costs
- **Complexity**: Managing multi-tier cache

```mermaid
graph TB
    subgraph "Users Worldwide"
        USUser[US User]
        EUUser[EU User]
        AsiaUser[Asia User]
    end

    subgraph "Tier 3: Edge Locations - 200+ Cities"
        EdgeUS1[Edge US-NY<br/>Cache: 90% Hit]
        EdgeUS2[Edge US-LA<br/>Cache: 90% Hit]
        EdgeEU1[Edge EU-London<br/>Cache: 90% Hit]
        EdgeEU2[Edge EU-Paris<br/>Cache: 90% Hit]
        EdgeAsia1[Edge Asia-Tokyo<br/>Cache: 90% Hit]
        EdgeAsia2[Edge Asia-Singapore<br/>Cache: 90% Hit]
    end

    subgraph "Tier 2: Shield PoPs - Regional Hubs"
        ShieldUS[Shield US-East<br/>Cache: 95% Hit]
        ShieldEU[Shield EU-West<br/>Cache: 95% Hit]
        ShieldAsia[Shield Asia-Pacific<br/>Cache: 95% Hit]
    end

    subgraph "Tier 1: Origin - S3 Buckets"
        S3US[S3 US-East<br/>Origin Storage]
        S3EU[S3 EU-West<br/>Origin Storage]
        S3Asia[S3 Asia-Pacific<br/>Origin Storage]
    end

    subgraph "CDN Configuration"
        CloudflareDNS[Cloudflare DNS<br/>Geo-Routing]
        CacheControl[Cache-Control<br/>max-age=604800]
        Purge[Purge API<br/>Invalidation]
    end

    USUser --> CloudflareDNS
    EUUser --> CloudflareDNS
    AsiaUser --> CloudflareDNS
    CloudflareDNS -->|Route| EdgeUS1
    CloudflareDNS -->|Route| EdgeUS2
    CloudflareDNS -->|Route| EdgeEU1
    CloudflareDNS -->|Route| EdgeEU2
    CloudflareDNS -->|Route| EdgeAsia1
    CloudflareDNS -->|Route| EdgeAsia2
    EdgeUS1 -->|Miss| ShieldUS
    EdgeUS2 -->|Miss| ShieldUS
    EdgeEU1 -->|Miss| ShieldEU
    EdgeEU2 -->|Miss| ShieldEU
    EdgeAsia1 -->|Miss| ShieldAsia
    EdgeAsia2 -->|Miss| ShieldAsia
    ShieldUS -->|Miss 5%| S3US
    ShieldEU -->|Miss 5%| S3EU
    ShieldAsia -->|Miss 5%| S3Asia
    S3US --> CacheControl
    Purge --> EdgeUS1
    Purge --> EdgeEU1
    Purge --> EdgeAsia1
    style CloudflareDNS fill: #ff9800
    style ShieldUS fill: #4caf50
    style S3US fill: #2196f3
```

---

## 13. Feed Stitcher Component

**Flow Explanation:**

This diagram zooms into the Feed Stitcher component, which is the critical piece that merges follower posts, celebrity
posts, recommended posts, and ads into a unified, ranked feed.

**Steps:**

1. **Parallel Fetch**: Simultaneously fetch from 4 sources
    - Follower posts from Redis (pre-computed timeline)
    - Celebrity posts from PostgreSQL (fanout-on-read)
    - Recommended posts from ML engine
    - Ads from ad service
2. **Merge Strategy**: Mix with ratio 60% follower : 30% reco : 10% ads
3. **Deduplication**: Remove posts user has already seen (Bloom filter)
4. **Final Ranking**: Apply ML model for personalized ordering
5. **Return**: Top 50 posts

**Algorithm:**

```
feed = []
for i in 0..49:
  if i % 10 < 6:
    feed.add(follower_posts.pop())
  elif i % 10 < 9:
    feed.add(recommended_posts.pop())
  else:
    feed.add(ads.pop())

feed = ranking_model.score(feed, user_context)
return feed.sort_by_score()
```

**Performance:**

- **Latency**: 30-50ms total
- **Throughput**: 347k requests/sec
- **Accuracy**: 85% user engagement rate

**Benefits:**

- **Balanced Feed**: Social + discovery + monetization
- **Personalized**: ML ranking adapts to user preferences
- **Scalable**: Parallel fetching reduces latency

**Trade-offs:**

- **Complexity**: Many moving parts
- **Tuning**: Mix ratios need A/B testing
- **Staleness**: Pre-computed timelines may be minutes old

```mermaid
graph TB
    subgraph "Input Sources - Parallel Fetch"
        FollowerPosts[Follower Posts<br/>Redis Timeline<br/>Pre-computed]
        CelebrityPosts[Celebrity Posts<br/>PostgreSQL<br/>Fanout-on-Read]
        RecoPosts[Recommended Posts<br/>ML Engine<br/>Personalized]
        AdsPosts[Ads<br/>Ad Service<br/>Targeted]
    end

    subgraph "Feed Stitcher Logic"
        ParallelFetch[Parallel Fetch<br/>4 Async Tasks]
        MergeStrategy[Merge Strategy<br/>60% Follower<br/>30% Reco<br/>10% Ads]
        Dedup[Deduplication<br/>Bloom Filter<br/>Remove Seen Posts]
        FinalRanking[Final Ranking<br/>ML Model<br/>LightGBM]
        TopK[Select Top 50]
    end

    subgraph "Context & Features"
        UserContext[User Context<br/>Past Likes/Saves<br/>Time of Day]
        PostFeatures[Post Features<br/>Engagement Rate<br/>Freshness]
    end

    subgraph "Output"
        RankedFeed[Ranked Feed<br/>50 Posts<br/>Personalized Order]
    end

    FollowerPosts --> ParallelFetch
    CelebrityPosts --> ParallelFetch
    RecoPosts --> ParallelFetch
    AdsPosts --> ParallelFetch
    ParallelFetch --> MergeStrategy
    MergeStrategy --> Dedup
    Dedup --> FinalRanking
    UserContext --> FinalRanking
    PostFeatures --> FinalRanking
    FinalRanking --> TopK
    TopK --> RankedFeed
    style ParallelFetch fill: #ff9800
    style FinalRanking fill: #9c27b0
```

---

## 14. Monitoring and Observability

**Flow Explanation:**

This diagram shows the comprehensive monitoring and observability infrastructure that tracks system health, performance,
and user behavior.

**Monitoring Layers:**

1. **Application Metrics**: Prometheus collects metrics from all services
2. **Distributed Tracing**: Jaeger tracks request flow across microservices
3. **Logging**: ELK stack aggregates logs from all components
4. **Alerting**: PagerDuty notifies on-call engineers
5. **Dashboards**: Grafana visualizes metrics in real-time

**Key Metrics Tracked:**

- **Latency**: Feed load p50, p95, p99
- **Throughput**: QPS per service
- **Error Rate**: 4xx, 5xx responses
- **Cache Hit Rate**: Redis, CDN
- **Database Performance**: Query latency, connection pool
- **ML Model**: Inference latency, accuracy
- **Infrastructure**: CPU, memory, network

**Alerting Rules:**

- Feed latency p95 > 300ms → Page on-call
- Error rate > 1% → Page on-call
- CDN hit rate < 85% → Investigate
- Database replica lag > 10 seconds → Alert

**Benefits:**

- **Proactive Detection**: Catch issues before users notice
- **Root Cause Analysis**: Quickly identify bottlenecks
- **Performance Optimization**: Data-driven decisions

**Trade-offs:**

- **Cost**: Monitoring infrastructure overhead
- **Alert Fatigue**: Too many alerts = ignored alerts
- **Complexity**: Many tools to learn and maintain

```mermaid
graph TB
    subgraph "Application Services"
        FeedService[Feed Service]
        UploadService[Upload Service]
        EngageService[Engagement Service]
        RecoService[Recommendation Service]
    end

    subgraph "Metrics Collection"
        Prometheus[Prometheus<br/>Metrics Scraping]
        PrometheusDB[(Prometheus<br/>Time-Series DB)]
    end

    subgraph "Distributed Tracing"
        Jaeger[Jaeger<br/>Trace Collection]
        JaegerDB[(Jaeger Storage<br/>Cassandra)]
    end

    subgraph "Log Aggregation"
        Logstash[Logstash<br/>Log Parsing]
        Elasticsearch[(Elasticsearch<br/>Log Storage)]
        Kibana[Kibana<br/>Log Visualization]
    end

    subgraph "Alerting"
        Alertmanager[Alertmanager<br/>Alert Routing]
        PagerDuty[PagerDuty<br/>On-Call Notification]
        Slack[Slack<br/>Team Channel]
    end

    subgraph "Visualization"
        Grafana[Grafana<br/>Dashboards]
        Dashboard1[Feed Performance<br/>Dashboard]
        Dashboard2[Infrastructure<br/>Dashboard]
        Dashboard3[Business Metrics<br/>Dashboard]
    end

    subgraph "Key Metrics"
        FeedLatency[Feed Latency p95]
        CDNHitRate[CDN Hit Rate]
        ErrorRate[Error Rate 4xx/5xx]
        DBQueryTime[DB Query Latency]
    end

    FeedService --> Prometheus
    UploadService --> Prometheus
    EngageService --> Prometheus
    RecoService --> Prometheus
    FeedService --> Jaeger
    UploadService --> Jaeger
    FeedService --> Logstash
    UploadService --> Logstash
    Prometheus --> PrometheusDB
    PrometheusDB --> Grafana
    Jaeger --> JaegerDB
    Logstash --> Elasticsearch
    Elasticsearch --> Kibana
    Grafana --> Dashboard1
    Grafana --> Dashboard2
    Grafana --> Dashboard3
    PrometheusDB --> FeedLatency
    PrometheusDB --> CDNHitRate
    PrometheusDB --> ErrorRate
    PrometheusDB --> DBQueryTime
    FeedLatency --> Alertmanager
    ErrorRate --> Alertmanager
    Alertmanager --> PagerDuty
    Alertmanager --> Slack
    style Alertmanager fill: #f44336
    style Grafana fill: #ff9800
```