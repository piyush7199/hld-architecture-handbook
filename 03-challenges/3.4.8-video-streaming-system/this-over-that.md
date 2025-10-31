# Video Streaming System - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made when designing a video streaming
platform like YouTube or Netflix.

---

## Table of Contents

1. [HLS vs DASH for Adaptive Streaming](#1-hls-vs-dash-for-adaptive-streaming)
2. [Kafka vs SQS for Transcoding Queue](#2-kafka-vs-sqs-for-transcoding-queue)
3. [Cassandra vs DynamoDB for Metadata](#3-cassandra-vs-dynamodb-for-metadata)
4. [CDN vs Direct S3 Streaming](#4-cdn-vs-direct-s3-streaming)
5. [Resumable vs Single-Request Upload](#5-resumable-vs-single-request-upload)
6. [Synchronous vs Asynchronous Transcoding](#6-synchronous-vs-asynchronous-transcoding)
7. [Multi-Image vs Single-Image Transcoding](#7-multi-image-vs-single-image-transcoding)
8. [S3 vs Database for Video Storage](#8-s3-vs-database-for-video-storage)
9. [WebSocket vs Polling for Real-Time Updates](#9-websocket-vs-polling-for-real-time-updates)
10. [Multi-Region vs Single-Region Deployment](#10-multi-region-vs-single-region-deployment)

---

## 1. HLS vs DASH for Adaptive Streaming

### The Problem

Users have varying network conditions (fiber, 4G, 3G, 2G). Video must adapt quality dynamically to prevent buffering. We
need a streaming protocol that supports adaptive bitrate (ABR).

### Options Considered

| Feature               | HLS (HTTP Live Streaming)      | DASH (Dynamic Adaptive Streaming) |
|-----------------------|--------------------------------|-----------------------------------|
| **Created By**        | Apple (proprietary)            | MPEG (open standard)              |
| **Browser Support**   | ✅ Safari native, others via JS | ⚠️ No Safari native support       |
| **Mobile Support**    | ✅ iOS/Android native           | ⚠️ Android native, iOS needs JS   |
| **Segment Format**    | MPEG-TS (.ts files)            | MP4 fragments (.m4s files)        |
| **Manifest Format**   | .m3u8 (text)                   | .mpd (XML)                        |
| **Codec Flexibility** | ⚠️ Limited (H.264, H.265)      | ✅ Flexible (any codec)            |
| **Latency**           | ⚠️ 15-30 seconds               | ✅ 5-10 seconds (LL-DASH)          |
| **Complexity**        | ✅ Simple                       | ⚠️ Complex                        |
| **Adoption**          | ✅ 70% of platforms             | ⚠️ 30% of platforms               |

### Decision Made

**HLS (HTTP Live Streaming) as Primary, DASH as Optional**

### Rationale

1. **Browser Compatibility**: Safari (20% of users) only supports HLS natively
2. **Mobile Native Support**: iOS requires HLS for App Store approval
3. **Simpler Implementation**: .m3u8 manifests easier to parse than XML
4. **Wider Ecosystem**: More players, tools, CDN support for HLS
5. **Good Enough Latency**: 15-30 seconds acceptable for VOD (not live)

**Why NOT DASH Only**:

- No Safari support (20% of users would need JavaScript player)
- iOS apps rejected from App Store without HLS support
- More complex to implement (XML parsing, multiple profiles)
- Smaller ecosystem (fewer battle-tested players)

**Hybrid Approach**:

- Serve HLS to 80% of users (Safari, iOS, simple Android)
- Serve DASH to 20% (Android Chrome for lower latency)

### Implementation Details

**HLS Master Manifest**:

```
#EXTM3U
#EXT-X-VERSION:6
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360,CODECS="avc1.42e00a,mp4a.40.2"
360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=854x480
480p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1280x720
720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=8000000,RESOLUTION=1920x1080
1080p/playlist.m3u8
```

**Segment Naming**:

```
video_123/
  360p/
    segment_000.ts (10 seconds)
    segment_001.ts
    segment_002.ts
  720p/
    segment_000.ts
    segment_001.ts
  master.m3u8
```

### Trade-offs Accepted

| What We Gain                           | What We Sacrifice                                 |
|----------------------------------------|---------------------------------------------------|
| ✅ Universal compatibility (100% users) | ❌ Higher latency (15-30s vs 5-10s DASH)           |
| ✅ Simple implementation                | ❌ Less codec flexibility (stuck with H.264/H.265) |
| ✅ iOS App Store approval               | ❌ Larger segment files (MPEG-TS overhead)         |
| ✅ Mature ecosystem                     | ❌ Proprietary (Apple controls spec)               |

### When to Reconsider

**Switch to DASH if**:

- Need ultra-low latency (< 5 seconds)
- Want codec flexibility (AV1, VP9)
- Don't care about iOS/Safari users
- Live streaming (DASH better for live)

**Switch to WebRTC if**:

- Need sub-second latency (live video calls)
- Willing to sacrifice scalability
- Have budget for WebRTC infrastructure

### Real-World Examples

- **Netflix**: Uses DASH (Android) + HLS (iOS) hybrid
- **YouTube**: Uses DASH for all platforms (custom player)
- **Twitch**: Uses HLS for VOD, custom for live
- **Disney+**: HLS primary, DASH secondary

---

## 2. Kafka vs SQS for Transcoding Queue

### The Problem

Transcoding is slow (5-30 minutes per video). Need a queue to buffer 500k daily uploads and ensure no job is lost. Queue
must handle traffic spikes (10x during viral events).

### Options Considered

| Feature           | Kafka                     | AWS SQS                            | RabbitMQ                  |
|-------------------|---------------------------|------------------------------------|---------------------------|
| **Throughput**    | ✅ 1M+ msg/sec             | ⚠️ 3k msg/sec per queue            | ⚠️ 50k msg/sec            |
| **Ordering**      | ✅ Per-partition FIFO      | ❌ Best-effort (FIFO queue limited) | ⚠️ Per-queue FIFO         |
| **Retention**     | ✅ Days/weeks (replay)     | ⚠️ 14 days max                     | ⚠️ Memory-based           |
| **Durability**    | ✅ Disk-persisted          | ✅ Replicated                       | ⚠️ Optional               |
| **Scalability**   | ✅ Horizontal (partitions) | ✅ Fully managed                    | ⚠️ Vertical (single node) |
| **Complexity**    | ❌ High (ZooKeeper, ops)   | ✅ Zero (managed)                   | ⚠️ Moderate               |
| **Cost**          | ⚠️ $0.10/GB + EC2         | ✅ $0.40/million msgs               | ⚠️ Self-hosted            |
| **Replayability** | ✅ Yes                     | ❌ No                               | ❌ No                      |

### Decision Made

**Kafka for Primary Queue, SQS for Dead Letter Queue (DLQ)**

### Rationale

1. **High Throughput**: 500k uploads/day = 5.8 msg/sec (Kafka easily handles this)
2. **Ordering Guarantee**: Same video re-uploads shouldn't race (partition by video_id)
3. **Replay Capability**: Debug failed transcodes by replaying from offset
4. **Retention**: Keep jobs for 7 days (allows late processing for non-critical videos)
5. **Data Locality**: Partition by video_id → same video always goes to same worker (cache locality)

**Why NOT SQS Only**:

- FIFO queues limited to 300 msg/sec (need multiple queues)
- No replay (can't reprocess failed jobs from history)
- No ordering across queues (can't partition by video_id)
- More expensive at scale ($0.40/million vs $0.10/GB Kafka)

**Hybrid Approach**:

- **Kafka**: Primary queue for fresh uploads
- **SQS DLQ**: Dead letter queue for repeatedly failing jobs (manual review)

### Implementation Details

**Kafka Topic Structure**:

```
transcode-jobs (50 partitions)
├── partition 0: video_id % 50 == 0
├── partition 1: video_id % 50 == 1
├── ...
└── partition 49: video_id % 50 == 49

transcode-priority (10 partitions)
└── High-priority videos (verified channels, paid)
```

**Producer Code**:

```
// Publish transcode job
def publish_transcode_job(video_id, s3_key, priority="normal"):
    topic = "transcode-priority" if priority == "high" else "transcode-jobs"
    partition = hash(video_id) % (10 if priority == "high" else 50)
    
    kafka.produce(
        topic=topic,
        partition=partition,
        key=video_id,
        value={
            "video_id": video_id,
            "s3_key": s3_key,
            "timestamp": now(),
            "retry_count": 0
        }
    )
```

**Consumer with Retry Logic**:

```
def consume_transcode_jobs():
    consumer = kafka.consumer(group_id="transcode-workers")
    consumer.subscribe(["transcode-jobs", "transcode-priority"])
    
    for message in consumer:
        job = json.loads(message.value)
        
        try:
            transcode_video(job)
            consumer.commit()  # Success
        except Exception as e:
            job["retry_count"] += 1
            
            if job["retry_count"] > 3:
                # Move to DLQ for manual review
                sqs.send_message(queue="transcode-dlq", body=job)
            else:
                # Retry: publish back to Kafka with delay
                kafka.produce(topic=message.topic, value=job)
            
            consumer.commit()  # Acknowledge original message
```

### Trade-offs Accepted

| What We Gain                    | What We Sacrifice                             |
|---------------------------------|-----------------------------------------------|
| ✅ High throughput (1M+ msg/sec) | ❌ Operational complexity (ZooKeeper, brokers) |
| ✅ Ordering per video            | ❌ More expensive at low scale                 |
| ✅ Replay capability (debugging) | ❌ Requires Kafka expertise                    |
| ✅ Data locality (caching)       | ❌ Rebalancing delay (30s when scaling)        |

### When to Reconsider

**Switch to SQS if**:

- Low scale (<10k uploads/day)
- Don't need ordering
- Want zero operational overhead
- Team lacks Kafka expertise

**Switch to RabbitMQ if**:

- Need complex routing (exchange types)
- Already using RabbitMQ
- Scale < 50k msg/sec

### Real-World Examples

- **Netflix**: Kafka for encoding pipeline
- **YouTube**: Custom queue (Spanner-based)
- **Vimeo**: RabbitMQ for smaller scale
- **TikTok**: Kafka for massive scale

---

## 3. Cassandra vs DynamoDB for Metadata

### The Problem

Store metadata for 1 billion videos: title, description, views, CDN URLs. Need low-latency reads (< 10ms), high write
throughput (10k QPS), and multi-region replication.

### Options Considered

| Feature               | Cassandra                   | DynamoDB                 | PostgreSQL                      |
|-----------------------|-----------------------------|--------------------------|---------------------------------|
| **Read Latency**      | ✅ 1-5ms                     | ✅ 1-5ms                  | ⚠️ 10-50ms                      |
| **Write Throughput**  | ✅ 100k+ writes/sec          | ✅ 100k+ WCU              | ⚠️ 10k writes/sec               |
| **Scalability**       | ✅ Linear (add nodes)        | ✅ Auto-scaling           | ⚠️ Vertical (read replicas)     |
| **Multi-Region**      | ✅ Built-in (multi-DC)       | ✅ Global tables          | ⚠️ Manual replication           |
| **Query Flexibility** | ❌ Limited (partition key)   | ❌ Limited (PK + SK)      | ✅ Full SQL (JOIN, GROUP BY)     |
| **Cost**              | ⚠️ $0.10/GB + EC2           | ⚠️ $0.25/GB + throughput | ✅ $0.05/GB + compute            |
| **Operational**       | ⚠️ Complex (manage cluster) | ✅ Fully managed          | ⚠️ Moderate (backups, replicas) |

### Decision Made

**Cassandra for Metadata, PostgreSQL for Analytics, Redis for Cache**

### Rationale

1. **Horizontal Scalability**: Partition 1B videos across 100 nodes (10M per node)
2. **Low Latency**: Single-digit millisecond reads (no disk seeks, memory + SSD)
3. **Multi-Region**: Built-in multi-datacenter replication (eventual consistency OK)
4. **No Single Point of Failure**: Peer-to-peer architecture (no master)
5. **Write Throughput**: Handle 10k video uploads/sec + 100k view count updates/sec

**Why NOT DynamoDB**:

- More expensive at scale ($0.25/GB vs $0.10/GB)
- Throughput cost high (pay per RCU/WCU)
- Less control (can't tune compaction, caching)
- Vendor lock-in (AWS only)

**Why NOT PostgreSQL Only**:

- Vertical scaling limits (read replicas add complexity)
- Slower for key-value lookups (10-50ms vs 1-5ms)
- Multi-region replication complex (manual setup)
- Not designed for 1B rows single table

**Hybrid Approach**:

- **Cassandra**: Hot metadata (video details, view counts) - 90% of queries
- **PostgreSQL**: Analytics (complex queries, reports) - 10% of queries
- **Redis**: Cache layer (80% hit rate) - sub-millisecond

### Implementation Details

**Cassandra Schema**:

```sql
CREATE TABLE videos (
    video_id TEXT PRIMARY KEY,
    title TEXT,
    description TEXT,
    channel_id TEXT,
    upload_date TIMESTAMP,
    duration INT,
    views COUNTER,
    likes COUNTER,
    cdn_urls MAP<TEXT, TEXT>,  -- {720p: 'cdn.../720p/', 1080p: '...'}
    thumbnail_url TEXT,
    tags LIST<TEXT>,
    status TEXT,  -- 'processing', 'ready', 'failed'
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX ON videos (channel_id);  -- Secondary index for channel queries
CREATE INDEX ON videos (status);      -- Query by status
```

**Partition Strategy**:

```
Partition by video_id (ensures single-partition queries)
Replication factor: 3 (across 3 datacenters)
Consistency level: LOCAL_QUORUM (2/3 replicas in local DC)
```

**Cache-Aside Pattern**:

```python
def get_video_metadata(video_id):
    # 1. Check Redis cache (1ms)
    cache_key = f"video:{video_id}"
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # 2. Query Cassandra (5ms)
    row = cassandra.execute(
        "SELECT * FROM videos WHERE video_id = ?",
        [video_id]
    ).one()
    
    # 3. Cache for 1 hour
    redis.setex(cache_key, 3600, json.dumps(row))
    
    return row
```

### Trade-offs Accepted

| What We Gain                     | What We Sacrifice                         |
|----------------------------------|-------------------------------------------|
| ✅ Low latency (1-5ms)            | ❌ Limited query flexibility (no JOIN)     |
| ✅ Linear scalability (add nodes) | ❌ Operational complexity (manage cluster) |
| ✅ Multi-region built-in          | ❌ Eventual consistency (not ACID)         |
| ✅ No SPOF (peer-to-peer)         | ❌ Requires Cassandra expertise            |

### When to Reconsider

**Switch to DynamoDB if**:

- Want fully managed (zero ops)
- Small scale (<100M videos)
- On AWS (want native integration)
- Budget allows (higher cost)

**Switch to PostgreSQL if**:

- Need complex queries (JOIN, analytics)
- Small scale (<10M videos)
- ACID transactions required
- Team knows SQL well

### Real-World Examples

- **Netflix**: Cassandra for metadata (billions of rows)
- **YouTube**: BigTable (Google's Cassandra equivalent)
- **Disney+**: DynamoDB (AWS native)
- **Hulu**: Cassandra + PostgreSQL hybrid

---

## 4. CDN vs Direct S3 Streaming

### The Problem

Serve 300 PB/day of video data to 100M concurrent viewers. Direct S3 streaming costs $0.09/GB. Need to minimize
bandwidth costs and latency.

### Options Considered

| Approach             | Latency (User in London) | Bandwidth Cost             | Throughput                  | Complexity     |
|----------------------|--------------------------|----------------------------|-----------------------------|----------------|
| **Direct S3**        | ❌ 150ms (US-East)        | ❌ $0.09/GB                 | ⚠️ 5,500 req/sec per prefix | ✅ Simple       |
| **CDN (CloudFront)** | ✅ 20ms (London edge)     | ✅ $0.01/GB (99% cache hit) | ✅ Unlimited                 | ⚠️ Moderate    |
| **Custom CDN**       | ✅ 15ms (ISP peering)     | ✅ $0.005/GB (negotiated)   | ✅ Unlimited                 | ❌ Complex      |
| **P2P (BitTorrent)** | ✅ 10ms (peer nearby)     | ✅ $0/GB (free)             | ✅ Scales with users         | ❌ Very complex |

### Decision Made

**CloudFront CDN (Primary), S3 as Origin, Custom CDN for Top 1%**

### Rationale

1. **Cost Savings**: $27M/month (S3) → $3.24M/month (CDN) = $23.76M saved (88% reduction)
2. **Latency Improvement**: 150ms → 20ms (7.5x faster)
3. **Scalability**: S3 limited to 5,500 req/sec per prefix, CDN unlimited
4. **Cache Hit Rate**: 99% at edge (only 1% goes to origin)
5. **Global Distribution**: 1000+ edge locations worldwide

**Cost Calculation**:

```
Without CDN:
- 300 PB/month × $0.09/GB = $27M/month

With CDN (99% cache hit):
- Origin (1%): 3 PB/month × $0.09/GB = $270k
- CDN (99%): 297 PB/month × $0.01/GB = $2.97M
- Total: $3.24M/month (88% savings)
```

**Why NOT Direct S3**:

- Too expensive ($27M/month vs $3.24M/month)
- High latency (150ms vs 20ms)
- Request limits (5,500 req/sec)
- No cache (every request hits origin)

**Why NOT P2P**:

- Complex to implement (WebRTC, NAT traversal)
- Privacy concerns (users see each other's IPs)
- Unreliable (peers go offline)
- Not supported on mobile (iOS blocks)

**Hybrid Approach**:

- **CloudFront**: 99% of traffic (standard CDN)
- **Custom CDN (Open Connect)**: Top 1% of traffic (Netflix model - deploy servers in ISPs)

### Implementation Details

**CDN Configuration**:

```yaml
CloudFront Distribution:
  Origins:
    - Domain: processed-videos.s3.us-east-1.amazonaws.com
      OriginShield:
        Enabled: true
        Region: us-east-1

  CacheBehaviors:
    - PathPattern: "*.ts"  # Video segments
      TTL:
        Min: 86400      # 1 day
        Default: 604800  # 7 days
        Max: 31536000   # 1 year

    - PathPattern: "*.m3u8"  # Manifests
      TTL:
        Min: 60         # 1 minute
        Default: 3600   # 1 hour
        Max: 86400      # 1 day

  Compression: Enabled  # Gzip manifests
  PriceClass: PriceClass_All  # Use all edge locations

  Logging:
    Bucket: cdn-logs.s3.amazonaws.com
    Prefix: cloudfront/
```

**Cache Warming**:

```python
def warm_cdn_cache(video_id):
    """Pre-populate CDN with popular video segments"""
    qualities = ["360p", "720p", "1080p"]
    
    for quality in qualities:
        # Fetch first 10 segments (first 100 seconds)
        for i in range(10):
            segment_url = f"https://cdn.example.com/{video_id}/{quality}/segment_{i:03d}.ts"
            
            # Request from multiple edge locations
            for edge in ["us-east-1", "eu-west-1", "ap-southeast-1"]:
                requests.get(segment_url, headers={"X-Edge": edge})
```

### Trade-offs Accepted

| What We Gain                        | What We Sacrifice                               |
|-------------------------------------|-------------------------------------------------|
| ✅ 88% cost savings ($23.76M/month)  | ❌ CDN contract negotiations (volume discounts)  |
| ✅ 7.5x lower latency (150ms → 20ms) | ❌ Cache invalidation complexity (stale content) |
| ✅ Unlimited throughput              | ❌ Additional moving part (CDN failures)         |
| ✅ Global distribution               | ❌ Vendor dependency (CloudFront)                |

### When to Reconsider

**Switch to Direct S3 if**:

- Very small scale (<1k concurrent viewers)
- Low budget (can't afford CDN minimum)
- Users in single region (no need for global distribution)

**Switch to Custom CDN if**:

- Netflix scale (petabytes daily)
- Can negotiate ISP partnerships
- Want ultimate control (caching logic)
- Have $10M+ infrastructure budget

### Real-World Examples

- **Netflix**: Custom CDN (Open Connect appliances in ISPs)
- **YouTube**: Google's global CDN (billions of users)
- **Amazon Prime Video**: CloudFront (own CDN)
- **Twitch**: CloudFront + custom edge (hybrid)

---

## 5. Resumable vs Single-Request Upload

### The Problem

Users upload videos up to 50 GB. Network interruptions are common (mobile users, flaky WiFi). Should uploads be
resumable or force restart from beginning?

### Options Considered

| Approach                | User Experience              | Reliability               | Complexity  | Bandwidth Waste     |
|-------------------------|------------------------------|---------------------------|-------------|---------------------|
| **Single Request**      | ❌ Terrible (restart on fail) | ❌ Poor (50% success rate) | ✅ Simple    | ❌ High (50% wasted) |
| **Resumable (Chunked)** | ✅ Excellent (resume on fail) | ✅ Excellent (99% success) | ⚠️ Moderate | ✅ Low (<1% wasted)  |
| **Streaming (gRPC)**    | ✅ Good (resume chunks)       | ✅ Good (95% success)      | ⚠️ Moderate | ✅ Low               |

### Decision Made

**Resumable Upload (TUS Protocol) with S3 Multipart API**

### Rationale

1. **User Experience**: Don't make users re-upload 5 GB if they fail at 4.9 GB
2. **Reliability**: Mobile users lose connection frequently (99% success vs 50%)
3. **Bandwidth Savings**: Only re-upload failed chunks (saves 50% bandwidth)
4. **Industry Standard**: TUS protocol (used by Vimeo, Dropbox)
5. **S3 Native Support**: S3 Multipart Upload API (up to 10,000 parts, 5 GB each)

**Why NOT Single Request**:

- Poor user experience (restart from 0% on any error)
- Low success rate (50% for 5 GB uploads)
- Wastes bandwidth (re-upload entire file)
- Timeouts (HTTP timeout after 60 seconds)

**Why NOT gRPC Streaming Only**:

- More complex server logic (maintain stream state)
- Less battle-tested (TUS more mature)
- Requires HTTP/2 (not universally supported)

### Implementation Details

**Upload Flow**:

```
1. Client splits file into 10 MB chunks (500 chunks for 5 GB)
2. Client POST /upload/init → Server returns upload_id
3. For each chunk:
   - Client POST /upload/chunk (upload_id, chunk_num, data)
   - Server uploads to S3 Multipart API
   - Server marks chunk complete in Redis
   - Server returns acknowledgment
4. Client POST /upload/complete → Server finalizes multipart upload
5. Server triggers transcode job in Kafka
```

**State Management (Redis)**:

```python
# Track upload state
def init_upload(filename, size):
    upload_id = str(uuid4())
    
    redis.hset(f"upload:{upload_id}", mapping={
        "filename": filename,
        "size": size,
        "chunks_total": math.ceil(size / CHUNK_SIZE),
        "chunks_completed": "[]",  # JSON list
        "s3_upload_id": None,
        "created_at": now()
    })
    
    redis.expire(f"upload:{upload_id}", 86400)  # 24 hour TTL
    return upload_id

def upload_chunk(upload_id, chunk_num, data):
    # Upload to S3 Multipart
    s3_upload_id = redis.hget(f"upload:{upload_id}", "s3_upload_id")
    etag = s3.upload_part(
        UploadId=s3_upload_id,
        PartNumber=chunk_num,
        Body=data
    )
    
    # Mark chunk complete
    completed = json.loads(redis.hget(f"upload:{upload_id}", "chunks_completed"))
    completed.append({"chunk_num": chunk_num, "etag": etag})
    redis.hset(f"upload:{upload_id}", "chunks_completed", json.dumps(completed))
    
    return {"chunk_num": chunk_num, "status": "completed"}
```

**Client Resume Logic**:

```javascript
async function uploadVideoResumable(file) {
    const CHUNK_SIZE = 10 * 1024 * 1024;  // 10 MB
    const chunks = Math.ceil(file.size / CHUNK_SIZE);
    
    // Init upload
    const {upload_id} = await fetch('/upload/init', {
        method: 'POST',
        body: JSON.stringify({filename: file.name, size: file.size})
    }).then(r => r.json());
    
    // Upload chunks
    for (let i = 0; i < chunks; i++) {
        const start = i * CHUNK_SIZE;
        const end = Math.min(start + CHUNK_SIZE, file.size);
        const chunk = file.slice(start, end);
        
        try {
            await fetch('/upload/chunk', {
                method: 'POST',
                body: chunk,
                headers: {
                    'X-Upload-ID': upload_id,
                    'X-Chunk-Number': i,
                    'Content-Type': 'application/octet-stream'
                }
            });
            
            updateProgress((i + 1) / chunks * 100);
        } catch (error) {
            // Resume: retry this chunk
            console.log(`Chunk ${i} failed, retrying...`);
            i--;  // Retry same chunk
            await sleep(1000);
        }
    }
    
    // Complete upload
    await fetch('/upload/complete', {
        method: 'POST',
        body: JSON.stringify({upload_id})
    });
}
```

### Trade-offs Accepted

| What We Gain                 | What We Sacrifice                    |
|------------------------------|--------------------------------------|
| ✅ 99% upload success rate    | ❌ More complex client logic          |
| ✅ Better UX (resume on fail) | ❌ Server state management (Redis)    |
| ✅ 50% bandwidth savings      | ❌ More API endpoints (3 vs 1)        |
| ✅ Mobile-friendly            | ❌ Cleanup required (expired uploads) |

### When to Reconsider

**Switch to Single Request if**:

- Small files only (<50 MB)
- Reliable network (datacenter to datacenter)
- Simple requirements (prototyping)

**Switch to gRPC Streaming if**:

- Want bidirectional communication
- Need lower overhead (binary protocol)
- Have gRPC infrastructure

### Real-World Examples

- **YouTube**: Resumable upload (TUS-like protocol)
- **Vimeo**: TUS protocol (resumable.js)
- **Dropbox**: Chunked upload (custom protocol)
- **Google Drive**: Resumable upload (Google's protocol)

---

## 6. Synchronous vs Asynchronous Transcoding

### The Problem

Transcoding takes 5-30 minutes. Should the upload API wait for transcoding to complete before returning to the user?

### Options Considered

| Approach         | User Experience          | API Response Time | Throughput               | Complexity  |
|------------------|--------------------------|-------------------|--------------------------|-------------|
| **Synchronous**  | ❌ Terrible (30 min wait) | ❌ 30 minutes      | ❌ Low (1 req = 1 worker) | ✅ Simple    |
| **Asynchronous** | ✅ Excellent (instant)    | ✅ 200ms           | ✅ High (decoupled)       | ⚠️ Moderate |
| **Hybrid**       | ⚠️ OK (poll status)      | ⚠️ 5 seconds      | ⚠️ Medium                | ⚠️ Moderate |

### Decision Made

**Asynchronous Transcoding with Status Polling/WebSocket**

### Rationale

1. **User Experience**: User shouldn't wait 30 minutes (browser timeout at 60s)
2. **API Throughput**: Upload service can handle 10k req/sec (not blocked by transcode)
3. **Resource Efficiency**: Transcoding workers scale independently of upload API
4. **Fault Tolerance**: Failed transcode doesn't fail upload (can retry)
5. **Priority Queuing**: High-priority videos processed first (paid users)

**Why NOT Synchronous**:

- Browser/mobile timeouts (60 second limit)
- Terrible UX (user waits 30 minutes)
- API gateway blocked (1 request = 1 thread for 30 min)
- Can't scale (need 10k workers for 10k concurrent uploads)

**Why NOT Hybrid (Long Polling with Timeout)**:

- Still wastes API Gateway resources (held connections)
- Partial improvement (still need status polling afterward)
- More complex than pure async

### Implementation Details

**Upload API (Instant Response)**:

```python
@app.post("/upload/complete")
async def complete_upload(upload_id: str):
    # 1. Finalize S3 multipart upload (100ms)
    s3_key = finalize_s3_multipart(upload_id)
    video_id = generate_video_id()
    
    # 2. Create metadata entry (10ms)
    cassandra.execute("""
        INSERT INTO videos (video_id, title, status, s3_key, created_at)
        VALUES (?, ?, 'processing', ?, ?)
    """, [video_id, title, s3_key, now()])
    
    # 3. Publish transcode job (async, 5ms)
    kafka.produce(
        topic="transcode-jobs",
        key=video_id,
        value={"video_id": video_id, "s3_key": s3_key}
    )
    
    # 4. Return immediately (total: 200ms)
    return {
        "video_id": video_id,
        "status": "processing",
        "estimated_time": "5-10 minutes"
    }
```

**Status Polling (Client)**:

```javascript
async function waitForTranscode(video_id) {
    while (true) {
        const {status} = await fetch(`/video/${video_id}/status`)
            .then(r => r.json());
        
        if (status === "ready") {
            console.log("Video is ready!");
            break;
        } else if (status === "failed") {
            console.error("Transcoding failed");
            break;
        }
        
        // Poll every 5 seconds
        await sleep(5000);
    }
}
```

**WebSocket Alternative (Real-Time)**:

```javascript
const ws = new WebSocket(`wss://api.example.com/video/${video_id}/status`);

ws.onmessage = (event) => {
    const {status, progress} = JSON.parse(event.data);
    
    if (status === "processing") {
        updateProgress(progress);  // Show 45% complete
    } else if (status === "ready") {
        showSuccessMessage();
        ws.close();
    }
};
```

### Trade-offs Accepted

| What We Gain                           | What We Sacrifice                     |
|----------------------------------------|---------------------------------------|
| ✅ Instant API response (200ms)         | ❌ User must wait/poll for completion  |
| ✅ High API throughput (10k req/sec)    | ❌ More complex (status tracking)      |
| ✅ Independent scaling (API vs workers) | ❌ Two-phase UX (upload then wait)     |
| ✅ Fault tolerance (retry failed jobs)  | ❌ WebSocket infrastructure (optional) |

### When to Reconsider

**Switch to Synchronous if**:

- Transcode time < 5 seconds (fast enough to wait)
- Low traffic (< 10 concurrent uploads)
- Simple prototype (don't care about UX)

**Switch to Hybrid if**:

- Want balance (wait up to 10s, then async)
- Can tolerate held connections
- Small scale (< 100 concurrent uploads)

### Real-World Examples

- **YouTube**: Async (polling + notifications)
- **Vimeo**: Async with WebSocket updates
- **TikTok**: Async (optimized for mobile)
- **Instagram**: Async (background processing)

---

## 7. Multi-Image vs Single-Image Transcoding

### The Problem

Support 20+ video codecs and qualities. Should we use one Docker image with all codecs, or separate images per codec?

### Options Considered

| Approach                    | Image Size  | Build Time | Deployment | Flexibility | Maintenance |
|-----------------------------|-------------|------------|------------|-------------|-------------|
| **Single Monolithic**       | ❌ 10 GB     | ❌ 60 min   | ❌ Slow     | ❌ Low       | ❌ Complex   |
| **Multi-Image (per codec)** | ✅ 2 GB each | ✅ 10 min   | ✅ Fast     | ✅ High      | ✅ Simple    |
| **Layered (base + codec)**  | ⚠️ 4 GB     | ⚠️ 20 min  | ✅ Fast     | ✅ High      | ⚠️ Moderate |

### Decision Made

**Multi-Image Strategy with Base Image + Codec Layers**

### Rationale

1. **Fast Builds**: Update H.265 codec → rebuild only H.265 image (10 min vs 60 min)
2. **Fast Deployment**: Deploy H.265 workers → only H.265 pods restart
3. **Small Images**: 2 GB per codec vs 10 GB monolithic (5x smaller)
4. **Flexibility**: Can scale H.264 and H.265 workers independently
5. **Resource Efficiency**: Workers only load codecs they use

**Why NOT Monolithic**:

- 10 GB image takes 20 minutes to pull (cold start)
- Update one codec → rebuild entire image (60 minutes)
- Deploy → restart all workers (downtime)
- Wasted RAM (load codecs never used)

### Implementation Details

**Base Image (Shared)**:

```dockerfile
FROM ubuntu:22.04 as base

RUN apt-get update && apt-get install -y \
    python3 \
    ffmpeg-base \
    libx264-common \
    ca-certificates \
    curl

# Common transcoding scripts
COPY transcode_worker.py /app/
COPY utils.py /app/

WORKDIR /app
```

**H.264 Image**:

```dockerfile
FROM transcode-base:latest

RUN apt-get install -y \
    libx264-dev \
    libfdk-aac-dev

ENV CODEC=h264
CMD ["python3", "transcode_worker.py", "--codec=h264"]
```

**H.265 Image**:

```dockerfile
FROM transcode-base:latest

RUN apt-get install -y \
    libx265-dev \
    libfdk-aac-dev

ENV CODEC=h265
CMD ["python3", "transcode_worker.py", "--codec=h265"]
```

**Worker Deployment**:

```yaml
# H.264 workers (most common, 80% of videos)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: transcode-h264
spec:
  replicas: 50
  template:
    spec:
      containers:
        - name: transcoder
          image: transcode:h264-v2.3
          resources:
            requests:
              cpu: "8"
              memory: "16Gi"

# H.265 workers (high quality, 20% of videos)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: transcode-h265
spec:
  replicas: 10
  template:
    spec:
      containers:
        - name: transcoder
          image: transcode:h265-v2.3
          resources:
            requests:
              cpu: "16"  # H.265 needs more CPU
              memory: "32Gi"
```

### Trade-offs Accepted

| What We Gain                           | What We Sacrifice                    |
|----------------------------------------|--------------------------------------|
| ✅ Fast builds (10 min vs 60 min)       | ❌ More Dockerfiles to maintain       |
| ✅ Fast deployment (per-codec)          | ❌ More complex CI/CD pipeline        |
| ✅ Small images (2 GB vs 10 GB)         | ❌ Must route jobs to correct workers |
| ✅ Independent scaling (H.264 vs H.265) | ❌ More images to store/manage        |

### When to Reconsider

**Switch to Monolithic if**:

- Support < 3 codecs (small image size)
- All videos use all codecs (no waste)
- Simple deployment preferred
- Team small (can't maintain multiple images)

**Switch to Serverless (Lambda) if**:

- Sporadic traffic (not 24/7)
- Short transcode times (< 15 minutes)
- Want zero ops (fully managed)

### Real-World Examples

- **Netflix**: Multi-image (per-codec workers)
- **YouTube**: Layered (base + codec layers)
- **Vimeo**: Monolithic (smaller scale)
- **AWS Elemental**: Serverless (Lambda-based)

---

## 8. S3 vs Database for Video Storage

### The Problem

Store 500 PB of video files (raw + transcoded). Should we store in object storage (S3) or database (PostgreSQL BLOBs)?

### Options Considered

| Feature             | S3                         | PostgreSQL BYTEA       | Database (MongoDB GridFS) |
|---------------------|----------------------------|------------------------|---------------------------|
| **Cost**            | ✅ $0.02/GB                 | ❌ $0.10/GB             | ⚠️ $0.05/GB               |
| **Scalability**     | ✅ Unlimited                | ❌ Limited (table size) | ⚠️ Horizontal (sharding)  |
| **CDN Integration** | ✅ Native (S3 → CloudFront) | ❌ None (need proxy)    | ❌ None                    |
| **Throughput**      | ✅ Unlimited                | ❌ IOPS bottleneck      | ⚠️ Shard-dependent        |
| **Backup**          | ✅ Built-in versioning      | ⚠️ Must configure      | ⚠️ Must configure         |
| **Latency**         | ✅ 10-50ms                  | ⚠️ 50-200ms            | ⚠️ 100-500ms              |

### Decision Made

**S3 for Video Storage, Database for Metadata Only**

### Rationale

1. **Cost**: $0.02/GB (S3) vs $0.10/GB (PostgreSQL) = 5x cheaper
2. **Scalability**: S3 unlimited, database limited to TB-scale
3. **CDN Integration**: S3 → CloudFront native, database needs proxy
4. **Purpose-Built**: Object storage designed for large files, databases for structured data
5. **Durability**: S3 11-nines durability, database requires manual backup

**Why NOT Database**:

- 5x more expensive per GB
- Not designed for large BLOBs (video is 100 MB to 10 GB)
- Database IOPS exhausted by video reads
- Can't leverage CDN (database not HTTP-accessible)
- Backup/restore slow (include video in DB backup)

### Implementation Details

**S3 Bucket Structure**:

```
videos-raw/
  2024/
    01/
      15/
        video_123_raw.mp4 (5 GB)

videos-processed/
  video_123/
    360p/
      segment_000.ts
      segment_001.ts
      playlist.m3u8
    720p/
      segment_000.ts
      segment_001.ts
      playlist.m3u8
    master.m3u8
```

**Metadata in Database**:

```sql
CREATE TABLE videos (
    video_id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT,
    -- NO video data here, only URLs
    raw_s3_key TEXT,  -- s3://videos-raw/2024/01/15/video_123_raw.mp4
    cdn_base_url TEXT,  -- https://d123abc.cloudfront.net/video_123/
    thumbnail_s3_key TEXT,
    duration INTEGER,
    file_size BIGINT,
    created_at TIMESTAMP
);
```

**Access Pattern**:

```python
def get_video_stream_url(video_id):
    # 1. Get metadata from database (10ms)
    video = db.query("SELECT cdn_base_url FROM videos WHERE video_id = ?", [video_id])
    
    # 2. Return CDN URL (client fetches from CDN, not from our API)
    return {
        "master_m3u8": f"{video.cdn_base_url}/master.m3u8",
        "thumbnail": f"{video.cdn_base_url}/thumbnail.jpg"
    }

# Client directly requests from CDN
# GET https://d123abc.cloudfront.net/video_123/master.m3u8
```

### Trade-offs Accepted

| What We Gain                    | What We Sacrifice                           |
|---------------------------------|---------------------------------------------|
| ✅ 5x cheaper storage ($0.02/GB) | ❌ Two systems to manage (S3 + DB)           |
| ✅ Unlimited scalability         | ❌ Must sync metadata (eventual consistency) |
| ✅ Native CDN integration        | ❌ More complex architecture                 |
| ✅ 11-nines durability           | ❌ S3 outages affect availability            |

### When to Reconsider

**Switch to Database if**:

- Small videos only (< 10 MB)
- Low scale (< 1 TB total)
- Need ACID transactions (video + metadata atomic)
- Want single system (simplicity)

**Use GridFS (MongoDB) if**:

- Already using MongoDB
- Need sharding for scale
- Files 10-100 MB (GridFS sweet spot)

### Real-World Examples

- **Netflix**: S3 + custom storage (petabyte scale)
- **YouTube**: Google Cloud Storage + BigTable
- **Vimeo**: S3 + PostgreSQL (metadata)
- **TikTok**: Custom object storage (massive scale)

---

## 9. WebSocket vs Polling for Real-Time Updates

### The Problem

Show users real-time transcoding progress (0% → 45% → 100%). Should we use WebSockets or polling?

### Options Considered

| Approach                     | Latency   | Server Load                  | Complexity  | Firewall Friendly |
|------------------------------|-----------|------------------------------|-------------|-------------------|
| **Polling (GET /status)**    | ⚠️ 5s avg | ❌ High (constant requests)   | ✅ Simple    | ✅ Yes             |
| **Long Polling**             | ✅ 1s      | ⚠️ Medium (held connections) | ⚠️ Moderate | ✅ Yes             |
| **WebSocket**                | ✅ 100ms   | ✅ Low (single connection)    | ⚠️ Moderate | ⚠️ Sometimes      |
| **Server-Sent Events (SSE)** | ✅ 100ms   | ✅ Low                        | ✅ Simple    | ✅ Yes             |

### Decision Made

**WebSocket for Real-Time, Polling as Fallback**

### Rationale

1. **Low Latency**: 100ms update (vs 5s polling)
2. **Low Server Load**: 1 connection (vs 1 req/5s polling)
3. **Bidirectional**: Can cancel transcode (send command to server)
4. **Better UX**: Smooth progress bar (not jumpy)
5. **Efficient**: Push updates only when progress changes

**Why NOT Polling Only**:

- High server load (100k users × 1 req/5s = 20k QPS)
- Poor UX (5-second delay, jumpy progress bar)
- Wastes bandwidth (most polls return "no change")

**Why NOT SSE Only**:

- One-directional (can't cancel transcode)
- Browser limits (6 connections per domain)
- Less mature than WebSocket

### Implementation Details

**WebSocket Server (Backend)**:

```python
from fastapi import WebSocket

@app.websocket("/ws/transcode/{video_id}")
async def transcode_status_websocket(websocket: WebSocket, video_id: str):
    await websocket.accept()
    
    # Subscribe to Redis Pub/Sub for this video
    pubsub = redis.pubsub()
    pubsub.subscribe(f"transcode:{video_id}")
    
    try:
        for message in pubsub.listen():
            if message['type'] == 'message':
                # Forward update to client
                data = json.loads(message['data'])
                await websocket.send_json(data)
                
                # Close if complete
                if data['status'] == 'ready':
                    break
    finally:
        pubsub.unsubscribe()
        await websocket.close()
```

**Worker Publishes Progress**:

```python
def transcode_video(video_id):
    # Transcode quality by quality
    for i, quality in enumerate(["360p", "720p", "1080p"]):
        transcode_quality(video_id, quality)
        
        # Publish progress
        progress = (i + 1) / 3 * 100
        redis.publish(f"transcode:{video_id}", json.dumps({
            "status": "processing",
            "progress": progress,
            "current_quality": quality
        }))
    
    # Final update
    redis.publish(f"transcode:{video_id}", json.dumps({
        "status": "ready",
        "progress": 100
    }))
```

**Client (WebSocket with Polling Fallback)**:

```javascript
async function trackTranscodeProgress(video_id) {
    try {
        // Try WebSocket first
        const ws = new WebSocket(`wss://api.example.com/ws/transcode/${video_id}`);
        
        ws.onmessage = (event) => {
            const {status, progress} = JSON.parse(event.data);
            updateProgressBar(progress);
            
            if (status === 'ready') {
                showVideo();
                ws.close();
            }
        };
        
        ws.onerror = () => {
            // Fallback to polling
            pollTranscodeStatus(video_id);
        };
    } catch (error) {
        // Fallback to polling
        pollTranscodeStatus(video_id);
    }
}

async function pollTranscodeStatus(video_id) {
    while (true) {
        const {status, progress} = await fetch(`/api/transcode/${video_id}/status`)
            .then(r => r.json());
        
        updateProgressBar(progress);
        
        if (status === 'ready') {
            showVideo();
            break;
        }
        
        await sleep(5000);  // Poll every 5 seconds
    }
}
```

### Trade-offs Accepted

| What We Gain                          | What We Sacrifice                        |
|---------------------------------------|------------------------------------------|
| ✅ Real-time updates (100ms)           | ❌ WebSocket infrastructure               |
| ✅ Low server load (1 conn vs 20k QPS) | ❌ Firewall issues (some block WebSocket) |
| ✅ Bidirectional (cancel transcode)    | ❌ Connection management (heartbeat)      |
| ✅ Better UX (smooth progress)         | ❌ More complex client code               |

### When to Reconsider

**Switch to Polling if**:

- Low traffic (< 1k concurrent users)
- Simple requirements (5-second delay OK)
- Want maximum compatibility (firewalls)

**Switch to SSE if**:

- One-directional only (no cancel needed)
- Want simpler protocol (HTTP-based)
- Better firewall compatibility

### Real-World Examples

- **YouTube**: WebSocket with polling fallback
- **Vimeo**: Server-Sent Events (SSE)
- **TikTok**: WebSocket (mobile optimized)
- **Instagram**: Polling (simple, mobile-friendly)

---

## 10. Multi-Region vs Single-Region Deployment

### The Problem

Serve 100M users globally. Deploy in single region (US-East) or multiple regions (US, EU, Asia)?

### Options Considered

| Approach                          | Latency (Global)         | Cost           | Complexity  | Availability |
|-----------------------------------|--------------------------|----------------|-------------|--------------|
| **Single Region**                 | ❌ 150ms avg (Asia users) | ✅ $100k/month  | ✅ Simple    | ⚠️ 99.9%     |
| **Multi-Region (Active-Passive)** | ⚠️ 100ms avg             | ⚠️ $150k/month | ⚠️ Moderate | ✅ 99.95%     |
| **Multi-Region (Active-Active)**  | ✅ 20ms avg               | ❌ $200k/month  | ❌ Complex   | ✅ 99.99%     |

### Decision Made

**Multi-Region Active-Active (US-East, EU-West, Asia-Pacific)**

### Rationale

1. **Low Latency**: 20ms avg (vs 150ms single region)
2. **High Availability**: 99.99% uptime (region failover)
3. **Data Sovereignty**: EU users' data stays in EU (GDPR)
4. **Load Distribution**: Spread 100M users across 3 regions
5. **Disaster Recovery**: RTO < 1 minute

**Latency by Region**:

```
Single Region (US-East):
- US users: 20ms ✅
- EU users: 100ms ⚠️
- Asia users: 200ms ❌
- Average: 106ms

Multi-Region:
- US users → US-East: 20ms ✅
- EU users → EU-West: 20ms ✅
- Asia users → Asia-Pacific: 20ms ✅
- Average: 20ms
```

**Cost Comparison**:

```
Single Region:
- Infrastructure: $80k/month
- CDN: $20k/month (higher, no regional caches)
- Total: $100k/month

Multi-Region:
- Infrastructure: $150k/month (3x regions, shared load)
- CDN: $30k/month (regional caches)
- Data Transfer (cross-region): $20k/month
- Total: $200k/month
```

**Why NOT Single Region**:

- Poor latency for global users (150ms avg)
- Single point of failure (region outage = downtime)
- GDPR concerns (EU data in US)
- CDN costs higher (no regional cache)

**Why NOT Active-Passive**:

- Wasted resources (passive region idle)
- Slower failover (need to warm up passive)
- Still poor latency for some users

### Implementation Details

**Regional Deployment**:

```
US-East (Primary):
- API Gateway (10 instances)
- Cassandra (5 nodes)
- Redis (3 nodes)
- Transcoding Workers (20 instances)
- S3 (origin bucket)

EU-West (Full):
- API Gateway (8 instances)
- Cassandra (5 nodes, replicates from US)
- Redis (3 nodes)
- Transcoding Workers (15 instances)
- S3 (replica bucket)

Asia-Pacific (Full):
- API Gateway (12 instances, largest user base)
- Cassandra (5 nodes, replicates from US)
- Redis (3 nodes)
- Transcoding Workers (25 instances)
- S3 (replica bucket)
```

**GeoDNS Routing (Route53)**:

```json
{
  "HostedZone": "api.example.com",
  "RoutingPolicy": "Latency",
  "Records": [
    {
      "Name": "api.example.com",
      "Type": "A",
      "Region": "us-east-1",
      "Value": "54.123.45.67",
      "HealthCheck": "hc-us-east"
    },
    {
      "Name": "api.example.com",
      "Type": "A",
      "Region": "eu-west-1",
      "Value": "18.234.56.78",
      "HealthCheck": "hc-eu-west"
    },
    {
      "Name": "api.example.com",
      "Type": "A",
      "Region": "ap-southeast-1",
      "Value": "13.345.67.89",
      "HealthCheck": "hc-asia"
    }
  ]
}
```

**Cassandra Multi-DC Replication**:

```sql
CREATE KEYSPACE videos WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'us-east': 3,
  'eu-west': 3,
  'asia-pacific': 3
};

-- Read/write with LOCAL_QUORUM
-- (only need 2/3 replicas in local datacenter)
```

### Trade-offs Accepted

| What We Gain                      | What We Sacrifice                       |
|-----------------------------------|-----------------------------------------|
| ✅ 5x lower latency (150ms → 20ms) | ❌ 2x cost ($100k → $200k)               |
| ✅ 99.99% availability             | ❌ Complex deployment (3 regions)        |
| ✅ GDPR compliance                 | ❌ Cross-region replication lag (500ms)  |
| ✅ Load distribution               | ❌ Monitoring complexity (3x dashboards) |

### When to Reconsider

**Switch to Single Region if**:

- Users in single geography (US-only startup)
- Small scale (< 1M users)
- Budget constrained (can't afford 2x cost)
- Simple operations preferred

**Switch to Active-Passive if**:

- Want disaster recovery but not global latency
- Can tolerate 5-minute failover
- Budget < $150k/month

### Real-World Examples

- **Netflix**: Multi-region active-active (global CDN)
- **YouTube**: Multi-region (Google's global network)
- **TikTok**: Multi-region (massive Asia presence)
- **Vimeo**: Single region (US-focused, smaller scale)

---

**Summary Table: All Design Decisions**

| Decision               | Chosen Approach                | Key Rationale                            | When to Reconsider                        |
|------------------------|--------------------------------|------------------------------------------|-------------------------------------------|
| **Streaming Protocol** | HLS (primary), DASH (optional) | Universal compatibility, iOS requirement | Switch to DASH for low latency            |
| **Transcode Queue**    | Kafka                          | High throughput, ordering, replay        | Switch to SQS for simplicity at low scale |
| **Metadata DB**        | Cassandra + Redis              | Low latency, linear scaling              | Switch to DynamoDB for fully managed      |
| **Content Delivery**   | CloudFront CDN                 | 88% cost savings, 7.5x faster            | Switch to custom CDN at Netflix scale     |
| **Upload Method**      | Resumable (TUS)                | 99% success vs 50% single-request        | Switch to single-request for small files  |
| **Transcode Timing**   | Asynchronous                   | Instant API response, high throughput    | Switch to sync if < 5s transcode time     |
| **Transcode Images**   | Multi-image (per codec)        | Fast builds, independent scaling         | Switch to monolithic if < 3 codecs        |
| **Video Storage**      | S3                             | 5x cheaper, CDN integration              | Switch to DB for small videos (< 10 MB)   |
| **Real-Time Updates**  | WebSocket + polling fallback   | Low latency, low server load             | Switch to polling for simplicity          |
| **Deployment**         | Multi-region active-active     | 5x lower latency, 99.99% uptime          | Switch to single region for US-only users |
