# 3.2.2 Design a Notification Service

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions. 
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow, scaling strategies
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows, failure scenarios, retry mechanisms
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## Quick Overview

Design a scalable, reliable notification system that can deliver real-time notifications to users across multiple channels (Email, SMS, Push Notifications, and In-App notifications). The system must handle hundreds of millions of notifications daily, guarantee delivery even during third-party service outages, and provide tracking for delivery status.

## Key Metrics

| Metric | Value |
|--------|-------|
| **Daily Active Users** | $100$ Million |
| **Daily Notifications** | $1$ Billion |
| **Average QPS** | $\sim 11,574$ |
| **Peak QPS** | $\sim 57,870$ (5x average) |
| **Delivery Latency** | $<2$ seconds (real-time) |
| **Log Storage** | $\sim 500$ GB/day, $183$ TB/year |
| **Concurrent WebSocket Connections** | $10$ Million |

## System Overview

```
Event Sources → API Gateway → Notification Service → Kafka Topics
                                                        ↓
                              ┌─────────────────────────┼─────────────────────┐
                              ▼                         ▼                     ▼
                         Email Worker              SMS Worker           Push Worker
                              ↓                         ↓                     ↓
                         SendGrid                   Twilio               FCM/APNS
                         
                         Web Worker → WebSocket Server → Browser/Mobile App
                         
                         All Workers → Cassandra (Logs)
```

## Key Components

1. **Event Sources**: Microservices that trigger notifications
2. **API Gateway**: Authentication, rate limiting, load balancing
3. **Notification Service**: Core orchestration (preferences, routing, deduplication)
4. **Kafka**: Message stream for guaranteed delivery and buffering
5. **Channel Workers**: Dedicated workers for Email, SMS, Push, Web
6. **Third-Party Providers**: SendGrid, Twilio, FCM, APNS
7. **WebSocket Server**: Real-time in-app notifications
8. **Cassandra**: High-throughput logging database

## Core Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Communication Pattern** | Kafka (Async) | Handle 57k QPS peaks, guaranteed delivery, retry capability |
| **Logging Database** | Cassandra | 1B writes/day, horizontal scalability, time-series optimized |
| **Real-Time Channel** | WebSockets | Low latency (<100ms), efficient, bi-directional |
| **Worker Architecture** | Multi-Channel | Independent scaling, failure isolation per channel |
| **Caching Layer** | Redis | Reduce DB load, <1ms lookup for preferences |
| **Resiliency** | DLQ + Circuit Breaker | No message loss, protect against third-party failures |

## Data Model Overview

### User Preferences (PostgreSQL)
```sql
CREATE TABLE user_notification_preferences (
    user_id BIGINT PRIMARY KEY,
    email_enabled BOOLEAN DEFAULT TRUE,
    sms_enabled BOOLEAN DEFAULT TRUE,
    push_enabled BOOLEAN DEFAULT TRUE,
    web_enabled BOOLEAN DEFAULT TRUE,
    frequency_limit INT DEFAULT 50,
    quiet_hours_start TIME,
    quiet_hours_end TIME
);
```

### Notification Logs (Cassandra)
```sql
CREATE TABLE notification_logs (
    notification_id UUID,
    user_id BIGINT,
    channel TEXT,
    status TEXT, -- 'sent', 'delivered', 'failed', 'opened'
    created_at TIMESTAMP,
    delivered_at TIMESTAMP,
    PRIMARY KEY ((user_id), created_at, notification_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

## Why This Over That?

### Why Kafka over Synchronous REST?
- **Problem**: Must handle 57k QPS peaks without overwhelming workers
- **Solution**: Kafka buffers traffic, provides guaranteed delivery and retry
- **Trade-off**: Adds ~100ms latency (acceptable for most notifications)

### Why Cassandra over PostgreSQL for Logs?
- **Problem**: 1 billion writes per day overwhelms RDBMS
- **Solution**: Cassandra provides high write throughput, horizontal scaling
- **Trade-off**: No complex joins, eventual consistency

### Why WebSockets over Long Polling?
- **Problem**: Need <2 second latency for real-time notifications
- **Solution**: WebSockets provide <100ms latency with persistent connections
- **Trade-off**: More complex connection management

### Why Separate Workers per Channel?
- **Problem**: Each channel has different rate limits and failure modes
- **Solution**: Independent workers allow per-channel scaling and isolation
- **Trade-off**: More microservices to manage

## Workflow Summary

### 1. Notification Creation
```
Event Source → POST /notifications
  → API Gateway (auth + rate limit)
  → Notification Service:
      - Check idempotency (Redis)
      - Fetch user preferences (Redis/PostgreSQL)
      - Fetch template (Redis/PostgreSQL)
      - Route to channels
  → Publish to Kafka topics
  → Return 202 Accepted
```

### 2. Notification Delivery
```
Kafka Topic → Channel Worker:
  - Email Worker → SendGrid API → Email delivered
  - SMS Worker → Twilio API → SMS delivered
  - Push Worker → FCM/APNS → Push notification delivered
  - Web Worker → WebSocket Server → In-app notification delivered
  
All Workers → Log status to Cassandra
```

### 3. Failure Handling
```
Worker → Third-Party API (fails)
  → Retry with exponential backoff (3 attempts)
  → If still failing:
      - Circuit Breaker opens
      - Move to Dead Letter Queue
      - Alert monitoring system
```

*See [sequence-diagrams.md](./sequence-diagrams.md) for detailed flows*

## Key Bottlenecks and Solutions

### 1. Third-Party Rate Limits
**Problem**: SendGrid allows 10k emails/minute, Twilio limits SMS to 100/second

**Solutions**:
- Token Bucket rate limiter per worker
- Circuit Breaker to prevent overwhelming APIs
- Multiple providers with failover
- *See pseudocode.md::TokenBucket and CircuitBreaker*

### 2. WebSocket Connection Management
**Problem**: 10M concurrent connections require significant resources

**Solutions**:
- Horizontal scaling: 1,000 servers × 10k connections each
- Redis pub/sub for cross-server communication
- Sticky sessions for connection affinity
- Heartbeat mechanism to detect disconnections

### 3. Kafka Consumer Lag
**Problem**: Adding workers triggers partition rebalancing

**Solutions**:
- Over-partition: 12 partitions for 6 workers
- Incremental scaling
- Cooperative rebalancing (Kafka 2.4+)
- Monitor lag metrics

### 4. Cassandra Write Bottleneck
**Problem**: 1B writes/day may overwhelm cluster

**Solutions**:
- Horizontal scaling (add nodes)
- Batch writes (100 logs per batch)
- Async writes (don't wait for confirmation)
- TTL for old logs (delete after 1 year)

### 5. Geo-Partitioning for Compliance
**Problem**: Data residency laws (GDPR) require regional storage

**Solutions**:
- Regional Kafka clusters (US, EU, APAC)
- Regional Cassandra clusters
- Regional workers for low latency
- Cross-region replication for templates

*See [hld-diagram.md](./hld-diagram.md) for scaling architecture diagrams*

## Common Anti-Patterns

> 📚 **Note:** For detailed pseudocode implementations of anti-patterns and their solutions, see **[3.2.2-design-notification-service.md](./3.2.2-design-notification-service.md#7-common-anti-patterns)** and **[pseudocode.md](./pseudocode.md)**.

Below are high-level descriptions of common mistakes and their solutions:

### Anti-Pattern 1: Synchronous Notification Delivery

**Problem:**

❌ Blocking the user while sending notifications

```
// BAD: Synchronous delivery
function createOrder(order):
  saveOrder(order)
  sendEmailNotification(order.user_id, order.id)  // Blocks for 2-3 seconds!
  sendSMSNotification(order.user_id)              // Blocks for 1-2 seconds!
  return "Order created successfully"

User Experience: Waits 3-5 seconds for order confirmation!
```

**Solution:** ✅ Use async message queue, return immediately

```
// GOOD: Asynchronous delivery
function createOrder(order):
  saveOrder(order)
  
  // Publish to Kafka (non-blocking, <10ms)
  kafka.publish("notifications", {
    type: "ORDER_CREATED",
    user_id: order.user_id,
    order_id: order.id
  })
  
  return "Order created successfully"  // Returns in <50ms

// Separate Notification Worker
worker.consume("notifications", function(event):
  if event.type == "ORDER_CREATED":
    sendEmail(event.user_id, "Order Confirmation", event.order_id)
    sendSMS(event.user_id, "Your order is confirmed!")
)
```

*See `pseudocode.md::processNotification()` for implementation*

---

### Anti-Pattern 2: No Idempotency (Duplicate Notifications)

**Problem:** ❌ Duplicate events cause duplicate notifications

```
// Scenario: User clicks "Confirm Order" button twice
POST /orders → OrderService → Notification (email sent)
POST /orders → OrderService → Notification (email sent again!) ❌

User receives: 2 identical "Order Confirmed" emails
```

**Solution:** ✅ Use idempotency keys with Redis

```
function processNotification(event):
  idempotency_key = event.idempotency_key  // e.g., "order:12345:notification"
  
  // Check if already processed
  if redis.EXISTS(idempotency_key):
    return "ALREADY_PROCESSED"  // Skip duplicate
  
  // Process notification
  sendNotification(event)
  
  // Mark as processed (24h TTL)
  redis.SETEX(idempotency_key, 86400, "1")
  
  return "SUCCESS"
```

**Key Points:**
- Use unique idempotency key: `{resource_type}:{resource_id}:{action}`
- Set TTL to prevent Redis memory bloat (24 hours typical)
- Store in cache before calling third-party APIs

*See `pseudocode.md::checkIdempotency()` for implementation*

---

### Anti-Pattern 3: No Rate Limiting (API Throttling)

**Problem:** ❌ Overwhelming third-party APIs leads to blocked API keys

```
// Example: SendGrid rate limit is 10,000 emails/min
// Black Friday Sale: 50,000 orders in 1 minute

for order in orders:  // 50,000 iterations
  sendgrid.send_email(order.user_email, "Order Confirmed")

Result: 
- First 10,000 emails sent ✅
- Next 40,000 emails: HTTP 429 (Too Many Requests) ❌
- SendGrid blocks API key for 10 minutes!
```

**Solution:** ✅ Token Bucket rate limiter per third-party service

```
class TokenBucket:
  def __init__(self, capacity, refill_rate):
    self.capacity = capacity       // 10,000 tokens
    self.tokens = capacity
    self.refill_rate = refill_rate // 166.67 tokens/sec (10k/min)
    self.last_refill = now()
  
  def consume(self, tokens=1):
    self.refill()
    if self.tokens >= tokens:
      self.tokens -= tokens
      return true  // Allowed
    else:
      return false  // Rate limited, retry later

email_worker:
  bucket = TokenBucket(10000, 166.67)  // SendGrid limits
  
  for notification in kafka_stream:
    if bucket.consume(1):
      sendgrid.send_email(notification)
    else:
      sleep(1)  // Wait for token refill
      requeue(notification)
```

**Benefits:**
- Never exceed third-party rate limits
- Smooth out traffic bursts
- Prevent API key bans

*See `pseudocode.md::TokenBucket` for full implementation*

---

### Anti-Pattern 4: No Circuit Breaker (Cascading Failures)

> 📊 **See solution diagram:** [Circuit Breaker Flow](./sequence-diagrams.md#circuit-breaker-flow)

**Problem:** ❌ Continuous retries during third-party outage overwhelm the system

```
// Scenario: Twilio SMS service is down (100% failure rate)
// SMS Worker keeps retrying every message:

for notification in kafka_stream:
  try:
    twilio.send_sms(notification)  // Fails after 10-second timeout
  except TimeoutError:
    retry_with_backoff(notification)  // Retries 3 times (30 seconds wasted)

Result:
- Worker is stuck retrying failed SMS for 30 seconds each
- Kafka consumer lag grows: 10,000 messages behind
- Other notifications (Email, Push) are delayed
- System appears unresponsive
```

**Solution:** ✅ Circuit Breaker to fail fast during outages

```
class CircuitBreaker:
  states = [CLOSED, OPEN, HALF_OPEN]
  
  def call(self, function, *args):
    if self.state == OPEN:
      if (now() - self.last_failure_time) > self.timeout:
        self.state = HALF_OPEN  // Try one request
      else:
        raise CircuitOpenError()  // Fail fast, don't retry
    
    try:
      result = function(*args)
      self.on_success()
      return result
    except Exception as e:
      self.on_failure()
      raise e
  
  def on_failure(self):
    self.failure_count += 1
    if self.failure_count >= self.threshold:  // e.g., 5 failures
      self.state = OPEN  // Stop calling for 60 seconds
      self.last_failure_time = now()

// Usage
circuit_breaker = CircuitBreaker(threshold=5, timeout=60)

for notification in kafka_stream:
  try:
    circuit_breaker.call(twilio.send_sms, notification)
  except CircuitOpenError:
    move_to_dead_letter_queue(notification)  // Don't retry
    continue  // Process next message immediately
```

**Benefits:**
- Fail fast (< 1ms) instead of waiting for timeout (10 seconds)
- Prevent cascading failures
- Allow third-party service to recover
- Continue processing other notifications

*See `pseudocode.md::CircuitBreaker` for full implementation*

---

### Anti-Pattern 5: Single Worker for All Channels

**Problem:** ❌ Cannot scale channels independently

```
// BAD: Single worker handles all channels
worker:
  for notification in kafka_stream:
    if notification.channel == "EMAIL":
      send_email(notification)  // Takes 500ms
    elif notification.channel == "SMS":
      send_sms(notification)  // Takes 200ms
    elif notification.channel == "PUSH":
      send_push(notification)  // Takes 50ms
    elif notification.channel == "WEB":
      send_websocket(notification)  // Takes 10ms

Problem:
- Slow email delivery (500ms) blocks fast push notifications (50ms)
- Cannot scale email workers independently
- One channel failure affects all channels
```

**Solution:** ✅ Separate workers per channel

```
// GOOD: Dedicated workers per channel
Kafka Topics:
- email-notifications → Email Worker Pool (10 workers)
- sms-notifications → SMS Worker Pool (5 workers)
- push-notifications → Push Worker Pool (20 workers)
- web-notifications → WebSocket Worker Pool (50 workers)

email_worker:
  for notification in kafka.consume("email-notifications"):
    send_email(notification)

sms_worker:
  for notification in kafka.consume("sms-notifications"):
    send_sms(notification)

// Scale independently:
- Email slow? Add more email workers
- Push notifications spike? Add more push workers
- SMS fails? Doesn't affect email/push
```

**Benefits:**
- Independent scaling per channel
- Failure isolation
- Optimized worker configuration (timeouts, retries) per channel

---

### Anti-Pattern 6: Storing Sensitive Data in Logs

**Problem:** ❌ Logging full notification payload exposes PII

```
// BAD: Log full payload
logger.info("Sending notification", {
  user_id: 12345,
  email: "john.doe@example.com",  // PII ❌
  phone: "+1-555-1234",            // PII ❌
  message: "Your password reset code is: 789456"  // Sensitive ❌
})

Cassandra notification_logs:
INSERT INTO notification_logs VALUES (
  ...,
  payload: '{"email":"john.doe@example.com","phone":"+1-555-1234",...}'
)

Risk:
- GDPR violation (storing PII without consent)
- Security risk (logs accessible by many engineers)
- Compliance issues (PCI DSS for payment notifications)
```

**Solution:** ✅ Redact sensitive fields, hash user identifiers

```
// GOOD: Redact PII and sensitive data
logger.info("Sending notification", {
  user_id_hash: sha256("12345"),  // Hash user_id
  email: "j****e@example.com",    // Mask email
  phone: "+1-***-1234",            // Mask phone
  message_type: "PASSWORD_RESET",  // Don't log content
  notification_id: "notif_abc123"
})

Cassandra notification_logs:
INSERT INTO notification_logs VALUES (
  notification_id: "notif_abc123",
  user_id_hash: sha256(user_id),  // Can't reverse to actual user
  channel: "EMAIL",
  status: "SENT",
  created_at: now()
  // No sensitive payload stored
)
```

**Best Practices:**
- Hash user identifiers (user_id → sha256(user_id))
- Mask email: `john.doe@example.com` → `j****e@example.com`
- Mask phone: `+1-555-1234` → `+1-***-1234`
- Never log message content (only metadata)
- Set log retention to 30 days (GDPR compliance)

---

### Anti-Pattern 7: No Retry Logic with Exponential Backoff

**Problem:** ❌ Immediate retries overwhelm failing services

```
// BAD: Retry immediately 3 times
function send_notification(notification):
  for attempt in range(3):
    try:
      third_party_api.send(notification)
      return SUCCESS
    except TemporaryError:
      continue  // Retry immediately ❌

Result:
- If API is overloaded, 3 rapid requests make it worse
- No time for service to recover
- Wastes resources
```

**Solution:** ✅ Exponential backoff with jitter

```
// GOOD: Exponential backoff
function send_notification_with_retry(notification, max_retries=3):
  for attempt in range(max_retries):
    try:
      third_party_api.send(notification)
      return SUCCESS
    except TemporaryError as e:
      if attempt == max_retries - 1:
        raise e  // Final attempt failed
      
      // Exponential backoff: 2^attempt seconds
      delay = (2 ** attempt) + random.uniform(0, 1)  // Add jitter
      // Attempt 0: 1-2 seconds
      // Attempt 1: 2-3 seconds
      // Attempt 2: 4-5 seconds
      
      sleep(delay)

Retry Timeline:
0s: Initial attempt (fails)
1-2s: Retry 1 (fails)
3-5s: Retry 2 (fails)
7-12s: Retry 3 (final attempt)
```

**Benefits:**
- Give service time to recover
- Jitter prevents thundering herd
- Standard industry practice (AWS, Google use this)

*See `pseudocode.md::retryWithBackoff()` for implementation*

---

### Anti-Pattern 8: Hardcoded Notification Templates

**Problem:** ❌ Changing notification text requires code deployment

```
// BAD: Hardcoded template
function send_order_confirmation(order):
  message = "Hello, your order #" + order.id + " is confirmed!"
  send_email(order.user_email, "Order Confirmed", message)

Problem:
- Want to A/B test subject lines? Need code change
- Want to add user's name? Need code change
- Want to localize to Spanish? Need code change
- Marketing wants to update copy? Need engineering time
```

**Solution:** ✅ Database-stored templates with variable substitution

```
// GOOD: Template-based notifications
PostgreSQL templates table:
CREATE TABLE notification_templates (
  template_id VARCHAR PRIMARY KEY,
  channel VARCHAR,
  subject VARCHAR,
  body TEXT,
  locale VARCHAR DEFAULT 'en',
  variables JSONB
);

INSERT INTO notification_templates VALUES (
  'ORDER_CONFIRMATION',
  'EMAIL',
  'Order {{order_id}} Confirmed! 🎉',
  'Hello {{user_name}}, your order #{{order_id}} is confirmed!',
  'en',
  '["user_name", "order_id"]'
);

// Code
function send_order_confirmation(order):
  template = get_template("ORDER_CONFIRMATION", locale=order.user_locale)
  
  message = template.render({
    "user_name": order.user_name,
    "order_id": order.id
  })
  
  send_email(order.user_email, template.subject, message)

Benefits:
- Non-engineers can edit templates
- Easy A/B testing (create template variants)
- Localization (store templates per locale)
- Version control (track template changes)
```

*See `pseudocode.md::enrichTemplate()` for implementation*

---

## Alternative Approaches

### 1. Serverless (AWS Lambda)
**Pros**: Auto-scaling, no server management, pay per use  
**Cons**: Cold starts (1-3s), hard to maintain WebSocket connections  
**When to Use**: Small scale (<10M notifications/day)

### 2. Pull-Based (Polling)
**Pros**: Simpler, firewall-friendly, stateless  
**Cons**: Higher latency (30s+), wasted resources, battery drain  
**When to Use**: Non-critical notifications with >30s latency acceptable

### 3. Single Database (PostgreSQL)
**Pros**: Simplicity, ACID guarantees, powerful queries  
**Cons**: Doesn't scale to 1B writes/day, expensive  
**When to Use**: Small scale (<1M notifications/day)

### 4. RabbitMQ Instead of Kafka
**Pros**: Lower latency (~5ms), simpler setup  
**Cons**: Lower throughput (~20k msg/s), no message replay  
**When to Use**: Lower scale or need advanced routing

*See [this-over-that.md](./this-over-that.md) for detailed comparisons*

## Monitoring

### Key Metrics
- **Business**: Delivery rate, latency (P50/P95/P99), open rate
- **System**: Kafka lag, worker throughput, cache hit rate
- **Third-Party**: API latency, error rate, cost

### Critical Alerts
- Consumer lag > 10 minutes
- Worker error rate > 10%
- DLQ message count > 10,000
- Cassandra write latency > 100ms

### Dashboards
- Overview: Total notifications, delivery rate, latency
- Per-Channel: Email, SMS, Push, Web metrics
- Kafka: Consumer lag, throughput, partitions
- Third-Party: SendGrid, Twilio, FCM/APNS status

## Deep Dive: Handling Edge Cases

### Edge Case 1: Handling Webhook Delivery Failures

**Problem:** Third-party services (FCM, APNS) deliver push notifications but our tracking fails.

**Scenario:**
```
1. Push Worker sends notification to FCM → Success (FCM returns 200 OK)
2. Worker tries to log delivery status to Cassandra → Cassandra timeout ❌
3. Worker crashes before logging → Notification sent but not tracked!
```

**Solution:** Use idempotent logging with retry

```
function send_push_notification(notification):
  // 1. Send to FCM
  response = fcm.send(notification)
  
  // 2. Log delivery (with retry)
  log_delivery_with_retry(notification.id, response.status, max_retries=5)
  
function log_delivery_with_retry(notification_id, status, max_retries):
  for attempt in range(max_retries):
    try:
      cassandra.insert({
        notification_id: notification_id,
        status: status,
        timestamp: now()
      })
      return SUCCESS
    except CassandraTimeout:
      if attempt == max_retries - 1:
        // Final fallback: Log to local file for batch upload later
        append_to_local_log_file(notification_id, status)
      else:
        sleep(2 ** attempt)  // Exponential backoff
```

**Benefits:**
- Never lose tracking data
- Eventual consistency acceptable (logs can be delayed by minutes)
- Local fallback prevents complete data loss

---

### Edge Case 2: Dealing with Invalid Device Tokens

**Problem:** Mobile app device tokens expire or become invalid (user uninstalls app).

**Scenario:**
```
Push Worker → FCM.send(token="expired_token_123")
FCM Response: HTTP 400 "Invalid Registration Token"

Problem:
- Should we retry? No, token is permanently invalid
- Should we keep sending? No, wastes API calls
- Should we delete token? Yes, but how?
```

**Solution:** Automated token cleanup based on FCM/APNS feedback

```
function send_push_with_token_cleanup(notification):
  response = fcm.send(notification.device_token, notification.message)
  
  if response.status == "INVALID_TOKEN":
    // Token is permanently invalid
    delete_device_token(notification.user_id, notification.device_token)
    log_metric("push.invalid_token", 1)
    return PERMANENT_FAILURE
  
  elif response.status == "NOT_REGISTERED":
    // User uninstalled app
    delete_device_token(notification.user_id, notification.device_token)
    return PERMANENT_FAILURE
  
  elif response.status == "MESSAGE_TOO_BIG":
    // Notification payload exceeds 4KB limit
    truncate_and_retry(notification)
    return RETRY
  
  elif response.status == "SUCCESS":
    return SUCCESS

function delete_device_token(user_id, token):
  postgresql.DELETE_FROM("device_tokens", where={
    user_id: user_id,
    token: token
  })
  redis.DEL("device:token:" + token)  // Remove from cache
```

**Cleanup Strategy:**
- Immediate deletion for invalid tokens
- Batch cleanup job runs daily to remove tokens with > 30 days of failures
- Monitor "invalid_token" metric (should be < 5%)

---

### Edge Case 3: User Preference Changes Mid-Delivery

**Problem:** User opts out of notifications while fanout is in progress.

**Scenario:**
```
Timeline:
0ms: User follows celebrity with 5M followers
100ms: Celebrity posts tweet
200ms: Fanout Worker starts processing (5M notifications queued)
5s: User realizes mistake, unfollows celebrity
10s: Fanout still sending notifications to user ❌

Problem: User receives 1,000+ notifications before fanout completes
```

**Solution:** Just-in-time preference checking + rate limiting

```
// Notification Worker
function process_notification(notification):
  // 1. Just-in-time preference check (even if cached earlier)
  preferences = get_user_preferences(notification.user_id)
  
  if not preferences.enabled:
    return "USER_OPTED_OUT"  // Skip delivery
  
  // 2. Per-user rate limiting
  rate_key = "notif:rate:" + notification.user_id
  count = redis.INCR(rate_key)
  redis.EXPIRE(rate_key, 3600)  // 1-hour window
  
  if count > preferences.hourly_limit:  // e.g., 50 notifications/hour
    return "RATE_LIMITED"  // Skip delivery
  
  // 3. Check quiet hours
  if is_quiet_hours(preferences.quiet_hours_start, preferences.quiet_hours_end):
    requeue_for_later(notification)
    return "DEFERRED"
  
  // 4. Send notification
  send_notification(notification)
```

**Trade-off:**
- Adds 5ms latency (Redis cache lookup)
- Prevents notification fatigue
- Respects user preferences in real-time

*See `pseudocode.md::checkUserPreferences()` for implementation*

---

### Edge Case 4: Multi-Region Data Residency (GDPR Compliance)

**Problem:** EU users' notification logs must stay in EU (GDPR).

**Scenario:**
```
User in Germany (user_id=12345, region=EU)
Notification Service in US → Sends notification
Cassandra US Cluster → Stores log ❌ GDPR violation!

Compliance Issue:
- GDPR requires EU data to stay in EU
- Fines up to 4% of global revenue
```

**Solution:** Regional data partitioning

```
// Routing Table (PostgreSQL)
CREATE TABLE user_regions (
  user_id BIGINT PRIMARY KEY,
  region VARCHAR,  // 'US', 'EU', 'APAC'
  created_at TIMESTAMP
);

// Notification Service
function process_notification(notification):
  // 1. Lookup user region
  region = get_user_region(notification.user_id)
  
  // 2. Route to regional Kafka cluster
  kafka_cluster = get_regional_kafka(region)
  kafka_cluster.publish("notifications-" + region, notification)

// Regional Workers (deployed per region)
EU_Worker:
  kafka = connect_to_kafka("kafka-eu.example.com")
  cassandra = connect_to_cassandra("cassandra-eu.example.com")
  
  for notification in kafka.consume("notifications-EU"):
    send_notification(notification)
    cassandra.log(notification)  // Logs stay in EU ✅

// Regional Cassandra Clusters
Cassandra-US: US datacenter (Virginia)
Cassandra-EU: EU datacenter (Frankfurt) → GDPR compliant
Cassandra-APAC: Asia datacenter (Singapore)
```

**Architecture:**
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Kafka-US    │     │  Kafka-EU    │     │  Kafka-APAC  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Workers-US  │     │  Workers-EU  │     │  Workers-APAC│
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│Cassandra-US  │     │Cassandra-EU  │     │Cassandra-APAC│
└──────────────┘     └──────────────┘     └──────────────┘
```

**Benefits:**
- GDPR compliant (EU data in EU)
- Low latency (workers close to users)
- Fault isolation (EU outage doesn't affect US)

**Trade-off:**
- Higher infrastructure cost (3× clusters)
- More complex deployment
- Cross-region analytics require data aggregation

---

## Interview Discussion Points

### Question 1: How would you handle 10× scale (10 billion notifications/day)?

**Answer:**

**Current Scale:** 1B notifications/day, 11,574 QPS average, 57,870 QPS peak

**10× Scale:** 10B notifications/day, 115,740 QPS average, 578,700 QPS peak

**Challenges:**

1. **Kafka Throughput Bottleneck**
   - Current: 1 Kafka cluster, 12 partitions per topic
   - At 10×: Need 578k messages/sec
   - **Solution:**
     - Increase partitions: 12 → 120 partitions
     - Scale Kafka brokers: 3 → 30 brokers
     - Use Kafka quotas to prevent producer overload
     - Cost: $10k → $100k/month

2. **Cassandra Write Bottleneck**
   - Current: 10B writes/day = 115k writes/sec
   - At 10×: 100B writes/day = 1.15M writes/sec
   - **Solution:**
     - Scale Cassandra cluster: 6 nodes → 60 nodes
     - Increase replication factor:2 → 3 for durability
     - Use batched writes (100 inserts per batch)
     - Add write-optimized instance types (i3.8xlarge)
     - Cost: $15k → $150k/month

3. **Worker Scaling**
   - Current: 50 workers total across all channels
   - At 10×: Need 500 workers
   - **Solution:**
     - Auto-scaling groups based on Kafka consumer lag
     - Kubernetes HPA (Horizontal Pod Autoscaler)
     - Target metric: Consumer lag < 1,000 messages
     - Cost: $20k → $200k/month

4. **Redis Cache Bottleneck (User Preferences)**
   - Current: 100M users, 10 GB cache
   - At 10×: 1B users, 100 GB cache
   - **Solution:**
     - Shard Redis: 1 cluster → 10 sharded clusters
     - Use Redis Cluster mode (automatic sharding)
     - Consistent hashing for user_id → shard mapping
     - Cost: $5k → $50k/month

5. **Third-Party Rate Limits**
   - Current: SendGrid 10k emails/min
   - At 10×: Need 100k emails/min
   - **Solution:**
     - Negotiate enterprise contracts (higher limits)
     - Use multiple third-party providers (SendGrid + Mailgun + Amazon SES)
     - Distribute load across providers (33% each)
     - Failover if one provider is down

**Total Cost Estimate:**
- Current (1B/day): ~$50k/month
- 10× Scale (10B/day): ~$500k/month
- **Cost per notification: $0.000005 (half a cent per 1,000 notifications)**

---

### Question 2: How do you prevent notification spam and fatigue?

**Answer:**

**Problem:** Users overwhelmed by notifications → unsubscribe or uninstall app.

**Multi-Layer Solution:**

**1. Frequency Limiting (Per-User Rate Limiting)**
```
Per-User Limits:
- Max 50 notifications per hour
- Max 200 notifications per day
- Max 1,000 notifications per week

Implementation:
function should_send_notification(user_id):
  hour_key = "notif:hour:" + user_id
  day_key = "notif:day:" + user_id
  
  hour_count = redis.INCR(hour_key)
  redis.EXPIRE(hour_key, 3600)  // 1 hour TTL
  
  day_count = redis.INCR(day_key)
  redis.EXPIRE(day_key, 86400)  // 1 day TTL
  
  if hour_count > 50 or day_count > 200:
    return false  // Skip notification
  
  return true
```

**2. Intelligent Batching (Digest Mode)**
```
Instead of:
- 10:00 AM: "John liked your post"
- 10:05 AM: "Sarah commented on your post"
- 10:10 AM: "Mike shared your post"
= 3 separate notifications (annoying!)

Batching:
- Queue notifications for 30 minutes
- Send single digest: "John, Sarah, and 2 others interacted with your post"
= 1 notification (better UX!)

Implementation:
function queue_notification(notification):
  digest_key = "digest:" + notification.user_id + ":" + notification.type
  redis.LPUSH(digest_key, notification)
  redis.EXPIRE(digest_key, 1800)  // 30-minute batch window
  
  // Schedule digest job
  schedule_job(send_digest, user_id=notification.user_id, delay=1800)

function send_digest(user_id):
  notifications = redis.LRANGE("digest:" + user_id + ":*", 0, -1)
  
  if len(notifications) == 1:
    send_individual_notification(notifications[0])
  else:
    send_digest_notification(user_id, notifications)
```

**3. Quiet Hours**
```
User Preferences:
- Quiet hours: 10 PM - 8 AM
- Only allow critical notifications (e.g., security alerts)

Implementation:
function should_send_now(notification, preferences):
  if is_quiet_hours(preferences.quiet_hours_start, preferences.quiet_hours_end):
    if notification.priority == "CRITICAL":
      return true  // Send security alerts immediately
    else:
      defer_until_morning(notification, preferences.quiet_hours_end)
      return false
  
  return true
```

**4. Smart Prioritization (Only Send High-Value Notifications)**
```
Machine Learning Model:
- Train model on historical data: (notification, user) → engagement_score
- Features: notification_type, user_activity, time_of_day, device_type
- Predict: Will user engage with this notification? (click/open rate)

function should_send_notification(notification, user):
  engagement_score = ml_model.predict(notification, user)
  
  if engagement_score > 0.3:  // 30% predicted engagement
    send_notification(notification)
  else:
    skip_notification(notification)  // Don't annoy user with low-value notif

Result:
- 50% fewer notifications sent
- 2× higher engagement rate
- Happier users
```

**5. User Control (Granular Preferences)**
```
PostgreSQL user_preferences:
{
  email_enabled: true,
  sms_enabled: false,  // User opted out of SMS
  push_enabled: true,
  
  // Granular control
  notification_types: {
    "NEW_FOLLOWER": true,
    "COMMENT": true,
    "LIKE": false,  // Don't send likes notifications
    "MENTION": true
  },
  
  frequency: "DIGEST",  // Options: INSTANT, DIGEST, DAILY
  quiet_hours: "22:00-08:00"
}
```

**Monitoring:**
- Track notification fatigue metrics:
  - Opt-out rate (should be < 2%)
  - App uninstall rate after notification spike
  - Engagement rate (should be > 20%)

---

### Question 3: How do you ensure exactly-once delivery?

**Answer:**

**Challenge:** Prevent duplicate notifications to users.

**Problem Scenarios:**

1. **Producer Retries (Kafka)**
```
Scenario:
- Notification Service publishes to Kafka
- Kafka acknowledges but network fails before acknowledgment reaches producer
- Producer retries → Duplicate message in Kafka ❌

Solution: Idempotent Producer (Kafka 0.11+)
kafka_producer = KafkaProducer(enable_idempotence=true)
// Kafka automatically deduplicates based on sequence number
```

2. **Consumer Retries**
```
Scenario:
- Worker processes notification, sends email
- Worker crashes before committing Kafka offset
- Kafka redelivers message → Duplicate email sent ❌

Solution: Idempotency Key
function process_notification(notification):
  idempotency_key = "notif:" + notification.id
  
  if redis.EXISTS(idempotency_key):
    return "ALREADY_PROCESSED"
  
  send_notification(notification)
  
  redis.SETEX(idempotency_key, 86400, "1")  // 24h TTL
  kafka.commit_offset()
```

3. **Third-Party API Retries**
```
Scenario:
- Worker sends email to SendGrid
- SendGrid processes email but response times out
- Worker retries → Duplicate email ❌

Solution: API Idempotency Keys
sendgrid.send_email(
  to: user.email,
  subject: "Order Confirmed",
  body: message,
  idempotency_key: "order:" + order.id + ":notification"  // SendGrid deduplicates
)
```

**Complete Exactly-Once Flow:**
```
1. Producer (Notification Service):
   - Enable idempotent producer (Kafka handles dedup)
   
2. Consumer (Worker):
   - Check idempotency key before processing
   - Process notification
   - Store idempotency key
   - Commit Kafka offset (atomic with idempotency key)
   
3. Third-Party API:
   - Use API's idempotency key feature
   - If API doesn't support: Store API request ID in Redis
```

**Trade-off:**
- Adds 5ms latency (Redis idempotency check)
- Requires Redis storage (1 byte per notification × 24h × 1B notifications/day = 24 GB)
- **Worth it:** Prevents annoying duplicate notifications

---

### Question 4: How do you handle third-party provider downtime?

**Answer:**

**Scenario:** SendGrid email service is completely down for 4 hours.

**Multi-Layer Strategy:**

**1. Circuit Breaker (Fail Fast)**
```
When SendGrid fails:
- After 5 consecutive failures: Open circuit
- Stop sending requests to SendGrid for 60 seconds
- Move messages to Dead Letter Queue
- Alert on-call engineer
```

**2. Multi-Provider Failover**
```
Email Providers (in priority order):
1. SendGrid (primary, 70% of traffic)
2. Mailgun (secondary, 20% of traffic)
3. Amazon SES (fallback, 10% of traffic)

function send_email_with_failover(notification):
  providers = [SendGrid, Mailgun, AmazonSES]
  
  for provider in providers:
    if circuit_breaker[provider].is_open():
      continue  // Skip failed provider
    
    try:
      provider.send_email(notification)
      return SUCCESS
    except Exception as e:
      circuit_breaker[provider].record_failure()
      continue  // Try next provider
  
  // All providers failed
  move_to_dead_letter_queue(notification)
```

**3. Dead Letter Queue Processing**
```
DLQ Worker (runs hourly):
  dlq_messages = kafka.consume("dlq-email", limit=10000)
  
  for message in dlq_messages:
    age_hours = (now() - message.timestamp) / 3600
    
    if age_hours < 24:
      // Retry with exponential backoff
      retry_notification(message)
    elif age_hours < 168:  // 7 days
      // Move to cold DLQ for manual review
      cold_storage.archive(message)
    else:
      // Too old, discard
      delete_notification(message)
```

**4. Degraded Mode Operation**
```
When primary provider is down:
- Continue serving requests (don't fail user requests)
- Use secondary providers
- Send critical notifications only (priority > HIGH)
- Defer non-critical notifications to DLQ
- Display status page: "Email notifications delayed, we're working on it"
```

**5. Monitoring and Alerting**
```
PagerDuty Alerts:
- Critical: Circuit breaker opened for primary provider → Page on-call
- Warning: DLQ message count > 10,000 → Slack alert
- Info: Secondary provider usage > 50% → Email alert

Runbook:
1. Check provider status page (status.sendgrid.com)
2. If provider outage: Wait for recovery, monitor DLQ
3. If our issue: Check API keys, rate limits, account status
4. If prolonged outage (> 4 hours): Switch to secondary provider as primary
```

**Cost Trade-off:**
- Multi-provider setup: 20% higher cost (maintaining relationships with 3 providers)
- **Worth it:** 99.99% uptime vs 99.5% with single provider

---

### Question 5: How do you implement A/B testing for notifications?

**Answer:**

**Goal:** Test different notification copy, subject lines, send times to optimize engagement.

**Example A/B Test:** Which subject line has higher open rate?
- **Variant A:** "Your order is confirmed!"
- **Variant B:** "Great news! Your order #{{order_id}} is on the way 🎉"

**Implementation:**

**1. Template Variants (Database)**
```
PostgreSQL notification_templates:
CREATE TABLE notification_templates (
  template_id VARCHAR PRIMARY KEY,
  variant VARCHAR,  // 'A', 'B', 'control'
  channel VARCHAR,
  subject VARCHAR,
  body TEXT,
  enabled BOOLEAN,
  traffic_percentage INT  // % of users who see this variant
);

INSERT INTO notification_templates VALUES
('ORDER_CONFIRMATION', 'A', 'EMAIL', 'Your order is confirmed!', '...', true, 50),
('ORDER_CONFIRMATION', 'B', 'EMAIL', 'Great news! Your order #{{order_id}} is on the way 🎉', '...', true, 50);
```

**2. User Assignment (Consistent Hashing)**
```
function get_template_variant(template_id, user_id):
  // Consistent assignment: same user always gets same variant
  hash = murmurhash(template_id + ":" + user_id)
  bucket = hash % 100  // 0-99
  
  // Example: Variant A gets 0-49, Variant B gets 50-99
  if bucket < 50:
    return get_template(template_id, variant='A')
  else:
    return get_template(template_id, variant='B')

function send_notification(user_id, template_id):
  template = get_template_variant(template_id, user_id)
  message = template.render(user_data)
  send_email(user.email, template.subject, message)
  
  // Track variant for analytics
  log_notification(
    user_id: user_id,
    template_id: template_id,
    variant: template.variant,
    timestamp: now()
  )
```

**3. Analytics Tracking**
```
Cassandra analytics table:
CREATE TABLE notification_analytics (
  experiment_id VARCHAR,
  variant VARCHAR,
  event_type VARCHAR,  // 'SENT', 'DELIVERED', 'OPENED', 'CLICKED'
  user_id BIGINT,
  timestamp TIMESTAMP,
  PRIMARY KEY ((experiment_id, variant), timestamp)
);

Track events:
- SENT: Notification sent
- DELIVERED: Third-party confirmed delivery
- OPENED: User opened email (tracking pixel)
- CLICKED: User clicked link in email
```

**4. Results Dashboard**
```
Query after 7 days (10,000 users per variant):

Variant A (Control):
- Sent: 10,000
- Delivered: 9,500 (95%)
- Opened: 1,900 (20% open rate)
- Clicked: 380 (4% CTR)

Variant B (New Copy):
- Sent: 10,000
- Delivered: 9,500 (95%)
- Opened: 2,850 (30% open rate) ⬆️ 50% improvement!
- Clicked: 570 (6% CTR) ⬆️ 50% improvement!

Decision: Roll out Variant B to 100% of users
```

**5. Statistical Significance**
```
Use Chi-Square Test to determine if result is statistically significant:

function is_statistically_significant(variant_a, variant_b):
  chi_square, p_value = chi_square_test(variant_a.opens, variant_b.opens)
  
  if p_value < 0.05:  // 95% confidence
    return true  // Result is significant
  else:
    return false  // Not enough data, continue test
```

**Best Practices:**
- Run test for at least 7 days (capture weekly patterns)
- Minimum sample size: 1,000 users per variant
- Test one variable at a time (subject line OR body, not both)
- Monitor for negative metrics (unsubscribe rate)
- Use multivariate testing for complex scenarios

---

## Real-World Examples

- **Slack**: WebSockets for real-time, Kafka for events, Cassandra for history
- **Uber**: WebSockets for driver location, push for ride status, SMS fallback
- **Amazon**: SQS for queuing, DynamoDB for logs, A/B testing for copy
- **Facebook**: Persistent connections, push notifications, multi-region deployment

## Cost Analysis (AWS Example)

**Assumptions:**
- 1 billion notifications/day
- 11,574 QPS average, 57,870 QPS peak
- 100M daily active users
- 183 TB/year log storage

| Component                         | Specification                        | Monthly Cost        |
|-----------------------------------|--------------------------------------|---------------------|
| **Notification Service (EC2)**    | 10× t3.xlarge (4 vCPU, 16 GB)       | $1,360              |
| **Kafka Cluster**                 | 3× kafka.m5.xlarge                   | $1,092              |
| **Email Workers (EC2)**           | 10× t3.medium (2 vCPU, 4 GB)        | $408                |
| **SMS Workers (EC2)**             | 5× t3.medium                         | $204                |
| **Push Workers (EC2)**            | 20× t3.medium                        | $816                |
| **WebSocket Servers (EC2)**       | 50× t3.medium (10k connections each) | $2,040              |
| **Redis (User Preferences)**      | cache.r5.large (13 GB RAM)           | $137                |
| **Cassandra Cluster (Logs)**      | 6× i3.xlarge (4 vCPU, 30 GB RAM)     | $2,394              |
| **PostgreSQL (Preferences)**      | db.r5.large (2 vCPU, 16 GB)          | $182                |
| **Application Load Balancer**     | Standard ALB                         | $20                 |
| **CloudWatch Monitoring**         | Logs + Metrics + Dashboards          | $200                |
| **VPC, NAT Gateway**              | Multi-AZ setup                       | $90                 |
| **Data Transfer**                 | 5 TB/month egress                    | $450                |
| **Third-Party Providers**         |                                      |                     |
| ├─ SendGrid (Email)               | 400M emails/month                    | $800                |
| ├─ Twilio (SMS)                   | 150M SMS/month @ $0.01 each          | $1,500              |
| ├─ FCM/APNS (Push)                | Free (first 20M/day)                 | $0                  |
| **Total Infrastructure**          |                                      | **~$9,693/month**   |
| **Total Third-Party**             |                                      | **~$2,300/month**   |
| **Grand Total**                   |                                      | **~$12,000/month**  |

**Cost Breakdown by Channel:**
```
Per Notification Cost:
- Email: $0.002 per email (SendGrid)
- SMS: $0.01 per SMS (Twilio)
- Push: Free (Google/Apple)
- Web: ~$0.00001 per notification (infrastructure only)

Cost per 1,000 notifications:
- Email: $2.00
- SMS: $10.00
- Push: $0.00
- Web: $0.01
```

**Cost Optimization:**

| Optimization                     | Savings       | Notes                                      |
|----------------------------------|---------------|--------------------------------------------|
| **Reserved Instances (1-year)**  | -40% EC2      | Save $2,000/month on EC2                   |
| **Spot Instances for Workers**   | -70% EC2      | Use Spot for 50% of workers → Save $1,400/month |
| **Cassandra on Local SSD**       | -30% storage  | Use i3 instances vs EBS → Save $700/month  |
| **Push Instead of SMS**          | -90% per msg  | Encourage push over SMS → Save $1,350/month|
| **Email Batching (Digest Mode)** | -50% emails   | Reduce email volume → Save $400/month      |
| **Redis Cluster Tier**           | -25% cache    | Use open-source Redis vs ElastiCache → Save $35/month |

**Optimized Cost: ~$7,100/month** (~40% savings)

**Cost per Notification:**
- Current: $12,000 / 1B notifications = **$0.000012 per notification**
- Optimized: $7,100 / 1B notifications = **$0.000007 per notification**
- **Industry Benchmark:** $0.00001 per notification (we're 30% cheaper!)

---

## Trade-offs Summary

| Decision | Gain | Sacrifice |
|----------|------|-----------|
| Kafka for Async | Guaranteed delivery, scalability | ~100ms latency, complexity |
| Cassandra for Logs | High write throughput | No joins, eventual consistency |
| WebSockets | Low latency (<100ms) | Connection management complexity |
| Multi-Channel Workers | Independent scaling | More microservices |
| Redis Cache | <1ms lookup | Stale data (5 min TTL) |
| Circuit Breaker | Fail fast, prevent cascading failures | Temporary notification delays |
| Idempotency Keys | Prevent duplicate notifications | 5ms latency overhead, Redis storage |
| Template-Based | Easy A/B testing, localization | DB lookup overhead (~10ms) |

---

## Summary

A notification service requires careful balance between **real-time delivery**, **reliability**, and **cost**:

**Key Design Choices:**

1. ✅ **Kafka for Async Processing** to handle 57k QPS peaks and guarantee delivery
2. ✅ **Multi-Channel Architecture** (Email, SMS, Push, Web) with independent workers for scalability
3. ✅ **Cassandra for Logs** to handle 1B writes/day (115k writes/sec)
4. ✅ **WebSockets for Real-Time** in-app notifications with < 100ms latency
5. ✅ **Circuit Breaker + DLQ** for resilience against third-party failures
6. ✅ **Idempotency Keys** to prevent duplicate notifications
7. ✅ **Rate Limiting (Token Bucket)** to respect third-party API limits
8. ✅ **Redis Cache** for user preferences (<1ms lookup)

**Performance Characteristics:**

- **Throughput:** 11,574 QPS average, 57,870 QPS peak (5× average)
- **Latency:** <2 seconds for real-time notifications (push, web)
- **Latency:** <100ms for async delivery (email, SMS eventually sent)
- **WebSocket Connections:** 10M concurrent connections supported
- **Delivery Rate:** 99.9% for critical notifications, 99.5% overall
- **Cassandra Write Throughput:** 115k writes/sec sustained

**Critical Components:**

- **Kafka:** Must handle 57k QPS peaks, 12 partitions per topic minimum
- **Cassandra:** Write-optimized, 6+ nodes for 1B writes/day
- **WebSocket Server:** Sticky sessions, Redis pub/sub for cross-server communication
- **Circuit Breaker:** Fail fast after 5 failures, 60-second timeout
- **Dead Letter Queue:** Retry failed notifications, manual investigation

**Scalability Path:**

1. **0-1M notifications/day:** Single server, PostgreSQL, synchronous delivery
2. **1M-100M/day:** Add Kafka, Redis cache, separate workers
3. **100M-1B/day:** Scale to 50 workers, Cassandra for logs, multi-region
4. **1B-10B/day:** 500 workers, sharded Redis, 10× Cassandra nodes, regional Kafka clusters

**Common Pitfalls to Avoid:**

1. ❌ Synchronous notification delivery (blocks user requests)
2. ❌ No idempotency handling (duplicate notifications)
3. ❌ No rate limiting (overwhelm third-party APIs)
4. ❌ No circuit breaker (cascading failures during outages)
5. ❌ Single worker for all channels (cannot scale independently)
6. ❌ Storing sensitive data in logs (GDPR violations)
7. ❌ No retry logic with exponential backoff (overwhelm failing services)
8. ❌ Hardcoded notification templates (requires deployment for copy changes)

**Recommended Stack:**

- **API Gateway:** Kong or AWS API Gateway (auth, rate limiting)
- **Message Queue:** Apache Kafka (fault-tolerant, high throughput)
- **Notification Service:** Go or Java (stateless, horizontally scalable)
- **Workers:** Python (asyncio) or Go (goroutines) for async I/O
- **Cache:** Redis Cluster (user preferences, idempotency keys)
- **Logs Database:** Cassandra or ScyllaDB (write-optimized)
- **User Preferences:** PostgreSQL (ACID, relations)
- **Third-Party Providers:** SendGrid (email), Twilio (SMS), FCM/APNS (push)
- **WebSocket:** Socket.io or native WebSockets with sticky sessions
- **Monitoring:** Prometheus + Grafana + PagerDuty + Distributed Tracing (Jaeger)

**Cost Efficiency:**

- Optimize with Reserved Instances (save 40% on EC2)
- Use Spot Instances for non-critical workers (save 70%)
- Encourage push over SMS (10× cheaper)
- Implement digest mode for email (50% fewer emails)
- **Optimized cost:** ~$7,100/month for 1B notifications/day ($0.000007 per notification)

**Real-World Comparisons:**

| Feature                  | Our Design               | Slack                | Uber                 | Amazon SNS         |
|--------------------------|--------------------------|----------------------|----------------------|--------------------|
| **Channels**             | Email, SMS, Push, Web    | Push, Web, Email     | Push, SMS            | Multi-channel      |
| **Scale**                | 1B notifications/day     | Unknown (very high)  | Unknown (very high)  | Trillions/day      |
| **Real-Time**            | WebSockets (<100ms)      | WebSockets           | WebSockets           | Pub/Sub            |
| **Async Queue**          | Kafka                    | Kafka                | Proprietary          | AWS SQS/SNS        |
| **Logs**                 | Cassandra                | Cassandra            | Cassandra            | DynamoDB           |
| **Third-Party**          | SendGrid, Twilio, FCM    | In-house + third-party| Twilio, FCM, APNS   | Multiple providers |
| **Cost (1B notif/day)**  | ~$7,100/month            | Unknown              | Unknown              | ~$10/month + usage |

This design provides a **production-ready, scalable blueprint** for building a notification service that can handle billions of notifications per day across multiple channels while maintaining high reliability and cost efficiency! 🚀

---

## Getting Started

1. **Read Main Document**: [3.2.2-design-notification-service.md](./3.2.2-design-notification-service.md)
2. **Explore Diagrams**: [hld-diagram.md](./hld-diagram.md) and [sequence-diagrams.md](./sequence-diagrams.md)
3. **Understand Decisions**: [this-over-that.md](./this-over-that.md)
4. **Study Algorithms**: [pseudocode.md](./pseudocode.md)

## References

- **[2.3.1 Asynchronous Communication](../../02-components/2.3.1-asynchronous-communication.md)**
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)**
- **[2.0.3 Real-Time Communication](../../02-components/2.0.3-real-time-communication.md)**
- **[2.5.1 Rate Limiting Algorithms](../../02-components/2.5.1-rate-limiting-algorithms.md)**
- **[3.2.1 Design Twitter Timeline](../3.2.1-twitter-timeline/)** - Similar fanout patterns

## Getting Started

1. **Read Main Document**: [3.2.2-design-notification-service.md](./3.2.2-design-notification-service.md)
2. **Explore Diagrams**: [hld-diagram.md](./hld-diagram.md) and [sequence-diagrams.md](./sequence-diagrams.md)
3. **Understand Decisions**: [this-over-that.md](./this-over-that.md)
4. **Study Algorithms**: [pseudocode.md](./pseudocode.md)

## References

- **[2.3.1 Asynchronous Communication](../../02-components/2.3.1-asynchronous-communication.md)**
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)**
- **[2.0.3 Real-Time Communication](../../02-components/2.0.3-real-time-communication.md)**
- **[2.5.1 Rate Limiting Algorithms](../../02-components/2.5.1-rate-limiting-algorithms.md)**
- **[3.2.1 Design Twitter Timeline](../3.2.1-twitter-timeline/)** - Similar fanout patterns

