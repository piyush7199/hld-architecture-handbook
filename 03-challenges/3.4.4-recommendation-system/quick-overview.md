# Recommendation System - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A recommendation system that provides real-time, highly personalized recommendations to users (products, content,
videos, articles) with extremely low latency (<50ms). The system serves personalized recommendations instantly (100k
QPS, <50ms p99 latency), trains ML models offline on petabytes of historical data (clickstream, purchases, ratings),
updates real-time features (user's last 5 views, current session context), scales to 100M daily active users with
diverse recommendation scenarios, and handles cold-start problem for new users and new items.

**Similar Systems:** Netflix recommendations, Amazon product recommendations, YouTube video suggestions, TikTok "For
You" feed, Spotify Discover Weekly

**Key Characteristics:**

- **Real-time serving** - <50ms p99 latency for recommendation API calls
- **Lambda architecture** - Batch training (offline) + Speed layer (real-time serving)
- **Personalized recommendations** - Based on user history, preferences, and real-time context
- **Cold start handling** - Recommendations for new users and new items
- **A/B testing support** - Multiple recommendation algorithms simultaneously
- **High throughput** - 100k QPS for read requests

---

## Requirements & Scale

### Functional Requirements

1. **Personalized Feed** - Generate "For You" recommendations based on user history and preferences
2. **Similar Items** - "You may also like" recommendations based on current item
3. **Real-Time Context** - Incorporate current session behavior (last 5 views, search queries)
4. **Cold Start Handling** - Provide recommendations for new users (no history) and new items
5. **A/B Testing** - Support multiple recommendation algorithms simultaneously
6. **Explainability** - Provide reasons for recommendations ("Because you watched...")

### Non-Functional Requirements

1. **Low Latency (Serving):** <50ms p99 latency for recommendation API calls
2. **High Throughput:** 100k QPS for read requests (recommendation serving)
3. **High Availability:** 99.99% uptime (recommendations critical for engagement)
4. **Model Freshness:** Models retrained daily (24-hour lag acceptable)
5. **Feature Freshness:** Real-time features updated within 1 second
6. **Scalability:** Handle 100M DAU with 10+ recommendations per user per day

### Scale Estimation

| Metric                      | Calculation                                  | Result                        |
|-----------------------------|----------------------------------------------|-------------------------------|
| **Daily Active Users**      | Given                                        | 100 Million DAU               |
| **Recommendation Requests** | 100M users × 10 requests/user/day            | 1 Billion requests/day        |
| **Peak QPS**                | 1B requests / 86,400 sec × 3x peak           | **100,000 QPS** (peak)        |
| **Avg QPS**                 | 1B requests / 86,400 sec                     | 11,574 QPS (average)          |
| **Latency Target**          | Given                                        | <50ms p99                     |
| **Training Data**           | 100M users × 100 actions/user/day × 365 days | **3.65 trillion events/year** |
| **Training Data Size**      | 3.65T events × 100 bytes/event               | **365 TB/year**               |
| **Feature Store QPS**       | Same as recommendation QPS                   | 100,000 reads/sec             |
| **Model Storage**           | 100M users × 128D embeddings × 4 bytes       | ~50 GB (user embeddings)      |

**Storage Breakdown:**

- **User Embeddings:** 100M users × 128 dimensions × 4 bytes = 50 GB
- **Item Embeddings:** 10M items × 128 dimensions × 4 bytes = 5 GB
- **Feature Store:** 100M users × 1 KB features = 100 GB
- **Training Data (1 year):** 365 TB (compressed: ~100 TB with Parquet)

---

## Key Components

| Component               | Responsibility                               | Technology           | Scalability                   |
|-------------------------|----------------------------------------------|----------------------|-------------------------------|
| **Recommendation API**  | Serve personalized recommendations           | Go, Java (gRPC/REST) | Horizontal (100 instances)    |
| **Feature Store**       | Store and serve real-time user/item features | Redis Cluster        | Horizontal (sharded)          |
| **Model Store**         | Store pre-trained embeddings and models      | S3, In-memory cache  | Distributed                   |
| **Batch Training**      | Train ML models on historical data           | Apache Spark         | Horizontal (100-node cluster) |
| **Stream Processing**   | Update real-time features from events        | Kafka Streams, Flink | Horizontal                    |
| **Event Collection**    | Collect user interactions                    | Kafka                | Horizontal (partitioned)      |
| **A/B Testing Service** | Route users to different algorithms          | Consistent hashing   | Horizontal                    |

---

## Lambda Architecture

The system follows **Lambda Architecture** with two distinct processing paths:

**1. Batch Layer (Offline Training)**

- Process historical data (petabytes) to train accurate ML models
- Runs daily (24-hour lag acceptable)
- Uses Spark/Hadoop for distributed training

**2. Speed Layer (Real-Time Serving)**

- Serves recommendations in <50ms using pre-trained models
- Updates real-time features (user's last actions) from Kafka stream
- Uses Redis Feature Store for low-latency lookups

---

## Recommendation Flow

**Steps:**

1. **Receive Request**
   ```
   GET /recommendations?user_id=123&context=homepage&num_results=20
   ```

2. **Fetch User Features** (from Redis Feature Store)
    - Recent views, preferences, demographics
    - Latency: <1ms (Redis local cache)

3. **Load ML Model** (from in-memory cache)
    - User embedding, item embeddings
    - Latency: ~0ms (pre-loaded in memory)

4. **Candidate Generation** (100-1000 candidates)
    - Collaborative Filtering: Top 500 items based on user embedding similarity
    - Content-Based: Top 300 items based on recent views
    - Popular Items: Top 200 items (trending, high engagement)

5. **Ranking** (Score 1000 candidates)
    - Multi-layer perceptron (MLP) model
    - Features: user embedding · item embedding, popularity, price, category match
    - Output: Personalized score for each candidate

6. **Post-Processing**
    - Deduplication (remove already viewed items)
    - Diversity (ensure variety in categories)
    - Business rules (promote sponsored items, new arrivals)

7. **Return Top N** (e.g., 20 items)
    - Latency: ~30ms (p50), ~50ms (p99)

---

## Feature Store (Redis)

**Purpose:** Store and serve real-time user/item features with <1ms latency.

**Tech Stack:**

- Redis Cluster (6 nodes, 3 replicas)
- Memory: 500 GB (100M users × 5 KB features)
- Persistence: AOF (Append-Only File) for durability

**Feature Update Pipeline:**

1. **Event Collection** (Kafka)
    - User action (view, click, purchase) → Kafka topic `user_events`

2. **Stream Processing** (Kafka Streams or Flink)
    - Aggregate recent actions (last 5 views) in sliding window
    - Compute session context (current category, page views)

3. **Write to Redis**
    - Update `user:{user_id}:features` with latest data
    - Latency: <100ms (event → Redis)

**Example: Update Recent Views**

When user views item "789":

1. Fetch current recent_views: ["456", "123", "999", "888", "777"]
2. Add new item to front: ["789", "456", "123", "999", "888"]
3. Trim to max 5 items
4. Write back to Redis
5. Set TTL: 24 hours

---

## Batch Training (Spark)

**Purpose:** Train ML models on historical data (365 TB) to generate accurate user/item embeddings.

**Tech Stack:**

- Apache Spark (100-node cluster)
- Training Data: S3 (Parquet format, partitioned by date)
- Training Framework: Spark MLlib (ALS), PyTorch (Deep Learning)

**Training Pipeline (Daily Job):**

1. **Data Collection** (6 hours, 12 AM - 6 AM)
    - Read last 30 days of user interactions from S3
    - Filter out bots, invalid events
    - Aggregate implicit feedback (views, clicks) and explicit feedback (ratings)

2. **Feature Engineering** (1 hour)
    - User features: Interaction history, demographics, engagement metrics
    - Item features: Category, price, popularity, tags

3. **Model Training** (4 hours)
    - **Collaborative Filtering:** Matrix Factorization (ALS algorithm)
        - Input: User-item interaction matrix (100M users × 10M items)
        - Output: User embeddings (128D), Item embeddings (128D)
        - Algorithm: Alternating Least Squares (Spark MLlib)

    - **Deep Learning:** Two-Tower Model (optional, for higher accuracy)
        - User tower: [user features] → MLP → 128D embedding
        - Item tower: [item features] → MLP → 128D embedding
        - Training: Contrastive learning (positive pairs: user-item interactions)

4. **Model Evaluation** (1 hour)
    - Offline metrics: Precision@10, Recall@10, NDCG@10, AUC
    - Hold-out validation set (last 7 days)
    - Compare to baseline model (previous day)

5. **Model Deployment** (30 min)
    - Export embeddings to S3 (Parquet format)
    - Blue-green deployment (gradual rollout to 10% → 100%)
    - Monitor online metrics (CTR, engagement)

**Total Runtime:** ~12 hours (overnight job)

---

## Candidate Generation and Ranking

**Candidate Generation (Recall):**

Goal: Generate 1000 candidates from 10M item catalog in <10ms.

**Three Sources:**

1. **Collaborative Filtering** (500 candidates)
    - Compute: user_embedding · item_embeddings (dot product)
    - Use Approximate Nearest Neighbors (ANN) for fast search
    - Algorithm: HNSW (Hierarchical Navigable Small World)
    - Latency: <5ms for 500 candidates

2. **Content-Based** (300 candidates)
    - Based on user's recent views and preferences
    - Filter items by category, price range, brand
    - Use inverted index for fast filtering

3. **Popular Items** (200 candidates)
    - Trending items (high engagement in last 24 hours)
    - New arrivals (recently added to catalog)
    - Editorial picks (curated by content team)

**Ranking (Precision):**

Goal: Score 1000 candidates and return top 20 in <20ms.

**Ranking Model:**

Input features (per candidate):

- User embedding · Item embedding (dot product)
- User features: age, gender, location
- Item features: price, category, popularity, rating
- Context features: time of day, device type
- Interaction features: category match, brand match

Model: Multi-layer Perceptron (MLP) with 3 hidden layers

- Layer 1: 128 neurons (ReLU)
- Layer 2: 64 neurons (ReLU)
- Layer 3: 32 neurons (ReLU)
- Output: 1 neuron (sigmoid) → score [0, 1]

Training: Logistic regression (binary classification)

- Positive examples: User clicked/purchased item
- Negative examples: User viewed but did not click

**Inference:**

- Batch inference: Score all 1000 candidates in one forward pass
- Latency: <10ms (CPU) or <2ms (GPU)

---

## Bottlenecks & Solutions

### Bottleneck 1: Serving Latency (>50ms)

**Problem:**

- Multiple network calls: Feature Store (Redis), Model Store (S3)
- Complex computation: Candidate generation (ANN), ranking (MLP inference)
- Target: <50ms p99, but currently ~80ms

**Solutions:**

1. **Pre-Compute Popular Recommendations**
    - Cache top 1000 recommendations for generic queries (no user context)
    - Covers 30% of traffic (anonymous users, cold start)
    - Latency: <5ms (Redis cache hit)

2. **In-Memory Caching**
    - Load user/item embeddings in serving instance memory
    - Avoids Redis roundtrip for hot users/items
    - Cache hit rate: 80% (Pareto principle: 20% of users generate 80% of traffic)

3. **Model Simplification**
    - Use lightweight ranking model (logistic regression instead of deep MLP)
    - Trade-off: 2% accuracy loss for 10x faster inference

4. **Parallelization**
    - Fetch features and generate candidates in parallel (goroutines)
    - Use gRPC multiplexing for batch requests

5. **Edge Caching (CDN)**
    - Cache recommendations at CDN edge (CloudFlare, Fastly)
    - TTL: 1 minute (acceptable staleness for non-personalized scenarios)

### Bottleneck 2: Feature Store Overload (100k QPS)

**Problem:**

- Redis cluster handles 100k reads/sec
- Each read: ~1 KB data (user features)
- Memory: 500 GB (exceeds single-node capacity)

**Solutions:**

1. **Redis Cluster Sharding**
    - Shard by user_id (consistent hashing)
    - 10 Redis nodes (10k QPS each)
    - Replication: 3x (master + 2 replicas)

2. **Feature Compression**
    - Store only essential features (recent 5 views, not full history)
    - Use efficient encoding (protobuf instead of JSON)
    - Reduce payload size: 1 KB → 200 bytes (5x compression)

3. **Read Replicas**
    - Route reads to replicas (read-heavy workload)
    - Master: Handles writes only (~1k QPS)
    - Replicas: Handle reads (100k QPS split across replicas)

4. **Local Cache (Serving Instance)**
    - Cache hot user features in serving instance memory
    - Eviction: LRU (Least Recently Used)
    - Cache hit rate: 70% (reduces Redis load by 70%)

### Bottleneck 3: Batch Training Scalability (365 TB)

**Problem:**

- Training data: 365 TB/year (3.65 trillion events)
- Training time: 12 hours (too long, blocks next-day deployment)
- Spark cluster: 100 nodes (expensive)

**Solutions:**

1. **Incremental Training**
    - Train only on last 30 days (not full year)
    - Reduces data: 365 TB → 30 TB (12x smaller)
    - Training time: 12 hours → 2 hours

2. **Sample-Based Training**
    - Train on 10% random sample (representative)
    - Reduces data: 30 TB → 3 TB (10x smaller)
    - Trade-off: 1-2% accuracy loss

3. **Model Distillation**
    - Train large, accurate model offline (weekly)
    - Distill to smaller model (daily fine-tuning)
    - Faster training, similar accuracy

4. **Feature Pre-Aggregation**
    - Pre-compute aggregates (user's top categories, brands) offline
    - Training uses pre-aggregated features (not raw events)
    - Reduces training complexity

### Bottleneck 4: Cold Start Problem

**Problem:**

- New users: No interaction history → Cannot generate personalized recommendations
- New items: No user interactions → Cannot compute item embeddings

**Solutions:**

1. **New User Cold Start**
    - **Option 1:** Demographic-based recommendations (age, gender, location)
    - **Option 2:** Onboarding quiz ("Select your interests")
    - **Option 3:** Popular items (trending, editorial picks)

2. **New Item Cold Start**
    - **Option 1:** Content-based features (category, tags, price)
    - **Option 2:** Similar items (based on content features, not interactions)
    - **Option 3:** Promote as "New Arrivals" (boost in ranking)

3. **Hybrid Approach**
    - Combine collaborative filtering (if available) + content-based (always available)
    - Weight: 70% collaborative + 30% content-based (for warm users)
    - Weight: 0% collaborative + 100% content-based (for cold users)

---

## Common Anti-Patterns to Avoid

### 1. Synchronous Model Training During Serving

❌ **Bad:** Training model on-the-fly during recommendation request

- Latency: >5 seconds (unacceptable)

✅ **Good:** Pre-Train Models Offline (Batch Layer)

- Train models once per day in batch job
- Serve pre-computed embeddings (cached in memory)
- Latency: <50ms

### 2. Single Recommendation Algorithm

❌ **Bad:** Only collaborative filtering (fails for cold start) or only content-based (ignores user preferences)

✅ **Good:** Hybrid Approach (Ensemble)

- Combine collaborative filtering + content-based + popularity
- Weight: 60% collaborative + 30% content-based + 10% popular
- Handles cold start gracefully

### 3. No Real-Time Features

❌ **Bad:** Only use batch features (updated daily)

- Ignores user's current session behavior

✅ **Good:** Real-Time Feature Store (Speed Layer)

- Update features in real-time (<1 second) from Kafka stream
- Incorporate current session context (last 5 views)
- Improves personalization significantly (+15% CTR)

### 4. No Diversity in Recommendations

❌ **Bad:** All recommendations from same category (filter bubble)

- Example: User views 1 laptop → Recommended 20 laptops

✅ **Good:** Diversity Post-Processing

- Ensure variety in categories (max 5 items per category)
- Add exploration (10% random items from different categories)
- Balance exploitation (user preferences) vs exploration (discovery)

### 5. Ignoring Negative Signals

❌ **Bad:** Only use positive feedback (clicks, purchases)

- Ignores negative signals (user skipped item, closed page quickly)

✅ **Good:** Incorporate Negative Feedback

- Explicit: User dislikes item (thumbs down)
- Implicit: Low dwell time (<5 seconds), did not click, did not purchase
- Use in training: Negative examples for logistic regression

---

## Multi-Algorithm Serving (A/B Testing)

**Challenge:** Run multiple recommendation algorithms simultaneously for A/B testing.

**Architecture:**

```
                     [Recommendation API]
                             |
          ┌──────────────────┼──────────────────┐
          │                  │                  │
    [Algorithm A]      [Algorithm B]      [Algorithm C]
  Collaborative      Content-Based      Deep Learning
   Filtering             (Baseline)       (Two-Tower)
      50%                  25%                 25%
```

**Implementation:**

1. **User Bucketing** (Consistent Hashing)
    - Hash user_id → Assign to bucket [A, B, C]
    - Bucket A: 50% of users (Collaborative Filtering)
    - Bucket B: 25% of users (Content-Based)
    - Bucket C: 25% of users (Deep Learning)

2. **Model Versioning**
    - Store multiple models in Model Store (S3)
    - Model A: `s3://models/collaborative-v2024-10-29.pkl`
    - Model B: `s3://models/content-based-v2024-10-29.pkl`
    - Model C: `s3://models/two-tower-v2024-10-29.pkl`

3. **Metrics Collection**
    - Track CTR, conversion rate per algorithm
    - Compare performance (A vs B vs C)
    - Gradual rollout: Start with 10% traffic, increase to 50% if better

---

## Monitoring & Observability

### Key Metrics

**Business Metrics:**
| Metric | Target | Alert Threshold |
|--------------------------|---------------|-----------------|
| Click-Through Rate (CTR) | >5% | <4% |
| Conversion Rate | >2% | <1.5% |
| Engagement (dwell time)  | >60 seconds | <45 seconds |
| Diversity (categories)   | >5 categories | <3 categories |
| Cold Start Coverage | >95% | <90% |

**System Metrics:**
| Metric | Target | Alert Threshold |
|-----------------------|-----------|-----------------|
| API Latency (p99)     | <50ms | >100ms |
| API QPS | 100k | <50k (degraded) |
| Feature Store Latency | <1ms | >5ms |
| Cache Hit Rate | >80% | <70% |
| Model Freshness | <24 hours | >48 hours |
| Error Rate | <0.1% | >1% |

### Dashboards

1. **Real-Time Dashboard** (Grafana)
    - API QPS, latency (p50, p95, p99)
    - Error rate, timeout rate
    - Cache hit rate (Feature Store, Model Store)

2. **Business Dashboard** (Looker/Tableau)
    - Daily CTR, conversion rate
    - A/B test results (algorithm comparison)
    - Revenue impact (attributable to recommendations)

3. **Training Dashboard** (MLflow)
    - Training job status (running, failed, completed)
    - Offline metrics (Precision@10, NDCG@10)
    - Model version, deployment status

### Alerts

**Critical Alerts** (PagerDuty, 24/7 on-call):

- API latency >100ms for 5 minutes → Page on-call engineer
- Error rate >1% for 5 minutes → Page on-call engineer
- Feature Store down (Redis cluster unavailable) → Page on-call engineer

**Warning Alerts** (Slack, email):

- CTR drops >10% compared to yesterday → Alert ML team
- Model training job failed → Alert ML engineers
- Cache hit rate <70% → Alert infrastructure team

---

## Trade-offs Summary

| What We Gain                              | What We Sacrifice                               |
|-------------------------------------------|-------------------------------------------------|
| ✅ Low latency serving (<50ms)             | ❌ Complex architecture (Lambda with two layers) |
| ✅ High accuracy (collaborative filtering) | ❌ 24-hour model lag (batch training)            |
| ✅ Real-time features (<1 second)          | ❌ Additional infrastructure (Kafka, Redis)      |
| ✅ Scalable (100k QPS)                     | ❌ High cost ($25k/month)                        |
| ✅ Handles cold start (hybrid approach)    | ❌ Model complexity (multiple algorithms)        |
| ✅ A/B testing support                     | ❌ More operational overhead                     |

---

## Real-World Examples

### Netflix

- **Scale:** 200M users, 10k+ recommendations per user per day
- **Architecture:** Lambda (offline training + online serving)
- **Algorithms:** Matrix factorization, deep learning (two-tower), bandits
- **Key Innovation:** Row-based personalization (even thumbnail images personalized)

### Amazon

- **Scale:** 300M users, item-to-item collaborative filtering
- **Architecture:** Batch pre-computation (item-item similarity matrix)
- **Algorithms:** "Customers who bought X also bought Y"
- **Key Innovation:** Real-time updates (purchase → update recommendations within 1 minute)

### YouTube

- **Scale:** 2B users, 1B hours watched daily
- **Architecture:** Two-stage (candidate generation + ranking)
- **Algorithms:** Deep neural networks (watch time prediction)
- **Key Innovation:** Real-time features (watch history in current session)

---

## Key Takeaways

1. **Lambda Architecture** separates batch training (offline) from real-time serving (online)
2. **Two-stage approach** (candidate generation + ranking) balances recall and precision
3. **Feature Store (Redis)** enables <1ms feature lookups for real-time personalization
4. **Collaborative Filtering** (matrix factorization) generates accurate user/item embeddings
5. **Real-time features** (last 5 views, session context) improve CTR by 15%+
6. **Hybrid approach** (collaborative + content-based) handles cold start gracefully
7. **A/B testing** enables algorithm comparison and gradual rollout
8. **In-memory caching** reduces latency by 80% (hot users/items)
9. **ANN search** (HNSW) enables fast candidate generation from 10M item catalog
10. **Model distillation** reduces training time while maintaining accuracy

---

## Recommended Stack

- **Serving API:** Go or Java (gRPC/REST), 100 instances
- **Feature Store:** Redis Cluster (6 nodes, sharded by user_id)
- **Model Store:** S3 (embeddings), In-memory cache (hot models)
- **Batch Training:** Apache Spark (100-node cluster), Spark MLlib
- **Stream Processing:** Kafka Streams or Apache Flink
- **Event Collection:** Apache Kafka
- **Monitoring:** Prometheus + Grafana (metrics), MLflow (model tracking)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (Lambda architecture, feature store, model serving)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (real-time serving, batch training,
  feature updates)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (candidate generation, ranking, feature updates)

