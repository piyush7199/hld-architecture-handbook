# Video Streaming System - Sequence Diagrams

This document contains detailed sequence diagrams showing interaction flows for the Video Streaming System.

---

## Table of Contents

1. [Video Upload Flow (Chunked Upload)](#1-video-upload-flow-chunked-upload)
2. [Transcoding Pipeline Execution](#2-transcoding-pipeline-execution)
3. [Video Streaming Flow (HLS Playback)](#3-video-streaming-flow-hls-playback)
4. [Adaptive Bitrate Switching](#4-adaptive-bitrate-switching)
5. [CDN Cache Hit and Miss](#5-cdn-cache-hit-and-miss)
6. [Search Query Execution](#6-search-query-execution)
7. [Recommendation Generation](#7-recommendation-generation)
8. [Analytics Event Processing](#8-analytics-event-processing)
9. [Multi-Region Failover](#9-multi-region-failover)
10. [Live Streaming Flow](#10-live-streaming-flow)

---

## 1. Video Upload Flow (Chunked Upload)

**Flow:**

Complete flow of a creator uploading a 5 GB video using resumable chunked upload.

**Steps:**

1. **Init Upload** (0ms): Creator requests upload, receives upload_id
2. **Chunk Loop** (4000s): Upload 500 chunks of 10 MB each
3. **S3 Multipart** (per chunk): Store each chunk in S3
4. **Redis State** (per chunk): Mark chunk complete for resumability
5. **Complete Upload** (100ms): Assemble all chunks into final video
6. **Trigger Transcode** (50ms): Publish job to Kafka queue
7. **Return Response** (50ms): Creator receives video_id

**Performance:**

- Upload speed: 10 Mbps = ~67 minutes for 5 GB
- Resumable: If interrupted, resume from last chunk
- Failure handling: Retry individual chunks (idempotent)

```mermaid
sequenceDiagram
    participant Creator
    participant WebApp
    participant UploadService
    participant Redis
    participant S3
    participant Kafka
    Creator ->> WebApp: 1. Click Upload<br/>Select video file 5 GB
    WebApp ->> WebApp: 2. Split file<br/>500 chunks × 10 MB
    WebApp ->> UploadService: 3. POST /upload/init<br/>{filename, size, type}
    UploadService ->> Redis: 4. Create state<br/>SET upload:uuid {chunks: []}
    Redis -->> UploadService: ✓ Created
    UploadService -->> WebApp: 5. {upload_id, chunk_size: 10MB}

    loop For each chunk 1 to 500
        WebApp ->> UploadService: 6. POST /upload/chunk<br/>{upload_id, chunk_num, data}
        UploadService ->> S3: 7. Multipart upload<br/>Part number chunk_num
        S3 -->> UploadService: ✓ ETag
        UploadService ->> Redis: 8. SADD upload:id:chunks chunk_num
        Redis -->> UploadService: ✓ Added
        UploadService -->> WebApp: 9. {chunk_num completed}
        WebApp ->> Creator: Progress 60% (300/500)
    end

    WebApp ->> UploadService: 10. POST /upload/complete<br/>{upload_id}
    UploadService ->> Redis: 11. Check SCARD<br/>Verify all 500 chunks
    Redis -->> UploadService: Count: 500 ✓
    UploadService ->> S3: 12. Complete multipart<br/>Combine parts
    S3 -->> UploadService: ✓ s3://raw/video_123.mp4
    UploadService ->> Kafka: 13. Publish job<br/>{video_id, s3_key}
    Kafka -->> UploadService: ✓ Queued
    UploadService ->> Redis: 14. DEL upload:id<br/>Cleanup state
    UploadService -->> WebApp: 15. {video_id, status: processing}
    WebApp -->> Creator: ✅ Upload complete!<br/>Processing video...
    Note over Creator, Kafka: Total: ~67 min upload + instant processing trigger
```

---

## 2. Transcoding Pipeline Execution

**Flow:**

Worker picks up transcode job and processes video into multiple formats.

**Steps:**

1. **Consume Job** (0ms): Worker polls Kafka, gets transcode job
2. **Download Raw** (120s): Fetch 5 GB from S3 (42 MB/sec)
3. **Decode** (30s): Extract audio/video tracks with FFmpeg
4. **Parallel Transcode** (300s): 6 qualities processed simultaneously
5. **Segment** (60s): Split each quality into 10-second chunks
6. **Upload Segments** (180s): Store HLS segments in S3
7. **Generate Manifest** (5s): Create master.m3u8 file
8. **Update Metadata** (10s): Mark video as "Ready" in Cassandra
9. **CDN Invalidate** (5s): Clear CDN cache for new video
10. **Acknowledge** (1s): Commit Kafka offset

**Total Time**: ~710 seconds (11.8 minutes) for 5-minute video

**Performance**: 2.4x real-time (5-min video takes 11.8 min)

```mermaid
sequenceDiagram
    participant Kafka
    participant Worker
    participant S3Raw
    participant FFmpeg
    participant S3Processed
    participant Cassandra
    participant CDN
    Kafka ->> Worker: 1. Poll queue<br/>Message: transcode job
    Worker ->> Worker: 2. Parse job<br/>{video_id, s3_key}
    Worker ->> S3Raw: 3. Download raw<br/>GET s3://raw/video_123.mp4
    S3Raw -->> Worker: 4. 5 GB video (120s)
    Worker ->> FFmpeg: 5. Decode<br/>Extract tracks (30s)
    FFmpeg -->> Worker: ✓ Audio + Video tracks

    par Parallel Transcode (300s)
        Worker ->> FFmpeg: 6a. Encode 144p<br/>200 Kbps
        Worker ->> FFmpeg: 6b. Encode 360p<br/>800 Kbps
        Worker ->> FFmpeg: 6c. Encode 720p<br/>5 Mbps
        Worker ->> FFmpeg: 6d. Encode 1080p<br/>8 Mbps
        Worker ->> FFmpeg: 6e. Encode 4K<br/>25 Mbps
    end

    FFmpeg -->> Worker: ✓ All qualities encoded
    Worker ->> Worker: 7. Segment videos<br/>10-second chunks (60s)

    loop For each quality
        Worker ->> S3Processed: 8. Upload segments<br/>video_123/720p/seg_*.ts
    end

    S3Processed -->> Worker: ✓ All uploaded (180s)
    Worker ->> Worker: 9. Generate manifest<br/>master.m3u8 (5s)
    Worker ->> S3Processed: 10. Upload manifest
    S3Processed -->> Worker: ✓ Manifest uploaded
    Worker ->> Cassandra: 11. UPDATE videos<br/>SET status='Ready'<br/>formats={720p: 'cdn...'}
    Cassandra -->> Worker: ✓ Updated (10s)
    Worker ->> CDN: 12. POST /invalidate<br/>video_123/*
    CDN -->> Worker: ✓ Cache cleared (5s)
    Worker ->> Kafka: 13. Commit offset<br/>ACK message
    Kafka -->> Worker: ✓ Acknowledged
    Note over Kafka, CDN: Total: ~710s (11.8 min) for 5-min video
```

---

## 3. Video Streaming Flow (HLS Playback)

**Flow:**

User watches a video from start to finish using HLS adaptive streaming.

**Steps:**

1. **Request Metadata** (50ms): Fetch video info from API
2. **Check Cache** (1ms): Redis cache hit for metadata
3. **Fetch Manifest** (20ms): Download master.m3u8 from CDN
4. **Select Quality** (10ms): Client estimates bandwidth, chooses 720p
5. **Download Segments** (loop): Fetch segments sequentially
6. **Playback** (continuous): Decode and display video
7. **Buffer Management** (ongoing): Maintain 30-second buffer
8. **Send Analytics** (every 10s): Track watch progress

**Total Startup**: < 2 seconds (metadata + manifest + first segment)

```mermaid
sequenceDiagram
    participant User
    participant VideoPlayer
    participant APIGateway
    participant Redis
    participant Cassandra
    participant CDN
    participant S3
    participant Analytics
    User ->> VideoPlayer: 1. Click play<br/>video_123
    VideoPlayer ->> APIGateway: 2. GET /video/123
    APIGateway ->> Redis: 3. Check cache<br/>GET video:123
    Redis -->> APIGateway: 4. Cache HIT (1ms)<br/>{title, cdn_url, formats}
    APIGateway -->> VideoPlayer: 5. Metadata (50ms total)
    VideoPlayer ->> CDN: 6. GET /video_123/master.m3u8
    CDN -->> VideoPlayer: 7. Manifest (20ms)<br/>Lists: 144p, 360p, 720p, 1080p
    VideoPlayer ->> VideoPlayer: 8. Estimate bandwidth<br/>Initial: 6 Mbps → Choose 720p
    VideoPlayer ->> CDN: 9. GET /video_123/720p/segment_001.ts
    CDN -->> VideoPlayer: 10. Segment 1 (50ms)
    VideoPlayer ->> VideoPlayer: 11. Decode & display<br/>Start playback
    User ->> User: ▶️ Video starts (< 2s startup)

    loop Continuous playback
        VideoPlayer ->> CDN: 12. GET segment_N.ts
        CDN -->> VideoPlayer: Segment N (50ms)
        VideoPlayer ->> VideoPlayer: Buffer segment<br/>Maintain 30s buffer
        VideoPlayer ->> Analytics: 13. Send event<br/>{position, quality, buffering}
    end

    Analytics ->> Analytics: Track watch time<br/>Update view count
    Note over User, Analytics: Seamless playback with < 2s startup
```

---

## 4. Adaptive Bitrate Switching

**Flow:**

Client dynamically switches video quality based on network conditions.

**Steps:**

1. **Initial Playback** (0s): Start at 720p based on initial bandwidth estimate
2. **Monitor Bandwidth** (ongoing): Measure download time per segment
3. **Detect Degradation** (30s): Bandwidth drops from 6 Mbps to 2 Mbps
4. **Switch Quality** (31s): Download next segment at 480p instead
5. **Seamless Transition** (31s): No interruption to playback
6. **Continue Monitoring** (ongoing): Ready to switch back to 720p if bandwidth improves

**Performance:**

- Quality switch latency: < 1 second (at segment boundary)
- Buffer maintained: 30 seconds (prevents buffering during switch)
- User experience: Smooth, no interruption

```mermaid
sequenceDiagram
    participant VideoPlayer
    participant BandwidthMonitor
    participant ABRAlgorithm
    participant CDN
    VideoPlayer ->> CDN: 1. GET 720p/segment_001.ts
    CDN -->> VideoPlayer: Segment 1 (40ms)<br/>5 MB in 40ms = 1 Gbps
    VideoPlayer ->> BandwidthMonitor: 2. Measure<br/>Download: 1 Gbps
    BandwidthMonitor ->> ABRAlgorithm: 3. Bandwidth: 1 Gbps<br/>Current: 720p (5 Mbps)<br/>Decision: Stay 720p
    VideoPlayer ->> CDN: 4. GET 720p/segment_002.ts
    CDN -->> VideoPlayer: Segment 2 (50ms)
    VideoPlayer ->> BandwidthMonitor: Measure: 800 Mbps
    BandwidthMonitor ->> ABRAlgorithm: BW: 800 Mbps OK
    VideoPlayer ->> CDN: 5. GET 720p/segment_003.ts
    CDN -->> VideoPlayer: Segment 3 (50ms)
    Note over VideoPlayer: Network congestion starts
    VideoPlayer ->> CDN: 6. GET 720p/segment_004.ts
    CDN -->> VideoPlayer: Segment 4 (2000ms SLOW)
    VideoPlayer ->> BandwidthMonitor: 7. Measure<br/>Download: 20 Mbps (SLOW)
    BandwidthMonitor ->> ABRAlgorithm: 8. BW: 20 Mbps<br/>Current: 720p (5 Mbps)<br/>Status: Still OK but declining
    VideoPlayer ->> CDN: 9. GET 720p/segment_005.ts
    CDN -->> VideoPlayer: Segment 5 (5000ms VERY SLOW)
    VideoPlayer ->> BandwidthMonitor: 10. Measure<br/>Download: 8 Mbps (CRITICAL)
    BandwidthMonitor ->> ABRAlgorithm: 11. BW: 8 Mbps<br/>Current: 720p (5 Mbps)<br/>Decision: SWITCH to 480p
    ABRAlgorithm ->> VideoPlayer: 12. Switch quality<br/>Next segment: 480p
    VideoPlayer ->> CDN: 13. GET 480p/segment_006.ts<br/>(1.5 Mbps bitrate)
    CDN -->> VideoPlayer: Segment 6 (1500ms)<br/>Smooth download
    VideoPlayer ->> VideoPlayer: 14. Seamless playback<br/>No buffering, quality change imperceptible
    BandwidthMonitor ->> ABRAlgorithm: BW: Stable at 8 Mbps<br/>480p appropriate
    Note over VideoPlayer, CDN: Continues at 480p until bandwidth improves
```

---

## 5. CDN Cache Hit and Miss

**Flow:**

Shows the difference between CDN cache hit (fast) and cache miss (slow).

**Steps:**

**Cache Hit** (99% of requests):

1. User requests segment → Edge location
2. Edge has segment cached → Return immediately (20ms)

**Cache Miss** (1% of requests):

1. User requests segment → Edge location
2. Edge cache miss → Query regional cache
3. Regional cache miss → Fetch from S3 origin
4. S3 returns segment → Cache at regional and edge
5. Edge returns to user (200ms total)

**Performance:**

- Cache hit: 20ms latency
- Cache miss: 200ms latency (10x slower)
- Cost: Hit $0.01/GB, Miss $0.09/GB (9x more expensive)

```mermaid
sequenceDiagram
    participant User1 as User 1<br/>London
    participant EdgeLondon as CDN Edge London
    participant RegionalEU as CDN Regional EU
    participant S3Origin as S3 Origin US-East
    participant User2 as User 2<br/>London
    Note over User1, User2: Scenario A: Cache MISS (first request)
    User1 ->> EdgeLondon: 1. GET video_123/720p/seg_042.ts
    EdgeLondon ->> EdgeLondon: 2. Check cache<br/>MISS (not cached)
    EdgeLondon ->> RegionalEU: 3. Forward request
    RegionalEU ->> RegionalEU: 4. Check cache<br/>MISS (not cached)
    RegionalEU ->> S3Origin: 5. Fetch from origin
    S3Origin -->> RegionalEU: 6. Segment (150ms)<br/>5 MB data
    RegionalEU ->> RegionalEU: 7. Cache segment<br/>TTL: 24 hours
    RegionalEU -->> EdgeLondon: 8. Return segment (150ms)
    EdgeLondon ->> EdgeLondon: 9. Cache segment<br/>TTL: 7 days
    EdgeLondon -->> User1: 10. Segment (200ms total)
    Note over User1, S3Origin: Cost: $0.09/GB (S3 transfer)
    Note over User1, User2: Scenario B: Cache HIT (subsequent requests)
    User2 ->> EdgeLondon: 11. GET video_123/720p/seg_042.ts<br/>(same segment)
    EdgeLondon ->> EdgeLondon: 12. Check cache<br/>HIT ✓
    EdgeLondon -->> User2: 13. Segment (20ms)<br/>Served from edge
    Note over User2, EdgeLondon: Cost: $0.01/GB (CDN transfer)
    Note over User1, User2: Result: 10x faster, 9x cheaper
```

---

## 6. Search Query Execution

**Flow:**

User searches for videos using full-text search powered by Elasticsearch.

**Steps:**

1. **Search Query** (0ms): User types "funny cat videos"
2. **API Gateway** (10ms): Parse query, validate
3. **Elasticsearch** (50ms): Full-text search across 1B videos
4. **Ranking** (20ms): Score by relevance + popularity
5. **Fetch Thumbnails** (30ms): Get thumbnail URLs from metadata
6. **Return Results** (10ms): Top 20 videos
7. **Display** (50ms): Render search results page

**Total Latency**: ~170ms (fast enough for real-time search)

**Ranking Factors:**

- Text relevance (BM25 score)
- View count (popularity boost)
- Upload date (recency boost)
- User preferences (personalization)

```mermaid
sequenceDiagram
    participant User
    participant SearchUI
    participant APIGateway
    participant Elasticsearch
    participant Cassandra
    participant CDN
    User ->> SearchUI: 1. Type "funny cat videos"<br/>Press Enter
    SearchUI ->> APIGateway: 2. GET /search?q=funny+cat+videos<br/>&limit=20
    APIGateway ->> APIGateway: 3. Parse query<br/>Extract keywords
    APIGateway ->> Elasticsearch: 4. POST /videos/_search<br/>{<br/> query: multi_match<br/> fields: [title, description, tags]<br/> sort: [views DESC, _score DESC]<br/>}
    Elasticsearch ->> Elasticsearch: 5. Search index<br/>1 billion documents<br/>Match: 1.2M results
    Elasticsearch ->> Elasticsearch: 6. Rank results<br/>- Text relevance BM25<br/>- Boost by views<br/>- Filter by date
    Elasticsearch -->> APIGateway: 7. Top 20 results (50ms)<br/>[video_123, video_456, ...]

    par Fetch metadata for top 20
        APIGateway ->> Cassandra: 8. Batch GET<br/>video_123, video_456, ...
        Cassandra -->> APIGateway: Metadata (30ms)
    end

    APIGateway ->> APIGateway: 9. Enrich results<br/>Add thumbnail URLs
    APIGateway -->> SearchUI: 10. Response (170ms total)<br/>{<br/> results: [<br/> {title, thumbnail, views, duration}<br/> ]<br/>}
    SearchUI ->> CDN: 11. Prefetch thumbnails<br/>20 thumbnail images
    CDN -->> SearchUI: Thumbnails (50ms)
    SearchUI ->> User: 12. Display results<br/>20 videos with thumbnails
    Note over User, CDN: Total: 220ms (search + thumbnails)
```

---

## 7. Recommendation Generation

**Flow:**

System generates personalized recommendations for a user.

**Steps:**

1. **User Request** (0ms): User loads homepage
2. **Check Cache** (1ms): Redis has pre-computed recommendations
3. **Return Recommendations** (10ms): Top 20 videos
4. **Background Update** (async): Track view, update model
5. **Batch Training** (daily): Train ML model on all user data
6. **Deploy Model** (daily): Update recommendations in Redis

**Performance:**

- Serving latency: < 10ms (cached)
- Model training: 24 hours (Spark cluster)
- Accuracy: 60% click-through rate

```mermaid
sequenceDiagram
    participant User
    participant WebApp
    participant RecAPI as Recommendation API
    participant RedisCache
    participant Cassandra
    participant KafkaEvents
    participant SparkBatch
    participant MLModel
    User ->> WebApp: 1. Load homepage
    WebApp ->> RecAPI: 2. GET /recommendations?user_id=456
    RecAPI ->> RedisCache: 3. GET user:456:recs
    RedisCache -->> RecAPI: 4. Cache HIT (1ms)<br/>[<br/> {video_789, score: 0.95},<br/> {video_123, score: 0.87},<br/> ...<br/>]
    RecAPI -->> WebApp: 5. Top 20 videos (10ms)
    WebApp -->> User: 6. Display recommendations
    User ->> WebApp: 7. Click video_789
    WebApp ->> KafkaEvents: 8. Publish event<br/>{user_id, video_id, action: click}
    Note over KafkaEvents, MLModel: Offline Training Pipeline (Daily)
    SparkBatch ->> Cassandra: 9. Load watch history<br/>100M users × 100 videos
    Cassandra -->> SparkBatch: User-video matrix
    SparkBatch ->> SparkBatch: 10. Feature engineering<br/>- User preferences<br/>- Video features<br/>- Collaborative filtering
    SparkBatch ->> MLModel: 11. Train model<br/>ALS Matrix Factorization<br/>24 hours
    MLModel ->> MLModel: 12. Evaluate<br/>Validation set accuracy
    MLModel ->> RedisCache: 13. Deploy model<br/>Update recommendations<br/>for all users
    RedisCache ->> RedisCache: 14. Refresh cache<br/>New recs for user:456
    Note over User, MLModel: Next homepage load will have updated recs
```

---

## 8. Analytics Event Processing

**Flow:**

Real-time processing of video watch events for analytics.

**Steps:**

1. **Client Events** (every 10s): Player sends watch progress
2. **Kafka Ingestion** (10ms): Buffer events in Kafka topic
3. **Spark Streaming** (micro-batch 10s): Aggregate events
4. **Update Metrics** (50ms): Increment view count, watch time
5. **Sink to Warehouse** (async): Store for historical analysis
6. **Creator Dashboard** (real-time): Show live metrics

**Metrics Tracked:**

- View count (unique views per video)
- Watch time (total minutes watched)
- Audience retention (% who watch X% of video)
- Engagement rate (likes + comments + shares / views)

```mermaid
sequenceDiagram
    participant VideoPlayer
    participant KafkaEvents
    participant SparkStreaming
    participant RedisMetrics
    participant Cassandra
    participant BigQuery
    participant Dashboard

    loop Every 10 seconds
        VideoPlayer ->> KafkaEvents: 1. Send event<br/>{<br/> video_id: 123,<br/> user_id: 456,<br/> position: 30s,<br/> quality: 720p,<br/> buffering: false<br/>}
    end

    KafkaEvents ->> KafkaEvents: 2. Buffer events<br/>Partition by video_id
    KafkaEvents ->> SparkStreaming: 3. Stream events<br/>Micro-batch every 10s
    SparkStreaming ->> SparkStreaming: 4. Aggregate<br/>- Group by video_id<br/>- Count unique users<br/>- Sum watch time

    par Update metrics
        SparkStreaming ->> RedisMetrics: 5. INCR video:123:views
        SparkStreaming ->> RedisMetrics: 6. INCRBY video:123:watch_time 600
        SparkStreaming ->> Cassandra: 7. UPDATE videos<br/>SET views = views + 1
    end

    SparkStreaming ->> BigQuery: 8. Sink events<br/>Raw events for analytics
    Dashboard ->> RedisMetrics: 9. GET video:123:views
    RedisMetrics -->> Dashboard: 10. Current: 1,523,847 views
    Dashboard ->> BigQuery: 11. Query analytics<br/>SELECT retention by minute
    BigQuery -->> Dashboard: 12. Retention curve
    Dashboard ->> Dashboard: 13. Render chart<br/>Real-time metrics
    Note over VideoPlayer, Dashboard: Latency: <30s from event to dashboard
```

---

## 9. Multi-Region Failover

**Flow:**

Automatic failover when a region becomes unavailable.

**Steps:**

1. **Normal Operation** (T0): User in London routed to EU region
2. **Region Failure** (T1): EU region health check fails
3. **Detection** (T2): Route53 detects unhealthy region (30s)
4. **DNS Update** (T3): Remove EU from DNS records
5. **Reroute** (T4): New requests routed to US region
6. **Graceful Degradation** (T4): Slightly higher latency, but service available
7. **Recovery** (T10): EU region recovers, added back to DNS

**RTO** (Recovery Time Objective): < 1 minute

**RPO** (Recovery Point Objective): 0 (no data loss, multi-master Cassandra)

```mermaid
sequenceDiagram
    participant User as User London
    participant Route53
    participant EURegion as EU Region
    participant USRegion as US Region
    participant HealthCheck
    Note over User, HealthCheck: T0: Normal operation
    User ->> Route53: 1. DNS lookup<br/>video.example.com
    Route53 -->> User: 2. EU region IP<br/>(lowest latency)
    User ->> EURegion: 3. GET /video/123
    EURegion -->> User: 4. Video stream (50ms)
    Note over EURegion: T1: Region failure (network outage)
    HealthCheck ->> EURegion: 5. Health check (every 10s)
    EURegion --x HealthCheck: 6. TIMEOUT (no response)
    HealthCheck ->> HealthCheck: 7. Retry (3 attempts)<br/>All fail
    Note over HealthCheck: T2: After 30s, mark unhealthy
    HealthCheck ->> Route53: 8. Report EU unhealthy
    Route53 ->> Route53: 9. Update DNS<br/>Remove EU from rotation
    Note over User, USRegion: T4: New requests rerouted
    User ->> Route53: 10. DNS lookup<br/>(TTL expired)
    Route53 -->> User: 11. US region IP<br/>(fallback)
    User ->> USRegion: 12. GET /video/123
    USRegion -->> User: 13. Video stream (150ms)<br/>Higher latency but works
    User ->> User: ⚠️ Slightly slower<br/>but service available
    Note over EURegion: T10: Region recovers
    HealthCheck ->> EURegion: 14. Health check
    EURegion -->> HealthCheck: 15. ✓ Healthy
    HealthCheck ->> Route53: 16. Report EU healthy
    Route53 ->> Route53: 17. Add EU back<br/>to DNS rotation
    Note over User, EURegion: New requests use EU again (low latency)
```

---

## 10. Live Streaming Flow

**Flow:**

Streamer broadcasts live video to millions of viewers.

**Steps:**

1. **RTMP Ingest** (0ms): Streamer pushes via OBS
2. **Live Transcoder** (real-time): Convert to multiple qualities
3. **HLS/WebRTC Output** (real-time): Generate segments on-the-fly
4. **CDN Distribution** (100ms): Push to edge locations
5. **Viewer Playback** (3-5s latency): Watch with minimal delay
6. **Chat Integration** (real-time): WebSocket for live chat

**Latency:**

- WebRTC: 3 seconds (ultra-low latency)
- LL-HLS: 10 seconds (low latency)
- Standard HLS: 30 seconds (higher latency but scalable)

```mermaid
sequenceDiagram
    participant Streamer
    participant OBS
    participant IngestServer
    participant LiveTranscoder
    participant CDN
    participant Viewer1 as Viewer 1<br/>WebRTC
    participant Viewer2 as Viewer 2<br/>HLS
    participant ChatServer
    Streamer ->> OBS: 1. Start stream<br/>Webcam + mic
    OBS ->> IngestServer: 2. RTMP push<br/>rtmp://ingest/live/stream_key<br/>1080p 8 Mbps
    IngestServer ->> IngestServer: 3. Validate key<br/>Authenticate streamer
    IngestServer ->> LiveTranscoder: 4. Forward stream<br/>Raw RTMP

    par Real-time transcoding
        LiveTranscoder ->> LiveTranscoder: 5a. Encode 360p
        LiveTranscoder ->> LiveTranscoder: 5b. Encode 720p
        LiveTranscoder ->> LiveTranscoder: 5c. Encode 1080p
    end

    par Multi-format output
        LiveTranscoder ->> CDN: 6a. WebRTC output<br/>Ultra-low latency
        LiveTranscoder ->> CDN: 6b. HLS output<br/>2-second segments
    end

    CDN ->> CDN: 7. Distribute to edges<br/>1000+ locations
    Viewer1 ->> CDN: 8. Connect WebRTC<br/>Request stream
    CDN -->> Viewer1: 9. Stream (3s latency)<br/>Real-time playback
    Viewer2 ->> CDN: 10. GET master.m3u8
    CDN -->> Viewer2: 11. Manifest
    Viewer2 ->> CDN: 12. GET 720p/live_seg.ts
    CDN -->> Viewer2: 13. Segment (10s latency)

    par Live chat
        Viewer1 ->> ChatServer: 14. WebSocket connect
        Viewer1 ->> ChatServer: 15. Send message<br/>"Great stream!"
        ChatServer ->> Viewer1: Broadcast to all
        ChatServer ->> Viewer2: Message received
    end

    Streamer ->> OBS: 16. End stream
    OBS ->> IngestServer: 17. Disconnect RTMP
    IngestServer ->> CDN: 18. Stop broadcast
    CDN ->> Viewer1: Stream ended
    CDN ->> Viewer2: Stream ended
    Note over Streamer, Viewer2: Total: 3-10s latency (acceptable for live)
```
