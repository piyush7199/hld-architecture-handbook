# Distributed Monitoring System - Pseudocode Implementations

This document contains detailed algorithm implementations for the Distributed Monitoring System. The main challenge
document references these functions.

---

## Table of Contents

1. [Metrics Collection](#1-metrics-collection)
2. [Time-Series Encoding](#2-time-series-encoding)
3. [Downsampling and Rollups](#3-downsampling-and-rollups)
4. [Alerting Engine](#4-alerting-engine)
5. [Query Processing](#5-query-processing)
6. [Cardinality Management](#6-cardinality-management)
7. [Data Retention](#7-data-retention)
8. [Aggregation Functions](#8-aggregation-functions)

---

## 1. Metrics Collection

### scrape_endpoint()

**Purpose:** Scrape metrics from a target endpoint (Prometheus-style pull model).

**Parameters:**

- endpoint (string): Target URL to scrape (e.g., "http://server-123:9090/metrics")
- timeout (int): Scrape timeout in milliseconds (default: 10000)

**Returns:**

- metrics (array): Array of metric data points [{name, labels, value, timestamp}]

**Algorithm:**

```
function scrape_endpoint(endpoint, timeout=10000):
  try:
    // Send HTTP GET request with timeout
    response = http_get(endpoint, timeout=timeout, 
                       headers={"Accept": "text/plain"})
    
    if response.status_code != 200:
      log_error("Scrape failed", endpoint, response.status_code)
      return []
    
    // Parse Prometheus exposition format
    // Format: metric_name{label1="value1",label2="value2"} value timestamp
    metrics = []
    lines = response.body.split("\n")
    
    for line in lines:
      // Skip comments and empty lines
      if line.startsWith("#") or line.trim() == "":
        continue
      
      // Parse metric line
      metric = parse_prometheus_metric(line)
      
      if metric:
        // Add scrape metadata
        metric.scrape_time = now()
        metric.scrape_endpoint = endpoint
        metrics.append(metric)
    
    return metrics
    
  catch TimeoutException:
    log_error("Scrape timeout", endpoint)
    increment_counter("scrape_timeout_total", labels={endpoint: endpoint})
    return []
    
  catch Exception as e:
    log_error("Scrape exception", endpoint, e)
    increment_counter("scrape_error_total", labels={endpoint: endpoint})
    return []
```

**Time Complexity:** O(n) where n = number of metric lines

**Example Usage:**

```
metrics = scrape_endpoint("http://app-server-1:9090/metrics", timeout=5000)
// Returns: [
//   {name: "http_requests_total", labels: {method: "GET", status: "200"}, value: 1234, timestamp: 1698765432},
//   {name: "cpu_usage_percent", labels: {core: "0"}, value: 45.2, timestamp: 1698765432},
//   ...
// ]
```

---

### parse_prometheus_metric()

**Purpose:** Parse a single Prometheus metric line into structured format.

**Parameters:**

- line (string): Metric line (e.g., 'http_requests_total{method="GET"} 1234 1698765432')

**Returns:**

- metric (object): Parsed metric {name, labels, value, timestamp} or null if invalid

**Algorithm:**

```
function parse_prometheus_metric(line):
  // Regex pattern for Prometheus metric format
  // metric_name{label1="value1",label2="value2"} value timestamp
  
  // Split by whitespace
  parts = line.trim().split(/\s+/)
  
  if len(parts) < 2:
    return null  // Invalid format
  
  metric_part = parts[0]  // metric_name{labels}
  value = parseFloat(parts[1])
  timestamp = len(parts) >= 3 ? parseInt(parts[2]) : now()
  
  // Extract metric name and labels
  if "{" in metric_part:
    // Has labels
    name_end = metric_part.indexOf("{")
    metric_name = metric_part[0:name_end]
    
    labels_str = metric_part[name_end+1:-1]  // Remove { and }
    labels = parse_labels(labels_str)
  else:
    // No labels
    metric_name = metric_part
    labels = {}
  
  return {
    name: metric_name,
    labels: labels,
    value: value,
    timestamp: timestamp
  }
```

**Time Complexity:** O(m) where m = length of line

**Example Usage:**

```
line = 'http_requests_total{method="GET",status="200"} 1234 1698765432'
metric = parse_prometheus_metric(line)
// Returns: {
//   name: "http_requests_total",
//   labels: {method: "GET", status: "200"},
//   value: 1234,
//   timestamp: 1698765432
// }
```

---

### parse_labels()

**Purpose:** Parse Prometheus label string into key-value map.

**Parameters:**

- labels_str (string): Label string (e.g., 'method="GET",status="200"')

**Returns:**

- labels (object): Label map {key: value}

**Algorithm:**

```
function parse_labels(labels_str):
  labels = {}
  
  // Split by comma
  pairs = labels_str.split(",")
  
  for pair in pairs:
    // Split by =
    parts = pair.split("=")
    
    if len(parts) != 2:
      continue  // Invalid pair
    
    key = parts[0].trim()
    value = parts[1].trim()
    
    // Remove quotes from value
    if value.startsWith('"') and value.endsWith('"'):
      value = value[1:-1]
    
    labels[key] = value
  
  return labels
```

**Time Complexity:** O(n) where n = number of labels

**Example Usage:**

```
labels_str = 'method="GET",status="200",path="/api/users"'
labels = parse_labels(labels_str)
// Returns: {method: "GET", status: "200", path: "/api/users"}
```

---

## 2. Time-Series Encoding

### encode_time_series()

**Purpose:** Encode time-series data points using delta-of-delta encoding and compression.

**Parameters:**

- data_points (array): Array of data points [{timestamp, value}]

**Returns:**

- encoded_bytes (bytes): Compressed binary representation

**Algorithm:**

```
function encode_time_series(data_points):
  if len(data_points) == 0:
    return bytes()
  
  // Sort by timestamp (must be sequential)
  data_points = sort_by_timestamp(data_points)
  
  // Initialize bit stream
  bit_stream = BitStream()
  
  // Write first timestamp (full 64-bit)
  first_timestamp = data_points[0].timestamp
  bit_stream.write_int64(first_timestamp)
  
  // Write first value (full 64-bit float)
  first_value = data_points[0].value
  bit_stream.write_float64(first_value)
  
  // Delta-of-delta encoding for timestamps
  prev_timestamp = first_timestamp
  prev_delta = 0
  
  // XOR encoding for values
  prev_value_bits = float_to_bits(first_value)
  
  for i in range(1, len(data_points)):
    current_timestamp = data_points[i].timestamp
    current_value = data_points[i].value
    
    // Timestamp encoding (delta-of-delta)
    delta = current_timestamp - prev_timestamp
    delta_of_delta = delta - prev_delta
    
    if delta_of_delta == 0:
      // Most common case: regular interval
      bit_stream.write_bit(0)
    else if delta_of_delta >= -63 and delta_of_delta <= 64:
      // Small delta: 2 bits header + 7 bits value
      bit_stream.write_bits(2, 0b10)
      bit_stream.write_bits(7, delta_of_delta + 63)
    else if delta_of_delta >= -255 and delta_of_delta <= 256:
      // Medium delta: 3 bits header + 9 bits value
      bit_stream.write_bits(3, 0b110)
      bit_stream.write_bits(9, delta_of_delta + 255)
    else:
      // Large delta: 4 bits header + 32 bits value
      bit_stream.write_bits(4, 0b1110)
      bit_stream.write_int32(delta_of_delta)
    
    // Value encoding (XOR with previous)
    current_value_bits = float_to_bits(current_value)
    xor_value = current_value_bits XOR prev_value_bits
    
    if xor_value == 0:
      // Value unchanged
      bit_stream.write_bit(0)
    else:
      // Value changed
      bit_stream.write_bit(1)
      
      // Find leading and trailing zeros
      leading_zeros = count_leading_zeros(xor_value)
      trailing_zeros = count_trailing_zeros(xor_value)
      meaningful_bits = 64 - leading_zeros - trailing_zeros
      
      // Write leading zeros (5 bits)
      bit_stream.write_bits(5, leading_zeros)
      
      // Write meaningful bits length (6 bits)
      bit_stream.write_bits(6, meaningful_bits)
      
      // Write meaningful bits
      bit_stream.write_bits(meaningful_bits, 
                           xor_value >> trailing_zeros)
    
    prev_timestamp = current_timestamp
    prev_delta = delta
    prev_value_bits = current_value_bits
  
  // Compress bit stream using LZ4 or Snappy
  compressed = compress_lz4(bit_stream.to_bytes())
  
  return compressed
```

**Time Complexity:** O(n) where n = number of data points

**Example Usage:**

```
data_points = [
  {timestamp: 1698765432, value: 45.2},
  {timestamp: 1698765433, value: 45.3},
  {timestamp: 1698765434, value: 45.1},
  ...
]

encoded = encode_time_series(data_points)
// Returns: <binary compressed data>
// Compression ratio: ~20:1 for typical metrics (12x from encoding + 1.7x from LZ4)
```

---

## 3. Downsampling and Rollups

### compute_rollup()

**Purpose:** Compute downsampled aggregates (rollups) from raw time-series data.

**Parameters:**

- data_points (array): Array of raw data points [{timestamp, value}]
- rollup_interval (int): Rollup interval in seconds (e.g., 60 for 1-minute rollup)
- aggregation_function (string): Aggregation function ("avg", "sum", "min", "max", "count")

**Returns:**

- rollup_data (array): Array of rolled-up data points [{timestamp, value}]

**Algorithm:**

```
function compute_rollup(data_points, rollup_interval, aggregation_function):
  if len(data_points) == 0:
    return []
  
  // Sort by timestamp
  data_points = sort_by_timestamp(data_points)
  
  // Group by rollup windows
  rollup_buckets = {}
  
  for point in data_points:
    // Compute bucket timestamp (aligned to rollup_interval)
    bucket_timestamp = (point.timestamp // rollup_interval) * rollup_interval
    
    if bucket_timestamp not in rollup_buckets:
      rollup_buckets[bucket_timestamp] = []
    
    rollup_buckets[bucket_timestamp].append(point.value)
  
  // Compute aggregates for each bucket
  rollup_data = []
  
  for bucket_timestamp in sorted(rollup_buckets.keys()):
    values = rollup_buckets[bucket_timestamp]
    
    // Apply aggregation function
    if aggregation_function == "avg":
      agg_value = sum(values) / len(values)
    else if aggregation_function == "sum":
      agg_value = sum(values)
    else if aggregation_function == "min":
      agg_value = min(values)
    else if aggregation_function == "max":
      agg_value = max(values)
    else if aggregation_function == "count":
      agg_value = len(values)
    else:
      throw Error("Unknown aggregation function: " + aggregation_function)
    
    rollup_data.append({
      timestamp: bucket_timestamp,
      value: agg_value,
      count: len(values)  // Store count for debugging
    })
  
  return rollup_data
```

**Time Complexity:** O(n log n) where n = number of data points (due to sorting)

**Example Usage:**

```
// Raw data (1-second resolution)
raw_data = [
  {timestamp: 1698765432, value: 45.2},
  {timestamp: 1698765433, value: 45.3},
  {timestamp: 1698765434, value: 45.1},
  ...
  {timestamp: 1698765491, value: 46.0}
]

// Compute 1-minute rollup (avg)
rollup_1min = compute_rollup(raw_data, rollup_interval=60, aggregation_function="avg")
// Returns: [
//   {timestamp: 1698765420, value: 45.5, count: 60},
//   {timestamp: 1698765480, value: 46.2, count: 12}
// ]

// 60 data points reduced to 2 (30x compression)
```

---

## 4. Alerting Engine

### evaluate_alert_rule()

**Purpose:** Evaluate an alert rule against incoming metrics in real-time.

**Parameters:**

- rule (object): Alert rule {name, condition, threshold, duration, labels}
- metrics (array): Recent metric data points (sliding window)

**Returns:**

- alert (object): Alert object {firing: boolean, labels, annotations} or null

**Algorithm:**

```
function evaluate_alert_rule(rule, metrics):
  // Example rule:
  // {
  //   name: "HighCPU",
  //   condition: "cpu_usage > 80",
  //   duration: 60,  // Must be true for 60 seconds
  //   labels: {severity: "warning", team: "platform"}
  // }
  
  // Filter metrics matching rule's metric name
  metric_name = extract_metric_name(rule.condition)
  relevant_metrics = filter_by_name(metrics, metric_name)
  
  if len(relevant_metrics) == 0:
    return null  // No data
  
  // Sort by timestamp
  relevant_metrics = sort_by_timestamp(relevant_metrics)
  
  // Evaluate condition for each data point
  firing_start = null
  current_time = now()
  
  for point in relevant_metrics:
    condition_met = evaluate_condition(rule.condition, point.value)
    
    if condition_met:
      if firing_start == null:
        firing_start = point.timestamp
    else:
      firing_start = null  // Reset
  
  // Check if duration threshold is met
  if firing_start != null:
    firing_duration = current_time - firing_start
    
    if firing_duration >= rule.duration:
      // Alert should fire
      return {
        firing: true,
        rule_name: rule.name,
        labels: rule.labels,
        annotations: {
          summary: "Alert " + rule.name + " is firing",
          description: rule.condition + " for " + firing_duration + " seconds",
          current_value: relevant_metrics[-1].value
        },
        starts_at: firing_start,
        evaluated_at: current_time
      }
  
  // Alert not firing
  return {
    firing: false,
    rule_name: rule.name
  }
```

**Time Complexity:** O(n log n) where n = number of metrics in window

**Example Usage:**

```
rule = {
  name: "HighCPU",
  condition: "cpu_usage > 80",
  duration: 60,  // 60 seconds
  labels: {severity: "warning", team: "platform"}
}

metrics = [
  {timestamp: 1698765432, name: "cpu_usage", value: 85.0},
  {timestamp: 1698765433, name: "cpu_usage", value: 86.0},
  ...
  {timestamp: 1698765492, name: "cpu_usage", value: 87.0}  // 60 seconds of high CPU
]

alert = evaluate_alert_rule(rule, metrics)
// Returns: {
//   firing: true,
//   rule_name: "HighCPU",
//   labels: {severity: "warning", team: "platform"},
//   annotations: {
//     summary: "Alert HighCPU is firing",
//     description: "cpu_usage > 80 for 60 seconds",
//     current_value: 87.0
//   },
//   starts_at: 1698765432,
//   evaluated_at: 1698765492
// }
```

---

### evaluate_condition()

**Purpose:** Evaluate a simple condition expression against a value.

**Parameters:**

- condition (string): Condition expression (e.g., "cpu_usage > 80")
- value (float): Current metric value

**Returns:**

- result (boolean): True if condition is met

**Algorithm:**

```
function evaluate_condition(condition, value):
  // Parse condition: "metric_name operator threshold"
  // Supported operators: >, <, >=, <=, ==, !=
  
  parts = condition.split()
  
  if len(parts) != 3:
    throw Error("Invalid condition format: " + condition)
  
  metric_name = parts[0]
  operator = parts[1]
  threshold = parseFloat(parts[2])
  
  // Evaluate operator
  if operator == ">":
    return value > threshold
  else if operator == "<":
    return value < threshold
  else if operator == ">=":
    return value >= threshold
  else if operator == "<=":
    return value <= threshold
  else if operator == "==":
    return value == threshold
  else if operator == "!=":
    return value != threshold
  else:
    throw Error("Unknown operator: " + operator)
```

**Time Complexity:** O(1)

**Example Usage:**

```
result = evaluate_condition("cpu_usage > 80", 85.0)
// Returns: true

result = evaluate_condition("latency_ms < 100", 150.0)
// Returns: false
```

---

## 5. Query Processing

### execute_range_query()

**Purpose:** Execute a time-series range query with automatic rollup selection.

**Parameters:**

- metric_name (string): Metric to query
- labels (object): Label filters {key: value}
- start_time (int): Query start timestamp
- end_time (int): Query end timestamp

**Returns:**

- result (array): Time-series data [{timestamp, value}]

**Algorithm:**

```
function execute_range_query(metric_name, labels, start_time, end_time):
  // Calculate time range
  time_range = end_time - start_time
  
  // Select optimal rollup based on time range
  rollup_resolution = select_rollup_resolution(time_range)
  
  // Build storage query
  storage_query = {
    metric: metric_name,
    labels: labels,
    start: start_time,
    end: end_time,
    resolution: rollup_resolution
  }
  
  // Check cache
  cache_key = compute_cache_key(storage_query)
  cached_result = redis.get(cache_key)
  
  if cached_result:
    return deserialize(cached_result)
  
  // Query TSDB
  if rollup_resolution == "raw":
    result = query_raw_data(storage_query)
  else:
    // Query rollup table
    result = query_rollup_data(storage_query, rollup_resolution)
  
  // Cache result
  ttl = compute_cache_ttl(time_range)
  redis.setex(cache_key, ttl, serialize(result))
  
  return result
```

**Time Complexity:** O(log n) where n = total data points (due to time-based indexing)

**Example Usage:**

```
result = execute_range_query(
  metric_name="cpu_usage",
  labels={host: "server-123", region: "us-east-1"},
  start_time=1698765432,
  end_time=1698851832  // 1 day later
)
// Returns: [{timestamp: 1698765432, value: 45.2}, ...]
// Automatically uses 1-minute rollup (1,440 points instead of 86,400 raw)
```

---

### select_rollup_resolution()

**Purpose:** Select optimal rollup resolution based on query time range.

**Parameters:**

- time_range (int): Query time range in seconds

**Returns:**

- resolution (string): Selected resolution ("raw", "1min", "5min", "1hour", "1day")

**Algorithm:**

```
function select_rollup_resolution(time_range):
  // Optimization: Return ~2000-5000 data points for efficient visualization
  
  if time_range <= 3600:
    // Last 1 hour: use raw data (1-second resolution)
    return "raw"
  else if time_range <= 86400:
    // Last 1 day: use 1-minute rollup (1,440 points)
    return "1min"
  else if time_range <= 604800:
    // Last 7 days: use 5-minute rollup (2,016 points)
    return "5min"
  else if time_range <= 2592000:
    // Last 30 days: use 1-hour rollup (720 points)
    return "1hour"
  else:
    // >30 days: use 1-day rollup
    return "1day"
```

**Time Complexity:** O(1)

**Example Usage:**

```
resolution = select_rollup_resolution(86400)  // 1 day
// Returns: "1min"

resolution = select_rollup_resolution(2592000)  // 30 days
// Returns: "1hour"
```

---

## 6. Cardinality Management

### estimate_cardinality()

**Purpose:** Estimate cardinality (number of unique time series) for a metric using HyperLogLog.

**Parameters:**

- metric_name (string): Metric to analyze
- time_window (int): Time window in seconds (default: 3600 for last hour)

**Returns:**

- cardinality (int): Estimated number of unique time series

**Algorithm:**

```
function estimate_cardinality(metric_name, time_window=3600):
  // Use HyperLogLog for efficient cardinality estimation
  // Memory: ~1.5KB per metric (0.81% standard error)
  
  hll_key = "cardinality:" + metric_name
  
  // Get or create HyperLogLog counter
  hll = redis.hyperloglog_get(hll_key)
  
  if not hll:
    hll = HyperLogLog(precision=14)  // 2^14 = 16,384 registers
  
  // Get recent time series for this metric
  current_time = now()
  start_time = current_time - time_window
  
  time_series_list = tsdb.list_series(
    metric=metric_name,
    start=start_time,
    end=current_time
  )
  
  // Add each unique series to HyperLogLog
  for series in time_series_list:
    // Generate series ID from metric name + labels
    series_id = generate_series_id(series.metric, series.labels)
    hll.add(series_id)
  
  // Store HyperLogLog in Redis
  redis.hyperloglog_set(hll_key, hll, ttl=3600)
  
  // Return estimated cardinality
  return hll.count()
```

**Time Complexity:** O(n) where n = number of time series

**Example Usage:**

```
cardinality = estimate_cardinality("http_requests_total", time_window=3600)
// Returns: 125000 (estimated unique time series in last hour)

// Actual metric: http_requests_total{method, status, path, host}
// If 10 hosts × 5 methods × 5 statuses × 500 paths = 125,000 unique series
```

---

### detect_high_cardinality()

**Purpose:** Detect high-cardinality metrics that may cause performance issues.

**Parameters:**

- threshold (int): Cardinality threshold (default: 100000)

**Returns:**

- high_cardinality_metrics (array): List of metrics exceeding threshold

**Algorithm:**

```
function detect_high_cardinality(threshold=100000):
  high_cardinality_metrics = []
  
  // Get all metric names
  all_metrics = tsdb.list_all_metrics()
  
  for metric_name in all_metrics:
    // Estimate cardinality
    cardinality = estimate_cardinality(metric_name, time_window=3600)
    
    if cardinality > threshold:
      // Analyze label distribution
      label_distribution = analyze_label_distribution(metric_name)
      
      high_cardinality_metrics.append({
        metric: metric_name,
        cardinality: cardinality,
        label_distribution: label_distribution,
        severity: categorize_severity(cardinality)
      })
  
  // Sort by cardinality (descending)
  high_cardinality_metrics = sort_by_cardinality(high_cardinality_metrics, descending=true)
  
  return high_cardinality_metrics
```

**Time Complexity:** O(m × n) where m = number of metrics, n = avg time series per metric

**Example Usage:**

```
high_card = detect_high_cardinality(threshold=100000)
// Returns: [
//   {
//     metric: "http_requests_total",
//     cardinality: 500000,
//     label_distribution: {path: 450000, host: 10, method: 5, status: 5},
//     severity: "critical"
//   },
//   ...
// ]
```

---

## 7. Data Retention

### apply_retention_policy()

**Purpose:** Apply data retention policy to delete old data and rollups.

**Parameters:**

- retention_config (object): Retention configuration {raw: 7days, 1min: 30days, 1hour: 365days}

**Returns:**

- deleted_count (int): Number of data blocks deleted

**Algorithm:**

```
function apply_retention_policy(retention_config):
  current_time = now()
  deleted_count = 0
  
  // Process each resolution level
  for resolution, retention_days in retention_config:
    // Calculate cutoff timestamp
    retention_seconds = retention_days * 86400
    cutoff_timestamp = current_time - retention_seconds
    
    // Find data blocks older than cutoff
    if resolution == "raw":
      old_blocks = tsdb.find_raw_blocks(older_than=cutoff_timestamp)
    else:
      old_blocks = tsdb.find_rollup_blocks(
        resolution=resolution,
        older_than=cutoff_timestamp
      )
    
    // Delete old blocks
    for block in old_blocks:
      // Mark block for deletion (tombstone)
      tsdb.mark_deleted(block.id)
      deleted_count++
    
    log_info("Retention cleanup", 
            resolution=resolution,
            deleted_blocks=len(old_blocks),
            cutoff_timestamp=cutoff_timestamp)
  
  // Trigger compaction to reclaim disk space
  tsdb.trigger_compaction()
  
  return deleted_count
```

**Time Complexity:** O(b) where b = number of blocks to delete

**Example Usage:**

```
retention_config = {
  "raw": 7,      // Keep raw data for 7 days
  "1min": 30,    // Keep 1-minute rollups for 30 days
  "1hour": 365,  // Keep 1-hour rollups for 1 year
  "1day": 730    // Keep 1-day rollups for 2 years
}

deleted_count = apply_retention_policy(retention_config)
// Returns: 1523 (number of blocks deleted)
```

---

## 8. Aggregation Functions

### compute_percentile()

**Purpose:** Compute percentile (e.g., p50, p99) from a stream of values.

**Parameters:**

- values (array): Array of numeric values
- percentile (float): Percentile to compute (e.g., 50.0 for p50, 99.0 for p99)

**Returns:**

- result (float): Computed percentile value

**Algorithm:**

```
function compute_percentile(values, percentile):
  if len(values) == 0:
    return null
  
  // Sort values (required for exact percentile)
  sorted_values = sort(values)
  
  // Calculate index
  // percentile = 50 means 50th percentile (median)
  // Index = (percentile / 100) * (n - 1)
  
  n = len(sorted_values)
  index = (percentile / 100.0) * (n - 1)
  
  // Linear interpolation if index is not an integer
  lower_index = floor(index)
  upper_index = ceil(index)
  
  if lower_index == upper_index:
    return sorted_values[lower_index]
  
  // Interpolate
  lower_value = sorted_values[lower_index]
  upper_value = sorted_values[upper_index]
  fraction = index - lower_index
  
  result = lower_value + fraction * (upper_value - lower_value)
  
  return result
```

**Time Complexity:** O(n log n) due to sorting

**Example Usage:**

```
latencies = [10, 15, 20, 25, 30, 35, 40, 50, 100, 500]

p50 = compute_percentile(latencies, 50.0)
// Returns: 32.5 (median)

p99 = compute_percentile(latencies, 99.0)
// Returns: 500.0 (99th percentile)
```

---

### compute_rate()

**Purpose:** Compute per-second rate of change for a counter metric.

**Parameters:**

- data_points (array): Array of counter data points [{timestamp, value}]
- time_window (int): Time window for rate calculation (default: 60 seconds)

**Returns:**

- rate (float): Per-second rate

**Algorithm:**

```
function compute_rate(data_points, time_window=60):
  if len(data_points) < 2:
    return 0.0
  
  // Sort by timestamp
  data_points = sort_by_timestamp(data_points)
  
  // Get first and last points in time window
  current_time = now()
  window_start = current_time - time_window
  
  // Filter points within window
  windowed_points = []
  for point in data_points:
    if point.timestamp >= window_start:
      windowed_points.append(point)
  
  if len(windowed_points) < 2:
    return 0.0
  
  first_point = windowed_points[0]
  last_point = windowed_points[-1]
  
  // Calculate rate
  value_delta = last_point.value - first_point.value
  time_delta = last_point.timestamp - first_point.timestamp
  
  if time_delta == 0:
    return 0.0
  
  // Handle counter resets (value decreases)
  if value_delta < 0:
    // Assume counter wrapped around (64-bit counter)
    max_counter_value = 2^64 - 1
    value_delta = (max_counter_value - first_point.value) + last_point.value
  
  rate = value_delta / time_delta
  
  return rate
```

**Time Complexity:** O(n log n) due to sorting

**Example Usage:**

```
// Counter metric: http_requests_total
data_points = [
  {timestamp: 1698765432, value: 1000},
  {timestamp: 1698765433, value: 1005},
  {timestamp: 1698765434, value: 1012},
  ...
  {timestamp: 1698765492, value: 1300}
]

rate = compute_rate(data_points, time_window=60)
// Returns: 5.0 (300 requests in 60 seconds = 5 requests/sec)
```

---

## Summary

This pseudocode document provides detailed algorithm implementations for all major components of the Distributed
Monitoring System. Key algorithms include:

1. **Metrics Collection:** Prometheus-style scraping and parsing
2. **Time-Series Encoding:** Delta-of-delta and XOR encoding for compression
3. **Downsampling:** Rollup computation with various aggregation functions
4. **Alerting:** Real-time rule evaluation with duration thresholds
5. **Query Processing:** Automatic rollup selection and caching
6. **Cardinality Management:** HyperLogLog-based estimation
7. **Data Retention:** Automated cleanup with configurable policies
8. **Aggregation Functions:** Percentile and rate calculations

All functions are optimized for the scale requirements: 1M writes/sec, 63 trillion data points over 2 years, with
efficient compression (20:1 ratio) and query performance (<1 second for dashboard queries).

