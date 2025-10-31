# 3.4.4 Design a Recommendation System (Real-Time Personalization)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, feature store, model serving, and Lambda
  architecture
- **[Sequence Diagrams](./sequence-diagrams.md)** - Real-time serving flows, batch training, feature computation, and
  similarity search
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices and ML
  infrastructure
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations for recommendation logic

---

## Problem Statement

Design a highly scalable, low-latency recommendation system that provides personalized content (products, videos,
articles) to 100 million daily active users. The system must serve recommendations in under 50ms while continuously
learning from user behavior through both real-time features and offline batch training. Think Netflix's "Recommended For
You", Amazon's "Products You May Like", or TikTok's "For You Page".

---

## 1. Requirements and Scale Estimation

### Functional Requirements (FRs)

| Requirement                 | Description                                                          | Priority    |
|-----------------------------|----------------------------------------------------------------------|-------------|
| **Real-Time Serving**       | Generate personalized recommendations instantly (<50ms)              | Must Have   |
| **Offline Training**        | Train ML models on historical user behavior (clicks, views, ratings) | Must Have   |
| **Feature Store**           | Maintain real-time features (e.g., "last 5 items viewed")            | Must Have   |
| **Item-to-Item Similarity** | Recommend items similar to currently viewed item                     | Must Have   |
| **Multi-Strategy Blending** | Combine collaborative filtering, content-based, and popularity       | Must Have   |
| **A/B Testing**             | Support multiple recommendation strategies simultaneously            | Should Have |
| **Cold Start Handling**     | Serve recommendations for new users/items with no history            | Should Have |
| **Diversity & Exploration** | Balance relevance with discovery of new content                      | Should Have |

### Non-Functional Requirements (NFRs)

| Requirement           | Target             | Rationale                              |
|-----------------------|--------------------|----------------------------------------|
| **Low Latency**       | < 50 ms (p99)      | Cannot delay page load                 |
| **High Availability** | 99.99% uptime      | Core user experience feature           |
| **High Throughput**   | 100k+ requests/sec | Peak traffic during prime hours        |
| **Model Freshness**   | < 24 hours         | Capture evolving user preferences      |
| **Scalability**       | Petabytes of data  | Years of clickstream logs for training |

### Scale Estimation

| Metric                       | Assumption                                    | Calculation                                             | Result                                         |
|------------------------------|-----------------------------------------------|---------------------------------------------------------|------------------------------------------------|
| **Daily Active Users (DAU)** | 100 Million users                             | -                                                       | -                                              |
| **Recommendation QPS**       | 100M users × 10 sessions × 3 recs/session     | $\frac{3 \text{B}}{24 \text{h} \times 3600 \text{s/h}}$ | $\sim 35,000$ QPS (avg), $\sim 100,000$ (peak) |
| **Clickstream Events**       | 100M DAU × 50 events/day                      | -                                                       | $\sim 5$ Billion events per day                |
| **Training Data (5 years)**  | $5 \text{B events/day} \times 365 \times 5$   | $9.125 \text{T} \times 1 \text{kB/event}$               | $\sim 9$ Petabytes of historical data          |
| **Feature Store Lookups**    | 100k QPS × 1 user feature + 10 item features  | -                                                       | $\sim 1.1$ Million Redis lookups per second    |
| **Model Size**               | 10M users × 10M items × 100 dims (embeddings) | $10^{14} \times 4 \text{ bytes (float32)}$              | $\sim 400$ TB (sparse, compressed to ~10 TB)   |

---

## 2. High-Level Architecture

> 📊 **See detailed architecture diagrams:** [HLD Diagrams](./hld-diagram.md)

The system follows the **Lambda Architecture** pattern (see **2.3.5**), splitting into two independent loops:

1. **Slow Path (Batch Layer):** Offline training of large ML models on historical data using Hadoop/Spark.
2. **Fast Path (Speed Layer):** Real-time feature computation and low-latency serving using Kafka + Redis.

### Key Components

| Component                     | Responsibility                                  | Technology Options                 | Scalability              |
|-------------------------------|-------------------------------------------------|------------------------------------|--------------------------|
| **Clickstream Ingestion**     | Capture user events (views, clicks, buys)       | Kafka, Kinesis, Pulsar             | Horizontal (partitions)  |
| **Feature Store (Real-Time)** | Low-latency feature lookups (<5ms)              | Redis Cluster, DynamoDB            | Horizontal (sharding)    |
| **Feature Store (Offline)**   | Historical feature computation                  | Spark, Flink → S3/HDFS             | Horizontal (batch)       |
| **Batch Training Cluster**    | Train large ML models (collaborative filtering) | Hadoop, Spark MLlib, TensorFlow    | Horizontal (distributed) |
| **Model Store**               | Store trained model artifacts                   | S3, MinIO, Model Registry (MLflow) | Object storage           |
| **Model Serving API**         | Serve recommendations via API                   | TensorFlow Serving, gRPC, FastAPI  | Horizontal (stateless)   |
| **Similarity Index (ANN)**    | Fast nearest neighbor search for items          | FAISS, Annoy, HNSW, Milvus         | Horizontal (partitioned) |
| **Recommendation Service**    | Orchestrate feature fetch + model inference     | Go, Java, Python (stateless)       | Horizontal               |
| **Cache Layer**               | Cache popular recommendations                   | Redis, Memcached                   | Horizontal (sharding)    |

---

## 3. Detailed Component Design

### 3.1 Data Model and Storage

The system stores three critical types of data:

#### 3.1.1 User-Item Interaction Events (Clickstream)

Stored in **Kafka** (streaming buffer) → **Cassandra** or **ClickHouse** (historical store).

**Schema (Kafka Event):**

| Field        | Type    | Notes                                 |
|--------------|---------|---------------------------------------|
| `user_id`    | `INT64` | Unique user identifier                |
| `item_id`    | `INT64` | Unique item identifier                |
| `event_type` | `ENUM`  | `VIEW`, `CLICK`, `PURCHASE`, `RATING` |
| `timestamp`  | `INT64` | Unix timestamp (ms)                   |
| `session_id` | `UUID`  | Session identifier                    |
| `context`    | `JSON`  | Device, location, referrer            |

*See pseudocode.md::ClickstreamEvent for full schema definition.*

#### 3.1.2 User Features (Real-Time + Batch)

Stored in **Redis** (real-time) + **S3** (batch snapshots).

**Real-Time Features (Redis):**

| Key              | Value (JSON)                                                      | TTL    |
|------------------|-------------------------------------------------------------------|--------|
| `user:{user_id}` | `{"last_5_items": [123, 456, ...], "last_action_ts": 1698765432}` | 7 days |

**Batch Features (S3 Parquet):**

| Field                  | Type           | Notes                            |
|------------------------|----------------|----------------------------------|
| `user_id`              | `INT64`        | Primary key                      |
| `total_purchases`      | `INT32`        | Lifetime purchase count          |
| `avg_session_duration` | `FLOAT32`      | Average session length (seconds) |
| `preferred_categories` | `ARRAY<INT>`   | Top 5 category IDs by engagement |
| `user_embedding`       | `ARRAY<FLOAT>` | 128-dim learned vector           |

*See pseudocode.md::UserFeatures for feature engineering logic.*

#### 3.1.3 Item Features and Embeddings

Stored in **PostgreSQL** (metadata) + **FAISS/Milvus** (vector index).

**Item Metadata (PostgreSQL):**

| Field         | Type        | Notes                    |
|---------------|-------------|--------------------------|
| `item_id`     | `BIGINT`    | Primary key              |
| `category_id` | `INT`       | Category classification  |
| `created_at`  | `TIMESTAMP` | Item creation time       |
| `popularity`  | `FLOAT`     | Global popularity score  |
| `metadata`    | `JSONB`     | Title, description, tags |

**Item Embeddings (FAISS Index):**

- **128-dimensional vectors** learned via collaborative filtering or content-based models.
- **HNSW algorithm** for approximate nearest neighbor (ANN) search in <5ms.

*See pseudocode.md::ItemEmbedding for embedding generation.*

---

### 3.2 The Lambda Architecture (Batch + Speed Layers)

#### Why Lambda Architecture?

| Aspect          | Batch Layer (Slow Path)        | Speed Layer (Fast Path)        |
|-----------------|--------------------------------|--------------------------------|
| **Purpose**     | Train accurate, complex models | Capture real-time user signals |
| **Latency**     | Hours (24h model refresh)      | Milliseconds (<50ms serving)   |
| **Data Volume** | Petabytes (historical)         | GB/s (streaming)               |
| **Technology**  | Hadoop/Spark                   | Kafka + Redis + Flink          |
| **Output**      | Static trained model           | Real-time features             |

*See this-over-that.md for detailed comparison with Kappa Architecture.*

---

### 3.3 Recommendation Serving Flow (Real-Time)

> 📊 **See detailed sequence diagrams:** [Sequence Diagrams](./sequence-diagrams.md)

**Step-by-Step Flow (Target: <50ms end-to-end):**

1. **Client Request** → API Gateway → Recommendation Service
    - Input: `user_id`, `context` (current page, device)
    - *Latency Budget: 0ms*

2. **Feature Fetch (Parallel gRPC Calls)**
    - **User Features:** Fetch from Redis Feature Store (`user:{user_id}`)
        - Returns: Last 5 viewed items, last action timestamp
        - *Latency: 2ms*
    - **Item Candidate Fetch:** Query Similarity Index (FAISS) for item-to-item recs
        - Returns: Top 100 candidate items
        - *Latency: 5ms*
    - *Total Parallel Latency: 5ms*

3. **Model Inference**
    - Load model from Model Serving API (TensorFlow Serving/gRPC)
    - Compute scores for candidate items using user features + item embeddings
    - *Latency: 20ms*

4. **Ranking & Filtering**
    - Apply business rules (remove already purchased, diversify categories)
    - Re-rank top 20 items
    - *Latency: 3ms*

5. **Response to Client**
    - Return top 10 recommendations
    - *Total Latency: ~30ms (within 50ms budget)*

*See pseudocode.md::generate_recommendations() for detailed algorithm.*

---

### 3.4 Candidate Generation Strategies

The system uses a **multi-strategy approach** to generate candidate items:

| Strategy                    | Description                                            | Candidate Count | Latency |
|-----------------------------|--------------------------------------------------------|-----------------|---------|
| **Collaborative Filtering** | Users who liked X also liked Y (Matrix Factorization)  | 50              | 20ms    |
| **Content-Based**           | Items similar to user's past interactions (embeddings) | 30              | 5ms     |
| **Item-to-Item Similarity** | Items similar to currently viewed item (ANN search)    | 50              | 5ms     |
| **Trending/Popular**        | Globally popular items in user's preferred categories  | 20              | 2ms     |

**Total Candidate Pool:** ~150 items → Ranked by ML model → Top 10 served

*See pseudocode.md::CandidateGenerator for implementation.*

---

### 3.5 Offline Training Pipeline (Batch Layer)

> 📊 **See batch training architecture:** [HLD Diagrams - Batch Training](./hld-diagram.md#batch-training-pipeline)

**Training Schedule:** Daily (every 24 hours)

**Pipeline Steps:**

1. **Data Extraction** (4 hours)
    - Read last 90 days of clickstream data from Cassandra/ClickHouse
    - Filter noise (bots, outliers)
    - Output: Parquet files in S3 (~5 TB)

2. **Feature Engineering** (6 hours)
    - Compute user features (total purchases, avg session length, preferred categories)
    - Compute item features (popularity, engagement rate)
    - Output: Feature tables in S3

3. **Model Training** (10 hours)
    - **Collaborative Filtering:** Alternating Least Squares (ALS) on user-item matrix
    - **Deep Learning:** Two-tower neural network (user tower + item tower)
    - Output: Model artifacts (10 GB) saved to Model Store (S3/MLflow)

4. **Model Deployment** (1 hour)
    - Load new model into TensorFlow Serving
    - Gradual rollout (10% → 50% → 100% traffic)
    - Monitor A/B test metrics (CTR, conversion rate)

**Total Training Time:** ~21 hours (leaves 3 hours buffer before next cycle)

*See pseudocode.md::train_collaborative_filtering() for training logic.*

---

### 3.6 Feature Store Architecture

The **Feature Store** is the bridge between the batch layer and the speed layer, providing:

1. **Real-Time Features:** Derived from Kafka stream (last 5 views, last action timestamp)
2. **Batch Features:** Pre-computed from historical data (user embeddings, lifetime stats)

**Architecture:**

```
Kafka (Events)
  ↓
Kafka Streams / Flink (Stream Processor)
  ↓
Redis Cluster (Real-Time Feature Store)
  ↓
Recommendation Service (Fetch features in <5ms)
```

**Key Design Decisions:**

| Aspect                | Decision                   | Rationale                                                  |
|-----------------------|----------------------------|------------------------------------------------------------|
| **Real-Time Storage** | Redis Cluster              | Sub-5ms latency, supports atomic updates (INCR)            |
| **Batch Storage**     | S3 (Parquet)               | Cost-effective for petabyte-scale historical data          |
| **Stream Processor**  | Kafka Streams              | Exactly-once semantics, fault-tolerant stateful processing |
| **Feature Format**    | JSON (Redis), Parquet (S3) | JSON for flexibility, Parquet for compression              |

*See this-over-that.md for Redis vs DynamoDB comparison.*

---

### 3.7 Similarity Search (Item-to-Item)

**Problem:** Given an item `item_id`, find the 50 most similar items in <5ms from a catalog of 10 million items.

**Solution:** **Approximate Nearest Neighbors (ANN)** using **FAISS** with **HNSW** (Hierarchical Navigable Small World)
index.

**How It Works:**

1. Each item is represented as a **128-dimensional embedding vector** (learned during training).
2. All item vectors are indexed in **FAISS** using the **HNSW algorithm**.
3. At query time, compute similarity via **cosine distance** in the embedding space.

**Performance:**

- **Query Latency:** <5ms for top-50 similar items
- **Index Size:** 10M items × 128 dims × 4 bytes = ~5 GB (fits in RAM)
- **Accuracy:** 95% recall@50 (acceptable trade-off for speed)

*See pseudocode.md::find_similar_items() for implementation.*

---

## 4. Architectural Decisions (This Over That)

> 📊 **See detailed analysis:** [Design Decisions](./this-over-that.md)

### 4.1 Lambda Architecture vs Kappa Architecture

| Aspect                | Lambda (Chosen)                 | Kappa (Alternative)               |
|-----------------------|---------------------------------|-----------------------------------|
| **Training Data**     | Batch (historical, petabytes)   | Stream-only (recent, limited)     |
| **Model Accuracy**    | High (trained on full history)  | Lower (limited lookback)          |
| **System Complexity** | Higher (two separate pipelines) | Lower (single streaming pipeline) |
| **Latency**           | High (24h model refresh)        | Low (continuous retraining)       |

**Decision:** Lambda Architecture for now. Migrate to Kappa when stream processing matures.

---

### 4.2 Redis vs DynamoDB for Feature Store

| Aspect          | Redis (Chosen)                  | DynamoDB (Alternative)      |
|-----------------|---------------------------------|-----------------------------|
| **Latency**     | <2ms (in-memory)                | 5-10ms (SSD-backed)         |
| **Cost**        | Moderate (pay for RAM)          | Higher (pay for throughput) |
| **Scalability** | Manual sharding (Redis Cluster) | Auto-scaling (serverless)   |
| **Persistence** | Optional (AOF/RDB snapshots)    | Fully durable               |

**Decision:** Redis for <50ms latency requirement. DynamoDB as fallback for durability.

---

### 4.3 TensorFlow Serving vs Custom gRPC Service

| Aspect               | TensorFlow Serving (Chosen)      | Custom Service (Alternative)        |
|----------------------|----------------------------------|-------------------------------------|
| **Latency**          | 10-20ms (batching, GPU)          | 5-10ms (CPU, no framework overhead) |
| **Flexibility**      | Limited (TensorFlow models only) | Full control (any algorithm)        |
| **Ops Complexity**   | Lower (managed infra)            | Higher (custom deployment)          |
| **Model Versioning** | Built-in (A/B testing support)   | Manual implementation               |

**Decision:** TensorFlow Serving for managed ops. Migrate to custom service if latency becomes critical.

---

## 5. Bottlenecks and Future Scaling

### 5.1 Serving Latency (Current: 30-50ms)

**Problem:** Multiple network calls (Redis, Model Serving, FAISS) add up to 30-50ms.

**Mitigations:**

1. **Pre-Compute Popular Recommendations**
    - Cache top 10 recommendations for 80% of users (based on rough user segments)
    - Store in Redis with 1-hour TTL
    - Reduces latency to <5ms for cache hits

2. **Single-Flight gRPC Calls**
    - Parallelize feature fetch and candidate generation using Go's goroutines or async I/O
    - Reduces sequential latency from 27ms → 5ms (longest call)

3. **Model Optimization**
    - Quantize model weights (float32 → int8) for 4x smaller size and faster inference
    - Use TensorFlow Lite or ONNX Runtime for optimized serving

*See pseudocode.md::optimize_latency() for implementation.*

---

### 5.2 Feature Store Throughput (Current: 1.1M lookups/sec)

**Problem:** Redis is single-threaded per shard, limiting throughput.

**Mitigations:**

1. **Increase Redis Cluster Shards**
    - Scale from 10 shards → 50 shards (distribute load)
    - Use consistent hashing to route `user_id` to specific shard

2. **Read Replicas**
    - Deploy 2 read replicas per master shard
    - Route all feature reads to replicas (master handles writes only)

3. **Feature Denormalization**
    - Duplicate frequently accessed features across multiple keys
    - Trade storage for reduced query complexity

---

### 5.3 Model Freshness (Current: 24 hours)

**Problem:** User preferences change quickly (e.g., breaking news, trending products). 24-hour model refresh is too
slow.

**Mitigations:**

1. **Incremental Model Updates**
    - Instead of full retrain, update model incrementally every 1-2 hours using mini-batch gradient descent
    - Requires streaming ML framework (Flink ML, River)

2. **Hybrid Real-Time Boosting**
    - Use batch model as base score
    - Apply real-time boost based on trending signals (e.g., +10% score for items clicked in last hour)
    - Implemented in ranking layer (no model retrain needed)

3. **Migrate to Kappa Architecture**
    - Replace batch training with continuous stream-based training
    - Requires significant engineering investment (future roadmap)

*See this-over-that.md::Model-Freshness for detailed trade-offs.*

---

### 5.4 Cold Start Problem

**Problem:** New users have no history. New items have no engagement data.

**Mitigations:**

**New Users:**

1. Explicit onboarding (ask for interests during signup)
2. Serve globally popular items in selected categories
3. Aggressively explore (serve diverse content to learn preferences fast)

**New Items:**

1. Use content-based filtering (metadata, title, description similarity)
2. Manual curation (editorial picks for new items)
3. Exploration budget (show new items to 5% of users, monitor engagement)

*See pseudocode.md::handle_cold_start() for implementation.*

---

## 6. Common Anti-Patterns

### ❌ **Anti-Pattern 1: Synchronous Model Inference in Critical Path**

**Problem:** Calling TensorFlow Serving synchronously adds 20ms to every request, violating latency SLA.

**Why It's Bad:**

- P99 latency spikes to 80-100ms during model server load
- Single point of failure (model server down = no recommendations)

**Solution:**

✅ **Use Pre-Computed Recommendations**

- Generate and cache recommendations for all users every 1 hour
- Serve from Redis cache (<2ms)
- Fall back to real-time inference only for cache misses (5% of requests)

```
function get_recommendations(user_id):
  cached = redis.get(f"recs:{user_id}")
  if cached:
    return cached  // <2ms
  
  // Cache miss: compute in real-time
  features = fetch_features(user_id)
  recommendations = model_serving_api.predict(features)  // 20ms
  redis.set(f"recs:{user_id}", recommendations, ttl=3600)
  return recommendations
```

---

### ❌ **Anti-Pattern 2: Training on Raw Clickstream Events**

**Problem:** Directly training on raw events (views, clicks) without aggregation or filtering.

**Why It's Bad:**

- Bots and outliers pollute training data (one user with 10,000 clicks/day)
- Implicit feedback (views) is noisy (accidental clicks, curiosity browsing)
- Model overfits to power users

**Solution:**

✅ **Feature Engineering + Data Cleaning**

- Filter bot traffic (>100 events/minute from single user)
- Aggregate events into sessions (group by 30-minute window)
- Weight events by type: `PURCHASE=10`, `CLICK=3`, `VIEW=1`
- Cap user engagement (max 50 events/day per user for training)

*See pseudocode.md::clean_training_data() for implementation.*

---

### ❌ **Anti-Pattern 3: Using Single Recommendation Strategy**

**Problem:** Relying only on collaborative filtering (users who liked X also liked Y).

**Why It's Bad:**

- **Filter Bubble:** Users only see items similar to past behavior (no discovery)
- **Cold Start:** Fails for new users/items with no history
- **Popularity Bias:** Always recommends popular items (ignores niche interests)

**Solution:**

✅ **Multi-Strategy Blending + Exploration**

- Blend 4 strategies: collaborative (50%), content-based (20%), popular (20%), random exploration (10%)
- Use Thompson Sampling for exploration (balance exploitation vs exploration)
- Enforce diversity constraint (no more than 3 items from same category in top 10)

*See pseudocode.md::blend_strategies() for implementation.*

---

### ❌ **Anti-Pattern 4: Ignoring Temporal Dynamics**

**Problem:** Treating all historical data equally (5-year-old click has same weight as today's click).

**Why It's Bad:**

- User preferences evolve (past interests may no longer be relevant)
- Seasonal trends ignored (e.g., holiday shopping patterns)
- Model becomes stale and less accurate over time

**Solution:**

✅ **Time-Decay Weighting + Sliding Window**

- Apply exponential decay to historical events: $\text{weight} = e^{-\lambda \cdot \text{days\_ago}}$
- Use 90-day sliding window for training (ignore older data)
- Separate models for different time periods (e.g., holiday season model)

*See pseudocode.md::apply_time_decay() for implementation.*

---

## 7. Alternative Approaches

### 7.1 Kappa Architecture (Stream-Only)

**How It Works:**

- Replace batch training with continuous stream-based training using Flink ML or Kafka Streams
- Model updates incrementally every hour instead of daily full retrain

**Pros:**

- ✅ Lower model freshness latency (1 hour vs 24 hours)
- ✅ Simpler architecture (single pipeline)
- ✅ Captures breaking trends faster

**Cons:**

- ❌ Limited training data (only recent events, not full history)
- ❌ Stream processing is complex and expensive
- ❌ Requires mature streaming ML infrastructure

**When to Use:** When model freshness is critical (news, social media, live events).

---

### 7.2 Federated Learning (On-Device Training)

**How It Works:**

- Train recommendation models locally on user devices (mobile phones)
- Aggregate model updates on server without seeing raw user data

**Pros:**

- ✅ Privacy-preserving (user data never leaves device)
- ✅ Captures hyper-personalized signals

**Cons:**

- ❌ High device compute/battery cost
- ❌ Complex model aggregation logic
- ❌ Limited to simple models (cannot train large neural networks)

**When to Use:** Privacy-sensitive applications (healthcare, finance).

---

### 7.3 Graph-Based Recommendations (GNN)

**How It Works:**

- Model users and items as nodes in a graph (edges = interactions)
- Use Graph Neural Networks (GNN) to learn embeddings that capture graph structure

**Pros:**

- ✅ Captures complex relationships (friends-of-friends, co-purchased items)
- ✅ Better cold start (propagate signals through graph)

**Cons:**

- ❌ Computationally expensive (graph traversal + message passing)
- ❌ Hard to scale to billions of nodes/edges
- ❌ Requires specialized infrastructure (Neo4j, DGL)

**When to Use:** Social networks, e-commerce with rich relationship data.

---

## 8. Monitoring and Observability

### Key Metrics

| Metric                       | Target    | Alert Threshold | Purpose                        |
|------------------------------|-----------|-----------------|--------------------------------|
| **Serving Latency (p99)**    | <50ms     | >100ms          | User experience                |
| **Cache Hit Rate**           | >80%      | <70%            | Reduce model inference load    |
| **Model Inference Time**     | <20ms     | >50ms           | Detect model server overload   |
| **Feature Fetch Latency**    | <5ms      | >10ms           | Redis performance              |
| **CTR (Click-Through Rate)** | 5-10%     | <3%             | Recommendation relevance       |
| **Conversion Rate**          | 2-5%      | <1%             | Business impact                |
| **Model Training Duration**  | <20 hours | >24 hours       | Ensure daily refresh completes |
| **Feature Store QPS**        | 100k      | >200k           | Capacity planning              |

*See this-over-that.md::Monitoring for detailed dashboard design.*

---

## 9. Trade-offs Summary

| What We Gain                            | What We Sacrifice                                             |
|-----------------------------------------|---------------------------------------------------------------|
| ✅ **High Accuracy** (petabyte training) | ❌ Model freshness (24h lag)                                   |
| ✅ **Low Latency** (<50ms serving)       | ❌ High infrastructure cost (Redis, FAISS, TensorFlow Serving) |
| ✅ **Scalability** (100k QPS)            | ❌ System complexity (Lambda Architecture)                     |
| ✅ **Multi-Strategy Flexibility**        | ❌ Harder to debug/explain recommendations                     |
| ✅ **Handles Cold Start** (hybrid)       | ❌ Suboptimal for new users initially                          |

---

## 10. Real-World Examples

| Company     | Use Case                | Key Techniques                                                       |
|-------------|-------------------------|----------------------------------------------------------------------|
| **Netflix** | Video recommendations   | Two-tower neural network, A/B testing, matrix factorization          |
| **Amazon**  | Product recommendations | Item-to-item collaborative filtering, real-time session features     |
| **YouTube** | Video recommendations   | Deep neural networks, candidate generation + ranking, exploration    |
| **Spotify** | Music recommendations   | Collaborative filtering, NLP on song metadata, session-based recs    |
| **TikTok**  | For You Page            | Real-time features, short-term engagement signals, diversity ranking |

---

## 11. Key Takeaways

1. **Lambda Architecture** is essential for balancing accuracy (batch training) and freshness (real-time features).
2. **Feature Store** is the critical component bridging batch and speed layers.
3. **Pre-compute and cache** aggressively to meet <50ms latency SLA (80% cache hit rate).
4. **Multi-strategy blending** (collaborative + content-based + exploration) prevents filter bubbles.
5. **ANN (FAISS/HNSW)** enables sub-5ms similarity search for item-to-item recommendations.
6. **Cold start** requires hybrid approach (explicit signals + exploration + content-based).
7. **Model freshness vs accuracy** is the fundamental trade-off (24h batch vs real-time stream).

---

## 12. References

### Academic Papers

- **[Collaborative Filtering for Implicit Feedback Datasets](http://yifanhu.net/PUB/cf.pdf)** - Hu, Koren, Volinsky (
  2008) - Foundation for ALS algorithm
- *
  *[Deep Neural Networks for YouTube Recommendations](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/45530.pdf)
  ** - Covington et al. (2016) - Two-tower architecture
- **[The Netflix Recommender System: Algorithms, Business Value, and Innovation](https://dl.acm.org/doi/10.1145/2843948)
  ** - Gomez-Uribe, Hunt (2015) - Production ML at scale

### Technical Documentation

- **[FAISS: A Library for Efficient Similarity Search](https://github.com/facebookresearch/faiss)** - Facebook Research
- **[TensorFlow Serving](https://www.tensorflow.org/tfx/guide/serving)** - Google
- **[MLflow Model Registry](https://www.mlflow.org/docs/latest/model-registry.html)** - Databricks

### Related Chapters

- **[2.3.5 Batch vs Stream Processing](../../02-components/2.3.5-batch-vs-stream-processing.md)** - Lambda vs Kappa
  architecture
- **[2.1.1 RDBMS Deep Dive](../../02-components/2.1.1-rdbms-deep-dive.md)** - PostgreSQL for item metadata
- **[2.2.1 Caching Deep Dive](../../02-components/2.2.1-caching-deep-dive.md)** - Redis feature store design
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)** - Clickstream ingestion
- **[3.1.3 Distributed ID Generator](../3.1.3-distributed-id-generator/)** - User/Item ID generation
- **[3.4.3 Distributed Monitoring System](../3.4.3-monitoring-system/)** - Monitoring ML pipelines

---

## 13. Performance Summary

| Component              | Latency  | Throughput | Bottleneck          | Mitigation            |
|------------------------|----------|------------|---------------------|-----------------------|
| **Feature Fetch**      | 2-5ms    | 1.1M QPS   | Redis single-thread | Increase shards (50+) |
| **Model Inference**    | 10-20ms  | 10k QPS    | GPU compute         | Batch inference       |
| **ANN Search (FAISS)** | 2-5ms    | 100k QPS   | Index size (RAM)    | Partition by category |
| **Total Serving**      | 30-50ms  | 100k QPS   | Sequential calls    | Parallelize + cache   |
| **Batch Training**     | 20 hours | 1 job/day  | Spark shuffle       | Optimize joins        |

---
