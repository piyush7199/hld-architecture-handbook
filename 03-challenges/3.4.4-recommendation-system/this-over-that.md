# Recommendation System - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made in the recommendation system design.
Each decision is examined with alternative approaches, trade-offs, and conditions for reconsideration.

---

## Table of Contents

1. [Lambda Architecture vs Kappa Architecture](#1-lambda-architecture-vs-kappa-architecture)
2. [Redis vs DynamoDB for Feature Store](#2-redis-vs-dynamodb-for-feature-store)
3. [TensorFlow Serving vs Custom gRPC Service](#3-tensorflow-serving-vs-custom-grpc-service)
4. [FAISS vs Elasticsearch for Similarity Search](#4-faiss-vs-elasticsearch-for-similarity-search)
5. [Batch Pre-Computation vs Real-Time Inference](#5-batch-pre-computation-vs-real-time-inference)
6. [Kafka Streams vs Flink for Real-Time Features](#6-kafka-streams-vs-flink-for-real-time-features)
7. [Collaborative Filtering (ALS) vs Deep Learning (Neural Networks)](#7-collaborative-filtering-als-vs-deep-learning-neural-networks)

---

## 1. Lambda Architecture vs Kappa Architecture

### The Problem

We need to provide accurate, personalized recommendations to 100M users with <50ms latency, while continuously learning
from user behavior. The challenge is balancing:

- **Accuracy:** Requires training on full historical data (petabytes)
- **Freshness:** Need to capture recent user actions (last 5 minutes)
- **Latency:** Must serve recommendations in <50ms
- **Cost:** Training large models is expensive

Two architectural approaches exist: **Lambda (batch + stream)** and **Kappa (stream-only)**.

---

### Options Considered

| Aspect                   | Lambda Architecture (Chosen)                    | Kappa Architecture (Alternative)          |
|--------------------------|-------------------------------------------------|-------------------------------------------|
| **Data Processing**      | Two pipelines: Batch (Spark) + Stream (Kafka)   | Single pipeline: Stream-only (Flink)      |
| **Training Data**        | Petabytes (full history, 5 years)               | Limited (90-day sliding window)           |
| **Model Accuracy**       | High (trained on full history)                  | Lower (limited lookback)                  |
| **Model Freshness**      | 24 hours (daily batch retrain)                  | 1-2 hours (continuous retraining)         |
| **System Complexity**    | High (two separate pipelines to maintain)       | Lower (single streaming pipeline)         |
| **Infrastructure Cost**  | Moderate (batch cluster runs once per day)      | Higher (streaming cluster runs 24/7)      |
| **Operational Overhead** | High (manage Spark + Kafka + Flink)             | Moderate (manage Flink only)              |
| **Use Case Fit**         | E-commerce, Netflix (slow-changing preferences) | News, Social Media (fast-changing trends) |

---

### Decision Made

**Lambda Architecture** (Batch layer for training + Speed layer for real-time features)

**Why Lambda?**

1. **Accuracy is Critical:** E-commerce and video streaming recommendations require high accuracy. Training on 5 years
   of data (petabytes) produces significantly better models than 90-day windows.
2. **Cost-Effective:** Batch training runs once per day for 20 hours. Kappa would require 24/7 streaming computation,
   costing 5x more.
3. **Proven at Scale:** Netflix, YouTube, Amazon all use Lambda Architecture for recommendations.
4. **Acceptable Freshness:** 24-hour model refresh is acceptable for e-commerce (user preferences change slowly).
   Real-time features (last 5 viewed items) capture immediate behavior.

---

### Rationale

**Why Training on Full Historical Data Matters:**

- **Long-Tail Items:** Rarely purchased items (e.g., niche books) only have enough signals across 5 years of data.
  90-day windows miss these patterns.
- **Seasonality:** Holiday shopping patterns, summer vs winter preferences require multi-year data to learn.
- **Cold Start:** New users benefit from global patterns learned across millions of historical users.

**Example:**

- User A last purchased a camping tent 2 years ago (summer vacation).
- Batch model (5 years): Learns User A is interested in outdoor gear.
- Stream model (90 days): No camping signals → misses outdoor recommendations.

**Why 24-Hour Freshness is Acceptable:**

- E-commerce: User preferences are stable (if you like laptops, you'll still like laptops tomorrow).
- Real-time features handle immediate behavior: "Last 5 viewed items" captured via Kafka Streams.
- Trending items handled separately: Pre-computed daily and boosted in ranking layer.

---

### Implementation Details

**Lambda Architecture Components:**

1. **Batch Layer (Slow Path):**
    - **Input:** 5 years of clickstream data from Cassandra/ClickHouse (9 PB)
    - **Processing:** Spark cluster reads 90-day sliding window (5 TB) daily
    - **Output:** Trained model (10 GB) → Model Store (S3/MLflow)
    - **Duration:** 20 hours (runs overnight)

2. **Speed Layer (Fast Path):**
    - **Input:** Real-time clickstream events from Kafka (100k events/sec)
    - **Processing:** Kafka Streams computes windowed aggregations (last 5 items, last action timestamp)
    - **Output:** Real-time features → Redis Feature Store (<5ms lookup)
    - **Duration:** Continuous (real-time)

3. **Serving Layer:**
    - Fetches user features from Redis (fast path) - 2ms
    - Loads model from TensorFlow Serving (slow path) - 20ms
    - Combines both to generate recommendations - <50ms total

**Code Flow:**

```
// Serving Layer (Recommendation Service)
function generate_recommendations(user_id):
  // Fast path: real-time features
  real_time_features = redis.get(f"user:{user_id}")  // 2ms
  // Example: {last_5_items: [1,2,3,4,5], last_action_ts: 1698765432}
  
  // Slow path: batch-trained model
  model = load_model_from_tfserving()  // 20ms
  
  // Combine: real-time features fed into batch model
  candidates = generate_candidates(user_id, real_time_features)
  scores = model.predict(candidates, real_time_features)
  
  return top_n(scores, n=10)
```

*See pseudocode.md::generate_recommendations() for full implementation.*

---

### Trade-offs Accepted

| What We Gain                                 | What We Sacrifice                              |
|----------------------------------------------|------------------------------------------------|
| ✅ High accuracy (trained on full history)    | ❌ Model freshness (24h lag)                    |
| ✅ Cost-effective (batch cluster runs once)   | ❌ System complexity (two pipelines)            |
| ✅ Scalability (batch and stream independent) | ❌ Operational overhead (Spark + Kafka + Flink) |
| ✅ Long-tail item coverage                    | ❌ Slower to adapt to breaking trends           |

**Accepted Risk:** Breaking trends (e.g., viral TikTok product) take 24 hours to appear in recommendations.

**Mitigation:** Real-time trending signals boost recent popular items in the ranking layer (no model retrain needed).

---

### When to Reconsider

**Switch to Kappa Architecture if:**

1. **User Preferences Change Rapidly:**
    - News recommendations (breaking news changes minute-by-minute)
    - Social media (trending topics shift hourly)
    - Live sports betting (odds change in real-time)

2. **Model Freshness is Business-Critical:**
    - Revenue directly tied to capturing immediate trends
    - 24-hour lag costs millions in lost sales

3. **Streaming ML Infrastructure Matures:**
    - Flink ML or River (Python streaming ML) becomes production-ready
    - Cost of 24/7 streaming drops significantly (serverless streaming)

4. **Dataset Size Becomes Unmanageable:**
    - If batch training takes >24 hours, daily refresh is impossible
    - Must switch to incremental stream-based training

**Example:** **TikTok** uses Kappa Architecture because video popularity changes hourly. A 24-hour model refresh would
miss viral content.

---

## 2. Redis vs DynamoDB for Feature Store

### The Problem

The Feature Store must provide <5ms latency for fetching user features (last 5 viewed items, preferred categories, user
embeddings) at 100k QPS. Two primary options: **Redis (in-memory)** or **DynamoDB (SSD-backed, serverless)**.

---

### Options Considered

| Aspect               | Redis (Chosen)                     | DynamoDB (Alternative)           |
|----------------------|------------------------------------|----------------------------------|
| **Latency**          | <2ms (in-memory)                   | 5-10ms (SSD-backed)              |
| **Throughput**       | 1M+ ops/sec (per cluster)          | Auto-scales (pay for throughput) |
| **Cost**             | Moderate ($5k/month for 50 shards) | Higher ($15k/month for 1M QPS)   |
| **Scalability**      | Manual sharding (Redis Cluster)    | Auto-scaling (serverless)        |
| **Persistence**      | Optional (AOF/RDB snapshots)       | Fully durable (replicated 3x)    |
| **Consistency**      | Eventual (async replication)       | Eventual (DynamoDB Streams)      |
| **Operational Load** | High (manage sharding, failover)   | Low (fully managed)              |
| **Multi-Region**     | Manual setup (Redis replication)   | Built-in (Global Tables)         |

---

### Decision Made

**Redis Cluster** (50 shards, 2 replicas per shard)

**Why Redis?**

1. **Latency is Critical:** <50ms total recommendation latency budget. Redis <2ms << DynamoDB 5-10ms. Every millisecond
   counts.
2. **Cost-Effective:** Redis is 3x cheaper than DynamoDB at 1M QPS.
3. **Predictable Performance:** In-memory lookups are consistent. DynamoDB latency spikes during throttling.
4. **Mature Ecosystem:** Battle-tested at scale (Twitter, Instagram, Pinterest use Redis for feeds).

---

### Rationale

**Latency Breakdown (50ms total budget):**

| Component             | Latency  | Percentage |
|-----------------------|----------|------------|
| API Gateway           | 5ms      | 10%        |
| Feature Fetch (Redis) | **2ms**  | **4%**     |
| Model Inference       | 20ms     | 40%        |
| Ranking               | 3ms      | 6%         |
| **Total**             | **30ms** | **60%**    |

**If using DynamoDB (5-10ms instead of 2ms):**

- Total latency: 33-38ms → P99 could breach 50ms SLA during spikes.

**Cost Comparison (1M reads/sec):**

**Redis:**

- 50 shards × 3 nodes (1 master + 2 replicas) = 150 nodes
- r6g.xlarge (32 GB RAM) @ $0.25/hour = $150/hour = $108k/month
- With Reserved Instances (60% discount) = $43k/month

**DynamoDB:**

- 1M reads/sec × 4 KB = 4 GB/sec
- On-Demand pricing: $1.25 per million reads = $1.25/sec = $3.25M/month
- Provisioned capacity (cheaper): 1M RCU @ $0.00013/hour = $94/hour = $68k/month

**Winner:** Redis is ~1.5x cheaper (with Reserved Instances).

---

### Implementation Details

**Redis Cluster Setup:**

```
// Sharding by user_id
function get_shard_id(user_id):
  return crc32(user_id) % 50  // 50 shards

// Feature lookup
function get_user_features(user_id):
  shard_id = get_shard_id(user_id)
  redis_client = get_redis_client(shard_id)
  
  // Fetch from read replica (2 replicas per shard)
  features = redis_client.get(f"user:{user_id}")
  
  if features is None:
    // Fallback to S3 (batch features)
    features = fetch_from_s3(user_id)
    redis_client.set(f"user:{user_id}", features, ttl=86400)
  
  return features
```

*See pseudocode.md::get_user_features() for full implementation.*

**Failover Strategy:**

- **Redis Sentinel:** Monitors master health (ping every 1 second)
- **Automatic Failover:** If master down, promote replica to master (<5 seconds)
- **Data Loss:** Minimal (replication lag <100ms → ~100 events lost, rebuilt from Kafka)

---

### Trade-offs Accepted

| What We Gain                    | What We Sacrifice                          |
|---------------------------------|--------------------------------------------|
| ✅ Ultra-low latency (<2ms)      | ❌ Manual sharding complexity               |
| ✅ Cost-effective (3x cheaper)   | ❌ Operational overhead (manage cluster)    |
| ✅ High throughput (1M+ ops/sec) | ❌ Eventual consistency (async replication) |
| ✅ Predictable performance       | ❌ Multi-region setup is manual             |

**Accepted Risk:** If Redis cluster fails, Feature Store is unavailable (single point of failure).

**Mitigation:**

- 99.99% availability (Redis Sentinel auto-failover)
- Fallback: Serve pre-computed recommendations from cache (no real-time features)

---

### When to Reconsider

**Switch to DynamoDB if:**

1. **Operational Complexity is Too High:**
    - Team lacks Redis expertise
    - Prefer fully managed solution (serverless)

2. **Multi-Region is Critical:**
    - Need active-active multi-region writes
    - DynamoDB Global Tables simplify cross-region replication

3. **Durability is Non-Negotiable:**
    - Cannot tolerate any data loss (even 100 events)
    - Redis AOF/RDB snapshots are not sufficient

4. **Cost is Not a Concern:**
    - Willing to pay 3x for operational simplicity

**Example:** **Shopify** uses DynamoDB for feature store because they prioritize operational simplicity over latency (
5-10ms is acceptable for their use case).

---

## 3. TensorFlow Serving vs Custom gRPC Service

### The Problem

We need to serve ML model inference at 100k QPS with <20ms latency. Two approaches: **TensorFlow Serving (managed
framework)** or **Custom gRPC Service (full control)**.

---

### Options Considered

| Aspect               | TensorFlow Serving (Chosen)      | Custom gRPC Service (Alternative)   |
|----------------------|----------------------------------|-------------------------------------|
| **Latency**          | 10-20ms (batching, GPU)          | 5-10ms (CPU, no framework overhead) |
| **Flexibility**      | Limited (TensorFlow models only) | Full control (any algorithm)        |
| **Ops Complexity**   | Lower (managed infrastructure)   | Higher (custom deployment)          |
| **Model Versioning** | Built-in (A/B testing support)   | Manual implementation               |
| **Monitoring**       | Built-in (Prometheus metrics)    | Custom instrumentation              |
| **GPU Support**      | Native (CUDA optimized)          | Requires custom integration         |
| **Batching**         | Automatic (dynamic batching)     | Manual implementation               |
| **Deployment**       | Docker + Kubernetes (standard)   | Custom build/deploy pipeline        |

---

### Decision Made

**TensorFlow Serving** (with gradual migration path to custom service if needed)

**Why TensorFlow Serving?**

1. **Operational Simplicity:** Built-in model versioning, A/B testing, monitoring. Custom service would require building
   all this from scratch (6+ months dev time).
2. **GPU Support:** Native CUDA optimization for GPU inference (4x faster). Custom service requires complex GPU
   integration.
3. **Batching:** Automatic dynamic batching (group 10-100 requests together → 10x throughput). Manual batching is
   error-prone.
4. **Industry Standard:** Used by Google, Uber, Airbnb. Proven at scale.

---

### Rationale

**Latency Analysis:**

| Component         | TensorFlow Serving | Custom Service |
|-------------------|--------------------|----------------|
| Model Load        | 10ms               | 5ms            |
| Batching Overhead | 5ms                | 0ms (no batch) |
| Inference         | 10ms               | 10ms           |
| Serialization     | 2ms                | 1ms            |
| **Total**         | **27ms**           | **16ms**       |

**Custom service is 10ms faster. But:**

- TensorFlow Serving provides automatic batching → 10x throughput (100k QPS vs 10k QPS)
- Custom service would need manual batching → complex, bug-prone

**Development Time:**

- TensorFlow Serving: 1 week to deploy (Docker + Kubernetes + config)
- Custom gRPC service: 6 months (build batching, versioning, monitoring, GPU support)

**When Latency Becomes Critical:**

- Current: 27ms inference + 2ms features + 5ms ranking = 34ms total (within 50ms SLA)
- If latency requirement drops to <30ms total → must migrate to custom service

---

### Implementation Details

**TensorFlow Serving Deployment:**

```
// Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tensorflow-serving
spec:
  replicas: 10
  template:
    spec:
      containers:
      - name: serving
        image: tensorflow/serving:2.13.0-gpu
        args:
          - --model_name=recommendation_model
          - --model_base_path=s3://models/recommendation/
          - --batching_parameters_file=/config/batching.txt
        resources:
          limits:
            nvidia.com/gpu: 1  // T4 GPU
```

**Batching Configuration:**

```
// /config/batching.txt
max_batch_size { value: 100 }
batch_timeout_micros { value: 5000 }  // 5ms
max_enqueued_batches { value: 100 }
num_batch_threads { value: 8 }
```

**Client-Side Inference:**

```
// Recommendation Service
function score_candidates(user_features, candidates):
  // Prepare gRPC request
  request = {
    "model_spec": {"name": "recommendation_model", "version": 42},
    "inputs": {
      "user_features": user_features,
      "item_features": [candidates[0].features, candidates[1].features, ...]
    }
  }
  
  // Call TensorFlow Serving via gRPC
  response = tf_serving_client.predict(request, timeout=50ms)
  
  // Extract scores
  scores = response["outputs"]["scores"]
  return scores
```

*See pseudocode.md::score_candidates() for full implementation.*

---

### Trade-offs Accepted

| What We Gain                          | What We Sacrifice                |
|---------------------------------------|----------------------------------|
| ✅ Operational simplicity              | ❌ 10ms higher latency vs custom  |
| ✅ Built-in A/B testing                | ❌ Limited to TensorFlow models   |
| ✅ GPU support (4x faster)             | ❌ Framework overhead             |
| ✅ Automatic batching (10x throughput) | ❌ Less control over optimization |

**Accepted Risk:** If TensorFlow Serving has a bug/vulnerability, we're blocked on TensorFlow team to fix it.

**Mitigation:**

- Pin to stable version (not latest)
- Monitor TensorFlow GitHub issues
- Plan migration path to custom service (fallback option)

---

### When to Reconsider

**Migrate to Custom gRPC Service if:**

1. **Latency Requirement Tightens:**
    - P99 must be <30ms total (currently 50ms)
    - 10ms savings from custom service becomes critical

2. **Model Complexity Increases:**
    - Need to serve non-TensorFlow models (PyTorch, XGBoost, custom algorithms)
    - TensorFlow Serving cannot support the model

3. **Cost Optimization:**
    - TensorFlow Serving GPU instances are expensive ($1k/month per instance)
    - Custom CPU-only service could be 10x cheaper

4. **Vendor Lock-In Concerns:**
    - Want to avoid dependency on TensorFlow ecosystem
    - Migrate to framework-agnostic serving

**Example:** **DoorDash** migrated from TensorFlow Serving to custom Rust-based service to reduce P99 latency from
50ms → 15ms (critical for real-time delivery).

---

## 4. FAISS vs Elasticsearch for Similarity Search

### The Problem

We need to find the top-50 similar items for item-to-item recommendations in <5ms from a catalog of 10 million items.
Two options: **FAISS (vector search)** or **Elasticsearch (full-text search with vector plugin)**.

---

### Options Considered

| Aspect            | FAISS (Chosen)                        | Elasticsearch (Alternative)               |
|-------------------|---------------------------------------|-------------------------------------------|
| **Query Latency** | <5ms (approximate nearest neighbor)   | 20-50ms (exact or approximate)            |
| **Index Size**    | 5 GB (10M items × 128 dims × 4 bytes) | 20 GB (includes metadata, inverted index) |
| **Accuracy**      | 95% recall@50 (HNSW algorithm)        | 100% (exact) or 90% (approximate)         |
| **Scalability**   | Horizontal (partition by category)    | Horizontal (sharding)                     |
| **Use Case**      | Pure vector similarity                | Hybrid (text + vector search)             |
| **Complexity**    | Low (single-purpose library)          | High (full search platform)               |
| **Cost**          | Low (runs in-memory on app servers)   | High (dedicated Elasticsearch cluster)    |

---

### Decision Made

**FAISS** (with HNSW algorithm for approximate nearest neighbor search)

**Why FAISS?**

1. **Speed:** <5ms << Elasticsearch 20-50ms. Speed is critical for <50ms total latency budget.
2. **Simplicity:** Single-purpose library. Elasticsearch is overkill for pure vector search.
3. **Cost:** Runs in-memory on existing application servers (no dedicated cluster needed).
4. **Accuracy:** 95% recall@50 is acceptable. The 5% missed items are typically very low-quality matches.

---

### Rationale

**Latency Breakdown:**

| Approach                  | Latency | Explanation                                    |
|---------------------------|---------|------------------------------------------------|
| **FAISS (HNSW)**          | **5ms** | Approximate search (95% recall)                |
| **Elasticsearch (Exact)** | 50ms    | Exact K-NN (100% recall, slow)                 |
| **Elasticsearch (ANN)**   | 20ms    | Approximate (90% recall, uses HNSW internally) |

**Why FAISS is Faster:**

- **In-Memory:** Index loaded into RAM (5 GB). Elasticsearch reads from disk (even with SSD).
- **Single-Purpose:** No query parsing, no aggregations, no scoring. Just vector distance computation.
- **Optimized Algorithms:** HNSW algorithm in C++ (hand-optimized by Facebook Research).

**Accuracy Trade-off:**

- **Exact K-NN:** Compute distance to all 10M items → O(N) = 10M operations → 50ms
- **HNSW (Approximate):** Navigate hierarchical graph → O(log N) = 23 hops → 5ms
- **Recall@50:** FAISS finds 47-48 of the true top-50 items (95% recall) → Acceptable for recommendations

**Example:**

- True top-50 similar items: [item_1, item_2, ..., item_50]
- FAISS top-50: [item_1, item_2, ..., item_47, item_999, item_888] (3 wrong items)
- Impact: User sees 47 relevant + 3 slightly less relevant items → Not noticeable

---

### Implementation Details

**FAISS Index Building (Offline, once per day):**

```
// Build FAISS index
function build_faiss_index(item_embeddings):
  // item_embeddings: 10M items × 128 dims
  
  // 1. Create HNSW index
  dimension = 128
  index = faiss.IndexHNSWFlat(dimension, 32)  // 32 = number of neighbors
  index.hnsw.efConstruction = 40  // Build-time accuracy
  
  // 2. Add all item embeddings
  index.add(item_embeddings)  // Takes ~1 hour for 10M items
  
  // 3. Save to disk
  faiss.write_index(index, "/data/faiss_index.bin")  // 5 GB file
  
  return index
```

**Query Time (Online, <5ms):**

```
// Find similar items
function find_similar_items(item_id, top_k=50):
  // 1. Fetch item embedding
  item_embedding = get_item_embedding(item_id)  // 128-dim vector
  
  // 2. FAISS search
  index.hnsw.efSearch = 50  // Query-time accuracy
  distances, item_ids = index.search(item_embedding, top_k)
  
  // 3. Return results
  return item_ids  // Top-50 similar items
```

*See pseudocode.md::find_similar_items() for full implementation.*

**Partitioning Strategy:**

- Split index by category (electronics, fashion, books)
- Each partition: 1M items, 500 MB
- Query only relevant partition → 10x faster

---

### Trade-offs Accepted

| What We Gain                 | What We Sacrifice                     |
|------------------------------|---------------------------------------|
| ✅ Ultra-low latency (<5ms)   | ❌ 5% accuracy loss (95% recall)       |
| ✅ Cost-effective (in-memory) | ❌ Cannot do hybrid text+vector search |
| ✅ Simple (single-purpose)    | ❌ Manual partitioning for scale       |
| ✅ High throughput (100k QPS) | ❌ Index rebuild takes 1 hour daily    |

**Accepted Risk:** 5% of recommended items are suboptimal (not true top-50).

**Mitigation:**

- The "wrong" items are still similar (just ranked 51-53 instead of 48-50) → Minimal impact
- Re-ranking layer downstream improves final quality

---

### When to Reconsider

**Switch to Elasticsearch if:**

1. **Need Hybrid Search:**
    - Combine vector similarity with text filters (e.g., "show similar laptops under $1000")
    - FAISS cannot filter by metadata

2. **Accuracy is Critical:**
    - Cannot tolerate 5% recall loss
    - Need exact K-NN (100% recall)

3. **Already Using Elasticsearch:**
    - Team has Elasticsearch expertise
    - Don't want to manage separate FAISS infrastructure

4. **Need Real-Time Index Updates:**
    - FAISS requires full index rebuild (1 hour)
    - Elasticsearch supports incremental updates

**Example:** **Spotify** uses Elasticsearch for music search because they need hybrid search (artist name + genre +
audio features). FAISS alone cannot support text filters.

---

## 5. Batch Pre-Computation vs Real-Time Inference

### The Problem

We need to serve 100k recommendations/sec with <50ms latency. Two strategies:

1. **Pre-Compute:** Generate recommendations for all users every 1 hour, cache in Redis (80% hit rate).
2. **Real-Time:** Compute recommendations on-demand for every request (100% fresh).

---

### Options Considered

| Aspect           | Pre-Computation (Chosen)            | Real-Time Inference (Alternative)    |
|------------------|-------------------------------------|--------------------------------------|
| **Latency**      | <2ms (cache hit)                    | 30-50ms (full inference)             |
| **Freshness**    | 1-hour stale (cache TTL)            | Real-time (0ms stale)                |
| **Throughput**   | 100k+ QPS (Redis)                   | 10k QPS (model inference limit)      |
| **Cost**         | Low (Redis cache)                   | High (10x more model servers)        |
| **Complexity**   | Moderate (background job + cache)   | Low (stateless service)              |
| **Failure Mode** | Graceful (serve stale if job fails) | Hard failure (no recs if model down) |

---

### Decision Made

**Hybrid Approach:**

- **80% of requests:** Pre-computed recommendations (cache hit)
- **20% of requests:** Real-time inference (cache miss)

**Why Hybrid?**

1. **Cost-Effective:** 80% cache hit rate reduces model inference load by 5x (10 TensorFlow Serving instances instead of
   50).
2. **Low Latency:** Cache hits return in <2ms (vs 30ms for full inference).
3. **Acceptable Freshness:** 1-hour staleness is fine for most users (preferences change slowly).
4. **Scalable:** Redis handles 1M+ QPS easily. Model inference is the bottleneck.

---

### Rationale

**Cost Comparison (100k QPS):**

**Real-Time Only:**

- 100k QPS × 30ms = 3000 concurrent requests
- TensorFlow Serving: 100 QPS per instance → Need 1000 instances
- Cost: 1000 × r5.2xlarge @ $0.50/hour = $500/hour = $360k/month

**Hybrid (80% Cache):**

- Cache hits: 80k QPS × <1ms Redis = negligible cost
- Cache misses: 20k QPS → Need 200 TensorFlow Serving instances
- Cost: 200 × $0.50/hour = $100/hour = $72k/month
- **Savings: $288k/month (80% reduction)**

**Freshness Trade-off:**

- User A's recommendations are 1 hour old (pre-computed at 2:00 PM, served at 2:45 PM).
- User A's last 5 viewed items (real-time features) are still fresh (from Redis Feature Store).
- Impact: Recommendations reflect behavior from 1 hour ago. Acceptable for e-commerce.

**When Staleness Matters:**

- Breaking news: User just viewed "iPhone 15 review" → Should immediately see iPhone accessories.
- Solution: Real-time features (last 5 items) are fed into pre-computed model → Partially fresh.

---

### Implementation Details

**Background Pre-Computation Job (Airflow DAG):**

```
// Runs every 1 hour
function precompute_recommendations_job():
  // 1. Identify users to refresh (50M active users)
  active_users = get_active_users_last_24h()  // 50M users
  
  // 2. Batch inference (parallel)
  for batch in chunks(active_users, batch_size=1000):
    // Fetch features
    features = fetch_features_batch(batch)
    
    // Model inference (batched)
    recommendations = model_serving.predict_batch(features)
    
    // Cache in Redis
    for user_id, recs in zip(batch, recommendations):
      redis.set(f"recs:{user_id}", recs, ttl=3600)  // 1 hour TTL
  
  // Total time: 1 hour (50M users × 30ms / 1000 batch / 100 parallel workers)
```

**Serving Path:**

```
function get_recommendations(user_id):
  // 1. Check cache
  cached = redis.get(f"recs:{user_id}")
  if cached:
    return cached  // Cache hit (80%)
  
  // 2. Cache miss: compute in real-time
  features = fetch_features(user_id)
  recommendations = model_serving.predict(features)
  
  // 3. Update cache (for next request)
  redis.set(f"recs:{user_id}", recommendations, ttl=3600)
  
  return recommendations
```

*See pseudocode.md::get_recommendations() for full implementation.*

---

### Trade-offs Accepted

| What We Gain                         | What We Sacrifice                  |
|--------------------------------------|------------------------------------|
| ✅ 80% cost reduction                 | ❌ 1-hour staleness                 |
| ✅ Ultra-low latency (<2ms)           | ❌ Background job complexity        |
| ✅ High throughput (1M QPS)           | ❌ Cache invalidation challenges    |
| ✅ Graceful degradation (serve stale) | ❌ Storage cost (50 GB Redis cache) |

**Accepted Risk:** User sees outdated recommendations if their preferences changed in the last hour.

**Mitigation:**

- Real-time features (last 5 items) partially compensate for staleness
- Trending items are updated separately (hourly boost in ranking layer)

---

### When to Reconsider

**Switch to Real-Time Only if:**

1. **Freshness is Critical:**
    - User behavior changes rapidly (news, live events, stock trading)
    - 1-hour staleness is unacceptable

2. **Cost is Not a Concern:**
    - Willing to pay 5x for real-time freshness

3. **Small User Base:**
    - <1M users → Pre-computation overhead is unnecessary
    - Real-time inference is cheap enough

4. **Personalization Depth Increases:**
    - Need to incorporate real-time context (current page, device, time of day)
    - Pre-computed recs cannot adapt to context

**Example:** **Bloomberg Terminal** uses real-time-only (no caching) because financial news recommendations must reflect
breaking events instantly (staleness is unacceptable).

---

## 6. Kafka Streams vs Flink for Real-Time Features

### The Problem

We need to compute real-time features (last 5 viewed items, last action timestamp) from a Kafka stream of 100k
events/sec with <10ms latency. Two options: **Kafka Streams** (library) or **Apache Flink** (framework).

---

### Options Considered

| Aspect               | Kafka Streams (Chosen)               | Apache Flink (Alternative)         |
|----------------------|--------------------------------------|------------------------------------|
| **Latency**          | <10ms (embedded in app)              | 50-100ms (network hop to cluster)  |
| **Throughput**       | 100k events/sec                      | 1M+ events/sec                     |
| **Deployment**       | Embedded (runs in JVM)               | Separate cluster (Kubernetes)      |
| **State Management** | RocksDB (local disk)                 | RocksDB (distributed)              |
| **Fault Tolerance**  | Exactly-once (Kafka transactions)    | Exactly-once (checkpointing)       |
| **Ops Complexity**   | Low (no separate cluster)            | High (manage Flink cluster)        |
| **Scalability**      | Horizontal (add more app instances)  | Horizontal (add more TaskManagers) |
| **Use Case**         | Simple transformations (filter, map) | Complex CEP, windowing, joins      |

---

### Decision Made

**Kafka Streams** (embedded in Recommendation Service)

**Why Kafka Streams?**

1. **Low Latency:** Embedded in app (no network hop). Flink requires network call to cluster → +50ms.
2. **Operational Simplicity:** No separate cluster to manage. Just deploy more app instances.
3. **Exactly-Once Semantics:** Built-in (Kafka transactions). Critical for feature correctness.
4. **Use Case Fit:** Simple stateful transformations (update last 5 items). No need for Flink's complex CEP.

---

### Rationale

**Latency Comparison:**

| Architecture      | Latency  | Components                                                |
|-------------------|----------|-----------------------------------------------------------|
| **Kafka Streams** | **10ms** | Event → Kafka → Kafka Streams (same JVM) → Redis          |
| **Flink**         | **60ms** | Event → Kafka → Network → Flink Cluster → Network → Redis |

**Why Flink is Slower:**

- **Network Overhead:** 2 network hops (Kafka → Flink, Flink → Redis) = +50ms
- **Serialization:** Extra serialization/deserialization at cluster boundary
- **Resource Contention:** Flink cluster shared by multiple jobs

**When Flink is Better:**

- Complex event processing (CEP): "Detect users who viewed 3 items in 10 minutes"
- Large windowed aggregations: "Compute hourly top-100 popular items"
- Multi-stream joins: "Join clickstream + purchase stream + inventory stream"

**Our Use Case is Simple:**

```
// Kafka Streams pseudocode
stream.groupByKey(user_id)
      .aggregate(
        initializer: {last_5_items: []},
        aggregator: (state, event) -> {
          state.last_5_items.append(event.item_id)
          state.last_5_items = state.last_5_items[-5:]  // Keep last 5
          return state
        }
      )
      .toStream()
      .foreach((user_id, state) -> redis.set(f"user:{user_id}", state))
```

No complex CEP, no joins. Kafka Streams is sufficient.

---

### Implementation Details

**Kafka Streams App:**

```
// Embedded in Recommendation Service
function start_kafka_streams():
  // 1. Configure
  config = {
    "application.id": "feature-store-updater",
    "bootstrap.servers": "kafka:9092",
    "processing.guarantee": "exactly_once",  // Critical
    "state.dir": "/tmp/kafka-streams-state"
  }
  
  // 2. Build topology
  builder = StreamsBuilder()
  
  stream = builder.stream("clickstream.events")
  
  // 3. Stateful aggregation
  aggregated = stream
    .groupByKey((event) -> event.user_id)
    .aggregate(
      initializer: UserState(),
      aggregator: update_user_state,  // See pseudocode.md
      materialized: Stores.persistent("user-state-store")
    )
  
  // 4. Sink to Redis
  aggregated.toStream().foreach((user_id, state) -> {
    redis.set(f"user:{user_id}", state, ttl=604800)  // 7 days
  })
  
  // 5. Start
  streams = KafkaStreams(builder.build(), config)
  streams.start()
```

*See pseudocode.md::start_kafka_streams() for full implementation.*

---

### Trade-offs Accepted

| What We Gain             | What We Sacrifice                   |
|--------------------------|-------------------------------------|
| ✅ Low latency (<10ms)    | ❌ Limited to simple transformations |
| ✅ Operational simplicity | ❌ Cannot do complex CEP             |
| ✅ Exactly-once semantics | ❌ State tied to app instances       |
| ✅ No separate cluster    | ❌ Scaling tied to app scaling       |

**Accepted Risk:** If Kafka Streams app crashes, state is lost (RocksDB on local disk).

**Mitigation:**

- State is rebuilt from Kafka log (exactly-once semantics)
- Worst case: 5-10 minutes to rebuild state from last checkpoint

---

### When to Reconsider

**Switch to Flink if:**

1. **Need Complex Event Processing:**
    - "Detect fraud: User viewed 10+ items in 1 minute without purchase"
    - Kafka Streams cannot express complex patterns

2. **Need Multi-Stream Joins:**
    - Join clickstream + purchase + inventory streams
    - Kafka Streams joins are limited

3. **Need Global Aggregations:**
    - "Top-100 popular items globally across all users"
    - Kafka Streams aggregates by key only

4. **Latency is Not Critical:**
    - Willing to accept 50-100ms for advanced features

**Example:** **Uber** uses Flink for fraud detection (complex CEP: "Detect driver who accepted 5 rides then canceled all
within 10 minutes"). Kafka Streams cannot handle this.

---

## 7. Collaborative Filtering (ALS) vs Deep Learning (Neural Networks)

### The Problem

We need to train a recommendation model on petabytes of clickstream data (5 years, 10M users, 10M items). Two
approaches: **Collaborative Filtering (ALS)** or **Deep Learning (Two-Tower Neural Network)**.

---

### Options Considered

| Aspect                  | ALS (Chosen for Batch)            | Neural Networks (Chosen for Online) |
|-------------------------|-----------------------------------|-------------------------------------|
| **Training Time**       | 6 hours (Spark MLlib)             | 4 hours (TensorFlow, GPU)           |
| **Model Size**          | 10 GB (user + item embeddings)    | 2 GB (dense weights)                |
| **Interpretability**    | High (embeddings have meaning)    | Low (black box)                     |
| **Cold Start**          | Poor (requires user-item matrix)  | Good (can use content features)     |
| **Scalability**         | Excellent (distributed via Spark) | Good (requires GPU cluster)         |
| **Accuracy**            | High (for implicit feedback)      | Very High (learns complex patterns) |
| **Feature Engineering** | Minimal (just user-item matrix)   | Extensive (need user/item features) |

---

### Decision Made

**Hybrid Approach:**

- **Batch Training (Offline):** ALS for user-item embeddings (6 hours, Spark)
- **Online Training (Real-Time Boost):** Two-tower neural network for contextual features (4 hours, TensorFlow)

**Why Hybrid?**

1. **ALS for Scale:** Collaborative filtering scales to petabytes with Spark. Neural networks require expensive GPU
   clusters.
2. **Neural Networks for Accuracy:** Deep learning captures complex patterns (e.g., sequential behavior, time decay).
   ALS is linear.
3. **Complementary:** ALS provides strong baseline embeddings. Neural network refines with contextual features.

---

### Rationale

**ALS (Alternating Least Squares) - Collaborative Filtering:**

- **Algorithm:** Factorize user-item matrix into user embeddings (U) and item embeddings (I).
- **Training:** Alternately optimize U (holding I fixed) and I (holding U fixed).
- **Output:** 100-dim embeddings for each user and item.
- **Scalability:** Distributed via Spark (10 machines, 6 hours for 10M × 10M matrix).

**Pros:**

- ✅ Proven at scale (Netflix Prize, Spotify)
- ✅ Interpretable (embeddings represent latent tastes: "comedy vs drama", "action vs romance")
- ✅ Fast training (6 hours << neural networks 24+ hours)

**Cons:**

- ❌ Linear model (cannot capture complex interactions)
- ❌ Cold start (new users have no embedding)

**Two-Tower Neural Network:**

- **Architecture:**
    - User Tower: Encode user features (demographics, history, context) → 128-dim vector
    - Item Tower: Encode item features (metadata, popularity, category) → 128-dim vector
    - Final Layer: Dot product(user vector, item vector) → predicted score
- **Training:** Supervised learning on user-item interactions (clicks, purchases).
- **Output:** Dense neural network weights (2 GB).

**Pros:**

- ✅ High accuracy (learns nonlinear patterns: "Users who view laptops at night buy accessories")
- ✅ Good cold start (uses content features, not just user-item matrix)
- ✅ Contextual (incorporates time, device, location)

**Cons:**

- ❌ Requires GPU cluster (expensive: $10k/month)
- ❌ Black box (hard to debug why item X was recommended)
- ❌ Overfitting risk (requires careful regularization)

---

### Implementation Details

**Hybrid Model Architecture:**

```
// Stage 1: ALS (Spark MLlib) - Collaborative Filtering
function train_als_model(user_item_interactions):
  // Input: 5 years of clicks/purchases (sparse matrix: 10M × 10M)
  
  // 1. Build user-item matrix
  matrix = build_sparse_matrix(user_item_interactions)
  
  // 2. Train ALS
  als = ALS(rank=100, maxIter=10, regParam=0.01, implicitPrefs=True)
  model = als.fit(matrix)
  
  // 3. Extract embeddings
  user_embeddings = model.userFactors  // 10M users × 100 dims
  item_embeddings = model.itemFactors  // 10M items × 100 dims
  
  return user_embeddings, item_embeddings

// Stage 2: Two-Tower Neural Network (TensorFlow)
function train_two_tower_model(user_features, item_features, interactions):
  // Input: user features, item features, labels (click/no-click)
  
  // 1. User Tower
  user_input = Input(shape=(num_user_features,))
  user_tower = Dense(256, activation='relu')(user_input)
  user_tower = Dense(128, activation='relu')(user_tower)
  user_embedding = Dense(128)(user_tower)  // 128-dim output
  
  // 2. Item Tower
  item_input = Input(shape=(num_item_features,))
  item_tower = Dense(256, activation='relu')(item_input)
  item_tower = Dense(128, activation='relu')(item_tower)
  item_embedding = Dense(128)(item_tower)  // 128-dim output
  
  // 3. Combine (dot product)
  score = Dot(axes=1)([user_embedding, item_embedding])
  score = Activation('sigmoid')(score)  // 0-1 probability
  
  // 4. Train
  model = Model(inputs=[user_input, item_input], outputs=score)
  model.compile(optimizer='adam', loss='binary_crossentropy')
  model.fit([user_features, item_features], interactions, epochs=10)
  
  return model
```

*See pseudocode.md::train_als_model() and pseudocode.md::train_two_tower_model() for full implementation.*

**Ensemble Strategy (Serving):**

```
// Combine ALS + Neural Network
function generate_recommendations(user_id):
  // 1. ALS score (baseline)
  user_embedding_als = fetch_user_embedding_als(user_id)
  als_scores = compute_cosine_similarity(user_embedding_als, all_item_embeddings_als)
  
  // 2. Neural network score (refinement)
  user_features = fetch_user_features(user_id)
  nn_scores = two_tower_model.predict(user_features, all_item_features)
  
  // 3. Blend (70% ALS, 30% Neural Network)
  final_scores = 0.7 * als_scores + 0.3 * nn_scores
  
  // 4. Return top-N
  return top_n(final_scores, n=10)
```

---

### Trade-offs Accepted

| What We Gain                        | What We Sacrifice                         |
|-------------------------------------|-------------------------------------------|
| ✅ High accuracy (hybrid model)      | ❌ Complex pipeline (two models)           |
| ✅ Scalability (ALS via Spark)       | ❌ Higher inference cost (two predictions) |
| ✅ Cold start (neural network)       | ❌ Hard to debug (two systems)             |
| ✅ Interpretability (ALS embeddings) | ❌ Longer training time (6h + 4h = 10h)    |

**Accepted Risk:** If one model fails (e.g., neural network training crashes), recommendations degrade (fall back to ALS
only).

**Mitigation:**

- ALS is the baseline (always available)
- Neural network is optional boost (can run with ALS only)

---

### When to Reconsider

**Use Neural Network Only if:**

1. **Cold Start is Critical:**
    - Majority of users are new (high churn, e.g., e-commerce flash sales)
    - ALS embeddings unavailable for most users

2. **Contextual Features are Essential:**
    - Time of day, device, location heavily impact recommendations
    - ALS cannot incorporate context

3. **Budget for GPUs:**
    - Willing to invest in GPU cluster for training and serving
    - Neural networks provide +5-10% accuracy boost

**Use ALS Only if:**

1. **Simplicity is Priority:**
    - Small team, limited ML expertise
    - ALS is easier to debug and maintain

2. **Budget is Tight:**
    - Cannot afford GPU cluster
    - ALS runs on commodity CPUs via Spark

**Example:** **Spotify** uses hybrid (ALS for audio embeddings + neural network for playlist continuation). ALS alone
was insufficient for sequential playlists.

---

## Summary: Key Design Decisions

| Decision                 | Chosen Approach               | Alternative           | Key Trade-Off                        |
|--------------------------|-------------------------------|-----------------------|--------------------------------------|
| **1. Architecture**      | Lambda (batch + stream)       | Kappa (stream-only)   | Accuracy vs Freshness                |
| **2. Feature Store**     | Redis (in-memory)             | DynamoDB (managed)    | Latency vs Operational Simplicity    |
| **3. Model Serving**     | TensorFlow Serving            | Custom gRPC           | Operational Simplicity vs Latency    |
| **4. Similarity Search** | FAISS (ANN)                   | Elasticsearch (exact) | Speed vs Accuracy                    |
| **5. Serving Strategy**  | Pre-Compute + Cache (80%)     | Real-Time Only        | Cost vs Freshness                    |
| **6. Stream Processing** | Kafka Streams                 | Flink                 | Simplicity vs Complex CEP            |
| **7. ML Algorithm**      | Hybrid (ALS + Neural Network) | Neural Network Only   | Scalability + Accuracy vs Complexity |

