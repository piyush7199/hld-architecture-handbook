# Notification Service - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions for the Notification Service, explaining why specific choices were made over alternatives.

## Table of Contents

1. [Decision 1: Kafka vs Synchronous REST APIs](#decision-1-kafka-vs-synchronous-rest-apis)
2. [Decision 2: Cassandra vs PostgreSQL for Logs](#decision-2-cassandra-vs-postgresql-for-logs)
3. [Decision 3: WebSockets vs Long Polling](#decision-3-websockets-vs-long-polling)
4. [Decision 4: Multi-Channel Workers vs Single Worker](#decision-4-multi-channel-workers-vs-single-worker)
5. [Decision 5: Redis Cache vs No Cache](#decision-5-redis-cache-vs-no-cache)
6. [Decision 6: Template-Based vs Hard-Coded Notifications](#decision-6-template-based-vs-hard-coded-notifications)
7. [Decision 7: Push Model vs Pull Model](#decision-7-push-model-vs-pull-model)
8. [Decision 8: Dedicated Workers vs Serverless (AWS Lambda)](#decision-8-dedicated-workers-vs-serverless-aws-lambda)
9. [Decision 9: Token Bucket vs Fixed Window Rate Limiting](#decision-9-token-bucket-vs-fixed-window-rate-limiting)
10. [Decision 10: Circuit Breaker Pattern vs No Circuit Breaker](#decision-10-circuit-breaker-pattern-vs-no-circuit-breaker)

---

## Decision 1: Kafka vs Synchronous REST APIs

### The Problem

The Notification Service needs to handle huge bursts of traffic (57k QPS peak) and guarantee delivery even if downstream services fail. The question is: Should the Notification Service call channel workers synchronously via REST APIs, or should it use an asynchronous message queue like Kafka?

### Options Considered

| Option | Pros | Cons | Performance | Cost | Complexity |
|--------|------|------|-------------|------|------------|
| **Synchronous REST** | Simple implementation, immediate feedback, easy debugging | Cannot handle traffic bursts, no retry mechanism, tight coupling, blocking calls | Limited by slowest service (~5s for email), cannot scale beyond synchronous capacity | Lower (no Kafka cluster) | Low |
| **Kafka** | Buffers traffic spikes, guaranteed delivery, replay capability, decoupling | Adds latency (~100ms), requires Kafka expertise, more components to manage | Can handle 1M+ msg/s, non-blocking | Higher (Kafka cluster ~$5k/month) | High |
| **RabbitMQ** | Simpler than Kafka, lower latency (~5ms), traditional queue semantics | Lower throughput (~20k msg/s), no replay, messages deleted after consumption | Moderate (20k msg/s per broker) | Moderate (~$2k/month) | Medium |
| **AWS SQS** | Fully managed, no infrastructure, auto-scaling, pay-per-use | No ordering guarantees, higher latency (~10-20ms), vendor lock-in | Moderate (depends on provisioned throughput) | Pay-per-request | Low (managed) |

### Decision Made

**Use Apache Kafka as the central message stream between Notification Service and Channel Workers.**

**Rationale:**

1. **Traffic Bursts**: Must handle peak loads of 57k QPS during major events (e.g., product launches, flash sales). Synchronous REST would overwhelm workers and cause failures.

2. **Guaranteed Delivery**: Kafka persists messages to disk with configurable retention (7 days). Even if all workers are down, no notifications are lost—they're processed when workers recover.

3. **Retry Mechanism**: Workers can retry failed deliveries by not committing the Kafka offset. The message remains available for reprocessing.

4. **Decoupling**: Notification Service doesn't need to wait for channel workers to respond. It publishes to Kafka and immediately returns 202 Accepted to the caller.

5. **Scalability**: Can easily add more worker instances without changing producer code. Kafka automatically distributes messages across consumers via partition assignment.

6. **Replay Capability**: For debugging or reprocessing, we can replay messages from Kafka by resetting consumer group offsets.

7. **Ordering**: Kafka guarantees ordering within a partition. By partitioning by `user_id`, we ensure all notifications for a given user are processed in order.

8. **High Throughput**: Kafka can handle 1M+ messages per second, far exceeding our peak requirement of 57k QPS.

### Implementation Details

**Kafka Topic Structure:**
```
- email-notifications (12 partitions, replication factor: 3)
- sms-notifications (6 partitions, replication factor: 3)
- push-notifications (12 partitions, replication factor: 3)
- web-notifications (6 partitions, replication factor: 3)
- dlq-email (DLQ for failed emails)
- dlq-sms (DLQ for failed SMS)
- dlq-push (DLQ for failed push notifications)
- dlq-web (DLQ for failed web notifications)
```

**Partitioning Strategy:**
- Partition by `user_id` using hash function: `partition = hash(user_id) % num_partitions`
- Ensures all notifications for a user go to same partition (ordering)
- Distributes load evenly across partitions

**Consumer Groups:**
- Each channel has its own consumer group (email-workers, sms-workers, etc.)
- Multiple worker instances in each group for parallelism
- Kafka automatically distributes partitions across consumers

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| Guaranteed delivery, no message loss | ~100ms added latency (acceptable for most notifications) |
| Can handle 57k QPS peaks | Increased system complexity (Kafka cluster management) |
| Automatic retry and replay | Eventual consistency (notifications may arrive slightly out of order across channels) |
| Worker independence | Need Kafka expertise on team |
| Horizontal scalability | Higher infrastructure cost (~$5k/month for Kafka cluster) |

### When to Reconsider

**Consider Synchronous REST if:**
- Notification volume is low (<1000 QPS)
- All notifications require immediate feedback within same request
- Team lacks Kafka expertise and cannot invest in learning
- Budget is extremely tight

**Consider RabbitMQ if:**
- Notification volume is moderate (5k-20k QPS)
- Lower latency is critical (<10ms vs Kafka's ~100ms)
- Don't need message replay capability
- Prefer traditional queue semantics

**Consider AWS SQS if:**
- Want fully managed solution with zero infrastructure management
- Message ordering is not critical
- Vendor lock-in with AWS is acceptable
- Pay-per-use pricing is more cost-effective for your scale

### Real-World Examples

- **Uber**: Uses Kafka for ride matching events, notification triggers
- **Netflix**: Uses Kafka for asynchronous microservice communication, including notifications
- **LinkedIn**: Uses Kafka for feed updates, notification distribution
- **Slack**: Uses Kafka for message delivery and notification fan-out

---

## Decision 2: Cassandra vs PostgreSQL for Logs

### The Problem

The system needs to log every notification event for auditing and analytics. With 1 billion notifications per day, we need a database that can handle extremely high write throughput while remaining cost-effective. Should we use a traditional relational database (PostgreSQL) or a NoSQL wide-column store (Cassandra)?

### Options Considered

| Option | Pros | Cons | Write Throughput | Cost | Query Flexibility |
|--------|------|------|------------------|------|-------------------|
| **PostgreSQL** | ACID guarantees, powerful queries (joins, aggregations), familiar SQL, excellent for analytics | Limited write scalability (~10k writes/sec per instance), vertical scaling expensive, complex sharding | 10k writes/sec per instance | High (vertical scaling), ~$10k/month for 100k writes/sec | Excellent (full SQL) |
| **Cassandra** | Extremely high write throughput (100k+ writes/sec per node), linear horizontal scaling, time-series optimized | No joins, limited aggregations, eventual consistency, requires specific query patterns | 100k+ writes/sec per node | Lower (horizontal scaling), ~$3k/month for same throughput | Limited (denormalized queries) |
| **MongoDB** | Flexible schema, powerful query language, easier to learn than Cassandra | Lower write throughput than Cassandra (~50k writes/sec), not time-series optimized | 50k writes/sec | Moderate, ~$5k/month | Good (aggregation framework) |
| **DynamoDB** | Fully managed, auto-scaling, predictable latency, pay-per-use | Vendor lock-in, expensive at high scale, limited query patterns | Depends on provisioned capacity | High at scale (~$8k/month), pay-per-request | Limited (key-value) |
| **ClickHouse** | Optimized for analytics, columnar storage, compression, fast aggregations | Not ideal for transactional writes, complex setup, less mature ecosystem | Very high (columnar storage) | Moderate | Excellent for analytics |

### Decision Made

**Use Apache Cassandra for notification logs.**

### Rationale

1. **Write-Heavy Workload**: 1 billion writes per day = ~11,574 writes/second average, ~57,870 writes/second peak. PostgreSQL would require complex sharding and expensive vertical scaling.

2. **Time-Series Data**: Notification logs are append-only time-series data with natural partitioning by `user_id` and `created_at`. Cassandra is optimized for this use case.

3. **Horizontal Scalability**: Can easily add more Cassandra nodes to increase write capacity. Near-linear scalability (10 nodes = 1M writes/second).

4. **Cost-Effective**: At our scale, Cassandra is ~3x cheaper than PostgreSQL for equivalent write throughput.

5. **Simple Query Pattern**: Most queries are:
   - Fetch logs for a user in a time range: `SELECT * FROM logs WHERE user_id = ? AND created_at > ?`
   - Fetch log by notification_id: `SELECT * FROM logs WHERE notification_id = ?`
   - No complex joins needed (denormalized data)

6. **Availability**: Cassandra's replication (RF=3) ensures high availability. Can tolerate node failures without data loss.

7. **TTL Support**: Cassandra has built-in Time-To-Live (TTL) support. Can automatically delete logs older than 1 year to manage storage.

### Implementation Details

**Schema Design:**
```sql
-- Main logs table (partitioned by user_id)
CREATE TABLE notification_logs (
    user_id BIGINT,
    created_at TIMESTAMP,
    notification_id UUID,
    channel TEXT, -- 'email', 'sms', 'push', 'web'
    status TEXT, -- 'sent', 'delivered', 'failed', 'opened', 'clicked'
    template_id TEXT,
    delivered_at TIMESTAMP,
    error_message TEXT,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY ((user_id), created_at, notification_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Lookup table by notification_id
CREATE TABLE notification_status (
    notification_id UUID PRIMARY KEY,
    user_id BIGINT,
    channel TEXT,
    status TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Optional: Table for analytics (aggregated metrics)
CREATE TABLE daily_notification_stats (
    date DATE,
    channel TEXT,
    hour INT,
    sent_count COUNTER,
    delivered_count COUNTER,
    failed_count COUNTER,
    PRIMARY KEY ((date, channel), hour)
);
```

**Partitioning Strategy:**
- Partition by `user_id` to ensure all logs for a user are co-located
- Cluster by `created_at DESC` for efficient time-range queries
- Each partition typically contains 1-10k rows (good size)

**Compaction Strategy:**
- Use TimeWindowCompactionStrategy for time-series data
- Automatically merge SSTables in time windows
- Efficient for delete-heavy workloads (TTL)

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| 100k+ writes/second per node | No complex joins (must denormalize data) |
| Linear horizontal scaling | Eventual consistency (logs may take 1-2 seconds to appear) |
| Cost-effective at scale (~3x cheaper) | Limited ad-hoc queries (need predefined query patterns) |
| Built-in TTL for automatic deletion | Learning curve for Cassandra-specific concepts |
| High availability (RF=3) | No ACID transactions across partitions |

### When to Reconsider

**Consider PostgreSQL if:**
- Notification volume is low (<10M per day)
- Need complex analytics with joins and aggregations
- ACID guarantees are critical for compliance
- Team has deep PostgreSQL expertise but no Cassandra experience
- Budget allows for expensive vertical scaling

**Consider ClickHouse if:**
- Primary use case is analytics (dashboards, reports, aggregations)
- Need columnar storage for efficient compression
- Can tolerate eventual consistency for analytics data
- Have heavy read queries that benefit from columnar format

**Consider DynamoDB if:**
- Want fully managed solution with zero operational overhead
- Using AWS ecosystem extensively
- Query patterns are simple (key-value lookups)
- Can tolerate higher cost at scale

**Hybrid Approach:**
- Use Cassandra for real-time logs (write-optimized)
- Stream to ClickHouse or Data Warehouse for analytics (read-optimized)
- Best of both worlds: fast writes + powerful analytics

### Real-World Examples

- **Netflix**: Uses Cassandra for video viewing history, user activity logs
- **Uber**: Uses Cassandra for trip data, driver location history
- **Apple**: Uses Cassandra for iCloud data, storing billions of records
- **Instagram**: Uses Cassandra for user feed data, notifications

---

## Decision 3: WebSockets vs Long Polling

### The Problem

For real-time in-app notifications (bell icon), users need to be notified immediately when events occur. What communication protocol should we use to deliver notifications with minimal latency while being efficient on server resources and mobile battery?

### Options Considered

| Option | Pros | Cons | Latency | Efficiency | Browser Support |
|--------|------|------|---------|------------|-----------------|
| **WebSockets** | True real-time (<100ms), bi-directional, low overhead, battery-friendly | Complex connection management, requires sticky sessions, firewall issues (rare) | <100ms | Very high (single connection) | Excellent (all modern browsers) |
| **Long Polling** | Simple implementation, firewall-friendly, works on old browsers | Higher latency (1-5s), wasted requests, higher server load, battery drain | 1-5s | Low (repeated HTTP handshakes) | Universal |
| **Server-Sent Events (SSE)** | Simple, uni-directional push, automatic reconnection | No IE support, uni-directional only (no client → server), connection limits | <500ms | High (persistent HTTP) | Good (except IE) |
| **HTTP/2 Server Push** | Leverages HTTP/2, no extra connections, efficient | Limited browser support, complex implementation, not designed for notifications | Variable | Moderate | Limited |

### Decision Made

**Use WebSockets for real-time web notifications with Server-Sent Events (SSE) as fallback.**

### Rationale

1. **Latency**: WebSockets provide <100ms latency from event trigger to user seeing notification. Long polling has 1-5 second latency depending on polling interval.

2. **Efficiency**: Single persistent connection vs repeated HTTP requests. Long polling with 30-second intervals = 2,880 requests/day/user. WebSockets = 1 connection.

3. **Battery Life**: Mobile devices benefit significantly from WebSockets. Long polling drains battery due to constant wake-ups for new requests.

4. **Bi-directional**: WebSockets allow clients to send acknowledgments back to server. Useful for tracking when user actually saw the notification.

5. **Industry Standard**: Used by Slack, WhatsApp, Facebook Messenger, and other real-time applications.

6. **Scalability**: 10,000 concurrent connections per server is easily achievable. Total capacity: 1,000 servers = 10M concurrent connections.

### Implementation Details

**Connection Management:**
```
1. Client connects: wss://notifications.example.com/ws
2. Server assigns connection to WebSocket server pool (sticky session)
3. Store mapping in Redis: user:123 → server:42
4. Heartbeat every 30 seconds to detect disconnections
5. On disconnect: Remove from Redis, close socket
6. On notification: Lookup Redis, publish to correct server via Redis Pub/Sub
```

**Load Balancing:**
- Use sticky sessions (hash by user_id) to route same user to same server
- Helps with reconnection (same server has context)
- Distribute load evenly across WebSocket server pool

**Fallback Strategy:**
```
if (WebSocket supported && not blocked) {
    use WebSocket
} else if (SSE supported) {
    use Server-Sent Events
} else {
    fallback to long polling (30-second interval)
}
```

**Heartbeat Protocol:**
```
Client → Server: {"type": "ping"} (every 30 seconds)
Server → Client: {"type": "pong"}

If 3 consecutive pings missed:
    - Server closes connection
    - Client detects and reconnects with exponential backoff
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| Real-time delivery (<100ms) | Complex connection management (track connections in Redis) |
| Battery efficiency | Requires sticky sessions (limits load balancer flexibility) |
| Bi-directional communication | Rare firewall issues (< 1% of corporate networks) |
| Low overhead (single connection) | Stateful architecture (more complex than stateless) |
| Industry-proven pattern | Need fallback mechanism for edge cases |

### When to Reconsider

**Consider Long Polling if:**
- Latency requirement is relaxed (>30 seconds acceptable)
- User base is mostly on old browsers (IE 9 and below)
- Notification volume is very low (<1 per minute per user)
- Cannot implement sticky sessions in load balancer
- Most users are behind corporate firewalls that block WebSockets

**Consider Server-Sent Events (SSE) if:**
- Only need uni-directional push (server → client)
- Don't need to receive acknowledgments from clients
- Want simpler implementation than WebSockets
- Can accept slightly higher latency (500ms vs <100ms)

**Consider HTTP/2 Server Push if:**
- Already using HTTP/2 for all communication
- Notification volume is low and predictable
- Don't need persistent real-time connection

### Real-World Examples

- **Slack**: Uses WebSockets for real-time messaging and notifications
- **WhatsApp Web**: WebSockets for message delivery
- **Facebook**: WebSockets for messenger, news feed updates
- **Trello**: WebSockets for real-time board updates

---

## Decision 4: Multi-Channel Workers vs Single Worker

### The Problem

The system needs to deliver notifications across multiple channels (Email, SMS, Push, Web). Should we have a single worker service that handles all channels, or separate worker microservices for each channel?

### Options Considered

| Option | Pros | Cons | Scalability | Complexity | Failure Isolation |
|--------|------|------|-------------|------------|-------------------|
| **Single Worker** | Simpler architecture, less overhead, easier deployment, shared code | Cannot scale channels independently, rate limiting complex, failures affect all channels | Limited (scale all or nothing) | Low | None (all channels fail together) |
| **Multi-Channel Workers** | Independent scaling, failure isolation, channel-specific optimizations, clear ownership | More microservices to manage, code duplication, more Kafka topics | Excellent (per-channel) | High | Excellent |
| **Hybrid (Single with Handlers)** | Moderate complexity, some code reuse, easier than multi-service | Still cannot scale independently, rate limiting conflicts between channels | Moderate | Moderate | Partial |

### Decision Made

**Use separate worker microservices for each channel (Email Worker, SMS Worker, Push Worker, Web Worker).**

### Rationale

1. **Independent Scaling**: Email has different load patterns than Push. During marketing campaigns, email volume spikes 10x while push remains stable. Need to scale email workers independently.

2. **Failure Isolation**: If Email Worker crashes due to SendGrid API issue, Push and SMS notifications continue to work. Single worker = all channels fail.

3. **Rate Limiting**: Each third-party provider has different rate limits:
   - SendGrid: 1,000 emails/minute
   - Twilio: 100 SMS/second
   - FCM: 3,000 push/second
   
   Separate workers can implement channel-specific rate limiters without conflicts.

4. **Specialized Logic**: Each channel has unique requirements:
   - Push: Batch API calls (100 notifications per request)
   - Email: Template rendering with HTML/CSS
   - SMS: Character limits, cost optimization
   - Web: WebSocket connection management
   
   Dedicated workers can optimize for channel-specific needs.

5. **Team Ownership**: Different teams can own different channels. Mobile team owns Push Worker, Messaging team owns Email Worker.

6. **Circuit Breaker**: Each channel needs its own circuit breaker. If SendGrid is down, only Email circuit breaker opens. Other channels continue working.

7. **Deployment**: Can deploy Email Worker updates without affecting Push or SMS workers. Reduces blast radius.

### Implementation Details

**Worker Architecture:**
```
Email Worker:
- Consumes from: email-notifications topic
- Rate Limiter: Token Bucket (1,000/minute)
- Circuit Breaker: Threshold 5, Timeout 60s
- API Client: SendGrid SDK
- Batch Size: 1 (individual emails)

SMS Worker:
- Consumes from: sms-notifications topic
- Rate Limiter: Token Bucket (100/second)
- Circuit Breaker: Threshold 5, Timeout 60s
- API Client: Twilio SDK
- Batch Size: 1

Push Worker:
- Consumes from: push-notifications topic
- Rate Limiter: Token Bucket (3,000/second)
- Circuit Breaker: Threshold 5, Timeout 60s
- API Client: FCM SDK
- Batch Size: 100 (batch API)

Web Worker:
- Consumes from: web-notifications topic
- No rate limiter needed (internal)
- Circuit Breaker: Not needed (internal)
- Delivery: Redis Pub/Sub → WebSocket Server
- Batch Size: 1
```

**Shared Library:**
- Create shared library for common code (logging, metrics, base worker)
- Each worker imports library and implements channel-specific logic
- Reduces code duplication while maintaining separation

**Kafka Topic Strategy:**
- One topic per channel: email-notifications, sms-notifications, etc.
- Each worker has its own consumer group
- Workers don't interfere with each other

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| Independent scaling per channel | More microservices to manage (4 vs 1) |
| Failure isolation (email down ≠ push down) | Code duplication for common logic |
| Channel-specific optimizations | More Kafka topics to maintain |
| Clear team ownership | Higher operational overhead |
| Flexible deployment (update one without others) | More complex monitoring (4 dashboards vs 1) |

### When to Reconsider

**Consider Single Worker if:**
- Notification volume is very low (<1,000 per day)
- All channels have similar load patterns
- Team is very small (1-2 engineers)
- Operational simplicity is more important than scalability
- Channels have very similar delivery logic

**Consider Hybrid Approach if:**
- Want moderate complexity
- Can accept scaling all channels together
- Have some shared logic but channel-specific parts are minimal
- Budget is tight (fewer servers)

### Real-World Examples

- **Twilio**: Separate services for Email (SendGrid) and SMS (Twilio)
- **AWS**: Separate services: SNS (push/SMS), SES (email)
- **Slack**: Separate workers for web notifications, mobile push, email
- **Uber**: Separate workers for push, SMS, email notifications

---

## Decision 5: Redis Cache vs No Cache

### The Problem

The Notification Service needs to fetch user preferences for every notification. With 11,574 QPS average (57,870 peak), querying PostgreSQL directly would overwhelm the database and add significant latency. Should we introduce a caching layer?

### Options Considered

| Option | Pros | Cons | Latency | DB Load | Cost |
|--------|------|------|---------|---------|------|
| **No Cache (Direct DB)** | Simple architecture, always fresh data, no cache invalidation complexity | High DB load (11k QPS), higher latency (10-50ms), requires large DB instance | 10-50ms per query | Very high | High (large DB instance: ~$5k/month) |
| **Redis Cache** | Low latency (<1ms), reduces DB load by 95%+, horizontal scaling | Stale data (TTL-based), cache invalidation complexity, additional component | <1ms per query | Very low | Lower (small DB + Redis: ~$3k/month) |
| **Application-Level Cache** | No external dependency, simplest, zero network latency | Cannot share across instances, limited memory, stale data, no distribution | <0.1ms | Moderate | Lowest |
| **CDN/Edge Cache** | Global distribution, lowest latency for geo-distributed users | Complex invalidation, high cost, not suitable for user-specific data | <10ms globally | Very low | Highest |

### Decision Made

**Use Redis as a caching layer for user preferences, templates, and device tokens.**

### Rationale

1. **Reduced DB Load**: Without cache, PostgreSQL would handle 11,574 reads/second. With 98% cache hit rate, only 232 reads/second hit the database.

2. **Low Latency**: Redis lookup: <1ms. PostgreSQL query: 10-50ms. For API response time SLA (<100ms), every millisecond counts.

3. **Horizontal Scaling**: Redis Cluster can scale to millions of requests per second by adding more shards. PostgreSQL read scaling requires read replicas (more complex).

4. **Cost Savings**: Caching allows using smaller PostgreSQL instance. Small DB ($500/month) + Redis cluster ($2.5k/month) = $3k total vs Large DB without cache ($5k/month).

5. **Improved Availability**: If PostgreSQL has temporary issues, Redis cache keeps the system operational for recently accessed users.

6. **Multi-Purpose**: Redis also used for:
   - Idempotency keys (prevent duplicate notifications)
   - WebSocket connection registry (track which server has user's connection)
   - Rate limiting state (token bucket counters)

### Implementation Details

**Cache Strategy:**
```
Cache-Aside Pattern:
1. Check Redis for key (e.g., user:prefs:123)
2. If found (cache hit): Return cached data
3. If not found (cache miss): Query PostgreSQL
4. Write result to Redis with TTL
5. Return data to caller

Cache Invalidation:
- On user preference update: DELETE key from Redis
- Or: Publish invalidation event to Redis Pub/Sub
- Workers subscribe and invalidate their local caches
```

**TTL Strategy:**
```
user:prefs:* → TTL: 5 minutes (preferences rarely change)
template:* → TTL: 1 hour (templates almost never change)
device:tokens:* → TTL: 10 minutes (tokens change when user reinstalls app)
idempotency:* → TTL: 24 hours (prevent duplicates within a day)
```

**Redis Cluster Configuration:**
```
- 3 master shards (for horizontal scaling)
- 3 slave replicas (for high availability)
- Consistent hashing for data distribution
- Memory: 64GB per shard = 192GB total
- Cost: ~$2.5k/month on AWS ElastiCache
```

**Cache Hit Rate Monitoring:**
```
Target: >95% cache hit rate
Alert if: Cache hit rate <80% for 5 minutes
Metrics:
- Cache hits per second
- Cache misses per second
- Average lookup latency
- Memory usage per shard
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| <1ms latency (vs 10-50ms) | Stale data for up to TTL duration (5 minutes) |
| 95%+ reduction in DB load | Cache invalidation complexity |
| Horizontal scalability (millions of req/s) | Additional component to manage (Redis cluster) |
| Cost savings (~40% cheaper) | Memory cost (~$2.5k/month for Redis) |
| Improved system resilience | Learning curve for Redis best practices |

### When to Reconsider

**Consider No Cache if:**
- Notification volume is very low (<100 QPS)
- Real-time preference updates are critical (cannot tolerate any staleness)
- Team lacks Redis expertise
- Budget is extremely tight
- All users have different preferences (cache hit rate <50%)

**Consider Application-Level Cache if:**
- Data is static or rarely changes
- Single-server deployment (no horizontal scaling needed)
- Want to avoid external dependencies
- Cache size fits in memory (< 1GB)

**Consider Alternative Cache (Memcached) if:**
- Don't need advanced Redis features (Pub/Sub, data structures)
- Pure key-value caching is sufficient
- Want simpler, more focused caching solution
- Multi-threaded performance is critical

### Real-World Examples

- **Twitter**: Uses Redis for user timeline cache, trending topics
- **Instagram**: Uses Redis for user session data, feed cache
- **Stack Overflow**: Uses Redis for page cache, session state
- **GitHub**: Uses Redis for job queuing, caching API responses

---

## Decision 6: Template-Based vs Hard-Coded Notifications

### The Problem

Notification content (subject lines, message body, variables) needs to be managed. Should we hard-code notification content in application code, or should we store templates in a database and substitute variables at runtime?

### Options Considered

| Option | Pros | Cons | Flexibility | Localization | A/B Testing |
|--------|------|------|-------------|--------------|-------------|
| **Hard-Coded** | Simple, type-safe, fast, no DB lookup | Requires code deploy for changes, no non-technical editing, difficult localization | None | Very difficult | Impossible |
| **Template-Based (DB)** | Non-engineers can edit, easy A/B testing, localization support, versioning | Additional DB lookup, parsing overhead, potential injection issues | Excellent | Easy | Easy |
| **Config Files** | Middle ground, version controlled, no DB | Requires deployment, not as flexible as DB | Moderate | Moderate | Difficult |
| **External CMS** | Feature-rich editing, preview, workflow | Vendor dependency, higher cost, complex integration | Excellent | Excellent | Excellent |

### Decision Made

**Use database-stored templates with variable substitution at runtime.**

### Rationale

1. **Content Management**: Marketing and product teams can update notification copy without engineering involvement. Reduces bottlenecks and deployment friction.

2. **A/B Testing**: Can easily test different subject lines and message copy to optimize engagement rates. Create multiple template versions and randomly assign.

3. **Localization**: Support multiple languages by creating template variants for each locale. Easy to add new languages without code changes.

4. **Consistency**: Ensures consistent branding and messaging across all notifications. Templates are reviewed and approved before use.

5. **Reusability**: Same template can be used across multiple event types with different variable values.

6. **Versioning**: Can track template changes over time, rollback if needed. Useful for compliance and auditing.

7. **Dynamic Content**: Can include conditional logic in templates (e.g., show different content based on user segment).

### Implementation Details

**Template Storage:**
```sql
CREATE TABLE notification_templates (
    template_id VARCHAR(100) PRIMARY KEY,
    template_name VARCHAR(255),
    channel VARCHAR(20), -- 'email', 'sms', 'push', 'web'
    locale VARCHAR(10) DEFAULT 'en', -- 'en', 'es', 'fr', etc.
    subject VARCHAR(500), -- for email
    body_template TEXT, -- with {{variable}} placeholders
    priority VARCHAR(20) DEFAULT 'normal',
    active BOOLEAN DEFAULT TRUE,
    version INT DEFAULT 1,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE INDEX idx_templates_channel_locale ON notification_templates(channel, locale);
```

**Template Syntax:**
```
Subject: Your order {{order_id}} has been confirmed

Body:
Hi {{user_name}},

Thank you for your order! Your order {{order_id}} for {{total_amount}} 
has been confirmed and will be delivered by {{delivery_date}}.

{{#if is_premium}}
As a premium member, you get free shipping!
{{/if}}

Track your order: {{tracking_url}}
```

**Variable Substitution:**
- Use Mustache or Handlebars templating engine
- Parse template once, cache compiled version in Redis
- Substitute variables at runtime (10-20ms overhead)
- Escape user-provided content to prevent injection

**Cache Strategy:**
- Cache templates in Redis with 1-hour TTL
- Invalidate cache when template is updated
- 99%+ cache hit rate (templates rarely change)

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| Non-engineers can update content | Additional DB lookup per notification (mitigated by cache) |
| Easy A/B testing and localization | Template parsing overhead (~10ms) |
| Consistent branding and messaging | Potential security risk (template injection) |
| Version control and rollback | More complex than hard-coded strings |
| Faster iteration (no code deploy) | Need template validation and preview tools |

### When to Reconsider

**Consider Hard-Coded if:**
- Notification content never changes
- Team is all engineers, no content writers
- Performance is absolutely critical (every millisecond counts)
- Security concerns outweigh flexibility benefits
- Very simple notifications with no variables

**Consider Config Files if:**
- Want version control without database
- Deploy frequently anyway (CI/CD)
- Don't need runtime updates
- Team is comfortable with YAML/JSON

**Consider External CMS if:**
- Need rich text editing with WYSIWYG
- Complex approval workflows required
- Budget allows for commercial CMS
- Need advanced features (scheduling, segmentation)

### Real-World Examples

- **Twilio SendGrid**: Template engine for transactional emails
- **Mailchimp**: Rich template editor with merge tags
- **Amazon SES**: Template management via API
- **Intercom**: Message templates with liquid syntax

---

## Decision 7: Push Model vs Pull Model

### The Problem

How should notifications be delivered to end users? Should the server push notifications to clients (Push Model), or should clients periodically query the server for new notifications (Pull Model)?

### Options Considered

| Option | Pros | Cons | Latency | Server Load | Client Battery |
|--------|------|------|---------|-------------|----------------|
| **Push Model** | Real-time delivery, efficient, low latency, battery-friendly | Complex connection management, requires persistent connections | <100ms | Moderate (connection management) | Excellent |
| **Pull Model (Polling)** | Simple implementation, stateless, works everywhere | High latency, wasted requests, high server load, battery drain | 15-60s | Very high | Poor |
| **Hybrid** | Best of both worlds, graceful degradation | Most complex, requires both systems | Variable | Moderate | Good |

### Decision Made

**Use Push Model for real-time channels (WebSocket, Mobile Push) with Pull Model as fallback.**

### Rationale

1. **Latency**: Push model delivers notifications in <100ms. Poll model with 30-second intervals has 15-second average latency.

2. **Efficiency**: Push model uses persistent connection. Poll model makes 2,880 requests/day/user even when there are no notifications.

3. **Server Resources**: Poll model requires handling 11,574 requests/second just for checking if there are new notifications. Push model only sends when there are notifications.

4. **Battery Life**: Mobile polling drains battery significantly. Push notifications use OS-optimized delivery channels (APNs, FCM).

5. **User Experience**: Real-time push provides better UX. Users see notifications immediately when events occur.

6. **Cost**: Poll model requires more server resources to handle constant polling requests. Push model only uses resources when delivering actual notifications.

### Implementation Details

**Push Channels:**
```
1. WebSocket (Web/Mobile Web):
   - Persistent connection
   - Server pushes notifications in real-time
   - Heartbeat to detect disconnections
   - Automatic reconnection

2. Mobile Push (iOS/Android):
   - FCM (Firebase Cloud Messaging) for Android
   - APNS (Apple Push Notification Service) for iOS
   - OS handles connection management
   - Delivery even when app is backgrounded

3. Email/SMS:
   - Inherently push-based
   - Third-party providers deliver to users
```

**Pull Fallback:**
```
When push is unavailable:
1. WebSocket blocked by firewall → Poll every 30 seconds
2. User logged out → Poll on app open
3. Push notifications disabled → Poll periodically
4. Connection lost → Poll while reconnecting
```

**Hybrid Flow:**
```
1. Primary: Use WebSocket/Push for real-time delivery
2. Fallback: If disconnected >5 minutes, client polls REST API
3. Sync: On reconnect, fetch missed notifications via REST
4. Recovery: Client sends last_notification_id to fetch gaps
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| <100ms real-time delivery | Complex connection management (WebSocket) |
| Efficient (only send when needed) | Persistent connections (memory usage) |
| Battery-friendly | Stateful architecture (harder to scale) |
| Better user experience | Need fallback mechanism |
| Lower server load | Requires push notification setup (FCM/APNS) |

### When to Reconsider

**Consider Pull Model if:**
- Notification volume is very low (<1 per hour per user)
- Latency requirement is relaxed (>1 minute acceptable)
- Cannot implement persistent connections
- All users behind firewalls that block WebSockets
- Stateless architecture is required

**Consider Hybrid if:**
- Need graceful degradation
- Some users on restricted networks
- Want best UX but need reliability
- Can invest in maintaining both systems

### Real-World Examples

- **WhatsApp**: Push model (WebSocket for web, native push for mobile)
- **Slack**: Hybrid (WebSocket for active users, polling for backgrounded)
- **Twitter**: Push model for real-time updates
- **Email**: Inherently push-based (SMTP)

---

## Decision 8: Dedicated Workers vs Serverless (AWS Lambda)

### The Problem

Should we deploy channel workers as dedicated long-running services (EC2, Kubernetes pods), or should we use serverless functions (AWS Lambda, Azure Functions) triggered by Kafka/SQS events?

### Options Considered

| Option | Pros | Cons | Cost | Scalability | Cold Start |
|--------|------|------|------|-------------|------------|
| **Dedicated Workers** | Predictable performance, no cold starts, full control, supports Kafka | Always running (fixed cost), manual scaling, infrastructure management | Fixed (~$1k/month per channel) | Manual (add instances) | None |
| **AWS Lambda** | Auto-scaling, pay-per-invocation, no infrastructure management, handles bursts | Cold starts (1-3s), 15-min timeout, limited to SQS/EventBridge (not Kafka directly), vendor lock-in | Variable (can be cheaper at low volume) | Automatic | 1-3s |
| **Kubernetes** | Container orchestration, auto-scaling, portable, supports Kafka | Complex setup, requires K8s expertise, more operational overhead | Moderate (~$2k/month with auto-scaling) | Automatic (HPA) | None |
| **Hybrid** | Use Lambda for bursts, dedicated for baseline | Most complex, requires managing both | Optimized | Excellent | Partial |

### Decision Made

**Use dedicated worker services (EC2 or Kubernetes pods) for channel workers.**

### Rationale

1. **Cold Starts**: Lambda has 1-3 second cold start latency. For real-time notifications (<2s SLA), this is unacceptable. Dedicated workers are always warm.

2. **Kafka Integration**: Workers need to consume from Kafka with consumer groups. Lambda doesn't natively support Kafka (requires Kafka connector or MSK trigger which adds complexity).

3. **Persistent Connections**: Workers maintain connections to third-party APIs (SendGrid, Twilio). Lambda's ephemeral nature means reconnecting on every invocation.

4. **Cost Predictability**: At 1 billion notifications/day, dedicated workers ($1k/month) are cheaper than Lambda ($5k/month at $0.20 per 1M requests).

5. **Circuit Breaker State**: Circuit breakers maintain state across requests. Lambda's stateless nature makes this difficult.

6. **Throughput**: Dedicated workers can batch process messages more efficiently. Lambda processes one event at a time (or small batches).

7. **WebSocket Servers**: Web Worker manages persistent WebSocket connections. Impossible with Lambda's request-response model.

### Implementation Details

**Deployment Options:**

**Option A: EC2 Auto Scaling Groups**
```
- Email Worker: 3-10 EC2 instances (t3.medium)
- SMS Worker: 2-5 instances
- Push Worker: 3-10 instances
- Web Worker: 2-5 instances
- Cost: ~$1,000/month per channel at baseline
- Auto-scaling based on Kafka consumer lag
```

**Option B: Kubernetes (EKS)**
```
- Deploy workers as K8s Deployments
- Horizontal Pod Autoscaler (HPA) based on:
  - Kafka consumer lag metric
  - CPU/memory usage
- ConfigMaps for configuration
- Secrets for API keys
- Cost: ~$2,000/month total (includes EKS control plane)
```

**Auto-Scaling Strategy:**
```
Metric: Kafka Consumer Lag
- If lag > 10,000 messages: Scale up by 2 instances
- If lag < 1,000 messages: Scale down by 1 instance
- Min instances: 2 (high availability)
- Max instances: 20 (cost control)
- Cooldown: 5 minutes (prevent flapping)
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| No cold starts, predictable latency | Always-on cost (even during low traffic) |
| Native Kafka support | Manual infrastructure management |
| Persistent connections to third-parties | Need to handle scaling manually |
| Stateful circuit breakers | More operational complexity than serverless |
| More cost-effective at high scale | Less elastic than Lambda |

### When to Reconsider

**Consider AWS Lambda if:**
- Notification volume is low and bursty (<10M per day)
- Can tolerate 1-3 second cold start latency
- Don't need Kafka (can use SQS instead)
- Want zero infrastructure management
- Pay-per-use pricing is more cost-effective at your scale

**Consider Kubernetes if:**
- Already using K8s for other services
- Need portability across cloud providers
- Have K8s expertise on team
- Want automatic scaling with more control than EC2 ASG

**Hybrid Approach:**
- Use Lambda for non-critical notifications (marketing emails, digests)
- Use dedicated workers for real-time notifications (push, web)
- Best of both worlds: cost optimization + performance

### Real-World Examples

- **Uber**: Dedicated services for critical notifications
- **Netflix**: Mix of dedicated services and Lambda for different use cases
- **Airbnb**: Kubernetes for all microservices including workers
- **Twitter**: Dedicated services for real-time delivery

---

## Decision 9: Token Bucket vs Fixed Window Rate Limiting

### The Problem

Workers need to rate limit calls to third-party APIs to stay within provider limits (e.g., Twilio: 100 SMS/second). What rate limiting algorithm should we use?

### Options Considered

| Option | Pros | Cons | Burst Handling | Accuracy | Complexity |
|--------|------|------|----------------|----------|------------|
| **Token Bucket** | Allows bursts, smooth rate limiting, industry standard | Slightly more complex | Excellent (can accumulate tokens) | Excellent | Medium |
| **Fixed Window** | Very simple, easy to understand, low overhead | Burst at window boundaries, unfair | Poor (2x limit at boundaries) | Moderate | Low |
| **Sliding Window** | More accurate than fixed window, fair distribution | More complex, higher memory usage | Good | Excellent | High |
| **Leaky Bucket** | Perfectly smooth rate, simple concept | No burst tolerance, may drop requests | None (strict rate) | Excellent | Medium |

### Decision Made

**Use Token Bucket algorithm for rate limiting third-party API calls.**

### Rationale

1. **Burst Handling**: Token Bucket allows short bursts up to bucket capacity. If no requests for 10 seconds, can send 10 immediately. Useful for bursty notification traffic.

2. **Smooth Rate Limiting**: Tokens refill at constant rate. Over time, average rate converges to configured limit. Prevents thundering herd at window boundaries.

3. **Industry Standard**: Used by AWS (API Gateway), Google Cloud (Cloud Endpoints), and most rate limiting libraries.

4. **Flexible**: Can configure both burst size (bucket capacity) and sustained rate (refill rate). Twilio: 100 tokens, refill 100/second.

5. **Fair**: Requests are only delayed when out of tokens, not rejected. Fixed window can reject requests even if overall rate is acceptable.

6. **Simple Implementation**: Easier than Sliding Window, more flexible than Leaky Bucket.

### Implementation Details

**Token Bucket Algorithm:**
```
class TokenBucket:
    capacity: int       # Maximum tokens (burst size)
    tokens: float       # Current tokens available
    refill_rate: float  # Tokens per second
    last_refill: timestamp

    function consume(count=1):
        // Refill tokens based on time elapsed
        now = current_time()
        elapsed = now - last_refill
        tokens += elapsed * refill_rate
        tokens = min(tokens, capacity)  // Cap at capacity
        last_refill = now
        
        // Try to consume tokens
        if tokens >= count:
            tokens -= count
            return True  // Request allowed
        else:
            return False  // Request blocked (wait)
```

**Configuration Examples:**
```
SendGrid (Email):
- Capacity: 1000 tokens
- Refill Rate: 16.67 tokens/second (1000/minute)
- Allows burst of 1000 emails, then 16.67/second sustained

Twilio (SMS):
- Capacity: 100 tokens
- Refill Rate: 100 tokens/second
- Allows burst of 100 SMS, then 100/second sustained

FCM (Push):
- Capacity: 3000 tokens
- Refill Rate: 3000 tokens/second
- High throughput with large burst capacity
```

**Storage:**
- Store token bucket state in Redis for distributed rate limiting
- Key: `ratelimit:email:bucket`
- Value: `{tokens: 50.3, last_refill: 1634567890.123}`
- TTL: 1 hour (recreate if expired)

**Atomic Operations:**
- Use Redis Lua script for atomic consume operation
- Prevents race conditions with multiple workers
- Single round-trip to Redis

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| Allows traffic bursts (up to capacity) | Slightly more complex than fixed window |
| Smooth rate limiting (no boundary issues) | Requires storing state (tokens, timestamp) |
| Flexible configuration (burst + sustained rate) | Need atomic operations in distributed setup |
| Industry-standard algorithm | Higher memory usage than simple counter |
| Fair distribution over time | Small computational overhead (refill calculation) |

### When to Reconsider

**Consider Fixed Window if:**
- Simplicity is more important than accuracy
- Traffic is perfectly smooth (no bursts)
- Can tolerate 2x rate spikes at window boundaries
- Very low latency required (simpler calculation)

**Consider Sliding Window if:**
- Need most accurate rate limiting
- Cannot tolerate any boundary issues
- Have resources for higher complexity
- Memory usage is not a concern

**Consider Leaky Bucket if:**
- Need perfectly smooth rate (no bursts allowed)
- Want simple queuing semantics
- Can buffer requests (not applicable for notifications)

### Real-World Examples

- **AWS API Gateway**: Token Bucket for API rate limiting
- **Google Cloud Endpoints**: Token Bucket for quota management
- **Stripe API**: Token Bucket for rate limiting
- **NGINX**: Token Bucket (limit_req module)

---

## Decision 10: Circuit Breaker Pattern vs No Circuit Breaker

### The Problem

When a third-party provider (SendGrid, Twilio, FCM) has an outage, should workers continue trying to call the failing API, or should we implement a Circuit Breaker to fail fast?

### Options Considered

| Option | Pros | Cons | Failure Detection | Recovery | Complexity |
|--------|------|------|-------------------|----------|------------|
| **No Circuit Breaker** | Simple, keeps trying, no false positives | Wastes resources on failing calls, cascading failures, slow error response | Manual (logs/alerts) | Manual | None |
| **Circuit Breaker** | Fast failure, automatic recovery testing, prevents cascading failures | False positives possible, adds complexity, requires tuning | Automatic (threshold-based) | Automatic | Medium |
| **Simple Retry with Backoff** | Simple, handles transient failures | No protection against sustained outages, keeps hammering failing service | None | None | Low |
| **Bulkhead Pattern** | Resource isolation, prevents total failure | Doesn't stop failed calls, more complex | Partial | Partial | High |

### Decision Made

**Implement Circuit Breaker pattern for all third-party API calls.**

### Rationale

1. **Fast Failure**: When SendGrid is down, circuit breaker opens after 5 failures. Subsequent requests fail immediately (<1ms) instead of waiting 5 seconds for timeout.

2. **Prevent Cascading Failures**: Without circuit breaker, failing API calls can exhaust thread pools, block Kafka consumers, and cause workers to crash. Circuit breaker isolates failures.

3. **Automatic Recovery**: Circuit breaker periodically tests if provider has recovered (every 60 seconds). When recovered, automatically resumes normal operation. No manual intervention needed.

4. **Resource Protection**: Don't waste worker resources making calls to a service that's clearly down. Use those resources for other channels that are working.

5. **Better User Experience**: Fail fast and move to Dead Letter Queue rather than making users wait for long timeouts.

6. **Observability**: Circuit breaker state (OPEN/CLOSED/HALF-OPEN) provides clear signal for monitoring. Alert when circuit opens.

### Implementation Details

**Circuit Breaker States:**
```
CLOSED (Normal):
- All requests pass through
- Track success/failure rate
- If failures >= threshold: Transition to OPEN

OPEN (Failing):
- All requests fail immediately (no API call)
- After timeout (60s): Transition to HALF-OPEN

HALF-OPEN (Testing):
- Allow single test request
- If success: Transition to CLOSED (recovered)
- If failure: Transition back to OPEN (still down)
```

**Configuration:**
```
Email Worker (SendGrid):
- Failure Threshold: 5 consecutive failures
- Timeout: 60 seconds
- Success Threshold: 3 consecutive successes (to fully recover)

SMS Worker (Twilio):
- Failure Threshold: 5 consecutive failures
- Timeout: 60 seconds
- Success Threshold: 3 consecutive successes

Push Worker (FCM):
- Failure Threshold: 5 consecutive failures
- Timeout: 60 seconds
- Success Threshold: 3 consecutive successes
```

**Storage:**
- Store circuit state in Redis for distributed workers
- Key: `circuit:email:state`
- Value: `{state: 'OPEN', failures: 5, last_failure: timestamp}`
- All workers share same circuit state

**Alerting:**
```
Alert: Circuit Opened
- Severity: HIGH
- Channel: PagerDuty
- Message: "Email circuit breaker opened - SendGrid down"
- Runbook: Check SendGrid status page, verify API keys

Alert: Circuit Closed
- Severity: INFO
- Channel: Slack
- Message: "Email circuit breaker closed - SendGrid recovered"
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| Fast failure (<1ms vs 5s timeout) | False positives possible (temporary blip triggers circuit) |
| Automatic recovery testing | Configuration complexity (thresholds, timeouts) |
| Prevent cascading failures | Additional state to manage (Redis) |
| Resource protection | Some requests may fail during HALF-OPEN testing |
| Clear observability (circuit state) | Need monitoring and alerting for circuit state |

### When to Reconsider

**Consider No Circuit Breaker if:**
- Third-party services are extremely reliable (99.99%+ uptime)
- Notification volume is very low (<100 per day)
- Can tolerate slow error responses (5+ second timeouts)
- Don't have resources to implement circuit breaker
- Team lacks experience with circuit breaker pattern

**Consider Simple Retry if:**
- Most failures are transient (network blips)
- Services recover quickly (<10 seconds)
- Don't need fast failure
- Want minimal complexity

**Consider Bulkhead Pattern (in addition) if:**
- Want to isolate resources per channel
- Prevent one channel from consuming all threads
- Need more granular isolation
- Have complex microservices architecture

### Real-World Examples

- **Netflix Hystrix**: Circuit breaker library (now in maintenance mode)
- **Resilience4j**: Modern circuit breaker for Java
- **Go-Resilience**: Circuit breaker for Go
- **AWS Lambda**: Built-in circuit breaker for event sources

---

## Summary: Design Decisions Comparison

This table summarizes all 10 major design decisions for the Notification Service:

| Decision | What We Chose | Primary Benefit | Main Trade-off | When to Reconsider |
|----------|---------------|-----------------|----------------|-------------------|
| **1. Communication** | Kafka (Async) | Guaranteed delivery, handles 57k QPS peaks | ~100ms added latency | Low volume (<1k QPS) |
| **2. Logging Database** | Cassandra | 100k+ writes/second, cost-effective | No complex joins | Low volume (<10M/day) |
| **3. Real-Time Channel** | WebSockets | <100ms latency, battery-friendly | Complex connection management | Latency >30s acceptable |
| **4. Worker Architecture** | Multi-Channel Workers | Independent scaling, failure isolation | 4 microservices vs 1 | Very low volume |
| **5. Caching** | Redis | <1ms lookup, 95% DB load reduction | Stale data (5 min TTL) | Real-time updates critical |
| **6. Content Management** | Template-Based (DB) | Non-engineers can edit, A/B testing | Template parsing overhead | Content never changes |
| **7. Delivery Model** | Push (WebSocket/FCM/APNS) | Real-time, efficient, low latency | Complex connection management | Latency >1 min acceptable |
| **8. Worker Deployment** | Dedicated Services (EC2/K8s) | No cold starts, Kafka support | Always-on cost | Low/bursty volume |
| **9. Rate Limiting** | Token Bucket | Allows bursts, smooth rate limiting | Higher complexity vs fixed window | Perfectly smooth traffic |
| **10. Failure Handling** | Circuit Breaker | Fast failure, automatic recovery | Configuration complexity | Services extremely reliable |

### Cost Analysis

**Monthly Infrastructure Costs (at 1B notifications/day):**

| Component | Choice | Monthly Cost | Alternative | Alternative Cost | Savings |
|-----------|--------|--------------|-------------|------------------|---------|
| Message Queue | Kafka (self-hosted) | $5,000 | AWS SQS | $8,000 | $3,000 |
| Logging Database | Cassandra (10 nodes) | $3,000 | PostgreSQL (large instance) | $10,000 | $7,000 |
| Cache | Redis Cluster | $2,500 | No cache (large PostgreSQL) | $5,000 | $2,500 |
| Workers | EC2 Dedicated (20 instances) | $4,000 | AWS Lambda | $6,000 | $2,000 |
| WebSocket Servers | EC2 (100 instances) | $2,000 | Long Polling (more servers) | $4,000 | $2,000 |
| **Total** | **Our Architecture** | **$16,500** | **Alternative** | **$33,000** | **$16,500 (50% savings)** |

### Performance Characteristics

| Metric | Target | Achieved | How |
|--------|--------|----------|-----|
| **API Response Time** | <100ms | 60ms | Redis cache, async Kafka |
| **Email Delivery** | <5s | 3-5s | SendGrid SMTP |
| **SMS Delivery** | <3s | 1-3s | Twilio API |
| **Push Delivery** | <2s | 1-2s | FCM/APNS batch API |
| **Web Delivery** | <500ms | <100ms | WebSockets |
| **Throughput (Peak)** | 57k QPS | 100k+ QPS | Kafka + horizontal scaling |
| **Availability** | 99.9% | 99.95% | Multi-region, circuit breakers |

### Scalability Roadmap

**Current Scale (100M DAU, 1B notifications/day):**
- Kafka: 3 brokers, 36 partitions
- Cassandra: 10 nodes
- Redis: 3 shards
- Workers: 20 instances total
- WebSocket: 100 servers (10M connections)

**10x Scale (1B DAU, 10B notifications/day):**
- Kafka: 9 brokers, 108 partitions (+3x)
- Cassandra: 30 nodes (+3x)
- Redis: 9 shards (+3x)
- Workers: 60 instances (+3x)
- WebSocket: 1,000 servers (100M connections) (+10x)
- **Cost:** ~$50k/month (+3x from current)

**Key Scaling Strategies:**
1. Horizontal scaling for all components
2. Regional deployment (US, EU, APAC)
3. Cassandra partitioning optimization
4. Kafka partition rebalancing
5. Worker auto-scaling based on consumer lag

---

## Conclusion

These 10 design decisions form the foundation of a scalable, reliable notification service capable of handling 1 billion notifications per day across multiple channels. The key themes are:

1. **Asynchronous Processing**: Kafka decouples services and handles traffic bursts
2. **Specialized Components**: Cassandra for writes, Redis for reads, Kafka for messaging
3. **Real-Time Delivery**: WebSockets and push notifications for low latency
4. **Failure Resilience**: Circuit breakers, DLQs, and automatic recovery
5. **Cost Optimization**: Right-sized components save 50% vs alternatives
6. **Operational Excellence**: Monitoring, alerting, and observability built-in

**Next Steps:**
- Review **[hld-diagram.md](./hld-diagram.md)** for visual architecture
- Study **[sequence-diagrams.md](./sequence-diagrams.md)** for interaction flows
- Examine **[pseudocode.md](./pseudocode.md)** for implementation details
- Read main **[3.2.2-design-notification-service.md](./3.2.2-design-notification-service.md)** for complete guide

