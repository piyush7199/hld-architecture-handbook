# 3.4.8 Design a Video Streaming System (YouTube / Netflix)

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

Design a video streaming platform like YouTube or Netflix that allows users to upload, process, store, and stream video
content
globally with high availability and low latency. The system must handle massive scale (petabytes of data, 100M+
concurrent
viewers), support adaptive bitrate streaming, and minimize bandwidth costs while providing a seamless viewing
experience.

**Key Challenges:**

- **Massive Storage**: Store petabytes of video content in multiple formats
- **High Bandwidth**: Serve 90% of internet traffic during peak hours
- **Complex Processing**: Transcode videos into multiple formats (144p to 4K)
- **Global Distribution**: Deliver content with <2s startup latency worldwide
- **Cost Optimization**: Bandwidth and storage costs dominate operational expenses
- **Adaptive Streaming**: Switch quality dynamically based on network conditions

---

## 2. Requirements and Scale Estimation

### 2.1 Functional Requirements (FRs)

1. **Video Upload**: Accept large video files (up to 50 GB) and process them
2. **Video Transcoding**: Convert uploaded video into multiple formats and bitrates (144p to 4K)
3. **Streaming (VOD)**: Deliver video efficiently with minimal buffering and high availability
4. **Playback Control**: Support seeking, pausing, resuming, and speed control
5. **Search and Discovery**: Find videos by title, tags, description
6. **Recommendations**: Suggest relevant videos based on watch history
7. **User Engagement**: Likes, comments, subscriptions, playlists

### 2.2 Non-Functional Requirements (NFRs)

1. **High Throughput (Read)**: Must serve petabytes of data daily (~90% of all internet traffic for Netflix/YouTube)
2. **Low Latency (Startup)**: Video playback must start in under 2 seconds
3. **Fault Tolerance**: Transcoding must be resilient; a single server failure must not lose the entire job
4. **Cost Efficiency**: Bandwidth and storage are the dominant costs and must be aggressively minimized
5. **High Availability**: 99.99% uptime for streaming service
6. **Scalability**: Handle traffic spikes during major events

### 2.3 Scale Estimation

| Metric                    | Assumption                                  | Calculation          | Result              |
|---------------------------|---------------------------------------------|----------------------|---------------------|
| **Total Users**           | 2 Billion registered users                  | -                    | -                   |
| **Daily Active Users**    | 500 Million DAU                             | -                    | -                   |
| **Concurrent Viewers**    | 100 Million peak concurrent streams         | -                    | 100M streams        |
| **Videos Uploaded Daily** | 500,000 new videos/day                      | -                    | 5.8 videos/sec      |
| **Average Video Size**    | 100 MB raw, 500 MB transcoded (all formats) | -                    | -                   |
| **Daily Upload Storage**  | 500k × 500 MB                               | $250$ TB/day         | ~90 PB/year         |
| **Total Video Catalog**   | 1 Billion videos                            | 1B × 500 MB          | ~500 PB storage     |
| **Daily Data Transfer**   | 100M concurrent × 1 GB/hour × 3 hours avg   | $300$ PB/day         | Massive CDN needed  |
| **Bandwidth**             | 300 PB / 86400 sec                          | ~3.5 million GB/sec  | 28 Tbps aggregate   |
| **Transcoding Time**      | 5 min video → 30 min processing             | 500k videos × 30 min | 10.4M CPU-hours/day |

**Key Insights:**

- **Storage grows at 90 PB/year** → Need cost-effective cold storage
- **Bandwidth dominates costs** → CDN is mandatory, not optional
- **Transcoding is CPU-intensive** → Requires massive worker fleet (10k+ servers)
- **Read:Write ratio is 1000:1** → Optimize for reads, async writes

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

The architecture is dominated by the **CDN** (for serving read traffic) and the **Transcoding Pipeline** (for processing
write
traffic).

### 3.1 Core Components

**Write Path (Upload & Processing):**

```
User → Upload Service → Raw Storage (S3) → Transcoding Queue (Kafka) → Worker Fleet → Processed Storage (S3) → CDN
```

**Read Path (Streaming):**

```
User → API Gateway → Metadata DB (Cassandra) → CDN (CloudFront) → User
```

### 3.2 Component Overview

1. **Upload Service**: Handles resumable uploads via chunked HTTP or gRPC streaming
2. **Raw Storage (S3)**: Stores original, unprocessed video files
3. **Transcoding Queue (Kafka)**: Buffers massive list of transcoding jobs, ensures durability
4. **Transcoding Worker Fleet**: CPU-heavy, auto-scaling cluster that converts video into different formats
5. **Processed Storage (S3)**: Stores transcoded video segments (HLS/DASH)
6. **CDN (Content Delivery Network)**: Caches and serves finished video segments to viewers globally
7. **Metadata Database (Cassandra/DynamoDB)**: Stores video titles, descriptions, CDN URLs for all formats
8. **Search Service (Elasticsearch)**: Full-text search for video discovery
9. **Recommendation Engine**: ML-based system for personalized video suggestions
10. **Analytics Pipeline**: Track views, watch time, engagement metrics

### 3.3 Architecture Diagram (ASCII)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            VIDEO STREAMING SYSTEM                            │
└─────────────────────────────────────────────────────────────────────────────┘

                                 WRITE PATH (Upload & Transcode)

┌─────────┐     ┌────────────┐     ┌─────────────┐     ┌──────────────────┐
│ Creator │────→│   Upload   │────→│ Raw Storage │────→│ Kafka Queue      │
│ (Web/   │     │   Service  │     │ (S3 Bucket) │     │ (transcode jobs) │
│  App)   │     │  (gRPC)    │     │             │     │                  │
└─────────┘     └────────────┘     └─────────────┘     └──────────────────┘
                                                                  │
                                                                  ▼
                                        ┌─────────────────────────────────────┐
                                        │   Transcoding Worker Fleet          │
                                        │   ┌───────┬───────┬───────┬───────┐ │
                                        │   │Worker1│Worker2│Worker3│WorkerN│ │
                                        │   │  CPU  │  CPU  │  CPU  │  CPU  │ │
                                        │   └───────┴───────┴───────┴───────┘ │
                                        │   Auto-Scaling Group (1k-10k pods)  │
                                        └─────────────────────────────────────┘
                                                      │
                                                      ▼
                                        ┌──────────────────────────┐
                                        │ Processed Storage (S3)   │
                                        │ ┌──────────────────────┐ │
                                        │ │ video_123/           │ │
                                        │ │   144p/segment_*.ts  │ │
                                        │ │   360p/segment_*.ts  │ │
                                        │ │   720p/segment_*.ts  │ │
                                        │ │   1080p/segment_*.ts │ │
                                        │ │   4k/segment_*.ts    │ │
                                        │ │   manifest.m3u8      │ │
                                        │ └──────────────────────┘ │
                                        └──────────────────────────┘
                                                      │
                                                      ▼
                                        ┌──────────────────────────┐
                                        │  CDN Origin (S3 + CF)    │
                                        └──────────────────────────┘

                                 READ PATH (Streaming)

┌─────────┐     ┌────────────┐     ┌──────────────┐     ┌────────────────┐
│ Viewer  │────→│ API Gateway│────→│  Metadata DB │────→│ CDN Edge       │
│ (Web/   │     │  (REST)    │     │  (Cassandra) │     │ (CloudFront)   │
│  App)   │     └────────────┘     │              │     │                │
└─────────┘            │            │ video_id:    │     │ ┌────────────┐ │
      ▲                │            │   title      │     │ │ Hot Cache  │ │
      │                │            │   formats[]  │     │ │ (Popular   │ │
      │                │            │   cdn_urls[] │     │ │  Videos)   │ │
      │                │            └──────────────┘     │ └────────────┘ │
      │                │                                 │                │
      │                └─────────────────────────────────→ Fetch segment  │
      │                                                  │ from S3 origin │
      └──────────────────────────────────────────────────┘                │
                         Stream HLS/DASH segments                         │
                                                                           │
                         ┌──────────────────────────────────────────────┐
                         │ CDN Global Distribution (1000+ Edge Locations)│
                         │ ┌───────┬───────┬───────┬───────┬───────┐    │
                         │ │  US   │  EU   │ Asia  │ LATAM │Africa │    │
                         │ │ East  │ West  │Pacific│       │       │    │
                         │ └───────┴───────┴───────┴───────┴───────┘    │
                         └──────────────────────────────────────────────┘
```

---

## 4. Detailed Component Design

### 4.1 Upload Service and Resumable Uploads

**Challenge**: Videos can be 50 GB+. Network interruptions are common. Users should not have to restart uploads.

**Solution**: Resumable Uploads (Chunked HTTP or gRPC Streaming)

**How It Works:**

1. **Client splits file into chunks** (10 MB each)
2. **Upload each chunk** with metadata (chunk_id, video_id, chunk_number)
3. **Server acknowledges each chunk** and stores in S3
4. **If upload fails**, client resumes from last acknowledged chunk
5. **After all chunks uploaded**, server assembles file and triggers transcoding

**Implementation:**

- Use **TUS Protocol** (Resumable Upload Protocol) over HTTP
- Or **gRPC Bidirectional Streaming** for efficient binary transfer
- Store upload state in **Redis** (tracks completed chunks)
- Use **S3 Multipart Upload API** (supports up to 10,000 parts, 5 GB each)

*See pseudocode.md::uploadVideoChunked() for implementation*

### 4.2 Transcoding Pipeline

**Challenge**: Convert raw video into multiple formats (144p, 360p, 720p, 1080p, 4K) with different codecs (H.264,
H.265,
VP9).

**Architecture:**

```
Raw Video → Kafka Queue → Worker Fleet → Processed Segments → S3 → CDN
```

**Transcoding Process:**

1. **Extract audio and video tracks** separately
2. **Encode video** into multiple resolutions
3. **Encode audio** into multiple bitrates
4. **Segment video** into small chunks (2-10 seconds each)
5. **Generate manifest files** (HLS `.m3u8` or DASH `.mpd`)
6. **Upload to S3** and invalidate CDN cache

**Transcoding Formats:**

| Quality   | Resolution | Bitrate (Video) | Bitrate (Audio) | Use Case   |
|-----------|------------|-----------------|-----------------|------------|
| **144p**  | 256×144    | 200 Kbps        | 64 Kbps         | Mobile 2G  |
| **360p**  | 640×360    | 800 Kbps        | 96 Kbps         | Mobile 3G  |
| **480p**  | 854×480    | 1.5 Mbps        | 128 Kbps        | Mobile 4G  |
| **720p**  | 1280×720   | 5 Mbps          | 128 Kbps        | HD Desktop |
| **1080p** | 1920×1080  | 8 Mbps          | 192 Kbps        | Full HD    |
| **4K**    | 3840×2160  | 25 Mbps         | 256 Kbps        | Ultra HD   |

**Worker Fleet:**

- **Auto-Scaling**: Scale based on Kafka queue depth (similar to judge workers)
- **Spot Instances**: Use AWS Spot for 70% cost savings (non-urgent transcode)
- **Dedicated Instances**: Use for live streams or high-priority content
- **FFmpeg**: Open-source tool for video processing

*See pseudocode.md::transcodeVideo() for implementation*

### 4.3 Adaptive Bitrate Streaming (ABR)

**Challenge**: Users have varying network conditions. Must switch quality dynamically without interrupting playback.

**Solution**: HLS (HTTP Live Streaming) or DASH (Dynamic Adaptive Streaming over HTTP)

**How HLS Works:**

1. **Server provides manifest file** (`.m3u8`) listing all available qualities
2. **Client downloads manifest**, selects initial quality based on bandwidth estimate
3. **Client downloads video segments** (2-10 sec chunks) via HTTP
4. **Client measures download speed** for each segment
5. **Client switches quality** if bandwidth increases/decreases

**Example HLS Manifest:**

```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=854x480
480p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1280x720
720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=8000000,RESOLUTION=1920x1080
1080p/playlist.m3u8
```

**Benefits:**

- **No Buffering**: Client downloads next segment before current ends
- **Quality Switching**: Seamless transition between qualities
- **HTTP-Based**: Works through firewalls, uses standard CDN

*See pseudocode.md::selectBitrate() for ABR algorithm*

### 4.4 CDN Strategy and Edge Caching

**Challenge**: Bandwidth costs dominate (90% of expenses). Must minimize origin requests.

**Solution**: Multi-Tier CDN with Aggressive Caching

**CDN Tiers:**

```
User → Edge (99% hit) → Regional (95% hit) → Origin (S3)
```

1. **Edge Locations** (1000+): Close to users, cache hot content (popular videos)
2. **Regional Caches** (50+): Aggregate traffic from multiple edges
3. **Origin (S3)**: Stores all content, serves cache misses

**Caching Strategy:**

- **Hot Content** (popular videos): Cache for 7 days at edge
- **Warm Content** (moderate traffic): Cache for 24 hours at regional
- **Cold Content** (rarely accessed): Fetch from origin on demand

**Cache Key:**

```
video_id + quality + segment_number
Example: video_123/720p/segment_042.ts
```

**CDN Optimization:**

- **Prefetching**: CDN prefetches next segment before client requests
- **Range Requests**: Support HTTP Range headers for seeking
- **Compression**: Use Gzip/Brotli for manifest files
- **Geo-Routing**: Route users to nearest edge location

*See pseudocode.md::calculateCDNCache() for cache TTL logic*

### 4.5 Metadata Management

**Challenge**: Store metadata for 1 billion videos with low-latency reads.

**Database Choice**: Cassandra or DynamoDB (NoSQL)

**Why NoSQL?**

- **Horizontal Scalability**: Partition by video_id across many nodes
- **Low Latency**: Single-digit millisecond reads
- **High Availability**: Multi-region replication

**Schema:**

```
Table: videos
Partition Key: video_id
Columns:
  - title (string)
  - description (string)
  - upload_date (timestamp)
  - duration (integer, seconds)
  - views (counter)
  - likes (counter)
  - channel_id (string)
  - formats (map<quality, cdn_url>)
  - thumbnail_url (string)
  - tags (list<string>)
```

**Caching:**

- **Redis Cache-Aside**: Cache metadata for popular videos (TTL 1 hour)
- **Cache Key**: `video:{video_id}`
- **Cache Hit Rate**: 80-90% for popular content

*See pseudocode.md::getVideoMetadata() for implementation*

### 4.6 Search and Discovery

**Challenge**: Users search for "cat videos", "how to cook pasta", etc. Need full-text search.

**Solution**: Elasticsearch

**Architecture:**

```
User → API Gateway → Elasticsearch → Return Results
                            ↑
                            │
                    Sync from Cassandra
                    (Change Data Capture)
```

**Search Index:**

```
{
  "video_id": "video_123",
  "title": "Funny Cat Compilation",
  "description": "Top 10 funny cat moments",
  "tags": ["cat", "funny", "animals"],
  "transcript": "full video transcript...",
  "upload_date": "2024-01-15",
  "views": 1000000,
  "duration": 600
}
```

**Search Query:**

```
GET /videos/_search
{
  "query": {
    "multi_match": {
      "query": "funny cat",
      "fields": ["title^3", "tags^2", "description", "transcript"]
    }
  },
  "sort": [
    {"views": "desc"},
    {"_score": "desc"}
  ]
}
```

**Ranking Factors:**

- **Relevance Score**: TF-IDF and BM25 algorithms
- **Popularity**: Views, likes, recency
- **Personalization**: User watch history, preferences

*See pseudocode.md::searchVideos() for implementation*

### 4.7 Recommendation Engine

**Challenge**: Suggest relevant videos to keep users engaged.

**Approaches:**

1. **Collaborative Filtering**: Users who watched A also watched B
2. **Content-Based**: Recommend similar videos based on tags, category
3. **Hybrid**: Combine both approaches

**Architecture:**

```
User Watch History → Feature Extraction → ML Model → Ranked Recommendations
```

**ML Model:**

- **Matrix Factorization**: SVD, ALS for collaborative filtering
- **Deep Learning**: Neural networks for complex patterns
- **Real-Time**: Update recommendations after each video watched

**Features:**

- User watch history (last 100 videos)
- Video metadata (tags, category, duration)
- User demographics (age, location, language)
- Time of day, device type

**Offline Batch Processing:**

- Pre-compute recommendations for all users (daily)
- Store in Redis for fast retrieval

**Online Real-Time:**

- Update recommendations after each video watched
- Use Kafka for real-time event streaming

*See pseudocode.md::generateRecommendations() for implementation*

---

## 5. Upload Flow (Step-by-Step)

**User uploads a 5 GB video:**

1. **Client**: Split file into 500 chunks (10 MB each)
2. **Client → Upload Service**: POST `/upload/init` → Returns `upload_id`
3. **For each chunk**:
    - Client → Upload Service: POST `/upload/chunk` (chunk_id, data)
    - Upload Service → S3: Multipart upload chunk
    - Upload Service → Redis: Mark chunk as complete
    - Upload Service → Client: Acknowledge chunk
4. **Client → Upload Service**: POST `/upload/complete` (upload_id)
5. **Upload Service**: Assemble chunks into single file in S3
6. **Upload Service → Kafka**: Publish transcode job
7. **Upload Service → Client**: Return video_id, status: "Processing"

**Total Time**: ~10 minutes for 5 GB (assuming 10 Mbps upload speed)

*See sequence-diagrams.md for detailed flow*

---

## 6. Transcoding Flow (Step-by-Step)

**Worker processes transcode job:**

1. **Worker**: Consume message from Kafka queue
2. **Worker → S3**: Download raw video (5 GB)
3. **Worker**: Extract audio and video tracks
4. **For each quality** (144p, 360p, 720p, 1080p, 4K):
    - Encode video with FFmpeg
    - Segment into 10-second chunks
    - Upload segments to S3
5. **Worker**: Generate HLS manifest (`.m3u8`)
6. **Worker → S3**: Upload manifest
7. **Worker → Metadata DB**: Update video status to "Ready"
8. **Worker → CDN**: Invalidate cache for new video
9. **Worker → Kafka**: Acknowledge job complete

**Total Time**: ~30 minutes for 5-minute video (6x real-time processing)

*See sequence-diagrams.md for detailed flow*

---

## 7. Streaming Flow (Step-by-Step)

**User watches a video:**

1. **Client → API Gateway**: GET `/video/:video_id`
2. **API Gateway → Redis**: Check cache for metadata
3. **If cache miss**:
    - API Gateway → Cassandra: Query video metadata
    - API Gateway → Redis: Cache metadata (TTL 1 hour)
4. **API Gateway → Client**: Return metadata + HLS manifest URL
5. **Client → CDN**: GET `/video_123/manifest.m3u8`
6. **CDN**: Parse manifest, return to client
7. **Client**: Estimate bandwidth, select quality (e.g., 720p)
8. **Client → CDN**: GET `/video_123/720p/segment_001.ts`
9. **CDN**: Return segment (cache hit 99%)
10. **Client**: Download and buffer next 3 segments
11. **Client**: Monitor bandwidth, switch quality if needed
12. **Repeat** for all segments until video ends

**Total Startup Time**: <2 seconds (fetch metadata + first segment)

*See sequence-diagrams.md for detailed flow*

---

## 8. Scalability and Performance Optimizations

### 8.1 Transcoding Optimization

**Challenge**: 500k videos/day × 30 min processing = 10.4M CPU-hours/day

**Solutions:**

1. **Parallel Processing**: Transcode all qualities in parallel (not sequential)
2. **Spot Instances**: Use AWS Spot for 70% cost savings
3. **Prioritization**: High-priority videos (verified channels) transcoded first
4. **Incremental Encoding**: Reuse keyframes across qualities
5. **Hardware Acceleration**: Use GPU for H.265 encoding (5x faster)

### 8.2 CDN Optimization

**Challenge**: 300 PB/day bandwidth = $3M/day cost at $0.01/GB

**Solutions:**

1. **Aggressive Caching**: 99% cache hit rate at edge
2. **CDN Negotiation**: Volume discounts with CloudFront, Akamai
3. **Compression**: Use H.265 (50% smaller than H.264)
4. **P2P Delivery**: Experimental peer-to-peer for popular videos

### 8.3 Storage Optimization

**Challenge**: 500 PB storage = $10M/month at $0.02/GB

**Solutions:**

1. **Cold Storage**: Move old videos to Glacier ($0.004/GB)
2. **Deduplication**: Detect and remove duplicate uploads
3. **Compression**: Use efficient codecs (H.265, AV1)
4. **Retention Policy**: Delete videos with <10 views after 1 year

### 8.4 Database Optimization

**Challenge**: 50k QPS for metadata reads

**Solutions:**

1. **Redis Caching**: 80% cache hit rate (10ms → 1ms)
2. **Cassandra Partitioning**: Shard by video_id
3. **Read Replicas**: 5 replicas per region for high availability
4. **Denormalization**: Store CDN URLs directly in metadata (no JOIN)

---

## 9. Availability and Fault Tolerance

### 9.1 Multi-Region Deployment

**Architecture:**

- **3 Regions**: US-East, EU-West, Asia-Pacific
- **Active-Active**: All regions serve traffic
- **Geo-Routing**: Route users to nearest region

**Replication:**

- **Metadata**: Multi-master Cassandra (eventual consistency)
- **Videos**: Replicate to 2 regions (S3 Cross-Region Replication)
- **CDN**: Automatically distributed globally

### 9.2 Failure Scenarios

**Scenario 1: Transcoding Worker Crashes**

- **Impact**: Job not completed
- **Recovery**: Kafka redelivers message after timeout
- **Prevention**: Idempotent workers (check if already processed)

**Scenario 2: CDN Edge Failure**

- **Impact**: Users routed to degraded edge
- **Recovery**: CDN auto-routes to next nearest edge
- **Prevention**: Health checks every 10 seconds

**Scenario 3: Metadata DB Failure**

- **Impact**: Cannot fetch video metadata
- **Recovery**: Failover to read replica (1 second)
- **Prevention**: Multi-region replication

**Scenario 4: S3 Outage**

- **Impact**: Cannot upload/download videos
- **Recovery**: Queue uploads, retry after recovery
- **Prevention**: Cross-region replication

---

## 10. Common Anti-Patterns

### ❌ **1. Storing Videos in Database**

**Problem**: Videos are large blobs, databases are not optimized for blob storage

**Why Bad**:

- Database IOPS exhausted (100s of MB/sec)
- Expensive (PostgreSQL storage $0.10/GB vs S3 $0.02/GB)
- Cannot leverage CDN (database not HTTP-accessible)

**Solution**: ✅ Store videos in S3, store metadata in database

---

### ❌ **2. Synchronous Transcoding**

**Problem**: User waits 30 minutes for video to process before upload completes

**Why Bad**:

- Terrible user experience (timeout)
- API Gateway connection times out (60 sec max)
- Cannot scale (one request = one worker)

**Solution**: ✅ Asynchronous transcoding with Kafka queue

---

### ❌ **3. Single Quality Encoding**

**Problem**: Only encode video at 1080p, let client scale down

**Why Bad**:

- Wastes bandwidth (mobile users download 1080p, display 360p)
- Poor UX on slow networks (constant buffering)
- Higher costs ($8 Mbps for 1080p vs $0.8 Mbps for 360p)

**Solution**: ✅ Adaptive bitrate streaming with multiple qualities

---

### ❌ **4. No CDN (Direct Streaming from Origin)**

**Problem**: Stream all videos directly from S3

**Why Bad**:

- High latency (S3 in one region, users worldwide)
- Bandwidth costs 10x higher ($0.09/GB S3 vs $0.01/GB CDN)
- S3 request limits (5,500 GET/sec per prefix)

**Solution**: ✅ Always use CDN for video delivery

---

### ❌ **5. Monolithic Transcode Service**

**Problem**: Single service handles upload, transcode, and streaming

**Why Bad**:

- Cannot scale independently (upload ≠ transcode ≠ streaming load)
- Deployment risk (update transcode logic, break streaming)
- Resource contention (CPU-heavy transcode starves API)

**Solution**: ✅ Microservices: Upload Service, Transcode Workers, Streaming API

---

## 11. Alternative Approaches

### 11.1 Peer-to-Peer (P2P) Delivery

**Approach**: Users download from each other (like BitTorrent)

**Pros**:

- Reduce CDN bandwidth costs by 50%
- Scales automatically with more viewers

**Cons**:

- Requires client-side software
- Privacy concerns (users see each other's IPs)
- Unreliable (peers may go offline)

**Used By**: Popcorn Time, Twitch (experimental)

### 11.2 Live Streaming Architecture

**Differences from VOD**:

- **No Pre-Transcoding**: Must transcode in real-time
- **Ultra-Low Latency**: <5 seconds delay (vs 30 seconds for VOD)
- **Protocols**: WebRTC, RTMP, SRT instead of HLS

**Architecture**:

```
Streamer → Ingest (RTMP) → Live Transcoder → CDN → Viewers
```

**Used By**: Twitch, YouTube Live, Facebook Live

### 11.3 User-Generated Content (UGC) Platform

**Example**: YouTube (anyone can upload)

**Additional Requirements**:

- **Content Moderation**: AI to detect inappropriate content
- **Copyright Detection**: Content ID system
- **Monetization**: Ads, creator revenue sharing
- **Channels & Subscriptions**: User profiles, followers

**Used By**: YouTube, TikTok, Vimeo

---

## 12. Monitoring and Observability

### 12.1 Key Metrics

**Upload Service**:

- Upload success rate (target: 99.9%)
- Average upload time
- Chunk failure rate

**Transcoding**:

- Queue depth (Kafka lag)
- Processing time per video
- Worker utilization (CPU, memory)
- Transcode success rate (target: 99.5%)

**Streaming**:

- Startup latency (p50, p95, p99)
- Buffering rate (target: <1% of playback time)
- CDN cache hit rate (target: 99%)
- Bandwidth usage (TB/day)

**Metadata DB**:

- Read/write QPS
- Latency (p50, p95, p99)
- Cache hit rate (target: 80%)

### 12.2 Alerts

- **Critical**: Transcode queue depth >100k (scale up workers)
- **Critical**: CDN cache hit rate <90% (investigate)
- **High**: Video startup latency >5s
- **High**: Upload success rate <95%
- **Medium**: Transcoding success rate <99%

### 12.3 Logging

- **Structured Logging**: JSON format for all services
- **Trace IDs**: Track request across services
- **Log Aggregation**: ELK Stack (Elasticsearch, Logstash, Kibana)

---

## 13. Interview Discussion Points

### 14.1 Why Kafka Over SQS for Transcode Queue?

**Kafka Advantages**:

- **Ordering**: Guarantee FIFO per partition
- **Replay**: Reprocess failed jobs
- **Throughput**: 1M+ msg/sec

**When to Use SQS**:

- Small scale (<10k videos/day)
- Want fully managed service
- Don't need ordering

### 14.2 Why NoSQL Over SQL for Metadata?

**Cassandra Advantages**:

- **Horizontal Scalability**: Add nodes to scale
- **Low Latency**: Single-digit ms reads
- **Multi-Region**: Built-in replication

**When to Use PostgreSQL**:

- Need complex queries (JOINs, aggregations)
- Small scale (<1M videos)
- ACID transactions required

### 14.3 HLS vs DASH?

**HLS (Apple)**:

- Better iOS/Safari support
- Simpler implementation
- Industry standard

**DASH (MPEG)**:

- Open standard
- More flexible
- Better for live streaming

**Decision**: Use HLS for broader compatibility

### 14.4 How to Handle Copyright Detection?

**Content ID System**:

1. **Fingerprinting**: Generate audio/video fingerprint (perceptual hash)
2. **Matching**: Compare against database of copyrighted content
3. **Action**: Block, mute audio, or monetize (ads)

**Implementation**:

- Use ML models for fingerprinting
- Store fingerprints in Elasticsearch
- Run detection during transcoding

---

## 14. Trade-offs Summary

| What We Gain                             | What We Sacrifice                      |
|------------------------------------------|----------------------------------------|
| ✅ Global low-latency streaming           | ❌ High CDN costs ($90M/month)          |
| ✅ Adaptive bitrate (smooth playback)     | ❌ 5x storage (multiple qualities)      |
| ✅ Asynchronous transcoding (fast upload) | ❌ Delayed availability (30 min wait)   |
| ✅ Horizontal scalability (NoSQL)         | ❌ Limited query flexibility (no JOINs) |
| ✅ Resumable uploads (reliability)        | ❌ Complex client logic                 |
| ✅ Auto-scaling workers (cost efficiency) | ❌ Cold start delays (spot instances)   |

---

## 15. Real-World Examples

### Netflix

- **CDN**: Custom CDN (Open Connect) deployed in ISPs
- **Encoding**: Per-title encoding (optimized bitrate per video)
- **Storage**: AWS S3 + proprietary systems
- **Scale**: 8 trillion hours watched annually

### YouTube

- **Upload**: Resumable uploads (YouTube API)
- **Transcoding**: Massive fleet of custom transcoders
- **Delivery**: Google's global CDN
- **Scale**: 500 hours of video uploaded every minute

### Twitch

- **Focus**: Live streaming (not VOD)
- **Latency**: <3 seconds (ultra-low latency)
- **Transcoding**: Real-time transcoding for live streams
- **Scale**: 140 million monthly active users

---

## 16. References

### Related System Design Components

- **[2.0.4 HTTP and REST APIs](../../02-components/2.0-communication/2.0.4-http-rest-deep-dive.md)** - Upload APIs,
  chunked transfers
- **[2.1.9 Cassandra Deep Dive](../../02-components/2.1-databases/2.1.9-cassandra-deep-dive.md)** - Metadata storage
- **[2.2.1 Caching Deep Dive](../../02-components/2.2.1-caching-deep-dive.md)** - CDN caching strategies
- **[2.3.1 Asynchronous Communication](../../02-components/2.3-messaging-streaming/2.3.1-asynchronous-communication.md)
  ** - Message queues for transcoding
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Transcoding queue
  implementation
- **[2.1.13 Elasticsearch Deep Dive](../../02-components/2.1-databases/2.1.13-elasticsearch-deep-dive.md)** - Video
  search and discovery
- **[2.1.4 Database Scaling](../../02-components/2.1.4-database-scaling.md)** - Database sharding strategies

### Related Design Challenges

- **[3.2.1 Twitter Timeline](../3.2.1-twitter-timeline/)** - Fanout patterns for recommendations
- **[3.4.7 Online Code Judge](../3.4.7-online-code-judge/)** - Worker fleet architecture
- **[3.4.4 Recommendation System](../3.4.4-recommendation-system/)** - Video recommendation algorithms

### External Resources

- **Netflix Tech Blog:** [Netflix Engineering Blog](https://netflixtechblog.com/) - Video streaming architecture
- **YouTube Architecture:** [How YouTube Works](https://www.youtube.com/intl/en-GB/howyoutubeworks/) - YouTube system
  design
- **FFmpeg Documentation:** [FFmpeg](https://ffmpeg.org/documentation.html) - Video transcoding library
- **HLS Specification:** [RFC 8216](https://datatracker.ietf.org/doc/html/rfc8216) - HTTP Live Streaming protocol
- **DASH Specification:** [DASH Industry Forum](https://dashif.org/) - Dynamic Adaptive Streaming over HTTP

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - CDN architecture, video streaming patterns
- *High Performance Browser Networking* by Ilya Grigorik - HTTP/2, video streaming protocols