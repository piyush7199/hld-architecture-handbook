# Global News Feed - Sequence Diagrams

## Table of Contents

1. [Article Ingestion Flow (RSS)](#1-article-ingestion-flow-rss)
2. [Article Deduplication Flow](#2-article-deduplication-flow)
3. [NLP Processing Pipeline](#3-nlp-processing-pipeline)
4. [Elasticsearch Bulk Indexing Flow](#4-elasticsearch-bulk-indexing-flow)
5. [Personalized Feed Request (Happy Path)](#5-personalized-feed-request-happy-path)
6. [Real-Time Feature Store Update](#6-real-time-feature-store-update)
7. [Trending Topics Detection](#7-trending-topics-detection)
8. [Batch ML Training Job](#8-batch-ml-training-job)
9. [Cache Miss Fallback Flow](#9-cache-miss-fallback-flow)
10. [Multi-Region Read Request](#10-multi-region-read-request)
11. [Cache Invalidation Flow](#11-cache-invalidation-flow)
12. [Failure Recovery - Kafka Consumer Lag](#12-failure-recovery---kafka-consumer-lag)

---

## 1. Article Ingestion Flow (RSS)

**Flow:**

Shows the complete flow of ingesting an RSS article from external source to Kafka topic.

**Steps:**

1. **Scheduler (0ms)** - Cron job triggers polling for RSS feed (every 5 minutes)
2. **Crawler (10ms)** - HTTP GET request to fetch RSS XML feed
3. **Rate Limiter (1ms)** - Check token bucket, allow if tokens available
4. **Parser (50ms)** - Parse XML, extract title, body, published_at, author
5. **Validator (10ms)** - Validate required fields, detect language
6. **Kafka Producer (5ms)** - Publish to `article.raw` topic with partition key = source_id
7. **Scheduler (76ms total)** - Mark source as successfully polled, schedule next poll

**Performance:**

- Total ingestion latency: ~76ms per article
- Throughput: 1,157 articles/sec sustained
- Failure handling: Retry with exponential backoff (max 3 retries)

**Edge Cases:**

- RSS feed unavailable (404/500) → Retry after 1 hour
- Malformed XML → Log error, skip article
- Duplicate URL detected → Skip (will be caught by deduplication stage)

```mermaid
sequenceDiagram
    participant Scheduler
    participant Crawler
    participant RateLimiter
    participant Parser
    participant Validator
    participant KafkaProducer
    participant Kafka
    Scheduler ->> Scheduler: Cron job triggers (0ms)
    Scheduler ->> Crawler: Poll RSS feed<br/>URL: cnn.com/rss
    Crawler ->> RateLimiter: Check rate limit<br/>source: CNN
    RateLimiter -->> Crawler: Allowed (tokens: 10/10)
    Crawler ->> Crawler: HTTP GET request (10ms)
    Note over Crawler: User-Agent: NewsAggregator/1.0
    Crawler -->> Parser: RSS XML response (10ms)
    Parser ->> Parser: Parse XML (50ms)
    Parser ->> Parser: Extract items
    Note over Parser: title, link, description<br/>pubDate, author
    Parser -->> Validator: Structured article (50ms)
    Validator ->> Validator: Validate fields (10ms)
    Validator ->> Validator: Detect language (English)
    Validator ->> Validator: Check required fields

    alt Valid Article
        Validator -->> KafkaProducer: Article object (60ms)
        KafkaProducer ->> Kafka: Publish to article.raw<br/>Partition: hash(source_id) (5ms)
        Kafka -->> KafkaProducer: Ack (offset: 12345)
        KafkaProducer -->> Scheduler: Success (76ms total)
        Scheduler ->> Scheduler: Update last_poll_time<br/>Schedule next poll: +5min
    else Invalid Article
        Validator -->> Scheduler: Validation error (60ms)
        Scheduler ->> Scheduler: Log error<br/>Skip article
    end

    Note over Scheduler, Kafka: Throughput: 1,157 articles/sec<br/>Failure: Retry with exponential backoff
```

---

## 2. Article Deduplication Flow

**Flow:**

Shows the two-stage deduplication process using Bloom Filter and LSH to identify exact and near-duplicate articles.

**Steps:**

1. **Dedup Service (0ms)** - Consume from Kafka `article.raw` topic
2. **Stage 1: URL Hash Check (1ms)** - Compute SHA-256 hash of URL
3. **Bloom Filter Lookup (1ms)** - Check if URL hash exists in Bloom Filter
4. **Stage 2: Content Similarity (50ms)** - If new URL, compute LSH signature
5. **LSH Index Query (20ms)** - Search for similar articles (threshold: 85%)
6. **Decision (71ms total)**:
    - **Duplicate found** → Group with existing story_id, publish to `article.deduplicated`
    - **New article** → Assign new story_id, add to LSH index, publish to `article.deduplicated`

**Performance:**

- Bloom Filter check: <1ms (in-memory)
- LSH similarity search: 10-50ms (Redis Sorted Set query)
- Overall deduplication latency: <100ms (p99)
- Accuracy: 99.9% (0.1% false positives)

**Edge Cases:**

- False positive (Bloom Filter) → Stage 2 LSH catches it
- Near-duplicate but <85% similarity → Treated as new article (acceptable)
- High-traffic burst → Redis read replicas handle load

```mermaid
sequenceDiagram
    participant Kafka
    participant DedupService
    participant BloomFilter
    participant LSHService
    participant LSHIndex
    participant StoryDB
    participant KafkaOut
    Kafka ->> DedupService: Consume article (0ms)<br/>Topic: article.raw
    DedupService ->> DedupService: Extract URL (1ms)
    DedupService ->> DedupService: Compute SHA-256 hash
    DedupService ->> BloomFilter: Check URL hash (1ms)
    BloomFilter -->> DedupService: Exists? (Boolean)

    alt URL Exists (Exact Duplicate)
        DedupService ->> StoryDB: Lookup story_id by URL
        StoryDB -->> DedupService: story_id: abc-123
        DedupService ->> KafkaOut: Publish (duplicate=true)<br/>story_id: abc-123
        Note over DedupService: Skip LSH stage<br/>Total: 10ms
    else New URL
        DedupService ->> LSHService: Compute LSH signature (50ms)
        LSHService ->> LSHService: Extract text content
        LSHService ->> LSHService: Generate shingles (n-grams)
        LSHService ->> LSHService: Compute MinHash (128 hashes)
        LSHService -->> DedupService: LSH signature (50ms)
        DedupService ->> LSHIndex: Query similar articles (20ms)<br/>Redis: ZRANGEBYSCORE
        LSHIndex -->> DedupService: Similar articles<br/>(cosine similarity > 0.85)

        alt Similar Article Found (Near-Duplicate)
            DedupService ->> StoryDB: Get story_id from similar article
            StoryDB -->> DedupService: story_id: xyz-789
            DedupService ->> StoryDB: Add article to story cluster
            DedupService ->> KafkaOut: Publish (duplicate=true)<br/>story_id: xyz-789 (71ms)
        else No Similar Article (New Story)
            DedupService ->> DedupService: Generate new story_id
            DedupService ->> StoryDB: Create new story cluster
            DedupService ->> LSHIndex: Add LSH signature<br/>Redis: ZADD
            DedupService ->> BloomFilter: Add URL hash
            DedupService ->> KafkaOut: Publish (duplicate=false)<br/>story_id: new-456 (71ms)
        end
    end

    Note over Kafka, KafkaOut: Throughput: 1,157 articles/sec<br/>Accuracy: 99.9%<br/>False positives: 0.1%
```

---

## 3. NLP Processing Pipeline

**Flow:**

Shows the parallel NLP processing pipeline that extracts keywords, entities, sentiment, topics, and quality scores.

**Steps:**

1. **NLP Service (0ms)** - Consume from Kafka `article.deduplicated` topic
2. **Tokenization (20ms)** - Split text into sentences and words using spaCy
3. **Parallel Processing (200ms)** - Run 4 NLP tasks in parallel:
    - **Keyword Extraction** - TF-IDF, top 20 keywords
    - **Named Entity Recognition** - Extract people, organizations, locations
    - **Sentiment Analysis** - VADER or BERT, score: -1 to +1
    - **Topic Classification** - BERT multi-label classifier
4. **Quality Scoring (50ms)** - Compute quality score based on source reputation, grammar, readability
5. **Output (270ms total)** - Publish enriched article to Kafka `article.nlp` topic

**Performance:**

- Processing time per article: 200-500ms
- GPU acceleration: 8x NVIDIA T4 GPUs for BERT models
- Batch processing: 32 articles per batch
- Throughput: 20 NLP workers × 5 articles/sec = 100 articles/sec per worker
- Total capacity: 2,000 articles/sec

**Edge Cases:**

- Non-English article → Use multilingual BERT model (slower)
- Very long article (>10,000 words) → Truncate to first 2,000 words
- GPU OOM error → Fallback to CPU (10x slower)

```mermaid
sequenceDiagram
    participant Kafka
    participant NLPService
    participant Tokenizer
    participant KeywordExtractor
    participant NER
    participant SentimentAnalyzer
    participant TopicClassifier
    participant QualityScorer
    participant KafkaOut
    Kafka ->> NLPService: Consume article (0ms)<br/>Topic: article.deduplicated
    NLPService ->> Tokenizer: Extract text (20ms)
    Tokenizer ->> Tokenizer: spaCy tokenization
    Tokenizer ->> Tokenizer: Sentence splitting
    Tokenizer ->> Tokenizer: Word tokenization
    Tokenizer -->> NLPService: Tokens (20ms)

    par Parallel NLP Tasks (200ms)
        NLPService ->> KeywordExtractor: Extract keywords
        KeywordExtractor ->> KeywordExtractor: Compute TF-IDF
        KeywordExtractor -->> NLPService: Top 20 keywords<br/>[AI, machine learning, GPT]
        NLPService ->> NER: Named Entity Recognition
        NER ->> NER: spaCy NER model
        NER -->> NLPService: Entities<br/>People: [Elon Musk]<br/>Org: [Tesla]<br/>Location: [California]
        NLPService ->> SentimentAnalyzer: Sentiment analysis
        SentimentAnalyzer ->> SentimentAnalyzer: BERT sentiment model<br/>GPU inference
        SentimentAnalyzer -->> NLPService: Sentiment score: 0.75<br/>(Positive)
        NLPService ->> TopicClassifier: Topic classification
        TopicClassifier ->> TopicClassifier: BERT multi-label<br/>GPU inference
        TopicClassifier -->> NLPService: Topics: [Tech, Business]<br/>Confidence: 0.92
    end

    NLPService ->> QualityScorer: Compute quality score (50ms)
    QualityScorer ->> QualityScorer: Check source reputation<br/>(CNN: 85/100)
    QualityScorer ->> QualityScorer: Grammar check<br/>(LanguageTool)
    QualityScorer ->> QualityScorer: Readability score<br/>(Flesch-Kincaid)
    QualityScorer -->> NLPService: Quality: 82/100 (270ms)
    NLPService ->> NLPService: Assemble enriched article
    NLPService ->> KafkaOut: Publish to article.nlp (270ms)
    KafkaOut -->> Kafka: Topic: article.nlp<br/>Partition: hash(story_id)
    Note over Kafka, KafkaOut: Throughput: 2,000 articles/sec<br/>GPU: 8x NVIDIA T4<br/>Batch size: 32
```

---

## 4. Elasticsearch Bulk Indexing Flow

**Flow:**

Shows the write buffer pattern using Redis Streams to absorb burst traffic before bulk writing to Elasticsearch.

**Steps:**

1. **Indexer Worker (0ms)** - Consume from Kafka `article.nlp` topic
2. **Write to Redis Streams (1ms)** - XADD to stream `es:write:buffer`
3. **Bulk Writer (5000ms)** - Poll Redis Streams every 5 seconds or when 1000 messages accumulated
4. **Read Batch (10ms)** - XREADGROUP to read 1000 messages
5. **Transform (50ms)** - Convert to Elasticsearch bulk format
6. **Bulk Write (200ms)** - POST to Elasticsearch `_bulk` API
7. **Acknowledge (5ms)** - XACK messages in Redis Streams

**Performance:**

- Write to Redis: <1ms per article
- Bulk write latency: 200-500ms for 1000 docs
- End-to-end indexing delay: <10 seconds (p99)
- Throughput: 1,500 docs/sec sustained, 5,000 docs/sec peak

**Benefits:**

- Smooths write spikes (burst of 5,000/sec absorbed, written at 1,500/sec)
- Efficient bulk writes (30x better throughput than individual writes)
- Backpressure handling (if ES is slow, buffer grows temporarily)

**Edge Cases:**

- Redis Streams full (>100k messages) → Reject new writes, return 503 Service Unavailable
- Elasticsearch bulk error (partial failure) → Retry failed documents
- Bulk writer crash → Consumer group rebalance, another worker takes over

```mermaid
sequenceDiagram
    participant Kafka
    participant IndexerWorker
    participant RedisStreams
    participant BulkWriter
    participant Elasticsearch
    participant Monitoring
    Kafka ->> IndexerWorker: Consume article (0ms)<br/>Topic: article.nlp
    IndexerWorker ->> IndexerWorker: Transform to ES document (5ms)
    IndexerWorker ->> RedisStreams: XADD es:write:buffer (1ms)
    RedisStreams -->> IndexerWorker: Message ID: 1234567-0
    IndexerWorker -->> Kafka: Commit offset (6ms)
    Note over RedisStreams: Buffer accumulates messages<br/>Wait for batch (1000 msgs or 5 sec)
    BulkWriter ->> RedisStreams: XREADGROUP (5000ms)<br/>Consumer group: es:bulk:writers<br/>Count: 1000
    RedisStreams -->> BulkWriter: 1000 messages (10ms)
    BulkWriter ->> BulkWriter: Transform to bulk format (50ms)
    Note over BulkWriter: Bulk format:<br/>{"index": {"_index": "news-2025-10"}}<br/>{"title": "...", "body": "..."}
    BulkWriter ->> Elasticsearch: POST /_bulk (200ms)<br/>1000 documents

    alt Bulk Success
        Elasticsearch -->> BulkWriter: 200 OK<br/>{"errors": false} (200ms)
        BulkWriter ->> RedisStreams: XACK messages (5ms)
        BulkWriter ->> Monitoring: Metric: bulk_write_success<br/>Count: 1000
    else Partial Failure
        Elasticsearch -->> BulkWriter: 200 OK<br/>{"errors": true, "items": [...]}
        BulkWriter ->> BulkWriter: Extract failed docs<br/>(50 failures)
        BulkWriter ->> RedisStreams: XADD failed docs<br/>Retry queue (5ms)
        BulkWriter ->> RedisStreams: XACK successful msgs (5ms)
        BulkWriter ->> Monitoring: Metric: bulk_write_partial_failure<br/>Failed: 50
    else Elasticsearch Unavailable
        Elasticsearch -->> BulkWriter: 503 Service Unavailable
        BulkWriter ->> BulkWriter: DO NOT XACK<br/>Messages remain in stream
        BulkWriter ->> Monitoring: Alert: ES cluster down
        Note over BulkWriter: Wait 10 seconds, retry
    end

    Note over Kafka, Monitoring: Throughput: 1,500 docs/sec sustained<br/>Peak: 5,000 docs/sec<br/>Indexing delay: <10 seconds (p99)
```

---

## 5. Personalized Feed Request (Happy Path)

**Flow:**

Shows the complete request flow when a user opens their personalized news feed.

**Steps:**

1. **User Request (0ms)** - Mobile app sends GET /feed request
2. **CDN Check (5ms)** - CDN checks cache, MISS for personalized feed
3. **API Gateway (10ms)** - Validates JWT token, checks rate limit (100 req/hour)
4. **Feed Service (15ms)** - Fetches user embedding from Redis (batch ML model)
5. **Real-Time Features (20ms)** - Fetches recent clicks/topics from Redis (real-time feature store)
6. **Elasticsearch Query (40ms)** - Personalized query with function_score
7. **Article Enrichment (55ms)** - Fetch article content from Redis cache (80% hit rate)
8. **Response (60ms total)** - Return JSON with 50 articles

**Performance:**

- p50 latency: 30ms
- p99 latency: 80ms
- QPS: 300,000 peak
- Cache hit rate: 80% (articles), 95% (user embeddings)

**Query:**

```
function_score:
  - filter: topics IN [Tech, Business] (user interests)
  - filter: published_at > now-24h (recent articles)
  - function: field_value_factor(quality_score)
  - function: decay(published_at, scale=12h)
  - function: script(user_embedding · article_embedding)
```

```mermaid
sequenceDiagram
    participant User
    participant CDN
    participant APIGateway
    participant FeedService
    participant Redis
    participant Elasticsearch
    participant ArticleService
    User ->> CDN: GET /feed?user=123 (0ms)
    CDN ->> CDN: Check cache (5ms)
    Note over CDN: Cache key: /feed?user=123<br/>Result: MISS (personalized)
    CDN ->> APIGateway: Forward request (5ms)
    APIGateway ->> APIGateway: Validate JWT token (5ms)
    APIGateway ->> APIGateway: Rate limit check<br/>Limit: 100 req/hour<br/>Current: 47
    APIGateway ->> FeedService: Authenticated request (10ms)
    FeedService ->> Redis: GET user:123:embedding (5ms)
    Redis -->> FeedService: User vector [0.1, 0.3, ...] (15ms)
    FeedService ->> Redis: GET user:123:recent_topics (5ms)
    Redis -->> FeedService: [Tech: 10, Business: 5] (20ms)
    FeedService ->> FeedService: Compute preference score (5ms)
    Note over FeedService: Combine batch + realtime<br/>0.7 × batch + 0.3 × realtime
    FeedService ->> Elasticsearch: Personalized query (20ms)
    Note over FeedService, Elasticsearch: function_score query<br/>Filter: topics IN [Tech, Business]<br/>Boost: quality_score, recency<br/>Script: user_embed · article_embed
    Elasticsearch -->> FeedService: Top 50 article IDs (40ms)
    FeedService ->> Redis: MGET articles (batch read) (15ms)
    Redis -->> FeedService: 40 articles (80% hit) (55ms)
    FeedService ->> ArticleService: Fetch missing 10 articles (5ms)
    ArticleService ->> Elasticsearch: GET /news/_mget
    Elasticsearch -->> ArticleService: 10 articles
    ArticleService -->> FeedService: Article content (60ms)
    FeedService ->> FeedService: Assemble response<br/>Apply post-filters
    FeedService -->> APIGateway: JSON response (60ms)
    Note over FeedService: 50 articles<br/>[{title, snippet, image, source}]
    APIGateway -->> CDN: Cache response (TTL=5min)
    CDN -->> User: Feed displayed (60ms total)
    Note over User, Elasticsearch: p50 latency: 30ms<br/>p99 latency: 80ms<br/>QPS: 300,000 peak
```

---

## 6. Real-Time Feature Store Update

**Flow:**

Shows how user actions (clicks) are captured in real-time and stored in the feature store to overcome the 24-hour batch
ML lag.

**Steps:**

1. **User Action (0ms)** - User clicks on article (Tech category)
2. **User API (10ms)** - POST /action (click event)
3. **Kafka Producer (15ms)** - Publish to `user.actions` topic
4. **Kafka Streams (20ms)** - Consume event, update stateful aggregation
5. **Window Aggregation (25ms)** - Count clicks per topic (rolling 1-hour window)
6. **Redis Update (30ms)** - Update user's recent topics in Redis Sorted Set
7. **Feed Service (next request)** - Reads updated features, reflects user's immediate interest

**Performance:**

- Event-to-feature latency: <100ms (p99)
- Throughput: 50,000 events/sec
- Redis read latency: <1ms
- Feature TTL: 7 days

**Benefits:**

- Immediate personalization (user's actions reflected within 1 minute)
- Handles trending topics and viral articles
- Solves cold start problem

**Example:**

- User clicks 5 Tech articles in 10 minutes
- Feature store updates: `user:123:recent_topics` → {Tech: 5, Business: 2}
- Next feed request: Tech articles ranked higher

```mermaid
sequenceDiagram
    participant User
    participant UserAPI
    participant Kafka
    participant KafkaStreams
    participant Redis
    participant FeedService
    User ->> UserAPI: Click on article (0ms)<br/>Article ID: xyz-789<br/>Topic: Tech
    UserAPI ->> UserAPI: Validate request (5ms)<br/>Auth token, user_id
    UserAPI ->> Kafka: Publish event (10ms)
    Note over UserAPI, Kafka: Topic: user.actions<br/>Event: {<br/> user_id: 123,<br/> action: click,<br/> article_id: xyz-789,<br/> topic: Tech,<br/> timestamp: 1698765432<br/>}
    Kafka -->> UserAPI: Ack (offset: 56789) (15ms)
    UserAPI -->> User: 200 OK (15ms)
    Kafka ->> KafkaStreams: Consume event (20ms)
    KafkaStreams ->> KafkaStreams: Stateful processing (5ms)
    Note over KafkaStreams: Window: Rolling 1-hour<br/>State store: user_topic_counts<br/>Operation: COUNT clicks per topic
    KafkaStreams ->> KafkaStreams: Update aggregation (5ms)
    Note over KafkaStreams: user_id: 123<br/>topic_counts: {<br/> Tech: 5 (+1),<br/> Business: 2<br/>}
    KafkaStreams ->> Redis: Update feature store (5ms)
    Note over KafkaStreams, Redis: ZADD user:123:recent_topics<br/>Tech 5<br/>Business 2<br/>TTL: 7 days
    Redis -->> KafkaStreams: OK (30ms)
    Note over User, Redis: Next feed request (after 1 minute)
    User ->> FeedService: GET /feed (later)
    FeedService ->> Redis: GET user:123:recent_topics
    Redis -->> FeedService: {Tech: 5, Business: 2}
    FeedService ->> FeedService: Boost Tech articles by 30%
    FeedService -->> User: Personalized feed<br/>(Tech articles ranked higher)
    Note over User, Redis: Event-to-feature latency: <100ms (p99)<br/>Throughput: 50,000 events/sec<br/>Immediate personalization
```

---

## 7. Trending Topics Detection

**Flow:**

Shows how trending topics are detected in real-time using windowed aggregations and velocity tracking.

**Steps:**

1. **Event Stream (0ms)** - User actions (clicks, shares) published to Kafka
2. **Flink Job (5ms)** - Consume events, maintain windowed state
3. **Window Aggregation (5 min)** - Count events per topic per 1-hour sliding window
4. **Baseline Comparison (5 min)** - Compare to 24-hour average from PostgreSQL
5. **Trend Score Calculation (5 min)** - Compute trend_score = (current - baseline) / baseline
6. **Velocity Tracking (5 min)** - Compute acceleration (rate of change)
7. **Redis Update (5 min)** - Store trending topics in Sorted Set with trend_score
8. **Feed Service** - Reads trending topics, boosts articles by 50%

**Detection Latency:** <5 minutes (window update interval)

**Example:**

- Topic: "Earthquake in Tokyo"
- Baseline (24h avg): 100 clicks/hour
- Current (last hour): 10,000 clicks/hour
- Trend score: (10,000 - 100) / 100 = 99.0 (9900% increase)
- Result: Marked as TRENDING

**Algorithm:**

```
trend_score = (current_count - baseline) / baseline

If trend_score > 2.0 (200% increase):
  Mark as trending
```

```mermaid
sequenceDiagram
    participant Users
    participant Kafka
    participant FlinkJob
    participant BaselineDB
    participant Redis
    participant FeedService
    Users ->> Kafka: User actions (0ms)<br/>Topic: user.actions<br/>Events: click, share, like
    Kafka ->> FlinkJob: Consume events (5ms)
    FlinkJob ->> FlinkJob: Windowed aggregation (5 min)
    Note over FlinkJob: Window: Sliding 1-hour<br/>Update: Every 5 minutes<br/>Operation: COUNT per topic<br/>State: {<br/> Tech: 5000,<br/> Sports: 3000,<br/> "Earthquake Tokyo": 10000<br/>}
    FlinkJob ->> BaselineDB: Query 24h average (5 min)
    Note over FlinkJob, BaselineDB: SELECT topic, AVG(count)<br/>FROM topic_stats<br/>WHERE timestamp > now() - 24h<br/>GROUP BY topic
    BaselineDB -->> FlinkJob: Baseline counts (5 min)
    Note over BaselineDB: Tech: 4500<br/>Sports: 2800<br/>"Earthquake Tokyo": 100
    FlinkJob ->> FlinkJob: Compute trend score (5 min)
    Note over FlinkJob: trend_score = (current - baseline) / baseline<br/><br/>Tech: (5000 - 4500) / 4500 = 0.11<br/>Sports: (3000 - 2800) / 2800 = 0.07<br/>"Earthquake Tokyo": (10000 - 100) / 100 = 99.0
    FlinkJob ->> FlinkJob: Velocity tracking (5 min)
    Note over FlinkJob: Acceleration = Δ(trend_score) / Δt<br/>"Earthquake Tokyo": Fast-rising (acceleration > 10)
    FlinkJob ->> FlinkJob: Filter trending (5 min)
    Note over FlinkJob: threshold: trend_score > 2.0<br/>Result: ["Earthquake Tokyo": 99.0]
    FlinkJob ->> Redis: Update trending topics (5 min)
    Note over FlinkJob, Redis: ZADD trending:topics:global<br/>"Earthquake Tokyo" 99.0<br/>TTL: 2 hours
    Redis -->> FlinkJob: OK (5 min)
    Note over Users, FeedService: Feed request (after 5 min)
    FeedService ->> Redis: ZREVRANGE trending:topics:global 0 10
    Redis -->> FeedService: ["Earthquake Tokyo": 99.0]
    FeedService ->> FeedService: Boost trending articles<br/>50% ranking boost
    FeedService -->> Users: Feed with trending articles
    Note over Users, FeedService: Detection latency: <5 min<br/>Update frequency: Every 5 min<br/>Threshold: 200% increase
```

---

## 8. Batch ML Training Job

**Flow:**

Shows the daily batch ML training pipeline for user personalization using collaborative filtering.

**Steps:**

1. **Scheduler (2 AM UTC)** - Cron job triggers daily ML training
2. **Data Collection (30 min)** - Spark job reads last 24 hours of user actions from Kafka
3. **Feature Engineering (1 hour)** - Compute user vectors and article vectors
4. **Model Training (2 hours)** - Train collaborative filtering model (ALS algorithm)
5. **Model Evaluation (30 min)** - Compute offline metrics (Precision, Recall, NDCG)
6. **Model Deployment (30 min)** - Blue-green deployment to production
7. **Embedding Update (30 min)** - Store user/article embeddings in Redis

**Total Runtime:** 2-4 hours

**Performance:**

- Training data: 100M user actions/day
- Model size: 10GB (100M users × 128D embeddings)
- Infrastructure: 50 Spark executors (4 cores, 16GB each)

**Limitations:**

- 24-hour lag (user's actions today won't affect recommendations until tomorrow)
- Mitigated by Real-Time Feature Store

```mermaid
sequenceDiagram
    participant Scheduler
    participant SparkJob
    participant Kafka
    participant UserDB
    participant ArticleDB
    participant MLTraining
    participant ModelRegistry
    participant Redis
    Scheduler ->> SparkJob: Trigger daily job (2 AM UTC)
    SparkJob ->> Kafka: Read user actions (30 min)
    Note over SparkJob, Kafka: Topic: user.actions<br/>Timerange: Last 24 hours<br/>Data: 100M events
    Kafka -->> SparkJob: User action events
    SparkJob ->> UserDB: Read user profiles
    UserDB -->> SparkJob: User demographics, interests
    SparkJob ->> ArticleDB: Read article metadata
    ArticleDB -->> SparkJob: Topics, keywords, quality_score
    SparkJob ->> SparkJob: Feature engineering (1 hour)
    Note over SparkJob: User features:<br/>- Read history vector<br/>- Topic preferences<br/>- Recency weighting<br/><br/>Article features:<br/>- TF-IDF vector<br/>- Entity vector<br/>- Quality score<br/><br/>Interaction features:<br/>- CTR per topic<br/>- Dwell time
    SparkJob ->> MLTraining: Train collaborative filtering (2 hours)
    MLTraining ->> MLTraining: Matrix factorization (ALS)
    Note over MLTraining: Algorithm: Alternating Least Squares<br/>User-Article matrix (100M × 100M)<br/>Latent factors: 128 dimensions<br/>Iterations: 10<br/>Regularization: 0.1
    MLTraining ->> MLTraining: Train content-based model
    Note over MLTraining: Cosine similarity<br/>User profile vs Article TF-IDF
    MLTraining ->> MLTraining: Ensemble hybrid model
    Note over MLTraining: Weighted combination:<br/>0.6 × collaborative + 0.4 × content-based
    MLTraining -->> SparkJob: Trained model
    SparkJob ->> SparkJob: Model evaluation (30 min)
    Note over SparkJob: Offline metrics:<br/>- Precision@10: 0.78<br/>- Recall@10: 0.45<br/>- NDCG@10: 0.82<br/><br/>Compare to baseline:<br/>If metrics improved, proceed

    alt Metrics Improved
        SparkJob ->> ModelRegistry: Save model (30 min)
        Note over SparkJob, ModelRegistry: S3 bucket: ml-models/<br/>Version: v2025-10-29<br/>Metadata: {metrics, config}
        ModelRegistry -->> SparkJob: Model saved
        SparkJob ->> SparkJob: Blue-green deployment (30 min)
        Note over SparkJob: Deploy to 10% canary<br/>Monitor for 1 hour<br/>Full rollout if successful
        SparkJob ->> Redis: Update embeddings (30 min)
        Note over SparkJob, Redis: HSET user:123:embedding [0.1, 0.3, ...]<br/>HSET article:xyz:embedding [0.2, 0.4, ...]<br/>TTL: 48 hours<br/>Total: 10GB data
        Redis -->> SparkJob: Embeddings stored
        SparkJob -->> Scheduler: Job completed (total: 4 hours)
    else Metrics Degraded
        SparkJob ->> SparkJob: Rollback to previous model
        SparkJob -->> Scheduler: Job failed (alert on-call)
    end

    Note over Scheduler, Redis: Total runtime: 2-4 hours<br/>Training data: 100M actions/day<br/>Model size: 10GB<br/>Limitation: 24-hour lag
```

---

## 9. Cache Miss Fallback Flow

**Flow:**

Shows the fallback flow when cache misses occur at multiple layers (CDN, Redis, Elasticsearch).

**Steps:**

1. **User Request (0ms)** - GET /feed request
2. **CDN Miss (10ms)** - Personalized feed not cached at edge
3. **Redis Miss (30ms)** - User embedding not in cache (cold user)
4. **Elasticsearch Query (100ms)** - Query user embedding from dedicated index
5. **Redis Miss (130ms)** - Article content not in cache (cold articles)
6. **Elasticsearch Query (200ms)** - Fetch full article documents
7. **Backfill Cache (250ms)** - Store fetched data in Redis for next request
8. **Response (300ms total)** - Return to user (slow path)

**Performance:**

- Cache hit latency: 30ms (p50)
- Cache miss latency: 300ms (p99)
- Cache miss rate: 20% (articles), 5% (user embeddings)

**Cache Miss Scenarios:**

- **Cold user** - New user, no embedding in cache
- **Cold article** - New article just published, not yet cached
- **Cache eviction** - LRU eviction due to memory pressure
- **Cache invalidation** - Article updated, cache intentionally purged

**Mitigation:**

- **Cache warming** - Pre-populate cache with popular articles
- **TTL tuning** - Balance freshness vs hit rate
- **Cache tiering** - Hot (Redis) + Warm (Elasticsearch request cache) + Cold (Elasticsearch disk)

```mermaid
sequenceDiagram
    participant User
    participant CDN
    participant APIGateway
    participant FeedService
    participant Redis
    participant Elasticsearch
    User ->> CDN: GET /feed?user=999 (0ms)<br/>New user
    CDN ->> CDN: Check cache (10ms)
    Note over CDN: Cache key: /feed?user=999<br/>Result: MISS (personalized)
    CDN ->> APIGateway: Forward request (10ms)
    APIGateway ->> FeedService: Authenticated request (20ms)
    FeedService ->> Redis: GET user:999:embedding (10ms)
    Redis -->> FeedService: MISS (cold user) (30ms)
    Note over Redis: User not in cache<br/>(new user or evicted)
    FeedService ->> Elasticsearch: Query user embedding (70ms)
    Note over FeedService, Elasticsearch: GET /users/_doc/999<br/>Index: users<br/>Field: embedding_vector
    Elasticsearch -->> FeedService: User embedding (100ms)
    Note over Elasticsearch: Fallback: If not found,<br/>use default embedding<br/>(average of all users)
    FeedService ->> Redis: SET user:999:embedding (5ms)
    Note over FeedService: Backfill cache<br/>TTL: 24 hours
    FeedService ->> Elasticsearch: Personalized query (30ms)
    Elasticsearch -->> FeedService: Top 50 article IDs (130ms)
    FeedService ->> Redis: MGET articles (20ms)
    Redis -->> FeedService: MISS for 30 articles (150ms)
    Note over Redis: Cold articles<br/>(just published)
    FeedService ->> Elasticsearch: GET /news/_mget (50ms)
    Note over FeedService, Elasticsearch: Batch read: 30 articles<br/>_source includes:<br/>title, body, image, source
    Elasticsearch -->> FeedService: Article content (200ms)
    FeedService ->> Redis: MSET articles (10ms)
    Note over FeedService: Backfill cache<br/>TTL: 1 hour<br/>30 articles stored
    FeedService ->> FeedService: Assemble response (50ms)
    FeedService -->> APIGateway: JSON response (250ms)
    APIGateway -->> CDN: Cache response (TTL=5min)
    CDN -->> User: Feed displayed (300ms total)
    Note over User, Elasticsearch: Cache miss latency: 300ms (p99)<br/>Cache hit latency: 30ms (p50)<br/>Cache miss rate: 20% articles, 5% users<br/><br/>Next request: 30ms (cached)
```

---

## 10. Multi-Region Read Request

**Flow:**

Shows how a user request is routed to the nearest region for low-latency access.

**Steps:**

1. **User Request (Tokyo, 0ms)** - User in Tokyo sends GET /feed request
2. **Route 53 Geo-Routing (10ms)** - DNS resolves to AP-Southeast region
3. **Regional CDN (20ms)** - CloudFront PoP in Tokyo serves cached content
4. **Regional API Gateway (30ms)** - Asia-Pacific API Gateway
5. **Regional Feed Service (50ms)** - Reads from regional Elasticsearch read replica
6. **Regional Redis (60ms)** - Fetches from regional Redis replica
7. **Response (80ms total)** - User receives feed (low latency)

**Multi-Region Architecture:**

- **Primary Region:** US-East (all writes)
- **Secondary Regions:** EU-West, AP-Southeast (read replicas)

**Replication Lag:**

- Kafka: 2-5 seconds (MirrorMaker 2)
- Elasticsearch: up to 1 hour (snapshot & restore)
- Redis: <1 second (active-passive replication)

**Performance:**

- US users: 30ms latency
- EU users: 50ms latency
- AP users: 80ms latency (without multi-region: 300ms transpacific)

```mermaid
sequenceDiagram
    participant UserTokyo as User (Tokyo)
    participant Route53
    participant CDN_AP as CDN PoP<br/>(Tokyo)
    participant API_AP as API Gateway<br/>(AP-Southeast)
    participant Feed_AP as Feed Service<br/>(AP-Southeast)
    participant Redis_AP as Redis<br/>(AP Replica)
    participant ES_AP as Elasticsearch<br/>(AP Read Replica)
    UserTokyo ->> Route53: GET /feed (0ms)<br/>DNS query: api.newsfeed.com
    Route53 ->> Route53: Geo-routing (10ms)
    Note over Route53: Client IP: 203.0.113.1<br/>Location: Tokyo, Japan<br/>Route to: AP-Southeast
    Route53 -->> UserTokyo: DNS response (10ms)<br/>IP: 52.220.XXX.XXX (Singapore)
    UserTokyo ->> CDN_AP: GET /feed (20ms)
    CDN_AP ->> CDN_AP: Check edge cache (5ms)
    Note over CDN_AP: Cache MISS<br/>(personalized feed)
    CDN_AP ->> API_AP: Forward request (30ms)
    API_AP ->> API_AP: Validate JWT (5ms)
    API_AP ->> Feed_AP: Authenticated request (40ms)
    Feed_AP ->> Redis_AP: GET user:123:embedding (10ms)
    Note over Feed_AP, Redis_AP: Redis replica (AP region)<br/>Replicated from US-East<br/>Lag: <1 second
    Redis_AP -->> Feed_AP: User embedding (50ms)
    Feed_AP ->> ES_AP: Personalized query (10ms)
    Note over Feed_AP, ES_AP: Elasticsearch read replica<br/>Snapshot restored from S3<br/>Lag: up to 1 hour
    ES_AP -->> Feed_AP: Top 50 article IDs (60ms)
    Feed_AP ->> Redis_AP: MGET articles (10ms)
    Redis_AP -->> Feed_AP: Article content (70ms)
    Feed_AP ->> Feed_AP: Assemble response (10ms)
    Feed_AP -->> API_AP: JSON response (80ms)
    API_AP -->> CDN_AP: Cache response (TTL=5min)
    CDN_AP -->> UserTokyo: Feed displayed (80ms total)
    Note over UserTokyo, ES_AP: Latency: 80ms (AP user)<br/>Without multi-region: 300ms<br/><br/>Replication lag:<br/>- Kafka: 2-5s<br/>- Elasticsearch: up to 1h<br/>- Redis: <1s<br/><br/>Trade-off: Eventual consistency
```

---

## 11. Cache Invalidation Flow

**Flow:**

Shows the event-driven cache invalidation flow when an article is updated.

**Steps:**

1. **Article Update (0ms)** - Admin updates article content via CMS
2. **Update Service (50ms)** - Update article in Elasticsearch
3. **Kafka Event (55ms)** - Publish `article.updated` event to Kafka
4. **Cache Invalidation Service (60ms)** - Consume event
5. **Redis Invalidation (65ms)** - DELETE article cache key
6. **CDN Purge (200ms)** - Purge CDN cache via API
7. **Next Request (after 1 min)** - User fetches feed, cache miss, fresh data served

**Invalidation Strategies:**

1. **Event-based** - Invalidate on update (this flow)
2. **Time-based** - TTL expiration (most common)
3. **Version-based** - Cache key includes version number
4. **Lazy invalidation** - Stale data acceptable for non-critical content

**Performance:**

- Invalidation latency: <5 seconds (Redis), <30 seconds (CDN)
- Kafka event lag: <100ms

**Edge Cases:**

- Kafka event lost → Cache serves stale data until TTL expires (acceptable)
- CDN purge fails → Retry with exponential backoff
- Multiple updates in quick succession → Coalesce invalidations (debounce)

```mermaid
sequenceDiagram
    participant Admin
    participant UpdateService
    participant Elasticsearch
    participant Kafka
    participant InvalidationService
    participant Redis
    participant CDN
    participant User
    Admin ->> UpdateService: POST /articles/xyz-789 (0ms)<br/>Update: title, body
    UpdateService ->> Elasticsearch: POST /news/_doc/xyz-789 (50ms)
    Note over UpdateService, Elasticsearch: Update document<br/>title: "Updated Title"<br/>body: "Updated content..."
    Elasticsearch -->> UpdateService: 200 OK (50ms)<br/>Updated successfully
    UpdateService ->> Kafka: Publish event (5ms)
    Note over UpdateService, Kafka: Topic: article.updated<br/>Event: {<br/> article_id: xyz-789,<br/> type: update,<br/> timestamp: 1698765432<br/>}
    Kafka -->> UpdateService: Ack (offset: 98765) (55ms)
    UpdateService -->> Admin: 200 OK (55ms)<br/>Article updated
    Kafka ->> InvalidationService: Consume event (60ms)
    InvalidationService ->> InvalidationService: Extract article_id (1ms)
    Note over InvalidationService: article_id: xyz-789<br/>Invalidation keys:<br/>- article:xyz-789<br/>- feed:* (articles containing this)
    InvalidationService ->> Redis: DEL article:xyz-789 (5ms)
    Redis -->> InvalidationService: OK (65ms)
    Note over Redis: Cache entry deleted<br/>Next read: cache miss
    InvalidationService ->> CDN: POST /purge (150ms)
    Note over InvalidationService, CDN: CDN API: CloudFront<br/>Purge paths:<br/>- /articles/xyz-789<br/>- /feed/* (wildcard purge)
    CDN -->> InvalidationService: 200 OK (200ms)<br/>Purge in progress
    Note over CDN: Purge propagates to all PoPs<br/>Estimated time: 30 seconds
    InvalidationService ->> InvalidationService: Log invalidation (5ms)
    Note over InvalidationService: Metrics:<br/>cache_invalidations_total: +1<br/>invalidation_latency_ms: 200
    Note over Admin, User: After 1 minute (cache cleared)
    User ->> CDN: GET /articles/xyz-789
    CDN ->> CDN: Check cache
    Note over CDN: Cache MISS<br/>(purged)
    CDN ->> UpdateService: Fetch fresh data
    UpdateService ->> Elasticsearch: GET /news/_doc/xyz-789
    Elasticsearch -->> UpdateService: Fresh article
    UpdateService -->> CDN: Fresh content<br/>(Updated title, body)
    CDN ->> Redis: SET article:xyz-789<br/>TTL: 1 hour
    CDN -->> User: Serve fresh content
    Note over Admin, User: Invalidation latency: <5s (Redis), <30s (CDN)<br/>Cache miss on next request<br/>Fresh data served
```

---

## 12. Failure Recovery - Kafka Consumer Lag

**Flow:**

Shows the failure recovery process when Kafka consumers fall behind due to slow processing or downtime.

**Scenario:**

- **NLP Service** experiences high CPU load due to spike in articles (breaking news event)
- Consumer lag increases from 0 to 50,000 messages
- Alert triggered when lag > 10,000

**Steps:**

1. **Monitoring (0ms)** - Prometheus detects consumer lag > 10,000
2. **Alert (30s)** - PagerDuty alert sent to on-call engineer
3. **Diagnosis (2 min)** - Engineer checks metrics, identifies high CPU
4. **Scale Up (5 min)** - Auto-scaling increases NLP workers from 20 to 40
5. **Catch Up (30 min)** - Consumers process backlog at 2x rate
6. **Recovery (60 min)** - Lag returns to normal (<1,000 messages)

**Failure Modes:**

1. **Slow processing** - NLP service overwhelmed (this scenario)
2. **Consumer crash** - Worker dies, consumer group rebalances
3. **Network partition** - Consumer can't reach Kafka brokers
4. **Kafka broker down** - Leader election, temporary unavailability

**Recovery Strategies:**

1. **Auto-scaling** - Increase consumer instances
2. **Backpressure** - Slow down producers (not applicable for news ingestion)
3. **Dead letter queue** - Move problematic messages to DLQ
4. **Skip and catch up** - Process only recent messages (trade-off: data loss)

**Monitoring Metrics:**

- `kafka_consumer_lag` - Messages behind
- `kafka_consumer_lag_seconds` - Time behind
- `consumer_processing_rate` - Messages/sec
- `consumer_error_rate` - Failures/sec

```mermaid
sequenceDiagram
    participant Kafka
    participant NLPConsumers as NLP Consumers<br/>(20 instances)
    participant Prometheus
    participant PagerDuty
    participant Engineer
    participant AutoScaler
    participant NewConsumers as NLP Consumers<br/>(+20 new instances)
    Note over Kafka, NLPConsumers: Normal operation<br/>Consumer lag: 0-100 messages
    Kafka ->> NLPConsumers: Events (0ms)<br/>Topic: article.deduplicated<br/>Rate: 1,000 msgs/sec
    NLPConsumers ->> NLPConsumers: Process events (200ms each)
    Note over NLPConsumers: Processing rate: 100 msgs/sec<br/>(20 instances × 5 msgs/sec)
    Note over Kafka: Breaking news event!<br/>Article spike: 5,000 msgs/sec
    Kafka ->> NLPConsumers: High traffic (30s)<br/>Rate: 5,000 msgs/sec
    NLPConsumers ->> NLPConsumers: CPU overload!
    Note over NLPConsumers: Processing rate: 100 msgs/sec<br/>Ingestion rate: 5,000 msgs/sec<br/>Lag increases: 4,900 msgs/sec
    NLPConsumers ->> Prometheus: Expose metrics (1 min)
    Note over Prometheus: kafka_consumer_lag = 50,000<br/>Threshold: 10,000
    Prometheus ->> Prometheus: Evaluate alert (1 min)
    Note over Prometheus: Alert: ConsumerLagHigh<br/>Severity: WARNING<br/>Lag: 50,000 > 10,000
    Prometheus ->> PagerDuty: Trigger alert (1 min)
    PagerDuty ->> Engineer: Send alert (1 min 30s)<br/>SMS + Email + Phone call
    Engineer ->> Prometheus: Check dashboard (2 min)
    Note over Engineer: Diagnosis:<br/>- CPU: 95% (high)<br/>- Lag: 50,000 messages<br/>- Processing time: 500ms/msg (slow)<br/><br/>Root cause: GPU contention<br/>(BERT model inference)
    Engineer ->> AutoScaler: Scale up NLP workers (5 min)
    Note over Engineer: Increase instances: 20 → 40<br/>ECS: Update desired count
    AutoScaler ->> NewConsumers: Launch 20 new instances (5 min)
    NewConsumers ->> Kafka: Join consumer group
    Kafka ->> Kafka: Rebalance partitions (10s)
    Note over Kafka: Partition assignment:<br/>24 partitions / 40 instances<br/>Each instance: 0-1 partition
    Kafka ->> NLPConsumers: Resume consuming (5 min 10s)
    Kafka ->> NewConsumers: Resume consuming (5 min 10s)
    Note over NLPConsumers, NewConsumers: Processing rate: 200 msgs/sec<br/>(40 instances × 5 msgs/sec)
    NLPConsumers ->> Prometheus: Lag decreasing (30 min)
    NewConsumers ->> Prometheus: Lag decreasing (30 min)
    Note over Prometheus: kafka_consumer_lag = 20,000 (decreasing)<br/>Rate: -1,000 msgs/sec
    Note over Kafka, NewConsumers: After 30 minutes
    NLPConsumers ->> Prometheus: Lag normal (60 min)
    NewConsumers ->> Prometheus: Lag normal (60 min)
    Note over Prometheus: kafka_consumer_lag = 500<br/>Alert: RESOLVED
    Prometheus ->> PagerDuty: Resolve alert (60 min)
    PagerDuty ->> Engineer: Alert resolved
    Engineer ->> AutoScaler: Scale down (65 min)
    Note over Engineer: Reduce instances: 40 → 25<br/>(Keep buffer for next spike)
    Note over Kafka, NewConsumers: Recovery complete<br/>Lag: <1,000 messages<br/>Processing rate: 125 msgs/sec
```

