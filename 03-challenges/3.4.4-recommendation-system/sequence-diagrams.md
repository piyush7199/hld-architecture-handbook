# Recommendation System - Sequence Diagrams

## Table of Contents

1. [Real-Time Recommendation Serving (Cache Hit)](#real-time-recommendation-serving-cache-hit)
2. [Real-Time Recommendation Serving (Cache Miss)](#real-time-recommendation-serving-cache-miss)
3. [Clickstream Event Ingestion](#clickstream-event-ingestion)
4. [Feature Store Update (Real-Time)](#feature-store-update-real-time)
5. [Model Training Pipeline (Batch)](#model-training-pipeline-batch)
6. [Model Deployment and A/B Test](#model-deployment-and-ab-test)
7. [Item-to-Item Similarity Search (FAISS)](#item-to-item-similarity-search-faiss)
8. [Multi-Strategy Candidate Generation](#multi-strategy-candidate-generation)
9. [Cold Start - New User](#cold-start---new-user)
10. [Cold Start - New Item](#cold-start---new-item)
11. [Feature Store Failover (Redis Master Down)](#feature-store-failover-redis-master-down)
12. [Model Serving Failure Fallback](#model-serving-failure-fallback)

---

## Real-Time Recommendation Serving (Cache Hit)

**Flow:**

This sequence shows the happy path where recommendations are served from cache (80% of requests).

**Steps:**

1. **Client Request** (0ms): User requests recommendations via mobile/web app
2. **API Gateway** (5ms): JWT validation, rate limit check
3. **Recommendation Service** (2ms): Check Redis cache for pre-computed recommendations
4. **Cache Hit** (2ms): Retrieve top 10 recommendations from Redis
5. **Response** (10ms total): Return recommendations to client

**Performance:**

- Total latency: ~10ms (within 50ms SLA)
- Cache hit rate: 80%
- No model inference needed (cost savings)

**Benefits:**

- Ultra-low latency for majority of requests
- Reduces load on model serving by 5x
- Scalable (Redis handles 1M+ QPS)

```mermaid
sequenceDiagram
    participant User as User<br/>Mobile/Web App
    participant Gateway as API Gateway<br/>Auth/Rate Limit
    participant RecService as Recommendation Service<br/>Orchestrator
    participant Cache as Redis Cache<br/>Pre-computed Recs<br/>TTL 1 hour
    User ->> Gateway: GET /recommendations?user_id=12345
    Note over User, Gateway: 0ms
    Gateway ->> Gateway: Validate JWT token<br/>Check rate limit (100 req/min)
    Note over Gateway: 5ms
    Gateway ->> RecService: Forward request<br/>user_id=12345, context=browsing
    Note over Gateway, RecService: 0ms
    RecService ->> Cache: GET recs:12345
    Note over RecService, Cache: 2ms - Cache check
    Cache -->> RecService: Cache HIT<br/>[item_101, item_205, ..., item_999]<br/>Top 10 recommendations
    Note over Cache, RecService: 2ms - Return cached data
    RecService ->> RecService: Format response<br/>Add metadata (titles, images, prices)
    Note over RecService: 1ms
    RecService -->> Gateway: 200 OK<br/>{recommendations: [...]}
    Note over RecService, Gateway: 0ms
    Gateway -->> User: 200 OK<br/>Display recommendations
    Note over Gateway, User: 0ms
    Note over User, Cache: Total: ~10ms<br/>Cache Hit (80% of requests)<br/>No model inference needed
```

---

## Real-Time Recommendation Serving (Cache Miss)

**Flow:**

This sequence shows the cold path where recommendations must be computed in real-time (20% of requests).

**Steps:**

1. **Client Request** (0ms): User requests recommendations
2. **API Gateway** (5ms): Validation
3. **Cache Miss** (2ms): Recommendations not in cache
4. **Parallel Feature Fetch** (5ms):
    - Fetch user features from Redis Feature Store (2ms)
    - Query FAISS for item-to-item candidates (5ms)
5. **Model Inference** (20ms): TensorFlow Serving scores all candidates
6. **Ranking** (3ms): Apply business rules, diversify, re-rank
7. **Update Cache** (async): Store result in cache for future requests
8. **Response** (35ms total): Return top 10 recommendations

**Performance:**

- Total latency: ~35ms (still within 50ms SLA)
- Parallel feature fetch minimizes sequential latency
- Cache update ensures next request is fast

```mermaid
sequenceDiagram
    participant User as User
    participant Gateway as API Gateway
    participant RecService as Recommendation Service
    participant Cache as Redis Cache
    participant FeatureStore as Redis Feature Store<br/>User Features
    participant FAISS as FAISS Index<br/>Item Similarity
    participant ModelServing as TensorFlow Serving<br/>Model Inference
    participant Ranker as Ranking Service
    User ->> Gateway: GET /recommendations?user_id=12345
    Gateway ->> Gateway: Validate JWT (5ms)
    Gateway ->> RecService: Forward request
    RecService ->> Cache: GET recs:12345
    Note over RecService, Cache: 2ms
    Cache -->> RecService: Cache MISS<br/>(No cached data)
    Note over RecService: Cache miss - compute in real-time

    par Parallel Feature Fetch
        RecService ->> FeatureStore: GET user:12345
        Note over RecService, FeatureStore: 2ms
        FeatureStore -->> RecService: {last_5_items: [1,2,3,4,5],<br/>user_embedding: [...],<br/>preferred_categories: [5,12,8]}
        RecService ->> FAISS: ANN search<br/>Query: similar to user's last 5 items
        Note over RecService, FAISS: 5ms
        FAISS -->> RecService: Top 100 candidate items<br/>[item_42, item_99, ...]
    end

    RecService ->> RecService: Prepare features<br/>Combine user + item features
    Note over RecService: 1ms
    RecService ->> ModelServing: gRPC inference request<br/>Batch: 100 candidates
    Note over RecService, ModelServing: 20ms - Model scoring
    ModelServing -->> RecService: Scores: [0.85, 0.82, 0.78, ...]
    RecService ->> Ranker: Re-rank candidates<br/>Apply business rules
    Note over RecService, Ranker: 3ms
    Ranker ->> Ranker: - Filter already purchased<br/>- Diversify categories (max 3 per cat)<br/>- Boost trending items
    Ranker -->> RecService: Top 10 items
    RecService ->> Cache: SET recs:12345 TTL 3600<br/>(Async, non-blocking)
    Note over RecService, Cache: Update cache for future requests
    RecService -->> Gateway: 200 OK<br/>{recommendations: [...]}
    Gateway -->> User: Display recommendations
    Note over User, Ranker: Total: ~35ms<br/>Cache Miss (20% of requests)<br/>Full model inference path
```

---

## Clickstream Event Ingestion

**Flow:**

This sequence shows how user events (views, clicks, purchases) are captured and ingested into the system.

**Steps:**

1. **User Action** (0ms): User views/clicks/purchases an item
2. **Client SDK** (5ms): Track event, batch locally (reduce network calls)
3. **Event Collector** (10ms): Validate event schema, enrich with metadata
4. **Kafka** (5ms): Write event to topic `clickstream.events` (partition by user_id)
5. **Kafka Consumers** (async):
    - Real-time feature computation (Kafka Streams)
    - Historical storage (Cassandra/ClickHouse)
    - Analytics dashboard (Real-time metrics)

**Performance:**

- Client latency: <20ms (non-blocking, async)
- Kafka throughput: 100k events/sec
- Event durability: Replicated 3x (no data loss)

**Benefits:**

- Decoupled ingestion (client doesn't wait for processing)
- Scalable (Kafka partitions distribute load)
- Multiple consumers (fan-out pattern)

```mermaid
sequenceDiagram
    participant User as User<br/>Mobile/Web App
    participant SDK as Client SDK<br/>Event Tracker
    participant Collector as Event Collector<br/>API
    participant Kafka as Kafka<br/>clickstream.events<br/>100k events/sec
    participant Streams as Kafka Streams<br/>Real-Time Features
    participant Storage as Cassandra/ClickHouse<br/>Historical Storage
    participant Analytics as Analytics Dashboard<br/>Real-Time Metrics
    User ->> User: Action: View item_id=42
    Note over User: User views product page
    User ->> SDK: trackEvent(VIEW, item_id=42)
    Note over User, SDK: 0ms
    SDK ->> SDK: Batch events locally<br/>(Send every 5 seconds or 10 events)
    Note over SDK: 5ms - Reduce network calls
    SDK ->> Collector: POST /events<br/>[{type: VIEW, user_id: 12345, item_id: 42, ts: 1698765432}]
    Note over SDK, Collector: 10ms - HTTP/2
    Collector ->> Collector: Validate schema<br/>Enrich with metadata:<br/>- IP address → location<br/>- User-Agent → device type<br/>- session_id
    Note over Collector: 5ms
    Collector ->> Kafka: Produce event<br/>Topic: clickstream.events<br/>Partition: hash(user_id) % 100
    Note over Collector, Kafka: 5ms - Async write
    Kafka -->> Collector: ACK (after replication)
    Collector -->> SDK: 202 Accepted
    Note over Collector, SDK: Non-blocking response
    SDK -->> User: Continue browsing
    Note over SDK, User: User doesn't wait
    Note over Kafka: Event replicated 3x<br/>Durability guaranteed

    par Kafka Consumers (Async)
        Kafka ->> Streams: Consume event<br/>Consumer Group: feature-store
        Note over Kafka, Streams: Real-time processing
        Streams ->> Streams: Update last_5_items<br/>Increment view count<br/>Update last_action_ts
        Streams ->> FeatureStore: SET user:12345
        Note over Streams: Update feature store
        Kafka ->> Storage: Consume event<br/>Consumer Group: historical-storage
        Note over Kafka, Storage: Batch write (every 10s)
        Storage ->> Storage: Insert event<br/>Partition by date
        Kafka ->> Analytics: Consume event<br/>Consumer Group: analytics
        Note over Kafka, Analytics: Real-time aggregation
        Analytics ->> Analytics: Update dashboard<br/>- Total views/hour<br/>- Popular items<br/>- Active users
    end

    Note over User, Analytics: Total client latency: ~20ms<br/>Background processing: async<br/>Throughput: 100k events/sec
```

---

## Feature Store Update (Real-Time)

**Flow:**

This sequence shows how real-time features are computed and updated in Redis from the Kafka stream.

**Steps:**

1. **Kafka Event** (0ms): Clickstream event arrives in Kafka topic
2. **Kafka Streams** (5ms): Stateful stream processing
    - Maintain windowed aggregations (last 5 viewed items, last action timestamp)
    - Group by user_id
    - Output update command
3. **Redis Update** (2ms): Atomic update of user features
    - Use ZADD for sorted sets (last 5 items)
    - Use SET for simple values (last action timestamp)
    - TTL: 7 days (auto-expire inactive users)
4. **Confirmation** (0ms): ACK back to Kafka Streams

**Performance:**

- End-to-end latency: <10ms (event → feature store)
- Throughput: 100k events/sec
- Exactly-once semantics (Kafka Streams)

**Trade-offs:**

- Eventual consistency (slight lag between event and feature update)
- Memory overhead (stateful processing)

```mermaid
sequenceDiagram
    participant Kafka as Kafka Topic<br/>clickstream.events<br/>Partition 42
    participant Streams as Kafka Streams App<br/>Stateful Processor<br/>Consumer Group: feature-store
    participant StateStore as State Store<br/>RocksDB Local<br/>Windowed Aggregations
    participant Redis as Redis Feature Store<br/>Shard 5<br/>user:12345
    Note over Kafka: Event: {<br/> user_id: 12345,<br/> type: VIEW,<br/> item_id: 789,<br/> ts: 1698765432<br/>}
    Kafka ->> Streams: Poll events<br/>(Batch of 100 events)
    Note over Kafka, Streams: 0ms - Continuous polling
    Streams ->> Streams: Process event<br/>Extract: user_id, item_id, ts
    Note over Streams: 1ms
    Streams ->> StateStore: GET user:12345<br/>Current state: last_5_items
    Note over Streams, StateStore: 2ms - Local disk read
    StateStore -->> Streams: Current: [123, 456, 789, 101, 202]
    Streams ->> Streams: Update state:<br/>- Append new item_id: 789<br/>- Keep last 5 (FIFO)<br/>- Update last_action_ts
    Note over Streams: 1ms - In-memory update
    Streams ->> StateStore: PUT user:12345<br/>Updated: [456, 789, 101, 202, 789]
    Note over Streams, StateStore: 1ms - Write to local state
    Streams ->> Redis: ZADD user:12345:last_items<br/> score=1698765432 member=789<br/>EXPIRE user:12345 604800 (7 days)<br/>SET user:12345:last_action_ts 1698765432
    Note over Streams, Redis: 2ms - Atomic pipeline
    Redis -->> Streams: OK (3 commands executed)
    Note over Redis, Streams: 0ms - ACK
    Streams ->> Kafka: Commit offset<br/>Partition 42, Offset 123456
    Note over Streams, Kafka: 2ms - Exactly-once semantics
    Note over Kafka, Redis: Total: ~10ms<br/>Exactly-once delivery<br/>State persisted locally + Redis
    Note over Redis: Feature store updated<br/>Ready for recommendation serving<br/>Latency: <10ms from event
```

---

## Model Training Pipeline (Batch)

**Flow:**

This sequence shows the end-to-end batch training pipeline that runs daily.

**Steps:**

1. **Airflow Scheduler** (0ms): Trigger DAG at 2 AM UTC
2. **Data Extraction** (4 hours):
    - Read 90 days of clickstream data from Cassandra (5 TB)
    - Filter bots, outliers, noise
    - Write clean data to S3 (Parquet)
3. **Feature Engineering** (6 hours):
    - Compute user features (Spark job, 4 hours)
    - Compute item features (Spark job, 3 hours)
    - Merge and write to S3
4. **Model Training** (10 hours):
    - Collaborative filtering (ALS, 6 hours)
    - Deep learning (Two-tower network, 4 hours)
    - Save artifacts (10 GB) to Model Store
5. **Model Deployment** (1 hour):
    - Upload to MLflow
    - A/B test on 10% traffic
    - Monitor metrics (CTR, latency)
    - Gradual rollout to 100%

**Performance:**

- Total duration: 21 hours (3-hour buffer)
- Training data: 5 TB (90 days)
- Model artifacts: 10 GB

```mermaid
sequenceDiagram
    participant Airflow as Airflow Scheduler<br/>DAG: daily_training
    participant Cassandra as Cassandra<br/>Historical Clickstream<br/>90 days, 5 TB
    participant SparkExtract as Spark Job<br/>Data Extraction
    participant S3Raw as S3 Bucket<br/>Raw Data<br/>Parquet 5 TB
    participant SparkFeatures as Spark Job<br/>Feature Engineering
    participant S3Features as S3 Bucket<br/>Feature Tables<br/>600 GB
    participant SparkML as Spark MLlib<br/>Collaborative Filtering
    participant TensorFlow as TensorFlow<br/>Deep Learning
    participant ModelStore as S3/MLflow<br/>Model Store
    participant TFServing as TensorFlow Serving<br/>Production
    Note over Airflow: 2 AM UTC - Start DAG
    Airflow ->> SparkExtract: Trigger Task 1:<br/>Data Extraction
    SparkExtract ->> Cassandra: SELECT * FROM clickstream<br/>WHERE date >= now() - 90 days
    Note over SparkExtract, Cassandra: 2 hours - Scan 5 TB
    Cassandra -->> SparkExtract: 450M events
    SparkExtract ->> SparkExtract: Filter:<br/>- Remove bots (>100 events/min)<br/>- Remove outliers<br/>- Validate schema
    Note over SparkExtract: 1 hour - Spark transformations
    SparkExtract ->> S3Raw: Write clean data<br/>Parquet format (gzip)
    Note over SparkExtract, S3Raw: 1 hour - Parallel write
    S3Raw -->> SparkExtract: Write complete (5 TB)
    SparkExtract -->> Airflow: Task 1 Complete (4 hours)
    Airflow ->> SparkFeatures: Trigger Task 2:<br/>Feature Engineering (parallel)

    par User Features (4h) and Item Features (3h)
        SparkFeatures ->> S3Raw: Read user events
        S3Raw -->> SparkFeatures: User interaction data
        SparkFeatures ->> SparkFeatures: Compute user features:<br/>- Total purchases<br/>- Avg session length<br/>- Preferred categories<br/>- User embeddings (collaborative filter)
        Note over SparkFeatures: 4 hours - Heavy aggregations
        SparkFeatures ->> S3Raw: Read item events
        S3Raw -->> SparkFeatures: Item interaction data
        SparkFeatures ->> SparkFeatures: Compute item features:<br/>- Popularity score<br/>- Engagement rate<br/>- Category distribution
        Note over SparkFeatures: 3 hours
    end

    SparkFeatures ->> SparkFeatures: Join user + item features
    Note over SparkFeatures: 2 hours - Spark join
    SparkFeatures ->> S3Features: Write feature tables
    Note over SparkFeatures, S3Features: 1 hour
    S3Features -->> SparkFeatures: Write complete (600 GB)
    SparkFeatures -->> Airflow: Task 2 Complete (6 hours)
    Airflow ->> SparkML: Trigger Task 3a:<br/>Collaborative Filtering
    Airflow ->> TensorFlow: Trigger Task 3b:<br/>Deep Learning (parallel)

    par Model Training (10h)
        SparkML ->> S3Features: Read user-item matrix
        SparkML ->> SparkML: Train ALS model<br/>Rank: 100<br/>Iterations: 10<br/>Lambda: 0.01
        Note over SparkML: 6 hours - Distributed training
        SparkML ->> ModelStore: Save model artifacts<br/>User embeddings: 5 GB<br/>Item embeddings: 5 GB
        TensorFlow ->> S3Features: Read feature tables
        TensorFlow ->> TensorFlow: Train two-tower network<br/>User tower: 128 dims<br/>Item tower: 128 dims<br/>Epochs: 10
        Note over TensorFlow: 4 hours - GPU training
        TensorFlow ->> ModelStore: Save model artifacts<br/>Weights: 2 GB
    end

    ModelStore -->> Airflow: Task 3 Complete (10 hours)
    Airflow ->> ModelStore: Trigger Task 4:<br/>Model Deployment
    ModelStore ->> ModelStore: Upload to MLflow<br/>Version: v43<br/>Metadata: training_date, accuracy
    Note over ModelStore: 10 minutes
    ModelStore ->> TFServing: Load model v43<br/>A/B test: 10% traffic
    Note over ModelStore, TFServing: 10 minutes
    TFServing -->> ModelStore: Model loaded
    TFServing ->> TFServing: Monitor metrics:<br/>- CTR: 5.8% (+11.5%)<br/>- Latency: 30ms<br/>- Error rate: 0.01%
    Note over TFServing: 30 minutes - Collect data
    TFServing -->> ModelStore: Metrics PASS<br/>CTR improvement significant
    ModelStore ->> TFServing: Gradual rollout:<br/>10% → 50% → 100%
    Note over ModelStore, TFServing: 10 minutes
    TFServing -->> ModelStore: Rollout complete
    ModelStore -->> Airflow: Task 4 Complete (1 hour)
    Airflow -->> Airflow: DAG Success<br/>Total: 21 hours
    Note over Airflow: 3-hour buffer before next run
```

---

## Model Deployment and A/B Test

**Flow:**

This sequence shows how a new model is deployed with A/B testing to validate performance before full rollout.

**Steps:**

1. **Model Store** (0ms): New model v43 is trained and ready
2. **A/B Controller** (1ms): Configure experiment (10% canary traffic)
3. **TensorFlow Serving** (5ms): Load new model alongside old model
4. **Request Routing** (0ms): Route 10% of traffic to new model (hash-based on user_id)
5. **Metric Collection** (30 min): Collect metrics (CTR, latency, errors)
6. **Statistical Analysis** (5 min): Compare control vs treatment
7. **Decision** (0ms): If p-value < 0.05 and CTR improved → Promote
8. **Gradual Rollout** (10 min): 10% → 50% → 100%

**Performance:**

- Deployment time: <1 hour (from training to 100% rollout)
- Safety: Catch regressions before impacting all users
- Data-driven: Statistical significance testing

```mermaid
sequenceDiagram
    participant ModelStore as Model Store<br/>S3/MLflow
    participant ABController as A/B Test Controller<br/>Experiment Management
    participant TFServing_Old as TensorFlow Serving<br/>Model v42 (Control)<br/>90% traffic
    participant TFServing_New as TensorFlow Serving<br/>Model v43 (Treatment)<br/>10% traffic
    participant RecService as Recommendation Service
    participant User_Control as User (Control Group)<br/>user_id: 12345<br/>Hash % 100 = 85
    participant User_Treatment as User (Treatment Group)<br/>user_id: 67890<br/>Hash % 100 = 5
    participant Kafka as Kafka<br/>Experiment Events
    participant ClickHouse as ClickHouse<br/>Analytics DB
    participant Dashboard as A/B Test Dashboard
    Note over ModelStore: New model v43 trained<br/>Ready for deployment
    ModelStore ->> ABController: Register experiment:<br/>exp_id: exp_42<br/>control: v42 (90%)<br/>treatment: v43 (10%)
    Note over ModelStore, ABController: 1 minute - Configure
    ABController ->> ABController: Hash function:<br/>user_id % 100<br/>0-9 → Treatment (10%)<br/>10-99 → Control (90%)
    ABController ->> TFServing_New: Load model v43
    Note over ABController, TFServing_New: 5 minutes - Load 10 GB model
    TFServing_New -->> ABController: Model loaded
    Note over ABController: Experiment ACTIVE<br/>Start routing traffic
    User_Control ->> RecService: GET /recommendations?user_id=12345
    RecService ->> ABController: Get experiment group<br/>user_id=12345
    ABController ->> ABController: Hash(12345) % 100 = 85<br/>Assign: Control Group (v42)
    ABController -->> RecService: group=control, model=v42
    RecService ->> TFServing_Old: gRPC inference<br/>model=v42
    Note over RecService, TFServing_Old: 20ms - Model scoring
    TFServing_Old -->> RecService: Recommendations
    RecService -->> User_Control: Return recs
    RecService ->> Kafka: Log event:<br/>{user: 12345, group: control, model: v42, recs: [...], ts: ...}
    User_Treatment ->> RecService: GET /recommendations?user_id=67890
    RecService ->> ABController: Get experiment group<br/>user_id=67890
    ABController ->> ABController: Hash(67890) % 100 = 5<br/>Assign: Treatment Group (v43)
    ABController -->> RecService: group=treatment, model=v43
    RecService ->> TFServing_New: gRPC inference<br/>model=v43
    Note over RecService, TFServing_New: 22ms - New model (slightly slower)
    TFServing_New -->> RecService: Recommendations
    RecService -->> User_Treatment: Return recs
    RecService ->> Kafka: Log event:<br/>{user: 67890, group: treatment, model: v43, recs: [...], ts: ...}
    Note over Kafka: Collect data for 30 minutes<br/>~1M requests (10% = 100k treatment)
    Kafka ->> ClickHouse: Stream experiment events<br/>Aggregate by group
    Note over Kafka, ClickHouse: Real-time ingestion
    ClickHouse ->> ClickHouse: Compute metrics:<br/>Control: CTR=5.2%, Conversion=2.1%, Latency=28ms<br/>Treatment: CTR=5.8%, Conversion=2.4%, Latency=30ms
    Dashboard ->> ClickHouse: Query experiment results<br/>exp_id=exp_42
    ClickHouse -->> Dashboard: Metrics by group
    Dashboard ->> Dashboard: Statistical test:<br/>- CTR improvement: +11.5%<br/>- p-value: 0.001 (significant)<br/>- Latency: +2ms (acceptable)<br/>Decision: PROMOTE
    Note over Dashboard: 5 minutes - Analysis
    Dashboard ->> ABController: Approve rollout<br/>exp_id=exp_42, decision=promote
    ABController ->> ABController: Gradual rollout:<br/>Step 1: 10% → 25% (wait 10 min)<br/>Step 2: 25% → 50% (wait 10 min)<br/>Step 3: 50% → 100%
    ABController ->> TFServing_New: Update traffic:<br/>25% → 50% → 100%
    Note over ABController, TFServing_New: 10 minutes total
    ABController ->> TFServing_Old: Retire model v42<br/>Unload from memory
    Note over ABController, TFServing_Old: Old model decommissioned
    ABController -->> Dashboard: Rollout complete<br/>Model v43 at 100%
    Note over ModelStore, Dashboard: Total deployment: 1 hour<br/>Safe: Caught +11.5% CTR improvement<br/>Risk: Minimal (gradual rollout)
```

---

## Item-to-Item Similarity Search (FAISS)

**Flow:**

This sequence shows how item-to-item recommendations are generated using FAISS approximate nearest neighbor search.

**Steps:**

1. **User Views Item** (0ms): User lands on product page (item_id=42)
2. **Recommendation Service** (1ms): Request similar items
3. **FAISS Index** (5ms):
    - Fetch item embedding (128-dim vector)
    - Run HNSW algorithm (approximate search)
    - Return top-50 similar items (cosine distance)
4. **Metadata Fetch** (2ms): Fetch item details from PostgreSQL/cache
5. **Response** (10ms total): Return "You May Also Like" recommendations

**Performance:**

- Query latency: <5ms for 10M items
- Accuracy: 95% recall@50 (acceptable trade-off)
- Throughput: 100k QPS (horizontally scaled)

**Trade-offs:**

- Approximate (not exact K-NN) for speed
- Index size: 5 GB RAM (must fit in memory)

```mermaid
sequenceDiagram
    participant User as User<br/>Viewing Item Page
    participant App as Mobile/Web App
    participant RecService as Recommendation Service
    participant FAISS as FAISS Index<br/>10M items<br/>128-dim embeddings<br/>HNSW Algorithm
    participant Metadata as PostgreSQL<br/>Item Metadata<br/>(Title, Price, Image)
    participant Cache as Redis Cache<br/>Item Details
    User ->> App: View product page<br/>item_id=42 (Laptop)
    Note over User, App: 0ms
    App ->> RecService: GET /similar?item_id=42
    Note over App, RecService: 0ms
    RecService ->> FAISS: Query: find_similar(item_id=42, top_k=50)
    Note over RecService, FAISS: 1ms - gRPC call
    FAISS ->> FAISS: Step 1: Fetch item embedding<br/>item_42 → [0.12, -0.34, 0.56, ..., 0.78]<br/>(128 dimensions)
    Note over FAISS: 1ms - RAM lookup
    FAISS ->> FAISS: Step 2: HNSW search<br/>Layer 2 (coarse): Navigate to region<br/>Layer 1 (medium): Narrow down<br/>Layer 0 (fine): Compute cosine distance
    Note over FAISS: 3ms - Approximate search
    FAISS -->> RecService: Top-50 similar items:<br/>[<br/> {item_id: 101, score: 0.95},<br/> {item_id: 205, score: 0.92},<br/> ...<br/> {item_id: 999, score: 0.75}<br/>]
    Note over FAISS, RecService: 5ms total FAISS query
    RecService ->> RecService: Filter results:<br/>- Remove out-of-stock items<br/>- Remove already purchased<br/>- Keep top-20
    Note over RecService: 1ms
    RecService ->> Cache: MGET item:101 item:205 ... item:999<br/>Fetch metadata for 20 items
    Note over RecService, Cache: 2ms - Batch fetch
    Cache -->> RecService: Cache HIT (15 items)<br/>Cache MISS (5 items)
    RecService ->> Metadata: SELECT * FROM items<br/>WHERE item_id IN (5 missed items)
    Note over RecService, Metadata: 3ms - Fallback to DB
    Metadata -->> RecService: Item details
    RecService ->> Cache: SET item:... (5 items)<br/>TTL 1 hour
    Note over RecService, Cache: Update cache (async)
    RecService ->> RecService: Format response:<br/>{<br/> recommendations: [<br/> {item_id, title, price, image, score},<br/> ...<br/> ]<br/>}
    Note over RecService: 1ms
    RecService -->> App: 200 OK<br/>Top-20 similar items
    Note over RecService, App: 0ms
    App -->> User: Display "You May Also Like"<br/>Carousel with 20 items
    Note over App, User: 0ms
    Note over User, Cache: Total: ~10ms<br/>FAISS: 5ms (most critical)<br/>Metadata fetch: 2-3ms<br/>95% accuracy (acceptable)
```

---

## Multi-Strategy Candidate Generation

**Flow:**

This sequence shows how the system generates candidates using 4 parallel strategies and then re-ranks them.

**Steps:**

1. **Request** (0ms): User requests recommendations
2. **Parallel Candidate Generation** (20ms max):
    - Strategy 1: Collaborative Filtering (50 candidates, 20ms)
    - Strategy 2: Content-Based (30 candidates, 5ms)
    - Strategy 3: Item-to-Item (50 candidates, 5ms)
    - Strategy 4: Trending (20 candidates, 2ms)
3. **Merge & Deduplicate** (2ms): Combine all candidates (~150 items)
4. **Model Scoring** (20ms): Score all candidates with ML model
5. **Re-Ranking** (3ms): Apply business rules, diversify categories
6. **Response** (25ms total): Return top 10

**Performance:**

- Parallel execution: 20ms (max of all strategies)
- Total latency: ~25ms (well within 50ms SLA)
- Coverage: Multiple strategies prevent filter bubble

```mermaid
sequenceDiagram
    participant User as User
    participant RecService as Recommendation Service<br/>Orchestrator
    participant CollabFilter as Collaborative Filter<br/>ALS Model
    participant ContentBased as Content-Based<br/>TF-IDF/BERT
    participant FAISS as Item-to-Item<br/>FAISS ANN
    participant Trending as Trending Cache<br/>Redis
    participant CandidatePool as Candidate Pool<br/>~150 items
    participant ModelServing as TensorFlow Serving<br/>Ranking Model
    participant Ranker as Ranking Service<br/>Business Rules
    User ->> RecService: GET /recommendations?user_id=12345
    Note over User, RecService: 0ms
    Note over RecService: Launch 4 parallel strategies

    par Strategy 1: Collaborative Filtering
        RecService ->> CollabFilter: Get user embedding<br/>user_id=12345
        CollabFilter ->> CollabFilter: Fetch user_embedding from ALS model<br/>Compute cosine similarity with all items
        Note over CollabFilter: 20ms - Heavy computation
        CollabFilter -->> RecService: 50 candidates<br/>[item_101, item_205, ...]
    end

    par Strategy 2: Content-Based
        RecService ->> ContentBased: Find items similar to<br/>user's past interactions
        ContentBased ->> ContentBased: Fetch user history:<br/>last_5_items = [1, 2, 3, 4, 5]<br/>Compute TF-IDF similarity
        Note over ContentBased: 5ms - Fast text similarity
        ContentBased -->> RecService: 30 candidates<br/>[item_55, item_88, ...]
    end

    par Strategy 3: Item-to-Item
        RecService ->> FAISS: ANN search<br/>Query: similar to last viewed item
        FAISS ->> FAISS: HNSW algorithm<br/>Find top-50 neighbors
        Note over FAISS: 5ms - ANN search
        FAISS -->> RecService: 50 candidates<br/>[item_42, item_99, ...]
    end

    par Strategy 4: Trending
        RecService ->> Trending: GET trending:category:electronics
        Trending -->> RecService: 20 trending items<br/>[item_777, item_888, ...]
        Note over Trending: 2ms - Redis cache
    end

    Note over RecService: All strategies complete<br/>Max latency: 20ms (collab filter)
    RecService ->> CandidatePool: Merge candidates:<br/>50 + 30 + 50 + 20 = 150 items
    Note over RecService, CandidatePool: 1ms
    CandidatePool ->> CandidatePool: Deduplicate:<br/>Remove duplicates across strategies<br/>Remove already purchased items<br/>Final: ~120 unique candidates
    Note over CandidatePool: 1ms
    CandidatePool ->> RecService: 120 unique candidates
    RecService ->> ModelServing: gRPC batch inference<br/>Score all 120 candidates<br/>Input: user_features + item_features
    Note over RecService, ModelServing: 20ms - Batch scoring
    ModelServing -->> RecService: Scores: [0.85, 0.82, 0.78, ...]
    RecService ->> Ranker: Re-rank with business rules
    Note over RecService, Ranker: 3ms
    Ranker ->> Ranker: Apply rules:<br/>1. Diversify categories (max 3 per category)<br/>2. Boost trending items (+10%)<br/>3. Filter low-margin items<br/>4. Ensure variety (different brands)
    Ranker -->> RecService: Top 10 items<br/>[item_101, item_205, item_42, ...]
    RecService -->> User: 200 OK<br/>Recommendations
    Note over User, Ranker: Total: ~25ms<br/>Parallel strategies: 20ms<br/>Scoring + re-ranking: 5ms<br/>Multi-strategy prevents filter bubble
```

---

## Cold Start - New User

**Flow:**

This sequence shows how the system handles a brand new user with no interaction history.

**Steps:**

1. **New User Signup** (0ms): User creates account
2. **Onboarding Survey** (30s): Explicit preference collection
    - Select 3 categories of interest
    - Rate sample items
3. **Generate Recommendations** (10ms):
    - Serve popular items in selected categories (30%)
    - Serve diverse exploration items (70%)
4. **Fast Learning** (after 10 interactions):
    - Build basic user profile
    - Switch to personalized recommendations

**Performance:**

- Initial recommendations: <10ms (pre-computed popular items)
- Learning speed: 10 interactions for basic personalization
- Exploration: 70% diverse content in first 7 days

**Benefits:**

- Avoid empty feed (poor UX)
- Fast learning (aggressive exploration)
- Explicit signals (onboarding survey)

```mermaid
sequenceDiagram
    participant User as New User<br/>No History
    participant App as Mobile/Web App
    participant RecService as Recommendation Service
    participant Onboarding as Onboarding Service
    participant PopularItems as Popular Items Cache<br/>Redis<br/>Pre-computed
    participant ExplorationEngine as Exploration Engine<br/>Diversity Sampling
    participant FeatureStore as Redis Feature Store
    User ->> App: Sign up<br/>Create account
    Note over User, App: 0ms
    App ->> Onboarding: POST /onboarding<br/>user_id=999999
    Note over App, Onboarding: 0ms
    Onboarding -->> App: Show onboarding survey:<br/>1. Select 3 categories<br/>2. Rate 5 sample items
    App -->> User: Display survey
    User ->> App: Selections:<br/>Categories: [Fashion, Technology, Sports]<br/>Ratings: [item_1: 5, item_2: 3, ...]
    Note over User, App: 30 seconds - User input
    App ->> Onboarding: POST /onboarding/submit
    Onboarding ->> FeatureStore: Initialize user profile:<br/>SET user:999999<br/>{<br/> preferred_categories: [Fashion, Technology, Sports],<br/> explicit_ratings: [item_1: 5, ...],<br/> created_at: 1698765432,<br/> interaction_count: 0<br/>}
    Note over Onboarding, FeatureStore: 2ms
    FeatureStore -->> Onboarding: Profile created
    Onboarding -->> App: Onboarding complete
    User ->> App: Request recommendations<br/>First feed load
    App ->> RecService: GET /recommendations?user_id=999999
    RecService ->> FeatureStore: GET user:999999
    FeatureStore -->> RecService: User profile:<br/>interaction_count=0 (NEW USER)
    Note over RecService: Detect new user<br/>Apply cold start strategy
    RecService ->> PopularItems: MGET popular:Fashion popular:Technology popular:Sports
    Note over RecService, PopularItems: 2ms
    PopularItems -->> RecService: 30 popular items<br/>(10 per category)
    RecService ->> ExplorationEngine: GET diverse_sample<br/>categories=[Fashion, Technology, Sports]<br/>count=70
    Note over RecService, ExplorationEngine: 5ms
    ExplorationEngine ->> ExplorationEngine: Sample diverse items:<br/>- 10 categories (not just 3 selected)<br/>- Ensure variety (different brands, price points)<br/>- Avoid popular items (force exploration)
    ExplorationEngine -->> RecService: 70 diverse items
    RecService ->> RecService: Blend recommendations:<br/>30% Popular (30 items)<br/>70% Exploration (70 items)<br/>Shuffle for variety<br/>Take top 10
    Note over RecService: 1ms
    RecService -->> App: 200 OK<br/>Top 10 recommendations
    Note over RecService, App: Total: ~10ms
    App -->> User: Display feed<br/>(Diverse, not personalized yet)
    Note over User: User interacts:<br/>View, click, purchase

    loop Fast Learning (First 10 interactions)
        User ->> App: Interaction:<br/>View item_42
        App ->> FeatureStore: Update profile:<br/>- Append to last_5_items<br/>- Increment interaction_count<br/>- Update last_action_ts
        Note over App, FeatureStore: 2ms per interaction
        FeatureStore ->> FeatureStore: Check interaction_count:<br/>If >= 10: Mark user as "learned"
    end

    Note over FeatureStore: After 10 interactions:<br/>interaction_count = 10<br/>User profile established
    User ->> App: Request recommendations<br/>(After 10 interactions)
    App ->> RecService: GET /recommendations?user_id=999999
    RecService ->> FeatureStore: GET user:999999
    FeatureStore -->> RecService: interaction_count=10<br/>(Enough data for personalization)
    Note over RecService: Switch to personalized mode<br/>Use collaborative filter + content-based
    RecService -->> App: Personalized recommendations<br/>(Not cold start anymore)
    Note over User, FeatureStore: Total cold start period: ~10 interactions<br/>Exploration: 70% (first 7 days) → 30% (after)<br/>Fast learning: Basic profile after 10 views
```

---

## Cold Start - New Item

**Flow:**

This sequence shows how the system handles a newly added item with no engagement history.

**Steps:**

1. **Item Added** (0ms): Merchant uploads new product (item_id=888888)
2. **Metadata Extraction** (5s): Extract features (title, description, category, images)
3. **Content-Based Embedding** (10s): Generate embedding using BERT (text similarity)
4. **Exploration Strategy** (ongoing):
    - Show to 5% of users (random sample)
    - Monitor engagement (CTR, purchases)
    - If CTR > threshold (3%) → Promote to wider audience
5. **Bootstrapping** (24 hours): Collect enough interactions to train embeddings

**Performance:**

- Initial visibility: 5% of users (exploration budget)
- Promotion threshold: CTR > 3% after 1000 impressions
- Full personalization: After 24 hours (next batch training)

**Benefits:**

- New items get visibility (avoid "rich get richer")
- Data-driven promotion (only boost high-quality items)
- Content-based fallback (before collaborative signals available)

```mermaid
sequenceDiagram
    participant Merchant as Merchant<br/>Seller
    participant CatalogService as Catalog Service<br/>Item Management
    participant PostgreSQL as PostgreSQL<br/>Item Metadata
    participant BERT as BERT Service<br/>Text Embeddings
    participant FAISS as FAISS Index<br/>Item Similarity
    participant ExplorationEngine as Exploration Engine<br/>5% Sample
    participant RecService as Recommendation Service
    participant User as User<br/>(5% sample)
    participant Analytics as Analytics Service<br/>Engagement Tracking
    participant BatchJob as Batch Training Job<br/>Daily
    Merchant ->> CatalogService: POST /items<br/>Create new item:<br/>title="New Laptop X1"<br/>category="Electronics"<br/>description="..."<br/>price=$1200
    Note over Merchant, CatalogService: 0ms
    CatalogService ->> PostgreSQL: INSERT INTO items<br/>(item_id=888888, title, category, ...)
    Note over CatalogService, PostgreSQL: 5ms
    PostgreSQL -->> CatalogService: Item created
    CatalogService ->> BERT: Generate embedding<br/>Input: title + description
    Note over CatalogService, BERT: 10 seconds - BERT inference
    BERT ->> BERT: Encode text:<br/>"New Laptop X1 with ..." → 128-dim vector
    BERT -->> CatalogService: Embedding: [0.23, -0.45, 0.67, ...]
    CatalogService ->> PostgreSQL: UPDATE items<br/>SET embedding = [...]<br/>WHERE item_id=888888
    Note over CatalogService, PostgreSQL: 2ms
    CatalogService ->> FAISS: Add to index<br/>item_id=888888, embedding=[...]
    Note over CatalogService, FAISS: 5ms - Incremental index update
    FAISS -->> CatalogService: Index updated
    CatalogService ->> ExplorationEngine: Register new item<br/>item_id=888888<br/>exploration_budget=5%
    Note over CatalogService, ExplorationEngine: 1ms
    ExplorationEngine ->> ExplorationEngine: Mark item as "new"<br/>Show to 5% of users<br/>Monitor engagement
    CatalogService -->> Merchant: Item published<br/>Now visible to 5% of users
    Note over ExplorationEngine: Exploration strategy active<br/>Item 888888 shown to 5% sample

    loop 5% User Traffic
        User ->> RecService: GET /recommendations
        RecService ->> RecService: Random: 5% chance
        Note over RecService: Include new item in exploration slot
        RecService ->> FAISS: Find similar items to 888888<br/>(Content-based)
        FAISS -->> RecService: Similar items (based on BERT embedding)
        RecService ->> RecService: Blend recommendations:<br/>- 8 personalized items<br/>- 1 new item (888888)<br/>- 1 trending item
        RecService -->> User: Display recommendations<br/>(Including new item)

        alt User Clicks New Item
            User ->> Analytics: Click event:<br/>item_id=888888
            Analytics ->> Analytics: Increment metrics:<br/>impressions += 1<br/>clicks += 1
        else User Ignores
            User ->> Analytics: Impression event:<br/>item_id=888888
            Analytics ->> Analytics: Increment:<br/>impressions += 1
        end
    end

    Note over Analytics: After 1000 impressions:<br/>impressions=1000<br/>clicks=40<br/>CTR=4% (above 3% threshold)
    Analytics ->> ExplorationEngine: Item 888888 performance:<br/>CTR=4% > 3% (threshold)<br/>Decision: PROMOTE
    Note over Analytics, ExplorationEngine: 1 hour - Collect data
    ExplorationEngine ->> ExplorationEngine: Update exploration budget:<br/>5% → 20% (wider audience)
    Note over ExplorationEngine: Item 888888 now visible<br/>to 20% of users (promoted)
    Note over ExplorationEngine: Wait for batch training<br/>to learn collaborative signals
    BatchJob ->> PostgreSQL: Daily training:<br/>Read item 888888 interactions
    Note over BatchJob: 24 hours later
    BatchJob ->> BatchJob: Train collaborative embeddings:<br/>Users who viewed 888888 also viewed...<br/>Generate collaborative embedding
    BatchJob ->> FAISS: Update index:<br/>Replace content-based embedding<br/>with collaborative embedding
    Note over BatchJob, FAISS: 1 hour
    FAISS -->> BatchJob: Index updated
    Note over FAISS: Item 888888 now has<br/>collaborative signals<br/>Full personalization available
    User ->> RecService: GET /recommendations
    RecService ->> RecService: Item 888888 no longer "new"<br/>Use standard recommendation logic
    RecService -->> User: Personalized recommendations<br/>(May include 888888 if relevant)
    Note over Merchant, User: Total cold start: 24-48 hours<br/>Exploration: 5% → 20% → Full<br/>Bootstrapping: Content-based → Collaborative
```

---

## Feature Store Failover (Redis Master Down)

**Flow:**

This sequence shows the failover process when a Redis master shard fails.

**Steps:**

1. **Failure Detection** (5s): Redis Sentinel detects master is down
2. **Leader Election** (3s): Sentinel promotes replica to new master
3. **DNS Update** (2s): Update service discovery (new master endpoint)
4. **Client Reconnect** (1s): Recommendation Service reconnects to new master
5. **Resume Operations** (0s): System back to normal

**Performance:**

- Total downtime: <10 seconds
- Data loss: Minimal (replication lag <100ms)
- Automatic recovery: No manual intervention

**Trade-offs:**

- Eventual consistency (replicas lag behind master by ~100ms)
- Brief unavailability during failover

```mermaid
sequenceDiagram
    participant RecService as Recommendation Service<br/>Client
    participant Sentinel as Redis Sentinel<br/>Failover Coordinator
    participant Master as Redis Master<br/>Shard 5<br/>user:2M-2.5M
    participant Replica1 as Redis Replica 5-1<br/>Read replica
    participant Replica2 as Redis Replica 5-2<br/>Read replica
    participant ServiceDiscovery as Service Discovery<br/>DNS/Consul
    Note over RecService, Master: Normal operations
    RecService ->> Master: SET user:2345678<br/>{last_5_items: [...], ...}
    Master -->> RecService: OK
    Master-. -> Replica1: Async replication<br/>(lag: 50ms)
    Master-. -> Replica2: Async replication<br/>(lag: 50ms)
    Note over Master: FAILURE<br/>Master node crashes<br/>Network partition
    RecService ->> Master: SET user:2345679<br/>{...}
    Note over RecService, Master: Timeout - No response
    Sentinel ->> Master: Health check ping<br/>(every 1 second)
    Note over Sentinel, Master: 5 seconds - No response
    Sentinel ->> Sentinel: Mark master as DOWN<br/>Start failover process
    Note over Sentinel: 1 second - Quorum consensus
    Sentinel ->> Replica1: SLAVEOF NO ONE<br/>Promote to master
    Note over Sentinel, Replica1: 2 seconds - Promotion
    Replica1 ->> Replica1: Transition to master:<br/>- Stop replication<br/>- Accept writes<br/>- Reconfigure
    Note over Replica1: 1 second
    Replica1 -->> Sentinel: Promotion complete<br/>I am now master
    Sentinel ->> Replica2: SLAVEOF new_master_ip<br/>Replicate from Replica1 (now master)
    Note over Sentinel, Replica2: 1 second - Reconfigure
    Replica2 ->> Replica1: Start replication
    Note over Replica2, Replica1: Sync data
    Sentinel ->> ServiceDiscovery: Update endpoint:<br/>redis-shard-5-master:<br/>old_ip → new_ip (Replica1)
    Note over Sentinel, ServiceDiscovery: 2 seconds - DNS TTL
    ServiceDiscovery -->> Sentinel: DNS updated
    RecService ->> RecService: Connection failed<br/>Retry with new endpoint
    Note over RecService: 1 second - Exponential backoff
    RecService ->> ServiceDiscovery: Resolve redis-shard-5-master
    ServiceDiscovery -->> RecService: new_ip (Replica1)
    RecService ->> Replica1: SET user:2345679<br/>{...} (retry)
    Note over RecService, Replica1: Connected to new master
    Replica1 -->> RecService: OK
    Note over RecService, Replica2: System recovered<br/>Total downtime: ~10 seconds
    RecService ->> Replica1: SET user:2345680<br/>{...}
    Replica1 -->> RecService: OK
    Replica1-. -> Replica2: Async replication
    Note over Replica1, Replica2: Normal operations resumed
    Note over Master: Old master still down<br/>Will be repaired manually<br/>and rejoin as replica
    Note over RecService, Replica2: Data loss analysis:<br/>- Replication lag: 50-100ms<br/>- Lost writes: ~10 events<br/>- Impact: Minimal (features rebuilt from Kafka)
```

---

## Model Serving Failure Fallback

**Flow:**

This sequence shows the fallback strategy when TensorFlow Serving (model inference) fails.

**Steps:**

1. **Normal Request** (0ms): User requests recommendations
2. **Model Serving Failure** (5s timeout): TensorFlow Serving is down or overloaded
3. **Fallback Strategy** (5ms):
    - Serve pre-computed recommendations from cache
    - If cache miss: Serve popular items (degraded mode)
4. **Alerting** (1s): Notify on-call engineer via PagerDuty
5. **Mitigation** (10 min): Restart TensorFlow Serving or route to backup instance

**Performance:**

- Fallback latency: <5ms (cache-based)
- Availability: 99.99% (graceful degradation)
- Impact: Reduced personalization quality during incident

**Trade-offs:**

- Recommendations are stale (pre-computed)
- No real-time personalization during incident
- But system remains available (better than 500 errors)

```mermaid
sequenceDiagram
    participant User as User
    participant RecService as Recommendation Service<br/>Orchestrator
    participant FeatureStore as Redis Feature Store
    participant ModelServing as TensorFlow Serving<br/>Model Inference<br/>PRIMARY
    participant PrecomputedCache as Redis Cache<br/>Pre-computed Recs<br/>TTL 1 hour
    participant PopularItems as Popular Items<br/>Fallback Cache
    participant Alerting as PagerDuty<br/>Alert On-Call
    participant BackupModel as TensorFlow Serving<br/>BACKUP Instance
    User ->> RecService: GET /recommendations?user_id=12345
    Note over User, RecService: 0ms
    RecService ->> FeatureStore: GET user:12345
    Note over RecService, FeatureStore: 2ms
    FeatureStore -->> RecService: User features
    RecService ->> ModelServing: gRPC inference request<br/>Timeout: 5 seconds
    Note over RecService, ModelServing: Attempt primary model
    Note over ModelServing: FAILURE<br/>Model serving down<br/>OOM error / Pod crash
    ModelServing --x RecService: Timeout after 5 seconds<br/>gRPC error: UNAVAILABLE
    Note over ModelServing, RecService: Connection failed
    RecService ->> RecService: Detect failure<br/>Increment error_count metric<br/>Decision: Use fallback strategy
    Note over RecService: 0ms - Circuit breaker
    Note over RecService: Fallback Strategy 1:<br/>Pre-computed recommendations
    RecService ->> PrecomputedCache: GET recs:12345
    Note over RecService, PrecomputedCache: 2ms

    alt Cache Hit (80%)
        PrecomputedCache -->> RecService: Pre-computed recommendations<br/>[item_101, item_205, ...]
        Note over PrecomputedCache, RecService: Success - Serve stale recs
        RecService -->> User: 200 OK<br/>Recommendations (pre-computed)
        Note over RecService, User: Latency: ~5ms<br/>Degraded: Stale recs (up to 1h old)
    else Cache Miss (20%)
        PrecomputedCache -->> RecService: Cache MISS
        Note over RecService: Fallback Strategy 2:<br/>Popular items (degraded mode)
        RecService ->> FeatureStore: GET user:12345:preferred_categories
        FeatureStore -->> RecService: [Electronics, Fashion, Sports]
        RecService ->> PopularItems: MGET popular:Electronics popular:Fashion popular:Sports
        Note over RecService, PopularItems: 2ms
        PopularItems -->> RecService: 30 popular items<br/>(10 per category)
        RecService ->> RecService: Shuffle and take top 10<br/>No personalization
        Note over RecService: 1ms
        RecService -->> User: 200 OK<br/>Popular recommendations<br/>(Not personalized)
        Note over RecService, User: Latency: ~8ms<br/>Degraded: No personalization
    end

    Note over RecService: System degraded but available<br/>Better than 500 errors
    RecService ->> Alerting: Send alert:<br/>PagerDuty: HIGH<br/>Model serving down<br/>Fallback active
    Note over RecService, Alerting: 1 second - Alert
    Alerting -->> Alerting: Page on-call engineer<br/>Subject: TensorFlow Serving DOWN
    Note over ModelServing: On-call engineer investigates<br/>Root cause: OOM (out of memory)
    Note over BackupModel: Backup instance available<br/>In different AZ
    RecService ->> RecService: Circuit breaker:<br/>Stop calling primary<br/>Route to backup
    Note over RecService: 10 minutes - Manual failover
    RecService ->> BackupModel: gRPC inference request
    Note over RecService, BackupModel: Route to backup
    BackupModel -->> RecService: Scores: [0.85, 0.82, ...]
    RecService -->> User: 200 OK<br/>Personalized recommendations<br/>(from backup)
    Note over RecService, User: System recovered<br/>Full personalization restored
    Note over ModelServing: Primary restarted<br/>Memory limit increased<br/>Rejoins cluster
    RecService ->> RecService: Circuit breaker reset<br/>Resume calling primary
    Note over RecService: Health checks pass
    RecService ->> ModelServing: gRPC inference request
    ModelServing -->> RecService: Success
    Note over User, ModelServing: Incident resolved<br/>Total downtime: 10 minutes<br/>Availability maintained: 99.99%<br/>Impact: Degraded personalization (not outage)
```