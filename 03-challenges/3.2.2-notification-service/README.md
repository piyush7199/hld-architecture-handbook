# 3.2.2 Design a Notification Service

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow, scaling
  strategies
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows, failure scenarios, retry mechanisms
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## 1. Problem Statement

Design a scalable, reliable notification system that can deliver real-time notifications to users across multiple
channels (Email, SMS, Push Notifications, and In-App notifications). The system must handle hundreds of millions of
notifications daily, guarantee delivery even during third-party service outages, and provide tracking for delivery
status.

**Key Challenges:**

- **Multi-Channel Complexity**: Each channel (Email, SMS, Push, Web) has different delivery mechanisms, rate limits, and
  third-party providers
- **High Volume & Bursts**: Must handle peak loads of 57k+ QPS during major events (e.g., product launches, breaking
  news)
- **Guaranteed Delivery**: Cannot lose notifications even if downstream services are temporarily unavailable
- **Real-Time Requirements**: Push and in-app notifications must arrive within 2 seconds
- **Third-Party Dependencies**: Reliance on external providers (Twilio, SendGrid, FCM, APNS) with their own rate limits
  and availability

---

## 2. Requirements and Scale Estimation

### 2.1 Functional Requirements (FRs)

1. **Multi-Channel Delivery**: Support notifications via:
    - $\text{Email}$ (transactional emails, newsletters)
    - $\text{SMS}$ (one-time passwords, urgent alerts)
    - $\text{Push}$ $\text{Notification}$ (mobile apps via FCM/APNS)
    - $\text{Web}$ $\text{Notifications}$ (in-app bell icon, browser push)

2. **Real-Time Delivery**: Notifications must be delivered near-instantly when triggered by user actions (e.g., friend
   request, order confirmation)

3. **User Preferences**: Respect user notification preferences (opt-in/opt-out per channel, frequency limits)

4. **Delivery Status Tracking**: Log and track delivery status for auditing:
    - Sent, Delivered, Failed, Opened, Clicked

5. **Rate Limiting**: Prevent notification fatigue by limiting frequency per user

6. **Template Management**: Support customizable notification templates with variable substitution

7. **Priority Levels**: Support different priority levels (critical, high, normal, low)

### 2.2 Non-Functional Requirements (NFRs)

1. **Resiliency**: The system must not lose any notification, even if third-party
   providers ($\text{APNS}$, $\text{FCM}$, $\text{SendGrid}$) are temporarily down

2. **Scalability**: Must handle hundreds of millions of notifications per day, with high variance in traffic

3. **Low Latency (Real-Time)**: Web/Push notifications must arrive in $<2$ seconds

4. **High Availability**: $99.9\%$ uptime for notification ingestion (can tolerate eventual delivery for non-critical
   notifications)

5. **Idempotency**: Duplicate events should not result in duplicate notifications to users

6. **Observability**: Comprehensive logging and monitoring of delivery rates, failures, and latencies

### 2.3 Scale Estimation

| Metric                     | Assumption                                | Calculation                              | Result                                               |
|----------------------------|-------------------------------------------|------------------------------------------|------------------------------------------------------|
| **Total Users**            | $100$ Million $\text{DAU}$                | -                                        | -                                                    |
| **Daily Notifications**    | $1$ Billion $\text{events}$               | $\frac{1 \text{B}}{86400 \text{ s/day}}$ | $\sim 11,574$ QPS (average)                          |
| **Peak Throughput**        | Peak hours are $5$x $\text{average}$      | $11,574 \times 5$                        | $\sim 57,870$ $\text{QPS}$ $\text{peak}$             |
| **Channel Distribution**   | Email: 40%, Push: 35%, SMS: 15%, Web: 10% | -                                        | $\sim 400$ M emails, $350$ M push, $150$ M SMS daily |
| **Database Writes (Logs)** | Every notification generates a log entry  | $1$ B writes/day × $500$ bytes/entry     | $\sim 500$ GB/day, $\sim 183$ TB/year                |
| **Notification Payload**   | Average payload size                      | $\sim 1$ KB per notification             | $\sim 1$ TB/day data transfer                        |
| **WebSocket Connections**  | $10\%$ of DAU online simultaneously       | $10$ M concurrent connections            | Need connection pooling                              |

**Key Observations:**

- **Write-Heavy Workload**: $1$ billion writes per day for logging requires horizontally scalable database
- **Burst Handling**: Must handle $5$x average load during peak events
- **Multi-Region**: $100$ M global users require geo-distributed deployment

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

The system uses an event-driven architecture with message queues to decouple notification generation from delivery,
ensuring resilience and scalability.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              EVENT SOURCES                                       │
│  (Order Service, Social Service, Payment Service, Marketing Service, etc.)      │
└────────────┬────────────────────────────────────────────────────────────────────┘
             │ REST API / gRPC
             │ POST /notifications
             ▼
┌────────────────────────────┐
│     API Gateway            │ ◄────── Authentication (JWT)
│  - Rate Limiting           │         Authorization
│  - Request Validation      │         Load Balancing
│  - Auth/Authz              │
└────────────┬───────────────┘
             │
             ▼
┌────────────────────────────┐
│  Notification Service      │
│  - Fetch User Preferences  │◄─────── Redis Cache (User Preferences)
│  - Deduplicate Events      │         PostgreSQL (User Settings)
│  - Enrich with Template    │
│  - Route to Channels       │
└────────────┬───────────────┘
             │
             │ Publish Event
             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Kafka Message Stream                             │
│  Topics: email-notifications, sms-notifications,                     │
│          push-notifications, web-notifications                        │
└────┬──────────┬──────────┬──────────┬─────────────────────────────────┘
     │          │          │          │
     ▼          ▼          ▼          ▼
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────────┐
│  Email  │ │   SMS   │ │  Push   │ │   Web/WSS    │
│ Worker  │ │ Worker  │ │ Worker  │ │   Worker     │
│         │ │         │ │         │ │              │
│ (Pool)  │ │ (Pool)  │ │ (Pool)  │ │  (Pool)      │
└────┬────┘ └────┬────┘ └────┬────┘ └───────┬──────┘
     │          │          │              │
     ▼          ▼          ▼              ▼
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────────────┐
│SendGrid │ │ Twilio  │ │FCM/APNS │ │  WebSocket     │
│  (SMTP) │ │  (SMS)  │ │ (Push)  │ │   Server       │
└─────────┘ └─────────┘ └─────────┘ └────────┬───────┘
                                              │
     ┌────────────────────────────────────────┘
     │
     ▼ Persistent Connection
┌──────────────────┐
│  Browser/Mobile  │
│   (End Users)    │
└──────────────────┘

                 │
                 ▼ Log Delivery Status
      ┌──────────────────────────┐
      │   Cassandra (Logs DB)    │
      │  - Notification History   │
      │  - Delivery Status        │
      │  - User Engagement        │
      └──────────────────────────┘

      ┌──────────────────────────┐
      │    Monitoring Stack       │
      │  - Prometheus (Metrics)   │
      │  - Grafana (Dashboards)   │
      │  - ELK (Log Aggregation)  │
      └──────────────────────────┘
```

### 3.1 Component Overview

1. **Event Sources**: Microservices that trigger notifications (Order Service, Social Service, etc.)
2. **API Gateway**: Entry point for notification requests with rate limiting and authentication
3. **Notification Service**: Core orchestration service that fetches preferences, deduplicates, and routes to channels
4. **Kafka Message Stream**: Central buffer ensuring no notification is lost, enabling retry and replay
5. **Channel Workers**: Dedicated microservices for each channel (Email, SMS, Push, Web)
6. **Third-Party Providers**: External services (SendGrid, Twilio, FCM, APNS)
7. **WebSocket Server**: Maintains persistent connections for real-time in-app notifications
8. **Cassandra Logs DB**: High-throughput database for notification history and status tracking

---

## 4. Data Model

### 4.1 User Preferences (PostgreSQL)

Stores user notification preferences and settings.

```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    email VARCHAR(255),
    phone_number VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_notification_preferences (
    user_id BIGINT PRIMARY KEY,
    email_enabled BOOLEAN DEFAULT TRUE,
    sms_enabled BOOLEAN DEFAULT TRUE,
    push_enabled BOOLEAN DEFAULT TRUE,
    web_enabled BOOLEAN DEFAULT TRUE,
    frequency_limit INT DEFAULT 50, -- max notifications per day
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE notification_subscriptions (
    subscription_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    category VARCHAR(50), -- e.g., 'orders', 'social', 'marketing'
    channel VARCHAR(20), -- 'email', 'sms', 'push', 'web'
    enabled BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE INDEX idx_user_prefs ON user_notification_preferences(user_id);
CREATE INDEX idx_subscriptions_user ON notification_subscriptions(user_id);
```

### 4.2 Notification Templates (PostgreSQL)

```sql
CREATE TABLE notification_templates (
    template_id VARCHAR(100) PRIMARY KEY,
    template_name VARCHAR(255),
    channel VARCHAR(20), -- 'email', 'sms', 'push', 'web'
    subject VARCHAR(500), -- for email
    body_template TEXT, -- with {{variables}}
    priority VARCHAR(20) DEFAULT 'normal', -- 'critical', 'high', 'normal', 'low'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_templates_channel ON notification_templates(channel);
```

### 4.3 Notification Logs (Cassandra)

High-volume append-only logs for notification history and delivery status.

```sql
CREATE TABLE notification_logs (
    notification_id UUID,
    user_id BIGINT,
    channel TEXT, -- 'email', 'sms', 'push', 'web'
    template_id TEXT,
    status TEXT, -- 'sent', 'delivered', 'failed', 'opened', 'clicked'
    created_at TIMESTAMP,
    delivered_at TIMESTAMP,
    error_message TEXT,
    metadata MAP<TEXT, TEXT>, -- additional context
    PRIMARY KEY ((user_id), created_at, notification_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Query by notification_id for status lookup
CREATE TABLE notification_status (
    notification_id UUID PRIMARY KEY,
    user_id BIGINT,
    channel TEXT,
    status TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);
```

**Why Cassandra?**

- Extremely high write throughput ($1$ billion writes/day)
- Time-series data with natural partitioning by `user_id` and `created_at`
- Horizontal scalability without complex sharding logic
- Trade-off: Limited ad-hoc query capabilities (no joins)

### 4.4 Device Tokens (Redis + PostgreSQL)

For push notifications, store device tokens.

```sql
CREATE TABLE device_tokens (
    token_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    device_token VARCHAR(500), -- FCM/APNS token
    platform VARCHAR(20), -- 'ios', 'android', 'web'
    device_id VARCHAR(255),
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE INDEX idx_device_tokens_user ON device_tokens(user_id);
CREATE INDEX idx_device_tokens_token ON device_tokens(device_token);
```

**Caching Strategy**: Cache active device tokens in Redis for fast lookup.

---

## 5. Component Design

### 5.1 API Gateway

**Responsibilities:**

- Request validation and rate limiting (protect against notification spam)
- Authentication (JWT) and authorization
- Load balancing across Notification Service instances

**Rate Limiting**: Token Bucket algorithm per API key/user to prevent abuse

- *See pseudocode.md::TokenBucket class for implementation*

### 5.2 Notification Service

**Core orchestration service** that processes incoming notification requests.

**Responsibilities:**

1. **Fetch User Preferences**: Query Redis cache (fallback to PostgreSQL)
2. **Deduplicate Events**: Use idempotency key to prevent duplicate notifications
3. **Enrich with Template**: Fetch template and substitute variables
4. **Route to Channels**: Determine which channels to use based on user preferences
5. **Publish to Kafka**: Send notification events to appropriate Kafka topics

**Key Operations:**

- *See pseudocode.md::NotificationService.processNotification() for implementation*
- *See pseudocode.md::DeduplicationService.isDuplicate() for deduplication logic*

**Caching Layer (Redis)**:

- Cache user preferences (TTL: $5$ minutes)
- Cache notification templates (TTL: $1$ hour)
- Cache device tokens (TTL: $10$ minutes)

### 5.3 Kafka Message Stream

**Why Kafka over Synchronous REST?**

The system must handle huge bursts of traffic ($\sim 57$k QPS peak) and guarantee delivery even if downstream services
fail. Kafka provides:

- **Buffering**: Absorbs traffic spikes without overwhelming workers
- **Guaranteed Delivery**: Messages are persisted and can be replayed
- **Retry Mechanism**: Workers can retry failed deliveries
- **Scalability**: Partitions allow horizontal scaling of consumers

**Trade-off**: Adds a small delay ($\sim 100$ ms) to the notification process, but this is acceptable for non-critical
notifications.

**Topic Structure:**

```
- email-notifications (12 partitions)
- sms-notifications (6 partitions)
- push-notifications (12 partitions)
- web-notifications (6 partitions)
- dlq-email (dead letter queue)
- dlq-sms
- dlq-push
- dlq-web
```

**Partitioning Strategy**: Partition by `user_id` to ensure ordering per user.

### 5.4 Channel Workers

Each channel has dedicated worker microservices that consume from Kafka and deliver notifications.

#### 5.4.1 Email Worker

**Responsibilities:**

- Consume from `email-notifications` topic
- Call SendGrid/AWS SES API
- Handle rate limits from email provider
- Log delivery status to Cassandra

**Circuit Breaker**: If SendGrid API fails repeatedly, open circuit and route to DLQ

- *See pseudocode.md::CircuitBreaker class for implementation*

**Retry Strategy**: Exponential backoff with max 3 retries

- *See pseudocode.md::EmailWorker.sendEmail() for implementation*

#### 5.4.2 SMS Worker

**Responsibilities:**

- Consume from `sms-notifications` topic
- Call Twilio/AWS SNS API
- Handle strict rate limits (Twilio: $\sim 100$ requests/second)
- Log delivery status

**Token Bucket Rate Limiter**: Prevent overwhelming Twilio API

- *See pseudocode.md::RateLimiter class for implementation*

#### 5.4.3 Push Notification Worker

**Responsibilities:**

- Consume from `push-notifications` topic
- Call FCM (Android) or APNS (iOS) APIs
- Handle device token expiration and cleanup
- Batch notifications to reduce API calls

**Batch Processing**: Group notifications by device to reduce API calls

- *See pseudocode.md::PushWorker.batchSend() for implementation*

**Device Token Cleanup**: Remove invalid tokens from database

#### 5.4.4 Web/WebSocket Worker

**Responsibilities:**

- Consume from `web-notifications` topic
- Maintain persistent WebSocket connections with browsers/mobile apps
- Push notifications to connected clients in real-time
- Fall back to polling for disconnected clients

**Connection Management**: Use Redis to track which WebSocket server holds each user's connection

- *See pseudocode.md::WebSocketManager for implementation*

### 5.5 WebSocket Server

**Real-Time Delivery** for in-app notifications and browser push.

**Why WebSockets over Long Polling?**

- True bi-directional real-time communication
- Lower latency ($<100$ ms vs $1$-$5$ seconds for polling)
- Minimal overhead (no repeated HTTP handshakes)
- Battery efficient on mobile devices

**Trade-off**: More complex connection management, requires sticky sessions

**Architecture:**

- Load balancer with sticky sessions (hash by user_id)
- Redis pub/sub for cross-server communication
- Heartbeat mechanism to detect disconnections

**Scaling**: $10$ M concurrent connections → $1,000$ servers with $10$k connections each

### 5.6 Dead Letter Queue (DLQ)

**Why DLQ?**

If a Push Worker repeatedly fails to deliver to an endpoint (e.g., FCM token is expired), the message is moved to a DLQ
for:

- Manual investigation
- Delayed batch retries
- Device token cleanup

**Trade-off**: Requires dedicated monitoring system for the DLQ.

**DLQ Topics:**

- `dlq-email`: Failed email deliveries
- `dlq-sms`: Failed SMS deliveries
- `dlq-push`: Failed push notifications
- `dlq-web`: Failed web notifications

**Alerting**: Trigger alerts if DLQ message count exceeds threshold.

---

## 6. Why This Over That? (Key Design Decisions)

### 6.1 Why Kafka over Synchronous REST APIs?

**Decision**: Use Kafka as the central message stream between Notification Service and Channel Workers.

**Rationale:**

1. **Traffic Bursts**: Must handle peak loads of $57$k QPS during major events (e.g., product launches)
2. **Guaranteed Delivery**: Kafka persists messages, ensuring no notification is lost even if workers are down
3. **Retry Mechanism**: Workers can retry failed deliveries without losing messages
4. **Decoupling**: Notification Service doesn't need to wait for channel workers to respond
5. **Scalability**: Add more worker instances without changing producer code

**Trade-offs Accepted:**

- Adds $\sim 100$ ms latency (acceptable for most notifications)
- Increased system complexity (requires Kafka cluster management)
- Eventual consistency (notifications may arrive slightly out of order)

**When to Reconsider**: If all notifications require < $50$ ms latency (unlikely for this use case)

### 6.2 Why Cassandra over PostgreSQL for Logs?

**Decision**: Use Cassandra for notification logs instead of PostgreSQL.

**Rationale:**

1. **Write-Heavy Workload**: $1$ billion writes per day overwhelms traditional RDBMS
2. **Time-Series Data**: Natural partitioning by `user_id` and `created_at`
3. **Horizontal Scalability**: Easily add nodes to handle increased load
4. **Append-Only**: No updates needed, perfect for Cassandra's strengths
5. **Cost**: More cost-effective at scale than vertical scaling PostgreSQL

**Trade-offs Accepted:**

- No complex joins (must denormalize data)
- Eventual consistency (logs may take a few seconds to appear)
- Limited ad-hoc queries (need predefined query patterns)

**When to Reconsider**: If complex analytics queries are required (consider adding data warehouse for analytics)

### 6.3 Why WebSockets over Long Polling?

**Decision**: Use WebSockets for real-time web notifications.

**Rationale:**

1. **Latency**: WebSockets provide $<100$ ms latency vs $1$-$5$ seconds for long polling
2. **Efficiency**: No repeated HTTP handshakes
3. **Battery Life**: More efficient on mobile devices
4. **Bi-directional**: Can receive acknowledgments from clients
5. **Industry Standard**: Used by Slack, WhatsApp, Facebook for real-time features

**Trade-offs Accepted:**

- More complex connection management (sticky sessions, connection tracking)
- Requires dedicated WebSocket servers
- Fallback mechanism needed for older browsers

**When to Reconsider**: If clients are mostly behind corporate firewalls that block WebSockets (rare today)

### 6.4 Why Multi-Channel Workers over Single Worker?

**Decision**: Separate worker microservices for each channel (Email, SMS, Push, Web).

**Rationale:**

1. **Independent Scaling**: Scale Email workers independently of SMS workers based on load
2. **Failure Isolation**: Email failures don't affect Push notifications
3. **Rate Limiting**: Each channel has different rate limits from providers
4. **Specialized Logic**: Each channel has unique requirements (e.g., device token cleanup for Push)
5. **Team Ownership**: Different teams can own different channels

**Trade-offs Accepted:**

- More microservices to manage (increased operational complexity)
- Code duplication for common logic (mitigated by shared libraries)
- More Kafka topics to maintain

**When to Reconsider**: For very small scale ($<1$ M notifications/day), a single worker with multiple handlers may
suffice

### 6.5 Why Redis Cache for User Preferences?

**Decision**: Cache user notification preferences in Redis.

**Rationale:**

1. **Reduced DB Load**: User preferences are read on every notification (high read QPS)
2. **Latency**: Redis provides $<1$ ms lookup vs $10$-$50$ ms for PostgreSQL
3. **Scalability**: Redis cluster can handle millions of reads per second
4. **Consistency**: Preferences don't change frequently, TTL of $5$ minutes is acceptable
5. **Cost**: Reduces load on PostgreSQL, allowing smaller DB instances

**Trade-offs Accepted:**

- Stale data for up to $5$ minutes (user changes preference, still receives notifications)
- Additional system component (Redis cluster to manage)
- Cache invalidation complexity (need to invalidate on preference update)

**When to Reconsider**: If real-time preference updates are critical (reduce TTL or use cache invalidation)

### 6.6 Why Template-Based Notifications?

**Decision**: Store notification templates in database and substitute variables at runtime.

**Rationale:**

1. **Content Management**: Non-engineers can update templates without code changes
2. **A/B Testing**: Easily test different notification copy
3. **Localization**: Support multiple languages with template variants
4. **Consistency**: Ensure consistent branding and messaging
5. **Reusability**: Same template can be used across multiple event types

**Trade-offs Accepted:**

- Additional database lookup for each notification
- Template parsing overhead ($\sim 10$ ms)
- More complex notification service logic

**When to Reconsider**: For very simple systems with static notification content

---

## 7. Detailed Workflow

### 7.1 Notification Creation Flow

1. **Event Source** (e.g., Order Service) triggers notification:
   ```
   POST /api/v1/notifications
   {
     "user_id": 123456,
     "event_type": "order_confirmed",
     "template_id": "order_confirmation",
     "variables": {
       "order_id": "ORD-789",
       "total_amount": "$99.99"
     },
     "priority": "high",
     "idempotency_key": "order-789-confirmation"
   }
   ```

2. **API Gateway** validates request, checks rate limits, forwards to Notification Service

3. **Notification Service**:
    - Check idempotency key in Redis (TTL: $24$ hours)
    - If duplicate, return $200$ OK (idempotent)
    - Fetch user preferences from Redis (cache miss → PostgreSQL)
    - Fetch template from cache (cache miss → PostgreSQL)
    - Substitute variables in template
    - Determine channels based on user preferences
    - Publish to Kafka topics (email, push, web)
    - Return $202$ Accepted

4. **Kafka** persists messages across partitions

5. **Channel Workers** consume messages:
    - Email Worker → SendGrid API
    - Push Worker → FCM/APNS APIs
    - Web Worker → WebSocket Server

6. **Third-Party APIs** deliver notification to end user

7. **Workers** log delivery status to Cassandra

**Total Latency**: $\sim 200$ ms for API response, $\sim 2$ seconds for actual delivery

*See sequence-diagrams.md for detailed interaction flows*

### 7.2 Failure Handling

**Scenario 1: Third-Party Provider Down (e.g., SendGrid outage)**

- Email Worker cannot reach SendGrid API
- Circuit Breaker opens after $5$ consecutive failures
- Messages accumulate in Kafka (up to retention period, e.g., $7$ days)
- When SendGrid recovers, Circuit Breaker closes
- Workers resume processing from last committed offset
- **Result**: No messages lost, delivery delayed

**Scenario 2: Device Token Expired (Push Notification)**

- Push Worker receives error from FCM: "Invalid device token"
- Worker marks token as inactive in database
- Message moved to DLQ for manual investigation
- User receives notification via fallback channel (email or web)
- **Result**: Graceful degradation, user still notified

**Scenario 3: WebSocket Connection Lost**

- WebSocket server detects disconnection via heartbeat
- Server removes connection from Redis registry
- Next notification for that user triggers fallback mechanism:
    - Store notification in database
    - User fetches on next app open via REST API
- **Result**: Notification stored for later retrieval

*See sequence-diagrams.md for detailed failure scenarios*

---

## 8. Bottlenecks and Scaling

### 8.1 Third-Party Rate Limits

**Problem**: External providers (Twilio, SendGrid) impose strict rate limits that can block delivery.

**Solutions**:

1. **Token Bucket Rate Limiter**: Implement local rate limiter in each worker to stay below provider limits
    - *See pseudocode.md::TokenBucket class*
2. **Circuit Breaker**: Prevent overwhelming third-party APIs during outages
    - *See pseudocode.md::CircuitBreaker class*
3. **Multiple Providers**: Use multiple SMS providers (Twilio + AWS SNS) with failover
4. **Batch APIs**: Use batch APIs where available (FCM supports batch sends)

**Example**: Twilio allows $100$ requests/second. With $5$ SMS workers, each worker limits to $20$ requests/second.

### 8.2 WebSocket Connection Management

**Problem**: $10$ M concurrent connections require significant server resources.

**Solutions**:

1. **Horizontal Scaling**: Deploy $1,000$ WebSocket servers, each handling $10$k connections
2. **Connection Pooling**: Reuse connections within server
3. **Redis Pub/Sub**: Track which server holds each user's connection for efficient routing
4. **Heartbeat**: Detect and close stale connections to free resources
5. **Load Balancing**: Use sticky sessions (hash by user_id) to route reconnections to same server

**Resource Requirements**: Each connection uses $\sim 10$ KB memory → $10$k connections = $100$ MB per server

### 8.3 Kafka Partition Rebalancing

**Problem**: Adding new worker instances triggers partition rebalancing, causing temporary slowdown.

**Solutions**:

1. **Over-Partition**: Use more partitions than workers (e.g., $12$ partitions for $6$ workers) to minimize rebalance
   impact
2. **Incremental Rebalancing**: Add workers gradually, not all at once
3. **Cooperative Rebalancing**: Use Kafka's cooperative rebalancing protocol (Kafka 2.4+)
4. **Monitor Lag**: Alert if consumer lag exceeds threshold during rebalance

### 8.4 Cassandra Write Bottleneck

**Problem**: $1$ billion writes per day may overwhelm Cassandra cluster.

**Solutions**:

1. **Horizontal Scaling**: Add more Cassandra nodes (easy with Cassandra's architecture)
2. **Batching**: Batch writes from workers (insert $100$ logs at once)
3. **Async Writes**: Workers don't wait for Cassandra write confirmation
4. **Compression**: Enable compression to reduce storage and I/O
5. **TTL**: Set Time-To-Live on logs (e.g., delete logs older than $1$ year)

**Capacity Planning**: $10$-node Cassandra cluster can handle $\sim 100$k writes/second

### 8.5 Geo-Partitioning

**Problem**: Global system needs to comply with data residency laws (e.g., GDPR in EU).

**Future Scaling**:

1. **Regional Kafka Clusters**: Deploy Kafka clusters in each region (US, EU, APAC)
2. **Regional Cassandra Clusters**: Store logs in user's region
3. **Regional Channel Workers**: Deploy workers in each region to reduce latency
4. **Cross-Region Replication**: Replicate critical data (templates, preferences) across regions for redundancy

**Partitioning Strategy**: Partition by `country_code` or `region` to ensure data residency compliance.

---

## 9. Common Anti-Patterns

### ❌ Anti-Pattern 1: Synchronous Notification Delivery

**Problem**: Calling third-party APIs synchronously from application code:

```
POST /orders → Order Service → SendGrid API (5 seconds) → Response to user
```

**Why it's bad**:

- User waits for email to send before getting order confirmation
- If SendGrid is down, order creation fails
- No retry mechanism for failed deliveries
- Cannot handle traffic spikes

✅ **Best Practice**: Async notification with message queue:

```
POST /orders → Order Service → Kafka → Response (200ms)
                ↓
            Email Worker → SendGrid (async)
```

### ❌ Anti-Pattern 2: No Idempotency

**Problem**: Duplicate events result in duplicate notifications to users.

**Example**: User clicks "Submit Order" twice due to slow response → Receives 2 order confirmation emails.

✅ **Best Practice**: Use idempotency keys:

- *See pseudocode.md::DeduplicationService.isDuplicate()*
- Store idempotency key in Redis with TTL
- Return success for duplicate requests without re-sending

### ❌ Anti-Pattern 3: Storing Notifications in RDBMS

**Problem**: Using PostgreSQL for high-volume notification logs:

- PostgreSQL struggles with $> 10$k writes/second
- Vertical scaling is expensive
- Requires complex sharding for horizontal scaling

✅ **Best Practice**: Use Cassandra or similar NoSQL database optimized for write-heavy workloads.

### ❌ Anti-Pattern 4: Ignoring Rate Limits

**Problem**: Workers send requests to third-party APIs without rate limiting:

- Twilio blocks API key after exceeding limit
- All notifications fail until limit resets
- Poor user experience

✅ **Best Practice**: Implement Token Bucket rate limiter:

- *See pseudocode.md::TokenBucket class*
- Stay below provider limits with buffer (e.g., use $80\%$ of limit)

### ❌ Anti-Pattern 5: No Circuit Breaker

**Problem**: When third-party API is down, workers continuously retry:

- Wastes resources on failing requests
- Delays processing of other notifications
- Increases load on already struggling provider

✅ **Best Practice**: Implement Circuit Breaker pattern:

- *See pseudocode.md::CircuitBreaker class*
- Open circuit after N consecutive failures
- Periodically test with single request to detect recovery

### ❌ Anti-Pattern 6: Single Worker for All Channels

**Problem**: One worker handles Email, SMS, Push, and Web notifications:

- Cannot scale channels independently
- Failures in one channel affect all channels
- Rate limiting becomes complex

✅ **Best Practice**: Separate workers per channel with dedicated Kafka topics.

---

## 10. Alternative Approaches

### 10.1 Alternative 1: Serverless Architecture (AWS Lambda)

**Approach**: Replace workers with AWS Lambda functions triggered by Kafka or SQS.

**Pros**:

- **Auto-Scaling**: Automatically scales to handle traffic spikes
- **No Server Management**: Fully managed by AWS
- **Cost**: Pay per invocation (can be cheaper at low volumes)

**Cons**:

- **Cold Starts**: $\sim 1$-$3$ second latency for cold starts (problematic for real-time)
- **Timeouts**: Lambda has $15$ minute max execution time
- **Vendor Lock-In**: Hard to migrate away from AWS
- **Connection Limits**: Hard to maintain persistent WebSocket connections

**When to Use**: Suitable for small to medium scale ($< 10$ M notifications/day) with bursty traffic.

### 10.2 Alternative 2: Pull-Based Architecture

**Approach**: Instead of pushing notifications, clients poll the server for new notifications.

**Pros**:

- **Simpler**: No WebSocket connection management
- **Firewall-Friendly**: Works in restrictive corporate networks
- **Stateless**: No need to track connections

**Cons**:

- **Higher Latency**: Polling interval (e.g., every $30$ seconds) delays notifications
- **Wasted Resources**: Clients poll even when there are no notifications
- **Battery Drain**: Frequent polling drains mobile battery

**When to Use**: Suitable for non-critical notifications where latency > $30$ seconds is acceptable.

### 10.3 Alternative 3: Single Database for Everything

**Approach**: Use PostgreSQL for user preferences, templates, AND notification logs.

**Pros**:

- **Simplicity**: One database to manage
- **ACID Guarantees**: Strong consistency for all data
- **Powerful Queries**: Can run complex analytics queries

**Cons**:

- **Scalability**: PostgreSQL struggles with $1$ billion writes/day
- **Cost**: Expensive to vertically scale
- **Single Point of Failure**: Database outage affects entire system

**When to Use**: Suitable for small scale ($< 1$ M notifications/day) where simplicity is prioritized over scalability.

### 10.4 Alternative 4: Using RabbitMQ Instead of Kafka

**Approach**: Use RabbitMQ as the message broker instead of Kafka.

**Comparison**:

| Aspect               | Kafka                            | RabbitMQ                                |
|----------------------|----------------------------------|-----------------------------------------|
| **Throughput**       | Very High ($> 1$ M msg/s)        | Moderate ($\sim 20$k msg/s)             |
| **Message Ordering** | Guaranteed per partition         | Guaranteed per queue                    |
| **Message Replay**   | Yes (retain logs)                | No (messages deleted after consumption) |
| **Latency**          | $\sim 10$ ms                     | $\sim 5$ ms                             |
| **Complexity**       | Higher                           | Lower                                   |
| **Use Case**         | High-throughput, event streaming | Traditional message queue               |

**When to Use RabbitMQ**: For smaller scale or when you need advanced routing (topic exchanges, fanout).

**Why We Chose Kafka**: Higher throughput, message replay for debugging, and log retention for auditing.

---

## 11. Monitoring and Observability

### 11.1 Key Metrics

**Business Metrics**:

- **Notification Volume**: Total notifications sent per day/hour (by channel)
- **Delivery Rate**: Percentage of successfully delivered notifications
- **Delivery Latency**: P50, P95, P99 latency from event trigger to delivery
- **User Engagement**: Open rate, click rate for emails and push notifications

**System Metrics**:

- **API Gateway**: Request rate, error rate, latency
- **Notification Service**: Processing rate, cache hit rate, error rate
- **Kafka**: Consumer lag, partition count, throughput
- **Workers**: Messages processed per second, error rate, retry rate
- **Cassandra**: Write throughput, read latency, disk usage
- **WebSocket**: Active connections, disconnection rate, message latency

**Third-Party Metrics**:

- **SendGrid**: Email delivery rate, bounce rate, spam rate
- **Twilio**: SMS delivery rate, cost per SMS
- **FCM/APNS**: Push delivery rate, invalid token rate

### 11.2 Alerting Rules

**Critical Alerts** (PagerDuty):

- Consumer lag > $10$ minutes on any Kafka topic
- Notification Service error rate > $5\%$
- Any worker error rate > $10\%$
- Cassandra write latency > $100$ ms
- DLQ message count > $10,000$

**Warning Alerts** (Slack):

- Cache hit rate < $80\%$
- Third-party API latency > $2$ seconds
- WebSocket disconnection rate > $10\%$
- Disk usage > $70\%$ on any node

### 11.3 Dashboards

**Grafana Dashboards**:

1. **Overview Dashboard**: Total notifications, delivery rate, latency
2. **Channel Dashboard**: Metrics per channel (Email, SMS, Push, Web)
3. **Kafka Dashboard**: Consumer lag, throughput, partition distribution
4. **Worker Dashboard**: Processing rate, error rate, retry rate per worker
5. **Third-Party Dashboard**: API latency, error rate for SendGrid, Twilio, FCM/APNS

### 11.4 Distributed Tracing

Use **Jaeger** or **Zipkin** to trace notification flow:

- Trace ID: Unique ID for each notification
- Spans: API Gateway → Notification Service → Kafka → Worker → Third-Party API
- Visualize end-to-end latency and identify bottlenecks

### 11.5 Log Aggregation

Use **ELK Stack** (Elasticsearch, Logstash, Kibana) for centralized logging:

- Collect logs from all services
- Query logs by `notification_id`, `user_id`, `channel`
- Set up alerts for specific error patterns

---

## 12. Trade-offs Summary

| Decision                     | What We Gain                                       | What We Sacrifice                                 |
|------------------------------|----------------------------------------------------|---------------------------------------------------|
| **Kafka for Async**          | Guaranteed delivery, scalability, retry capability | $\sim 100$ ms added latency, increased complexity |
| **Cassandra for Logs**       | High write throughput, horizontal scalability      | No complex joins, eventual consistency            |
| **WebSockets for Real-Time** | Low latency ($<100$ ms), efficient                 | Complex connection management, sticky sessions    |
| **Multi-Channel Workers**    | Independent scaling, failure isolation             | More microservices, operational overhead          |
| **Redis Cache**              | Low latency ($<1$ ms), reduced DB load             | Stale data (up to $5$ min), cache management      |
| **Template-Based**           | Flexibility, localization, A/B testing             | Additional DB lookups, parsing overhead           |
| **Circuit Breaker**          | Prevent cascading failures, fast recovery          | Some notifications may be delayed during recovery |
| **Idempotency Keys**         | No duplicate notifications                         | Requires client to generate unique keys           |

---

## 13. Real-World Examples

### 13.1 Slack Notifications

**Architecture**:

- WebSockets for real-time in-app notifications
- Push notifications via FCM/APNS for mobile
- Email notifications for mentions and digests
- Kafka for event streaming
- Cassandra for message history

**Scale**: $> 10$ M daily active users, $> 2$ billion messages per day

### 13.2 Uber Notifications

**Architecture**:

- Real-time driver location updates via WebSockets
- Push notifications for ride status (driver arriving, trip started)
- SMS fallback for critical notifications (payment issues)
- Multi-region deployment for low latency
- Redis for caching driver locations and user preferences

**Scale**: $> 100$ M users globally, millions of rides per day

### 13.3 Amazon Order Notifications

**Architecture**:

- Email for order confirmations and shipping updates
- Push notifications via mobile app
- SMS for delivery alerts
- SQS (similar to Kafka) for message queuing
- DynamoDB (NoSQL) for notification logs
- A/B testing for notification copy to optimize engagement

**Scale**: Hundreds of millions of orders per day

### 13.4 Facebook Notifications

**Architecture**:

- Real-time notifications via persistent connections (similar to WebSockets)
- Push notifications for mobile
- Email digests for unread notifications
- Sophisticated preference system (granular control per notification type)
- Multi-region deployment with geo-partitioning

**Scale**: $> 2$ billion users, trillions of notifications per year

---

## 14. References

### Related System Design Components

- **[2.3.1 Asynchronous Communication](../../02-components/2.3.1-asynchronous-communication.md)** - Queues vs Streams,
  Pub/Sub models
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)** - Partitions, Consumer Groups, Offset
  Management
- **[2.3.3 Advanced Message Queues](../../02-components/2.3.3-advanced-message-queues.md)** - RabbitMQ, SQS, Dead-Letter
  Queues
- **[2.0.3 Real-Time Communication](../../02-components/2.0.3-real-time-communication.md)** - WebSockets, SSE, Long
  Polling
- **[2.5.1 Rate Limiting Algorithms](../../02-components/2.5.1-rate-limiting-algorithms.md)** - Token Bucket, Leaky
  Bucket
- **[2.2.1 Caching Deep Dive](../../02-components/2.2.1-caching-deep-dive.md)** - Cache strategies, eviction policies
- **[2.1.4 Database Scaling](../../02-components/2.1.4-database-scaling.md)** - Sharding, Replication strategies

### Related Design Challenges

- **[3.2.1 Twitter Timeline](../3.2.1-twitter-timeline/)** - Similar fanout and real-time notification patterns
- **[3.1.2 Distributed Cache](../3.1.2-distributed-cache/)** - Caching strategies used in this design
- **[3.3.1 Live Chat System](../3.3.1-live-chat-system/)** - WebSocket real-time communication patterns

### External Resources

- **Kafka Documentation:** [Apache Kafka](https://kafka.apache.org/documentation/) - Message queue and streaming
  platform
- **AWS SNS/SQS Documentation:** [Amazon SNS](https://docs.aws.amazon.com/sns/) - Managed notification and queue
  services
- **FCM Documentation:** [Firebase Cloud Messaging](https://firebase.google.com/docs/cloud-messaging) - Push
  notifications for Android/iOS
- **APNS Documentation:
  ** [Apple Push Notification Service](https://developer.apple.com/documentation/usernotifications) - Push notifications
  for iOS
- **SendGrid Documentation:** [SendGrid API](https://docs.sendgrid.com/) - Email delivery service
- **Twilio Documentation:** [Twilio API](https://www.twilio.com/docs) - SMS and voice communication

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Deep dive into distributed systems, message queues, and
  event-driven architecture
- *Building Microservices* by Sam Newman - Microservices patterns and best practices
- *Kafka: The Definitive Guide* by Neha Narkhede - Comprehensive guide to Apache Kafka