# Recommendation System - High-Level Design

## Table of Contents

1. [System Architecture (Lambda Architecture)](#system-architecture-lambda-architecture)
2. [Real-Time Serving Path](#real-time-serving-path)
3. [Batch Training Pipeline](#batch-training-pipeline)
4. [Feature Store Architecture](#feature-store-architecture)
5. [Model Serving Infrastructure](#model-serving-infrastructure)
6. [Candidate Generation Strategy](#candidate-generation-strategy)
7. [Item-to-Item Similarity (FAISS/ANN)](#item-to-item-similarity-faiss-ann)
8. [Redis Feature Store Sharding](#redis-feature-store-sharding)
9. [Multi-Region Deployment](#multi-region-deployment)
10. [Pre-Computed Recommendations Cache](#pre-computed-recommendations-cache)
11. [A/B Testing Infrastructure](#ab-testing-infrastructure)
12. [Cold Start Handling](#cold-start-handling)
13. [Model Training DAG (Airflow)](#model-training-dag-airflow)
14. [Monitoring and Alerting Dashboard](#monitoring-and-alerting-dashboard)

---

## System Architecture (Lambda Architecture)

**Flow Explanation:**

This diagram shows the complete Lambda Architecture with two independent processing paths:

**Batch Layer (Slow Path):**

1. Historical clickstream data (5B events/day) stored in Cassandra/ClickHouse
2. Daily Spark jobs process petabytes of data to train ML models
3. Trained models stored in Model Store (S3/MLflow)
4. Takes 20 hours to complete

**Speed Layer (Fast Path):**

1. Real-time events ingested via Kafka (100k events/sec)
2. Kafka Streams computes real-time features (last 5 views, last action timestamp)
3. Features stored in Redis for <5ms lookups
4. Updates continuously in real-time

**Serving Layer:**

- Fetches features from Redis (fast path)
- Loads model from Model Store (slow path)
- Combines both to generate recommendations in <50ms

**Benefits:**

- High accuracy (trained on full historical data)
- Low latency (real-time features available instantly)
- Scalable (batch and stream layers scale independently)

**Trade-offs:**

- Complex (two separate pipelines to maintain)
- Model freshness lag (24h for batch retrain)

```mermaid
graph TB
    subgraph "Data Sources"
        Users[Users<br/>100M DAU<br/>50 events/user/day]
    end

    subgraph "Ingestion Layer"
        Kafka[Kafka<br/>Clickstream Events<br/>5B events/day<br/>100k events/sec]
    end

    Users -->|View, Click, Purchase| Kafka

    subgraph "Batch Layer - Slow Path"
        HistoricalDB[(Cassandra/ClickHouse<br/>Historical Store<br/>9 PB - 5 years<br/>90-day sliding window)]
        SparkCluster[Spark Cluster<br/>Feature Engineering<br/>Model Training<br/>20 hours daily]
        ModelStore[(Model Store<br/>S3/MLflow<br/>10 GB artifacts<br/>Version control)]
    end

    Kafka -->|Persist historical data| HistoricalDB
    HistoricalDB -->|90 days data 5TB| SparkCluster
    SparkCluster -->|Trained model| ModelStore

    subgraph "Speed Layer - Fast Path"
        KafkaStreams[Kafka Streams<br/>Real-Time Features<br/>Last 5 views<br/>Last action timestamp]
        RedisCluster[Redis Cluster<br/>Feature Store<br/>50 shards<br/>sub-5ms latency]
    end

    Kafka -->|Real - time stream| KafkaStreams
    KafkaStreams -->|Update features| RedisCluster

    subgraph "Serving Layer"
        RecService[Recommendation Service<br/>Orchestration<br/>100k QPS]
        ModelServing[TensorFlow Serving<br/>Model Inference<br/>10-20ms latency]
        FAISSIndex[FAISS Index<br/>Item Similarity<br/>10M items<br/>5GB RAM]
    end

    RecService -->|Fetch user features| RedisCluster
    RecService -->|Load model| ModelServing
    ModelServing -->|Read artifacts| ModelStore
    RecService -->|Query similar items| FAISSIndex

    subgraph "Cache Layer"
        RecsCache[Redis Cache<br/>Pre-computed recs<br/>TTL 1 hour<br/>80% hit rate]
    end

    RecService -->|Cache popular recs| RecsCache
    Users -->|Request recommendations| RecService
    RecService -->|Top 10 items| Users
    style Kafka fill: #ff9999
    style RedisCluster fill: #99ccff
    style SparkCluster fill: #ffcc99
    style RecService fill: #99ff99
```

---

## Real-Time Serving Path

**Flow Explanation:**

This diagram shows the critical serving path that must complete in <50ms:

**Steps:**

1. **Client Request** (0ms): User requests recommendations (user_id, context)
2. **Cache Check** (2ms): Check Redis for pre-computed recommendations (80% hit rate)
3. **Parallel Feature Fetch** (5ms): Fetch user features + query FAISS for candidates
    - User features: Redis lookup (2ms)
    - Item candidates: FAISS ANN search (5ms)
4. **Model Inference** (20ms): TensorFlow Serving scores all candidates
5. **Ranking & Filtering** (3ms): Apply business rules, diversify, re-rank
6. **Response** (30ms total): Return top 10 recommendations

**Performance:**

- P50 latency: 25ms (cache hits)
- P99 latency: 45ms (cache misses + full inference)
- Throughput: 100k QPS (horizontally scaled)

**Benefits:**

- Meets <50ms SLA
- Cache dramatically reduces load (80% of requests skip inference)
- Parallel calls minimize sequential latency

```mermaid
graph LR
    Client[Client<br/>User Request<br/>user_id context]
    Client -->|1 . Request recs 0ms| RecAPI[Recommendation API<br/>100k QPS]
    RecAPI -->|2 . Check cache 2ms| Cache[Redis Cache<br/>Pre-computed Recs<br/>80% hit rate]
    Cache -->|Cache Hit 2ms| Client
    RecAPI -->|3a . Fetch features 2ms| FeatureStore[Redis Feature Store<br/>User features<br/>Last 5 views]
    RecAPI -->|3b . Get candidates 5ms| FAISS[FAISS Index<br/>Item Similarity<br/>Top 100 candidates]
    FeatureStore -->|User features| Inference[TensorFlow Serving<br/>Model Inference<br/>20ms]
    FAISS -->|Candidate items| Inference
    Inference -->|Scored candidates| Ranker[Ranking Service<br/>Filter + Diversify<br/>3ms]
    Ranker -->|Top 10 items| Client
    RecAPI -.->|Cache miss update| Cache
    style Cache fill: #99ccff
    style RecAPI fill: #99ff99
    style Inference fill: #ffcc99
```

---

## Batch Training Pipeline

**Flow Explanation:**

This diagram shows the daily batch training pipeline (20-hour cycle):

**Phase 1: Data Extraction** (4 hours)

1. Read 90 days of clickstream data from Cassandra (5 TB)
2. Filter bots, outliers, and noise
3. Output: Clean Parquet files in S3

**Phase 2: Feature Engineering** (6 hours)

1. Compute user features (total purchases, avg session length, preferred categories)
2. Compute item features (popularity, engagement rate, category distribution)
3. Output: Feature tables (Parquet in S3)

**Phase 3: Model Training** (10 hours)

1. **Collaborative Filtering:** ALS (Alternating Least Squares) on user-item matrix
2. **Deep Learning:** Two-tower neural network (user tower + item tower)
3. Output: Model artifacts (10 GB)

**Phase 4: Model Deployment** (1 hour)

1. Save to Model Store (S3/MLflow)
2. Gradual rollout (10% → 50% → 100% traffic)
3. Monitor A/B test metrics (CTR, conversion rate)

**Performance:**

- Total time: 21 hours (3-hour buffer before next cycle)
- Training data: 5 TB (90 days × 5 GB/day)
- Model size: 10 GB (compressed embeddings)

```mermaid
graph TB
subgraph "Data Sources"
Cassandra[(Cassandra/ClickHouse<br/>90 days data<br/>5 TB<br/>~450M events)]
end

subgraph "Phase 1 - Data Extraction 4h"
Extraction[Spark Job<br/>Filter bots/outliers<br/>Clean data]
RawData[(S3 Raw Data<br/>Parquet files<br/>5 TB)]
end

Cassandra -->|Read historical data|Extraction
Extraction -->|Write clean data|RawData

subgraph "Phase 2 - Feature Engineering 6h"
UserFeatures[Spark Job<br/>User Features<br/>Purchases, sessions, categories]
ItemFeatures[Spark Job<br/>Item Features<br/>Popularity, engagement, metadata]
FeatureTables[(S3 Feature Tables<br/>Parquet<br/>User table 500GB<br/>Item table 100GB)]
end

RawData -->|User events|UserFeatures
RawData -->|Item events|ItemFeatures
UserFeatures -->|Write|FeatureTables
ItemFeatures -->|Write|FeatureTables

subgraph "Phase 3 - Model Training 10h"
CollabFilter[Spark MLlib<br/>Collaborative Filtering<br/>ALS Algorithm<br/>6 hours]
DeepLearning[TensorFlow<br/>Two-Tower Network<br/>User/Item Embeddings<br/>4 hours]
ModelArtifacts[(Model Artifacts<br/>10 GB<br/>User embeddings 5GB<br/>Item embeddings 5GB)]
end

FeatureTables -->|User -Item matrix|CollabFilter
FeatureTables -->|Features| DeepLearning
CollabFilter -->|Trained model|ModelArtifacts
DeepLearning -->|Trained model|ModelArtifacts

subgraph "Phase 4 - Deployment 1h"
ModelStore[(Model Store<br/>S3/MLflow<br/>Version control<br/>10 versions retained)]
ABTest[A/B Test Controller<br/>10% → 50% → 100%<br/>Monitor CTR/Conversion]
TFServing[TensorFlow Serving<br/>Load new model<br/>10 instances]
end

ModelArtifacts -->|Upload|ModelStore
ModelStore -->|Gradual rollout| ABTest
ABTest -->|Load model|TFServing

subgraph "Monitoring"
Metrics[Prometheus<br/>CTR, Latency, Errors]
end

TFServing -.->|Export metrics|Metrics

style RawData fill: #ffcc99
style FeatureTables fill: #99ccff
style ModelStore fill: #ff9999
```

---

## Feature Store Architecture

**Flow Explanation:**

The Feature Store bridges the batch and speed layers, providing two types of features:

**Real-Time Features (Redis):**

- Updated continuously from Kafka stream
- Sub-5ms lookup latency
- Examples: last 5 viewed items, last action timestamp, current session duration
- TTL: 7 days (auto-expire old data)

**Batch Features (S3):**

- Computed daily from historical data
- Loaded into Redis periodically (every 1 hour)
- Examples: user embeddings, lifetime purchase count, preferred categories
- Size: 600 GB (100M users × 6 KB/user)

**Access Pattern:**

1. Recommendation Service requests features for `user_id`
2. Fetch from Redis (combines real-time + batch features in single key)
3. If cache miss: Fall back to S3 (slow path, rare)

**Performance:**

- Read latency: <2ms (Redis)
- Write throughput: 100k events/sec (Kafka Streams)
- Storage: 1 TB total (Redis + S3 snapshots)

```mermaid
graph TB
    subgraph "Streaming Path"
        Kafka[Kafka<br/>Clickstream Events<br/>100k events/sec]
        KafkaStreams[Kafka Streams<br/>Stateful Processing<br/>Windowed Aggregations]
    end

    Kafka -->|Real - time events| KafkaStreams

    subgraph "Batch Path"
        Spark[Spark Batch Job<br/>Daily Feature Computation<br/>User embeddings, stats]
        S3Batch[(S3 Feature Store<br/>Parquet snapshots<br/>600 GB<br/>Daily refresh)]
    end

    Spark -->|Write batch features| S3Batch

    subgraph "Feature Store - Redis Cluster"
        Redis1[Redis Shard 1<br/>user:0-2M]
        Redis2[Redis Shard 2<br/>user:2M-4M]
        Redis3[Redis Shard ...<br/>50 shards total]
    end

    KafkaStreams -->|Update real - time features| Redis1
    KafkaStreams -->|Update real - time features| Redis2
    KafkaStreams -->|Update real - time features| Redis3
    S3Batch -->|Periodic load every 1h| Redis1
    S3Batch -->|Periodic load every 1h| Redis2
    S3Batch -->|Periodic load every 1h| Redis3

    subgraph "Consumers"
        RecService[Recommendation Service<br/>Fetch features<br/>100k QPS]
    end

    RecService -->|GET user:12345 sub - 2ms| Redis1
    RecService -->|GET user:234567 sub - 2ms| Redis2
    RecService -->|GET user:456789 sub - 2ms| Redis3

    subgraph "Feature Schema"
        Schema["Key: user:{user_id}<br/>Value (JSON):<br/>{<br/> last_5_items: [123, 456, ...],<br/> last_action_ts: 1698765432,<br/> user_embedding: [0.12, -0.34, ...],<br/> total_purchases: 42,<br/> preferred_categories: [5, 12, 8]<br/>}<br/>TTL: 7 days"]
    end

    Redis1 -.->|Example| Schema
    style Kafka fill: #ff9999
    style Redis1 fill: #99ccff
    style Redis2 fill: #99ccff
    style Redis3 fill: #99ccff
    style S3Batch fill: #ffcc99
```

---

## Model Serving Infrastructure

**Flow Explanation:**

This diagram shows how trained models are served at scale:

**Components:**

1. **Model Store (S3/MLflow):** Centralized storage for all model versions
2. **TensorFlow Serving:** Stateless inference servers (10+ instances)
3. **Load Balancer:** Distributes inference requests across instances
4. **Model Registry:** Tracks model versions, A/B test assignments, performance metrics

**Model Loading:**

- Each TensorFlow Serving instance loads model into RAM (10 GB)
- Models are versioned (e.g., v42, v43)
- Gradual rollout: 10% of traffic tests new model before full rollout

**Inference Flow:**

1. Recommendation Service sends gRPC request with user/item features
2. Load balancer routes to available TensorFlow Serving instance
3. Model computes scores for all candidate items (batch inference)
4. Returns top-N scored items

**Performance:**

- Inference latency: 10-20ms per request
- Throughput: 10k QPS per instance (100k total with 10 instances)
- GPU acceleration: Optional (4x faster but 10x more expensive)

```mermaid
graph TB
    subgraph "Model Store"
        S3[(S3/MLflow<br/>Model Registry<br/>Version control<br/>v1, v2, ..., v50)]
        Metadata[Model Metadata<br/>Training date, accuracy<br/>A/B test results]
    end

    S3 -.->|Metadata| Metadata

    subgraph "TensorFlow Serving Cluster"
        LB[gRPC Load Balancer<br/>Round-robin<br/>Health checks]
        TFS1[TensorFlow Serving 1<br/>Model v42 prod 90%<br/>10 GB RAM<br/>10k QPS]
        TFS2[TensorFlow Serving 2<br/>Model v43 canary 10%<br/>10 GB RAM<br/>10k QPS]
        TFS3[TensorFlow Serving ...<br/>8 more instances<br/>Total: 100k QPS]
    end

    S3 -->|Load model v42| TFS1
    S3 -->|Load model v43 canary| TFS2
    S3 -->|Load model v42| TFS3
    LB -->|Route 90%| TFS1
    LB -->|Route 10% canary| TFS2
    LB -->|Route| TFS3

    subgraph "Recommendation Service"
        RecAPI[Recommendation API<br/>Orchestrator<br/>Prepares features]
    end

    RecAPI -->|gRPC inference request| LB
    LB -->|Scored items| RecAPI

    subgraph "A/B Test Controller"
        ABController[A/B Controller<br/>User assignment<br/>Metric tracking]
    end

    ABController -.->|Assign user to model| RecAPI
    ABController -.->|Monitor CTR/Conversion| Metadata

    subgraph "Monitoring"
        Prometheus[Prometheus<br/>Inference latency<br/>Error rate<br/>Model performance]
    end

    TFS1 -.->|Export metrics| Prometheus
    TFS2 -.->|Export metrics| Prometheus
    style S3 fill: #ffcc99
    style TFS1 fill: #99ff99
    style TFS2 fill: #ff9999
    style RecAPI fill: #99ccff
```

---

## Candidate Generation Strategy

**Flow Explanation:**

This diagram shows how the system generates candidate items using multiple strategies:

**4 Parallel Strategies:**

1. **Collaborative Filtering** (50 candidates, 20ms):
    - "Users who liked items similar to yours also liked..."
    - Uses ALS-trained embeddings
    - High relevance but prone to filter bubble

2. **Content-Based** (30 candidates, 5ms):
    - Items similar to user's past interactions (based on metadata)
    - Uses TF-IDF or BERT embeddings
    - Good for new items (cold start)

3. **Item-to-Item Similarity** (50 candidates, 5ms):
    - Items similar to currently viewed item
    - Uses FAISS ANN search
    - Effective for browse/discovery

4. **Trending/Popular** (20 candidates, 2ms):
    - Globally trending items in user's preferred categories
    - Pre-computed daily
    - Ensures exploration

**Blending:**

- Total: ~150 candidates
- Model scores all candidates
- Re-rank and diversify
- Return top 10

**Benefits:**

- Multi-strategy prevents filter bubble
- Parallel execution minimizes latency (20ms = max of all strategies)
- Redundancy improves robustness

```mermaid
graph TB
    User[User Request<br/>user_id: 12345<br/>context: browsing electronics]
    User -->|Generate candidates| CandidateGen[Candidate Generator<br/>Orchestrator]

    subgraph "Strategy 1 - Collaborative Filtering"
        CollabModel[ALS Model<br/>User embeddings<br/>Item embeddings]
        CollabCandidates[50 Candidates<br/>Latency: 20ms<br/>High relevance]
    end

    CandidateGen -->|Fetch user embedding| CollabModel
    CollabModel -->|Cosine similarity| CollabCandidates

    subgraph "Strategy 2 - Content-Based"
        ContentModel[TF-IDF/BERT<br/>Item metadata<br/>Title, description, tags]
        ContentCandidates[30 Candidates<br/>Latency: 5ms<br/>Good for cold start]
    end

    CandidateGen -->|Query similar metadata| ContentModel
    ContentModel -->|Text similarity| ContentCandidates

    subgraph "Strategy 3 - Item-to-Item"
        FAISS[FAISS Index<br/>10M items<br/>128-dim vectors]
        ItemCandidates[50 Candidates<br/>Latency: 5ms<br/>Browse discovery]
    end

    CandidateGen -->|ANN search| FAISS
    FAISS -->|Top - K similar| ItemCandidates

    subgraph "Strategy 4 - Trending"
        TrendingCache[Redis Cache<br/>Pre-computed trending<br/>By category]
        TrendingCandidates[20 Candidates<br/>Latency: 2ms<br/>Exploration]
    end

    CandidateGen -->|Fetch trending| TrendingCache
    TrendingCache -->|Popular items| TrendingCandidates

subgraph "Candidate Pool"
Pool[Total: ~150 Candidates<br/>Deduplicate<br/>Remove already interacted]
end

CollabCandidates -->|Merge|Pool
ContentCandidates -->|Merge|Pool
ItemCandidates -->|Merge|Pool
TrendingCandidates -->|Merge|Pool

subgraph "Ranking Model"
RankModel[TensorFlow Model<br/>Score all candidates<br/>Predicted CTR/Conversion]
TopN[Top 10 Items<br/>Diversified<br/>Max 3 per category]
end

Pool -->|150 candidates|RankModel
RankModel -->|Ranked scores|TopN

TopN -->|Return recommendations|User

style Pool fill: #ffcc99
style RankModel fill: #99ff99
```

---

## Item-to-Item Similarity (FAISS/ANN)

**Flow Explanation:**

This diagram shows how item-to-item similarity search works using FAISS:

**Index Building (Offline, once per day):**

1. Extract item embeddings from trained model (10M items × 128 dims)
2. Build FAISS index using HNSW algorithm
3. Load index into RAM (5 GB)

**Query Flow (Online, <5ms):**

1. User views item `item_id=42`
2. Fetch item embedding from FAISS index (128-dim vector)
3. Run ANN search to find top-50 similar items (cosine distance)
4. Return similar item IDs

**HNSW Algorithm:**

- Hierarchical graph structure (multi-layer skip list)
- Trade-off: 95% recall (acceptable) for 10x speedup
- Exact K-NN would take 100ms+ for 10M items

**Performance:**

- Query latency: <5ms per lookup
- Index size: 5 GB (fits in RAM)
- Throughput: 100k QPS (horizontally scaled)

**Partitioning:**

- Split by category (electronics, fashion, books)
- Each partition: 1M items, 500 MB
- Reduces search space for faster queries

```mermaid
graph TB
    subgraph "Index Building - Offline Daily"
        TrainedModel[(Trained Model<br/>Item Embeddings<br/>10M items<br/>128 dims each)]
        FAISSBuilder[FAISS Index Builder<br/>HNSW Algorithm<br/>Build time: 1 hour]
        FAISSIndex[(FAISS Index<br/>5 GB RAM<br/>Hierarchical graph)]
    end

    TrainedModel -->|Extract embeddings| FAISSBuilder
    FAISSBuilder -->|Build index| FAISSIndex

    subgraph "Query Flow - Online sub-5ms"
        UserView[User Views Item<br/>item_id: 42<br/>category: electronics]
        QueryAPI[Query API<br/>Fetch embedding<br/>Run ANN search]
        SimilarItems[Top-50 Similar Items<br/>Cosine distance<br/>Score: 0.85, 0.82, ...]
    end

    UserView -->|Request similar items| QueryAPI
    QueryAPI -->|Lookup embedding| FAISSIndex
    FAISSIndex -->|ANN search sub - 5ms| SimilarItems

    subgraph "HNSW Algorithm"
        HNSWDiagram["Layer 2 - Coarse<br/> [Item A] --- [Item B]<br/>Layer 1 - Medium<br/> [Item 1] --- [Item 2] --- [Item 3]<br/>Layer 0 - Fine<br/> [All 10M items]<br/><br/>Search: Start at top layer, descend to neighbors"]
    end

    FAISSIndex -.->|Algorithm| HNSWDiagram

    subgraph "Performance Metrics"
        Metrics["Query Latency: <5ms<br/>Recall@50: 95%<br/>Index Size: 5 GB<br/>Build Time: 1 hour daily<br/>Throughput: 100k QPS"]
    end

    QueryAPI -.->|Metrics| Metrics

    subgraph "Partitioning Strategy"
        Partition1[FAISS Index 1<br/>Electronics<br/>1M items, 500 MB]
        Partition2[FAISS Index 2<br/>Fashion<br/>1M items, 500 MB]
        Partition3[FAISS Index ...<br/>10 categories<br/>10M items total]
    end

    FAISSIndex -.->|Partition| Partition1
    FAISSIndex -.->|Partition| Partition2
    FAISSIndex -.->|Partition| Partition3
    style FAISSIndex fill: #99ccff
    style QueryAPI fill: #99ff99
    style SimilarItems fill: #ffcc99
```

---

## Redis Feature Store Sharding

**Flow Explanation:**

This diagram shows how the Redis Feature Store is sharded for horizontal scalability:

**Sharding Strategy:**

- **Key:** `user:{user_id}` (e.g., `user:12345`)
- **Hash Function:** CRC32(user_id) % 50 = shard_id
- **50 Shards:** Each shard handles ~2M users (100M users / 50)

**Per-Shard Architecture:**

- **1 Master:** Handles all writes (real-time feature updates)
- **2 Replicas:** Handle reads (recommendation serving)
- **Async Replication:** Master → Replicas (eventual consistency acceptable)

**Throughput:**

- Writes: 100k events/sec / 50 shards = 2k writes/sec per shard
- Reads: 1.1M lookups/sec / 50 shards = 22k reads/sec per shard
- Per-shard capacity: 25k ops/sec (within Redis limit)

**Failover:**

- If master fails, promote replica to master (Redis Sentinel)
- Downtime: <5 seconds
- Data loss: Minimal (replication lag <100ms)

**Benefits:**

- Horizontal scalability (add more shards as users grow)
- Read scaling (2 replicas per master = 3x read capacity)
- High availability (automatic failover)

```mermaid
graph TB
    subgraph "Kafka Streams - Writers"
        Writer1[Kafka Streams 1<br/>Process events<br/>Update features]
        Writer2[Kafka Streams 2]
        Writer3[Kafka Streams 3]
    end

    subgraph "Redis Shard 0 - user:0-2M"
        Master0[Redis Master 0<br/>Write: 2k/sec<br/>Data: 12 GB]
        Replica01[Redis Replica 0-1<br/>Read: 22k/sec]
        Replica02[Redis Replica 0-2<br/>Read: 22k/sec]
    end

    Writer1 -->|Hash user:12345 to shard 0| Master0
    Master0 -.->|Async replication| Replica01
    Master0 -.->|Async replication| Replica02

    subgraph "Redis Shard 1 - user:2M-4M"
        Master1[Redis Master 1<br/>Write: 2k/sec<br/>Data: 12 GB]
        Replica11[Redis Replica 1-1<br/>Read: 22k/sec]
        Replica12[Redis Replica 1-2<br/>Read: 22k/sec]
    end

    Writer2 -->|Hash user:234567 to shard 1| Master1
    Master1 -.->|Async replication| Replica11
    Master1 -.->|Async replication| Replica12

    subgraph "Redis Shard 49 - user:98M-100M"
        Master49[Redis Master 49<br/>Write: 2k/sec<br/>Data: 12 GB]
        Replica491[Redis Replica 49-1<br/>Read: 22k/sec]
        Replica492[Redis Replica 49-2<br/>Read: 22k/sec]
    end

    Writer3 -->|Hash user:9876543 to shard 49| Master49
    Master49 -.->|Async replication| Replica491
    Master49 -.->|Async replication| Replica492

    subgraph "Recommendation Service - Readers"
        Reader1[Rec Service 1<br/>Fetch features<br/>20k QPS]
        Reader2[Rec Service 2<br/>20k QPS]
        Reader3[Rec Service ...<br/>Total: 100k QPS]
    end

    Reader1 -->|GET user:12345| Replica01
    Reader1 -->|GET user:234567| Replica11
    Reader2 -->|GET user:9876543| Replica491
    Reader3 -->|Consistent hashing| Replica02

    subgraph "Redis Sentinel - Failover"
        Sentinel[Redis Sentinel<br/>Monitor masters<br/>Auto-failover<br/>Promote replica]
    end

    Sentinel -.->|Monitor| Master0
    Sentinel -.->|Monitor| Master1
    Sentinel -.->|Monitor| Master49
    style Master0 fill: #ff9999
    style Master1 fill: #ff9999
    style Master49 fill: #ff9999
    style Replica01 fill: #99ccff
    style Replica11 fill: #99ccff
    style Replica491 fill: #99ccff
```

---

## Multi-Region Deployment

**Flow Explanation:**

This diagram shows the global deployment across 3 regions for low latency:

**Regions:**

1. **US-East-1** (Primary): Batch training, model store, full feature store
2. **EU-West-1** (Secondary): Read replicas, cached recommendations
3. **AP-South-1** (Secondary): Read replicas, cached recommendations

**Data Replication:**

- **Kafka:** Multi-region replication (MirrorMaker2) for clickstream events
- **Redis:** Cross-region async replication for feature store
- **Model Store (S3):** Replicated to all regions for low-latency model loading

**Request Routing:**

- **GeoDNS:** Routes users to nearest region
- **All writes:** Routed to US-East-1 (primary)
- **Reads:** Served locally in each region

**Trade-offs:**

- Write latency: Higher for EU/AP users (cross-region network)
- Read latency: Low for all users (local cache)
- Consistency: Eventual (replication lag <1 second)

**Failover:**

- If primary region fails, promote EU-West-1 to primary
- Downtime: <1 minute
- Data loss: Minimal (Kafka replication lag <1 second)

```mermaid
graph TB
    subgraph "US-East-1 - Primary"
        DNS_US[GeoDNS<br/>Route US traffic]
        RecService_US[Recommendation Service<br/>100k QPS]
        Redis_US[Redis Feature Store<br/>50 shards<br/>Write master]
        Kafka_US[Kafka<br/>100k events/sec<br/>Write master]
        Spark_US[Spark Cluster<br/>Batch Training<br/>Daily]
        S3_US[(S3 Model Store<br/>Primary<br/>10 GB)]
    end

    DNS_US --> RecService_US
    RecService_US --> Redis_US
    Kafka_US --> Redis_US
    Spark_US --> S3_US

    subgraph "EU-West-1 - Secondary"
        DNS_EU[GeoDNS<br/>Route EU traffic]
        RecService_EU[Recommendation Service<br/>50k QPS]
        Redis_EU[Redis Feature Store<br/>Read replicas<br/>Async replication]
        Kafka_EU[Kafka<br/>MirrorMaker2<br/>Read replicas]
        S3_EU[(S3 Model Store<br/>Replica<br/>10 GB)]
    end

    DNS_EU --> RecService_EU
    RecService_EU --> Redis_EU
    Kafka_US -.->|Cross - region replication 100ms lag| Kafka_EU
    Redis_US -.->|Cross - region replication 1s lag| Redis_EU
    S3_US -.->|S3 replication| S3_EU

    subgraph "AP-South-1 - Secondary"
        DNS_AP[GeoDNS<br/>Route AP traffic]
        RecService_AP[Recommendation Service<br/>50k QPS]
        Redis_AP[Redis Feature Store<br/>Read replicas<br/>Async replication]
        Kafka_AP[Kafka<br/>MirrorMaker2<br/>Read replicas]
        S3_AP[(S3 Model Store<br/>Replica<br/>10 GB)]
    end

    DNS_AP --> RecService_AP
    RecService_AP --> Redis_AP
    Kafka_US -.->|Cross - region replication 200ms lag| Kafka_AP
    Redis_US -.->|Cross - region replication 2s lag| Redis_AP
    S3_US -.->|S3 replication| S3_AP

    subgraph "Global Users"
        Users_US[Users - Americas<br/>40M DAU]
        Users_EU[Users - Europe<br/>35M DAU]
        Users_AP[Users - Asia Pacific<br/>25M DAU]
    end

    Users_US -->|Low latency 20ms| DNS_US
    Users_EU -->|Low latency 30ms| DNS_EU
    Users_AP -->|Low latency 50ms| DNS_AP

    subgraph "Failover Strategy"
        FailoverNote["If US-East-1 fails:<br/>Promote EU-West-1 to primary<br/>Downtime: less than 1 minute"]
    end

    DNS_US -.->|Failover scenario| FailoverNote
    style Kafka_US fill: #ff9999
    style Redis_US fill: #99ccff
    style S3_US fill: #ffcc99
```

---

## Pre-Computed Recommendations Cache

**Flow Explanation:**

This diagram shows the caching strategy for pre-computed recommendations:

**Cache Workflow:**

**Batch Pre-Computation (Every 1 hour):**

1. Background job computes recommendations for all users
2. Stores top 10 recommendations in Redis cache
3. TTL: 1 hour (refreshes every hour)

**Serving:**

1. User requests recommendations
2. Check cache first (key: `recs:{user_id}`)
3. **Cache Hit (80% of requests):** Return instantly (<2ms)
4. **Cache Miss (20%):** Compute in real-time (30ms)

**Cache Segmentation:**

- **VIP Users (10M users, 10%):** Always cached (updated every 10 minutes)
- **Active Users (40M users, 40%):** Cached during peak hours
- **Inactive Users (50M users, 50%):** Not cached (low request rate)

**Benefits:**

- **80% cache hit rate** reduces load on model serving by 5x
- **<2ms latency** for cache hits (vs 30ms for cold path)
- **Cost savings:** Avoid expensive model inference for most requests

**Storage:**

- 50M cached users × 10 items/user × 100 bytes/item = 50 GB
- Redis cluster: 10 shards × 5 GB/shard

```mermaid
graph TB
subgraph "Background Pre-Computation Job"
CronJob[Airflow DAG<br/>Runs every 1 hour]
BatchService[Batch Rec Service<br/>Compute recs for all users<br/>50M users × 30ms = 500 hours<br/>Parallelized: 500 workers = 1 hour]
end

CronJob -->|Trigger|BatchService

subgraph "Feature Store"
FeatureStore[Redis Feature Store<br/>User features]
end

BatchService -->|Fetch features|FeatureStore

subgraph "Model Serving"
ModelAPI[TensorFlow Serving<br/>Batch inference<br/>10k users/sec]
end

BatchService -->|Batch inference|ModelAPI

subgraph "Cache - Redis Cluster"
Cache1[Redis Cache Shard 1<br/>recs:0-5M<br/>5 GB]
Cache2[Redis Cache Shard 2<br/>recs:5M-10M<br/>5 GB]
Cache3[Redis Cache ...<br/>10 shards<br/>50 GB total]
end

BatchService -->|Write recs TTL 1h|Cache1
BatchService -->|Write recs TTL 1h|Cache2
BatchService -->|Write recs TTL 1h|Cache3

subgraph "Serving Path"
UserRequest[User Request<br/>user_id: 12345]
RecService[Recommendation Service<br/>Check cache first]
end

UserRequest -->|Request recs|RecService

RecService -->|1 . Check cache 2ms| Cache1
Cache1 -->|Cache Hit 80%|UserRequest

RecService -->|2 . Cache Miss 20%|RealTime[Real-Time Computation<br/>Feature fetch + Model inference<br/>30ms]
RealTime -->|Computed recs|UserRequest
RealTime -.->|Update cache|Cache1

subgraph "Cache Segmentation"
Segment["VIP Users 10M always cached<br/>Active Users 40M peak hours<br/>Inactive Users 50M not cached"]
end

BatchService -.->|Prioritize| Segment

subgraph "Performance"
Perf["Cache Hit Rate: 80%<br/>Cache Latency: <2ms<br/>Cold Path Latency: 30ms<br/>Cost Savings: 5x<br/>Storage: 50 GB"]
end

Cache1 -.->|Metrics|Perf

style Cache1 fill: #99ccff
style Cache2 fill: #99ccff
style Cache3 fill: #99ccff
style RecService fill:#99ff99
```

---

## A/B Testing Infrastructure

**Flow Explanation:**

This diagram shows the A/B testing infrastructure for model experiments:

**Experiment Flow:**

1. **User Request:** User hits Recommendation Service
2. **Experiment Assignment:** A/B controller assigns user to experiment group
    - Control Group (90%): Model v42 (current production)
    - Treatment Group (10%): Model v43 (new candidate)
3. **Feature Fetch + Model Inference:** Based on group assignment
4. **Metric Tracking:** Log experiment ID, user ID, recommendations served, user actions

**Metric Collection:**

- **Impressions:** Recommendations shown to user
- **Clicks:** User clicked on recommendation
- **Conversions:** User purchased/engaged with recommended item
- **Latency:** Time to serve recommendations

**Analysis:**

- Compare CTR (Click-Through Rate) between control and treatment
- Statistical significance test (p-value < 0.05)
- If treatment wins, promote to production (gradual rollout)

**Benefits:**

- Safe model deployment (catch regressions early)
- Data-driven decisions (measure actual user impact)
- Parallel experiments (test multiple models simultaneously)

```mermaid
graph TB
    subgraph "Users"
        User[User Request<br/>user_id: 12345]
    end

    User -->|Request recs| RecService[Recommendation Service]

    subgraph "A/B Test Controller"
        ABController[A/B Controller<br/>Experiment Assignment<br/>Hash user_id to group]
        ExperimentConfig[(Experiment Config<br/>experiment_id: exp_42<br/>control: model_v42 90%<br/>treatment: model_v43 10%)]
    end

    RecService -->|1 . Get experiment group| ABController
    ABController -->|Load config| ExperimentConfig
    ABController -->|Assign group| RecService

    subgraph "Model Serving"
        ModelV42[TensorFlow Serving<br/>Model v42 Control<br/>90% traffic]
        ModelV43[TensorFlow Serving<br/>Model v43 Treatment<br/>10% traffic]
    end

    RecService -->|90% traffic| ModelV42
    RecService -->|10% traffic| ModelV43
    ModelV42 -->|Recommendations| RecService
    ModelV43 -->|Recommendations| RecService
    RecService -->|Return recs| User

    subgraph "Metric Collection"
        Kafka[Kafka<br/>Experiment Events<br/>Impressions, Clicks, Conversions]
        ClickHouse[(ClickHouse<br/>Analytics DB<br/>Aggregated metrics)]
    end

    RecService -.->|Log event: user, group, recs, action| Kafka
    User -.->|Click/Convert| Kafka
    Kafka -->|Stream to DB| ClickHouse

    subgraph "Analysis Dashboard"
        Dashboard["Experiment: exp_42<br/><br/>Control Group v42:<br/>CTR: 5.2%<br/>Conversion: 2.1%<br/>Latency: 28ms<br/><br/>Treatment Group v43:<br/>CTR: 5.8% (+11.5%)<br/>Conversion: 2.4% (+14.3%)<br/>Latency: 30ms<br/><br/>Result: Treatment WINS<br/>p-value: 0.001<br/>Decision: Promote to prod"]
    end

    ClickHouse -->|Query metrics| Dashboard

    subgraph "Gradual Rollout"
        Rollout["If treatment wins:<br/>Day 1: 10% traffic<br/>Day 2: 25% traffic<br/>Day 3: 50% traffic<br/>Day 4: 100% traffic"]
    end

    Dashboard -.->|Promote| Rollout
    Rollout -.->|Update config| ExperimentConfig
    style ABController fill: #99ccff
    style ModelV43 fill: #ff9999
    style Dashboard fill: #99ff99
```

---

## Cold Start Handling

**Flow Explanation:**

This diagram shows how the system handles cold start for new users and new items:

**New User Cold Start:**

1. **Onboarding:** Explicit preference collection during signup (select interests)
2. **Default Recommendations:** Serve globally popular items in selected categories
3. **Aggressive Exploration:** Show diverse content (10 categories) to learn preferences fast
4. **Fast Learning:** After 10 interactions, switch to personalized recommendations

**New Item Cold Start:**

1. **Content-Based Features:** Use item metadata (title, description, category) for similarity
2. **Editorial Curation:** Manual picks for high-quality new items
3. **Exploration Budget:** Show new items to 5% of users, monitor engagement
4. **Bootstrapping:** If engagement > threshold, promote to wider audience

**Hybrid Approach:**

- First 7 days: 70% exploration (diverse content) + 30% exploitation (popular items)
- After 7 days: 30% exploration + 70% personalized

**Benefits:**

- Prevents poor experience for new users (avoid "empty feed")
- Gives new items visibility (avoid "rich get richer")
- Fast learning (10 interactions sufficient for basic personalization)

```mermaid
graph TB
    subgraph "New User Scenario"
        NewUser[New User<br/>user_id: 999999<br/>No history<br/>0 interactions]
    end

    NewUser -->|Request recs| ColdStartHandler[Cold Start Handler<br/>Detect new user]

    subgraph "Onboarding Flow"
        Onboard[Onboarding Survey<br/>Select 3 categories:<br/>Fashion, Technology, Sports]
    end

    ColdStartHandler -->|1 . Show onboarding| Onboard
    Onboard -->|User selects interests| ColdStartHandler

    subgraph "Strategy 1 - Popular Items"
        Popular[Global Popular Items<br/>Filter by selected categories<br/>Pre-computed daily<br/>30 candidates]
    end

    ColdStartHandler -->|2 . Fetch popular 30%| Popular

    subgraph "Strategy 2 - Exploration"
        Explore[Diverse Exploration<br/>Sample from 10 categories<br/>Ensure variety<br/>70 candidates]
    end

    ColdStartHandler -->|3 . Fetch diverse 70%| Explore

    subgraph "Blended Recommendations"
        Blend[Blended Recs<br/>70% Exploration<br/>30% Popular<br/>Total: 100 candidates]
    end

    Popular -->|Merge| Blend
    Explore -->|Merge| Blend
    Blend -->|Return top 10| NewUser

    subgraph "Fast Learning"
        Interaction[User Interactions<br/>After 10 interactions:<br/>Switch to personalized recs]
    end

    NewUser -.->|Click, view, purchase| Interaction
    Interaction -.->|Update features| FeatureStore[Redis Feature Store]

    subgraph "New Item Scenario"
        NewItem[New Item<br/>item_id: 888888<br/>Created today<br/>0 engagement]
    end

    subgraph "Strategy 3 - Content-Based"
        ContentBased[Content-Based Filter<br/>Item metadata:<br/>title, description, category<br/>Find similar items]
    end

    NewItem -->|Recommend similar| ContentBased

    subgraph "Strategy 4 - Editorial"
        Editorial[Editorial Curation<br/>Manually curated<br/>High-quality new items<br/>Editors pick]
    end

    NewItem -->|Featured items| Editorial

    subgraph "Strategy 5 - Exploration Budget"
        ExplorationBudget["Exploration Budget<br/>Show new items to 5 percent of users<br/>Monitor engagement<br/>If CTR over 3 percent then Promote"]
    end

    NewItem -->|Expose to sample| ExplorationBudget
    ExplorationBudget -.->|High engagement| Promote[Promote to Wider Audience<br/>Increase exposure to 20 percent]

    subgraph "Learning Timeline"
        TimelinePhases["Day 1-7 - 70 percent Exploration<br/>Day 8 and beyond - 30 percent Exploration<br/>Gradual shift to personalized"]
    end

    ColdStartHandler -.->|Adjust over time| TimelinePhases
    style ColdStartHandler fill: #99ccff
    style Blend fill: #99ff99
    style ContentBased fill: #ffcc99
```

---

## Model Training DAG (Airflow)

**Flow Explanation:**

This diagram shows the Airflow DAG (Directed Acyclic Graph) for daily model training:

**DAG Structure (21-hour pipeline):**

**Phase 1: Data Extraction (4 hours)**

- Task 1: Read clickstream data from Cassandra (90 days, 5 TB)
- Task 2: Filter bots and outliers
- Task 3: Write clean data to S3 (Parquet)

**Phase 2: Feature Engineering (6 hours, parallel)**

- Task 4a: Compute user features (Spark job)
- Task 4b: Compute item features (Spark job)
- Task 5: Merge features and write to S3

**Phase 3: Model Training (10 hours, parallel)**

- Task 6a: Train collaborative filtering model (ALS, Spark MLlib, 6 hours)
- Task 6b: Train deep learning model (TensorFlow, 4 hours)
- Task 7: Combine models and save artifacts

**Phase 4: Model Deployment (1 hour, sequential)**

- Task 8: Upload model to Model Store (S3/MLflow)
- Task 9: Run A/B test on 10% traffic
- Task 10: Monitor metrics for 30 minutes
- Task 11: Gradual rollout to 100% (if metrics pass)

**Scheduling:**

- Runs daily at 2 AM UTC
- SLA: Must complete within 24 hours
- Retry: Up to 3 retries per task on failure
- Alerts: Email/Slack on task failure

**Benefits:**

- Fully automated pipeline
- Clear dependencies (tasks run in correct order)
- Fault tolerance (retries on failure)
- Observability (DAG visualization, logs, metrics)

```mermaid
graph TB
    subgraph "Phase 1 - Data Extraction 4h"
        Start[Start DAG<br/>Daily 2 AM UTC]
        T1[Task 1: Read Cassandra<br/>90 days clickstream<br/>5 TB data<br/>Duration: 2h]
        T2[Task 2: Filter Bots<br/>Remove outliers<br/>Spark job<br/>Duration: 1h]
        T3[Task 3: Write to S3<br/>Parquet format<br/>Duration: 1h]
    end

    Start --> T1
    T1 --> T2
    T2 --> T3

    subgraph "Phase 2 - Feature Engineering 6h"
        T4a[Task 4a: User Features<br/>Total purchases, sessions<br/>Spark job<br/>Duration: 4h]
        T4b[Task 4b: Item Features<br/>Popularity, engagement<br/>Spark job<br/>Duration: 3h]
        T5[Task 5: Merge Features<br/>Join user + item tables<br/>Duration: 2h]
    end

    T3 --> T4a
    T3 --> T4b
    T4a --> T5
    T4b --> T5

    subgraph "Phase 3 - Model Training 10h"
        T6a[Task 6a: Collaborative Filter<br/>ALS Algorithm<br/>Spark MLlib<br/>Duration: 6h]
        T6b[Task 6b: Deep Learning<br/>Two-Tower Network<br/>TensorFlow<br/>Duration: 4h]
        T7[Task 7: Combine Models<br/>Save artifacts 10 GB<br/>Duration: 30min]
    end

    T5 --> T6a
    T5 --> T6b
    T6a --> T7
    T6b --> T7

subgraph "Phase 4 - Deployment 1h"
T8[Task 8: Upload to S3<br/>Model Store MLflow<br/>Duration: 10min]
T9[Task 9: A/B Test<br/>Deploy to 10% traffic<br/>Duration: 10min]
T10[Task 10: Monitor Metrics<br/>CTR, Latency, Errors<br/>Duration: 30min]
T11[Task 11: Gradual Rollout<br/>If metrics pass<br/>10% → 50% → 100%<br/>Duration: 10min]
end

T7 --> T8
T8 --> T9
T9 --> T10
T10 --> T11

subgraph "Success/Failure"
Success[Success<br/>Model deployed to prod<br/>Total: 21h]
Failure[Failure<br/>Retry up to 3 times<br/>Alert: Email/Slack]
end

T11 -->|Metrics pass|Success
T10 -->|Metrics fail|Failure
Failure -.->|Retry|T1

subgraph "Monitoring"
Monitor["Airflow UI:<br/>DAG visualization<br/>Task logs<br/>Duration metrics<br/>SLA monitoring"]
end

Success -.->|Export metrics|Monitor
Failure -.->|Export metrics|Monitor

style T1 fill: #ffcc99
style T5 fill: #99ccff
style T7 fill:#ff9999
style Success fill: #99ff99
```

---

## Monitoring and Alerting Dashboard

**Flow Explanation:**

This diagram shows the comprehensive monitoring and alerting infrastructure:

**Key Metrics Monitored:**

**1. Serving Latency:**

- P50, P95, P99 latency for recommendation requests
- Target: P99 < 50ms
- Alert: P99 > 100ms

**2. Cache Hit Rate:**

- Percentage of requests served from cache
- Target: > 80%
- Alert: < 70%

**3. Model Performance:**

- CTR (Click-Through Rate): Target 5-10%
- Conversion Rate: Target 2-5%
- Alert: CTR < 3% (model degradation)

**4. Feature Store Performance:**

- Redis latency: Target < 5ms
- Redis QPS: Target 1.1M
- Alert: Latency > 10ms

**5. Batch Training Status:**

- Training duration: Target < 20 hours
- Training success rate: Target 100%
- Alert: Duration > 24 hours or failure

**Alerting Channels:**

- PagerDuty (critical alerts, on-call rotation)
- Slack (warning alerts, team channel)
- Email (daily summary reports)

**Dashboard Tools:**

- Grafana (real-time metrics visualization)
- Kibana (log analysis)
- Custom dashboard (business metrics: CTR, revenue impact)

```mermaid
graph TB
    subgraph "Data Sources"
        RecService[Recommendation Service<br/>Export metrics:<br/>Latency, QPS, Errors]
        FeatureStore[Redis Feature Store<br/>Export metrics:<br/>Latency, Hit rate, QPS]
        ModelServing[TensorFlow Serving<br/>Export metrics:<br/>Inference time, GPU usage]
        Kafka[Kafka<br/>Export metrics:<br/>Lag, Throughput]
        Airflow[Airflow DAG<br/>Export metrics:<br/>Training duration, Success rate]
    end

    subgraph "Metrics Collection"
        Prometheus[Prometheus<br/>Scrape metrics<br/>15s interval<br/>7-day retention]
    end

    RecService -.->|/metrics endpoint| Prometheus
    FeatureStore -.->|/metrics endpoint| Prometheus
    ModelServing -.->|/metrics endpoint| Prometheus
    Kafka -.->|JMX exporter| Prometheus
    Airflow -.->|Airflow exporter| Prometheus

    subgraph "Log Collection"
        Logs[Application Logs<br/>JSON format<br/>Error, Warning, Info]
        Elasticsearch[Elasticsearch<br/>Centralized logs<br/>30-day retention]
    end

    RecService -.->|Stream logs| Logs
    ModelServing -.->|Stream logs| Logs
    Logs -->|Logstash| Elasticsearch

    subgraph "Business Metrics"
        ClickHouse[(ClickHouse<br/>Analytics DB<br/>CTR, Conversion, Revenue)]
    end

    Kafka -.->|Stream user events| ClickHouse

    subgraph "Visualization - Grafana"
        Dashboard1["Dashboard 1 - Serving Health<br/>P99 Latency 45ms OK<br/>Cache Hit Rate 82 percent OK<br/>QPS 98k<br/>Error Rate 0.01 percent"]
        Dashboard2["Dashboard 2 - Feature Store<br/>Redis Latency 2ms OK<br/>Redis QPS 1.1M<br/>Memory Usage 75 percent<br/>Replication Lag 50ms"]
        Dashboard3["Dashboard 3 - Model Performance<br/>CTR 5.8 percent OK<br/>Conversion 2.4 percent OK<br/>Inference Time 18ms<br/>Model Version v43"]
    end

    Prometheus -->|Query metrics| Dashboard1
    Prometheus -->|Query metrics| Dashboard2
    ClickHouse -->|Query business metrics| Dashboard3

    subgraph "Alerting - Alertmanager"
        Alertmanager[Prometheus Alertmanager<br/>Rule engine<br/>Deduplication<br/>Routing]
        Rules["Alert Rules<br/>1. P99 Latency over 100ms<br/>2. Cache Hit Rate under 70 percent<br/>3. CTR under 3 percent<br/>4. Training Duration over 24h<br/>5. Redis Down"]
    end

    Prometheus -->|Trigger alerts| Alertmanager
    Alertmanager -->|Load rules| Rules

    subgraph "Alert Channels"
        PagerDuty[PagerDuty<br/>Critical alerts<br/>On-call rotation]
        Slack[Slack<br/>Warning alerts<br/>Team channel]
        Email[Email<br/>Daily summary<br/>Reports]
    end

    Alertmanager -->|Critical severity| PagerDuty
    Alertmanager -->|Warning severity| Slack
    Alertmanager -->|Daily digest| Email

    subgraph "Log Analysis - Kibana"
        Kibana[Kibana Dashboard<br/>Error analysis<br/>Slow queries<br/>User journey]
    end

    Elasticsearch -->|Query logs| Kibana
    style Prometheus fill: #ff9999
    style Dashboard1 fill: #99ff99
    style Dashboard2 fill: #99ff99
    style Dashboard3 fill: #99ff99
    style Alertmanager fill: #99ccff
```