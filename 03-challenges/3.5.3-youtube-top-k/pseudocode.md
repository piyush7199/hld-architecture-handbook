# YouTube Top K (Trending Algorithm) - Pseudocode Implementations

This document contains detailed algorithm implementations for the YouTube Top K trending system. The main challenge document references these functions.

---

## Table of Contents

1. [Event Ingestion](#1-event-ingestion)
2. [Fraud Detection](#2-fraud-detection)
3. [Stream Processing](#3-stream-processing)
4. [Trending Score Calculation](#4-trending-score-calculation)
5. [Top K Management](#5-top-k-management)
6. [Batch Processing](#6-batch-processing)
7. [Multi-Dimensional Ranking](#7-multi-dimensional-ranking)
8. [Utility Functions](#8-utility-functions)

---

## 1. Event Ingestion

### ingest_view_event()

**Purpose:** Validate, enrich, and publish view events to Kafka

**Parameters:**

- event (object): Raw view event from client
- api_context (object): Request context (IP, headers, timestamp)

**Returns:**

- response (object): {success: bool, error: string (if failed)}

**Algorithm:**

```
function ingest_view_event(event, api_context):
    // Step 1: Schema Validation
    required_fields = ["video_id", "user_id", "event_type"]
    for field in required_fields:
        if field not in event:
            return {success: false, error: "Missing required field: " + field}
    
    // Step 2: Rate Limiting (per user)
    user_key = "rate_limit:user:" + event.user_id
    current_count = redis.INCR(user_key)
    if current_count == 1:
        redis.EXPIRE(user_key, 60)  // 60 seconds window
    
    if current_count > 100:  // Max 100 views/min per user
        return {success: false, error: "Rate limit exceeded"}
    
    // Step 3: IP Rate Limiting
    ip_key = "rate_limit:ip:" + api_context.ip
    ip_count = redis.INCR(ip_key)
    if ip_count == 1:
        redis.EXPIRE(ip_key, 60)
    
    if ip_count > 500:  // Max 500 views/min per IP
        return {success: false, error: "IP rate limit exceeded"}
    
    // Step 4: Enrichment
    enriched_event = {
        ...event,
        event_id: generate_uuid(),
        ip_address: api_context.ip,
        country: geoip_lookup(api_context.ip),
        device_type: parse_user_agent(api_context.headers.user_agent),
        ingestion_timestamp: current_timestamp_millis(),
        server_timestamp: current_timestamp_iso8601()
    }
    
    // Step 5: Kafka Publish
    partition_key = hash(enriched_event.video_id) % KAFKA_NUM_PARTITIONS
    
    try:
        kafka_producer.send(
            topic="video_events",
            key=partition_key,
            value=json.stringify(enriched_event),
            compression="lz4",
            acks=1  // Leader acknowledgment only (fast)
        )
        return {success: true}
    catch KafkaTimeoutError as e:
        // Retry with exponential backoff
        return retry_with_backoff(enriched_event, max_retries=3)
```

**Time Complexity:** O(1) for all operations (hash lookups, Redis INCR)

**Example Usage:**

```
event = {
    video_id: "v123",
    user_id: "u456",
    event_type: "view",
    watch_duration_sec: 120
}

api_context = {
    ip: "1.2.3.4",
    headers: {user_agent: "Mozilla/5.0 ..."},
    timestamp: 1699012345678
}

result = ingest_view_event(event, api_context)
if result.success:
    log("Event ingested successfully")
else:
    log_error("Ingestion failed: " + result.error)
```

---

## 2. Fraud Detection

### fraud_detection_pipeline()

**Purpose:** Multi-stage fraud detection for view events

**Parameters:**

- event (object): View event to check
- bloom_filter (BloomFilter): Known bots/fraudulent IPs
- ml_model (Model): Fraud prediction model

**Returns:**

- result (object): {risk_score: float (0-1), action: string ("accept"|"flag"|"reject")}

**Algorithm:**

```
function fraud_detection_pipeline(event, bloom_filter, ml_model):
    // Stage 1: Bloom Filter (Known Bots)
    bot_key = event.user_id + ":" + event.ip_address
    if bloom_filter.contains(bot_key):
        log_fraud(event, "Known bot detected (Bloom filter)")
        return {risk_score: 1.0, action: "reject"}
    
    // Stage 2: Rate Pattern Analysis
    click_rate = get_user_click_rate(event.user_id, window_seconds=60)
    
    if click_rate > 100:  // > 100 views/min is suspicious
        log_fraud(event, "High frequency detected: " + click_rate + " views/min")
        risk_score = 0.8
        action = "flag"
    else:
        // Stage 3: ML Model Scoring
        features = extract_fraud_features(event)
        risk_score = ml_model.predict(features)
    
    // Stage 4: Decision Logic
    if risk_score < 0.3:
        return {risk_score: risk_score, action: "accept"}
    else if risk_score < 0.7:
        log_fraud(event, "Suspicious activity (risk: " + risk_score + ")")
        return {risk_score: risk_score, action: "flag"}
    else:
        log_fraud(event, "Fraud detected (risk: " + risk_score + ")")
        return {risk_score: risk_score, action: "reject"}
```

**Time Complexity:** O(1) + O(k) where k = ML model inference time (< 10ms)

---

### extract_fraud_features()

**Purpose:** Extract features for ML fraud detection

**Parameters:**

- event (object): View event

**Returns:**

- features (array): Feature vector for ML model

**Algorithm:**

```
function extract_fraud_features(event):
    features = []
    
    // Temporal features
    features.append(hour_of_day(event.timestamp))
    features.append(day_of_week(event.timestamp))
    
    // User behavior features
    user_history = get_user_history(event.user_id, last_n_days=7)
    features.append(len(user_history))  // Total views last 7 days
    features.append(unique_videos_count(user_history))  // Diversity
    features.append(average_watch_time(user_history))  // Engagement
    
    // IP reputation features
    ip_history = get_ip_history(event.ip_address, last_n_days=1)
    features.append(len(ip_history))  // Views from this IP
    features.append(unique_users_from_ip(ip_history))  // Shared IP?
    
    // Device features
    features.append(is_known_datacenter_ip(event.ip_address))  // 0 or 1
    features.append(is_tor_exit_node(event.ip_address))  // 0 or 1
    features.append(device_type_encoding(event.device_type))  // 0-3
    
    // Geographic features
    features.append(country_encoding(event.country))  // 0-200
    
    // Watch time features
    features.append(event.watch_duration_sec)
    video_length = get_video_length(event.video_id)
    features.append(event.watch_duration_sec / video_length)  // % watched
    
    return features
```

**Time Complexity:** O(1) assuming cached lookups

---

## 3. Stream Processing

### flink_aggregation_pipeline()

**Purpose:** Flink stream processing pipeline for real-time aggregation

**Parameters:**

- kafka_stream (DataStream): Input event stream from Kafka
- window_size_seconds (int): Tumbling window size (default: 60)

**Returns:**

- DataStream: Aggregated metrics stream

**Algorithm:**

```
function flink_aggregation_pipeline(kafka_stream, window_size_seconds=60):
    return kafka_stream
        // Step 1: Parse JSON events
        .map(event_string => parse_json(event_string))
        
        // Step 2: Fraud detection filter
        .filter(event => {
            result = fraud_detection_pipeline(event, bloom_filter, ml_model)
            
            if result.action == "reject":
                metrics.increment("fraud_rejected")
                return false  // Drop event
            else if result.action == "flag":
                metrics.increment("fraud_flagged")
                event.fraud_flag = true
                return true  // Keep but flag
            else:
                return true  // Accept
        })
        
        // Step 3: Key by video_id, country, category
        .keyBy(event => (event.video_id, event.country, event.category))
        
        // Step 4: Tumbling window aggregation
        .window(TumblingEventTimeWindows.of(Time.seconds(window_size_seconds)))
        
        // Step 5: Aggregate metrics
        .aggregate(new VideoMetricsAggregator())
        
        // Step 6: Write to InfluxDB and S3
        .addSink(new InfluxDBSink())
        .addSink(new S3Sink())
```

---

### VideoMetricsAggregator

**Purpose:** Aggregate function for Flink windowed aggregation

**Algorithm:**

```
class VideoMetricsAggregator extends AggregateFunction:
    function createAccumulator():
        return {
            view_count: 0,
            watch_time_total_sec: 0,
            like_count: 0,
            unique_users: new Set(),
            fraud_flagged_count: 0
        }
    
    function add(event, accumulator):
        accumulator.view_count += 1
        accumulator.watch_time_total_sec += event.watch_duration_sec
        
        if "like" in event.event_type:
            accumulator.like_count += 1
        
        accumulator.unique_users.add(event.user_id)
        
        if event.fraud_flag == true:
            accumulator.fraud_flagged_count += 1
        
        return accumulator
    
    function getResult(accumulator):
        return {
            view_count: accumulator.view_count,
            watch_time_total_sec: accumulator.watch_time_total_sec,
            like_count: accumulator.like_count,
            unique_user_count: accumulator.unique_users.size,
            fraud_flagged_count: accumulator.fraud_flagged_count,
            fraud_rate: accumulator.fraud_flagged_count / accumulator.view_count
        }
    
    function merge(acc1, acc2):
        return {
            view_count: acc1.view_count + acc2.view_count,
            watch_time_total_sec: acc1.watch_time_total_sec + acc2.watch_time_total_sec,
            like_count: acc1.like_count + acc2.like_count,
            unique_users: union(acc1.unique_users, acc2.unique_users),
            fraud_flagged_count: acc1.fraud_flagged_count + acc2.fraud_flagged_count
        }
```

**Time Complexity:** O(1) per event (amortized)

---

## 4. Trending Score Calculation

### calculate_trending_score()

**Purpose:** Calculate trending score with hyperbolic time decay

**Parameters:**

- video_id (string): Video identifier
- metrics (object): View counts, likes, watch time
- video_age_hours (float): Hours since video upload

**Returns:**

- float: Trending score

**Algorithm:**

```
function calculate_trending_score(video_id, metrics, video_age_hours):
    // Constants (tunable via config)
    ALPHA = 1.5          // Decay rate (higher = faster decay)
    T = 2                // Time constant (hours) - gives new videos boost
    BETA = 0.8           // Engagement scaling (diminishing returns)
    LIKE_WEIGHT = 10     // Likes weighted 10x views
    SHARE_WEIGHT = 20    // Shares weighted 20x views
    
    // Step 1: Calculate weighted engagement
    engagement = (
        metrics.view_count + 
        (metrics.like_count * LIKE_WEIGHT) + 
        (metrics.share_count * SHARE_WEIGHT)
    )
    
    // Step 2: Apply diminishing returns (power < 1)
    weighted_engagement = pow(engagement, BETA)
    
    // Step 3: Calculate time penalty (hyperbolic decay)
    // Age + T gives new videos boost (when age is small, denominator is small)
    time_penalty = pow(video_age_hours + T, ALPHA)
    
    // Step 4: Final score
    score = weighted_engagement / time_penalty
    
    // Step 5: Apply penalties/boosts (optional)
    if metrics.fraud_rate > 0.2:  // > 20% fraud
        score = score * 0.5  // 50% penalty
    
    if is_breaking_news(video_id):
        score = score * 1.5  // 50% boost for breaking news
    
    return score
```

**Time Complexity:** O(1)

**Example Usage:**

```
metrics = {
    view_count: 100000,
    like_count: 5000,
    share_count: 500,
    fraud_rate: 0.08
}

video_age_hours = 2.5  // Video uploaded 2.5 hours ago

score = calculate_trending_score("video_123", metrics, video_age_hours)
// score ≈ (100000 + 5000*10 + 500*20)^0.8 / (2.5 + 2)^1.5
// score ≈ (160000)^0.8 / (4.5)^1.5
// score ≈ 12589 / 9.55
// score ≈ 1318
```

---

### calculate_video_age_hours()

**Purpose:** Calculate video age in hours

**Parameters:**

- video_id (string): Video identifier
- current_time (timestamp): Current timestamp

**Returns:**

- float: Age in hours

**Algorithm:**

```
function calculate_video_age_hours(video_id, current_time):
    upload_time = get_video_upload_time(video_id)  // From metadata cache
    age_seconds = current_time - upload_time
    age_hours = age_seconds / 3600.0
    return age_hours
```

**Time Complexity:** O(1) assuming cached metadata

---

## 5. Top K Management

### update_top_k_redis()

**Purpose:** Update Top K list in Redis Sorted Set

**Parameters:**

- dimension (string): "global", "US", "Gaming", etc.
- video_scores (map): Map of video_id → score
- k (int): Top K size (default 100)

**Returns:**

- int: Number of videos updated

**Algorithm:**

```
function update_top_k_redis(dimension, video_scores, k=100):
    redis_key = "trending:" + dimension + ":top" + k
    
    // Use Redis pipeline for atomic batch operations
    pipeline = redis.pipeline()
    
    // Step 1: Update scores in sorted set
    for (video_id, score) in video_scores:
        pipeline.ZADD(redis_key, score, video_id)
    
    // Step 2: Get total count in ZSET
    pipeline.ZCARD(redis_key)
    
    // Execute to get count
    results = pipeline.execute()
    total_count = results[len(results) - 1]
    
    // Step 3: Keep only top K (remove lower-ranked)
    if total_count > k:
        // Remove rank 0 to (total_count - k - 1), keeping top k
        pipeline = redis.pipeline()
        pipeline.ZREMRANGEBYRANK(redis_key, 0, total_count - k - 1)
        
        // Step 4: Set expiration (24 hours)
        pipeline.EXPIRE(redis_key, 86400)
        
        pipeline.execute()
    
    return len(video_scores)
```

**Time Complexity:** O(m log N) where m = number of updates, N = ZSET size

---

### get_top_k_from_redis()

**Purpose:** Retrieve Top K list from Redis

**Parameters:**

- dimension (string): "global", "US", "Gaming", etc.
- k (int): Number of videos to retrieve (default 100)

**Returns:**

- list: [(video_id, score), ...] sorted by score descending

**Algorithm:**

```
function get_top_k_from_redis(dimension, k=100):
    redis_key = "trending:" + dimension + ":top" + k
    
    // Get top K with scores in descending order
    results = redis.ZREVRANGE(redis_key, 0, k-1, WITHSCORES=true)
    
    // Parse results [(video_id, score), ...]
    top_k = []
    for i in range(0, len(results), 2):
        video_id = results[i]
        score = float(results[i+1])
        top_k.append((video_id, score))
    
    return top_k
```

**Time Complexity:** O(log N + k) where N = total videos in ZSET

---

## 6. Batch Processing

### spark_batch_fraud_detection()

**Purpose:** Nightly Spark batch job for advanced fraud detection

**Parameters:**

- s3_path (string): Path to Parquet files
- date (string): Date to process (YYYY-MM-DD)

**Returns:**

- DataFrame: Clean events with fraud removed

**Algorithm:**

```
function spark_batch_fraud_detection(s3_path, date):
    // Step 1: Read S3 Parquet files
    events_df = spark.read.parquet(s3_path + "/date=" + date)
    
    // Step 2: Deduplication by event_id
    deduplicated_df = events_df.dropDuplicates(["event_id"])
    
    // Step 3: Extract features for ML model
    features_df = deduplicated_df.withColumn(
        "features",
        extract_fraud_features_udf(col("event"))
    )
    
    // Step 4: ML model scoring (XGBoost)
    ml_model = load_xgboost_model("s3://models/fraud_detection_v2.model")
    scored_df = ml_model.transform(features_df)
    
    // Step 5: Filter fraud (risk > 0.7)
    clean_df = scored_df.filter(col("risk_score") < 0.7)
    
    // Step 6: Mark suspicious (0.3 < risk < 0.7)
    clean_df = clean_df.withColumn(
        "fraud_flag",
        when(col("risk_score") > 0.3, true).otherwise(false)
    )
    
    return clean_df
```

**Time Complexity:** O(n) for n events (distributed across executors)

---

### spark_recalculate_trending_scores()

**Purpose:** Recalculate trending scores with clean data

**Parameters:**

- clean_events_df (DataFrame): Clean events from fraud detection
- current_time (timestamp): Current timestamp

**Returns:**

- DataFrame: Video scores

**Algorithm:**

```
function spark_recalculate_trending_scores(clean_events_df, current_time):
    // Step 1: Aggregate by video_id, date, country
    aggregated_df = clean_events_df.groupBy("video_id", "date", "country").agg(
        count("*").alias("view_count"),
        sum("watch_duration_sec").alias("total_watch_time"),
        countDistinct("user_id").alias("unique_users"),
        sum(when(col("event_type") == "like", 1).otherwise(0)).alias("like_count"),
        sum(when(col("event_type") == "share", 1).otherwise(0)).alias("share_count")
    )
    
    // Step 2: Join with video metadata (upload time)
    metadata_df = load_video_metadata()
    joined_df = aggregated_df.join(metadata_df, "video_id")
    
    // Step 3: Calculate trending score
    scored_df = joined_df.withColumn("video_age_hours",
        (current_time - col("upload_time")) / 3600.0
    ).withColumn("trending_score",
        calculate_trending_score_udf(
            col("video_id"),
            struct(col("view_count"), col("like_count"), col("share_count")),
            col("video_age_hours")
        )
    )
    
    return scored_df
```

---

## 7. Multi-Dimensional Ranking

### calculate_multi_dimensional_rankings()

**Purpose:** Calculate Top K for all dimensions in parallel

**Parameters:**

- video_scores_df (DataFrame): All videos with scores
- k (int): Top K size

**Returns:**

- map: {dimension: [(video_id, score), ...]}

**Algorithm:**

```
function calculate_multi_dimensional_rankings(video_scores_df, k=100):
    rankings = {}
    
    // Dimension 1: Global
    global_top_k = video_scores_df
        .orderBy(col("trending_score").desc())
        .limit(k)
        .select("video_id", "trending_score")
        .collect()
    rankings["global"] = global_top_k
    
    // Dimension 2: Regional (per country)
    countries = get_all_countries()  // 200 countries
    for country in countries:
        regional_top_k = video_scores_df
            .filter(col("country") == country)
            .orderBy(col("trending_score").desc())
            .limit(k)
            .select("video_id", "trending_score")
            .collect()
        rankings["regional:" + country] = regional_top_k
    
    // Dimension 3: Category
    categories = ["Gaming", "News", "Music", "Sports", ...]  // 20 categories
    for category in categories:
        category_top_k = video_scores_df
            .filter(col("category") == category)
            .orderBy(col("trending_score").desc())
            .limit(k)
            .select("video_id", "trending_score")
            .collect()
        rankings["category:" + category] = category_top_k
    
    return rankings
```

**Time Complexity:** O(n log k) per dimension, parallelized

---

## 8. Utility Functions

### bloom_filter_operations()

**Purpose:** Bloom filter operations for known bots

```
// Initialize Bloom filter
function create_bloom_filter(expected_items=1000000000, false_positive_rate=0.001):
    num_bits = calculate_optimal_bits(expected_items, false_positive_rate)
    num_hashes = calculate_optimal_hashes(expected_items, num_bits)
    return BloomFilter(num_bits, num_hashes)

// Add to Bloom filter
function bloom_add(bloom_filter, key):
    for i in range(bloom_filter.num_hashes):
        hash_val = hash_function_i(key, i)
        bit_index = hash_val % bloom_filter.num_bits
        bloom_filter.set_bit(bit_index, 1)

// Check Bloom filter
function bloom_contains(bloom_filter, key):
    for i in range(bloom_filter.num_hashes):
        hash_val = hash_function_i(key, i)
        bit_index = hash_val % bloom_filter.num_bits
        if bloom_filter.get_bit(bit_index) == 0:
            return false  // Definitely NOT in set
    return true  // Probably in set (possible false positive)
```

---

### Anti-Pattern Examples

### ❌ Bad: Real-Time Sorting

```
function get_top_k_bad(k):
    // DON'T: Query all videos and sort on every request
    all_videos = database.query("SELECT video_id, view_count FROM videos")
    
    scores = []
    for video in all_videos:
        score = calculate_trending_score(video)
        scores.append((video.id, score))
    
    scores.sort(reverse=True, key=lambda x: x[1])
    return scores[:k]

// Problem: O(n log n) where n = 1 billion videos
// Query time: MINUTES
// Database overload
```

### ✅ Good: Pre-Computed Top K

```
function get_top_k_good(dimension, k):
    // DO: Read pre-computed from Redis
    redis_key = "trending:" + dimension + ":top" + k
    return redis.ZREVRANGE(redis_key, 0, k-1, WITHSCORES=true)

// Time: O(log N + k) ≈ O(1) for k=100
// Query time: < 1ms
// No database load
```

---

**Note:** These implementations are production-ready patterns for trending algorithms at scale. Always benchmark and tune parameters for your specific workload. Consider caching, batching, and parallel processing for optimal performance.
