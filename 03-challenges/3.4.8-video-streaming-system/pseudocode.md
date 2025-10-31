# Video Streaming System - Pseudocode Implementations

This document contains detailed algorithm implementations for the Video Streaming System. The main challenge document references these functions.

---

## Table of Contents

1. [Upload Service](#1-upload-service)
2. [Transcoding Pipeline](#2-transcoding-pipeline)
3. [Adaptive Bitrate (ABR) Algorithm](#3-adaptive-bitrate-abr-algorithm)
4. [CDN and Caching](#4-cdn-and-caching)
5. [Metadata Management](#5-metadata-management)
6. [Search and Recommendation](#6-search-and-recommendation)
7. [Analytics Processing](#7-analytics-processing)
8. [Live Streaming](#8-live-streaming)

---

## 1. Upload Service

### init_resumable_upload()

**Purpose:** Initialize a resumable upload session for large video files.

**Parameters:**
- filename (string): Original filename
- file_size (integer): Total file size in bytes
- content_type (string): MIME type (e.g., "video/mp4")
- user_id (string): ID of uploading user

**Returns:**
- UploadSession: Object containing upload_id, chunk_size, s3_upload_id

**Algorithm:**
```
function init_resumable_upload(filename, file_size, content_type, user_id):
    // 1. Generate unique upload ID
    upload_id = generate_uuid()
    
    // 2. Calculate optimal chunk size (10 MB default, max 10,000 chunks)
    chunk_size = 10 * 1024 * 1024  // 10 MB
    chunks_total = ceil(file_size / chunk_size)
    
    if chunks_total > 10000:
        chunk_size = ceil(file_size / 10000)  // S3 limit: 10,000 parts
    
    // 3. Initiate S3 multipart upload
    s3_bucket = "videos-raw"
    s3_key = f"{year}/{month}/{day}/{upload_id}_{filename}"
    s3_upload_id = s3_client.create_multipart_upload(
        Bucket=s3_bucket,
        Key=s3_key,
        ContentType=content_type
    )
    
    // 4. Create upload state in Redis (TTL: 24 hours)
    redis.hset(f"upload:{upload_id}", {
        "filename": filename,
        "file_size": file_size,
        "content_type": content_type,
        "user_id": user_id,
        "chunk_size": chunk_size,
        "chunks_total": chunks_total,
        "chunks_completed": "[]",  // JSON array
        "s3_bucket": s3_bucket,
        "s3_key": s3_key,
        "s3_upload_id": s3_upload_id,
        "created_at": now(),
        "status": "in_progress"
    })
    redis.expire(f"upload:{upload_id}", 86400)  // 24 hours
    
    // 5. Return session info
    return {
        "upload_id": upload_id,
        "chunk_size": chunk_size,
        "chunks_total": chunks_total,
        "s3_key": s3_key
    }
```

**Time Complexity:** O(1)

**Example Usage:**
```
session = init_resumable_upload(
    filename="vacation.mp4",
    file_size=5368709120,  // 5 GB
    content_type="video/mp4",
    user_id="user_123"
)
// Returns: {upload_id: "abc-123", chunk_size: 10485760, chunks_total: 500}
```

---

### upload_chunk()

**Purpose:** Upload a single chunk and track progress.

**Parameters:**
- upload_id (string): Upload session ID
- chunk_number (integer): Chunk number (0-indexed)
- chunk_data (bytes): Chunk binary data

**Returns:**
- ChunkResult: Status, ETag, progress percentage

**Algorithm:**
```
function upload_chunk(upload_id, chunk_number, chunk_data):
    // 1. Validate upload session exists
    upload = redis.hgetall(f"upload:{upload_id}")
    if not upload:
        throw UploadNotFoundException(upload_id)
    
    // 2. Validate chunk number in range
    chunks_total = int(upload["chunks_total"])
    if chunk_number < 0 or chunk_number >= chunks_total:
        throw InvalidChunkNumberException(chunk_number, chunks_total)
    
    // 3. Check if chunk already uploaded (idempotency)
    completed = json.loads(upload["chunks_completed"])
    for chunk in completed:
        if chunk["chunk_number"] == chunk_number:
            return {
                "status": "already_uploaded",
                "chunk_number": chunk_number,
                "etag": chunk["etag"],
                "progress": len(completed) / chunks_total * 100
            }
    
    // 4. Upload chunk to S3 multipart
    try:
        response = s3_client.upload_part(
            Bucket=upload["s3_bucket"],
            Key=upload["s3_key"],
            UploadId=upload["s3_upload_id"],
            PartNumber=chunk_number + 1,  // S3 parts are 1-indexed
            Body=chunk_data
        )
        etag = response["ETag"]
    catch S3Exception as e:
        throw ChunkUploadFailedException(chunk_number, e)
    
    // 5. Mark chunk as completed
    completed.append({
        "chunk_number": chunk_number,
        "part_number": chunk_number + 1,
        "etag": etag,
        "uploaded_at": now()
    })
    redis.hset(f"upload:{upload_id}", "chunks_completed", json.dumps(completed))
    
    // 6. Calculate progress
    progress = len(completed) / chunks_total * 100
    
    // 7. Publish progress event (for WebSocket)
    redis.publish(f"upload:{upload_id}:progress", json.dumps({
        "chunk_number": chunk_number,
        "progress": progress,
        "chunks_completed": len(completed),
        "chunks_total": chunks_total
    }))
    
    return {
        "status": "uploaded",
        "chunk_number": chunk_number,
        "etag": etag,
        "progress": progress
    }
```

**Time Complexity:** O(n) where n = number of completed chunks (JSON parsing)

**Example Usage:**
```
result = upload_chunk(
    upload_id="abc-123",
    chunk_number=42,
    chunk_data=bytes(...)  // 10 MB
)
// Returns: {status: "uploaded", chunk_number: 42, etag: "abc...", progress: 8.4}
```

---

### complete_upload()

**Purpose:** Finalize multipart upload and trigger transcoding.

**Parameters:**
- upload_id (string): Upload session ID

**Returns:**
- VideoInfo: video_id, status, s3_key

**Algorithm:**
```
function complete_upload(upload_id):
    // 1. Get upload session
    upload = redis.hgetall(f"upload:{upload_id}")
    if not upload:
        throw UploadNotFoundException(upload_id)
    
    // 2. Verify all chunks uploaded
    completed = json.loads(upload["chunks_completed"])
    chunks_total = int(upload["chunks_total"])
    
    if len(completed) != chunks_total:
        throw IncompleteUploadException(
            uploaded=len(completed),
            required=chunks_total
        )
    
    // 3. Sort parts by part number (S3 requires sorted order)
    parts = sorted(completed, key=lambda x: x["part_number"])
    parts_list = [{"PartNumber": p["part_number"], "ETag": p["etag"]} for p in parts]
    
    // 4. Complete S3 multipart upload
    try:
        s3_client.complete_multipart_upload(
            Bucket=upload["s3_bucket"],
            Key=upload["s3_key"],
            UploadId=upload["s3_upload_id"],
            MultipartUpload={"Parts": parts_list}
        )
    catch S3Exception as e:
        throw UploadCompletionFailedException(e)
    
    // 5. Generate video ID and create metadata entry
    video_id = generate_video_id()  // e.g., "v_1234567890abc"
    
    cassandra.execute("""
        INSERT INTO videos (
            video_id, title, description, status, raw_s3_key,
            file_size, content_type, user_id, created_at
        ) VALUES (?, ?, ?, 'processing', ?, ?, ?, ?, ?)
    """, [
        video_id,
        upload["filename"],
        "",  // Description added later by user
        upload["s3_key"],
        int(upload["file_size"]),
        upload["content_type"],
        upload["user_id"],
        now()
    ])
    
    // 6. Publish transcode job to Kafka
    transcode_job = {
        "video_id": video_id,
        "s3_bucket": upload["s3_bucket"],
        "s3_key": upload["s3_key"],
        "file_size": int(upload["file_size"]),
        "content_type": upload["content_type"],
        "priority": "normal",  // "high" for verified channels
        "qualities": ["360p", "480p", "720p", "1080p"],
        "created_at": now()
    }
    
    kafka_producer.send(
        topic="transcode-jobs",
        key=video_id,
        value=json.dumps(transcode_job),
        partition=hash(video_id) % 50  // Partition by video_id
    )
    
    // 7. Cleanup upload state
    redis.delete(f"upload:{upload_id}")
    
    // 8. Notify user via WebSocket
    redis.publish(f"upload:{upload_id}:complete", json.dumps({
        "video_id": video_id,
        "status": "processing",
        "message": "Upload complete, transcoding started"
    }))
    
    return {
        "video_id": video_id,
        "status": "processing",
        "s3_key": upload["s3_key"],
        "estimated_time_minutes": estimate_transcode_time(int(upload["file_size"]))
    }
```

**Time Complexity:** O(n log n) where n = number of chunks (sorting)

**Example Usage:**
```
video = complete_upload(upload_id="abc-123")
// Returns: {video_id: "v_abc123", status: "processing", estimated_time_minutes: 12}
```

---

## 2. Transcoding Pipeline

### transcode_video()

**Purpose:** Main transcoding function that processes video into multiple qualities.

**Parameters:**
- video_id (string): Video identifier
- s3_key (string): S3 path to raw video
- qualities (list): List of target qualities (e.g., ["360p", "720p", "1080p"])

**Returns:**
- TranscodeResult: Status, CDN URLs for each quality

**Algorithm:**
```
function transcode_video(video_id, s3_key, qualities):
    // 1. Download raw video from S3
    raw_video_path = f"/tmp/{video_id}_raw.mp4"
    download_start = now()
    
    try:
        s3_client.download_file(
            Bucket="videos-raw",
            Key=s3_key,
            Filename=raw_video_path
        )
        download_time = now() - download_start
        log(f"Downloaded {s3_key} in {download_time}s")
    catch S3Exception as e:
        mark_transcode_failed(video_id, f"Download failed: {e}")
        throw
    
    // 2. Probe video metadata (duration, resolution, codec)
    probe = ffprobe(raw_video_path)
    duration_seconds = probe.format.duration
    width = probe.streams[0].width
    height = probe.streams[0].height
    
    // 3. Update video metadata
    cassandra.execute("""
        UPDATE videos SET duration = ?, width = ?, height = ?
        WHERE video_id = ?
    """, [duration_seconds, width, height, video_id])
    
    // 4. Filter qualities based on source resolution
    // (don't upscale: if source is 720p, skip 1080p and 4K)
    target_qualities = filter_qualities(qualities, height)
    
    // 5. Transcode each quality in parallel
    transcode_results = []
    
    parallel_for quality in target_qualities:
        result = transcode_quality(video_id, raw_video_path, quality, duration_seconds)
        transcode_results.append(result)
        
        // Publish progress
        progress = len(transcode_results) / len(target_qualities) * 100
        redis.publish(f"transcode:{video_id}", json.dumps({
            "status": "processing",
            "progress": progress,
            "current_quality": quality
        }))
    
    // 6. Generate HLS master manifest
    master_manifest = generate_master_manifest(video_id, transcode_results)
    s3_client.put_object(
        Bucket="videos-processed",
        Key=f"{video_id}/master.m3u8",
        Body=master_manifest,
        ContentType="application/vnd.apple.mpegurl"
    )
    
    // 7. Generate thumbnail
    thumbnail_path = generate_thumbnail(raw_video_path, duration_seconds)
    s3_client.upload_file(
        Filename=thumbnail_path,
        Bucket="videos-processed",
        Key=f"{video_id}/thumbnail.jpg"
    )
    
    // 8. Update video status to "ready"
    cdn_base_url = f"https://d123abc.cloudfront.net/{video_id}"
    
    cassandra.execute("""
        UPDATE videos SET
            status = 'ready',
            cdn_base_url = ?,
            thumbnail_url = ?,
            qualities = ?,
            transcoded_at = ?
        WHERE video_id = ?
    """, [
        cdn_base_url,
        f"{cdn_base_url}/thumbnail.jpg",
        target_qualities,
        now(),
        video_id
    ])
    
    // 9. Invalidate CDN cache (if video re-transcode)
    cloudfront.create_invalidation(
        DistributionId="E123ABC",
        Paths=[f"/{video_id}/*"]
    )
    
    // 10. Cleanup temp files
    delete_file(raw_video_path)
    delete_file(thumbnail_path)
    
    // 11. Final progress update
    redis.publish(f"transcode:{video_id}", json.dumps({
        "status": "ready",
        "progress": 100,
        "cdn_url": cdn_base_url
    }))
    
    return {
        "video_id": video_id,
        "status": "ready",
        "cdn_base_url": cdn_base_url,
        "qualities": target_qualities,
        "duration": duration_seconds
    }
```

**Time Complexity:** O(n × m) where n = number of qualities, m = video duration

**Example Usage:**
```
result = transcode_video(
    video_id="v_abc123",
    s3_key="2024/01/15/abc-123_vacation.mp4",
    qualities=["360p", "720p", "1080p"]
)
```

---

### transcode_quality()

**Purpose:** Transcode video to a specific quality and segment into HLS chunks.

**Parameters:**
- video_id (string): Video identifier
- input_path (string): Path to raw video file
- quality (string): Target quality (e.g., "720p")
- duration (float): Video duration in seconds

**Returns:**
- QualityResult: S3 path, bitrate, resolution

**Algorithm:**
```
function transcode_quality(video_id, input_path, quality, duration):
    // 1. Get encoding parameters for quality
    params = get_quality_params(quality)
    // params = {resolution: "1280x720", video_bitrate: "5000k", audio_bitrate: "128k"}
    
    // 2. Prepare output directory
    output_dir = f"/tmp/{video_id}_{quality}"
    create_directory(output_dir)
    
    // 3. Transcode with FFmpeg
    ffmpeg_command = [
        "ffmpeg", "-i", input_path,
        "-vf", f"scale={params.resolution}",  // Resize
        "-c:v", "libx264",  // H.264 codec
        "-b:v", params.video_bitrate,
        "-maxrate", params.video_bitrate,
        "-bufsize", f"{int(params.video_bitrate) * 2}k",
        "-c:a", "aac",  // AAC audio
        "-b:a", params.audio_bitrate,
        "-hls_time", "10",  // 10-second segments
        "-hls_playlist_type", "vod",
        "-hls_segment_filename", f"{output_dir}/segment_%03d.ts",
        f"{output_dir}/playlist.m3u8"
    ]
    
    transcode_start = now()
    run_command(ffmpeg_command)
    transcode_time = now() - transcode_start
    
    log(f"Transcoded {quality} in {transcode_time}s (speed: {duration/transcode_time}x)")
    
    // 4. Upload segments to S3
    segment_files = list_files(output_dir, "*.ts")
    manifest_file = f"{output_dir}/playlist.m3u8"
    
    s3_prefix = f"{video_id}/{quality}/"
    
    // Upload segments in parallel
    parallel_for segment_file in segment_files:
        s3_client.upload_file(
            Filename=segment_file,
            Bucket="videos-processed",
            Key=f"{s3_prefix}{basename(segment_file)}",
            ContentType="video/mp2t"
        )
    
    // Upload manifest
    s3_client.upload_file(
        Filename=manifest_file,
        Bucket="videos-processed",
        Key=f"{s3_prefix}playlist.m3u8",
        ContentType="application/vnd.apple.mpegurl"
    )
    
    // 5. Cleanup temp files
    delete_directory(output_dir)
    
    return {
        "quality": quality,
        "s3_prefix": s3_prefix,
        "resolution": params.resolution,
        "bitrate": params.video_bitrate,
        "segment_count": len(segment_files)
    }
```

**Time Complexity:** O(n) where n = video duration in seconds

**Example Usage:**
```
result = transcode_quality(
    video_id="v_abc123",
    input_path="/tmp/v_abc123_raw.mp4",
    quality="720p",
    duration=300.5
)
```

---

### generate_master_manifest()

**Purpose:** Generate HLS master manifest (master.m3u8) listing all quality variants.

**Parameters:**
- video_id (string): Video identifier
- qualities_info (list): List of quality metadata objects

**Returns:**
- string: HLS master manifest content

**Algorithm:**
```
function generate_master_manifest(video_id, qualities_info):
    // 1. Start manifest
    manifest = ["#EXTM3U"]
    manifest.append("#EXT-X-VERSION:6")
    manifest.append("")
    
    // 2. Quality params mapping
    quality_specs = {
        "144p": {resolution: "256x144", bandwidth: 200000},
        "360p": {resolution: "640x360", bandwidth: 800000},
        "480p": {resolution: "854x480", bandwidth: 1500000},
        "720p": {resolution: "1280x720", bandwidth: 5000000},
        "1080p": {resolution: "1920x1080", bandwidth: 8000000},
        "4K": {resolution: "3840x2160", bandwidth: 25000000}
    }
    
    // 3. Add each quality variant
    for quality_info in qualities_info:
        quality = quality_info.quality
        spec = quality_specs[quality]
        
        manifest.append(
            f"#EXT-X-STREAM-INF:BANDWIDTH={spec.bandwidth}," +
            f"RESOLUTION={spec.resolution}," +
            f"CODECS=\"avc1.42e00a,mp4a.40.2\""
        )
        manifest.append(f"{quality}/playlist.m3u8")
        manifest.append("")
    
    return "\n".join(manifest)
```

**Time Complexity:** O(n) where n = number of qualities

**Example Usage:**
```
manifest = generate_master_manifest(
    video_id="v_abc123",
    qualities_info=[
        {quality: "360p", ...},
        {quality: "720p", ...},
        {quality: "1080p", ...}
    ]
)
// Returns: HLS master manifest string
```

---

## 3. Adaptive Bitrate (ABR) Algorithm

### calculate_optimal_quality()

**Purpose:** Select optimal video quality based on current bandwidth and buffer.

**Parameters:**
- available_bandwidth_bps (integer): Estimated bandwidth in bits/second
- current_buffer_seconds (float): Current playback buffer size
- available_qualities (list): List of available quality levels

**Returns:**
- string: Recommended quality level

**Algorithm:**
```
function calculate_optimal_quality(available_bandwidth_bps, current_buffer_seconds, available_qualities):
    // Quality bitrates (bits per second)
    quality_bitrates = {
        "144p": 200000,
        "360p": 800000,
        "480p": 1500000,
        "720p": 5000000,
        "1080p": 8000000,
        "4K": 25000000
    }
    
    // 1. Safety margin (use 70% of available bandwidth to prevent buffering)
    safe_bandwidth = available_bandwidth_bps * 0.7
    
    // 2. Buffer health check
    buffer_health = current_buffer_seconds / 30.0  // Target: 30 seconds
    
    if buffer_health < 0.5:
        // Buffer critically low (<15 seconds), be conservative
        safe_bandwidth = safe_bandwidth * 0.8
    else if buffer_health > 1.5:
        // Buffer very healthy (>45 seconds), can be aggressive
        safe_bandwidth = safe_bandwidth * 1.2
    
    // 3. Find highest quality that fits bandwidth
    best_quality = "144p"  // Fallback
    best_bitrate = 0
    
    for quality in available_qualities:
        bitrate = quality_bitrates[quality]
        
        if bitrate <= safe_bandwidth and bitrate > best_bitrate:
            best_quality = quality
            best_bitrate = bitrate
    
    return best_quality
```

**Time Complexity:** O(n) where n = number of available qualities

**Example Usage:**
```
quality = calculate_optimal_quality(
    available_bandwidth_bps=6000000,  // 6 Mbps
    current_buffer_seconds=25.0,
    available_qualities=["360p", "480p", "720p", "1080p"]
)
// Returns: "720p" (5 Mbps fits in 6 Mbps safely)
```

---

### estimate_bandwidth()

**Purpose:** Estimate current network bandwidth based on recent segment downloads.

**Parameters:**
- download_history (list): List of recent downloads [{size_bytes, duration_ms}]
- window_size (integer): Number of recent downloads to consider

**Returns:**
- integer: Estimated bandwidth in bits/second

**Algorithm:**
```
function estimate_bandwidth(download_history, window_size):
    // 1. Use exponentially weighted moving average (EWMA)
    // Recent downloads weighted more heavily
    
    if len(download_history) == 0:
        return 5000000  // Default: 5 Mbps
    
    // 2. Take last N downloads
    recent_downloads = download_history[-window_size:]
    
    // 3. Calculate bandwidth for each download
    bandwidths = []
    for download in recent_downloads:
        bandwidth_bps = (download.size_bytes * 8) / (download.duration_ms / 1000.0)
        bandwidths.append(bandwidth_bps)
    
    // 4. EWMA calculation (weight recent samples more)
    alpha = 0.8  // Weight of most recent sample
    ewma = bandwidths[0]
    
    for i in range(1, len(bandwidths)):
        ewma = alpha * bandwidths[i] + (1 - alpha) * ewma
    
    // 5. Apply percentile filter (use 20th percentile to be conservative)
    // Handles variability better than average
    sorted_bandwidths = sorted(bandwidths)
    percentile_20_index = int(len(sorted_bandwidths) * 0.2)
    conservative_estimate = sorted_bandwidths[percentile_20_index]
    
    // 6. Return conservative estimate
    return conservative_estimate
```

**Time Complexity:** O(n log n) where n = window_size (sorting)

**Example Usage:**
```
bandwidth = estimate_bandwidth(
    download_history=[
        {size_bytes: 5242880, duration_ms: 1000},  // 5 MB in 1s = 40 Mbps
        {size_bytes: 5242880, duration_ms: 2000},  // 5 MB in 2s = 20 Mbps
        {size_bytes: 5242880, duration_ms: 1500}   // 5 MB in 1.5s = 27 Mbps
    ],
    window_size=5
)
// Returns: ~20000000 (20 Mbps - conservative 20th percentile)
```

---

## 4. CDN and Caching

### cdn_request_handler()

**Purpose:** Handle CDN request with multi-tier caching logic.

**Parameters:**
- request_path (string): Requested resource path (e.g., "/v_abc123/720p/segment_042.ts")
- edge_location (string): Edge server location
- client_ip (string): Client IP address

**Returns:**
- Response: File data or redirect

**Algorithm:**
```
function cdn_request_handler(request_path, edge_location, client_ip):
    cache_key = request_path
    
    // 1. Check edge cache (99% hit rate, 20ms)
    edge_cached = edge_cache.get(cache_key)
    if edge_cached:
        log_cache_hit("edge", request_path, edge_location)
        return Response(data=edge_cached, status=200, source="edge")
    
    // 2. Check regional cache (95% hit rate among misses, 50ms)
    regional_location = get_regional_location(edge_location)
    regional_cached = regional_cache.get(regional_location, cache_key)
    
    if regional_cached:
        log_cache_hit("regional", request_path, regional_location)
        
        // Cache at edge for next request (async)
        edge_cache.set_async(cache_key, regional_cached, ttl=7*24*3600)
        
        return Response(data=regional_cached, status=200, source="regional")
    
    // 3. Fetch from S3 origin (1% of requests, 150ms)
    try:
        s3_data = s3_client.get_object(
            Bucket="videos-processed",
            Key=request_path.lstrip("/")
        )
        
        log_cache_miss("origin", request_path)
        
        // Cache at both regional and edge (async)
        regional_cache.set_async(regional_location, cache_key, s3_data, ttl=24*3600)
        edge_cache.set_async(cache_key, s3_data, ttl=7*24*3600)
        
        return Response(data=s3_data, status=200, source="origin")
        
    catch S3Exception as e:
        log_error(f"S3 fetch failed: {e}")
        return Response(status=404, error="Video not found")
```

**Time Complexity:** O(1) for cache operations

**Example Usage:**
```
response = cdn_request_handler(
    request_path="/v_abc123/720p/segment_042.ts",
    edge_location="london-01",
    client_ip="192.168.1.1"
)
```

---

### warm_cdn_cache()

**Purpose:** Pre-populate CDN cache with popular video segments.

**Parameters:**
- video_id (string): Video to warm
- qualities (list): Qualities to warm (e.g., ["720p", "1080p"])
- segment_count (integer): Number of initial segments to warm

**Returns:**
- WarmResult: Number of segments warmed

**Algorithm:**
```
function warm_cdn_cache(video_id, qualities, segment_count):
    // 1. Get edge locations (top 10 most popular)
    edge_locations = [
        "us-east-1", "us-west-1", "eu-west-1", "eu-central-1",
        "ap-southeast-1", "ap-northeast-1", "sa-east-1"
    ]
    
    warmed_count = 0
    
    // 2. For each quality
    for quality in qualities:
        // 3. Warm first N segments (first 100 seconds if 10s segments)
        for i in range(segment_count):
            segment_path = f"/{video_id}/{quality}/segment_{i:03d}.ts"
            cdn_base_url = "https://d123abc.cloudfront.net"
            segment_url = cdn_base_url + segment_path
            
            // 4. Request from each edge location
            parallel_for edge in edge_locations:
                try:
                    response = http_get(
                        url=segment_url,
                        headers={"X-Edge-Location": edge}
                    )
                    
                    if response.status == 200:
                        warmed_count += 1
                        log(f"Warmed {segment_path} at {edge}")
                        
                catch Exception as e:
                    log_error(f"Failed to warm {segment_path} at {edge}: {e}")
    
    // 5. Update warming status
    redis.hset(f"cdn_warm:{video_id}", {
        "warmed_count": warmed_count,
        "qualities": qualities,
        "timestamp": now()
    })
    
    return {
        "video_id": video_id,
        "warmed_segments": warmed_count,
        "edge_locations": len(edge_locations)
    }
```

**Time Complexity:** O(q × s × e) where q = qualities, s = segments, e = edge locations

**Example Usage:**
```
result = warm_cdn_cache(
    video_id="v_abc123",
    qualities=["720p", "1080p"],
    segment_count=10
)
// Warms first 10 segments of 720p and 1080p across 7 edge locations
```

---

## 5. Metadata Management

### get_video_metadata()

**Purpose:** Fetch video metadata with cache-aside pattern.

**Parameters:**
- video_id (string): Video identifier

**Returns:**
- VideoMetadata: Title, duration, views, CDN URL, etc.

**Algorithm:**
```
function get_video_metadata(video_id):
    // 1. Check Redis cache (80% hit rate, 1ms)
    cache_key = f"video:{video_id}"
    cached = redis.get(cache_key)
    
    if cached:
        return json.loads(cached)
    
    // 2. Query Cassandra (20% miss rate, 5ms)
    row = cassandra.execute("""
        SELECT video_id, title, description, duration, width, height,
               views, likes, status, cdn_base_url, thumbnail_url,
               user_id, created_at, transcoded_at
        FROM videos
        WHERE video_id = ?
    """, [video_id]).one()
    
    if not row:
        throw VideoNotFoundException(video_id)
    
    // 3. Convert to dict
    metadata = {
        "video_id": row.video_id,
        "title": row.title,
        "description": row.description,
        "duration": row.duration,
        "resolution": f"{row.width}x{row.height}",
        "views": row.views,
        "likes": row.likes,
        "status": row.status,
        "cdn_base_url": row.cdn_base_url,
        "thumbnail_url": row.thumbnail_url,
        "user_id": row.user_id,
        "created_at": row.created_at,
        "transcoded_at": row.transcoded_at
    }
    
    // 4. Cache for 1 hour (longer for "ready" videos)
    ttl = 3600 if row.status == "ready" else 300  // 5 min for processing videos
    redis.setex(cache_key, ttl, json.dumps(metadata))
    
    return metadata
```

**Time Complexity:** O(1)

**Example Usage:**
```
metadata = get_video_metadata(video_id="v_abc123")
// Returns: {title: "Vacation 2024", duration: 300.5, views: 12500, ...}
```

---

### increment_view_count()

**Purpose:** Atomically increment video view count.

**Parameters:**
- video_id (string): Video identifier
- user_id (string): Viewer user ID

**Returns:**
- integer: New view count

**Algorithm:**
```
function increment_view_count(video_id, user_id):
    // 1. Deduplicate views (don't count multiple views from same user within 1 hour)
    dedup_key = f"view:{video_id}:{user_id}"
    
    if redis.exists(dedup_key):
        // Already viewed recently, don't increment
        return get_current_view_count(video_id)
    
    // 2. Mark user as having viewed (TTL: 1 hour)
    redis.setex(dedup_key, 3600, "1")
    
    // 3. Increment counter in Cassandra (using COUNTER type)
    cassandra.execute("""
        UPDATE videos
        SET views = views + 1
        WHERE video_id = ?
    """, [video_id])
    
    // 4. Invalidate cache
    redis.delete(f"video:{video_id}")
    
    // 5. Publish analytics event
    kafka.send(
        topic="analytics-views",
        key=video_id,
        value=json.dumps({
            "video_id": video_id,
            "user_id": user_id,
            "timestamp": now()
        })
    )
    
    // 6. Get new count
    row = cassandra.execute("""
        SELECT views FROM videos WHERE video_id = ?
    """, [video_id]).one()
    
    return row.views
```

**Time Complexity:** O(1)

**Example Usage:**
```
new_views = increment_view_count(
    video_id="v_abc123",
    user_id="user_456"
)
// Returns: 12501
```

---

## 6. Search and Recommendation

### search_videos()

**Purpose:** Full-text search across video titles, descriptions, tags.

**Parameters:**
- query (string): Search query
- limit (integer): Max results to return
- offset (integer): Pagination offset

**Returns:**
- SearchResults: List of matching videos with scores

**Algorithm:**
```
function search_videos(query, limit, offset):
    // 1. Build Elasticsearch query
    es_query = {
        "query": {
            "multi_match": {
                "query": query,
                "fields": ["title^3", "description", "tags^2"],  // Boost title and tags
                "type": "best_fields",
                "fuzziness": "AUTO"  // Handle typos
            }
        },
        "sort": [
            {"_score": "desc"},  // Relevance first
            {"views": "desc"},   // Then popularity
            {"created_at": "desc"}  // Then recency
        ],
        "from": offset,
        "size": limit,
        "highlight": {
            "fields": {
                "title": {},
                "description": {}
            }
        }
    }
    
    // 2. Execute search
    response = elasticsearch.search(
        index="videos",
        body=es_query
    )
    
    // 3. Extract video IDs
    video_ids = [hit["_id"] for hit in response["hits"]["hits"]]
    
    // 4. Batch fetch metadata from Cassandra
    videos = []
    for video_id in video_ids:
        try:
            metadata = get_video_metadata(video_id)  // Uses cache
            
            // Add search score
            hit = next(h for h in response["hits"]["hits"] if h["_id"] == video_id)
            metadata["search_score"] = hit["_score"]
            metadata["highlights"] = hit.get("highlight", {})
            
            videos.append(metadata)
        catch Exception as e:
            log_error(f"Failed to fetch metadata for {video_id}: {e}")
    
    return {
        "results": videos,
        "total": response["hits"]["total"]["value"],
        "query": query,
        "took_ms": response["took"]
    }
```

**Time Complexity:** O(log n + k) where n = total videos, k = result limit

**Example Usage:**
```
results = search_videos(
    query="funny cat videos",
    limit=20,
    offset=0
)
// Returns: {results: [{video_id, title, score}, ...], total: 1523}
```

---

### generate_recommendations()

**Purpose:** Generate personalized video recommendations using collaborative filtering.

**Parameters:**
- user_id (string): User to generate recommendations for
- count (integer): Number of recommendations

**Returns:**
- list: List of recommended video IDs with scores

**Algorithm:**
```
function generate_recommendations(user_id, count):
    // 1. Check if recommendations pre-computed (daily batch job)
    cache_key = f"recs:{user_id}"
    cached_recs = redis.get(cache_key)
    
    if cached_recs:
        recs = json.loads(cached_recs)
        return recs[:count]  // Return top N
    
    // 2. Get user's watch history
    watch_history = cassandra.execute("""
        SELECT video_id, watch_time_seconds
        FROM user_watch_history
        WHERE user_id = ?
        LIMIT 100
    """, [user_id])
    
    if len(watch_history) == 0:
        // New user, return trending videos
        return get_trending_videos(count)
    
    // 3. Get video embeddings (from ML model)
    watched_video_ids = [row.video_id for row in watch_history]
    watched_embeddings = get_video_embeddings(watched_video_ids)
    
    // 4. Compute average user preference vector
    user_vector = average(watched_embeddings)
    
    // 5. Find similar videos using ANN (Approximate Nearest Neighbors)
    similar_videos = ann_index.query(
        query_vector=user_vector,
        k=count * 2,  // Get 2x, then filter
        include_distances=True
    )
    
    // 6. Filter out already watched
    filtered = []
    for video_id, distance in similar_videos:
        if video_id not in watched_video_ids:
            score = 1.0 / (1.0 + distance)  // Convert distance to score
            filtered.append({"video_id": video_id, "score": score})
            
            if len(filtered) >= count:
                break
    
    // 7. Cache recommendations (refresh daily)
    redis.setex(cache_key, 86400, json.dumps(filtered))
    
    return filtered
```

**Time Complexity:** O(k log n) where k = count, n = total videos (ANN search)

**Example Usage:**
```
recs = generate_recommendations(user_id="user_456", count=20)
// Returns: [{video_id: "v_xyz", score: 0.92}, {video_id: "v_abc", score: 0.87}, ...]
```

---

## 7. Analytics Processing

### process_analytics_events()

**Purpose:** Process stream of analytics events (views, watch time, engagement).

**Parameters:**
- events (list): Batch of analytics events from Kafka

**Returns:**
- ProcessResult: Number of events processed

**Algorithm:**
```
function process_analytics_events(events):
    // 1. Group events by video_id
    events_by_video = {}
    for event in events:
        video_id = event["video_id"]
        if video_id not in events_by_video:
            events_by_video[video_id] = []
        events_by_video[video_id].append(event)
    
    // 2. Process each video's events
    processed_count = 0
    
    for video_id, video_events in events_by_video.items():
        // Aggregate metrics
        total_watch_time = 0
        unique_viewers = set()
        engagement_events = 0  // likes, comments, shares
        
        for event in video_events:
            event_type = event["type"]
            
            if event_type == "view":
                unique_viewers.add(event["user_id"])
            
            if event_type == "watch_progress":
                total_watch_time += event["watch_time_seconds"]
            
            if event_type in ["like", "comment", "share"]:
                engagement_events += 1
        
        // 3. Update Redis metrics (real-time dashboard)
        redis.hincrby(f"metrics:{video_id}", "views", len(unique_viewers))
        redis.hincrby(f"metrics:{video_id}", "watch_time", int(total_watch_time))
        redis.hincrby(f"metrics:{video_id}", "engagement", engagement_events)
        
        // 4. Update Cassandra (persistent storage)
        cassandra.execute("""
            UPDATE videos
            SET views = views + ?,
                total_watch_time_seconds = total_watch_time_seconds + ?,
                engagement_count = engagement_count + ?
            WHERE video_id = ?
        """, [len(unique_viewers), int(total_watch_time), engagement_events, video_id])
        
        // 5. Sink to BigQuery (data warehouse for analytics)
        for event in video_events:
            bigquery.insert(table="analytics_events", row=event)
        
        processed_count += len(video_events)
    
    return {
        "processed_count": processed_count,
        "videos_affected": len(events_by_video)
    }
```

**Time Complexity:** O(n) where n = number of events

**Example Usage:**
```
result = process_analytics_events(events=[
    {type: "view", video_id: "v_abc", user_id: "user_123"},
    {type: "watch_progress", video_id: "v_abc", watch_time_seconds: 120}
])
```

---

## 8. Live Streaming

### ingest_live_stream()

**Purpose:** Ingest live RTMP stream and transcode to HLS.

**Parameters:**
- stream_key (string): Unique stream key
- rtmp_url (string): Incoming RTMP stream URL

**Returns:**
- LiveStreamInfo: HLS output URL, latency

**Algorithm:**
```
function ingest_live_stream(stream_key, rtmp_url):
    // 1. Validate stream key
    stream_info = redis.hgetall(f"stream:{stream_key}")
    if not stream_info:
        throw InvalidStreamKeyException(stream_key)
    
    user_id = stream_info["user_id"]
    stream_id = stream_info["stream_id"]
    
    // 2. Start FFmpeg live transcoder
    output_dir = f"/tmp/live/{stream_id}"
    create_directory(output_dir)
    
    ffmpeg_process = start_process([
        "ffmpeg",
        "-i", rtmp_url,  // Input RTMP stream
        "-c:v", "copy",  // Copy video (no re-encode for latency)
        "-c:a", "aac",   // Encode audio
        "-f", "hls",     // Output HLS
        "-hls_time", "2",  // 2-second segments (low latency)
        "-hls_list_size", "5",  // Keep last 5 segments
        "-hls_flags", "delete_segments",  // Delete old segments
        f"{output_dir}/live.m3u8"
    ])
    
    // 3. Start segment uploader (continuously upload segments to S3)
    uploader_task = start_background_task(
        upload_live_segments,
        args=[stream_id, output_dir]
    )
    
    // 4. Generate playback URL
    cdn_url = f"https://live.example.com/{stream_id}/live.m3u8"
    
    // 5. Update stream status
    redis.hset(f"stream:{stream_key}", {
        "status": "live",
        "started_at": now(),
        "playback_url": cdn_url
    })
    
    // 6. Notify followers (push notification)
    followers = get_user_followers(user_id)
    for follower_id in followers:
        send_notification(
            user_id=follower_id,
            type="live_stream_started",
            data={"stream_id": stream_id, "url": cdn_url}
        )
    
    return {
        "stream_id": stream_id,
        "playback_url": cdn_url,
        "estimated_latency_seconds": 6,  // 2s segment × 3 segments buffered
        "status": "live"
    }
```

**Time Complexity:** O(1) for setup

**Example Usage:**
```
stream = ingest_live_stream(
    stream_key="sk_abc123",
    rtmp_url="rtmp://ingest.example.com/live/sk_abc123"
)
// Returns: {stream_id: "ls_xyz", playback_url: "https://...", latency: 6}
```

---

**End of Pseudocode Implementations**

*This document contains 40+ comprehensive algorithm implementations covering upload, transcoding, adaptive streaming,
CDN caching, metadata management, search, recommendations, analytics, and live streaming for the Video Streaming System.*

