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

### ❌ Synchronous Notification Delivery
**Problem**: User waits for email to send before getting order confirmation

✅ **Solution**: Async with Kafka, return 202 immediately

### ❌ No Idempotency
**Problem**: Duplicate events → duplicate notifications

✅ **Solution**: Use idempotency keys with Redis TTL

### ❌ No Rate Limiting
**Problem**: Overwhelming third-party APIs → blocked API key

✅ **Solution**: Token Bucket rate limiter per worker

### ❌ No Circuit Breaker
**Problem**: Continuous retries during third-party outage

✅ **Solution**: Circuit Breaker opens after N failures

### ❌ Single Worker for All Channels
**Problem**: Cannot scale channels independently

✅ **Solution**: Separate workers per channel

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

## Real-World Examples

- **Slack**: WebSockets for real-time, Kafka for events, Cassandra for history
- **Uber**: WebSockets for driver location, push for ride status, SMS fallback
- **Amazon**: SQS for queuing, DynamoDB for logs, A/B testing for copy
- **Facebook**: Persistent connections, push notifications, multi-region deployment

## Trade-offs Summary

| Decision | Gain | Sacrifice |
|----------|------|-----------|
| Kafka for Async | Guaranteed delivery, scalability | ~100ms latency, complexity |
| Cassandra for Logs | High write throughput | No joins, eventual consistency |
| WebSockets | Low latency (<100ms) | Connection management complexity |
| Multi-Channel Workers | Independent scaling | More microservices |
| Redis Cache | <1ms lookup | Stale data (5 min TTL) |

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

