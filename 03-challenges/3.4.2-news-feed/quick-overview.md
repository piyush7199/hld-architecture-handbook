# Global News Feed - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A global news aggregation platform that ingests 100M articles/day from millions of sources (RSS feeds, APIs, web
scraping), deduplicates stories across different publishers (same event, different wording), personalizes feeds for 100M
daily active users based on reading history, provides <50ms search across billions of documents, and detects trending
topics in real-time.

**Real-World Examples:**

- **Google News:** 100B articles indexed, 1B+ users, <50ms latency
- **Flipboard:** 34M feeds, 145M users, personalized content curation
- **Apple News:** 125M users, aggregates from 2000+ publishers
- **SmartNews:** 20M DAU, ML-powered personalization

**The Core Challenge:**

Traditional databases cannot handle:

1. **Massive write throughput:** 100M articles/day = 1,157 writes/sec
2. **Full-text search:** Billions of documents with <50ms latency
3. **Near-duplicate detection:** "Apple releases iPhone 15" from 10K sources
4. **Real-time personalization:** User reads article → feed updates instantly

---

## Requirements & Scale

### Functional Requirements

**Article Management:**

1. **Ingestion:** Crawl RSS feeds, APIs, web scraping (100M articles/day)
2. **Deduplication:** Group articles about same story from different sources
3. **Categorization:** Auto-categorize (Technology, Sports, Politics, etc.)
4. **Ranking:** Quality scoring (source reputation, freshness, engagement)

**User Experience:**

1. **Personalized Feed:** Ranked by user interests and reading history
2. **Full-Text Search:** Keyword search across all articles (<50ms)
3. **Trending Topics:** Real-time detection of viral stories
4. **Multi-Language:** Support 50+ languages (translation, indexing)

### Non-Functional Requirements

| Requirement                | Target           | Rationale                                   |
|----------------------------|------------------|---------------------------------------------|
| **Feed Latency**           | <50ms (p99)      | User experience (fast page load)            |
| **Search Latency**         | <50ms (p99)      | Competitive with Google Search              |
| **Ingestion Throughput**   | 1,157 writes/sec | 100M articles/day                           |
| **Read QPS**               | 300,000 QPS      | 100M DAU × 3 feed loads/day                 |
| **Availability**           | 99.9%            | AP system (eventual consistency acceptable) |
| **Personalization Lag**    | <1 hour          | Real-time feature updates                   |
| **Deduplication Accuracy** | 95%              | Minimize duplicate stories                  |

### Scale Estimation

| Metric                   | Calculation                        | Result                                 |
|--------------------------|------------------------------------|----------------------------------------|
| **Articles Ingested**    | 100M articles/day                  | 1,157 writes/sec                       |
| **Storage (5 years)**    | 100M/day × 365 × 5 × 10 KB         | 180 TB (raw) + 360 TB (index) = 540 TB |
| **Read QPS**             | 100M DAU × 3 feeds/day ÷ 86400 sec | 3,472 QPS avg, 300K QPS peak           |
| **Deduplication Checks** | 1,157 writes/sec × 10K comparisons | 11.57M comparisons/sec                 |
| **NLP Processing**       | 1,157 articles/sec × 500ms NLP     | 578 parallel NLP workers               |
| **Trending Topics**      | 1,157 articles/sec × 100 topics    | 115K topic updates/sec                 |

**Latency Budget Breakdown:**

```
Target: 50ms feed load

User Request → API Gateway:     5ms  (10%)
Personalization Service:       15ms  (30%)  ← Biggest component
Elasticsearch Query:           20ms  (40%)
Article Content Fetch (CDN):    5ms  (10%)
Network + Rendering:            5ms  (10%)
Total:                         50ms
```

---

## Key Components

| Component           | Responsibility                        | Technology      | Scalability                   |
|---------------------|---------------------------------------|-----------------|-------------------------------|
| **RSS Crawlers**    | Fetch articles from RSS feeds         | Python (Scrapy) | Horizontal (10K sources)      |
| **Kafka**           | Buffer ingestion (1,157 articles/sec) | Apache Kafka    | Horizontal (100 partitions)   |
| **Bloom Filter**    | Fast duplicate URL detection          | Redis           | In-memory (1B URLs)           |
| **LSH Service**     | Near-duplicate detection              | MinHash + LSH   | Horizontal (parallel workers) |
| **NLP Processing**  | Keyword extraction, embeddings        | spaCy, BERT     | Horizontal (578 workers)      |
| **Elasticsearch**   | Full-text search index                | Elasticsearch   | Horizontal (100 nodes)        |
| **Personalization** | User profile + ranking                | DynamoDB + ML   | Horizontal                    |
| **CDN**             | Cache article content                 | CloudFront      | Global edge                   |

---

## Architecture Flow

**Data Flow Summary:**

1. **Ingestion:** Crawlers → Kafka (1,157 articles/sec)
2. **Deduplication:** Bloom Filter → LSH → Unique articles
3. **NLP Processing:** Keyword extraction → Embeddings → Categorization
4. **Indexing:** Elasticsearch (full-text search) + Redis (cache)
5. **Personalization:** User profile → Ranked query → Feed
6. **Serving:** CDN (static) + Redis (hot data) + Elasticsearch (search)

---

## Deduplication (Bloom Filters + LSH)

### Bloom Filter (Fast URL Check)

**Purpose:** Quickly check if URL already seen (O(1) lookup).

**How it works:**

```
1. Hash URL → 3 bit positions (k=3 hash functions)
2. Check if all 3 bits are set
   - If YES → URL seen (0.01% false positive rate)
   - If NO → URL new (100% accurate)
3. If new → Set all 3 bits, proceed to LSH
```

**Performance:**

- Lookup: <1μs (in-memory)
- Storage: 1B URLs × 10 bits = 1.25 GB (fits in RAM)

### LSH (Locality-Sensitive Hashing)

**Purpose:** Detect near-duplicates (same story, different wording).

**How it works:**

```
1. MinHash: Convert article text → 128-bit signature
2. LSH: Hash signature → 16 bands (4 bits each)
3. If 2 articles share ≥1 band → Likely duplicate
4. Jaccard similarity >0.85 → Group as same story
```

**Example:**

```
Article A: "Apple releases iPhone 15 with new camera"
Article B: "Apple announces iPhone 15 featuring improved camera"

MinHash similarity: 0.92 → Duplicate!
```

**Performance:**

- Processing: 75ms per article
- Accuracy: 95% (catches paraphrased duplicates)

---

## NLP Processing

### Keyword Extraction (TextRank + TF-IDF)

**TextRank Algorithm:**

```
1. Build word graph (co-occurrence within 2-word window)
2. Run PageRank on graph (importance = incoming edges)
3. Top 10 words = Keywords
```

**TF-IDF:**

```
TF-IDF(word) = TF(word) × IDF(word)

TF(word) = count(word) / total_words
IDF(word) = log(total_articles / articles_containing_word)

Why? Downweight common words ("the", "is", "of")
```

### Named Entity Recognition (NER)

**spaCy NER Model:**

```python
doc = nlp("Apple announces iPhone 15 in Cupertino")

entities = {
  "ORG": ["Apple"],
  "PRODUCT": ["iPhone 15"],
  "GPE": ["Cupertino"]
}
```

**Use Cases:**

- **Story clustering:** Articles mentioning "Elon Musk" grouped together
- **Personalization:** User interested in "Tesla" → Show articles with "Tesla" entity
- **Trending entities:** Track "iPhone 15" mentions (viral detection)

### Embeddings (BERT for Semantic Similarity)

**BERT (Bidirectional Encoder Representations from Transformers):**

```
1. Input: Article text (truncated to 512 tokens)
2. BERT model: bert-base-uncased (110M parameters)
3. Output: 768-dimensional embedding vector

Example:
  "Apple releases iPhone 15" → [0.23, -0.45, 0.67, ..., 0.12]
  "New iPhone announced by Apple" → [0.25, -0.43, 0.65, ..., 0.14]
  
  Cosine similarity: 0.92 (very similar!)
```

**Use Cases:**

- **Semantic deduplication:** Catch paraphrased duplicates (LSH misses)
- **Recommendation:** "Articles similar to this one" (cosine similarity)
- **Search:** Semantic search (query embedding vs article embeddings)

**Performance:**

- BERT inference: 200ms per article (GPU: NVIDIA T4)
- Parallelization: 500 GPUs → 1,157 articles/sec throughput

---

## Elasticsearch Indexing

### Time-Based Indices (Hot/Warm/Cold)

```
articles-2024-01-01  (Hot: <1 day old, NVMe SSD, 10 shards)
articles-2024-01-02  (Hot)
...
articles-2024-12-31  (Warm: 1-30 days old, HDD, 5 shards)
articles-2023-*      (Cold: >30 days old, S3, 1 shard)
```

**Why Time-Based?**

- **Efficient writes:** New articles always go to today's index
- **Fast deletes:** Drop entire index (old articles) without reindexing
- **Index Lifecycle Management (ILM):** Auto-migrate to cold storage

---

## Personalization and Ranking

### User Profile (DynamoDB)

```json
{
  "user_id": "67890",
  "interests_explicit": [
    "Technology",
    "Sports"
  ],
  "interests_implicit": {
    "Technology": 0.85,
    "Politics": 0.45,
    "Entertainment": 0.30
  },
  "reading_history": [
    {
      "article_id": "123456789",
      "timestamp": 1640000000,
      "dwell_time": 45,
      "engaged": true
    }
  ]
}
```

### Ranking Algorithm

**Score = 0.4 × Relevance + 0.3 × Freshness + 0.2 × Quality + 0.1 × Diversity**

1. **Relevance:** User interests match article category/keywords
2. **Freshness:** Exponential decay (articles <1 hour old get boost)
3. **Quality:** Source reputation + engagement metrics
4. **Diversity:** Max 2 articles from same source (avoid filter bubble)

---

## Trending Topics

### Real-Time Detection (Sliding Window)

**Algorithm:**

```
1. Extract keywords from incoming articles (NLP)
2. Count keyword mentions in last 1 hour (sliding window)
3. Compare to baseline (average mentions/hour for that keyword)
4. Trending score = (current_count / baseline_count)
5. If score > 5× → Trending!

Example:
  Keyword: "Earthquake"
  Baseline: 10 mentions/hour
  Current: 500 mentions/hour
  Score: 500 / 10 = 50× → TRENDING
```

**Implementation: Redis Sorted Sets**

```redis
Key: trending:keywords:{timestamp_hour}
Value: Sorted Set { "Earthquake": 500, "iPhone": 250, ... }
TTL: 24 hours
```

---

## Bottlenecks & Solutions

### Bottleneck 1: LSH Deduplication (75ms per article)

**Problem:** 1,157 articles/sec × 75ms = 86 parallel workers needed.

**Solution 1: Approximate LSH**

- Reduce LSH bands: 16 bands → 8 bands
- Trade-off: 95% accuracy → 90% accuracy
- Speedup: 75ms → 40ms (1.8× faster)

**Solution 2: Pre-filter by Source Clusters**

```
High-quality sources (Reuters, AP) → Skip LSH (assume unique)
Medium-quality sources → Full LSH
Low-quality sources → Aggressive LSH
```

**Result:** 80% of articles skip LSH → 86 workers → 20 workers.

### Bottleneck 2: Elasticsearch Write Throughput

**Problem:** 1,157 writes/sec × 100 indices = 115,700 index ops/sec.

**Solution: Bulk Indexing with Write Buffer**

```
Buffer 1000 articles → Single bulk request
Reduces network roundtrips by 1000×
```

---

## Common Anti-Patterns to Avoid

### 1. Storing Full Article Text in PostgreSQL

❌ **Bad:**

```sql
CREATE TABLE articles (
  article_id BIGINT,
  title TEXT,
  content TEXT,  -- ❌ 10 KB per article!
  ...
);

-- 100M articles × 10 KB = 1 TB in PostgreSQL
-- Full-text search: SELECT * WHERE content LIKE '%keyword%'
-- Latency: 10+ seconds (full table scan!)
```

✅ **Good:** Use Elasticsearch for full-text search

```
PostgreSQL: Metadata only (title, URL, published_at)
Elasticsearch: Full content + inverted index
Redis: Hot data (last 24 hours)
```

### 2. Synchronous NLP Processing

❌ **Bad:**

```python
def ingest_article(article):
  save_to_db(article)
  keywords = extract_keywords(article)  # 50ms
  embedding = bert_embed(article)        # 200ms
  category = classify(article)           # 100ms
  index_to_elasticsearch(article)
  # Total: 350ms per article!
```

✅ **Good:** Async pipeline with Kafka

```python
def ingest_article(article):
  publish_to_kafka(article)  # 1ms (async)
  return  # Immediately return

# Separate workers consume Kafka and process NLP
```

### 3. No Deduplication

❌ **Bad:**

```
10K sources publish "Apple releases iPhone 15"
All 10K articles indexed → User sees same story 10K times
```

✅ **Good:** LSH deduplication + Story clustering

```
1. Detect duplicates via LSH
2. Create story_id (group duplicates)
3. Show only 1 article per story_id
4. Optionally: Show "10K sources covering this story"
```

---

## Trade-offs Summary

| What We Gain                   | What We Sacrifice                                   |
|--------------------------------|-----------------------------------------------------|
| ✅ Fast search (<50ms)          | ❌ Complex architecture (Elasticsearch + PostgreSQL) |
| ✅ Accurate deduplication (95%) | ❌ High compute cost (LSH + NLP)                     |
| ✅ Personalized feeds           | ❌ Privacy concerns (user tracking)                  |
| ✅ Real-time trending           | ❌ High storage cost (540 TB)                        |
| ✅ Multi-language support       | ❌ Complex NLP pipeline (50+ languages)              |

---

## Real-World Examples

### Google News

- **Scale:** 100B articles indexed, 1B+ users
- **Latency:** <50ms feed load
- **Deduplication:** Proprietary algorithm (likely LSH-based)
- **Personalization:** Google Search history integration

### Flipboard

- **Scale:** 34M feeds, 145M users
- **Architecture:** Microservices on AWS
- **Personalization:** ML-powered content curation
- **Unique:** Magazine-style UI (visual curation)

### Apple News

- **Scale:** 125M users, 2000+ publishers
- **Architecture:** iCloud infrastructure
- **Monetization:** Apple News+ subscription ($9.99/month)

---

## Key Takeaways

1. **Bloom Filter** enables fast URL duplicate detection (O(1) lookup, 0.01% false positives)
2. **LSH (MinHash)** detects near-duplicates with 95% accuracy (paraphrased articles)
3. **BERT embeddings** enable semantic search and similarity matching
4. **Time-based Elasticsearch indices** enable efficient writes and deletes (hot/warm/cold)
5. **Personalization** combines explicit interests + implicit behavior (reading history)
6. **Trending topics** detected via sliding window (Redis Sorted Sets)
7. **Async NLP processing** prevents blocking ingestion (Kafka pipeline)
8. **CDN caching** reduces latency for article content (CloudFront)
9. **Deduplication** improves UX (1 article per story, not 10K duplicates)
10. **Multi-language support** requires translation + language-specific indexing

---

## Recommended Stack

- **Ingestion:** Python (Scrapy) + Apache Kafka
- **Deduplication:** Redis (Bloom Filter) + MinHash (LSH)
- **NLP:** spaCy (NER) + BERT (embeddings) + TextRank (keywords)
- **Search:** Elasticsearch (full-text) + Redis (cache)
- **Storage:** PostgreSQL (metadata) + DynamoDB (user profiles)
- **Serving:** CDN (CloudFront) + API Gateway

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, ingestion pipeline, NLP processing)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (article ingestion, deduplication,
  personalization)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (LSH, NLP, ranking)

