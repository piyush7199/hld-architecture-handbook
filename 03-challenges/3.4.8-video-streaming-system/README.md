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

## Problem Statement

Design a video streaming platform like YouTube or Netflix that allows users to upload, process, store, and stream video
content
globally with high availability and low latency. The system must handle massive scale (petabytes of data, 100M+
concurrent
viewers), support adaptive bitrate streaming, and minimize bandwidth costs while providing seamless viewing experience.

**Key Challenges**: Massive storage, high bandwidth (90% of internet traffic), complex transcoding, global distribution
with <2s
startup latency, cost optimization.

---

## Requirements and Scale Estimation

### Functional Requirements

1. **Video Upload**: Accept large files (up to 50 GB)
2. **Video Transcoding**: Convert to multiple formats (144p to 4K)
3. **Streaming (VOD)**: Deliver efficiently with minimal buffering
4. **Playback Control**: Seeking, pausing, resuming, speed control
5. **Search and Discovery**: Find videos by title, tags
6. **Recommendations**: Suggest relevant videos

### Non-Functional Requirements

1. **High Throughput**: Serve petabytes daily (~90% of internet traffic)
2. **Low Latency**: Video startup <2 seconds
3. **Fault Tolerance**: Resilient transcoding
4. **Cost Efficiency**: Minimize bandwidth and storage costs
5. **High Availability**: 99.99% uptime

### Scale Estimation

| Metric                   | Value               | Notes                 |
|--------------------------|---------------------|-----------------------|
| **Total Users**          | 2B registered       | Massive user base     |
| **DAU**                  | 500M                | 25% daily active      |
| **Concurrent Viewers**   | 100M peak           | Prime time traffic    |
| **Videos Uploaded/Day**  | 500k                | 5.8 videos/sec        |
| **Daily Upload Storage** | 250 TB/day          | ~90 PB/year           |
| **Total Catalog**        | 1B videos           | ~500 PB storage       |
| **Daily Data Transfer**  | 300 PB/day          | Massive CDN load      |
| **Bandwidth**            | 28 Tbps             | Aggregate bandwidth   |
| **Transcoding**          | 10.4M CPU-hours/day | Requires 10k+ servers |

**Key Insights**: Storage grows at 90 PB/year, bandwidth dominates costs, transcoding is CPU-intensive, read:write ratio
is 1000:1.

---

## High-Level Architecture

The architecture is dominated by the **CDN** (serving read traffic) and the **Transcoding Pipeline** (processing write
traffic).

### Core Components

**Write Path**:

```
User → Upload Service → Raw Storage (S3) → Transcoding Queue (Kafka) → Worker Fleet → Processed Storage → CDN
```

**Read Path**:

```
User → API Gateway → Metadata DB (Cassandra) → CDN (CloudFront) → User
```

### Component Overview

1. **Upload Service**: Resumable uploads via chunked HTTP/gRPC
2. **Raw Storage (S3)**: Original unprocessed videos
3. **Transcoding Queue (Kafka)**: Buffers transcoding jobs
4. **Transcoding Worker Fleet**: CPU-heavy auto-scaling cluster
5. **Processed Storage (S3)**: Transcoded video segments (HLS/DASH)
6. **CDN**: Caches and serves video segments globally
7. **Metadata Database (Cassandra/DynamoDB)**: Video titles, CDN URLs
8. **Search Service (Elasticsearch)**: Full-text search
9. **Recommendation Engine**: ML-based suggestions
10. **Analytics Pipeline**: Views, watch time, engagement

---

## Detailed Component Design

### Upload Service and Resumable Uploads

**Challenge**: Videos can be 50 GB+. Network interruptions common. Users shouldn't restart uploads.

**Solution**: Resumable Uploads (Chunked HTTP or gRPC Streaming)

**How It Works**:

1. Client splits file into chunks (10 MB each)
2. Upload each chunk with metadata (chunk_id, video_id, chunk_number)
3. Server acknowledges each chunk, stores in S3
4. If upload fails, client resumes from last acknowledged chunk
5. After all chunks uploaded, server assembles file and triggers transcoding

**Implementation**:

- Use **TUS Protocol** (Resumable Upload Protocol) over HTTP
- Or **gRPC Bidirectional Streaming** for efficient binary transfer
- Store upload state in **Redis** (tracks completed chunks)
- Use **S3 Multipart Upload API** (supports up to 10,000 parts, 5 GB each)

*See pseudocode.md::uploadVideoChunked() for implementation*

### Transcoding Pipeline

**Challenge**: Convert raw video into multiple formats (144p, 360p, 720p, 1080p, 4K) with different codecs (H.264,
H.265, VP9).

**Architecture**:

```
Raw Video → Kafka Queue → Worker Fleet → Processed Segments → S3 → CDN
```

**Transcoding Process**:

1. Extract audio and video tracks separately
2. Encode video into multiple resolutions
3. Encode audio into multiple bitrates
4. Segment video into small chunks (2-10 seconds each)
5. Generate manifest files (HLS `.m3u8` or DASH `.mpd`)
6. Upload to S3 and invalidate CDN cache

**Transcoding Formats**:

| Quality   | Resolution | Video Bitrate | Audio Bitrate | Use Case   |
|-----------|------------|---------------|---------------|------------|
| **144p**  | 256×144    | 200 Kbps      | 64 Kbps       | Mobile 2G  |
| **360p**  | 640×360    | 800 Kbps      | 96 Kbps       | Mobile 3G  |
| **480p**  | 854×480    | 1.5 Mbps      | 128 Kbps      | Mobile 4G  |
| **720p**  | 1280×720   | 5 Mbps        | 128 Kbps      | HD Desktop |
| **1080p** | 1920×1080  | 8 Mbps        | 192 Kbps      | Full HD    |
| **4K**    | 3840×2160  | 25 Mbps       | 256 Kbps      | Ultra HD   |

**Worker Fleet**:

- **Auto-Scaling**: Scale based on Kafka queue depth
- **Spot Instances**: Use AWS Spot for 70% cost savings
- **Dedicated Instances**: For live streams or high-priority content
- **FFmpeg**: Open-source tool for video processing

*See pseudocode.md::transcodeVideo() for implementation*

### Adaptive Bitrate Streaming (ABR)

**Challenge**: Users have varying network conditions. Must switch quality dynamically without interrupting playback.

**Solution**: HLS (HTTP Live Streaming) or DASH (Dynamic Adaptive Streaming over HTTP)

**How HLS Works**:

1. Server provides manifest file (`.m3u8`) listing all available qualities
2. Client downloads manifest, selects initial quality based on bandwidth estimate
3. Client downloads video segments (2-10 sec chunks) via HTTP
4. Client measures download speed for each segment
5. Client switches quality if bandwidth increases/decreases

**Example HLS Manifest**:

```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=854x480
480p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1280x720
720p/playlist.m3u8
```

**Benefits**: No buffering, quality switching, HTTP-based (works through firewalls).

*See pseudocode.md::selectBitrate() for ABR algorithm*

### CDN Strategy and Edge Caching

**Challenge**: Bandwidth costs dominate (90% of expenses). Must minimize origin requests.

**Solution**: Multi-Tier CDN with Aggressive Caching

**CDN Tiers**:

```
User → Edge (99% hit) → Regional (95% hit) → Origin (S3)
```

1. **Edge Locations** (1000+): Close to users, cache hot content
2. **Regional Caches** (50+): Aggregate traffic from multiple edges
3. **Origin (S3)**: Stores all content, serves cache misses

**Caching Strategy**:

- **Hot Content**: Cache for 7 days at edge
- **Warm Content**: Cache for 24 hours at regional
- **Cold Content**: Fetch from origin on demand

**Cache Key**: `video_id + quality + segment_number` (e.g., `video_123/720p/segment_042.ts`)

**CDN Optimization**:

- **Prefetching**: CDN prefetches next segment before client requests
- **Range Requests**: Support HTTP Range headers for seeking
- **Compression**: Use Gzip/Brotli for manifest files
- **Geo-Routing**: Route users to nearest edge location

*See pseudocode.md::calculateCDNCache() for cache TTL logic*

### Metadata Management

**Challenge**: Store metadata for 1 billion videos with low-latency reads.

**Database Choice**: Cassandra or DynamoDB (NoSQL)

**Why NoSQL?**

- **Horizontal Scalability**: Partition by video_id across many nodes
- **Low Latency**: Single-digit millisecond reads
- **High Availability**: Multi-region replication

**Schema**:

```
Table: videos
Partition Key: video_id
Columns: title, description, upload_date, duration, views, likes, channel_id, formats, thumbnail_url, tags
```

**Caching**:

- **Redis Cache-Aside**: Cache metadata for popular videos (TTL 1 hour)
- **Cache Key**: `video:{video_id}`
- **Cache Hit Rate**: 80-90% for popular content

*See pseudocode.md::getVideoMetadata() for implementation*

### Search and Discovery

**Challenge**: Users search for "cat videos", "how to cook pasta", etc. Need full-text search.

**Solution**: Elasticsearch

**Architecture**:

```
User → API Gateway → Elasticsearch → Return Results
```

**Search Index**:

```json
{
  "video_id": "video_123",
  "title": "Funny Cat Compilation",
  "description": "Top 10 funny cat moments",
  "tags": [
    "cat",
    "funny",
    "animals"
  ],
  "transcript": "full video transcript...",
  "upload_date": "2024-01-15",
  "views": 1000000,
  "duration": 600
}
```

**Ranking Factors**:

- **Relevance Score**: TF-IDF and BM25 algorithms
- **Popularity**: Views, likes, recency
- **Personalization**: User watch history, preferences

*See pseudocode.md::searchVideos() for implementation*

### Recommendation Engine

**Challenge**: Suggest relevant videos to keep users engaged.

**Approaches**:

1. **Collaborative Filtering**: Users who watched A also watched B
2. **Content-Based**: Recommend similar videos based on tags, category
3. **Hybrid**: Combine both approaches

**Architecture**:

```
User Watch History → Feature Extraction → ML Model → Ranked Recommendations
```

**ML Model**:

- **Matrix Factorization**: SVD, ALS for collaborative filtering
- **Deep Learning**: Neural networks for complex patterns
- **Real-Time**: Update recommendations after each video watched

**Features**: User watch history (last 100 videos), video metadata (tags, category, duration), user demographics (age,
location,
language), time of day, device type.

**Offline Batch Processing**: Pre-compute recommendations for all users (daily), store in Redis for fast retrieval.

**Online Real-Time**: Update recommendations after each video watched, use Kafka for real-time event streaming.

*See pseudocode.md::generateRecommendations() for implementation*

---

## Upload Flow (Step-by-Step)

**User uploads a 5 GB video**:

1. **Client**: Split file into 500 chunks (10 MB each)
2. **Client → Upload Service**: POST `/upload/init` → Returns `upload_id`
3. **For each chunk**: Upload chunk, server stores in S3, acknowledges
4. **Client → Upload Service**: POST `/upload/complete` (upload_id)
5. **Upload Service**: Assemble chunks, publish transcode job to Kafka
6. **Upload Service → Client**: Return video_id, status: "Processing"

**Total Time**: ~10 minutes for 5 GB (assuming 10 Mbps upload speed)

*See sequence-diagrams.md for detailed flow*

---

## Transcoding Flow (Step-by-Step)

**Worker processes transcode job**:

1. **Worker**: Consume message from Kafka queue
2. **Worker → S3**: Download raw video (5 GB)
3. **Worker**: Extract audio and video tracks
4. **For each quality**: Encode video with FFmpeg, segment into chunks, upload to S3
5. **Worker**: Generate HLS manifest, upload to S3
6. **Worker → Metadata DB**: Update video status to "Ready"
7. **Worker → CDN**: Invalidate cache for new video
8. **Worker → Kafka**: Acknowledge job complete

**Total Time**: ~30 minutes for 5-minute video (6x real-time processing)

*See sequence-diagrams.md for detailed flow*

---

## Streaming Flow (Step-by-Step)

**User watches a video**:

1. **Client → API Gateway**: GET `/video/:video_id`
2. **API Gateway → Redis**: Check cache for metadata
3. **If cache miss**: Query Cassandra, cache result
4. **API Gateway → Client**: Return metadata + HLS manifest URL
5. **Client → CDN**: GET `/video_123/manifest.m3u8`
6. **CDN**: Parse manifest, return to client
7. **Client**: Estimate bandwidth, select quality (e.g., 720p)
8. **Client → CDN**: GET `/video_123/720p/segment_001.ts`
9. **CDN**: Return segment (cache hit 99%)
10. **Client**: Download and buffer next 3 segments
11. **Client**: Monitor bandwidth, switch quality if needed
12. **Repeat** for all segments

**Total Startup Time**: <2 seconds (fetch metadata + first segment)

*See sequence-diagrams.md for detailed flow*

---

## Scalability and Performance Optimizations

### Transcoding Optimization

**Challenge**: 500k videos/day × 30 min processing = 10.4M CPU-hours/day

**Solutions**:

1. **Parallel Processing**: Transcode all qualities in parallel
2. **Spot Instances**: Use AWS Spot for 70% cost savings
3. **Prioritization**: High-priority videos (verified channels) transcoded first
4. **Incremental Encoding**: Reuse keyframes across qualities
5. **Hardware Acceleration**: Use GPU for H.265 encoding (5x faster)

### CDN Optimization

**Challenge**: 300 PB/day bandwidth = $3M/day cost at $0.01/GB

**Solutions**:

1. **Aggressive Caching**: 99% cache hit rate at edge
2. **CDN Negotiation**: Volume discounts with CloudFront, Akamai
3. **Compression**: Use H.265 (50% smaller than H.264)
4. **P2P Delivery**: Experimental peer-to-peer for popular videos

### Storage Optimization

**Challenge**: 500 PB storage = $10M/month at $0.02/GB

**Solutions**:

1. **Cold Storage**: Move old videos to Glacier ($0.004/GB)
2. **Deduplication**: Detect and remove duplicate uploads
3. **Compression**: Use efficient codecs (H.265, AV1)
4. **Retention Policy**: Delete videos with <10 views after 1 year

### Database Optimization

**Challenge**: 50k QPS for metadata reads

**Solutions**:

1. **Redis Caching**: 80% cache hit rate (10ms → 1ms)
2. **Cassandra Partitioning**: Shard by video_id
3. **Read Replicas**: 5 replicas per region for high availability
4. **Denormalization**: Store CDN URLs directly (no JOIN)

---

## Availability and Fault Tolerance

### Multi-Region Deployment

**Architecture**:

- **3 Regions**: US-East, EU-West, Asia-Pacific
- **Active-Active**: All regions serve traffic
- **Geo-Routing**: Route users to nearest region

**Replication**:

- **Metadata**: Multi-master Cassandra (eventual consistency)
- **Videos**: Replicate to 2 regions (S3 Cross-Region Replication)
- **CDN**: Automatically distributed globally

### Failure Scenarios

**Transcoding Worker Crashes**: Kafka redelivers message after timeout, idempotent workers check if already processed.

**CDN Edge Failure**: CDN auto-routes to next nearest edge, health checks every 10 seconds.

**Metadata DB Failure**: Failover to read replica (1 second), multi-region replication.

**S3 Outage**: Queue uploads, retry after recovery, cross-region replication.

---

## Common Anti-Patterns

### ❌ **1. Storing Videos in Database**

**Problem**: Videos are large blobs, databases not optimized for blob storage.

**Why Bad**: Database IOPS exhausted, expensive ($0.10/GB vs $0.02/GB S3), cannot leverage CDN.

**Solution**: ✅ Store videos in S3, store metadata in database.

---

### ❌ **2. Synchronous Transcoding**

**Problem**: User waits 30 minutes for video to process.

**Why Bad**: Terrible UX (timeout), API Gateway connection times out, cannot scale.

**Solution**: ✅ Asynchronous transcoding with Kafka queue.

---

### ❌ **3. Single Quality Encoding**

**Problem**: Only encode at 1080p, let client scale down.

**Why Bad**: Wastes bandwidth, poor UX on slow networks, higher costs.

**Solution**: ✅ Adaptive bitrate streaming with multiple qualities.

---

### ❌ **4. No CDN (Direct Streaming from Origin)**

**Problem**: Stream all videos directly from S3.

**Why Bad**: High latency, bandwidth costs 10x higher, S3 request limits.

**Solution**: ✅ Always use CDN for video delivery.

---

### ❌ **5. Monolithic Transcode Service**

**Problem**: Single service handles upload, transcode, and streaming.

**Why Bad**: Cannot scale independently, deployment risk, resource contention.

**Solution**: ✅ Microservices: Upload Service, Transcode Workers, Streaming API.

---

## Alternative Approaches

### Peer-to-Peer (P2P) Delivery

**Approach**: Users download from each other (like BitTorrent).

**Pros**: Reduce CDN bandwidth costs by 50%, scales automatically with more viewers.

**Cons**: Requires client-side software, privacy concerns, unreliable (peers may go offline).

**Used By**: Popcorn Time, Twitch (experimental).

### Live Streaming Architecture

**Differences from VOD**: No pre-transcoding (must transcode in real-time), ultra-low latency (<5 seconds delay),
protocols:
WebRTC, RTMP, SRT instead of HLS.

**Architecture**: `Streamer → Ingest (RTMP) → Live Transcoder → CDN → Viewers`

**Used By**: Twitch, YouTube Live, Facebook Live.

### User-Generated Content (UGC) Platform

**Example**: YouTube (anyone can upload).

**Additional Requirements**: Content moderation (AI to detect inappropriate content), copyright detection (Content ID
system),
monetization (ads, creator revenue sharing), channels & subscriptions (user profiles, followers).

**Used By**: YouTube, TikTok, Vimeo.

---

## Monitoring and Observability

### Key Metrics

**Upload Service**: Upload success rate (target: 99.9%), average upload time, chunk failure rate.

**Transcoding**: Queue depth (Kafka lag), processing time per video, worker utilization, transcode success rate (target:
99.5%).

**Streaming**: Startup latency (p50, p95, p99), buffering rate (target: <1% of playback time), CDN cache hit rate (
target: 99%),
bandwidth usage (TB/day).

**Metadata DB**: Read/write QPS, latency (p50, p95, p99), cache hit rate (target: 80%).

### Alerts

- **Critical**: Transcode queue depth >100k (scale up workers), CDN cache hit rate <90% (investigate)
- **High**: Video startup latency >5s, upload success rate <95%
- **Medium**: Transcoding success rate <99%

### Logging

- **Structured Logging**: JSON format for all services
- **Trace IDs**: Track request across services
- **Log Aggregation**: ELK Stack (Elasticsearch, Logstash, Kibana)

---

## Cost Analysis

### Cost Breakdown (Monthly for 100M Users)

| Category              | Usage                   | Unit Cost  | Total Cost | % of Total |
|-----------------------|-------------------------|------------|------------|------------|
| **CDN Bandwidth**     | 9 PB (300 PB/day × 30%) | $0.01/GB   | $90M       | 75%        |
| **Storage (S3)**      | 500 PB                  | $0.02/GB   | $10M       | 8%         |
| **Transcoding (EC2)** | 300M CPU-hours          | $0.05/hour | $15M       | 13%        |
| **Metadata DB**       | Cassandra cluster       | $0.50/hour | $3.6M      | 3%         |
| **Other Services**    | API, Search, etc.       | -          | $1.4M      | 1%         |
| **Total**             | -                       | -          | **$120M**  | 100%       |

**Key Insights**: CDN is 75% of costs → Optimize cache hit rate. Storage grows linearly → Use cold storage. Transcoding
is
one-time → Use spot instances.

### Cost Optimization Strategies

1. **CDN**: Negotiate volume discounts (50% reduction at 10 PB/month)
2. **Storage**: Move videos with <100 views/year to Glacier ($0.004/GB)
3. **Transcoding**: Use spot instances (70% cheaper than on-demand)
4. **Compression**: Use H.265 (50% smaller, save $45M/month CDN)

---

## Trade-offs Summary

| What We Gain                             | What We Sacrifice                      |
|------------------------------------------|----------------------------------------|
| ✅ Global low-latency streaming           | ❌ High CDN costs ($90M/month)          |
| ✅ Adaptive bitrate (smooth playback)     | ❌ 5x storage (multiple qualities)      |
| ✅ Asynchronous transcoding (fast upload) | ❌ Delayed availability (30 min wait)   |
| ✅ Horizontal scalability (NoSQL)         | ❌ Limited query flexibility (no JOINs) |
| ✅ Resumable uploads (reliability)        | ❌ Complex client logic                 |
| ✅ Auto-scaling workers (cost efficiency) | ❌ Cold start delays (spot instances)   |

---

## Real-World Examples

**Netflix**: Custom CDN (Open Connect) deployed in ISPs, per-title encoding (optimized bitrate per video), AWS S3 +
proprietary
systems, 8 trillion hours watched annually.

**YouTube**: Resumable uploads (YouTube API), massive fleet of custom transcoders, Google's global CDN, 500 hours of
video uploaded
every minute.

**Twitch**: Focus on live streaming (not VOD), <3 seconds latency (ultra-low), real-time transcoding for live streams,
140 million
monthly active users.

---

## Interview Discussion Points

### Why Kafka Over SQS for Transcode Queue?

**Kafka Advantages**:

- **Ordering**: Guarantee FIFO per partition
- **Replay**: Reprocess failed jobs
- **Throughput**: 1M+ msg/sec
- **Retention**: Store messages for days (debugging)

**When to Use SQS**:

- Small scale (<10k videos/day)
- Want fully managed service
- Don't need ordering guarantees

**Decision**: Use Kafka for high throughput and replay capability.

### Why NoSQL Over SQL for Metadata?

**Cassandra Advantages**:

- **Horizontal Scalability**: Add nodes to scale linearly
- **Low Latency**: Single-digit ms reads (no disk seeks)
- **Multi-Region**: Built-in replication across continents
- **No Single Point of Failure**: Peer-to-peer architecture

**When to Use PostgreSQL**:

- Need complex queries (JOINs, aggregations)
- Small scale (<1M videos)
- ACID transactions required
- Relational data model important

**Decision**: Use Cassandra for massive scale and low latency.

### HLS vs DASH?

**HLS (Apple)**:

- Better iOS/Safari support
- Simpler implementation
- Industry standard (used by most platforms)
- Mature ecosystem

**DASH (MPEG)**:

- Open standard (not proprietary)
- More flexible codec support
- Better for live streaming
- Cross-platform (no vendor lock-in)

**Decision**: Use HLS for broader compatibility and simpler implementation.

### How to Handle Copyright Detection?

**Content ID System**:

1. **Fingerprinting**: Generate audio/video fingerprint (perceptual hash)
2. **Matching**: Compare against database of copyrighted content
3. **Action**: Block, mute audio, or monetize (ads)
4. **Appeals**: Allow users to dispute false positives

**Implementation**:

- Use ML models for fingerprinting (e.g., neural networks)
- Store fingerprints in Elasticsearch for fast matching
- Run detection during transcoding pipeline
- Maintain database of copyrighted content from rights holders

**Used By**: YouTube Content ID, Facebook Rights Manager

### Why Not Use TCP for Video Streaming?

**Problem with TCP**:

- **Head-of-Line Blocking**: Lost packet blocks all subsequent packets
- **Retransmissions**: Wastes bandwidth on packets that arrive too late
- **Latency**: TCP handshake adds delay

**Why HTTP/HLS Works**:

- **Each segment is independent**: Lost segment doesn't block others
- **Client can skip segments**: If segment lost, request next one
- **Works over TCP but segments are independent**: HTTP provides reliability without head-of-line blocking

**For Ultra-Low Latency (Live)**:

- Use **WebRTC** (UDP-based, sub-second latency)
- Use **QUIC** (UDP-based, HTTP/3)
- Trade reliability for latency

### How to Handle Peak Traffic (Super Bowl, New Year's Eve)?

**Strategies**:

1. **Pre-Scale CDN**: Notify CDN provider of expected traffic spike
2. **Pre-Warm Cache**: Pre-populate edge locations with popular content
3. **Auto-Scaling**: Scale workers based on queue depth (proactive)
4. **Traffic Shaping**: Rate limit uploads during peak viewing hours
5. **Degraded Service**: Temporarily reduce max quality (4K → 1080p)

**Real-World Example**:

- Netflix pre-deploys extra capacity to ISPs before major releases
- YouTube increases CDN capacity during major events
- Twitch uses dynamic bitrate capping during peak hours

### How to Monetize Platform?

**Revenue Streams**:

1. **Ads**: Pre-roll, mid-roll, banner ads (YouTube model)
2. **Subscriptions**: Premium ad-free tier (YouTube Premium, Netflix)
3. **Pay-Per-View**: Rent or buy individual videos
4. **Channel Memberships**: Users pay creators directly
5. **Super Chat**: Paid messages in live streams

**Technical Requirements**:

- **Ad Server Integration**: VAST/VPAID protocols
- **Payment Processing**: Stripe, PayPal integration
- **Analytics**: Track ad impressions, conversions
- **Creator Dashboard**: Show earnings, analytics

---

## Advanced Topics

### Thumbnail Generation

**Challenge**: Generate thumbnails automatically for video preview.

**Solution**:

1. **During Transcoding**: Extract frames at regular intervals (e.g., every 10 seconds)
2. **ML-Based Selection**: Use ML model to select best thumbnail (clear faces, high contrast)
3. **Multiple Thumbnails**: Generate 3-5 options, let creator choose
4. **Custom Thumbnails**: Allow creators to upload custom thumbnail

**Implementation**:

- Use FFmpeg to extract frames: `ffmpeg -i video.mp4 -vf fps=1/10 thumb_%03d.jpg`
- Resize to standard sizes (120×90, 320×180, 480×360, 1280×720)
- Store in S3, serve via CDN

### Subtitle and Closed Caption Support

**Challenge**: Support multiple languages and accessibility.

**Solution**:

1. **Upload Subtitles**: Accept WebVTT or SRT files
2. **Auto-Generate**: Use speech-to-text (Google Cloud Speech, AWS Transcribe)
3. **Translate**: Use machine translation for multiple languages
4. **Synchronization**: Ensure subtitles match video timing

**Technical Details**:

- Store subtitles as separate files (not burned into video)
- Use WebVTT format (standard for HTML5 video)
- Serve subtitles from CDN (same as video)
- Client fetches subtitle track based on user language preference

### Video Analytics and Engagement Tracking

**Challenge**: Track what users watch, when they drop off, engagement metrics.

**Solution**:

1. **Client-Side Events**: Client sends events every 10 seconds
    - Video started, paused, resumed, completed
    - Current playback position
    - Quality selected (ABR decision)
    - Buffering events
2. **Server-Side Processing**: Aggregate events in real-time
    - Kafka for event streaming
    - Spark/Flink for real-time aggregation
    - Store in data warehouse (BigQuery, Redshift)
3. **Metrics Calculated**:
    - **View Count**: Count of unique views
    - **Watch Time**: Total minutes watched
    - **Audience Retention**: % of users who watch X% of video
    - **Drop-Off Points**: Where users stop watching
    - **Engagement Rate**: Likes, comments, shares per view

**Use Cases**:

- **Creator Dashboard**: Show analytics to creators
- **Recommendation Engine**: Use watch patterns for recommendations
- **A/B Testing**: Test thumbnail, title variations
- **Monetization**: Calculate ad revenue based on watch time

### Content Moderation

**Challenge**: Detect and remove inappropriate content (violence, hate speech, NSFW).

**Solution**:

1. **Automated Detection**:
    - **Computer Vision**: Detect nudity, violence in video frames
    - **Audio Analysis**: Detect hate speech, profanity in audio
    - **Text Analysis**: Scan title, description, tags for violations
2. **Human Review**:
    - Flagged content reviewed by human moderators
    - Community reporting (users can flag videos)
    - Appeals process for false positives
3. **Actions**:
    - **Remove**: Delete video immediately
    - **Age-Restrict**: Require user to be 18+
    - **Demonetize**: Disable ads on video
    - **Strike System**: 3 strikes = channel ban

**ML Models**:

- Use pre-trained models (Google Cloud Vision, AWS Rekognition)
- Train custom models on platform-specific data
- Continuously improve models based on human review

### DRM (Digital Rights Management)

**Challenge**: Protect premium content from piracy.

**Solution**:

1. **Encryption**: Encrypt video segments (AES-128)
2. **License Server**: Client requests decryption key from license server
3. **Playback Restrictions**: Limit number of devices, concurrent streams
4. **Watermarking**: Embed user ID in video (invisible, forensic watermark)

**DRM Systems**:

- **Widevine** (Google, Android)
- **FairPlay** (Apple, iOS)
- **PlayReady** (Microsoft, Windows)

**Implementation**:

- Encrypt video during transcoding
- Store encryption keys securely (AWS KMS)
- License server validates user subscription
- Client decrypts video on-the-fly during playback

---

## Performance Tuning

### Video Preloading and Prefetching

**Challenge**: Reduce startup latency from 2s to <1s.

**Solution**:

1. **Manifest Preloading**: Load manifest as soon as page loads (before user clicks play)
2. **First Segment Prefetch**: Download first segment immediately after manifest
3. **Adaptive Prefetch**: Prefetch next 2-3 segments based on bandwidth
4. **Thumbnail Hover Preload**: When user hovers over thumbnail, start loading first segment

**Trade-offs**:

- ✅ Faster startup (reduces abandonment rate)
- ❌ Wastes bandwidth if user doesn't watch (mitigated by only prefetching first segment)

### Optimizing Transcoding Pipeline

**Challenge**: Reduce transcoding time from 30 min to 10 min.

**Solutions**:

1. **Parallel Transcoding**: Transcode all qualities in parallel (not sequential)
    - 6 qualities × 5 min each = 30 min (sequential)
    - 6 qualities in parallel = 5 min (assuming 6 workers)
2. **Two-Pass Encoding**: Use two-pass encoding for better quality
    - First pass: Analyze video, generate statistics
    - Second pass: Encode with optimized settings
3. **Hardware Acceleration**: Use GPU for encoding (NVENC, QuickSync)
    - 5x faster than CPU encoding
    - Requires GPU-enabled instances (expensive)
4. **Chunked Transcoding**: Split video into chunks, transcode in parallel
    - 10-minute video → 10 × 1-minute chunks
    - Each chunk transcoded independently
    - Combine chunks at end

### CDN Performance Tuning

**Challenge**: Achieve 99.9% cache hit rate (currently 95%).

**Solutions**:

1. **Cache Warming**: Pre-populate cache with popular content
2. **Longer TTL**: Increase cache TTL from 24h to 7 days
3. **Cache Everything**: Cache manifest files, thumbnails, subtitles
4. **Origin Shield**: Add extra caching layer between edge and origin
5. **Predictive Caching**: Use ML to predict which videos will be popular

**Metrics**:

- **Cache Hit Ratio**: 99.9% (target)
- **Origin Requests**: <0.1% of total requests
- **Bandwidth Savings**: 99.9% of traffic served from cache

---

## References

- **Related Chapters**:
    - [2.0.4 HTTP and REST APIs](../../02-components/2.0-communication/2.0.4-http-rest-deep-dive.md)
    - [2.1.4 Cassandra Deep Dive](../../02-components/2.1-databases/2.1.9-cassandra-deep-dive.md)
    - [2.2.1 Caching Strategies](../../02-components/2.2-caching/2.2.1-caching-strategies.md)
    - [2.3.1 Message Queues](../../02-components/2.3-messaging-streaming/2.3.1-message-queues.md)
    - [2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)
    - [3.2.1 Twitter Timeline](../3.2.1-twitter-timeline/3.2.1-design-twitter-timeline.md)
    - [3.4.7 Online Code Judge](../3.4.7-online-code-judge/3.4.7-design-online-code-judge.md)

- **External Resources**:
    - **Netflix Tech Blog**: https://netflixtechblog.com/
    - **YouTube Architecture**: https://www.youtube.com/intl/en-GB/howyoutubeworks/
    - **FFmpeg Documentation**: https://ffmpeg.org/documentation.html
    - **HLS Specification**: https://datatracker.ietf.org/doc/html/rfc8216
    - **DASH Specification**: https://dashif.org/
    - **AWS Elemental MediaConvert**: https://aws.amazon.com/mediaconvert/
    - **Cloudflare Stream**: https://www.cloudflare.com/products/cloudflare-stream/

---

**End of README**

*This document provides a comprehensive overview of designing a video streaming system like YouTube or Netflix, covering
upload,
transcoding, delivery, optimization strategies, and advanced topics. For detailed implementations, see the supplementary
files.*

