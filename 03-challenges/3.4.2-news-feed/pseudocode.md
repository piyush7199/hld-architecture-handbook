# Global News Feed - Pseudocode Implementations

This document contains detailed algorithm implementations for the Global News Feed system. The main challenge document
references these functions.

---

## Table of Contents

1. [Article Ingestion](#1-article-ingestion)
2. [Deduplication (Bloom Filter + LSH)](#2-deduplication-bloom-filter--lsh)
3. [NLP Processing](#3-nlp-processing)
4. [Write Buffer (Redis Streams)](#4-write-buffer-redis-streams)
5. [Personalization (ML Model)](#5-personalization-ml-model)
6. [Real-Time Feature Store](#6-real-time-feature-store)
7. [Trending Topics Detection](#7-trending-topics-detection)
8. [Feed Serving](#8-feed-serving)
9. [Cache Operations](#9-cache-operations)
10. [Multi-Region Replication](#10-multi-region-replication)

---

## 1. Article Ingestion

### ingest_rss_feed()

**Purpose:** Poll an RSS feed and publish articles to Kafka.

**Parameters:**

- source_id (string): Unique identifier for the RSS source (e.g., "cnn-rss")
- feed_url (string): URL of the RSS feed

**Returns:**

- count (int): Number of articles ingested

**Algorithm:**

```
function ingest_rss_feed(source_id, feed_url):
  // Check rate limit
  if not rate_limiter.allow(source_id):
    return 0  // Rate limit exceeded
  
  try:
    // Fetch RSS feed with timeout
    response = http_get(feed_url, timeout=10s)
    
    if response.status_code != 200:
      log_error("RSS fetch failed", source_id, response.status_code)
      return 0
    
    // Parse RSS XML
    feed = parse_rss_xml(response.body)
    
    count = 0
    for item in feed.items:
      article = {
        article_id: generate_uuid(),
        source_id: source_id,
        source_name: feed.title,
        title: item.title,
        body: item.description or item.content,
        url: item.link,
        published_at: item.pubDate or now(),
        author: item.author,
        image_url: extract_image(item),
        raw_html: item.content
      }
      
      // Validate required fields
      if not validate_article(article):
        log_error("Invalid article", article.url)
        continue
      
      // Detect language
      article.language = detect_language(article.title + " " + article.body)
      
      if article.language not in ["en", "es", "fr", "de", "zh"]:
        // Skip unsupported languages
        continue
      
      // Publish to Kafka
      kafka_producer.send(
        topic="article.raw",
        key=source_id,  // Partition by source
        value=article
      )
      
      count++
    
    // Update source metadata
    update_source_last_poll(source_id, now())
    
    return count
    
  catch Exception as e:
    log_error("RSS ingestion failed", source_id, e)
    return 0
```

**Time Complexity:** O(n) where n = number of items in RSS feed

**Example Usage:**

```
count = ingest_rss_feed("cnn-rss", "https://rss.cnn.com/rss/cnn_topstories.rss")
// Returns: 25 (25 articles ingested)
```

---

## 2. Deduplication (Bloom Filter + LSH)

### check_duplicate()

**Purpose:** Check if an article is a duplicate using two-stage detection (Bloom Filter + LSH).

**Parameters:**

- article (object): Article to check {url, title, body}

**Returns:**

- result (object): {is_duplicate: boolean, story_id: string or null}

**Algorithm:**

```
function check_duplicate(article):
  // Stage 1: Bloom Filter (Exact URL Match)
  url_hash = sha256(article.url)
  
  if bloom_filter.contains(url_hash):
    // Exact URL match (likely duplicate)
    story_id = db.get_story_by_url(article.url)
    
    if story_id:
      return {is_duplicate: true, story_id: story_id}
    else:
      // False positive (0.1% chance)
      // Continue to Stage 2
  
  // Stage 2: LSH (Content Similarity)
  lsh_signature = compute_lsh_signature(article.title + " " + article.body)
  
  similar_articles = lsh_index.query(lsh_signature, threshold=0.85)
  
  if len(similar_articles) > 0:
    // Near-duplicate found
    best_match = similar_articles[0]  // Highest similarity
    story_id = db.get_story_by_article(best_match.article_id)
    
    return {is_duplicate: true, story_id: story_id}
  
  // New article (not a duplicate)
  return {is_duplicate: false, story_id: null}
```

**Time Complexity:**

- Bloom Filter: O(1)
- LSH query: O(log n) where n = number of articles

**Example Usage:**

```
article = {
  url: "https://cnn.com/article-123",
  title: "Apple announces iPhone 16",
  body: "Apple today unveiled the new iPhone 16..."
}

result = check_duplicate(article)
// Returns: {is_duplicate: false, story_id: null}
```

---

### compute_lsh_signature()

**Purpose:** Compute LSH (MinHash) signature for content similarity matching.

**Parameters:**

- text (string): Article content (title + body)

**Returns:**

- signature (array): MinHash signature (128 hash values)

**Algorithm:**

```
function compute_lsh_signature(text):
  // Preprocessing
  text = text.toLowerCase()
  text = remove_html_tags(text)
  text = remove_stopwords(text)  // "the", "a", "is", etc.
  
  // Generate shingles (5-grams)
  shingles = []
  words = text.split(" ")
  
  for i in range(len(words) - 4):
    shingle = words[i:i+5].join(" ")  // 5 consecutive words
    shingles.append(shingle)
  
  // Compute MinHash signature
  signature = []
  num_hashes = 128
  
  for i in range(num_hashes):
    hash_func = create_hash_function(seed=i)
    min_hash = INFINITY
    
    for shingle in shingles:
      hash_value = hash_func(shingle)
      min_hash = min(min_hash, hash_value)
    
    signature.append(min_hash)
  
  return signature
```

**Time Complexity:** O(s × h) where s = number of shingles, h = number of hash functions (128)

**Example Usage:**

```
text = "Apple announces iPhone 16 with improved camera and battery life"
signature = compute_lsh_signature(text)
// Returns: [1234567, 9876543, 5555555, ...] (128 values)
```

---

### add_to_lsh_index()

**Purpose:** Add article's LSH signature to the index for future similarity queries.

**Parameters:**

- article_id (string): Unique article identifier
- signature (array): MinHash signature (128 values)

**Returns:**

- success (boolean): True if added successfully

**Algorithm:**

```
function add_to_lsh_index(article_id, signature):
  // LSH Banding Technique
  num_bands = 16
  rows_per_band = 128 / 16 = 8
  
  for band_idx in range(num_bands):
    start_row = band_idx * rows_per_band
    end_row = start_row + rows_per_band
    
    // Extract band (8 hash values)
    band = signature[start_row:end_row]
    
    // Hash the band to get a bucket ID
    bucket_id = hash(band)
    
    // Store in Redis Sorted Set
    // Score = current timestamp (for TTL cleanup)
    redis.zadd(
      key="lsh:band:" + band_idx + ":" + bucket_id,
      score=now(),
      member=article_id
    )
  
  return true
```

**Time Complexity:** O(b) where b = number of bands (16)

**Example Usage:**

```
signature = compute_lsh_signature(article.body)
success = add_to_lsh_index("article-123", signature)
// Returns: true
```

---

## 3. NLP Processing

### extract_keywords()

**Purpose:** Extract top keywords from article text using TF-IDF.

**Parameters:**

- text (string): Article content (title + body)

**Returns:**

- keywords (array): Top 20 keywords with scores [{keyword: string, score: float}]

**Algorithm:**

```
function extract_keywords(text):
  // Preprocessing
  text = text.toLowerCase()
  text = remove_html_tags(text)
  words = tokenize(text)
  words = remove_stopwords(words)
  
  // Compute Term Frequency (TF)
  tf = {}
  for word in words:
    tf[word] = tf.get(word, 0) + 1
  
  // Normalize TF
  max_freq = max(tf.values())
  for word in tf:
    tf[word] = tf[word] / max_freq
  
  // Compute Inverse Document Frequency (IDF)
  // Use pre-computed IDF values from corpus
  idf = load_idf_values()  // From Redis cache
  
  // Compute TF-IDF
  tfidf = {}
  for word in tf:
    tfidf[word] = tf[word] * idf.get(word, 1.0)
  
  // Sort by TF-IDF score and take top 20
  keywords = []
  sorted_words = sort_by_value(tfidf, descending=true)
  
  for i in range(min(20, len(sorted_words))):
    word = sorted_words[i]
    keywords.append({
      keyword: word,
      score: tfidf[word]
    })
  
  return keywords
```

**Time Complexity:** O(n log n) where n = number of unique words

**Example Usage:**

```
text = "Apple announces new iPhone 16 with AI-powered camera..."
keywords = extract_keywords(text)
// Returns: [
//   {keyword: "iphone", score: 0.95},
//   {keyword: "apple", score: 0.87},
//   {keyword: "ai", score: 0.76},
//   ...
// ]
```

---

### analyze_sentiment()

**Purpose:** Analyze sentiment of article using BERT model.

**Parameters:**

- text (string): Article content (first 512 tokens)

**Returns:**

- sentiment (object): {label: string, score: float}

**Algorithm:**

```
function analyze_sentiment(text):
  // Truncate to first 512 tokens (BERT limit)
  tokens = tokenize(text)
  if len(tokens) > 512:
    tokens = tokens[0:512]
  
  // Load pre-trained BERT sentiment model
  model = load_model("bert-sentiment-analyzer")
  
  // Tokenize for BERT
  input_ids = bert_tokenizer(tokens)
  
  // Run inference (GPU accelerated)
  output = model.predict(input_ids)
  
  // Output: [negative_prob, neutral_prob, positive_prob]
  negative_prob = output[0]
  neutral_prob = output[1]
  positive_prob = output[2]
  
  // Determine label
  max_prob = max(negative_prob, neutral_prob, positive_prob)
  
  if max_prob == positive_prob:
    label = "positive"
    score = positive_prob
  else if max_prob == negative_prob:
    label = "negative"
    score = -negative_prob  // Negative score
  else:
    label = "neutral"
    score = 0.0
  
  return {label: label, score: score}
```

**Time Complexity:** O(1) - Fixed inference time (~50ms on GPU)

**Example Usage:**

```
text = "This is an amazing breakthrough in AI technology!"
sentiment = analyze_sentiment(text)
// Returns: {label: "positive", score: 0.89}
```

---

### classify_topic()

**Purpose:** Classify article into topics using BERT multi-label classifier.

**Parameters:**

- text (string): Article content (title + first 512 tokens)

**Returns:**

- topics (array): Topics with confidence scores [{topic: string, confidence: float}]

**Algorithm:**

```
function classify_topic(text):
  // Predefined topics
  all_topics = ["Tech", "Business", "Sports", "Politics", "Entertainment", 
                "Science", "Health", "World", "US", "Opinion"]
  
  // Truncate to 512 tokens
  tokens = tokenize(text)[0:512]
  
  // Load pre-trained BERT multi-label classifier
  model = load_model("bert-topic-classifier")
  
  // Tokenize for BERT
  input_ids = bert_tokenizer(tokens)
  
  // Run inference (GPU accelerated)
  logits = model.predict(input_ids)
  
  // Apply sigmoid (multi-label, not softmax)
  probabilities = sigmoid(logits)
  
  // Filter topics with confidence > 0.5
  topics = []
  for i in range(len(all_topics)):
    if probabilities[i] > 0.5:
      topics.append({
        topic: all_topics[i],
        confidence: probabilities[i]
      })
  
  // Sort by confidence
  topics = sort_by_confidence(topics, descending=true)
  
  return topics
```

**Time Complexity:** O(1) - Fixed inference time (~50ms on GPU)

**Example Usage:**

```
text = "Apple announces new iPhone 16 with breakthrough AI chip..."
topics = classify_topic(text)
// Returns: [
//   {topic: "Tech", confidence: 0.95},
//   {topic: "Business", confidence: 0.78}
// ]
```

---

## 4. Write Buffer (Redis Streams)

### write_to_buffer()

**Purpose:** Write article to Redis Streams buffer before Elasticsearch indexing.

**Parameters:**

- article (object): Enriched article with NLP metadata

**Returns:**

- message_id (string): Redis Streams message ID

**Algorithm:**

```
function write_to_buffer(article):
  stream_key = "es:write:buffer"
  
  // Check if buffer is full (max 100k messages)
  buffer_length = redis.xlen(stream_key)
  
  if buffer_length >= 100000:
    throw Error("Buffer full, reject write")  // 503 Service Unavailable
  
  // Serialize article to JSON
  article_json = json.serialize(article)
  
  // Add to Redis Streams
  message_id = redis.xadd(
    key=stream_key,
    maxlen="~100000",  // Capped stream (approximate limit)
    fields={
      article: article_json,
      timestamp: now()
    }
  )
  
  return message_id
```

**Time Complexity:** O(1)

**Example Usage:**

```
article = {
  article_id: "xyz-789",
  title: "Apple announces iPhone 16",
  topics: ["Tech", "Business"],
  keywords: ["iphone", "apple", "ai"],
  ...
}

message_id = write_to_buffer(article)
// Returns: "1698765432000-0"
```

---

### bulk_index_from_buffer()

**Purpose:** Read batch of articles from Redis Streams and bulk index to Elasticsearch.

**Parameters:**

- consumer_group (string): Redis Streams consumer group name
- consumer_name (string): Unique consumer identifier

**Returns:**

- count (int): Number of articles indexed

**Algorithm:**

```
function bulk_index_from_buffer(consumer_group, consumer_name):
  stream_key = "es:write:buffer"
  batch_size = 1000
  block_timeout = 5000  // 5 seconds
  
  // Read batch from Redis Streams
  messages = redis.xreadgroup(
    group=consumer_group,
    consumer=consumer_name,
    streams={stream_key: ">"},  // ">" means unprocessed messages
    count=batch_size,
    block=block_timeout
  )
  
  if len(messages) == 0:
    return 0  // No messages to process
  
  // Transform to Elasticsearch bulk format
  bulk_body = []
  message_ids = []
  
  for message in messages:
    article = json.deserialize(message.fields.article)
    message_ids.append(message.id)
    
    // Bulk index action
    bulk_body.append({
      index: {
        _index: "news-" + current_month(),  // Time-based index
        _id: article.article_id
      }
    })
    
    // Document
    bulk_body.append({
      article_id: article.article_id,
      title: article.title,
      body: article.body,
      topics: article.topics,
      keywords: article.keywords,
      entities: article.entities,
      sentiment: article.sentiment,
      quality_score: article.quality_score,
      published_at: article.published_at,
      url: article.url
    })
  
  try:
    // Bulk write to Elasticsearch
    response = elasticsearch.bulk(body=bulk_body)
    
    if response.errors:
      // Partial failure: some docs failed to index
      failed_ids = extract_failed_ids(response)
      
      // Re-add failed docs to retry queue
      for article_id in failed_ids:
        redis.xadd(
          key="es:write:buffer:retry",
          fields={article_id: article_id}
        )
      
      // Acknowledge successful messages only
      successful_ids = [id for id in message_ids if id not in failed_ids]
      redis.xack(stream_key, consumer_group, *successful_ids)
      
      return len(successful_ids)
    else:
      // All docs indexed successfully
      redis.xack(stream_key, consumer_group, *message_ids)
      
      return len(messages)
  
  catch ElasticsearchException as e:
    // Elasticsearch unavailable: DO NOT acknowledge
    // Messages will be reprocessed by another consumer
    log_error("Elasticsearch bulk index failed", e)
    return 0
```

**Time Complexity:** O(n) where n = batch size (1000)

**Example Usage:**

```
count = bulk_index_from_buffer("es:bulk:writers", "worker-1")
// Returns: 1000 (1000 articles indexed)
```

---

## 5. Personalization (ML Model)

### train_collaborative_filtering()

**Purpose:** Train collaborative filtering model using ALS (Alternating Least Squares).

**Parameters:**

- user_article_interactions (dataframe): User-article interaction matrix

**Returns:**

- model (object): Trained ALS model

**Algorithm:**

```
function train_collaborative_filtering(user_article_interactions):
  // Spark MLlib ALS configuration
  als = ALS(
    rank=128,  // Embedding dimensions
    maxIter=10,  // Number of iterations
    regParam=0.1,  // Regularization parameter
    implicitPrefs=true,  // Implicit feedback (clicks, not ratings)
    alpha=0.01,  // Confidence weight
    userCol="user_id",
    itemCol="article_id",
    ratingCol="interaction_score"  // Implicit score (clicks, dwell time)
  )
  
  // Train model on Spark cluster
  model = als.fit(user_article_interactions)
  
  // Extract embeddings
  user_embeddings = model.userFactors  // 100M users × 128 dimensions
  article_embeddings = model.itemFactors  // 100M articles × 128 dimensions
  
  return {
    model: model,
    user_embeddings: user_embeddings,
    article_embeddings: article_embeddings
  }
```

**Time Complexity:** O(k × n × m) where k = iterations, n = users, m = articles

**Example Usage:**

```
interactions = load_interactions_from_kafka("user.actions", last_24_hours)
model = train_collaborative_filtering(interactions)
// Returns: Trained ALS model with embeddings
```

---

### compute_personalized_score()

**Purpose:** Compute personalized score for an article given user preferences.

**Parameters:**

- user_embedding (array): User embedding vector (128 dimensions)
- article_embedding (array): Article embedding vector (128 dimensions)
- user_recent_topics (object): Recent topics clicked by user {topic: count}
- article_topics (array): Article topics

**Returns:**

- score (float): Personalized score (0-1)

**Algorithm:**

```
function compute_personalized_score(user_embedding, article_embedding, 
                                   user_recent_topics, article_topics):
  // Collaborative filtering score (dot product)
  collab_score = dot_product(user_embedding, article_embedding)
  
  // Normalize to 0-1 range (embeddings are L2-normalized)
  collab_score = (collab_score + 1) / 2  // Range: [-1, 1] → [0, 1]
  
  // Content-based score (topic match)
  topic_match_score = 0.0
  total_clicks = sum(user_recent_topics.values())
  
  for topic in article_topics:
    if topic in user_recent_topics:
      // Weight by click frequency
      topic_match_score += user_recent_topics[topic] / total_clicks
  
  // Normalize topic match score
  if len(article_topics) > 0:
    topic_match_score = topic_match_score / len(article_topics)
  
  // Hybrid score (70% collaborative, 30% content-based)
  final_score = 0.7 * collab_score + 0.3 * topic_match_score
  
  return final_score
```

**Time Complexity:** O(d + t) where d = embedding dimensions (128), t = number of topics

**Example Usage:**

```
user_embedding = [0.1, 0.3, -0.2, ...] // 128 dimensions
article_embedding = [0.2, 0.4, -0.1, ...] // 128 dimensions
user_recent_topics = {Tech: 10, Business: 5}
article_topics = ["Tech", "Business"]

score = compute_personalized_score(user_embedding, article_embedding, 
                                   user_recent_topics, article_topics)
// Returns: 0.85
```

---

## 6. Real-Time Feature Store

### update_realtime_features()

**Purpose:** Update user's real-time features when they perform an action (click, share).

**Parameters:**

- user_id (string): User identifier
- action (object): User action {type: string, article_id: string, topic: string}

**Returns:**

- success (boolean): True if updated successfully

**Algorithm:**

```
function update_realtime_features(user_id, action):
  // Extract action details
  action_type = action.type  // "click", "share", "like"
  article_id = action.article_id
  topic = action.topic
  
  // Update recent topics (Sorted Set)
  recent_topics_key = "user:" + user_id + ":recent_topics"
  
  // Increment topic score (weighted by action type)
  weight = {
    "click": 1.0,
    "share": 3.0,  // Shares are stronger signals
    "like": 2.0
  }
  
  redis.zincrby(
    key=recent_topics_key,
    increment=weight[action_type],
    member=topic
  )
  
  // Set TTL (7 days)
  redis.expire(recent_topics_key, 7 * 24 * 3600)
  
  // Update recent reads (List)
  recent_reads_key = "user:" + user_id + ":recent_reads"
  
  // Add to front of list (most recent first)
  redis.lpush(recent_reads_key, article_id)
  
  // Trim to last 20 articles
  redis.ltrim(recent_reads_key, 0, 19)
  
  // Set TTL (7 days)
  redis.expire(recent_reads_key, 7 * 24 * 3600)
  
  // Update recency score (exponential decay)
  // More recent actions have higher weight
  recency_key = "user:" + user_id + ":recency_score"
  current_time = now()
  
  redis.hset(
    key=recency_key,
    field=article_id,
    value=current_time
  )
  
  redis.expire(recency_key, 7 * 24 * 3600)
  
  return true
```

**Time Complexity:** O(log n) where n = number of topics in Sorted Set

**Example Usage:**

```
action = {
  type: "click",
  article_id: "xyz-789",
  topic: "Tech"
}

success = update_realtime_features("user-123", action)
// Returns: true

// Redis state after update:
// user:123:recent_topics → {Tech: 11, Business: 5} (Tech incremented)
// user:123:recent_reads → ["xyz-789", "abc-456", ...] (xyz-789 added to front)
```

---

## 7. Trending Topics Detection

### detect_trending_topics()

**Purpose:** Detect trending topics using windowed aggregation and velocity tracking.

**Parameters:**

- time_window (int): Time window in seconds (default: 3600 for 1 hour)

**Returns:**

- trending_topics (array): Trending topics with scores [{topic: string, trend_score: float}]

**Algorithm:**

```
function detect_trending_topics(time_window=3600):
  current_time = now()
  window_start = current_time - time_window
  
  // Count events per topic in current window
  current_counts = {}
  
  events = kafka.consume(
    topic="user.actions",
    time_range=[window_start, current_time]
  )
  
  for event in events:
    topic = event.article_topic
    current_counts[topic] = current_counts.get(topic, 0) + 1
  
  // Load baseline (24-hour average) from database
  baseline_counts = db.query(
    "SELECT topic, AVG(count) as avg_count " +
    "FROM topic_stats " +
    "WHERE timestamp > ? " +
    "GROUP BY topic",
    params=[current_time - 24 * 3600]
  )
  
  // Compute trend score
  trending_topics = []
  
  for topic in current_counts:
    current = current_counts[topic]
    baseline = baseline_counts.get(topic, 1)  // Avoid division by zero
    
    // Trend score = percentage increase
    trend_score = (current - baseline) / baseline
    
    // Filter: only topics with >200% increase
    if trend_score > 2.0:
      trending_topics.append({
        topic: topic,
        trend_score: trend_score,
        current_count: current,
        baseline_count: baseline
      })
  
  // Sort by trend score (descending)
  trending_topics = sort_by_trend_score(trending_topics, descending=true)
  
  // Store in Redis (Sorted Set)
  redis_key = "trending:topics:global"
  
  for item in trending_topics:
    redis.zadd(
      key=redis_key,
      score=item.trend_score,
      member=item.topic
    )
  
  // Set TTL (2 hours)
  redis.expire(redis_key, 2 * 3600)
  
  return trending_topics
```

**Time Complexity:** O(n log n) where n = number of unique topics

**Example Usage:**

```
trending = detect_trending_topics(time_window=3600)
// Returns: [
//   {topic: "Earthquake Tokyo", trend_score: 99.0, current: 10000, baseline: 100},
//   {topic: "iPhone 16", trend_score: 5.5, current: 6500, baseline: 1000},
//   ...
// ]
```

---

## 8. Feed Serving

### get_personalized_feed()

**Purpose:** Generate personalized news feed for a user.

**Parameters:**

- user_id (string): User identifier
- page_size (int): Number of articles to return (default: 50)

**Returns:**

- feed (array): Personalized articles [{article_id, title, snippet, score}]

**Algorithm:**

```
function get_personalized_feed(user_id, page_size=50):
  // 1. Fetch user embeddings (batch ML model)
  user_embedding = redis.get("user:" + user_id + ":embedding")
  
  if not user_embedding:
    // Cold start: use default embedding (average of all users)
    user_embedding = get_default_user_embedding()
  
  // 2. Fetch real-time features
  recent_topics = redis.zrevrange(
    key="user:" + user_id + ":recent_topics",
    start=0,
    end=9,  // Top 10 topics
    withscores=true
  )
  
  recent_reads = redis.lrange(
    key="user:" + user_id + ":recent_reads",
    start=0,
    end=19  // Last 20 articles
  )
  
  // 3. Fetch trending topics
  trending_topics = redis.zrevrange(
    key="trending:topics:global",
    start=0,
    end=9,  // Top 10 trending
    withscores=true
  )
  
  // 4. Build Elasticsearch query (function_score)
  query = {
    query: {
      function_score: {
        query: {
          bool: {
            must: [
              // Recent articles (last 24 hours)
              {range: {published_at: {gte: "now-24h"}}}
            ],
            filter: [
              // User's interested topics
              {terms: {topics: recent_topics.keys()}}
            ],
            must_not: [
              // Exclude recently read articles
              {ids: {values: recent_reads}}
            ]
          }
        },
        functions: [
          // 1. Quality score boost
          {
            field_value_factor: {
              field: "quality_score",
              factor: 1.2,
              modifier: "sqrt"
            }
          },
          // 2. Recency decay
          {
            exp: {
              published_at: {
                scale: "12h",  // Half-life: 12 hours
                decay: 0.5
              }
            }
          },
          // 3. Personalization (user · article embeddings)
          {
            script_score: {
              script: {
                source: "cosineSimilarity(params.user_embedding, 'article_embedding') + 1.0",
                params: {
                  user_embedding: user_embedding
                }
              },
              weight: 2.0  // Higher weight for personalization
            }
          },
          // 4. Trending topics boost
          {
            script_score: {
              script: {
                source: """
                  double boost = 1.0;
                  for (topic in doc['topics']) {
                    if (params.trending.containsKey(topic.toString())) {
                      boost += params.trending[topic.toString()] * 0.5;
                    }
                  }
                  return boost;
                """,
                params: {
                  trending: trending_topics
                }
              }
            }
          }
        ],
        score_mode: "multiply",  // Multiply all function scores
        boost_mode: "multiply"   // Multiply with query score
      }
    },
    size: page_size,
    _source: ["article_id", "title", "snippet", "image_url", "source", "published_at"]
  }
  
  // 5. Execute Elasticsearch query
  response = elasticsearch.search(index="news-*", body=query)
  
  // 6. Enrich articles (fetch full content from cache)
  article_ids = [hit._id for hit in response.hits.hits]
  
  cached_articles = redis.mget(
    ["article:" + id for id in article_ids]
  )
  
  feed = []
  for i in range(len(article_ids)):
    article = cached_articles[i]
    
    if not article:
      // Cache miss: fetch from Elasticsearch
      article = elasticsearch.get(index="news-*", id=article_ids[i])
    
    feed.append({
      article_id: article.article_id,
      title: article.title,
      snippet: article.body[0:200] + "...",  // First 200 chars
      image_url: article.image_url,
      source: article.source_name,
      published_at: article.published_at,
      score: response.hits.hits[i]._score
    })
  
  return feed
```

**Time Complexity:** O(log n) where n = number of articles in Elasticsearch

**Example Usage:**

```
feed = get_personalized_feed("user-123", page_size=50)
// Returns: [
//   {article_id: "xyz-789", title: "Apple announces iPhone 16", score: 0.95, ...},
//   {article_id: "abc-456", title: "AI breakthrough", score: 0.87, ...},
//   ...
// ]
```

---

## 9. Cache Operations

### cache_article()

**Purpose:** Cache article content in Redis for fast retrieval.

**Parameters:**

- article (object): Article to cache
- ttl (int): Time-to-live in seconds (default: 3600 for 1 hour)

**Returns:**

- success (boolean): True if cached successfully

**Algorithm:**

```
function cache_article(article, ttl=3600):
  cache_key = "article:" + article.article_id
  
  // Serialize article to JSON
  article_json = json.serialize(article)
  
  // Set in Redis with TTL
  redis.setex(
    key=cache_key,
    value=article_json,
    ttl=ttl
  )
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
article = {article_id: "xyz-789", title: "Apple announces iPhone 16", ...}
cache_article(article, ttl=3600)
// Article cached for 1 hour
```

---

### invalidate_cache()

**Purpose:** Invalidate article cache when content is updated.

**Parameters:**

- article_id (string): Article identifier

**Returns:**

- success (boolean): True if invalidated successfully

**Algorithm:**

```
function invalidate_cache(article_id):
  cache_key = "article:" + article_id
  
  // Delete from Redis
  redis.del(cache_key)
  
  // Purge from CDN (CloudFront API)
  cdn.purge([
    "/articles/" + article_id,
    "/feed/*"  // Wildcard: all feeds containing this article
  ])
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
invalidate_cache("xyz-789")
// Article cache cleared, CDN purged
```

---

## 10. Multi-Region Replication

### replicate_to_regions()

**Purpose:** Replicate data to secondary regions (EU, AP) for low-latency global access.

**Parameters:**

- data_type (string): Type of data to replicate ("kafka", "redis", "elasticsearch")

**Returns:**

- success (boolean): True if replication initiated successfully

**Algorithm:**

```
function replicate_to_regions(data_type):
  if data_type == "kafka":
    // Kafka cross-region replication (MirrorMaker 2)
    mirrormaker_config = {
      source_cluster: "us-east-kafka",
      target_clusters: ["eu-west-kafka", "ap-southeast-kafka"],
      topics: ["user.actions"],  // Only replicate user actions
      replication_factor: 3
    }
    
    mirrormaker.start(mirrormaker_config)
    
  else if data_type == "redis":
    // Redis active-passive replication
    for region in ["eu-west", "ap-southeast"]:
      // Subscribe to Redis Streams in primary region
      stream_key = "replication:stream"
      
      while true:
        messages = redis_primary.xread(
          streams={stream_key: "$"},  // "$" means latest
          block=1000,  // 1 second timeout
          count=1000
        )
        
        for message in messages:
          // Replicate to secondary region
          redis_secondary[region].set(
            key=message.fields.key,
            value=message.fields.value,
            ttl=message.fields.ttl
          )
  
  else if data_type == "elasticsearch":
    // Elasticsearch snapshot & restore
    for region in ["eu-west", "ap-southeast"]:
      // 1. Create snapshot in primary region
      snapshot_name = "snapshot-" + current_time()
      
      elasticsearch_primary.create_snapshot(
        repository="s3-backup",
        snapshot=snapshot_name,
        indices="news-*"
      )
      
      // 2. Wait for snapshot to complete
      wait_for_snapshot(snapshot_name)
      
      // 3. Restore snapshot in secondary region
      elasticsearch_secondary[region].restore_snapshot(
        repository="s3-backup",
        snapshot=snapshot_name,
        indices="news-*"
      )
  
  return true
```

**Time Complexity:** Varies by data type

- Kafka: O(1) - Continuous replication
- Redis: O(n) where n = number of keys to replicate
- Elasticsearch: O(s) where s = snapshot size

**Example Usage:**

```
replicate_to_regions("kafka")
// Kafka MirrorMaker 2 started, replicating user.actions topic to EU/AP

replicate_to_regions("elasticsearch")
// Elasticsearch snapshot created and restored in EU/AP regions
```

---

## Summary

This pseudocode document provides detailed algorithm implementations for all major components of the Global News Feed
system. Key algorithms include:

1. **Ingestion:** RSS feed polling and article publishing
2. **Deduplication:** Two-stage detection (Bloom Filter + LSH)
3. **NLP:** Keyword extraction, sentiment analysis, topic classification
4. **Write Buffering:** Redis Streams for Elasticsearch bulk indexing
5. **Personalization:** Collaborative filtering + content-based hybrid model
6. **Real-Time Features:** User action tracking and feature store updates
7. **Trending Detection:** Windowed aggregation and velocity tracking
8. **Feed Serving:** Personalized ranking with multiple scoring functions
9. **Caching:** Article caching and cache invalidation
10. **Replication:** Multi-region data replication strategies

All functions are optimized for the scale requirements: 100M articles/day, 300k read QPS, <50ms feed latency.

