# Live Commenting System - Quick Overview

## Core Concept

A real-time commenting system for live streaming platforms (Facebook Live, Twitch) that delivers comments to millions of concurrent viewers with sub-second latency. It handles massive fanout (1 comment → 5M viewers), extreme write bursts (10k comments/sec), real-time moderation, and maintains persistent WebSocket connections for instant delivery.

**Key Challenge**: Broadcast 10k comments/sec to 5M concurrent viewers with <500ms latency, handle connection management, and provide real-time moderation.

---

## Requirements

### Functional
- **Real-Time Delivery**: Comments broadcast to all viewers within 500ms
- **High Fanout**: Single comment delivered to millions of viewers simultaneously
- **Persistent Connections**: Maintain WebSocket connections for instant push
- **Moderation**: Real-time spam/offensive content filtering
- **Comment History**: Store comments for replay/analytics
- **User Presence**: Show active viewer count
- **Emoji Reactions**: Support quick reactions (like, heart, fire)
- **Pinned Comments**: Allow broadcasters to pin important comments

### Non-Functional
- **Comment Latency**: <500ms (p95) for sub-second delivery
- **Connection Stability**: 99.9% uptime (uninterrupted streams)
- **Write Throughput**: 10k comments/sec per stream
- **Fanout Capacity**: 50B pushes per event (10k comments × 5M viewers)
- **Connection Limit**: 1M connections per server
- **Data Consistency**: Eventual consistency acceptable (real-time prioritized)

---

## Components

### 1. **WebSocket Connection Layer**
- **Servers**: 5,000 WebSocket servers (1M connections each = 5M total)
- **Connection Registry**: Redis Hash (`conn:user:{user_id} → ws-server-42`)
- **Heartbeat**: Client sends ping every 30 seconds
- **Sticky Sessions**: Load balancer routes to same server (connection affinity)

### 2. **Comment Ingestion Service**
- **Rate Limiting**: Per-user limit (10 comments/min)
- **Validation**: Check message length, format
- **Kafka**: Write to `comments` topic (partition by `stream_id`)
- **Async Moderation**: Don't block comment delivery (moderation happens async)

### 3. **Broadcast Coordinator**
- **Kafka Consumer**: Consumes from `comments` topic
- **Redis Pub/Sub**: Publishes to `stream:{stream_id}:comments` channel
- **Fanout**: 5,000 WebSocket servers subscribe to Redis channel
- **Adaptive Throttling**: Sample comments during extreme traffic (20% during peak)

### 4. **Moderation Service**
- **Pipeline**:
  1. Rule-Based Filter (5ms): Blacklisted words
  2. ML Model (50ms): Toxicity scoring
  3. User Reputation: Repeat offenders → shadow ban
- **Async Processing**: Comments published immediately, deleted if flagged (1-2s delay acceptable)

### 5. **Comment Storage (Cassandra)**
- **Time-Series Optimized**: Clustering by `comment_id` (TIMEUUID, DESC)
- **Partition Key**: `stream_id` (all comments for one stream together)
- **Queries**: Recent comments (last N comments for replay)

### 6. **Presence Service (Redis)**
- **Viewer Count**: HyperLogLog (`stream:viewers:{stream_id}`)
- **Connection Registry**: Hash (`conn:user:{user_id}`)
- **Active Streams Cache**: Hash (`stream:{stream_id}`)

---

## Architecture Flow

### Comment Ingestion Flow

```
1. Client → WebSocket Server: Send comment {stream_id, message}
2. WebSocket Server → Ingestion Service: Validate, rate limit
3. Ingestion Service → Kafka: Write to comments topic (partition by stream_id)
4. Ingestion Service → WebSocket Server: Return success
5. WebSocket Server → Client: Acknowledge comment
6. Broadcast Coordinator → Kafka: Consume comment
7. Broadcast Coordinator → Redis Pub/Sub: Publish to stream:{stream_id}:comments
8. 5,000 WebSocket Servers → Redis Pub/Sub: Subscribe to channel (parallel)
9. WebSocket Servers → Clients: Broadcast comment (5M viewers)
10. Moderation Service → Kafka: Consume comment (async)
11. Moderation Service → ML Model: Check toxicity
12. If flagged → Moderation Service → Kafka: Publish COMMENT_DELETED event
13. Broadcast Coordinator → Clients: Delete comment (if flagged)
```

### Connection Establishment Flow

```
1. Client → API Gateway: Request WebSocket connection
2. API Gateway → Load Balancer: Route to available WebSocket server
3. WebSocket Server → Authentication: Validate JWT token
4. WebSocket Server → Redis: Store connection mapping (conn:user:{user_id} → ws-server-42)
5. WebSocket Server → Redis Pub/Sub: Subscribe to stream:{stream_id}:comments
6. WebSocket Server → Client: WebSocket connection established
7. Client → WebSocket Server: Send ping every 30 seconds (heartbeat)
```

---

## Key Design Decisions

### 1. **Redis Pub/Sub for Fanout** (vs Kafka Direct Broadcast)

**Why Redis Pub/Sub?**
- **Low Latency**: <1ms internal routing
- **Automatic Fan-out**: Redis handles message duplication to all subscribers
- **Decoupling**: WebSocket servers don't need to know about each other
- **Simple**: No need to manage Kafka consumer groups per server

**Trade-off**: Redis Pub/Sub not persistent (acceptable for real-time comments) vs Kafka persistence

### 2. **Two-Tier Fanout Architecture**

**Tier 1: Kafka → Broadcast Coordinator**
- Kafka partitions by `stream_id` (ensures ordering per stream)
- Single coordinator consumes from Kafka (simplifies state)

**Tier 2: Redis Pub/Sub → WebSocket Servers**
- Redis Pub/Sub channels (`stream:{stream_id}:comments`)
- 5,000 WebSocket servers subscribe in parallel
- Automatic fan-out (Redis handles duplication)

**Trade-off**: Two hops (Kafka → Redis → WebSocket) vs single hop (Kafka → WebSocket, complex)

### 3. **Async Moderation** (vs Synchronous)

**Why Async?**
- **Write Speed Priority**: Comments published instantly (<50ms)
- **Eventual Consistency Acceptable**: Offensive comment visible for 1-2 seconds before removal
- **User Experience**: No blocking wait for AI model (100ms latency)
- **Scalability**: Moderation doesn't become bottleneck

**Trade-off**: Eventual consistency (1-2s delay) vs immediate blocking (slower)

### 4. **Connection Registry in Redis** (vs In-Memory)

**Why Redis?**
- **Connection Migration**: Allows finding which server holds a specific user's connection
- **Targeted Operations**: Disconnect/ban specific users
- **Scaling**: Supports connection migration during server scaling
- **Fault Tolerance**: Survives server restarts

**Trade-off**: Additional Redis lookup (1ms) vs in-memory (faster, but not fault-tolerant)

### 5. **Adaptive Throttling** (vs Always Broadcast All)

**Why?**
- **Bandwidth Savings**: 10k comments/sec × 5M viewers = 10 TB/sec (overwhelming)
- **User Experience**: Show banner "Chat moving fast! Showing sampled comments"
- **Graceful Degradation**: Always store all comments (available in replay)

**Implementation**: Sample 20% of comments during peak (>5000 comments/sec)

**Trade-off**: Some comments not shown in real-time vs bandwidth overload

### 6. **Cassandra for Comment Storage** (vs PostgreSQL)

**Why Cassandra?**
- **Time-Series Optimized**: Clustering by `comment_id` (TIMEUUID, DESC) for recent comments
- **High Write Throughput**: 100k+ writes/sec per node
- **Horizontal Scaling**: Linear scaling with nodes
- **Partition Key**: `stream_id` (all comments for one stream together)

**Trade-off**: Eventual consistency vs strong consistency

---

## Advanced Features

### 1. **Emoji Reactions (Fire, Heart, etc.)**

**Challenge**: Handle 100k reactions/sec (faster than text comments)

**Solution**: Aggregate reactions locally, flush every 500ms

```
Client sends: 🔥 reaction
  ↓
WebSocket Server buffers locally
  ↓
Every 500ms: Flush aggregated counts to Redis
  ↓
Redis: HINCRBY reactions:stream:789 fire 1250
  ↓
Broadcast aggregated update: {"fire": 15000, "heart": 8000}
```

**Why Aggregation?**
- **Reduces Traffic**: 100k reactions/sec → 2 updates/sec (50,000x reduction)
- **User Experience**: Show running total (doesn't need per-reaction update)
- **Bandwidth Savings**: 99.99% reduction in reaction traffic

### 2. **Pinned Comments**

**Flow**:
1. Broadcaster pins comment: `POST /stream/789/pin (comment_id=123)`
2. Store in Redis: `SET stream:789:pinned 123`
3. Broadcast event: `{"type": "COMMENT_PINNED", "comment_id": 123}`
4. Client UI: Display pinned comment at top (sticky)

### 3. **Slow Mode**

**Purpose**: Limit comment rate during moderation-heavy streams

**Implementation**: Per-user rate limit (e.g., 1 comment per 10 seconds)

---

## Bottlenecks & Solutions

### 1. **WebSocket Server CPU Bottleneck**

**Problem**: 1M connections × 10 messages/sec = 10M messages/sec per server

**Solution**: Batch message delivery
- Buffer messages for 10ms
- Send all messages in one WebSocket frame
- **Result**: 10M syscalls/sec → 100k syscalls/sec (100x reduction)

### 2. **Redis Pub/Sub Hot Key**

**Problem**: Single Redis Pub/Sub channel (`stream:789:comments`) becomes hot key

**Solution**: Shard Pub/Sub channels by WebSocket server
- Instead of: `stream:789:comments`
- Use: `stream:789:comments:shard0`, `stream:789:comments:shard1`, ... (100 shards)
- Broadcast coordinator publishes to all shards
- Each WebSocket server subscribes to 1 shard

**Result**: Distributes load across 100 Redis keys (100x reduction in contention)

### 3. **Network Bandwidth Bottleneck**

**Problem**: 10 TB/sec fanout bandwidth exceeds data center capacity

**Solution**: Aggressive compression + protocol optimization
1. **MessagePack Serialization**: 50% smaller than JSON
2. **WebSocket Compression**: permessage-deflate (60% reduction)
3. **Delta Encoding**: Send only changed fields
4. **Adaptive Sampling**: 80% reduction during peak

**Result**: 10 TB/sec → 0.5 TB/sec (20x reduction)

---

## Common Anti-Patterns

### ❌ **1. Polling for New Comments**

**Problem**: Client polls `/stream/789/comments?since=timestamp` every second

**Why It's Bad**:
- 5M clients × 1 req/sec = 5M QPS on API servers
- High latency (1-second delay)
- Wasted bandwidth (empty responses when no comments)

**Solution**: Use WebSockets for push-based delivery

### ❌ **2. Synchronous Moderation**

**Problem**: Comment submission blocked until moderation completes

**Why It's Bad**:
- ML model takes 100ms → user waits
- 3x slower comment delivery
- Moderation becomes scaling bottleneck

**Solution**: Async moderation with eventual consistency

### ❌ **3. Storing All Connections in Single Redis**

**Problem**: Connection registry stored in single Redis instance

**Why It's Bad**:
- 5M connections × 200 bytes = 1 GB
- Single point of failure
- Memory limit (single Redis: 64 GB max)

**Solution**: Shard connection registry across 100 Redis nodes

### ❌ **4. Broadcasting from Kafka Directly to WebSocket Servers**

**Problem**: Each WebSocket server consumes from Kafka

**Why It's Bad**:
- 5,000 servers × Kafka consumer groups = 5,000 consumer groups
- Complex state management
- Kafka partition limits (200 partitions max)

**Solution**: Two-tier fanout (Kafka → Redis Pub/Sub → WebSocket)

---

## Monitoring & Observability

### Key Metrics

- **Comment Latency P95**: <500ms
- **WebSocket Connection Count**: 5M concurrent
- **Comment Ingestion Rate**: 10k comments/sec (peak)
- **Fanout Latency**: <150ms (p95)
- **Redis Pub/Sub Latency**: <10ms
- **Kafka Consumer Lag**: <1 second
- **Moderation Queue Depth**: <1000
- **Connection Drop Rate**: <0.1%

### Alerts

- **Comment Latency P95** >1 second (5-minute window) → Page on-call
- **WebSocket Connection Drop Rate** >5% → Page on-call
- **Kafka Consumer Lag** >60 seconds → Page on-call
- **Redis CPU** >80% → Scale Redis cluster
- **Moderation Queue Depth** >10k → Scale moderation workers

---

## Trade-offs Summary

| Decision | What We Gain | What We Sacrifice |
|----------|--------------|-------------------|
| **Redis Pub/Sub Fanout** | ✅ Low latency, automatic fan-out | ❌ Not persistent (acceptable) |
| **Two-Tier Architecture** | ✅ Simple state management | ❌ Two hops (Kafka → Redis → WebSocket) |
| **Async Moderation** | ✅ Fast comment delivery | ❌ Eventual consistency (1-2s delay) |
| **Connection Registry** | ✅ Fault-tolerant, supports migration | ❌ Additional Redis lookup (1ms) |
| **Adaptive Throttling** | ✅ Bandwidth savings (80%) | ❌ Some comments not shown in real-time |
| **Cassandra Storage** | ✅ High write throughput | ❌ Eventual consistency |
| **Batch Message Delivery** | ✅ 100x reduction in syscalls | ❌ +10ms latency |

---

## Real-World Examples

### Twitch
- **Architecture**: IRC-based protocol, TMI (Twitch Messaging Interface), sharded chat servers (1,000+ servers)
- **Innovation**: IRC over WebSocket (mature IRC scalability patterns)
- **Scale**: 10M+ concurrent viewers, 1M+ messages/minute

### YouTube Live
- **Architecture**: gRPC streaming, Spanner (global consistency), Google CDN
- **Innovation**: Spanner for globally consistent ordering
- **Scale**: 30M+ concurrent viewers (record: FIFA World Cup final)

### Facebook Live
- **Architecture**: MQTT over WebSocket, TAO (custom graph database), edge deployment
- **Innovation**: MQTT over WebSocket (efficient mobile battery usage)
- **Scale**: 50M+ peak concurrent viewers (special events)

---

## Key Takeaways

1. **Two-Tier Fanout**: Kafka (ordering) → Redis Pub/Sub (fan-out) → WebSocket (delivery)
2. **Async Moderation**: Don't block comment delivery (moderation happens async)
3. **Connection Registry**: Redis for fault-tolerant connection mapping
4. **Adaptive Throttling**: Sample comments during peak (20% during >5000 comments/sec)
5. **Batch Message Delivery**: Buffer messages for 10ms (100x reduction in syscalls)
6. **Sharded Pub/Sub**: Distribute load across 100 Redis channels
7. **Emoji Aggregation**: Buffer reactions locally, flush every 500ms (50,000x reduction)
8. **Compression**: MessagePack + WebSocket compression (60% reduction)

---

## Recommended Stack

- **WebSocket Servers**: Go/Elixir (high concurrency, epoll/kqueue)
- **Message Queue**: Kafka (comment ingestion, ordering)
- **Pub/Sub**: Redis Pub/Sub (fan-out to WebSocket servers)
- **Comment Storage**: Cassandra (time-series, high write throughput)
- **Connection Registry**: Redis Cluster (fault-tolerant)
- **Moderation Service**: Python (ML models), TensorFlow
- **Presence Service**: Redis (HyperLogLog for viewer count)
- **Load Balancer**: NGINX / AWS NLB (sticky sessions)
- **Monitoring**: Prometheus, Grafana, OpenTelemetry

