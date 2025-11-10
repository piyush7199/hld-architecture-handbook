# Video Streaming System - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A video streaming platform like YouTube or Netflix that allows users to upload, process, store, and stream video content
globally with high availability and low latency. The system handles massive scale (petabytes of data, 100M+ concurrent
viewers), supports adaptive bitrate streaming, and minimizes bandwidth costs while providing a seamless viewing
experience.

**Key Challenges:**

- **Massive Storage:** Store petabytes of video content in multiple formats
- **High Bandwidth:** Serve 90% of internet traffic during peak hours
- **Complex Processing:** Transcode videos into multiple formats (144p to 4K)
- **Global Distribution:** Deliver content with <2s startup latency worldwide
- **Cost Optimization:** Bandwidth and storage costs dominate operational expenses
- **Adaptive Streaming:** Switch quality dynamically based on network conditions

**Key Characteristics:**

- **High throughput (read):** Serve petabytes of data daily (~90% of all internet traffic)
- **Low latency (startup):** Video playback must start in under 2 seconds
- **Fault tolerance:** Transcoding must be resilient; single server failure must not lose entire job
- **Cost efficiency:** Bandwidth and storage are dominant costs and must be aggressively minimized
- **High availability:** 99.99% uptime for streaming service
- **Scalability:** Handle traffic spikes during major events

---

## Requirements & Scale

### Functional Requirements

1. **Video Upload:** Accept large video files (up to 50 GB) and process them
2. **Video Transcoding:** Convert uploaded video into multiple formats and bitrates (144p to 4K)
3. **Streaming (VOD):** Deliver video efficiently with minimal buffering and high availability
4. **Playback Control:** Support seeking, pausing, resuming, and speed control
5. **Search and Discovery:** Find videos by title, tags, description
6. **Recommendations:** Suggest relevant videos based on watch history
7. **User Engagement:** Likes, comments, subscriptions, playlists

### Non-Functional Requirements

1. **High Throughput (Read):** Must serve petabytes of data daily (~90% of all internet traffic)
2. **Low Latency (Startup):** Video playback must start in under 2 seconds
3. **Fault Tolerance:** Transcoding must be resilient; single server failure must not lose entire job
4. **Cost Efficiency:** Bandwidth and storage are dominant costs and must be aggressively minimized
5. **High Availability:** 99.99% uptime for streaming service
6. **Scalability:** Handle traffic spikes during major events

### Scale Estimation

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

## Key Components

| Component                    | Responsibility                                                             | Technology          | Scalability               |
|------------------------------|----------------------------------------------------------------------------|---------------------|---------------------------|
| **Upload Service**           | Handle resumable uploads via chunked HTTP or gRPC streaming                | Go, Java (gRPC)     | Horizontal                |
| **Raw Storage**              | Store original, unprocessed video files                                    | S3                  | Distributed               |
| **Transcoding Queue**        | Buffer massive list of transcoding jobs, ensure durability                 | Apache Kafka        | Horizontal (partitions)   |
| **Transcoding Worker Fleet** | CPU-heavy, auto-scaling cluster that converts video into different formats | FFmpeg, Kubernetes  | Horizontal (10k+ workers) |
| **Processed Storage**        | Stores transcoded video segments (HLS/DASH)                                | S3                  | Distributed               |
| **CDN**                      | Caches and serves finished video segments to viewers globally              | CloudFront, Akamai  | Global edge network       |
| **Metadata Database**        | Stores video titles, descriptions, CDN URLs for all formats                | Cassandra, DynamoDB | Horizontal (sharded)      |
| **Search Service**           | Full-text search for video discovery                                       | Elasticsearch       | Horizontal (sharded)      |
| **Recommendation Engine**    | ML-based system for personalized video suggestions                         | ML models, Redis    | Horizontal                |
| **Analytics Pipeline**       | Track views, watch time, engagement metrics                                | Kafka, Spark        | Horizontal                |

---

## Upload Service and Resumable Uploads

**Challenge:** Videos can be 50 GB+. Network interruptions are common. Users should not have to restart uploads.

**Solution: Resumable Uploads (Chunked HTTP or gRPC Streaming)**

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

---

## Transcoding Pipeline

**Challenge:** Convert raw video into multiple formats (144p, 360p, 720p, 1080p, 4K) with different codecs (H.264,
H.265, VP9).

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

- **Auto-Scaling:** Scale based on Kafka queue depth
- **Spot Instances:** Use AWS Spot for 70% cost savings (non-urgent transcode)
- **Dedicated Instances:** Use for live streams or high-priority content
- **FFmpeg:** Open-source tool for video processing

---

## Adaptive Bitrate Streaming (ABR)

**Challenge:** Users have varying network conditions. Must switch quality dynamically without interrupting playback.

**Solution: HLS (HTTP Live Streaming) or DASH (Dynamic Adaptive Streaming over HTTP)**

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

- **No Buffering:** Client downloads next segment before current ends
- **Quality Switching:** Seamless transition between qualities
- **HTTP-Based:** Works through firewalls, uses standard CDN

---

## CDN Strategy and Edge Caching

**Challenge:** Bandwidth costs dominate (90% of expenses). Must minimize origin requests.

**Solution: Multi-Tier CDN with Aggressive Caching**

**CDN Tiers:**

```
User → Edge (99% hit) → Regional (95% hit) → Origin (S3)
```

1. **Edge Locations** (1000+): Close to users, cache hot content (popular videos)
2. **Regional Caches** (50+): Aggregate traffic from multiple edges
3. **Origin (S3):** Stores all content, serves cache misses

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

- **Prefetching:** CDN prefetches next segment before client requests
- **Range Requests:** Support HTTP Range headers for seeking
- **Compression:** Use Gzip/Brotli for manifest files
- **Geo-Routing:** Route users to nearest edge location

---

## Bottlenecks & Solutions

### Bottleneck 1: Transcoding Optimization

**Challenge:** 500k videos/day × 30 min processing = 10.4M CPU-hours/day

**Solutions:**

1. **Parallel Processing:** Transcode all qualities in parallel (not sequential)
2. **Spot Instances:** Use AWS Spot for 70% cost savings
3. **Prioritization:** High-priority videos (verified channels) transcoded first
4. **Incremental Encoding:** Reuse keyframes across qualities
5. **Hardware Acceleration:** Use GPU for H.265 encoding (5× faster)

### Bottleneck 2: CDN Optimization

**Challenge:** 300 PB/day bandwidth = $3M/day cost at $0.01/GB

**Solutions:**

1. **Aggressive Caching:** 99% cache hit rate at edge
2. **CDN Negotiation:** Volume discounts with CloudFront, Akamai
3. **Compression:** Use H.265 (50% smaller than H.264)
4. **P2P Delivery:** Experimental peer-to-peer for popular videos

### Bottleneck 3: Storage Optimization

**Challenge:** 500 PB storage = $10M/month at $0.02/GB

**Solutions:**

1. **Cold Storage:** Move old videos to Glacier ($0.004/GB)
2. **Deduplication:** Detect and remove duplicate uploads
3. **Compression:** Use efficient codecs (H.265, AV1)
4. **Retention Policy:** Delete videos with <10 views after 1 year

### Bottleneck 4: Database Optimization

**Challenge:** 50k QPS for metadata reads

**Solutions:**

1. **Redis Caching:** 80% cache hit rate (10ms → 1ms)
2. **Cassandra Partitioning:** Shard by video_id
3. **Read Replicas:** 5 replicas per region for high availability
4. **Denormalization:** Store CDN URLs directly in metadata (no JOIN)

---

## Common Anti-Patterns to Avoid

### 1. Storing Videos in Database

❌ **Bad:** Videos are large blobs, databases are not optimized for blob storage

**Why It's Bad:**

- Database IOPS exhausted (100s of MB/sec)
- Expensive (PostgreSQL storage $0.10/GB vs S3 $0.02/GB)
- Cannot leverage CDN (database not HTTP-accessible)

✅ **Good:** Store videos in S3, store metadata in database

### 2. Synchronous Transcoding

❌ **Bad:** User waits 30 minutes for video to process before upload completes

**Why It's Bad:**

- Terrible user experience (timeout)
- API Gateway connection times out (60 sec max)
- Cannot scale (one request = one worker)

✅ **Good:** Asynchronous transcoding with Kafka queue

### 3. Single Quality Encoding

❌ **Bad:** Only encode video at 1080p, let client scale down

**Why It's Bad:**

- Wastes bandwidth (mobile users download 1080p, display 360p)
- Poor UX on slow networks (constant buffering)
- Higher costs ($8 Mbps for 1080p vs $0.8 Mbps for 360p)

✅ **Good:** Adaptive bitrate streaming with multiple qualities

### 4. No CDN (Direct Streaming from Origin)

❌ **Bad:** Stream all videos directly from S3

**Why It's Bad:**

- High latency (S3 in one region, users worldwide)
- Bandwidth costs 10× higher ($0.09/GB S3 vs $0.01/GB CDN)
- S3 request limits (5,500 GET/sec per prefix)

✅ **Good:** Always use CDN for video delivery

### 5. Monolithic Transcode Service

❌ **Bad:** Single service handles upload, transcode, and streaming

**Why It's Bad:**

- Cannot scale independently (upload ≠ transcode ≠ streaming load)
- Deployment risk (update transcode logic, break streaming)
- Resource contention (CPU-heavy transcode starves API)

✅ **Good:** Microservices: Upload Service, Transcode Workers, Streaming API

---

## Monitoring & Observability

### Key Metrics

**Upload Service:**

- Upload success rate (target: 99.9%)
- Average upload time
- Chunk failure rate

**Transcoding:**

- Queue depth (Kafka lag)
- Processing time per video
- Worker utilization (CPU, memory)
- Transcode success rate (target: 99.5%)

**Streaming:**

- Startup latency (p50, p95, p99)
- Buffering rate (target: <1% of playback time)
- CDN cache hit rate (target: 99%)
- Bandwidth usage (TB/day)

**Metadata DB:**

- Read/write QPS
- Latency (p50, p95, p99)
- Cache hit rate (target: 80%)

### Alerts

**Critical Alerts:**

- Transcode queue depth >100k (scale up workers)
- CDN cache hit rate <90% (investigate)
- Video startup latency >5s
- Upload success rate <95%

**Warning Alerts:**

- Transcoding success rate <99%
- Worker CPU >90%
- Metadata DB latency >100ms

---

## Trade-offs Summary

| What We Gain                                | What We Sacrifice                                        |
|---------------------------------------------|----------------------------------------------------------|
| ✅ **Low latency** (<2s startup)             | ❌ **High CDN cost** ($3M/day bandwidth)                  |
| ✅ **Adaptive streaming** (seamless quality) | ❌ **Storage cost** (500 PB = $10M/month)                 |
| ✅ **Global distribution** (CDN)             | ❌ **Transcoding complexity** (multiple formats)          |
| ✅ **Fault tolerance** (queue-based)         | ❌ **Processing delay** (30 min transcode)                |
| ✅ **Scalability** (horizontal scaling)      | ❌ **Operational complexity** (10k+ workers)              |
| ✅ **Cost optimization** (tiered storage)    | ❌ **Cold storage latency** (Glacier retrieval 3-5 hours) |

---

## Real-World Examples

### Netflix

- **CDN:** Custom CDN (Open Connect) deployed in ISPs
- **Encoding:** Per-title encoding (optimized bitrate per video)
- **Storage:** AWS S3 + proprietary systems
- **Scale:** 8 trillion hours watched annually

### YouTube

- **Upload:** Resumable uploads (YouTube API)
- **Transcoding:** Massive fleet of custom transcoders
- **Delivery:** Google's global CDN
- **Scale:** 500 hours of video uploaded every minute

### Twitch

- **Focus:** Live streaming (not VOD)
- **Latency:** <3 seconds (ultra-low latency)
- **Transcoding:** Real-time transcoding for live streams
- **Scale:** 140 million monthly active users

---

## Key Takeaways

1. **CDN is mandatory** (not optional) - 99% cache hit rate reduces bandwidth costs by 10×
2. **Adaptive bitrate streaming** (HLS/DASH) enables seamless quality switching
3. **Asynchronous transcoding** (Kafka queue) decouples upload from processing
4. **Resumable uploads** (chunked HTTP) handle large files and network interruptions
5. **Tiered storage** (hot/warm/cold) reduces costs (S3 → Glacier)
6. **Parallel transcoding** (all qualities at once) reduces processing time
7. **Hardware acceleration** (GPU) speeds up H.265 encoding (5× faster)
8. **Metadata caching** (Redis) reduces database load (80% hit rate)
9. **Auto-scaling workers** (queue-driven) handle traffic spikes
10. **Deduplication** prevents duplicate uploads (save storage)

---

## Recommended Stack

- **Upload Service:** gRPC (Go/Java), S3 Multipart Upload
- **Transcoding Queue:** Apache Kafka (partitioned by video_id)
- **Transcoding Workers:** FFmpeg, Kubernetes (auto-scaling), GPU instances
- **Processed Storage:** S3 (HLS/DASH segments)
- **CDN:** CloudFront/Akamai (global edge network)
- **Metadata DB:** Cassandra/DynamoDB (sharded by video_id)
- **Search:** Elasticsearch (full-text search)
- **Caching:** Redis (metadata cache, 80% hit rate)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, upload flow, transcoding pipeline, streaming
  flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (upload, transcoding, streaming)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (upload, transcoding, ABR)