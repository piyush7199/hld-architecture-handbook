# 3.4.2 Design a Global News Feed (Google News/Aggregator)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, ingestion pipeline, NLP processing flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Article ingestion, deduplication, personalization flows
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## Problem Statement

Design a global news aggregation and personalization platform that ingests 100 million articles per day from millions of
sources, deduplicates stories across publishers, provides personalized feeds for 100 million daily active users, and
enables <50ms full-text search across billions of documents.

**Real-World Context:**

- **Google News:** 100B articles indexed, 1B+ users globally, <50ms latency
- **Flipboard:** 34M feeds aggregated, 145M users, ML-powered curation
- **Apple News:** 125M users, 2000+ publishers integrated
- **SmartNews:** 20M DAU, personalized with machine learning

**Core Technical Challenges:**

1. **Massive ingestion:** 100M articles/day = 1,157 writes/sec continuous throughput
2. **Near-duplicate detection:** Same story from 10K sources with different wording
3. **Full-text search:** Billions of documents with <50ms query latency
4. **Real-time personalization:** User reads article → feed updates instantly
5. **NLP at scale:** Keyword extraction, entity recognition, embeddings for 1,157 articles/sec

---

## Requirements and Scale Estimation

### Functional Requirements

**Content Management:**

- Ingest articles from RSS feeds, APIs, web scraping
- Detect and merge near-duplicate stories (LSH similarity)
- Auto-categorize content (Technology, Sports, Politics, etc.)
- Extract keywords, entities, and generate embeddings

**User Experience:**

- Personalized feed based on reading history and interests
- Full-text search across all articles (<50ms)
- Trending topics in real-time (1-minute updates)
- Multi-language support (50+ languages)

### Non-Functional Requirements

| Requirement                | Target           | Industry Benchmark                  |
|----------------------------|------------------|-------------------------------------|
| **Feed Latency (p99)**     | <50ms            | Google News: ~30ms                  |
| **Search Latency (p99)**   | <50ms            | Elasticsearch: 20-50ms              |
| **Ingestion Throughput**   | 1,157 writes/sec | 100M articles/day                   |
| **Read QPS**               | 300K QPS peak    | 100M DAU × 3 loads/day              |
| **Availability**           | 99.9%            | AP system (eventual consistency OK) |
| **Deduplication Accuracy** | 95%              | Industry standard: 90-98%           |
| **Personalization Lag**    | <1 hour          | Batch: 24h, Real-time: <1h          |

### Scale Estimation

| Metric                  | Calculation                                  | Result                                 |
|-------------------------|----------------------------------------------|----------------------------------------|
| **Articles/Day**        | 100M articles                                | 1,157 writes/sec                       |
| **Storage (5 years)**   | 100M × 365 × 5 × 10 KB                       | 180 TB raw + 360 TB index = **540 TB** |
| **Read QPS (peak)**     | 100M DAU × 3 feeds/day ÷ 28800 sec (8h peak) | **300,000 QPS**                        |
| **Deduplication**       | 1,157 articles/sec × 10K comparisons         | 11.57M comparisons/sec                 |
| **NLP Workers**         | 1,157 articles/sec × 500ms processing        | 578 parallel workers                   |
| **Elasticsearch Nodes** | 540 TB ÷ 5 TB/node                           | 100 nodes (3× replication)             |

**Latency Budget:**

```
Target: 50ms feed load

API Gateway:            5ms  (10%)
Personalization:       15ms  (30%)  ← Critical path
Elasticsearch Query:   20ms  (40%)  ← Largest component
CDN (Article Fetch):    5ms  (10%)
Network Overhead:       5ms  (10%)
Total:                 50ms
```

---

## High-Level Architecture

### System Components

```
┌─────────────────────────────────────────────────────────┐
│          INGESTION LAYER (1,157 writes/sec)             │
│                                                          │
│  RSS Crawlers → API Connectors → Web Scrapers          │
│  (10K sources)   (1K publishers)   (Newspaper3k)        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼ Raw Articles (JSON)
          ┌──────────────────────┐
          │   KAFKA INGESTION    │
          │   100 partitions     │
          │   7-day retention    │
          └────────┬─────────────┘
                   │
         ┌─────────┴─────────┐
         ▼                   ▼
    ┌────────┐         ┌─────────────┐
    │ BLOOM  │         │ POSTGRESQL  │
    │ FILTER │         │ (Metadata)  │
    │ 1B URLs│         │ Sharded 100×│
    └───┬────┘         └─────────────┘
        │
        ▼ Not Seen
    ┌──────────────────────────┐
    │  DEDUPLICATION SERVICE   │
    │  - LSH (MinHash)         │
    │  - Jaccard similarity    │
    │  - 95% accuracy          │
    └──────────┬───────────────┘
               │
               ▼ Unique Articles
    ┌──────────────────────────┐
    │  KAFKA PROCESSING TOPIC  │
    └──────────┬───────────────┘
               │
               ▼
    ┌──────────────────────────┐
    │    NLP PROCESSING        │
    │  - spaCy (keywords)      │
    │  - BERT (embeddings)     │
    │  - TextRank (summary)    │
    │  - Category ML model     │
    └──────────┬───────────────┘
               │
        ┌──────┴──────┐
        ▼             ▼
   ┌─────────┐   ┌─────────┐
   │ELASTIC  │   │ REDIS   │
   │ SEARCH  │   │ CACHE   │
   │ 180 TB  │   │ Hot 24h │
   └─────────┘   └─────────┘

┌─────────────────────────────────────────────────────────┐
│           SERVING LAYER (300K QPS)                      │
│                                                          │
│  User Profile Store (DynamoDB)                          │
│         ↓                                                │
│  Personalization Service (ML ranking)                   │
│         ↓                                                │
│  Feed Serving (CDN + Redis + Elasticsearch)            │
└─────────────────────────────────────────────────────────┘
```

**Data Flow:**

1. **Crawlers** → Kafka (1,157 articles/sec)
2. **Bloom Filter** → Deduplicate URLs
3. **LSH** → Detect near-duplicates (content similarity)
4. **NLP** → Extract keywords, embeddings, category
5. **Elasticsearch** → Index for full-text search
6. **Personalization** → Rank articles by user interests
7. **Serve** → CDN + Redis cache (hot data)

---

## Data Model

### Article Schema

**PostgreSQL (Metadata):**

```sql
CREATE TABLE articles (
    article_id BIGINT PRIMARY KEY,
    url TEXT UNIQUE NOT NULL,
    source_id INT,
    title TEXT,
    published_at TIMESTAMP,
    category VARCHAR(50),
    quality_score FLOAT,
    INDEX idx_published (published_at DESC)
);
```

**Elasticsearch (Full Content + Search):**

```json
{
  "mappings": {
    "properties": {
      "article_id": {
        "type": "long"
      },
      "title": {
        "type": "text",
        "analyzer": "english"
      },
      "content": {
        "type": "text"
      },
      "keywords": {
        "type": "keyword"
      },
      "category": {
        "type": "keyword"
      },
      "published_at": {
        "type": "date"
      },
      "quality_score": {
        "type": "float"
      },
      "embedding_vector": {
        "type": "dense_vector",
        "dims": 768
      }
    }
  }
}
```

**Why Elasticsearch?**

- Inverted index for O(log N) full-text search
- Aggregations for trending topics (bucketing)
- BM25 scoring for relevance ranking
- 100-node cluster handles 540 TB index

### User Profile Schema

**DynamoDB:**

```json
{
  "user_id": "67890",
  "interests_explicit": [
    "Technology",
    "Sports"
  ],
  "interests_implicit": {
    "Technology": 0.85,
    "Politics": 0.45
  },
  "reading_history": [
    {
      "article_id": "123",
      "timestamp": 1640000000,
      "dwell_time": 45
    }
  ]
}
```

**Why DynamoDB?**

- Single-digit ms reads (low latency)
- Auto-scaling (100M users)
- Schema-less (flexible user profiles)

---

## Ingestion and Deduplication

### RSS Crawling Strategy

**Crawl Frequency:**

- High-frequency: Every 5 min (Reuters, AP, Bloomberg)
- Medium-frequency: Every 30 min (local newspapers)
- Low-frequency: Every 2 hours (blogs, niche sites)

**Politeness:**

- Max 1 req/sec per domain (avoid overload)
- Respect robots.txt
- Identify as legitimate crawler

*Implementation: [pseudocode.md::crawl_rss_feed()](pseudocode.md)*

### Stage 1: Bloom Filter (URL Exact Match)

**Configuration:**

- Capacity: 1 billion URLs (1 year of data)
- False positive rate: 0.01% (1 in 10,000)
- Memory: ~1.2 GB
- Hash functions: 7 (optimal)

**Flow:**

```
1. Article arrives → Check URL in Bloom Filter
2. If Yes → Probably seen (skip or update metadata)
3. If No → Definitely new (add to Bloom Filter, continue to LSH)
```

*Implementation: [pseudocode.md::bloom_filter_check()](pseudocode.md)*

### Stage 2: LSH (Content Similarity)

**Problem:** Same story, different wording:

- Reuters: "Apple announces iPhone 15 with new features"
- TechCrunch: "iPhone 15 unveiled: What's new"
- The Verge: "Apple's iPhone 15: Everything you need to know"

**Solution: MinHash + LSH**

1. **MinHash Signature:**
    - Tokenize article into 3-word shingles
    - Generate 128-hash signature (compact representation)

2. **LSH Bucketing:**
    - Split signature into 16 bands × 8 rows
    - Hash each band → Bucket ID
    - Articles in same bucket → Likely similar

3. **Jaccard Similarity:**
    - Compute exact similarity for candidate pairs
    - If > 0.85 → Duplicate (merge into same story)

**Performance:**

- MinHash: 50ms per article
- LSH lookup: 5ms
- Jaccard: 10ms per pair (avg 3 pairs)
- **Total: 75ms deduplication overhead**

*Implementation: [pseudocode.md::minhash_signature()](pseudocode.md), [pseudocode.md::lsh_find_similar()](pseudocode.md)*

---

## NLP Processing

### Keyword Extraction (TextRank)

**Algorithm:**

```
1. Build word co-occurrence graph (2-word window)
2. Run PageRank on graph (importance = incoming edges)
3. Top 10 words = Keywords

Example:
  Article: "Apple announces iPhone 15 with new camera features"
  Keywords: ["iPhone", "Apple", "camera", "announces", "features"]
```

*Implementation: [pseudocode.md::extract_keywords()](pseudocode.md)*

### Named Entity Recognition (NER)

**spaCy NER:**

```python
import spacy
nlp = spacy.load("en_core_web_lg")
doc = nlp("Apple announces iPhone 15 in Cupertino")

Entities:
  ORG: ["Apple"]
  PRODUCT: ["iPhone 15"]
  GPE: ["Cupertino"]
```

**Use Cases:**

- Story clustering (articles mentioning "Elon Musk")
- Personalization (user interested in "Tesla")
- Trending entities ("iPhone 15" mentions spike)

### Embeddings (BERT)

**BERT (768-dimensional vectors):**

```
Input: "Apple releases iPhone 15" (truncated to 512 tokens)
Model: bert-base-uncased (110M parameters)
Output: [0.23, -0.45, 0.67, ..., 0.12]

Use Cases:
  - Semantic deduplication (cosine similarity)
  - Recommendation ("similar articles")
  - Semantic search (query embedding vs article embeddings)

Performance:
  - 200ms per article (GPU: NVIDIA T4)
  - 500 GPUs → 1,157 articles/sec throughput
```

---

## Elasticsearch Indexing and Search

### Time-Based Indices (Hot/Warm/Cold)

```
articles-2024-01-01  (Hot: <1 day, NVMe SSD, 10 shards)
articles-2024-01-02  (Hot)
articles-2024-12-31  (Warm: 1-30 days, HDD, 5 shards)
articles-2023-*      (Cold: >30 days, S3, 1 shard)
```

**Why Time-Based?**

- Efficient writes (today's index only)
- Fast deletes (drop entire index)
- Index Lifecycle Management (ILM: auto-migrate)

### Personalized Feed Query

**Elasticsearch Query DSL:**

```json
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {
              "terms": {
                "category": [
                  "Technology",
                  "Sports"
                ]
              }
            },
            {
              "range": {
                "published_at": {
                  "gte": "now-7d"
                }
              }
            }
          ],
          "should": [
            {
              "match": {
                "keywords": {
                  "query": "AI ML",
                  "boost": 2
                }
              }
            }
          ]
        }
      },
      "functions": [
        {
          "exp": {
            "published_at": {
              "origin": "now",
              "scale": "1d",
              "decay": 0.5
            }
          }
        },
        {
          "field_value_factor": {
            "field": "quality_score",
            "factor": 1.2
          }
        }
      ]
    }
  },
  "size": 100
}
```

**Query Breakdown:**

1. Filter: Categories [Technology, Sports], published last 7 days
2. Boost: Keywords match user interests (2× weight)
3. Recency decay: Exponential (half-life = 1 day)
4. Quality boost: Higher quality sources ranked higher
5. Return: Top 100 articles

**Latency:** 20ms (80% queries cached)

---

## Personalization and Ranking

### User Interest Model

**Explicit Interests (User-Selected):**

```json
{
  "user_id": "67890",
  "selected_topics": [
    "Technology",
    "Sports",
    "Politics"
  ]
}
```

**Implicit Interests (ML Model):**

```
Input:
  - Reading history (last 30 days)
  - Dwell time (time spent on article)
  - Engagement (clicks, shares, bookmarks)

Model: Collaborative Filtering (Matrix Factorization)
  - User embeddings: 100-dim vector
  - Article embeddings: 100-dim vector
  - Prediction: dot_product(user_embedding, article_embedding)

Output:
  - Interest scores: {"Technology": 0.85, "Politics": 0.45}

Training:
  - Daily batch job (Spark)
  - 1B records (100M users × 10 articles/user)
  - 1000 CPU cores × 2 hours
  - Cost: $100/day
```

### Ranking Formula

```
final_score = (relevance × 0.4) + 
              (recency × 0.3) + 
              (quality × 0.2) + 
              (engagement × 0.1)

Where:
  relevance = Elasticsearch BM25 score
  recency = exp(-age_hours / 24)
  quality = source.reputation_score
  engagement = clicks / impressions
```

*Implementation: [pseudocode.md::calculate_article_score()](pseudocode.md)*

### Real-Time Feature Store

**Problem:** User reads article → Feed should update instantly.

**Solution: Redis Feature Store**

```redis
Key: user:67890:recent_categories
Value: {"Technology": 3, "Sports": 1}  (last 1 hour)
TTL: 1 hour
```

**Update Flow:**

```
1. User reads article (category = "Technology")
2. Event → Kafka (user-activity topic)
3. Consumer → Increment Redis counter
4. Next feed load → Boost "Technology" category
```

**Latency:** <100ms (Redis read + Elasticsearch boost)

---

## Trending Topics

### Real-Time Detection (Sliding Window)

**Algorithm:**

```
1. Extract keywords from articles (NLP)
2. Count mentions in last 1 hour (sliding window)
3. Compare to baseline (avg mentions/hour)
4. Trending score = current / baseline
5. If score > 5× → Trending!

Example:
  Keyword: "Earthquake"
  Baseline: 10 mentions/hour
  Current: 500 mentions/hour
  Score: 500 / 10 = 50× → TRENDING
```

**Implementation: Redis Sorted Sets**

```redis
Key: trending:keywords:2024-01-01-12
Value: Sorted Set {"Earthquake": 500, "iPhone": 250}
TTL: 24 hours

Query Top 10:
ZREVRANGE trending:keywords:2024-01-01-12 0 9 WITHSCORES
```

*Implementation: [pseudocode.md::update_trending_topics()](pseudocode.md)*

### Elasticsearch Aggregation

**Query:**

```json
{
  "aggs": {
    "trending_keywords": {
      "terms": {
        "field": "keywords",
        "size": 100,
        "order": {
          "_count": "desc"
        }
      },
      "aggs": {
        "recent": {
          "filter": {
            "range": {
              "published_at": {
                "gte": "now-1h"
              }
            }
          }
        }
      }
    }
  }
}
```

**Result:** Top 100 keywords in last 1 hour, sorted by frequency.

---

## Bottlenecks and Scaling

### Bottleneck 1: LSH Deduplication (75ms per article)

**Problem:** 1,157 articles/sec × 75ms = 86 parallel workers.

**Solution 1: Approximate LSH**

- Reduce bands: 16 → 8 bands
- Trade-off: 95% accuracy → 90% accuracy
- Speedup: 75ms → 40ms (1.8× faster)

**Solution 2: Source Clustering**

- High-quality sources (Reuters, AP) → Skip LSH (assume unique)
- Medium-quality → Full LSH
- Low-quality → Aggressive LSH

**Result:** 80% skip LSH → 86 workers → 20 workers.

### Bottleneck 2: Elasticsearch Write Throughput

**Problem:** 1,157 writes/sec × 100 indices = 115,700 index ops/sec.

**Solution: Bulk Indexing with Buffer**

```
1. Articles → Redis Stream (buffer)
2. Bulk indexer (Flink) → Batch 1000 articles every 5 sec
3. Elasticsearch Bulk API → 200 articles/request
4. Result: 1,157 writes/sec → 6 bulk requests/sec (200× reduction)
```

**Trade-off:** 5-second indexing delay (acceptable).

### Bottleneck 3: Personalization Lag

**Problem:** User reads "AI" articles today, but model trained yesterday.

**Solution: Hybrid Model (Batch + Real-Time)**

```
Batch Model (Collaborative Filtering):
  - Daily training on 30-day history
  - Long-term interests

Real-Time Features (Redis):
  - Last 1 hour activity
  - Boost recent categories

Combined: 0.7 × batch_score + 0.3 × realtime_score
```

**Result:** Lag: 24 hours → <1 hour.

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: Storing Full Content in PostgreSQL

**Problem:**

```sql
CREATE TABLE articles (
  article_id BIGINT,
  content TEXT  -- ❌ 10 KB per article!
);
-- 100M articles × 10 KB = 1 TB in PostgreSQL
-- Full-text search: LIKE '%keyword%' → 10+ sec!
```

**✅ Best Practice:** Elasticsearch for full-text search.

```
PostgreSQL: Metadata only
Elasticsearch: Full content + inverted index
Redis: Hot data (24 hours)
```

---

### ❌ Anti-Pattern 2: Synchronous NLP Processing

**Problem:**

```python
def ingest(article):
  save_to_db(article)
  keywords = extract_keywords(article)  # 50ms
  embedding = bert_embed(article)       # 200ms
  category = classify(article)          # 100ms
  # Total: 350ms per article!
  # 1,157 articles/sec × 350ms = 405 workers!
```

**✅ Best Practice:** Async pipeline with Kafka.

```python
def ingest(article):
  publish_to_kafka(article)  # 1ms (async)
  return
# Separate workers consume and process
```

---

### ❌ Anti-Pattern 3: No Deduplication

**Problem:** 10K sources publish "Apple releases iPhone 15" → User sees same story 10K times.

**✅ Best Practice:** LSH deduplication + story clustering.

```
1. Detect duplicates via LSH
2. Create story_id (group duplicates)
3. Show 1 article per story_id
4. "10K sources covering this story"
```

---

## Alternative Approaches

### Alternative 1: Algolia (Managed Search)

| Factor            | Elasticsearch (Chosen) | Algolia                |
|-------------------|------------------------|------------------------|
| **Cost**          | ✅ $50K/month           | ❌ $500K/month (180 TB) |
| **Customization** | ✅ Full control         | ❌ Limited              |
| **Scale**         | ✅ 180 TB               | ⚠️ Expensive           |

**When to Use Algolia:** Smaller index (<10 TB), managed service preferred.

---

### Alternative 2: Apache Solr

| Factor        | Elasticsearch (Chosen) | Solr          |
|---------------|------------------------|---------------|
| **Community** | ✅ Larger               | ⚠️ Smaller    |
| **Ecosystem** | ✅ Kibana, Beats        | ⚠️ Limited    |
| **Cloud**     | ✅ Elastic Cloud        | ❌ No official |

**When to Use Solr:** Already invested in Solr, specific features needed.

---

## Monitoring and Observability

| Metric                          | Target           | Alert           |
|---------------------------------|------------------|-----------------|
| **Ingestion Rate**              | 1,157 writes/sec | <500 writes/sec |
| **Deduplication Accuracy**      | 95%              | <90%            |
| **NLP Lag**                     | <5 min           | >30 min         |
| **Elasticsearch Latency (p99)** | <50ms            | >100ms          |
| **Feed Load Latency (p99)**     | <50ms            | >100ms          |

**Tracing Example:**

```
Article ID: 123456789
  0ms:    Crawled from RSS
  10ms:   Published to Kafka
  100ms:  Bloom Filter (not seen)
  175ms:  LSH deduplication (unique)
  375ms:  NLP processing
  400ms:  Elasticsearch indexed
  500ms:  Visible in feed

Total: 500ms ingestion-to-visibility
```

---

## Cost Analysis

### Hardware (Monthly)

| Component         | Spec                          | Quantity | Cost            |
|-------------------|-------------------------------|----------|-----------------|
| **Elasticsearch** | r5.4xlarge (128 GB, 1 TB SSD) | 100      | $100K           |
| **Kafka**         | m5.2xlarge (32 GB)            | 20       | $10K            |
| **NLP GPU**       | p3.2xlarge (V100)             | 50       | $75K            |
| **Redis**         | r5.large (16 GB)              | 50       | $8K             |
| **PostgreSQL**    | db.r5.4xlarge                 | 10       | $15K            |
| **Total**         |                               |          | **$208K/month** |

### Operational (Annual)

| Item                             | Cost           |
|----------------------------------|----------------|
| Infrastructure                   | $2.5M/year     |
| Bandwidth (100 TB egress/month)  | $1M/year       |
| Personnel (20 engineers @ $150K) | $3M/year       |
| **Total**                        | **$6.5M/year** |

**Revenue:**

- Advertising: $0.01/impression
- 100M DAU × 10 impressions/day = $10M/day = **$3.65B/year**

**Profit:** $3.65B - $6.5M = **$3.64B/year** (99.8% margin)

---

## Real-World Examples

### Google News

- Scale: 100B articles, 1B+ users
- Latency: <50ms feed load
- Deduplication: Proprietary (LSH-based)
- Personalization: Search history integration

### Flipboard

- Scale: 34M feeds, 145M users
- Architecture: Microservices (AWS)
- Personalization: ML content curation
- UI: Magazine-style visual design

### Apple News

- Scale: 125M users, 2000+ publishers
- Architecture: iCloud infrastructure
- Monetization: Apple News+ ($9.99/month)

---

## Interview Discussion Points

### Q1: How detect breaking news in real-time?

**Answer:**

**Spike Detection:**

```
1. Count keyword mentions (last 5 min)
2. Compare to baseline (avg/hour)
3. If spike > 10× → Breaking news!

Example:
  "Earthquake": 10 mentions/hour baseline
  Spike: 500 mentions in 5 min → 6000/hour projected
  Ratio: 6000 / 10 = 600× → ALERT
```

**Implementation:** Redis Sorted Sets, Kafka Streams.

---

### Q2: Handle malicious source (1M fake articles)?

**Answer:**

**Reputation System:**

```
1. Track source metrics:
   - Article count (spike detection)
   - Duplicate rate (LSH matches)
   - Engagement (low = low quality)

2. If anomaly:
   - Quarantine source
   - Block new articles
   - Remove indexed articles

3. Reputation score:
   - Historical accuracy
   - User reports
   - Fact-checking (Snopes, PolitiFact)
```

**Rate Limiting:** Max 1000 articles/day per source.

---

## Trade-offs Summary

| Gain                        | Sacrifice                     |
|-----------------------------|-------------------------------|
| ✅ Fast search (<50ms)       | ❌ High storage (540 TB)       |
| ✅ Accurate dedup (95%)      | ⚠️ NLP cost ($75K/month GPUs) |
| ✅ Personalization (ML)      | ❌ 24-hour lag (batch)         |
| ✅ Real-time trending        | ⚠️ Eventual consistency       |
| ✅ Scalability (1B articles) | ❌ Operational complexity      |

**Best For:**

- Global news aggregators (Google News, Flipboard)
- Content curation (Pocket, Instapaper)
- Enterprise monitoring (Meltwater, Factiva)

**NOT For:**

- Small sites (<1M articles)
- Real-time chat (different latency needs)

---

## References

### Papers

- [LSH for Near-Duplicate Detection](https://www.cs.princeton.edu/courses/archive/spring13/cos598C/broder97resemblance.pdf) -
  Andrei Broder
- [BERT](https://arxiv.org/abs/1810.04805) - Devlin et al.
- [TextRank](https://web.eecs.umich.edu/~mihalcea/papers/mihalcea.emnlp04.pdf) - Mihalcea & Tarau

### Related Chapters

- [2.5.4 Bloom Filters](../../02-components/2.5-algorithms/2.5.4-bloom-filters.md)
- [2.1.13 Elasticsearch](../../02-components/2.1-databases/2.1.13-elasticsearch-deep-dive.md)
- [2.3.2 Kafka](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)

### Tools

- [spaCy](https://spacy.io/) - NLP library
- [Hugging Face](https://huggingface.co/) - BERT models
- [Newspaper3k](https://newspaper.readthedocs.io/) - Article extraction
- [datasketch](https://github.com/ekzhu/datasketch) - MinHash, LSH

