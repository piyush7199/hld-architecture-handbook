# Video Streaming System - High-Level Design

This document contains Mermaid diagrams showing the architecture and data flow for the Video Streaming System.

---

## Table of Contents

1. [Overall System Architecture](#1-overall-system-architecture)
2. [Upload Service Architecture](#2-upload-service-architecture)
3. [Resumable Upload Flow](#3-resumable-upload-flow)
4. [Transcoding Pipeline Architecture](#4-transcoding-pipeline-architecture)
5. [HLS Adaptive Bitrate Streaming](#5-hls-adaptive-bitrate-streaming)
6. [CDN Multi-Tier Architecture](#6-cdn-multi-tier-architecture)
7. [Metadata Database Design](#7-metadata-database-design)
8. [Search and Discovery Architecture](#8-search-and-discovery-architecture)
9. [Recommendation Engine Architecture](#9-recommendation-engine-architecture)
10. [Multi-Region Deployment](#10-multi-region-deployment)
11. [Analytics Pipeline](#11-analytics-pipeline)
12. [Live Streaming Architecture](#12-live-streaming-architecture)

---

## 1. Overall System Architecture

**Flow Explanation:**

This diagram shows the complete end-to-end architecture of the video streaming system.

**Components:**

1. **Upload Path**: Creator uploads video → Upload Service → S3 → Kafka → Transcode Workers
2. **Streaming Path**: Viewer requests video → API Gateway → Metadata DB → CDN → Client
3. **Search Path**: User searches → Elasticsearch → Results
4. **Analytics Path**: Watch events → Kafka → Stream Processing → Data Warehouse

**Benefits:**

- **Separation of Concerns**: Upload, streaming, search, and analytics are decoupled
- **Scalability**: Each component can scale independently
- **High Availability**: Multi-region deployment with CDN

**Trade-offs:**

- **Complexity**: Many components to manage
- **Consistency**: Eventual consistency between components

```mermaid
graph TB
    subgraph Creators
        Creator[Content Creator<br/>Upload Video]
    end

    subgraph Upload_Pipeline
        UploadSvc[Upload Service<br/>Chunked Upload]
        RawS3[Raw Storage<br/>S3 Bucket]
        Kafka[Kafka Queue<br/>Transcoding Jobs]
        TranscodeWorkers[Transcoding Workers<br/>Auto-Scaling Fleet]
        ProcessedS3[Processed Storage<br/>S3 HLS Segments]
    end

    subgraph Streaming_Pipeline
        Viewers[Viewers<br/>Watch Video]
        APIGateway[API Gateway<br/>REST API]
        MetadataDB[Metadata DB<br/>Cassandra]
        RedisCache[Redis Cache<br/>Hot Metadata]
        CDN[CDN<br/>CloudFront]
    end

    subgraph Search_Pipeline
        SearchUI[Search UI]
        Elasticsearch[Elasticsearch<br/>Full-Text Search]
        CDC[Change Data Capture<br/>Debezium]
    end

    subgraph Analytics_Pipeline
        EventStream[Event Stream<br/>Kafka]
        SparkStreaming[Spark Streaming<br/>Real-Time Processing]
        DataWarehouse[Data Warehouse<br/>BigQuery/Redshift]
        Dashboard[Analytics Dashboard<br/>Creator Metrics]
    end

    subgraph ML_Pipeline
        RecommendationEngine[Recommendation Engine<br/>ML Model]
        ModelTraining[Model Training<br/>Spark MLlib]
    end

    Creator -->|1 . Upload| UploadSvc
    UploadSvc -->|2 . Store Raw| RawS3
    UploadSvc -->|3 . Publish Job| Kafka
    Kafka -->|4 . Consume| TranscodeWorkers
    TranscodeWorkers -->|5 . Transcode| ProcessedS3
    TranscodeWorkers -->|6 . Update Metadata| MetadataDB
    Viewers -->|1 . Request Video| APIGateway
    APIGateway -->|2 . Check Cache| RedisCache
    APIGateway -->|3 . Query| MetadataDB
    APIGateway -->|4 . Return CDN URL| Viewers
    Viewers -->|5 . Stream| CDN
    CDN -->|6 . Fetch Origin| ProcessedS3
    SearchUI -->|1 . Search Query| Elasticsearch
    MetadataDB -->|2 . CDC| CDC
    CDC -->|3 . Index| Elasticsearch
    Viewers -->|Watch Events| EventStream
    EventStream -->|Process| SparkStreaming
    SparkStreaming -->|Store| DataWarehouse
    DataWarehouse -->|Visualize| Dashboard
    DataWarehouse -->|Train| ModelTraining
    ModelTraining -->|Deploy| RecommendationEngine
    RecommendationEngine -->|Suggestions| Viewers
    style UploadSvc fill: #ff9999
    style TranscodeWorkers fill: #ffcc99
    style CDN fill: #99ccff
    style MetadataDB fill: #ccff99
    style Elasticsearch fill: #ffccff
    style RecommendationEngine fill: #ffff99
```

---

## 2. Upload Service Architecture

**Flow Explanation:**

Shows the architecture of the Upload Service handling resumable uploads.

**Steps:**

1. **Load Balancer**: Distributes upload requests across multiple instances
2. **Upload Instances**: Stateless servers handling chunk uploads
3. **Redis**: Tracks upload state (completed chunks per upload_id)
4. **S3**: Stores raw video chunks using Multipart Upload API

**Benefits:**

- **Resumability**: Client can resume from last successful chunk
- **Horizontal Scaling**: Add more upload instances to handle load
- **Fault Tolerance**: Redis persists upload state

**Performance:**

- **Throughput**: 10k concurrent uploads
- **Latency**: < 100ms per chunk acknowledgment
- **Storage**: Temporary state in Redis (1 hour TTL)

```mermaid
graph TB
    Users[Multiple Uploaders<br/>5 GB videos]
    LB[Load Balancer<br/>Route by upload_id]

    subgraph Upload_Instances
        U1[Upload Instance 1]
        U2[Upload Instance 2]
        U3[Upload Instance 3]
        UN[Upload Instance N]
    end

    Redis[Redis<br/>Upload State<br/>TTL: 1 hour]
    S3Multi[S3 Multipart Upload<br/>Raw Video Chunks]
    KafkaQueue[Kafka<br/>Transcoding Queue]
    Users -->|POST /upload/chunk| LB
    LB -->|Sticky by upload_id| U1
    LB --> U2
    LB --> U3
    LB --> UN
    U1 -->|Set chunk_N complete| Redis
    U1 -->|Upload chunk| S3Multi
    U1 -.->|Check state| Redis
    U1 -->|All chunks done| KafkaQueue
    Redis -.->|State: upload_123<br/>chunks: 1,2,3 . . . 499,500| Redis
    style LB fill: #ffcc99
    style Redis fill: #ff9999
    style S3Multi fill: #99ccff
    style KafkaQueue fill: #ccff99
```

---

## 3. Resumable Upload Flow

**Flow Explanation:**

Details the step-by-step process of resumable chunk upload.

**Steps:**

1. **Init Upload**: Client POST /upload/init → Server returns upload_id
2. **Chunk Loop**: For each 10 MB chunk:
    - Client uploads chunk with (upload_id, chunk_number)
    - Server validates, stores in S3, marks chunk complete in Redis
3. **Complete Upload**: Client POST /upload/complete → Server assembles file
4. **Trigger Transcode**: Server publishes job to Kafka

**Failure Handling:**

- **Chunk Upload Fails**: Client retries specific chunk (idempotent)
- **Server Crashes**: New instance reads state from Redis
- **Network Interruption**: Client resumes from last acknowledged chunk

**Performance:**

- **Upload Speed**: 10 Mbps = 1.25 MB/sec = 8 seconds per 10 MB chunk
- **5 GB Video**: 500 chunks × 8 sec = 4000 sec = ~67 minutes

```mermaid
sequenceDiagram
    participant Client
    participant UploadService
    participant Redis
    participant S3
    participant Kafka
    Client ->> UploadService: 1. POST /upload/init<br/>{filename, size, content_type}
    UploadService ->> Redis: 2. Create upload state<br/>upload_id: uuid()
    Redis -->> UploadService: ✓ Created
    UploadService -->> Client: 3. {upload_id, chunk_size: 10MB}

    loop For each chunk (1 to 500)
        Client ->> UploadService: 4. POST /upload/chunk<br/>{upload_id, chunk_num, data}
        UploadService ->> S3: 5. Multipart upload chunk<br/>Part number: chunk_num
        S3 -->> UploadService: ✓ ETag
        UploadService ->> Redis: 6. Mark chunk complete<br/>SADD upload:123:chunks chunk_num
        Redis -->> UploadService: ✓ Added
        UploadService -->> Client: 7. {chunk_num, status: completed}
    end

    Client ->> UploadService: 8. POST /upload/complete<br/>{upload_id}
    UploadService ->> Redis: 9. Check all chunks<br/>SCARD upload:123:chunks
    Redis -->> UploadService: Count: 500 (all present)
    UploadService ->> S3: 10. Complete multipart upload<br/>Combine all parts
    S3 -->> UploadService: ✓ video_id
    UploadService ->> Kafka: 11. Publish transcode job<br/>{video_id, s3_key}
    Kafka -->> UploadService: ✓ Queued
    UploadService -->> Client: 12. {video_id, status: processing}
    Note over Client, Kafka: Total time: ~67 minutes for 5 GB
```

---

## 4. Transcoding Pipeline Architecture

**Flow Explanation:**

Shows the complete transcoding pipeline from raw video to HLS segments.

**Steps:**

1. **Kafka Consumer**: Worker picks up transcode job from queue
2. **Download**: Fetch raw video from S3 (5 GB)
3. **Decode**: Extract audio and video tracks
4. **Parallel Encode**: Transcode to 6 qualities simultaneously
5. **Segment**: Split into 10-second chunks
6. **Upload**: Store segments in S3
7. **Manifest**: Generate HLS manifest file
8. **Notify**: Update metadata DB with status "Ready"

**Benefits:**

- **Parallel Processing**: All qualities processed simultaneously (5x faster)
- **Auto-Scaling**: Workers scale based on queue depth
- **Fault Tolerance**: Failed jobs redelivered by Kafka

**Performance:**

- **Sequential**: 6 qualities × 5 min = 30 min
- **Parallel**: 6 qualities in parallel = 5 min (with 6 workers)
- **Real-Time Ratio**: 1x (5-min video takes 5 min to transcode)

```mermaid
graph LR
    Kafka[Kafka Queue<br/>Pending Jobs]

    subgraph Worker_Pool
        W1[Worker 1<br/>c5.9xlarge]
        W2[Worker 2]
        W3[Worker 3]
        WN[Worker N]
    end

    S3Raw[S3 Raw Storage<br/>Original Video]

    subgraph Transcoding_Process
        Download[Download Raw<br/>5 GB]
        FFmpeg[FFmpeg Decode<br/>Extract Tracks]

        subgraph Parallel_Encode
            E144[Encode 144p<br/>200 Kbps]
            E360[Encode 360p<br/>800 Kbps]
            E720[Encode 720p<br/>5 Mbps]
            E1080[Encode 1080p<br/>8 Mbps]
            E4K[Encode 4K<br/>25 Mbps]
        end

        Segment[Segment<br/>10-sec chunks]
        Manifest[Generate Manifest<br/>HLS m3u8]
    end

    S3Processed[S3 Processed Storage<br/>HLS Segments]
    MetadataDB[Metadata DB<br/>Update Status]
    CDN[CDN<br/>Invalidate Cache]
    Kafka -->|1 . Consume Job| W1
    W1 -->|2 . Download| S3Raw
    S3Raw -->|3 . 5 GB| Download
    Download --> FFmpeg
    FFmpeg -->|4 . Parallel| E144
    FFmpeg --> E360
    FFmpeg --> E720
    FFmpeg --> E1080
    FFmpeg --> E4K
    E144 --> Segment
    E360 --> Segment
    E720 --> Segment
    E1080 --> Segment
    E4K --> Segment
    Segment -->|5 . Upload| S3Processed
    Segment --> Manifest
    Manifest -->|6 . Upload| S3Processed
    S3Processed -->|7 . Notify| MetadataDB
    S3Processed -->|8 . Invalidate| CDN
    style FFmpeg fill: #ff9999
    style Parallel_Encode fill: #ffcc99
    style S3Processed fill: #99ccff
```

---

## 5. HLS Adaptive Bitrate Streaming

**Flow Explanation:**

Shows how clients adapt video quality based on bandwidth.

**Steps:**

1. **Initial Request**: Client requests master manifest (m3u8)
2. **Quality Selection**: Client estimates bandwidth, selects initial quality (720p)
3. **Segment Download**: Client downloads segments sequentially
4. **Bandwidth Monitoring**: Client measures download time per segment
5. **Quality Switch**: If bandwidth decreases, switch to lower quality (480p)
6. **Seamless Playback**: No interruption during quality change

**Benefits:**

- **No Buffering**: Always downloads quality that fits bandwidth
- **Optimal Experience**: Best quality for available bandwidth
- **Automatic**: No user intervention required

**Performance:**

- **Initial Startup**: < 2 seconds (manifest + first segment)
- **Quality Switch**: < 1 second (during segment boundary)
- **Buffer Size**: 30 seconds (3 segments × 10 sec)

```mermaid
graph TB
    Client[Video Player<br/>Client]
    MasterManifest[Master Manifest<br/>master.m3u8]

    subgraph Available_Qualities
        Q144[144p Playlist<br/>200 Kbps]
        Q360[360p Playlist<br/>800 Kbps]
        Q720[720p Playlist<br/>5 Mbps]
        Q1080[1080p Playlist<br/>8 Mbps]
    end

    subgraph Segment_Storage
        S720_1[720p/seg001.ts]
        S720_2[720p/seg002.ts]
        S720_3[720p/seg003.ts]
        S480_4[480p/seg004.ts]
    end

    BandwidthMonitor[Bandwidth Monitor<br/>Measure Download Speed]
    ABRLogic[ABR Algorithm<br/>Select Quality]
    Client -->|1 . GET| MasterManifest
    MasterManifest -->|2 . List| Q144
    MasterManifest --> Q360
    MasterManifest --> Q720
    MasterManifest --> Q1080
    Client -->|3 . Estimate BW<br/>Choose 720p| ABRLogic
    ABRLogic -->|4 . Select| Q720
    Q720 -->|5 . Download| S720_1
    S720_1 -->|6 . Measure| BandwidthMonitor
    BandwidthMonitor -->|7 . BW: 6 Mbps OK| ABRLogic
    ABRLogic --> S720_2
    S720_2 --> BandwidthMonitor
    BandwidthMonitor -->|BW: 5 . 5 Mbps OK| ABRLogic
    ABRLogic --> S720_3
    S720_3 --> BandwidthMonitor
    BandwidthMonitor -->|BW: 2 Mbps LOW| ABRLogic
    ABRLogic -->|8 . Switch to 480p| S480_4
    style BandwidthMonitor fill: #ff9999
    style ABRLogic fill: #ffcc99
    style Q720 fill: #99ccff
```

---

## 6. CDN Multi-Tier Architecture

**Flow Explanation:**

Shows the three-tier CDN architecture with edge, regional, and origin caches.

**Layers:**

1. **Edge Locations** (1000+): Closest to users, serve 99% of requests
2. **Regional Caches** (50+): Aggregate requests from multiple edges
3. **Origin (S3)**: Source of truth, serves cache misses

**Cache Strategy:**

- **Hot Videos** (1M+ views): Cached at edge for 7 days
- **Warm Videos** (10k-1M views): Cached at regional for 24 hours
- **Cold Videos** (<10k views): Fetched from origin on demand

**Performance:**

- **Edge Hit** (99%): 20ms latency
- **Regional Hit** (0.9%): 50ms latency
- **Origin Hit** (0.1%): 200ms latency

**Cost Savings**:

- **Without CDN**: 300 PB/month × $0.09/GB = $27M
- **With CDN (99% hit)**: 3 PB origin × $0.09/GB + 297 PB edge × $0.01/GB = $3.24M
- **Savings**: $23.76M/month (88% reduction)

```mermaid
graph TB
    subgraph Users_Worldwide
        US_User[US User]
        EU_User[EU User]
        Asia_User[Asia User]
    end

    subgraph Edge_Locations_1000+
        US_Edge[US Edge<br/>New York<br/>Cache: 99% hit<br/>TTL: 7 days]
        EU_Edge[EU Edge<br/>London<br/>Cache: 99% hit]
        Asia_Edge[Asia Edge<br/>Tokyo<br/>Cache: 99% hit]
    end

    subgraph Regional_Caches_50+
        US_Regional[US Regional<br/>Virginia<br/>Cache: 95% hit<br/>TTL: 24h]
        EU_Regional[EU Regional<br/>Frankfurt<br/>Cache: 95% hit]
        Asia_Regional[Asia Regional<br/>Singapore<br/>Cache: 95% hit]
    end

    subgraph Origin_S3
        S3_US_East[S3 US-East<br/>Origin Bucket<br/>All Content]
        S3_US_West[S3 US-West<br/>Replica]
        S3_EU[S3 EU-Central<br/>Replica]
    end

    US_User -->|1 . Request| US_Edge
    EU_User -->|1 . Request| EU_Edge
    Asia_User -->|1 . Request| Asia_Edge
    US_Edge -->|2 . Cache Miss<br/>1% of requests| US_Regional
    EU_Edge -->|2 . Cache Miss| EU_Regional
    Asia_Edge -->|2 . Cache Miss| Asia_Regional
    US_Regional -->|3 . Cache Miss<br/>0 . 05% of requests| S3_US_East
    EU_Regional -->|3 . Cache Miss| S3_EU
    Asia_Regional -->|3 . Cache Miss| S3_US_East
    S3_US_East -.->|Replication| S3_US_West
    S3_US_East -.->|Replication| S3_EU
    US_Edge -.->|Cache: video_123/720p/<br/>segment_042 . ts<br/>TTL: 7 days| US_Edge
    style US_Edge fill: #99ccff
    style US_Regional fill: #ffcc99
    style S3_US_East fill: #ff9999
```

---

## 7. Metadata Database Design

**Flow Explanation:**

Shows the metadata storage architecture with Cassandra and Redis caching.

**Schema Design:**

- **Partition Key**: video_id (ensures single-partition queries)
- **Columns**: title, description, duration, views (counter), cdn_urls (map)

**Caching Strategy:**

- **Redis Cache-Aside**: Cache popular video metadata
- **TTL**: 1 hour for metadata, 5 minutes for view counts
- **Cache Hit Rate**: 80% for popular videos

**Performance:**

- **Cache Hit**: 1ms (Redis)
- **Cache Miss**: 10ms (Cassandra)
- **Write**: 5ms (async, not critical path)

**Scalability**:

- **Cassandra**: 1 billion videos partitioned across 100 nodes
- **Redis**: Hot metadata for 10 million videos (10 GB)

```mermaid
graph TB
    Client[Client Request<br/>GET /video/123]
    APIGateway[API Gateway]

    subgraph Caching_Layer
        Redis[Redis Cache<br/>Hot Metadata<br/>TTL: 1 hour]
    end

    subgraph Database_Layer
        Cassandra[Cassandra Cluster<br/>100 Nodes<br/>1 Billion Videos]

        subgraph Cassandra_Schema
            VideoTable[videos Table<br/>Partition Key: video_id<br/>Columns:<br/>- title<br/>- description<br/>- upload_date<br/>- duration<br/>- views counter<br/>- likes counter<br/>- cdn_urls map<br/>- thumbnail_url]
        end
    end

    subgraph CDC_Pipeline
        Debezium[Debezium CDC<br/>Change Data Capture]
        KafkaConnect[Kafka Connect<br/>Stream Changes]
        Elasticsearch[Elasticsearch<br/>Search Index]
    end

    Client --> APIGateway
    APIGateway -->|1 . Check Cache| Redis
    Redis -->|2a . Cache HIT<br/>1ms| APIGateway
    Redis -->|2b . Cache MISS| Cassandra
    Cassandra -->|3 . Query<br/>10ms| Redis
    Cassandra --> VideoTable
    Redis -->|4 . Cache Result| APIGateway
    Cassandra -.->|5 . CDC Stream| Debezium
    Debezium -.-> KafkaConnect
    KafkaConnect -.-> Elasticsearch
    APIGateway -->|6 . Response| Client
    Redis -.->|Cache Entry:<br/>video:123<br/>title, cdn_urls<br/>720p, 1080p<br/>TTL: 3600s| Redis
    style Redis fill: #ff9999
    style Cassandra fill: #99ccff
    style VideoTable fill: #ffcc99
```

---

## 8. Search and Discovery Architecture

**Flow Explanation:**

Shows the search architecture with Elasticsearch powered by CDC from Cassandra.

**Components:**

1. **Cassandra**: Source of truth for video metadata
2. **Debezium**: Captures changes from Cassandra
3. **Kafka**: Streams changes to Elasticsearch
4. **Elasticsearch**: Full-text search index

**Search Features:**

- **Full-Text Search**: Title, description, transcript
- **Filters**: Duration, upload date, views, category
- **Ranking**: Relevance score + popularity + personalization

**Performance:**

- **Search Latency**: < 100ms (p95)
- **Index Size**: 1 billion documents, 500 GB
- **Refresh Rate**: Near real-time (< 1 second delay)

```mermaid
graph TB
    User[User Search<br/>funny cat videos]
    SearchUI[Search UI<br/>Web/Mobile]

    subgraph Search_Service
        SearchAPI[Search API<br/>REST Endpoint]
        Elasticsearch[Elasticsearch Cluster<br/>10 Nodes, 1B Documents]

        subgraph Index_Structure
            VideoIndex[videos Index<br/>Sharded by hash video_id<br/>Fields:<br/>- title text<br/>- description text<br/>- tags keyword<br/>- transcript text<br/>- views long<br/>- upload_date date<br/>- duration integer]
        end
    end

    subgraph Data_Source
        Cassandra[Cassandra<br/>Video Metadata]
        CDC[Debezium CDC<br/>Change Capture]
        KafkaStream[Kafka Stream<br/>video-changes]
        ElasticsearchConnector[Elasticsearch Connector<br/>Index Updates]
    end

    User -->|1 . Search Query| SearchUI
    SearchUI -->|2 . POST /search<br/>query: funny cat| SearchAPI
    SearchAPI -->|3 . Query ES| Elasticsearch
    Elasticsearch --> VideoIndex
    VideoIndex -->|4 . Match & Rank<br/> - Text Relevance BM25<br/> - Boost by views<br/> - Filter by date| Elasticsearch
    Elasticsearch -->|5 . Top 20 Results| SearchAPI
    SearchAPI -->|6 . Response| SearchUI
    SearchUI -->|7 . Display| User
    Cassandra -.->|Write video| CDC
    CDC -.->|Capture change| KafkaStream
    KafkaStream -.->|Stream event| ElasticsearchConnector
    ElasticsearchConnector -.->|Index/Update| Elasticsearch
    style Elasticsearch fill: #ffccff
    style VideoIndex fill: #ff9999
    style Cassandra fill: #99ccff
```

---

## 9. Recommendation Engine Architecture

**Flow Explanation:**

Shows the ML-based recommendation system architecture.

**Components:**

1. **Watch History**: User viewing patterns stored in Cassandra
2. **Feature Engineering**: Extract user and video features
3. **ML Model**: Collaborative filtering + content-based filtering
4. **Batch Processing**: Offline training daily with Spark
5. **Online Serving**: Real-time recommendations from Redis

**Models:**

- **Collaborative Filtering**: Matrix factorization (users × videos)
- **Content-Based**: Similar videos based on tags, category
- **Hybrid**: Combine both approaches

**Performance:**

- **Training**: 24 hours daily (Spark cluster)
- **Serving**: < 10ms (Redis cache)
- **Accuracy**: 60% click-through rate on recommendations

```mermaid
graph TB
    User[User<br/>Watch History]

    subgraph Online_Path_Real_Time
        WebApp[Web App<br/>Homepage]
        RecommendationAPI[Recommendation API<br/>REST Endpoint]
        RedisCache[Redis Cache<br/>Pre-computed Recs<br/>TTL: 6 hours]
    end

    subgraph Offline_Training_Pipeline
        DataWarehouse[Data Warehouse<br/>BigQuery<br/>Watch History<br/>100M users]
        SparkCluster[Spark Cluster<br/>Feature Engineering<br/>+ Model Training]

        subgraph ML_Models
            CollaborativeFiltering[Collaborative Filtering<br/>ALS Matrix Factorization<br/>Users x Videos]
            ContentBased[Content-Based<br/>Similar Videos<br/>by Tags/Category]
            Hybrid[Hybrid Model<br/>Ensemble]
        end

        ModelStore[Model Store<br/>S3 Versioned Models]
    end

    subgraph Real_Time_Update
        KafkaEvents[Kafka<br/>Watch Events Stream]
        SparkStreaming[Spark Streaming<br/>Update User Preferences]
    end

    User -->|1 . Request Homepage| WebApp
    WebApp -->|2 . GET /recommendations| RecommendationAPI
    RecommendationAPI -->|3 . Check Cache| RedisCache
    RedisCache -->|4 . Return Top 20| RecommendationAPI
    RecommendationAPI -->|5 . Response| WebApp
    WebApp -->|6 . Display| User
    DataWarehouse -.->|Daily ETL<br/>Batch Process| SparkCluster
    SparkCluster -.-> CollaborativeFiltering
    SparkCluster -.-> ContentBased
    CollaborativeFiltering -.-> Hybrid
    ContentBased -.-> Hybrid
    Hybrid -.->|Train & Evaluate| ModelStore
    ModelStore -.->|Deploy New Model| RedisCache
    User -.->|Watch Event| KafkaEvents
    KafkaEvents -.-> SparkStreaming
    SparkStreaming -.->|Update Preferences| RedisCache
    RedisCache -.->|Cache Entry:<br/>user:456:recs<br/>video_789 score:0 . 95<br/>video_123 score:0 . 87| RedisCache
    style RedisCache fill: #ff9999
    style Hybrid fill: #ffcc99
    style DataWarehouse fill: #99ccff
```

---

## 10. Multi-Region Deployment

**Flow Explanation:**

Shows the global multi-region deployment for low latency worldwide.

**Regions:**

1. **US-East**: Primary region, full stack
2. **EU-West**: Full stack, serves European users
3. **Asia-Pacific**: Full stack, serves Asian users

**Routing Strategy:**

- **GeoDNS**: Route users to nearest region based on IP
- **Active-Active**: All regions serve traffic simultaneously
- **Replication**: Cassandra multi-master, S3 cross-region replication

**Failover:**

- **Health Checks**: Monitor each region every 10 seconds
- **Automatic Failover**: If region unhealthy, route to next nearest
- **Recovery Time Objective (RTO)**: < 1 minute

```mermaid
graph TB
    subgraph Global_Users
        US_Users[US Users<br/>100M]
        EU_Users[EU Users<br/>80M]
        Asia_Users[Asia Users<br/>120M]
    end

    GeoDNS[GeoDNS<br/>Route53<br/>Latency-Based Routing]

    subgraph US_East_Region
        US_LB[US Load Balancer]
        US_API[US API Gateway]
        US_Cassandra[US Cassandra<br/>3 Nodes]
        US_Redis[US Redis]
        US_S3[US S3 Origin]
        US_CDN[US CDN Edges<br/>300 Locations]
    end

    subgraph EU_West_Region
        EU_LB[EU Load Balancer]
        EU_API[EU API Gateway]
        EU_Cassandra[EU Cassandra<br/>3 Nodes]
        EU_Redis[EU Redis]
        EU_S3[EU S3 Origin]
        EU_CDN[EU CDN Edges<br/>250 Locations]
    end

    subgraph Asia_Pacific_Region
        Asia_LB[Asia Load Balancer]
        Asia_API[Asia API Gateway]
        Asia_Cassandra[Asia Cassandra<br/>3 Nodes]
        Asia_Redis[Asia Redis]
        Asia_S3[Asia S3 Origin]
        Asia_CDN[Asia CDN Edges<br/>400 Locations]
    end

    US_Users --> GeoDNS
    EU_Users --> GeoDNS
    Asia_Users --> GeoDNS
    GeoDNS -->|Route by latency| US_LB
    GeoDNS --> EU_LB
    GeoDNS --> Asia_LB
    US_LB --> US_API
    US_API --> US_Cassandra
    US_API --> US_Redis
    US_Users -.->|Stream| US_CDN
    US_CDN -.-> US_S3
    EU_LB --> EU_API
    EU_API --> EU_Cassandra
    EU_API --> EU_Redis
    EU_Users -.->|Stream| EU_CDN
    EU_CDN -.-> EU_S3
    Asia_LB --> Asia_API
    Asia_API --> Asia_Cassandra
    Asia_API --> Asia_Redis
    Asia_Users -.->|Stream| Asia_CDN
    Asia_CDN -.-> Asia_S3
    US_Cassandra -.->|Multi - Master<br/>Replication| EU_Cassandra
    US_Cassandra -.->|Multi - Master| Asia_Cassandra
    EU_Cassandra -.->|Multi - Master| Asia_Cassandra
    US_S3 -.->|Cross - Region<br/>Replication| EU_S3
    US_S3 -.->|Cross - Region| Asia_S3
    style US_East_Region fill: #ff9999
    style EU_West_Region fill: #99ccff
    style Asia_Pacific_Region fill: #ffcc99
```

---

## 11. Analytics Pipeline

**Flow Explanation:**

Shows the analytics pipeline for tracking views, watch time, and engagement.

**Events Tracked:**

- Video started, paused, resumed, completed
- Quality switched, buffering events
- Likes, comments, shares

**Processing:**

1. **Client**: Sends events to Kafka every 10 seconds
2. **Spark Streaming**: Real-time aggregation (views, watch time)
3. **Data Warehouse**: Store for historical analysis
4. **Dashboard**: Creator analytics, platform metrics

**Metrics Calculated:**

- **View Count**: Unique views per video
- **Watch Time**: Total minutes watched
- **Audience Retention**: % who watch X% of video
- **Engagement Rate**: (Likes + Comments + Shares) / Views

```mermaid
graph TB
    subgraph Video_Players
        WebPlayer[Web Player]
        MobilePlayer[Mobile Player]
    end

    subgraph Event_Ingestion
        KafkaEvents[Kafka<br/>video-events Topic<br/>Partitioned by video_id]

        subgraph Event_Types
            ViewStart[Event: view_start<br/>video_id, user_id, timestamp]
            ViewProgress[Event: view_progress<br/>position, quality, buffering]
            ViewComplete[Event: view_complete<br/>watch_time, dropped_at]
            Engagement[Event: engagement<br/>like, comment, share]
        end
    end

    subgraph Real_Time_Processing
        SparkStreaming[Spark Streaming<br/>Micro-Batch 10s]

        subgraph Aggregations
            ViewCount[View Counter<br/>Increment views per video]
            WatchTime[Watch Time Aggregator<br/>Sum minutes watched]
            Retention[Retention Calculator<br/>% complete per segment]
        end

        RedisLive[Redis<br/>Real-Time Metrics<br/>TTL: 1 hour]
    end

    subgraph Batch_Processing
        DataWarehouse[Data Warehouse<br/>BigQuery/Redshift<br/>Historical Data]
        BatchJobs[Daily Batch Jobs<br/>- Top videos<br/>- Trending<br/>- User cohorts]
    end

    subgraph Dashboards
        CreatorDashboard[Creator Dashboard<br/>Analytics UI]
        PlatformMetrics[Platform Metrics<br/>Admin Dashboard]
    end

    WebPlayer -->|1 . Send Events<br/>Every 10s| KafkaEvents
    MobilePlayer -->|1 . Send Events| KafkaEvents
    KafkaEvents --> ViewStart
    KafkaEvents --> ViewProgress
    KafkaEvents --> ViewComplete
    KafkaEvents --> Engagement
    KafkaEvents -->|2 . Stream| SparkStreaming
    SparkStreaming --> ViewCount
    SparkStreaming --> WatchTime
    SparkStreaming --> Retention
    ViewCount -->|3 . Update| RedisLive
    WatchTime --> RedisLive
    Retention --> RedisLive
    SparkStreaming -.->|4 . Sink| DataWarehouse
    DataWarehouse -.->|5 . Run| BatchJobs
    RedisLive -->|6 . Query| CreatorDashboard
    BatchJobs --> CreatorDashboard
    BatchJobs --> PlatformMetrics
    style KafkaEvents fill: #ffcc99
    style SparkStreaming fill: #ff9999
    style DataWarehouse fill: #99ccff
```

---

## 12. Live Streaming Architecture

**Flow Explanation:**

Shows the architecture for live streaming (e.g., Twitch, YouTube Live).

**Key Differences from VOD:**

- **No Pre-Transcoding**: Must transcode in real-time
- **Ultra-Low Latency**: < 5 seconds delay (vs 30 seconds for VOD)
- **Protocols**: RTMP/WebRTC/SRT instead of HLS

**Components:**

1. **Ingest**: Streamer sends video via RTMP
2. **Live Transcoder**: Real-time transcoding to multiple qualities
3. **Origin**: WebRTC/LL-HLS for low latency
4. **CDN**: Distribute to millions of viewers
5. **Chat**: Real-time chat via WebSocket

**Performance:**

- **Latency**: 3-5 seconds (WebRTC), 10-15 seconds (LL-HLS)
- **Concurrent Viewers**: 1M+ per stream (major events)
- **Cost**: 5x higher than VOD (real-time processing)

```mermaid
graph TB
    Streamer[Streamer<br/>OBS Studio]

    subgraph Ingest_Layer
        RTMPIngest[RTMP Ingest<br/>Port 1935<br/>Nginx-RTMP]
        LoadBalancer[Ingest Load Balancer<br/>Sticky Sessions]
    end

    subgraph Live_Transcoding
        LiveTranscoder[Live Transcoder<br/>Real-Time FFmpeg<br/>c5.9xlarge GPU]

        subgraph Output_Formats
            RTMP_Out[RTMP Output<br/>Multiple Qualities]
            WebRTC_Out[WebRTC Output<br/>Ultra-Low Latency]
            HLS_Out[HLS Output<br/>Standard Latency]
        end
    end

    subgraph Distribution
        Origin[Origin Server<br/>Nginx<br/>Serve All Formats]
        CDN_Edge[CDN Edge Locations<br/>1000+ POPs]
    end

    subgraph Viewers
        WebRTC_Viewer[WebRTC Viewer<br/>3s latency]
        HLS_Viewer[HLS Viewer<br/>15s latency]
        Mobile_Viewer[Mobile Viewer<br/>Adaptive Bitrate]
    end

    subgraph Chat_System
        ChatWS[Chat WebSocket<br/>Real-Time Messages]
        ChatRedis[Chat Redis Pub/Sub<br/>Message Routing]
    end

    Streamer -->|1 . RTMP Push<br/>rtmp://ingest/live/stream_key| LoadBalancer
    LoadBalancer --> RTMPIngest
    RTMPIngest -->|2 . Raw Stream| LiveTranscoder
    LiveTranscoder -->|3 . Transcode| RTMP_Out
    LiveTranscoder --> WebRTC_Out
    LiveTranscoder --> HLS_Out
    RTMP_Out --> Origin
    WebRTC_Out --> Origin
    HLS_Out --> Origin
    Origin -->|4 . Distribute| CDN_Edge
    CDN_Edge -->|5 . Stream| WebRTC_Viewer
    CDN_Edge --> HLS_Viewer
    CDN_Edge --> Mobile_Viewer
    WebRTC_Viewer -.->|Chat Message| ChatWS
    ChatWS -.-> ChatRedis
    ChatRedis -.->|Broadcast| WebRTC_Viewer
    ChatRedis -.-> HLS_Viewer
    ChatRedis -.-> Mobile_Viewer
    LiveTranscoder -.->|Health Check<br/>Stream Status| Origin
    style LiveTranscoder fill: #ff9999
    style WebRTC_Out fill: #ffcc99
    style CDN_Edge fill: #99ccff
    style ChatRedis fill: #ccff99
```