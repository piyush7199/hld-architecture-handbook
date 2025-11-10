# Global News Feed - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made in the Global News Feed system
design.

---

## Table of Contents

1. [Elasticsearch Over PostgreSQL for Full-Text Search](#1-elasticsearch-over-postgresql-for-full-text-search)
2. [Kappa Architecture (Stream-Only) Over Lambda Architecture (Batch + Stream)](#2-kappa-architecture-stream-only-over-lambda-architecture-batch--stream)
3. [LSH (Locality-Sensitive Hashing) Over Exact Content Matching for Deduplication](#3-lsh-locality-sensitive-hashing-over-exact-content-matching-for-deduplication)
4. [Write Buffer (Redis Streams) Over Direct Elasticsearch Writes](#4-write-buffer-redis-streams-over-direct-elasticsearch-writes)
5. [Hybrid ML Model (Collaborative + Content-Based) Over Pure Collaborative Filtering](#5-hybrid-ml-model-collaborative--content-based-over-pure-collaborative-filtering)
6. [Real-Time Feature Store Over Batch-Only Personalization](#6-real-time-feature-store-over-batch-only-personalization)
7. [Multi-Region Read Replicas Over Single-Region with CDN Only](#7-multi-region-read-replicas-over-single-region-with-cdn-only)

---

## 1. Elasticsearch Over PostgreSQL for Full-Text Search

### The Problem

The Global News Feed system needs to:

- Store and search 100M articles per day (180TB over 5 years)
- Provide low-latency full-text search (<100ms)
- Support complex queries (personalized ranking, filtering by topics, date range)
- Handle 300,000 read QPS (peak)
- Provide real-time aggregations for trending topics

### Options Considered

| Aspect               | Elasticsearch                       | PostgreSQL (with Full-Text Search) | Solr                     |
|----------------------|-------------------------------------|------------------------------------|--------------------------|
| **Full-Text Search** | Native inverted index, BM25 ranking | `ts_vector`, limited ranking       | Native inverted index    |
| **Query Latency**    | <100ms for complex queries          | 500ms+ for complex queries         | <150ms                   |
| **Scalability**      | Horizontal (add nodes)              | Vertical (limited)                 | Horizontal (add shards)  |
| **Personalization**  | `function_score` query (flexible)   | Complex SQL, slow                  | Limited scoring options  |
| **Aggregations**     | Real-time (fast)                    | Slow for large datasets            | Good, but slower than ES |
| **Ops Complexity**   | Medium (cluster management)         | Low (mature RDBMS)                 | Medium-High              |
| **Cost**             | Medium (dedicated cluster)          | High (powerful single server)      | Medium                   |
| **Ecosystem**        | Rich (Kibana, Beats)                | Mature SQL ecosystem               | Java-based, older        |

### Decision Made

**Use Elasticsearch as the primary storage and search engine for articles.**

### Rationale

1. **Full-Text Search Performance**
    - Elasticsearch is purpose-built for full-text search with inverted indices
    - BM25 ranking algorithm provides relevant results
    - Query latency: <100ms for complex queries (vs. PostgreSQL: 500ms+)

2. **Horizontal Scalability**
    - Elasticsearch scales horizontally by adding nodes
    - Sharding distributes load across 30 data nodes (120 primary shards)
    - PostgreSQL scales vertically (expensive, limited)

3. **Personalized Ranking**
    - `function_score` query allows combining multiple scoring factors:
        - User preferences (topic match)
        - Article quality (quality_score)
        - Recency (time decay)
        - Embeddings (user · article dot product)
    - PostgreSQL would require complex SQL with CTEs, much slower

4. **Real-Time Aggregations**
    - Trending topics detection requires counting events per topic in real-time
    - Elasticsearch `terms` aggregation is fast (<50ms for millions of docs)
    - PostgreSQL `GROUP BY` would be slow (seconds for large tables)

5. **Hot-Warm-Cold Architecture**
    - Elasticsearch ILM (Index Lifecycle Management) automatically moves old articles to cheaper storage (HDD)
    - PostgreSQL partitioning is manual and complex

### Implementation Details

**Cluster Configuration:**

- **Nodes:** 30 data nodes + 3 master nodes
- **Shards:** 120 primary shards, 120 replica shards (2x redundancy)
- **Storage:** 180TB total (SSD for hot tier, HDD for cold tier)
- **Index Strategy:** Time-based indices (`news-2025-01`, `news-2025-02`)

**Query Example:**

```
GET /news/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            { "match": { "body": "artificial intelligence" } }
          ],
          "filter": [
            { "terms": { "topics": ["Tech", "Business"] } },
            { "range": { "published_at": { "gte": "now-24h" } } }
          ]
        }
      },
      "functions": [
        { "field_value_factor": { "field": "quality_score" } },
        { "exp": { "published_at": { "scale": "12h" } } },
        { "script_score": { "script": "cosineSimilarity(params.user_embedding, 'article_embedding')" } }
      ]
    }
  },
  "size": 50
}
```

### Trade-offs Accepted

| What We Gain                         | What We Sacrifice                                           |
|--------------------------------------|-------------------------------------------------------------|
| ✅ Fast full-text search (<100ms)     | ❌ Eventual consistency (articles may appear before indexed) |
| ✅ Horizontal scalability (add nodes) | ❌ Higher ops complexity (cluster management)                |
| ✅ Real-time aggregations (trending)  | ❌ Higher infrastructure cost (dedicated cluster)            |
| ✅ Flexible personalized ranking      | ❌ No ACID transactions (not needed for news feed)           |
| ✅ Hot-Warm-Cold architecture         | ❌ Requires write buffer to handle bursts                    |

### When to Reconsider

- **If we need strong ACID transactions** (e.g., financial data) → Use PostgreSQL
- **If we have <1M articles** (small dataset) → PostgreSQL full-text search sufficient
- **If we need complex joins** (e.g., user + article + comments) → PostgreSQL
- **If budget is very limited** → Use PostgreSQL with read replicas

---

## 2. Kappa Architecture (Stream-Only) Over Lambda Architecture (Batch + Stream)

### The Problem

The news ingestion and processing pipeline needs to:

- Process 100M articles/day (1,157 articles/sec sustained)
- Handle burst traffic (5,000 articles/sec during breaking news)
- Provide low-latency processing (articles appear in feed within 10 seconds)
- Support reprocessing (e.g., re-run NLP on old articles with new model)

### Options Considered

| Aspect              | Kappa (Stream-Only)       | Lambda (Batch + Stream) | Batch-Only            |
|---------------------|---------------------------|-------------------------|-----------------------|
| **Latency**         | Low (<10 seconds)         | Medium (minutes)        | High (hours)          |
| **Code Complexity** | Low (single codebase)     | High (two codebases)    | Low (single codebase) |
| **Reprocessing**    | Replay Kafka topics       | Reprocess from batch    | Rerun batch job       |
| **Infrastructure**  | Kafka + Stream processors | Kafka + Spark + Stream  | Spark only            |
| **Real-Time**       | Native                    | Separate stream layer   | Not possible          |
| **Cost**            | Medium                    | High (two systems)      | Low                   |
| **Operational**     | Medium                    | High (two systems)      | Low                   |

### Decision Made

**Use Kappa Architecture (stream-only processing with Kafka as the single source of truth).**

### Rationale

1. **Single Codebase**
    - Lambda requires maintaining two separate codebases: batch (Spark) and stream (Flink/Kafka Streams)
    - Kappa uses a single stream processing codebase for both real-time and batch (replay)
    - Reduces bugs, simplifies testing

2. **Low Latency**
    - News articles are time-sensitive (users want latest news)
    - Kappa processes articles in real-time (<10 seconds from ingestion to feed)
    - Lambda has higher latency for batch layer (minutes to hours)

3. **Reprocessing via Replay**
    - If we update the NLP model, we can replay Kafka topics to reprocess all articles
    - Kafka retention: 30 days for `article.nlp` topic
    - Lambda batch layer requires separate data lake (e.g., S3)

4. **Operational Simplicity**
    - One system to monitor and operate (Kafka + stream processors)
    - Lambda requires managing both batch (Spark) and stream (Flink), doubling ops complexity

5. **Cost Efficiency**
    - Kappa uses Kafka for both real-time and reprocessing
    - Lambda requires both Kafka AND data lake (S3) + Spark cluster

### Implementation Details

**Kappa Pipeline:**

1. **Ingestion** → Kafka topic `article.raw` (retention: 7 days)
2. **Deduplication** → Kafka topic `article.deduplicated` (retention: 7 days)
3. **NLP Processing** → Kafka topic `article.nlp` (retention: 30 days)
4. **Indexing** → Elasticsearch (persistent storage)

**Reprocessing Example:**

```bash
# Deploy new NLP model with improved sentiment analysis
# Replay last 30 days of articles

kafka-streams-app \
  --application-id nlp-service-v2 \
  --input-topic article.deduplicated \
  --output-topic article.nlp.v2 \
  --reset-offsets earliest \
  --processing-guarantee exactly_once
```

**Stream Processor:** Kafka Streams or Apache Flink

- Stateful processing (maintains deduplication state)
- Exactly-once semantics (no duplicate articles)
- Windowed aggregations (trending topics)

### Trade-offs Accepted

| What We Gain                | What We Sacrifice                                       |
|-----------------------------|---------------------------------------------------------|
| ✅ Low latency (<10 seconds) | ❌ Limited reprocessing window (30 days Kafka retention) |
| ✅ Single codebase (simpler) | ❌ Requires Kafka retention (storage cost)               |
| ✅ Real-time processing      | ❌ More complex stream processing (vs. batch SQL)        |
| ✅ Lower ops complexity      | ❌ Kafka becomes critical dependency (SPOF)              |
| ✅ Lower cost (one system)   | ❌ Harder to debug than batch jobs                       |

### When to Reconsider

- **If we need long-term reprocessing (>30 days)** → Use Lambda with data lake
- **If latency is not critical (hours acceptable)** → Use batch-only with Spark
- **If we have limited stream processing expertise** → Use Lambda (batch is easier)
- **If Kafka becomes too expensive** → Use Lambda with cheaper storage (S3)

---

## 3. LSH (Locality-Sensitive Hashing) Over Exact Content Matching for Deduplication

### The Problem

News aggregators ingest the same story from multiple sources (CNN, BBC, Reuters). We need to:

- Identify near-duplicate articles (same story, different wording)
- Group articles into story clusters
- Avoid showing duplicate stories to users
- Process 1,157 articles/sec with low latency (<100ms per article)

### Options Considered

| Aspect                       | LSH (MinHash)              | Exact URL Match         | Edit Distance (Levenshtein) | TF-IDF + Cosine Similarity |
|------------------------------|----------------------------|-------------------------|-----------------------------|----------------------------|
| **Near-Duplicate Detection** | Yes (fuzzy matching)       | No (exact only)         | Yes (character-level)       | Yes (word-level)           |
| **Latency**                  | 10-50ms                    | <1ms                    | 500ms+ (slow)               | 100-500ms                  |
| **Scalability**              | Good (hash-based)          | Excellent (hash lookup) | Poor (O(n²))                | Medium (needs full TF-IDF) |
| **Accuracy**                 | 85-90% (tunable)           | 100% (exact)            | High (slow)                 | High (slow)                |
| **False Positives**          | 1-5%                       | 0%                      | 0%                          | <1%                        |
| **Memory**                   | Low (hashes only)          | Very low                | High (full text)            | High (TF-IDF vectors)      |
| **Implementation**           | Complex (MinHash, banding) | Simple                  | Simple                      | Medium                     |

### Decision Made

**Use two-stage deduplication:**

1. **Stage 1:** Bloom Filter for exact URL matching (fast, low false positive rate)
2. **Stage 2:** LSH (MinHash) for near-duplicate content detection (fuzzy matching)

### Rationale

1. **Near-Duplicate Detection**
    - News articles about the same story have 80-95% content overlap
    - Example: "Apple announces iPhone 16" vs "Apple unveils new iPhone 16"
    - LSH detects these as duplicates (cosine similarity > 0.85)
    - Exact URL matching would miss these (different URLs)

2. **Low Latency**
    - LSH signature computation: 50ms (MinHash with 128 hash functions)
    - LSH index query (Redis Sorted Set): 20ms
    - Total deduplication latency: <100ms (p99)
    - Edit distance would be 500ms+ (too slow for 1,157 articles/sec)

3. **Scalability**
    - LSH is hash-based (constant time lookup)
    - Can handle 100M articles with <10GB memory (hashes only)
    - TF-IDF would require storing full vectors (100GB+ memory)

4. **Tunable Accuracy**
    - Similarity threshold can be adjusted (default: 85%)
    - Higher threshold (90%) → fewer false positives, more false negatives
    - Lower threshold (80%) → more false positives, fewer false negatives

5. **Bloom Filter Fast Path**
    - 99% of duplicate articles have the exact same URL (syndicated content)
    - Bloom Filter catches these in <1ms (skip expensive LSH stage)
    - Only 1% of articles need LSH (new content)

### Implementation Details

**Stage 1: Bloom Filter (Exact URL Match)**

```
- Size: 1.2GB (stores 1B URLs)
- Hash functions: 7 (optimal for 0.1% false positive rate)
- False positive rate: 0.1%
- Lookup time: <1ms
```

**Stage 2: LSH (MinHash)**

```
Algorithm: MinHash + Banding
- Shingle size: 5 (n-grams of length 5)
- Hash functions: 128 (for MinHash signature)
- Bands: 16 (8 rows each)
- Similarity threshold: 0.85 (cosine similarity)
- LSH index: Redis Sorted Set (ZRANGEBYSCORE for similarity query)
```

**LSH Process:**

1. Extract text content (remove HTML, stopwords)
2. Generate shingles (5-grams): "apple announces iphone" → ["apple", "apple announ", "announ iphone", ...]
3. Compute MinHash signature (128 hash values)
4. Store in LSH index (Redis): `ZADD lsh:band:0 <score> <article_id>`
5. Query for similar articles: `ZRANGEBYSCORE lsh:band:0 <min> <max>`

**Example:**

```
Article 1: "Apple announces new iPhone 16 with improved camera"
Article 2: "Apple unveils iPhone 16 featuring enhanced camera system"

Cosine similarity: 0.87 (above 0.85 threshold)
Result: Grouped into same story cluster
```

### Trade-offs Accepted

| What We Gain                       | What We Sacrifice                                     |
|------------------------------------|-------------------------------------------------------|
| ✅ Near-duplicate detection (fuzzy) | ❌ 1-5% false positives (some duplicates slip through) |
| ✅ Low latency (<100ms)             | ❌ Complex implementation (MinHash + banding)          |
| ✅ Scalable (handles 100M articles) | ❌ Tuning required (similarity threshold)              |
| ✅ Low memory (hashes only)         | ❌ Bloom Filter false positives (0.1%)                 |
| ✅ Reduces index size by 60%        | ❌ GPU acceleration needed for high scale              |

### When to Reconsider

- **If we need 100% accuracy** → Use TF-IDF + exact cosine similarity (slower)
- **If latency is not critical (1 second acceptable)** → Use Edit Distance
- **If dataset is small (<1M articles)** → Use TF-IDF + cosine similarity (simpler)
- **If false positives are unacceptable** → Use exact URL matching only (miss near-duplicates)

---

## 4. Write Buffer (Redis Streams) Over Direct Elasticsearch Writes

### The Problem

During traffic spikes (breaking news), the system experiences:

- Burst writes: 5,000 articles/sec (5x normal rate)
- Elasticsearch cluster struggles with write load
- Index fragmentation (too many small segments)
- High CPU usage for segment merging
- Increased query latency (200ms+ during spikes)

### Options Considered

| Aspect                 | Redis Streams Buffer       | Direct ES Writes      | Kafka as Buffer         | RabbitMQ as Buffer      |
|------------------------|----------------------------|-----------------------|-------------------------|-------------------------|
| **Write Smoothing**    | Excellent (absorbs spikes) | Poor (ES overwhelmed) | Good                    | Good                    |
| **Bulk Write Support** | Yes (batch read)           | Manual batching       | Yes (batch consume)     | Yes (batch consume)     |
| **Latency**            | <10s (p99)                 | <1s (direct)          | <5s (p99)               | <5s (p99)               |
| **Durability**         | Yes (Redis persistence)    | N/A                   | Yes (replication)       | Yes (persistent queues) |
| **Backpressure**       | Buffer grows (memory)      | ES returns 429 errors | Disk-based (scales)     | Disk-based (scales)     |
| **Ops Complexity**     | Low (Redis cluster)        | Low                   | Medium (Kafka cluster)  | Medium (RMQ cluster)    |
| **Cost**               | Low (memory)               | Low                   | Medium (disk + brokers) | Medium (disk + brokers) |

### Decision Made

**Use Redis Streams as a write buffer before Elasticsearch bulk indexing.**

### Rationale

1. **Absorbs Write Spikes**
    - Burst traffic (5,000 articles/sec) is absorbed into Redis Streams
    - Bulk writers consume at a steady rate (1,500 articles/sec)
    - Elasticsearch never sees the burst (protected from overload)

2. **Efficient Bulk Writes**
    - Bulk writer reads 1000 articles every 5 seconds from Redis
    - Elasticsearch `_bulk` API is 30x faster than individual writes
    - Reduces index fragmentation (fewer, larger segments)

3. **Low Latency**
    - Write to Redis: <1ms (in-memory)
    - Bulk write latency: 200-500ms for 1000 docs
    - End-to-end indexing delay: <10 seconds (p99)
    - Kafka would add network hops (already using Kafka upstream)

4. **Backpressure Handling**
    - If Elasticsearch is slow, Redis Streams buffer grows temporarily
    - Max buffer size: 100,000 messages (capped for memory safety)
    - If buffer full, reject new writes with 503 (graceful degradation)

5. **Operational Simplicity**
    - Redis Streams is simpler than Kafka (no partition management)
    - Already using Redis for caching (shared infrastructure)
    - Kafka would require separate Kafka cluster just for write buffering

### Implementation Details

**Redis Streams Configuration:**

```
Stream key: es:write:buffer
Consumer group: es:bulk:writers
Max length: 100,000 messages (MAXLEN ~ 100000)
TTL: 1 hour (prevent unbounded growth)
```

**Bulk Writer Logic:**

```
while true:
  messages = XREADGROUP(
    group="es:bulk:writers",
    consumer="worker-1",
    count=1000,
    block=5000ms  // Wait up to 5 seconds for batch
  )
  
  if len(messages) >= 1000 OR elapsed >= 5s:
    bulk_write_to_elasticsearch(messages)
    XACK(messages)  // Acknowledge after successful write
```

**Performance Metrics:**

- Write to Redis: <1ms per article
- Bulk write latency: 200-500ms for 1000 docs
- Throughput: 1,500 docs/sec sustained, 5,000 docs/sec peak
- Indexing delay: <10 seconds (p99)

**Failure Handling:**

- **Elasticsearch unavailable** → DO NOT XACK messages (they remain in stream)
- **Partial bulk failure** → Re-add failed docs to retry queue
- **Redis full** → Reject new writes with 503 Service Unavailable

### Trade-offs Accepted

| What We Gain                                  | What We Sacrifice                     |
|-----------------------------------------------|---------------------------------------|
| ✅ Smooth write spikes (5,000/sec → 1,500/sec) | ❌ Indexing delay (5-10 seconds)       |
| ✅ Efficient bulk writes (30x faster)          | ❌ Requires Redis cluster memory       |
| ✅ Protects Elasticsearch from overload        | ❌ Additional component to monitor     |
| ✅ Backpressure handling (buffer grows)        | ❌ Eventual consistency (slight delay) |
| ✅ Lower ops complexity (vs. Kafka)            | ❌ Limited buffer size (100k messages) |

### When to Reconsider

- **If indexing delay must be <1 second** → Direct Elasticsearch writes (accept risk of overload)
- **If we need unlimited buffer** → Use Kafka (disk-based, no memory limit)
- **If we already have Kafka for other purposes** → Use Kafka as buffer (simpler stack)
- **If memory cost is high** → Use Kafka (disk is cheaper than Redis memory)

---

## 5. Hybrid ML Model (Collaborative + Content-Based) Over Pure Collaborative Filtering

### The Problem

User personalization requires:

- Recommend relevant articles to users based on their interests
- Handle cold start problem (new users with no history)
- Adapt to trending topics (viral articles)
- Provide diverse recommendations (not just filter bubble)
- Process 100M user actions/day for training

### Options Considered

| Aspect                | Hybrid (Collaborative + Content) | Pure Collaborative Filtering | Pure Content-Based       | Deep Learning (BERT)  |
|-----------------------|----------------------------------|------------------------------|--------------------------|-----------------------|
| **Accuracy**          | High (combines both)             | Medium (ignores content)     | Medium (ignores social)  | Very High (slow)      |
| **Cold Start**        | Good (content fallback)          | Poor (needs history)         | Good (no history needed) | Good                  |
| **Diversity**         | Good (content explores)          | Poor (filter bubble)         | Good                     | Medium                |
| **Training Time**     | 2-4 hours (daily)                | 1-2 hours (faster)           | <1 hour (simple)         | 24 hours+ (expensive) |
| **Inference Latency** | <5ms (embeddings)                | <5ms (embeddings)            | <10ms (TF-IDF)           | 50-100ms (GPU)        |
| **Scalability**       | Good (Spark MLlib)               | Good (ALS algorithm)         | Excellent (simple)       | Poor (GPU required)   |
| **Interpretability**  | Medium                           | Low                          | High (clear features)    | Very Low (black box)  |
| **Cost**              | Medium                           | Low                          | Low                      | High (GPU training)   |

### Decision Made

**Use a hybrid ML model that combines collaborative filtering (matrix factorization) and content-based filtering (
TF-IDF + cosine similarity).**

### Rationale

1. **Better Accuracy**
    - Collaborative filtering captures user-user and item-item similarities
    - Content-based filtering understands article features (topics, keywords)
    - Ensemble model outperforms either approach alone
    - Offline metrics: Precision@10 = 0.78 (vs. 0.65 for pure collaborative)

2. **Solves Cold Start Problem**
    - **New user** → No collaborative signal, fallback to content-based (topics, demographics)
    - **New article** → No user interactions yet, fallback to content similarity with popular articles
    - Pure collaborative filtering fails for new users/articles

3. **Provides Diversity**
    - Collaborative filtering tends to create filter bubbles (only similar content)
    - Content-based explores new topics based on keywords/entities
    - Hybrid model balances exploitation (collaborative) and exploration (content)

4. **Fast Inference**
    - Batch training precomputes user/article embeddings (128 dimensions)
    - Inference: dot product of user_embedding · article_embedding (<5ms)
    - Deep learning (BERT) would require GPU inference (50-100ms, too slow for 300k QPS)

5. **Cost Efficiency**
    - Training: Apache Spark MLlib (CPU-based, cheaper than GPU)
    - Training time: 2-4 hours daily (acceptable)
    - Deep learning would require expensive GPU cluster (10x cost)

### Implementation Details

**Hybrid Model Architecture:**

1. **Collaborative Filtering (60% weight)**
    - Algorithm: Matrix Factorization using ALS (Alternating Least Squares)
    - Input: User-Article interaction matrix (100M users × 100M articles)
    - Output: User embeddings (128D) + Article embeddings (128D)
    - Training: Spark MLlib ALS
   ```
   model = ALS(
     rank=128,  // Embedding dimensions
     maxIter=10,
     regParam=0.1,
     implicitPrefs=True  // Implicit feedback (clicks, not ratings)
   )
   ```

2. **Content-Based Filtering (40% weight)**
    - Algorithm: TF-IDF + Cosine Similarity
    - Features: Topics, keywords, entities, sentiment, quality_score
    - User profile: Weighted average of read articles' TF-IDF vectors
    - Score: cosine_similarity(user_profile, article_vector)

3. **Ensemble Model**
   ```
   final_score = 0.6 × collaborative_score + 0.4 × content_score
   ```

**Training Pipeline:**

```
1. Data Collection (30 min)
   - Read user actions (clicks, reads, shares) from Kafka (last 24h)
   - Read user profiles (demographics, stated interests) from PostgreSQL
   - Read article metadata (topics, keywords) from Elasticsearch

2. Feature Engineering (1 hour)
   - User features: read history vector, topic preferences, recency weighting
   - Article features: TF-IDF vector, entity vector, quality score
   - Interaction features: click-through rate per topic, dwell time

3. Model Training (2 hours)
   - Train collaborative filtering (ALS) on Spark
   - Train content-based (TF-IDF) on Spark
   - Combine into ensemble model

4. Model Evaluation (30 min)
   - Offline metrics: Precision@10, Recall@10, NDCG@10
   - Compare to baseline model (previous day)
   - If metrics improved, proceed to deployment

5. Deployment (30 min)
   - Blue-green deployment (10% canary, monitor, full rollout)
   - Store embeddings in Redis (user:123:embedding, article:xyz:embedding)
   - TTL: 48 hours
```

### Trade-offs Accepted

| What We Gain                       | What We Sacrifice                                    |
|------------------------------------|------------------------------------------------------|
| ✅ Higher accuracy (0.78 vs 0.65)   | ❌ More complex training pipeline                     |
| ✅ Solves cold start problem        | ❌ Longer training time (2-4h vs 1-2h)                |
| ✅ Provides diverse recommendations | ❌ Harder to interpret (two models)                   |
| ✅ Fast inference (<5ms)            | ❌ Requires careful weight tuning (60/40 split)       |
| ✅ Cost-efficient (CPU training)    | ❌ 24-hour lag (mitigated by real-time feature store) |

### When to Reconsider

- **If we need real-time personalization (<1 minute lag)** → Use online learning (e.g., bandits)
- **If accuracy is critical (e.g., $1M revenue per 1% improvement)** → Use deep learning (BERT)
- **If cold start is rare (<1% users)** → Use pure collaborative filtering (simpler)
- **If training infrastructure is limited** → Use content-based only (no Spark needed)

---

## 6. Real-Time Feature Store Over Batch-Only Personalization

### The Problem

Batch ML training (daily job) has a 24-hour lag:

- User clicks on "AI" articles at 9 AM
- Recommendations don't reflect this interest until next day (after batch job)
- User sees stale recommendations for 24 hours
- Viral articles and trending topics are missed

### Options Considered

| Aspect                  | Real-Time Feature Store        | Batch-Only (Daily Job) | Online Learning (MAB) | Hybrid (Batch + Real-Time) |
|-------------------------|--------------------------------|------------------------|-----------------------|----------------------------|
| **Personalization Lag** | <1 minute                      | 24 hours               | <1 second             | <1 minute                  |
| **Implementation**      | Medium (Kafka Streams + Redis) | Simple (Spark job)     | Complex (bandits)     | Medium (both systems)      |
| **Accuracy**            | Good (recent + historical)     | Good (historical only) | Medium (exploration)  | Excellent (best of both)   |
| **Infrastructure Cost** | Medium                         | Low                    | Low                   | High                       |
| **Cold Start**          | Good (immediate signal)        | Poor (wait 24h)        | Excellent             | Good                       |
| **Trending Topics**     | Yes (real-time events)         | No (missed)            | Yes                   | Yes                        |
| **Ops Complexity**      | Medium                         | Low                    | High                  | High                       |

### Decision Made

**Use hybrid personalization: Batch ML (daily job) + Real-Time Feature Store (immediate actions).**

### Rationale

1. **Immediate Personalization**
    - User actions (clicks, shares) are reflected in recommendations within 1 minute
    - Real-time feature store captures recent interests (last 1-7 days)
    - Batch model captures long-term preferences (historical data)
    - Combined score: `0.7 × batch + 0.3 × realtime`

2. **Handles Trending Topics**
    - Breaking news: "Earthquake in Tokyo"
    - Real-time feature store detects spike in clicks (windowed aggregation)
    - User who clicks on earthquake articles immediately sees more related content
    - Batch model would miss this (24-hour lag)

3. **Solves Cold Start Problem**
    - New user signs up at 10 AM, clicks on 5 Tech articles
    - Real-time feature store: `user:123:recent_topics` → {Tech: 5}
    - Next feed request: Tech articles ranked higher (immediate feedback)
    - Batch-only: User sees generic recommendations until next day

4. **Cost-Benefit Balance**
    - Real-time system adds cost (Kafka Streams + Redis memory)
    - But improves engagement significantly (A/B test: +15% CTR)
    - Online learning (multi-armed bandits) would be more complex (no clear winner)

5. **Best of Both Worlds**
    - Batch model: Accurate, stable, captures long-term patterns
    - Real-time store: Fast, adapts to immediate changes, handles trends
    - Hybrid combines strengths of both

### Implementation Details

**Real-Time Feature Store Architecture:**

1. **Event Stream (Kafka)**
   ```
   Topic: user.actions
   Events: click, share, like, dwell_time
   Partition key: user_id (for stateful processing)
   Throughput: 50,000 events/sec
   ```

2. **Stream Processor (Kafka Streams)**
   ```
   Stateful processing:
   - Window: Rolling 1-hour (for recent activity)
   - State store: user_topic_counts (in-memory)
   - Operations:
     - Count clicks per topic
     - Track recently read articles (last 20)
     - Compute recency score (exponential decay)
   ```

3. **Feature Store (Redis)**
   ```
   Key: user:123:recent_topics
   Type: Sorted Set (topic, score)
   Value: {Tech: 5, Business: 2, Sports: 1}
   TTL: 7 days
   
   Key: user:123:recent_reads
   Type: List (article IDs)
   Value: [xyz-789, abc-123, ...]
   Max length: 20
   TTL: 7 days
   ```

4. **Feed Service Integration**
   ```
   1. Fetch batch embeddings from Redis
      user_embedding = GET user:123:embedding  // 128D vector
   
   2. Fetch real-time features from Redis
      recent_topics = ZREVRANGE user:123:recent_topics 0 10
      recent_reads = LRANGE user:123:recent_reads 0 20
   
   3. Compute combined score
      batch_score = user_embedding · article_embedding
      realtime_score = topic_match_score(recent_topics, article.topics)
      
      final_score = 0.7 × batch_score + 0.3 × realtime_score
   
   4. Boost trending topics by 50%
      if article.topic in trending_topics:
        final_score *= 1.5
   ```

**Performance:**

- Event-to-feature latency: <100ms (p99)
- Redis read latency: <1ms
- Combined scoring: <5ms
- Total feed latency: <50ms (p99)

### Trade-offs Accepted

| What We Gain                         | What We Sacrifice                                    |
|--------------------------------------|------------------------------------------------------|
| ✅ Immediate personalization (<1 min) | ❌ Higher infrastructure cost (Kafka Streams + Redis) |
| ✅ Handles trending topics            | ❌ More complex architecture (two systems)            |
| ✅ Solves cold start problem          | ❌ Requires careful weight tuning (70/30 split)       |
| ✅ +15% CTR improvement               | ❌ More operational complexity (monitor two systems)  |
| ✅ Best of batch + real-time          | ❌ Eventual consistency between systems               |

### When to Reconsider

- **If infrastructure cost is prohibitive** → Use batch-only (simpler, cheaper)
- **If latency requirements are relaxed (1 hour acceptable)** → Use batch-only
- **If we need <1 second personalization** → Use online learning (MAB, contextual bandits)
- **If feature store is too complex** → Use batch with higher frequency (every hour)

---

## 7. Multi-Region Read Replicas Over Single-Region with CDN Only

### The Problem

Global news feed users are distributed worldwide:

- US users: 40% of traffic
- EU users: 30% of traffic
- Asia-Pacific users: 30% of traffic
- Single-region (US-East) causes high latency for EU/AP users (200-300ms)

### Options Considered

| Aspect              | Multi-Region (Read Replicas) | Single-Region + CDN    | Multi-Region (Active-Active) | Edge Compute       |
|---------------------|------------------------------|------------------------|------------------------------|--------------------|
| **Read Latency**    | Low (50-80ms)                | Medium (200-300ms)     | Very Low (30-50ms)           | Very Low (20-30ms) |
| **Write Latency**   | Medium (writes to primary)   | Low (single region)    | Low (multi-master)           | N/A (read-only)    |
| **Consistency**     | Eventual (1h lag ES)         | Strong (single source) | Eventual (conflicts)         | Eventual           |
| **Cost**            | Medium (3x infrastructure)   | Low (single region)    | High (3x + sync)             | Medium             |
| **Ops Complexity**  | Medium (replica management)  | Low (single cluster)   | High (conflict resolution)   | Medium             |
| **Availability**    | High (region failover)       | Medium (single region) | Very High                    | High               |
| **GDPR Compliance** | Easy (regional data)         | Hard (data in US)      | Easy                         | Easy               |

### Decision Made

**Use multi-region deployment with read replicas (active-passive replication).**

**Architecture:**

- **Primary Region:** US-East (all writes)
- **Secondary Regions:** EU-West, AP-Southeast (read replicas)
- **Replication:** Kafka (MirrorMaker 2), Elasticsearch (snapshot/restore), Redis (active-passive)

### Rationale

1. **Significantly Lower Read Latency**
    - US users: 30ms (nearest region)
    - EU users: 50ms (vs. 200ms single-region)
    - AP users: 80ms (vs. 300ms single-region)
    - CDN alone can't cache personalized feeds (unique per user)

2. **Global Availability**
    - If US-East region fails, can failover to EU-West (manual process)
    - Each region can serve users independently (read-only)
    - Single-region has no failover (complete outage)

3. **GDPR Compliance**
    - EU users' data can be stored in EU region (data residency)
    - Single-region (US) violates GDPR requirements
    - Important for legal compliance

4. **Cost-Benefit Analysis**
   ```
   Single-Region Cost: $50k/month (US-East)
   Multi-Region Cost: $120k/month (US + EU + AP)
   
   Additional cost: $70k/month
   
   User impact:
   - 60% of users (EU + AP) experience 50-70% latency reduction
   - Engagement improvement: +10% CTR (better UX)
   - Revenue increase: $200k/month (+10% CTR × $2M revenue)
   
   ROI: ($200k - $70k) / $70k = 1.86 (positive ROI)
   ```

5. **Active-Active Too Complex**
    - Active-active (multi-master) requires conflict resolution
    - News feed has complex state (trending topics, personalization)
    - Not worth the complexity for read-heavy workload (95% reads)

### Implementation Details

**Multi-Region Architecture:**

**Primary Region (US-East):**

- All writes: Ingestion, user actions, ML training
- Full Kafka cluster (all topics)
- Full Elasticsearch cluster (primary shards)
- Full Redis cluster (primary)

**Secondary Regions (EU-West, AP-Southeast):**

- Reads only: Feed serving, search queries
- Partial Kafka (user.actions topic only)
- Elasticsearch read replicas (snapshot/restore from S3)
- Redis read replicas (active-passive replication)

**Replication Strategy:**

1. **Kafka Cross-Region Replication (MirrorMaker 2)**
   ```
   Source: US-East Kafka cluster
   Targets: EU-West, AP-Southeast
   Topics replicated: user.actions (for real-time feature store)
   Lag: 2-5 seconds (acceptable)
   Direction: One-way (primary → secondaries)
   ```

2. **Elasticsearch Cross-Cluster Replication**
   ```
   Strategy: Snapshot & Restore (via S3)
   Frequency: Every 1 hour
   Snapshot size: 50GB (compressed)
   Restore time: 30 minutes
   Lag: Up to 1 hour (acceptable for news feed)
   
   Process:
   1. US-East ES → Snapshot to S3 (every hour)
   2. S3 → Replicate to EU/AP regions (S3 cross-region replication)
   3. EU/AP ES → Restore from S3 snapshot
   ```

3. **Redis Active-Passive Replication**
   ```
   Strategy: Redis Streams replication
   Source: US-East Redis cluster
   Targets: EU-West, AP-Southeast
   Lag: <1 second (fast)
   Data replicated: Hot articles (last 24h), user embeddings
   
   Implementation:
   - Redis Streams (XADD) → Replicate to secondary regions
   - Secondary regions: Read-only replicas
   ```

**Geo-Routing (Route 53):**

```
DNS: api.newsfeed.com

Routing policy: Geolocation
- User in US → us-east.api.newsfeed.com (52.70.XXX.XXX)
- User in EU → eu-west.api.newsfeed.com (52.220.XXX.XXX)
- User in AP → ap-southeast.api.newsfeed.com (13.229.XXX.XXX)

Fallback: US-East (if no regional match)
```

### Trade-offs Accepted

| What We Gain                          | What We Sacrifice                           |
|---------------------------------------|---------------------------------------------|
| ✅ 50-70% latency reduction (EU/AP)    | ❌ 2.4x higher infrastructure cost           |
| ✅ Region failover (high availability) | ❌ Eventual consistency (up to 1h lag ES)    |
| ✅ GDPR compliance (EU data residency) | ❌ More ops complexity (manage 3 regions)    |
| ✅ +10% CTR improvement (better UX)    | ❌ Cross-region replication lag (2-5s Kafka) |
| ✅ Positive ROI ($130k/month gain)     | ❌ Multi-region monitoring/troubleshooting   |

### When to Reconsider

- **If budget is very limited** → Use single-region + CDN (cheaper)
- **If latency requirements are relaxed (200ms acceptable)** → Use single-region
- **If strong consistency is required** → Use single-region (no replication lag)
- **If we need multi-master writes** → Use active-active (more complex, higher cost)
- **If we can use edge compute** → Use CloudFlare Workers or Lambda@Edge (simpler)

---

## Summary Table: All Design Decisions

| Decision                    | Chosen Approach           | Alternative             | Key Reason                             |
|-----------------------------|---------------------------|-------------------------|----------------------------------------|
| **1. Search & Storage**     | Elasticsearch             | PostgreSQL              | Full-text search performance (<100ms)  |
| **2. Architecture Pattern** | Kappa (Stream-Only)       | Lambda (Batch + Stream) | Low latency + single codebase          |
| **3. Deduplication**        | LSH (MinHash)             | Edit Distance           | Near-duplicate detection + low latency |
| **4. Write Buffering**      | Redis Streams             | Direct ES Writes        | Smooth write spikes + bulk efficiency  |
| **5. ML Model**             | Hybrid (Collab + Content) | Pure Collaborative      | Cold start + diversity                 |
| **6. Personalization**      | Batch + Real-Time         | Batch-Only              | Immediate feedback + trending topics   |
| **7. Global Deployment**    | Multi-Region Replicas     | Single-Region + CDN     | Low latency (EU/AP) + GDPR             |