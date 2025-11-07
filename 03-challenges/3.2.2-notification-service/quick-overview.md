# Notification Service - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A scalable, reliable notification system that delivers real-time notifications to users across multiple channels (Email,
SMS, Push Notifications, and In-App notifications). The system handles hundreds of millions of notifications daily,
guarantees delivery even during third-party service outages, and provides tracking for delivery status.

**Key Characteristics:**

- **Multi-channel delivery** - Email, SMS, Push, Web notifications
- **High volume** - 1 billion notifications/day, peak loads of 57k+ QPS
- **Guaranteed delivery** - No notifications lost even if third-party services are down
- **Real-time requirements** - Push and in-app notifications arrive within 2 seconds
- **Third-party dependencies** - Reliance on external providers (Twilio, SendGrid, FCM, APNS)
- **Event-driven architecture** - Kafka message queues decouple notification generation from delivery

---

## Requirements & Scale

### Functional Requirements

1. **Multi-Channel Delivery**: Support notifications via:
    - Email (transactional emails, newsletters)
    - SMS (one-time passwords, urgent alerts)
    - Push Notification (mobile apps via FCM/APNS)
    - Web Notifications (in-app bell icon, browser push)

2. **Real-Time Delivery**: Notifications must be delivered near-instantly when triggered by user actions

3. **User Preferences**: Respect user notification preferences (opt-in/opt-out per channel, frequency limits)

4. **Delivery Status Tracking**: Log and track delivery status for auditing (Sent, Delivered, Failed, Opened, Clicked)

5. **Rate Limiting**: Prevent notification fatigue by limiting frequency per user

6. **Template Management**: Support customizable notification templates with variable substitution

7. **Priority Levels**: Support different priority levels (critical, high, normal, low)

### Scale Estimation

| Metric                     | Assumption                                | Calculation                     | Result                                  |
|----------------------------|-------------------------------------------|---------------------------------|-----------------------------------------|
| **Total Users**            | 100 Million DAU                           | -                               | -                                       |
| **Daily Notifications**    | 1 Billion events                          | 1B / 86400 s/day                | ~11,574 QPS (average)                   |
| **Peak Throughput**        | Peak hours are 5x average                 | 11,574 × 5                      | ~57,870 QPS peak                        |
| **Channel Distribution**   | Email: 40%, Push: 35%, SMS: 15%, Web: 10% | -                               | ~400M emails, 350M push, 150M SMS daily |
| **Database Writes (Logs)** | Every notification generates a log entry  | 1B writes/day × 500 bytes/entry | ~500 GB/day, ~183 TB/year               |
| **Notification Payload**   | Average payload size                      | ~1 KB per notification          | ~1 TB/day data transfer                 |
| **WebSocket Connections**  | 10% of DAU online simultaneously          | 10M concurrent connections      | Need connection pooling                 |

**Key Observations:**

- **Write-Heavy Workload:** 1 billion writes per day for logging requires horizontally scalable database
- **Burst Handling:** Must handle 5x average load during peak events
- **Multi-Region:** 100M global users require geo-distributed deployment

---

## Key Components

| Component                 | Responsibility                                           | Technology Options                  |
|---------------------------|----------------------------------------------------------|-------------------------------------|
| **API Gateway**           | Request validation, rate limiting, authentication        | Kong, AWS API Gateway               |
| **Notification Service**  | Core orchestration (preferences, deduplication, routing) | Go, Java, Kotlin (stateless)        |
| **Kafka Message Stream**  | Central buffer ensuring no notification is lost          | Apache Kafka                        |
| **Email Worker**          | Consume from Kafka, call SendGrid/AWS SES API            | Go, Java (consumer groups)          |
| **SMS Worker**            | Consume from Kafka, call Twilio/AWS SNS API              | Go, Java (consumer groups)          |
| **Push Worker**           | Consume from Kafka, call FCM/APNS APIs                   | Go, Java (consumer groups)          |
| **Web Worker**            | Consume from Kafka, push to WebSocket Server             | Go, Java (consumer groups)          |
| **WebSocket Server**      | Maintains persistent connections for real-time delivery  | Node.js, Go (connection management) |
| **User Preferences DB**   | Stores user notification preferences                     | PostgreSQL (with Redis cache)       |
| **Notification Logs DB**  | High-volume logs for notification history                | Cassandra (write-heavy)             |
| **Third-Party Providers** | External services for delivery                           | SendGrid, Twilio, FCM, APNS         |

---

## Data Model

### User Preferences (PostgreSQL)

Stores user notification preferences and settings:

- `users` table: User basic info (email, phone)
- `user_notification_preferences` table: Channel preferences, frequency limits, quiet hours
- `notification_subscriptions` table: Category-based subscriptions (orders, social, marketing)

**Caching Strategy:** Cache user preferences in Redis (TTL: 5 minutes) to reduce DB load.

### Notification Templates (PostgreSQL)

Stores customizable notification templates with variable substitution:

- `notification_templates` table: Template ID, channel, subject, body template, priority
- Templates support `{{variables}}` for dynamic content

**Caching Strategy:** Cache templates in Redis (TTL: 1 hour).

### Notification Logs (Cassandra)

High-volume append-only logs for notification history and delivery status:

- `notification_logs` table: Partitioned by `user_id`, clustered by `created_at`
- `notification_status` table: Indexed by `notification_id` for status lookup

**Why Cassandra?**

- Extremely high write throughput (1 billion writes/day)
- Time-series data with natural partitioning
- Horizontal scalability without complex sharding
- Trade-off: Limited ad-hoc query capabilities (no joins)

### Device Tokens (Redis + PostgreSQL)

For push notifications, store device tokens:

- `device_tokens` table: User ID, device token, platform (iOS/Android/Web), active status
- **Caching Strategy:** Cache active device tokens in Redis for fast lookup

---

## Architecture Flow

### Write Path: Notification Creation

1. **Event Source** (e.g., Order Service) triggers notification via REST API
2. **API Gateway** validates request, checks rate limits, authenticates
3. **Notification Service**:
    - Check idempotency key in Redis (prevent duplicates)
    - Fetch user preferences from Redis (cache miss → PostgreSQL)
    - Fetch template from cache (cache miss → PostgreSQL)
    - Substitute variables in template
    - Determine channels based on user preferences
    - Publish to Kafka topics (email, push, web, sms)
    - Return 202 Accepted
4. **Kafka** persists messages across partitions
5. **Channel Workers** consume messages and deliver via third-party APIs
6. **Workers** log delivery status to Cassandra

**Total Latency:** ~200 ms for API response, ~2 seconds for actual delivery

### Read Path: Real-Time Delivery

1. **WebSocket Server** maintains persistent connections with browsers/mobile apps
2. **Web Worker** consumes from `web-notifications` Kafka topic
3. **Worker** pushes notification to WebSocket connection
4. **Client** receives notification in real-time (< 100 ms)

**Fallback:** If WebSocket connection lost, notification stored in database, user fetches on next app open via REST API.

---

## Key Design Decisions

### Why Kafka over Synchronous REST APIs?

**Rationale:**

1. **Traffic Bursts:** Must handle peak loads of 57k QPS during major events
2. **Guaranteed Delivery:** Kafka persists messages, ensuring no notification is lost
3. **Retry Mechanism:** Workers can retry failed deliveries without losing messages
4. **Decoupling:** Notification Service doesn't need to wait for channel workers
5. **Scalability:** Add more worker instances without changing producer code

**Trade-offs:**

- Adds ~100 ms latency (acceptable for most notifications)
- Increased system complexity (requires Kafka cluster management)
- Eventual consistency (notifications may arrive slightly out of order)

### Why Cassandra over PostgreSQL for Logs?

**Rationale:**

1. **Write-Heavy Workload:** 1 billion writes per day overwhelms traditional RDBMS
2. **Time-Series Data:** Natural partitioning by `user_id` and `created_at`
3. **Horizontal Scalability:** Easily add nodes to handle increased load
4. **Append-Only:** No updates needed, perfect for Cassandra's strengths
5. **Cost:** More cost-effective at scale than vertical scaling PostgreSQL

**Trade-offs:**

- No complex joins (must denormalize data)
- Eventual consistency (logs may take a few seconds to appear)
- Limited ad-hoc queries (need predefined query patterns)

### Why WebSockets over Long Polling?

**Rationale:**

1. **Latency:** WebSockets provide <100 ms latency vs 1-5 seconds for long polling
2. **Efficiency:** No repeated HTTP handshakes
3. **Battery Life:** More efficient on mobile devices
4. **Bi-directional:** Can receive acknowledgments from clients
5. **Industry Standard:** Used by Slack, WhatsApp, Facebook for real-time features

**Trade-offs:**

- More complex connection management (sticky sessions, connection tracking)
- Requires dedicated WebSocket servers
- Fallback mechanism needed for older browsers

### Why Multi-Channel Workers over Single Worker?

**Rationale:**

1. **Independent Scaling:** Scale Email workers independently of SMS workers
2. **Failure Isolation:** Email failures don't affect Push notifications
3. **Rate Limiting:** Each channel has different rate limits from providers
4. **Specialized Logic:** Each channel has unique requirements
5. **Team Ownership:** Different teams can own different channels

**Trade-offs:**

- More microservices to manage (increased operational complexity)
- Code duplication for common logic (mitigated by shared libraries)
- More Kafka topics to maintain

---

## Failure Handling

### Scenario 1: Third-Party Provider Down (e.g., SendGrid outage)

- Email Worker cannot reach SendGrid API
- Circuit Breaker opens after 5 consecutive failures
- Messages accumulate in Kafka (up to retention period, e.g., 7 days)
- When SendGrid recovers, Circuit Breaker closes
- Workers resume processing from last committed offset
- **Result:** No messages lost, delivery delayed

### Scenario 2: Device Token Expired (Push Notification)

- Push Worker receives error from FCM: "Invalid device token"
- Worker marks token as inactive in database
- Message moved to DLQ for manual investigation
- User receives notification via fallback channel (email or web)
- **Result:** Graceful degradation, user still notified

### Scenario 3: WebSocket Connection Lost

- WebSocket server detects disconnection via heartbeat
- Server removes connection from Redis registry
- Next notification for that user triggers fallback mechanism:
    - Store notification in database
    - User fetches on next app open via REST API
- **Result:** Notification stored for later retrieval

---

## Bottlenecks & Solutions

### Bottleneck 1: Third-Party Rate Limits

**Problem:** External providers (Twilio, SendGrid) impose strict rate limits that can block delivery.

**Solutions:**

1. **Token Bucket Rate Limiter:** Implement local rate limiter in each worker to stay below provider limits
2. **Circuit Breaker:** Prevent overwhelming third-party APIs during outages
3. **Multiple Providers:** Use multiple SMS providers (Twilio + AWS SNS) with failover
4. **Batch APIs:** Use batch APIs where available (FCM supports batch sends)

**Example:** Twilio allows 100 requests/second. With 5 SMS workers, each worker limits to 20 requests/second.

### Bottleneck 2: WebSocket Connection Management

**Problem:** 10M concurrent connections require significant server resources.

**Solutions:**

1. **Connection Pooling:** Use connection pools to reuse WebSocket connections
2. **Horizontal Scaling:** 10M connections → 1,000 servers with 10k connections each
3. **Sticky Sessions:** Load balancer routes same user to same server (hash by user_id)
4. **Redis Pub/Sub:** Cross-server communication for notifications

### Bottleneck 3: Kafka Throughput

**Problem:** At peak (viral event), Kafka might receive 57k messages/sec.

**Solutions:**

1. **Partition Kafka Topics:** Use `user_id` as partition key → spread load across brokers
2. **Batch Processing:** Workers batch notifications to reduce API calls
3. **Scale Kafka Cluster:** Add more brokers as traffic grows
4. **Monitor Consumer Lag:** Alert when consumer lag exceeds threshold

### Bottleneck 4: Database Write Load

**Problem:** 1 billion writes/day for notification logs.

**Solutions:**

1. **Cassandra for Logs:** Optimized for write-heavy workloads
2. **Batch Writes:** Batch log entries before writing to Cassandra
3. **Sharding:** Partition by `user_id` across hundreds of nodes
4. **TTL on Logs:** Auto-delete logs older than 90 days

---

## Common Anti-Patterns to Avoid

### 1. Synchronous Notification Delivery

❌ **Bad:** Calling third-party APIs synchronously from application code

- User waits for email to send before getting order confirmation
- If SendGrid is down, order creation fails
- No retry mechanism for failed deliveries

✅ **Good:** Async notification with message queue

- Order Service → Kafka → Response (200ms)
- Email Worker → SendGrid (async)

### 2. No Idempotency

❌ **Bad:** Duplicate events result in duplicate notifications to users

✅ **Good:** Use idempotency keys

- Store idempotency key in Redis with TTL
- Return success for duplicate requests without re-sending

### 3. Storing Notifications in RDBMS

❌ **Bad:** Using PostgreSQL for high-volume notification logs

- PostgreSQL struggles with > 10k writes/second
- Vertical scaling is expensive

✅ **Good:** Use Cassandra or similar NoSQL database optimized for write-heavy workloads

### 4. Ignoring Rate Limits

❌ **Bad:** Workers send requests to third-party APIs without rate limiting

- Twilio blocks API key after exceeding limit
- All notifications fail until limit resets

✅ **Good:** Implement Token Bucket rate limiter

- Stay below provider limits with buffer (e.g., use 80% of limit)

### 5. No Circuit Breaker

❌ **Bad:** When third-party API is down, workers continuously retry

- Wastes resources on failing requests
- Delays processing of other notifications

✅ **Good:** Implement Circuit Breaker pattern

- Open circuit after N consecutive failures
- Periodically test with single request to detect recovery

### 6. Single Worker for All Channels

❌ **Bad:** One worker handles Email, SMS, Push, and Web notifications

- Cannot scale channels independently
- Failures in one channel affect all channels

✅ **Good:** Separate workers per channel with dedicated Kafka topics

---

## Monitoring & Observability

### Key Metrics

| Metric                         | Target        | Alert Threshold | Why Important          |
|--------------------------------|---------------|-----------------|------------------------|
| **Notification Delivery Rate** | Baseline ±20% | > 2x baseline   | Detect unusual traffic |
| **Delivery Latency (p99)**     | < 2 seconds   | > 10 seconds    | User experience        |
| **Delivery Success Rate**      | > 99%         | < 95%           | Service reliability    |
| **Kafka Consumer Lag**         | < 1 minute    | > 5 minutes     | Delivery delay         |
| **Circuit Breaker Open Rate**  | < 1%          | > 5%            | Third-party issues     |
| **DLQ Message Count**          | Minimal       | > 1000          | Failed deliveries      |
| **WebSocket Connection Count** | Baseline ±10% | > 2x baseline   | Connection issues      |
| **Error Rate**                 | < 0.1%        | > 1%            | Service health         |

---

## Trade-offs Summary

| Decision               | Choice                | Alternative   | Why Chosen                               | Trade-off                        |
|------------------------|-----------------------|---------------|------------------------------------------|----------------------------------|
| **Message Queue**      | Kafka                 | RabbitMQ, SQS | High throughput, message replay          | Increased complexity             |
| **Logs Database**      | Cassandra             | PostgreSQL    | Write-heavy workload, horizontal scaling | Limited query capabilities       |
| **Real-Time Delivery** | WebSockets            | Long Polling  | Low latency, efficiency                  | Connection management complexity |
| **Architecture**       | Multi-Channel Workers | Single Worker | Independent scaling, failure isolation   | More microservices to manage     |
| **Caching**            | Redis                 | Database-only | Sub-1ms lookups, reduced DB load         | Stale data for up to 5 minutes   |
| **Templates**          | Database-based        | Code-based    | Content management, A/B testing          | Additional DB lookup overhead    |

---

## Key Takeaways

1. **Kafka for async messaging** enables guaranteed delivery and handles traffic bursts
2. **Cassandra for logs** handles 1 billion writes/day with horizontal scaling
3. **WebSockets for real-time** provides <100 ms latency for in-app notifications
4. **Multi-channel workers** enable independent scaling and failure isolation
5. **Circuit breakers and rate limiters** protect against third-party failures
6. **Idempotency keys** prevent duplicate notifications
7. **Dead Letter Queues** handle failed deliveries gracefully
8. **Redis caching** reduces database load and improves latency
9. **Template-based notifications** enable content management without code changes
10. **Comprehensive monitoring** detects and resolves issues quickly

---

## Recommended Stack

- **Notification Service:** Go or Java (stateless, horizontal scaling)
- **Message Queue:** Apache Kafka (high throughput, message replay)
- **Logs Database:** Cassandra (write-heavy, horizontal scaling)
- **User Preferences DB:** PostgreSQL (ACID, with Redis cache)
- **Cache:** Redis Cluster (consistent hashing, high availability)
- **WebSocket Server:** Node.js or Go (connection management)
- **Email Provider:** SendGrid or AWS SES
- **SMS Provider:** Twilio or AWS SNS
- **Push Provider:** FCM (Android) and APNS (iOS)
- **Monitoring:** Prometheus + Grafana (metrics, dashboards)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, component design, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (notification creation, delivery,
  failure scenarios)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (rate limiting, circuit breaker, deduplication)