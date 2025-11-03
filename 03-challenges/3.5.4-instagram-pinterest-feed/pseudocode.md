# Instagram/Pinterest Feed - Pseudocode Implementations

This document contains detailed algorithm implementations for the Instagram/Pinterest Feed system. The main challenge
document references these functions.

---

## Table of Contents

1. [Media Upload Operations](#1-media-upload-operations)
2. [Feed Generation](#2-feed-generation)
3. [Fanout Strategies](#3-fanout-strategies)
4. [Engagement Operations](#4-engagement-operations)
5. [Image Processing](#5-image-processing)
6. [Recommendation Engine](#6-recommendation-engine)
7. [Cache Operations](#7-cache-operations)
8. [Visual Search](#8-visual-search)
9. [Hot Key Mitigation](#9-hot-key-mitigation)
10. [Circuit Breaker Pattern](#10-circuit-breaker-pattern)

---

## 1. Media Upload Operations

### generate_presigned_url()

**Purpose:** Generates a pre-signed S3 URL for direct client-side upload, bypassing application servers.

**Parameters:**

- `user_id` (int): User uploading the media
- `filename` (string): Original filename
- `file_size` (int): File size in bytes
- `content_type` (string): MIME type (e.g., "image/jpeg", "video/mp4")

**Returns:**

- `presigned_url` (string): S3 pre-signed URL valid for 15 minutes
- `upload_id` (string): Unique upload identifier
- `object_key` (string): S3 object key

**Algorithm:**

```
function generate_presigned_url(user_id, filename, file_size, content_type):
  // Step 1: Validate inputs
  if file_size > MAX_FILE_SIZE:
    throw ValidationError("File too large")
  
  if content_type not in ALLOWED_TYPES:
    throw ValidationError("Invalid content type")
  
  // Step 2: Check user quota
  user_quota = get_user_quota(user_id)
  if user_quota.remaining_uploads < 1:
    throw QuotaExceededError("Daily upload limit reached")
  
  // Step 3: Generate unique object key
  upload_id = generate_uuid()
  timestamp = current_timestamp()
  object_key = f"uploads/{user_id}/{timestamp}/{upload_id}.{get_extension(filename)}"
  
  // Step 4: Generate pre-signed URL
  presigned_url = s3_client.generate_presigned_url(
    method='PUT',
    bucket='instagram-media',
    key=object_key,
    expiration=900,  // 15 minutes
    params={
      'ContentType': content_type,
      'ContentLength': file_size
    }
  )
  
  // Step 5: Store upload metadata
  redis.SETEX(f"upload:{upload_id}", 900, JSON.stringify({
    user_id: user_id,
    object_key: object_key,
    file_size: file_size,
    status: 'pending'
  }))
  
  // Step 6: Decrement user quota
  redis.DECR(f"user_quota:{user_id}")
  
  return {
    presigned_url: presigned_url,
    upload_id: upload_id,
    object_key: object_key
  }
```

**Time Complexity:** O(1)

**Example Usage:**

```
result = generate_presigned_url(
  user_id=12345,
  filename="sunset.jpg",
  file_size=10485760,  // 10 MB
  content_type="image/jpeg"
)

// Client uploads directly to result.presigned_url
```

---

### process_media_job()

**Purpose:** Background worker processing for uploaded media (resize, transcode, extract features).

**Parameters:**

- `upload_id` (string): Upload identifier
- `object_key` (string): S3 object key

**Returns:**

- `post_id` (int): Created post identifier
- `media_urls` (dict): URLs for all generated versions

**Algorithm:**

```
function process_media_job(upload_id, object_key):
  // Step 1: Fetch upload metadata
  metadata = redis.GET(f"upload:{upload_id}")
  if metadata is None:
    throw Error("Upload metadata not found")
  
  user_id = metadata.user_id
  
  // Step 2: Download original from S3
  original_data = s3_client.download(object_key)
  
  // Step 3: Determine media type
  media_type = detect_media_type(original_data)
  
  if media_type == 'image':
    media_urls = process_image(original_data, object_key)
  elif media_type == 'video':
    media_urls = process_video(original_data, object_key)
  else:
    throw Error("Unsupported media type")
  
  // Step 4: Extract visual features (for recommendations)
  visual_features = extract_visual_features(original_data)
  
  // Step 5: Save post metadata to database
  post_id = postgres.INSERT(
    table='posts',
    values={
      user_id: user_id,
      media_urls: JSON.stringify(media_urls),
      media_type: media_type,
      created_at: NOW()
    }
  )
  
  // Step 6: Store visual embeddings in vector DB
  vector_db.insert(
    post_id=post_id,
    embedding=visual_features
  )
  
  // Step 7: Update upload status
  redis.SET(f"upload:{upload_id}", JSON.stringify({
    ...metadata,
    status: 'completed',
    post_id: post_id
  }))
  
  // Step 8: Trigger fanout
  kafka.publish("post.created", {
    post_id: post_id,
    user_id: user_id
  })
  
  return {
    post_id: post_id,
    media_urls: media_urls
  }
```

**Time Complexity:** O(n × m) where n = image dimensions, m = number of versions

**Example Usage:**

```
result = process_media_job(
  upload_id="abc-123-def",
  object_key="uploads/12345/1699123456/abc.jpg"
)
// Returns: {post_id: 789, media_urls: {...}}
```

---

## 2. Feed Generation

### generate_feed()

**Purpose:** Generates a personalized feed by combining follower posts, celebrity posts, and recommendations.

**Parameters:**

- `user_id` (int): User requesting feed
- `limit` (int): Number of posts to return (default: 50)
- `offset` (int): Pagination offset (default: 0)

**Returns:**

- `posts` (list): List of hydrated post objects

**Algorithm:**

```
function generate_feed(user_id, limit=50, offset=0):
  // Step 1: Parallel fetch from multiple sources
  results = parallel_execute([
    fetch_follower_posts(user_id, limit),
    fetch_celebrity_posts(user_id, limit),
    fetch_recommended_posts(user_id, limit),
    fetch_ads(user_id, limit)
  ])
  
  follower_posts = results[0]
  celebrity_posts = results[1]
  recommended_posts = results[2]
  ads = results[3]
  
  // Step 2: Merge with configurable ratio (60% follower, 30% reco, 10% ads)
  merged = stitch_feed(
    follower_posts,
    celebrity_posts,
    recommended_posts,
    ads,
    ratios=[0.6, 0.1, 0.3, 0.1]  // follower, celebrity, reco, ads
  )
  
  // Step 3: Deduplication (remove posts user has seen)
  seen_posts = get_seen_posts(user_id)
  deduplicated = []
  for post_id in merged:
    if not bloom_filter_check(seen_posts, post_id):
      deduplicated.append(post_id)
  
  // Step 4: Apply final ranking model
  ranked = ranking_model.score(
    post_ids=deduplicated,
    user_context=get_user_context(user_id)
  )
  
  // Step 5: Pagination
  paginated = ranked[offset:offset+limit]
  
  // Step 6: Hydrate post details
  posts = hydrate_posts(paginated)
  
  // Step 7: Mark posts as seen
  mark_posts_as_seen(user_id, paginated)
  
  return posts
```

**Time Complexity:** O(n log n) for ranking, where n = candidate posts

**Example Usage:**

```
posts = generate_feed(user_id=12345, limit=50, offset=0)
// Returns: [{post_id, user, caption, media_urls, like_count, ...}, ...]
```

---

### stitch_feed()

**Purpose:** Merges multiple post sources with specified ratios.

**Parameters:**

- `follower_posts` (list): Post IDs from followed users
- `celebrity_posts` (list): Post IDs from celebrities
- `recommended_posts` (list): Post IDs from ML recommendations
- `ads` (list): Ad IDs
- `ratios` (list): Mix ratios [follower, celebrity, reco, ads]

**Returns:**

- `merged` (list): Merged list of post IDs

**Algorithm:**

```
function stitch_feed(follower_posts, celebrity_posts, recommended_posts, ads, ratios):
  merged = []
  
  // Combine follower and celebrity posts into social feed
  social_posts = interleave(follower_posts, celebrity_posts)
  
  // Calculate cumulative ratios
  cumulative = [ratios[0], ratios[0] + ratios[1], ratios[0] + ratios[1] + ratios[2], 1.0]
  
  i = 0
  while len(merged) < 50 and (social_posts or recommended_posts or ads):
    // Determine which source to use based on ratio
    rand = random.uniform(0, 1)
    
    if rand < cumulative[0] and social_posts:
      // Social post (follower + celebrity)
      merged.append(social_posts.pop(0))
    elif rand < cumulative[1] and recommended_posts:
      // Recommended post
      merged.append(recommended_posts.pop(0))
    elif ads:
      // Ad
      merged.append(ads.pop(0))
    
    i += 1
  
  return merged
```

**Time Complexity:** O(n) where n = output size

**Example Usage:**

```
merged = stitch_feed(
  follower_posts=[1, 2, 3, 4, 5],
  celebrity_posts=[10, 11],
  recommended_posts=[20, 21, 22],
  ads=[100, 101],
  ratios=[0.6, 0.1, 0.3, 0.1]
)
// Returns: [1, 20, 2, 3, 100, 4, 21, 5, 10, ...]
```

---

### hydrate_posts()

**Purpose:** Fetches full post details from cache or database for a list of post IDs.

**Parameters:**

- `post_ids` (list): List of post IDs to hydrate

**Returns:**

- `posts` (list): List of post objects with full metadata

**Algorithm:**

```
function hydrate_posts(post_ids):
  posts = []
  cache_misses = []
  
  // Step 1: Batch fetch from Redis cache (pipeline)
  pipeline = redis.pipeline()
  for post_id in post_ids:
    pipeline.HGETALL(f"post:{post_id}")
  
  cached_results = pipeline.execute()
  
  // Step 2: Separate hits and misses
  for i, post_data in enumerate(cached_results):
    if post_data is not None:
      // Cache hit
      posts.append({
        ...post_data,
        post_id: post_ids[i]
      })
    else:
      // Cache miss
      cache_misses.append(post_ids[i])
  
  // Step 3: Fetch misses from database
  if cache_misses:
    db_results = postgres.SELECT(
      table='posts',
      where=f"post_id IN ({','.join(cache_misses)})"
    )
    
    // Step 4: Cache results and add to posts
    for post_data in db_results:
      // Write to cache
      redis.HMSET(f"post:{post_data.post_id}", post_data)
      redis.EXPIRE(f"post:{post_data.post_id}", 3600)  // 1 hour TTL
      
      posts.append(post_data)
  
  // Step 5: Fetch like/comment counts (from Redis counters)
  for post in posts:
    post.like_count = get_like_count(post.post_id)
    post.comment_count = get_comment_count(post.post_id)
  
  // Step 6: Fetch author details (user profiles)
  user_ids = [post.user_id for post in posts]
  users = hydrate_users(user_ids)
  user_map = {user.user_id: user for user in users}
  
  for post in posts:
    post.user = user_map[post.user_id]
  
  return posts
```

**Time Complexity:** O(n) with Redis pipelining, where n = number of posts

**Example Usage:**

```
posts = hydrate_posts([789, 790, 791])
// Returns: [
//   {post_id: 789, user: {...}, caption: "...", media_urls: {...}, like_count: 1234},
//   {post_id: 790, ...},
//   {post_id: 791, ...}
// ]
```

---

## 3. Fanout Strategies

### fanout_on_write()

**Purpose:** Pushes a new post to all followers' timelines (fanout-on-write strategy).

**Parameters:**

- `post_id` (int): Post identifier
- `user_id` (int): Author user ID

**Returns:**

- `fanout_count` (int): Number of followers fanout was performed to

**Algorithm:**

```
function fanout_on_write(post_id, user_id):
  // Step 1: Check follower count
  follower_count = get_follower_count(user_id)
  
  if follower_count > CELEBRITY_THRESHOLD:
    // Skip fanout for celebrities (use fanout-on-read instead)
    log("Skipping fanout for celebrity user", user_id)
    return 0
  
  // Step 2: Get follower list (paginated)
  followers = []
  offset = 0
  batch_size = 1000
  
  while True:
    batch = postgres.SELECT(
      table='follows',
      columns=['follower_id'],
      where=f"followee_id = {user_id}",
      limit=batch_size,
      offset=offset
    )
    
    if not batch:
      break
    
    followers.extend(batch)
    offset += batch_size
  
  // Step 3: Partition followers into batches for parallel processing
  batches = partition(followers, batch_size=100)
  
  // Step 4: Publish fanout jobs to Kafka (parallel workers)
  timestamp = current_timestamp()
  
  for batch in batches:
    kafka.publish("fanout.write", {
      post_id: post_id,
      follower_ids: batch,
      timestamp: timestamp
    })
  
  return len(followers)
```

**Time Complexity:** O(n) where n = number of followers

**Example Usage:**

```
count = fanout_on_write(post_id=789, user_id=123)
// Returns: 5000 (5,000 followers were fanout to)
```

---

### fanout_worker()

**Purpose:** Worker that processes fanout jobs and updates Redis timelines.

**Parameters:**

- `job` (dict): Fanout job containing post_id, follower_ids, timestamp

**Returns:**

- `success_count` (int): Number of successful timeline updates

**Algorithm:**

```
function fanout_worker(job):
  post_id = job.post_id
  follower_ids = job.follower_ids
  timestamp = job.timestamp
  
  success_count = 0
  
  // Use Redis pipeline for batch writes
  pipeline = redis.pipeline()
  
  for follower_id in follower_ids:
    timeline_key = f"timeline:{follower_id}"
    
    // Add post to follower's timeline (sorted set)
    pipeline.ZADD(timeline_key, timestamp, post_id)
    
    // Trim timeline to max 1000 posts (keep most recent)
    pipeline.ZREMRANGEBYRANK(timeline_key, 0, -1001)
    
    // Set TTL (24 hours)
    pipeline.EXPIRE(timeline_key, 86400)
  
  // Execute all commands in one batch
  results = pipeline.execute()
  
  // Count successes
  for result in results:
    if result == 1:  // ZADD returns 1 if new member added
      success_count += 1
  
  return success_count
```

**Time Complexity:** O(n log m) where n = batch size, m = timeline size

**Example Usage:**

```
count = fanout_worker({
  post_id: 789,
  follower_ids: [1, 2, 3, ..., 100],
  timestamp: 1699123456
})
// Returns: 100 (all 100 timelines updated successfully)
```

---

### fetch_celebrity_posts()

**Purpose:** Fetches recent posts from celebrities the user follows (fanout-on-read strategy).

**Parameters:**

- `user_id` (int): User requesting feed
- `limit` (int): Number of posts to fetch

**Returns:**

- `post_ids` (list): List of post IDs from celebrities

**Algorithm:**

```
function fetch_celebrity_posts(user_id, limit):
  // Step 1: Get celebrities that user follows
  celebrity_ids = postgres.SELECT(
    table='follows',
    columns=['followee_id'],
    where=f"follower_id = {user_id} AND followee.follower_count > {CELEBRITY_THRESHOLD}"
  )
  
  if not celebrity_ids:
    return []
  
  // Step 2: Fetch recent posts from those celebrities
  post_ids = postgres.SELECT(
    table='posts',
    columns=['post_id'],
    where=f"user_id IN ({','.join(celebrity_ids)}) AND deleted_at IS NULL",
    order_by='created_at DESC',
    limit=limit
  )
  
  return post_ids
```

**Time Complexity:** O(n × log m) where n = celebrity count, m = posts per celebrity

**Example Usage:**

```
post_ids = fetch_celebrity_posts(user_id=12345, limit=20)
// Returns: [456, 457, 458, ...] (20 posts from celebrities)
```

---

## 4. Engagement Operations

### like_post()

**Purpose:** Records a like on a post with multi-layer write (Redis, Cassandra, PostgreSQL).

**Parameters:**

- `user_id` (int): User liking the post
- `post_id` (int): Post being liked

**Returns:**

- `new_count` (int): Updated like count

**Algorithm:**

```
function like_post(user_id, post_id):
  // Step 1: Validate (check for duplicate)
  already_liked = redis.SISMEMBER(f"post:{post_id}:likers", user_id)
  
  if already_liked:
    throw DuplicateLikeError("User already liked this post")
  
  // Step 2: Layer 1 - Redis atomic counter (immediate response)
  shard_id = hash(user_id) % COUNTER_SHARDS
  new_count = redis.INCR(f"post:likes:{post_id}:{shard_id}")
  
  // Track unique likers
  redis.PFADD(f"post:likers:{post_id}", user_id)
  redis.SADD(f"post:{post_id}:likers", user_id)
  
  // Step 3: Layer 2 - Cassandra write (async, durable)
  async_execute(
    cassandra.INSERT(
      table='likes',
      values={
        post_id: post_id,
        user_id: user_id,
        created_at: NOW()
      }
    )
  )
  
  // Step 4: Publish event for Layer 3 (PostgreSQL batch update)
  kafka.publish("like.created", {
    post_id: post_id,
    user_id: user_id,
    timestamp: NOW()
  })
  
  // Step 5: Notify post author
  post_author = get_post_author(post_id)
  send_notification(post_author, f"User {user_id} liked your post")
  
  return new_count
```

**Time Complexity:** O(1) for Redis operations

**Example Usage:**

```
count = like_post(user_id=123, post_id=789)
// Returns: 1235 (new like count)
```

---

### get_like_count()

**Purpose:** Retrieves the total like count for a post (aggregates distributed counters).

**Parameters:**

- `post_id` (int): Post identifier

**Returns:**

- `total_likes` (int): Total like count

**Algorithm:**

```
function get_like_count(post_id):
  // Use Redis pipeline to fetch all shards in parallel
  pipeline = redis.pipeline()
  
  for shard_id in range(COUNTER_SHARDS):
    pipeline.GET(f"post:likes:{post_id}:{shard_id}")
  
  results = pipeline.execute()
  
  // Aggregate all shards
  total_likes = 0
  for count in results:
    if count is not None:
      total_likes += int(count)
  
  return total_likes
```

**Time Complexity:** O(k) where k = number of shards (constant)

**Example Usage:**

```
count = get_like_count(post_id=789)
// Returns: 1234567 (sum of all shards)
```

---

## 5. Image Processing

### ImageProcessor

**Purpose:** Processes uploaded images to generate multiple resolutions and formats.

**Algorithm:**

```
class ImageProcessor:
  
  function process_image(original_data, object_key):
    // Step 1: Load image
    image = load_image(original_data)
    
    // Step 2: Validate dimensions
    width, height = image.dimensions
    if width > MAX_WIDTH or height > MAX_HEIGHT:
      image = downscale(image, MAX_WIDTH, MAX_HEIGHT)
    
    // Step 3: Strip EXIF data (privacy)
    image = strip_exif(image)
    
    // Step 4: Generate versions in parallel
    versions = parallel_execute([
      generate_large(image, object_key),
      generate_medium(image, object_key),
      generate_thumbnail(image, object_key),
      generate_avif(image, object_key),
      generate_webp(image, object_key),
      generate_blurhash(image, object_key)
    ])
    
    media_urls = {
      'original': object_key,
      'large': versions[0],
      'medium': versions[1],
      'thumbnail': versions[2],
      'avif': versions[3],
      'webp': versions[4],
      'blurhash': versions[5]
    }
    
    return media_urls
  
  function generate_large(image, object_key):
    // Resize to 1080x1080 (feed size)
    resized = resize(image, 1080, 1080, method='lanczos')
    
    // Compress with quality=85
    compressed = compress(resized, format='JPEG', quality=85)
    
    // Upload to S3
    s3_key = object_key.replace('original', 'large')
    s3_client.upload(s3_key, compressed)
    
    return s3_key
  
  function generate_medium(image, object_key):
    resized = resize(image, 640, 640, method='lanczos')
    compressed = compress(resized, format='JPEG', quality=85)
    s3_key = object_key.replace('original', 'medium')
    s3_client.upload(s3_key, compressed)
    return s3_key
  
  function generate_thumbnail(image, object_key):
    resized = resize(image, 150, 150, method='lanczos')
    compressed = compress(resized, format='JPEG', quality=80)
    s3_key = object_key.replace('original', 'thumbnail')
    s3_client.upload(s3_key, compressed)
    return s3_key
  
  function generate_avif(image, object_key):
    resized = resize(image, 640, 640, method='lanczos')
    compressed = compress(resized, format='AVIF', quality=75)
    s3_key = object_key.replace('original', 'avif')
    s3_client.upload(s3_key, compressed)
    return s3_key
  
  function generate_webp(image, object_key):
    resized = resize(image, 640, 640, method='lanczos')
    compressed = compress(resized, format='WEBP', quality=80)
    s3_key = object_key.replace('original', 'webp')
    s3_client.upload(s3_key, compressed)
    return s3_key
  
  function generate_blurhash(image, object_key):
    // Resize to tiny 32x32 for placeholder
    tiny = resize(image, 32, 32, method='nearest')
    blurhash_string = encode_blurhash(tiny, components_x=4, components_y=3)
    return blurhash_string
```

**Time Complexity:** O(n × m) where n = image pixels, m = number of versions

**Example Usage:**

```
processor = ImageProcessor()
media_urls = processor.process_image(original_data, "uploads/123/abc.jpg")
// Returns: {original: "...", large: "...", medium: "...", ...}
```

---

## 6. Recommendation Engine

### RecommendationEngine

**Purpose:** Generates personalized content recommendations using ML models.

**Algorithm:**

```
class RecommendationEngine:
  
  function get_recommendations(user_id, limit=50):
    // Step 1: Generate candidates (1000 posts)
    candidates = generate_candidates(user_id, target=1000)
    
    // Step 2: Extract features
    user_features = get_user_features(user_id)
    post_features = get_post_features(candidates)
    context_features = get_context_features()
    
    // Step 3: Batch inference (ML model)
    scores = model.predict_batch({
      'user': user_features,
      'posts': post_features,
      'context': context_features
    })
    
    // Step 4: Combine scores with candidate IDs
    scored_candidates = []
    for i, post_id in enumerate(candidates):
      scored_candidates.append({
        'post_id': post_id,
        'score': scores[i]
      })
    
    // Step 5: Sort by score descending
    scored_candidates.sort(key=lambda x: x.score, reverse=True)
    
    // Step 6: Re-ranking (apply business rules)
    reranked = apply_business_rules(scored_candidates)
    
    // Step 7: Return top K
    top_k = [item.post_id for item in reranked[:limit]]
    
    return top_k
  
  function generate_candidates(user_id, target=1000):
    candidates = []
    
    // Source 1: Popular posts (last 24 hours)
    popular = get_popular_posts(limit=400)
    candidates.extend(popular)
    
    // Source 2: Posts from similar users (collaborative filtering)
    similar_users = get_similar_users(user_id, limit=50)
    for similar_user in similar_users:
      recent = get_recent_posts(similar_user, limit=6)
      candidates.extend(recent)
    
    // Source 3: Trending hashtags user follows
    trending = get_trending_hashtags(user_id, limit=200)
    candidates.extend(trending)
    
    // Source 4: Visual similarity (from user's past likes)
    past_likes = get_past_likes(user_id, limit=10)
    for liked_post in past_likes:
      similar = get_visually_similar(liked_post, limit=10)
      candidates.extend(similar)
    
    // Deduplicate
    candidates = list(set(candidates))
    
    // Shuffle and take target
    random.shuffle(candidates)
    return candidates[:target]
  
  function get_user_features(user_id):
    return {
      'user_id': user_id,
      'past_likes': get_past_likes_embedding(user_id),
      'activity_level': get_activity_score(user_id),
      'follower_count': get_follower_count(user_id),
      'account_age': get_account_age(user_id)
    }
  
  function get_post_features(post_ids):
    posts_data = []
    
    for post_id in post_ids:
      posts_data.append({
        'post_id': post_id,
        'like_count': get_like_count(post_id),
        'comment_count': get_comment_count(post_id),
        'engagement_rate': calculate_engagement_rate(post_id),
        'age_hours': get_post_age_hours(post_id),
        'visual_features': get_visual_embedding(post_id)
      })
    
    return posts_data
  
  function apply_business_rules(scored_candidates):
    reranked = []
    author_count = {}
    
    for item in scored_candidates:
      author = get_post_author(item.post_id)
      
      // Rule 1: Diversity - max 2 posts per author
      if author_count.get(author, 0) >= 2:
        continue
      
      // Rule 2: Freshness boost - posts < 6 hours old get +0.1
      age_hours = get_post_age_hours(item.post_id)
      if age_hours < 6:
        item.score += 0.1
      
      // Rule 3: Novelty penalty - if similar to recent views, -0.05
      if is_similar_to_recent_views(user_id, item.post_id):
        item.score -= 0.05
      
      reranked.append(item)
      author_count[author] = author_count.get(author, 0) + 1
    
    // Re-sort after adjustments
    reranked.sort(key=lambda x: x.score, reverse=True)
    
    return reranked
```

**Time Complexity:** O(n log n) for sorting, where n = candidates

**Example Usage:**

```
engine = RecommendationEngine()
recommended_posts = engine.get_recommendations(user_id=12345, limit=50)
// Returns: [456, 457, 458, ...] (50 recommended post IDs)
```

---

## 7. Cache Operations

### invalidate_cache()

**Purpose:** Invalidates all caches for a post (Redis, CDN) when post is edited or deleted.

**Parameters:**

- `post_id` (int): Post identifier
- `user_id` (int): Author user ID

**Returns:**

- `invalidated_count` (int): Number of cache entries invalidated

**Algorithm:**

```
function invalidate_cache(post_id, user_id):
  invalidated_count = 0
  
  // Step 1: Delete post cache
  result = redis.DEL(f"post:{post_id}")
  invalidated_count += result
  
  // Step 2: Delete like/comment counters
  for shard_id in range(COUNTER_SHARDS):
    redis.DEL(f"post:likes:{post_id}:{shard_id}")
    redis.DEL(f"post:comments:{post_id}:{shard_id}")
  
  // Step 3: Remove from all followers' timelines
  followers = get_followers(user_id)
  
  pipeline = redis.pipeline()
  for follower_id in followers:
    pipeline.ZREM(f"timeline:{follower_id}", post_id)
  
  results = pipeline.execute()
  invalidated_count += sum(results)
  
  // Step 4: Purge from CDN
  media_urls = get_media_urls(post_id)
  
  for url in media_urls.values():
    cdn_client.purge(url)
  
  invalidated_count += len(media_urls)
  
  return invalidated_count
```

**Time Complexity:** O(n) where n = number of followers

**Example Usage:**

```
count = invalidate_cache(post_id=789, user_id=123)
// Returns: 5234 (5,234 cache entries invalidated)
```

---

## 8. Visual Search

### visual_search()

**Purpose:** Finds visually similar images using CNN embeddings and vector search.

**Parameters:**

- `post_id` (int): Query image post ID
- `limit` (int): Number of similar images to return

**Returns:**

- `similar_post_ids` (list): List of similar post IDs

**Algorithm:**

```
function visual_search(post_id, limit=50):
  // Step 1: Fetch query embedding
  query_embedding = vector_db.get_embedding(post_id)
  
  if query_embedding is None:
    throw Error("Embedding not found for post")
  
  // Step 2: Run ANN search (Approximate Nearest Neighbor)
  results = vector_db.search(
    query_vector=query_embedding,
    k=100,  // Fetch 100 candidates
    algorithm='HNSW'  // Hierarchical Navigable Small World
  )
  
  // Step 3: Filter results
  filtered = []
  for result in results:
    candidate_post_id = result.post_id
    score = result.similarity_score
    
    // Skip the query post itself
    if candidate_post_id == post_id:
      continue
    
    // Skip deleted posts
    if is_post_deleted(candidate_post_id):
      continue
    
    filtered.append({
      'post_id': candidate_post_id,
      'score': score
    })
  
  // Step 4: Re-rank by engagement
  for item in filtered:
    engagement_score = get_engagement_score(item.post_id)
    item.score = item.score * 0.7 + engagement_score * 0.3
  
  // Step 5: Sort by combined score
  filtered.sort(key=lambda x: x.score, reverse=True)
  
  // Step 6: Return top K
  similar_post_ids = [item.post_id for item in filtered[:limit]]
  
  return similar_post_ids
```

**Time Complexity:** O(log n) for HNSW search, where n = index size

**Example Usage:**

```
similar = visual_search(post_id=789, limit=50)
// Returns: [790, 791, 792, ...] (50 visually similar posts)
```

---

## 9. Hot Key Mitigation

### distributed_counter()

**Purpose:** Handles high-throughput counter updates using distributed sharding.

**Parameters:**

- `key` (string): Base counter key
- `user_id` (int): User performing action (used for sharding)
- `increment` (int): Increment value (default: 1)

**Returns:**

- `new_value` (int): Approximate new counter value

**Algorithm:**

```
function distributed_counter(key, user_id, increment=1):
  // Step 1: Compute shard ID
  shard_id = hash(user_id) % COUNTER_SHARDS
  
  // Step 2: Increment specific shard
  shard_key = f"{key}:{shard_id}"
  new_shard_value = redis.INCRBY(shard_key, increment)
  
  // Step 3: Return approximate total (no need to sum all shards on write)
  // Note: For writes, we return the shard value as confirmation
  // For reads, use aggregate_distributed_counter()
  
  return new_shard_value
```

**Time Complexity:** O(1)

**Example Usage:**

```
// Write (like action)
new_value = distributed_counter("post:likes:789", user_id=123, increment=1)

// Read (get total count)
total = aggregate_distributed_counter("post:likes:789")
```

---

### aggregate_distributed_counter()

**Purpose:** Aggregates all shards to get total counter value.

**Parameters:**

- `key` (string): Base counter key

**Returns:**

- `total` (int): Total counter value across all shards

**Algorithm:**

```
function aggregate_distributed_counter(key):
  // Use Redis pipeline for parallel reads
  pipeline = redis.pipeline()
  
  for shard_id in range(COUNTER_SHARDS):
    shard_key = f"{key}:{shard_id}"
    pipeline.GET(shard_key)
  
  results = pipeline.execute()
  
  // Sum all shards
  total = 0
  for value in results:
    if value is not None:
      total += int(value)
  
  return total
```

**Time Complexity:** O(k) where k = number of shards (constant)

**Example Usage:**

```
total_likes = aggregate_distributed_counter("post:likes:789")
// Returns: 1234567 (sum of all 100 shards)
```

---

## 10. Circuit Breaker Pattern

### CircuitBreaker

**Purpose:** Prevents cascading failures by failing fast when downstream service is unhealthy.

**Algorithm:**

```
class CircuitBreaker:
  
  function __init__(failure_threshold=0.5, timeout=30):
    self.state = 'CLOSED'  // CLOSED, OPEN, HALF_OPEN
    self.failure_count = 0
    self.success_count = 0
    self.total_requests = 0
    self.failure_threshold = failure_threshold
    self.timeout = timeout
    self.last_failure_time = None
  
  function call(func, *args, **kwargs):
    // Check circuit state
    if self.state == 'OPEN':
      // Check if timeout has passed
      if time.now() - self.last_failure_time > self.timeout:
        self.state = 'HALF_OPEN'
        log("Circuit breaker entering HALF_OPEN state")
      else:
        throw CircuitBreakerOpen("Circuit breaker is OPEN")
    
    try:
      // Execute function
      result = func(*args, **kwargs)
      
      // Record success
      self.record_success()
      
      return result
    
    except Exception as e:
      // Record failure
      self.record_failure()
      
      throw e
  
  function record_success():
    self.success_count += 1
    self.total_requests += 1
    
    // If in HALF_OPEN, close circuit after success
    if self.state == 'HALF_OPEN':
      self.state = 'CLOSED'
      self.failure_count = 0
      log("Circuit breaker CLOSED (service recovered)")
  
  function record_failure():
    self.failure_count += 1
    self.total_requests += 1
    self.last_failure_time = time.now()
    
    // Check failure rate
    failure_rate = self.failure_count / self.total_requests
    
    if failure_rate > self.failure_threshold:
      self.state = 'OPEN'
      log("Circuit breaker OPEN (failure rate:", failure_rate, ")")
```

**Time Complexity:** O(1)

**Example Usage:**

```
// Protect recommendation engine calls
circuit_breaker = CircuitBreaker(failure_threshold=0.5, timeout=30)

try:
  result = circuit_breaker.call(recommendation_engine.get_recommendations, user_id=123)
except CircuitBreakerOpen:
  // Fallback: Return empty recommendations
  result = []
```
