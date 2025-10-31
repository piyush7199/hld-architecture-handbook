# Ad Click Aggregator - Pseudocode Implementations

This document contains detailed algorithm implementations for the Ad Click Aggregator system. The main challenge document references these functions.

---

## Table of Contents

1. [Event Ingestion](#1-event-ingestion)
2. [Fraud Detection](#2-fraud-detection)
3. [Stream Processing](#3-stream-processing)
4. [Real-Time Storage](#4-real-time-storage)
5. [Batch Processing](#5-batch-processing)
6. [Query Operations](#6-query-operations)

---

## 1. Event Ingestion

### api_gateway_handle_click()

**Purpose:** Accept click event via HTTP POST and buffer to Kafka.

**Parameters:**
- request (HttpRequest): HTTP POST request with click event JSON

**Returns:**
- HttpResponse: 202 Accepted (async response)

**Algorithm:**
```
function api_gateway_handle_click(request):
    // Extract and validate event
    event = parse_json(request.body)
    
    // Validate required fields
    if not validate_schema(event):
        return HttpResponse(400, "Invalid schema")
    
    // Rate limiting check
    client_ip = request.ip_address
    if rate_limiter.is_rate_limited(client_ip, limit=1000/sec):
        return HttpResponse(429, "Rate limit exceeded")
    
    // Enrich event
    event.server_timestamp = current_timestamp()
    event.geo = geoip_lookup(client_ip)
    
    // Buffer to Kafka producer (async)
    kafka_producer.send_async(
        topic="raw_clicks",
        key=event.campaign_id,  // Partition by campaign
        value=event,
        callback=log_kafka_error
    )
    
    // Return immediately (async)
    return HttpResponse(202, "Accepted")
```

**Time Complexity:** O(1)

**Example Usage:**
```
POST /click
Body: {"event_id": "uuid", "campaign_id": "123", "user_id": "user_456"}
Response: 202 Accepted
```

---

### validate_schema()

**Purpose:** Validate click event has all required fields.

**Parameters:**
- event (Dict): Click event data

**Returns:**
- bool: True if valid, False otherwise

**Algorithm:**
```
function validate_schema(event):
    required_fields = [
        "event_id",
        "timestamp",
        "campaign_id",
        "ad_id",
        "user_id"
    ]
    
    for field in required_fields:
        if field not in event:
            log_error("Missing field: " + field)
            return false
    
    // Validate timestamp within 5 minutes
    event_time = parse_timestamp(event.timestamp)
    server_time = current_timestamp()
    
    if abs(event_time - server_time) > 300_seconds:
        log_error("Timestamp out of range")
        return false
    
    return true
```

**Time Complexity:** O(1)

---

## 2. Fraud Detection

### fraud_detection_filter()

**Purpose:** Filter fraudulent clicks using Bloom filter and ML scoring.

**Parameters:**
- event (ClickEvent): Click event to check

**Returns:**
- FraudDecision: {is_fraud: bool, risk_score: float, reason: string}

**Algorithm:**
```
function fraud_detection_filter(event):
    // Stage 1: Bloom filter check (known bad actors)
    bad_actor_key = event.ip_address + ":" + event.user_agent
    
    if bloom_filter.contains(bad_actor_key):
        return FraudDecision(
            is_fraud=true,
            risk_score=1.0,
            reason="Known bad actor (Bloom filter)"
        )
    
    // Stage 2: Click pattern analysis
    pattern_score = analyze_click_pattern(event)
    
    if pattern_score > 0.7:
        return FraudDecision(
            is_fraud=true,
            risk_score=pattern_score,
            reason="Suspicious pattern"
        )
    
    // Stage 3: ML model scoring
    ml_score = ml_fraud_model.predict(event)
    
    if ml_score > 0.7:
        return FraudDecision(
            is_fraud=true,
            risk_score=ml_score,
            reason="ML model flagged"
        )
    
    // Clean event
    return FraudDecision(
        is_fraud=false,
        risk_score=ml_score,
        reason="Clean"
    )
```

**Time Complexity:** O(1) for Bloom filter, O(1) for ML inference

---

### analyze_click_pattern()

**Purpose:** Analyze click patterns to detect suspicious behavior.

**Parameters:**
- event (ClickEvent): Click event to analyze

**Returns:**
- float: Suspicion score (0.0-1.0)

**Algorithm:**
```
function analyze_click_pattern(event):
    suspicion_score = 0.0
    
    // Check 1: High frequency clicks from same IP
    ip_click_count = redis.get("click_count:" + event.ip_address + ":last_minute")
    
    if ip_click_count > 100:
        suspicion_score += 0.4
    else if ip_click_count > 50:
        suspicion_score += 0.2
    
    // Check 2: Sequential clicks (< 1 second apart)
    last_click_time = redis.get("last_click:" + event.user_id)
    
    if last_click_time != null:
        time_diff = event.timestamp - last_click_time
        if time_diff < 1_second:
            suspicion_score += 0.3
    
    // Check 3: Impossible geolocation
    if event.claimed_country != geoip_country(event.ip_address):
        suspicion_score += 0.2
    
    // Check 4: Known bot user-agent
    if is_bot_user_agent(event.user_agent):
        suspicion_score += 0.5
    
    // Update state
    redis.incr("click_count:" + event.ip_address + ":last_minute", ttl=60)
    redis.set("last_click:" + event.user_id, event.timestamp, ttl=60)
    
    return min(suspicion_score, 1.0)
```

**Time Complexity:** O(1)

---

### ml_fraud_scoring()

**Purpose:** Score event using ML model for fraud probability.

**Parameters:**
- event (ClickEvent): Event to score

**Returns:**
- float: Fraud probability (0.0-1.0)

**Algorithm:**
```
function ml_fraud_scoring(event):
    // Extract features
    features = extract_features(event)
    
    // Features include:
    // - IP address (hashed)
    // - User-agent (encoded)
    // - Time of day (0-23)
    // - Day of week (0-6)
    // - Referrer domain (categorical)
    // - Device type (categorical)
    // - Click velocity (clicks/min)
    
    // Random Forest model (pre-trained)
    fraud_probability = random_forest_model.predict_proba(features)
    
    return fraud_probability
```

**Time Complexity:** O(log n) for Random Forest with n trees

**Example:**
```
event = {ip: "1.2.3.4", user_agent: "Chrome", ...}
score = ml_fraud_scoring(event)
// score = 0.15 (low risk, clean)
```

---

## 3. Stream Processing

### windowed_aggregation()

**Purpose:** Aggregate click events in tumbling windows using Flink.

**Parameters:**
- event_stream (DataStream): Stream of click events
- window_size (Duration): Window size (e.g., 5 minutes)

**Returns:**
- DataStream<AggregatedStats>: Aggregated statistics per window

**Algorithm:**
```
function windowed_aggregation(event_stream, window_size):
    aggregated_stream = event_stream
        // Filter fraud
        .filter(event => !fraud_detection_filter(event).is_fraud)
        
        // Key by campaign_id and window
        .keyBy(event => event.campaign_id)
        
        // Tumbling window (5 minutes)
        .window(TumblingEventTimeWindows.of(window_size))
        
        // Allow late arrivals (5-minute grace period)
        .allowedLateness(5.minutes)
        
        // Aggregate function
        .aggregate(new AggregateFunction() {
            createAccumulator():
                return {
                    click_count: 0,
                    unique_users: Set(),
                    total_cost: 0.0,
                    by_country: Map(),
                    by_device: Map()
                }
            
            add(event, accumulator):
                accumulator.click_count += 1
                accumulator.unique_users.add(event.user_id)
                accumulator.total_cost += event.cost_per_click
                
                // Breakdown by dimension
                accumulator.by_country[event.country] += 1
                accumulator.by_device[event.device_type] += 1
                
                return accumulator
            
            getResult(accumulator):
                return AggregatedStats(
                    clicks=accumulator.click_count,
                    unique_users=accumulator.unique_users.size(),
                    total_cost=accumulator.total_cost,
                    by_country=accumulator.by_country,
                    by_device=accumulator.by_device
                )
            
            merge(acc1, acc2):
                return {
                    click_count: acc1.click_count + acc2.click_count,
                    unique_users: acc1.unique_users.union(acc2.unique_users),
                    total_cost: acc1.total_cost + acc2.total_cost,
                    by_country: merge_maps(acc1.by_country, acc2.by_country),
                    by_device: merge_maps(acc1.by_device, acc2.by_device)
                }
        })
    
    return aggregated_stream
```

**Time Complexity:** O(n) where n is events per window

---

### flink_checkpoint_state()

**Purpose:** Checkpoint Flink state for fault tolerance.

**Parameters:**
- state (KeyedState): Current state (RocksDB)
- checkpoint_id (Long): Checkpoint ID

**Returns:**
- bool: True if checkpoint successful

**Algorithm:**
```
function flink_checkpoint_state(state, checkpoint_id):
    try:
        // 1. Snapshot current state to RocksDB
        snapshot = state.create_snapshot()
        
        // 2. Upload incremental snapshot to S3
        s3_path = "s3://checkpoints/job-id/chk-" + checkpoint_id
        
        changed_keys = snapshot.get_changed_keys()  // Incremental
        for key in changed_keys:
            value = state.get(key)
            s3.put(s3_path + "/" + key, value)
        
        // 3. Commit Kafka offsets
        kafka_offsets = get_current_kafka_offsets()
        state.store("kafka_offsets", kafka_offsets)
        
        // 4. Mark checkpoint complete
        checkpoint_metadata = {
            checkpoint_id: checkpoint_id,
            timestamp: current_timestamp(),
            kafka_offsets: kafka_offsets,
            state_size: snapshot.size_bytes
        }
        
        s3.put(s3_path + "/metadata", checkpoint_metadata)
        
        log_info("Checkpoint " + checkpoint_id + " complete")
        return true
        
    catch error:
        log_error("Checkpoint failed: " + error)
        return false
```

**Time Complexity:** O(k) where k is number of changed keys

---

## 4. Real-Time Storage

### redis_update_counter()

**Purpose:** Update Redis counter for real-time dashboard.

**Parameters:**
- campaign_id (String): Campaign identifier
- window (String): Time window (e.g., "5min", "1hour")
- stats (AggregatedStats): Aggregated statistics

**Returns:**
- bool: True if update successful

**Algorithm:**
```
function redis_update_counter(campaign_id, window, stats):
    // Key pattern: counter:{campaign_id}:{window}:{dimension}
    base_key = "counter:" + campaign_id + ":" + window
    
    // Use Redis pipeline for batch writes
    pipeline = redis.pipeline()
    
    // Update total counters
    pipeline.hset(base_key + ":total", {
        "clicks": stats.clicks,
        "unique_users": stats.unique_users,
        "cost": stats.total_cost,
        "last_updated": current_timestamp()
    })
    
    // Set TTL (24 hours)
    pipeline.expire(base_key + ":total", 86400)
    
    // Update country breakdown
    for country, count in stats.by_country:
        country_key = base_key + ":" + country
        pipeline.hset(country_key, {
            "clicks": count
        })
        pipeline.expire(country_key, 86400)
    
    // Update device breakdown
    for device, count in stats.by_device:
        device_key = base_key + ":" + device
        pipeline.hset(device_key, {
            "clicks": count
        })
        pipeline.expire(device_key, 86400)
    
    // Execute pipeline (atomic batch)
    results = pipeline.execute()
    
    return all(results)  // True if all commands succeeded
```

**Time Complexity:** O(d) where d is number of dimensions

**Example:**
```
stats = {clicks: 1000, unique_users: 800, total_cost: 500.00}
redis_update_counter("campaign_123", "5min", stats)

// Redis keys created:
// counter:campaign_123:5min:total → {clicks: 1000, unique_users: 800, cost: 500.00}
// counter:campaign_123:5min:US → {clicks: 600}
// counter:campaign_123:5min:mobile → {clicks: 700}
```

---

## 5. Batch Processing

### batch_deduplication()

**Purpose:** Deduplicate events in batch processing using Spark.

**Parameters:**
- events_df (DataFrame): Spark DataFrame of events

**Returns:**
- DataFrame: Deduplicated events

**Algorithm:**
```
function batch_deduplication(events_df):
    // Group by event_id and keep first occurrence
    deduplicated = events_df
        .groupBy("event_id")
        .agg(
            first("timestamp").alias("timestamp"),
            first("campaign_id").alias("campaign_id"),
            first("ad_id").alias("ad_id"),
            first("user_id").alias("user_id"),
            first("cost_per_click").alias("cost_per_click"),
            count("*").alias("duplicate_count")
        )
        .filter("duplicate_count > 0")
    
    // Log duplicates
    duplicates = deduplicated.filter("duplicate_count > 1")
    
    if duplicates.count() > 0:
        log_warning("Found " + duplicates.count() + " duplicate events")
        duplicates.write.parquet("s3://logs/duplicates/" + date)
    
    // Return deduplicated events (drop duplicate_count column)
    return deduplicated.drop("duplicate_count")
```

**Time Complexity:** O(n log n) for groupBy

---

### batch_fraud_removal()

**Purpose:** Remove fraudulent events in batch using ML model.

**Parameters:**
- events_df (DataFrame): Deduplicated events

**Returns:**
- DataFrame: Clean events + fraud events (separate)

**Algorithm:**
```
function batch_fraud_removal(events_df):
    // Load ML model
    ml_model = load_model("s3://models/fraud_detection_v2.pkl")
    
    // Extract features for all events (vectorized)
    features_df = events_df
        .withColumn("features", extract_features_udf(
            col("ip_address"),
            col("user_agent"),
            col("timestamp"),
            col("referer")
        ))
    
    // Batch inference (100 events per batch)
    predictions_df = ml_model.transform(features_df)
    
    // Split clean vs fraud
    clean_events = predictions_df.filter("fraud_score < 0.7")
    fraud_events = predictions_df.filter("fraud_score >= 0.7")
    
    // Write fraud events to separate table
    fraud_events.write.parquet("s3://fraud/events/" + date)
    
    // Log fraud rate
    fraud_rate = fraud_events.count() / events_df.count()
    log_metric("fraud_rate", fraud_rate)
    
    return clean_events
```

**Time Complexity:** O(n) for inference

---

### batch_aggregate_billing()

**Purpose:** Aggregate clean events for billing database.

**Parameters:**
- clean_events (DataFrame): Clean, deduplicated events

**Returns:**
- DataFrame: Aggregated billing data

**Algorithm:**
```
function batch_aggregate_billing(clean_events):
    // Join with campaign metadata
    campaign_metadata = load_table("campaign_metadata")
    
    events_with_metadata = clean_events.join(
        campaign_metadata,
        on="campaign_id",
        how="left"
    )
    
    // Aggregate by campaign, date, and dimensions
    billing_data = events_with_metadata
        .groupBy(
            "campaign_id",
            "date",
            "country",
            "device_type"
        )
        .agg(
            count("*").alias("clicks"),
            countDistinct("user_id").alias("unique_users"),
            sum("cost_per_click").alias("total_cost"),
            min("timestamp").alias("first_click"),
            max("timestamp").alias("last_click")
        )
        .withColumn("date", to_date(col("timestamp")))
    
    return billing_data
```

**Time Complexity:** O(n log n) for groupBy + join

---

## 6. Query Operations

### dashboard_query_stats()

**Purpose:** Query Redis for dashboard statistics.

**Parameters:**
- campaign_id (String): Campaign to query
- window (String): Time window ("5min", "1hour", "1day")
- dimensions (List[String]): Breakdown dimensions (e.g., ["country", "device"])

**Returns:**
- Dict: Statistics with breakdowns

**Algorithm:**
```
function dashboard_query_stats(campaign_id, window, dimensions):
    base_key = "counter:" + campaign_id + ":" + window
    
    // Get total stats
    total_stats = redis.hgetall(base_key + ":total")
    
    if total_stats is null:
        // Cache miss - fallback to PostgreSQL
        return query_postgres_fallback(campaign_id, window)
    
    result = {
        "campaign_id": campaign_id,
        "window": window,
        "total": total_stats,
        "breakdowns": {}
    }
    
    // Get dimensional breakdowns
    for dimension in dimensions:
        // Use SCAN to get all keys for this dimension
        pattern = base_key + ":" + dimension + ":*"
        keys = redis.scan(pattern, count=100)
        
        // Use pipeline for batch GET
        pipeline = redis.pipeline()
        for key in keys:
            pipeline.hgetall(key)
        
        breakdown_data = pipeline.execute()
        
        result["breakdowns"][dimension] = breakdown_data
    
    return result
```

**Time Complexity:** O(d × k) where d is dimensions, k is keys per dimension

**Example:**
```
stats = dashboard_query_stats("campaign_123", "5min", ["country", "device"])

// Returns:
{
  "campaign_id": "campaign_123",
  "window": "5min",
  "total": {"clicks": 15234, "unique_users": 8901, "cost": 7617.00},
  "breakdowns": {
    "country": {
      "US": {"clicks": 8000},
      "UK": {"clicks": 3000},
      "CA": {"clicks": 2000}
    },
    "device": {
      "mobile": {"clicks": 10000},
      "desktop": {"clicks": 5000}
    }
  }
}
```

---

## Anti-Pattern Examples

### ❌ BAD: Synchronous Kafka Write

**Problem:** Blocks API response on Kafka confirmation.

```
function bad_api_handler(event):
    // BAD: Synchronous write
    kafka_producer.send_sync(event)  // Waits 50ms
    return HttpResponse(200, "OK")
```

### ✅ GOOD: Asynchronous Write

**Solution:** Return immediately, buffer and batch writes.

```
function good_api_handler(event):
    // GOOD: Asynchronous write
    kafka_producer.send_async(event)  // Returns immediately
    return HttpResponse(202, "Accepted")  // < 5ms
```

---

### ❌ BAD: No Fraud Detection

**Problem:** Count all clicks blindly.

```
function bad_aggregation(event):
    // BAD: No fraud check
    counter.increment(event.campaign_id, 1)
```

### ✅ GOOD: Multi-Stage Fraud Detection

**Solution:** Filter fraud before counting.

```
function good_aggregation(event):
    // GOOD: Fraud detection
    fraud_result = fraud_detection_filter(event)
    
    if not fraud_result.is_fraud:
        counter.increment(event.campaign_id, 1)
    else:
        fraud_db.log(event, fraud_result.reason)
```

---

These implementations provide production-ready algorithms for building an ad click aggregator that handles 500k events/sec with real-time analytics and accurate billing! 🚀

