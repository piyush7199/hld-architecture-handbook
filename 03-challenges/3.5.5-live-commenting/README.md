# 3.5.5 Design Live Commenting (Facebook Live/Twitch)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## Problem Statement

Design a real-time commenting system for live streaming platforms like **Facebook Live** or **Twitch** that delivers
comments to millions of concurrent viewers with sub-second latency. The system must handle massive fanout (1 comment → 5M
viewers), extreme write bursts (10k comments/sec), real-time moderation, and maintain persistent connections.

**Core Challenges:**

- **Massive Fanout**: One comment broadcast to 5 million viewers simultaneously
- **Real-Time Delivery**: Comments appear within 500ms
- **Connection Management**: Maintain millions of persistent WebSocket connections
- **Bursty Traffic**: Handle sudden spikes during viral moments
- **Moderation**: Filter offensive content without blocking delivery

Unlike traditional chat, live commenting requires broadcast-first architecture optimized for one-to-many delivery at
extreme scale.

---

## Requirements and Scale Estimation

### Functional Requirements

| Requirement | Description | Priority |
|------------|-------------|----------|
| **Real-Time Delivery** | Comments broadcast within 500ms | Critical |
| **High Fanout** | Single comment → millions of viewers | Critical |
| **Persistent Connections** | WebSocket connections for instant push | Critical |
| **Moderation** | Real-time spam/offensive content filtering | High |
| **Comment History** | Store comments for replay | Medium |
| **User Presence** | Show active viewer count | High |

### Non-Functional Requirements

| Requirement | Target | Justification |
|------------|--------|---------------|
| **Comment Latency** | $< 500$ ms (p95) | Sub-second delivery |
| **Connection Stability** | 99.9% uptime | Uninterrupted streams |
| **Write Throughput** | 10k comments/sec | Handle viral moments |
| **Fanout Capacity** | 50B pushes per event | 10k comments × 5M viewers |
| **Connection Limit** | 1M per server | Efficient resources |

### Scale Estimation

#### Traffic Metrics

| Metric | Assumption | Calculation | Result |
|--------|-----------|-------------|---------|
| **Concurrent Viewers** | 5M viewers per major event | - | 5M concurrent connections |
| **Comment Ingestion** | 2% active commenters | $5M \times 0.02$ | 100k active writers |
| **Peak Comment Rate** | 10% commenters/sec | $100k \times 0.1$ | 10k comments/sec |
| **Fanout Operations** | 10k × 5M viewers | $10k \times 5M$ | 50B pushes/sec peak |
| **Average Rate** | 1k comments/sec sustained | - | 1k comments/sec avg |
| **Event Duration** | 2 hours streaming | - | 2 hours |

#### Storage Estimation

| Component | Calculation | Result |
|-----------|-------------|---------|
| **Comment Size** | 200 bytes avg | 200 bytes |
| **Comments per Event** | 1k/sec × 7200 sec | 7.2M comments |
| **Storage per Event** | $7.2M \times 200$ bytes | 1.44 GB/event |
| **Daily Events** | 10k simultaneous streams | 10k events |
| **Daily Storage** | $10k \times 1.44$ GB | 14.4 TB/day |
| **Annual Storage** | $14.4$ TB × 365 | 5.26 PB/year |

#### Bandwidth Estimation

| Metric | Calculation | Result |
|--------|-------------|---------|
| **Comment Payload** | 200 bytes | 200 bytes |
| **Peak Fanout** | $10k \times 5M \times 200$ | 10 TB/sec peak |
| **Average Fanout** | $1k \times 5M \times 200$ | 1 TB/sec avg |
| **Daily Bandwidth** | 1 TB/sec × 86400 | 86 PB/day |
| **Monthly** | 86 PB × 30 | 2.58 EB/month |

**Key Insight**: Bandwidth dominates infrastructure cost. Optimization through compression and adaptive sampling is
critical.

---

## High-Level Architecture

Specialized **Pub/Sub architecture** optimized for **massive fanout-on-write** using **WebSockets**.

```
┌─────────────────────────────────────────────────────┐
│           Live Commenting System                     │
├─────────────────────────────────────────────────────┤
│                                                       │
│  Viewers (5M) → WebSocket Cluster (5k servers)      │
│                        ↓                             │
│               Broadcast Coordinator                  │
│                        ↓                             │
│                  Kafka Stream                        │
│                    ↙        ↘                        │
│           Moderation      Persistence                │
│           Service         (Cassandra)                │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Core Components

| Component | Technology | Purpose | Scalability |
|-----------|-----------|---------|-------------|
| **API Gateway** | Kong/NGINX | Rate limiting, auth | Horizontal |
| **Ingestion Service** | Go/Java | Accept comments | Horizontal |
| **Kafka** | Kafka Cluster | Buffer comment stream | Partitioned |
| **Moderation** | Python/ML | Spam detection | Horizontal |
| **Broadcast Coordinator** | Go | Fanout orchestration | Horizontal |
| **WebSocket Cluster** | Go/Elixir | Persistent connections | Horizontal |
| **Redis Pub/Sub** | Redis Cluster | Internal routing | Sharded |
| **Cassandra** | Cassandra | Comment storage | Partitioned |

---

## Data Models

### PostgreSQL (Metadata)

#### Streams Table

```sql
CREATE TABLE streams (
    stream_id BIGINT PRIMARY KEY,
    broadcaster_id BIGINT NOT NULL,
    title VARCHAR(500),
    status VARCHAR(20), -- 'live', 'ended'
    started_at TIMESTAMP,
    viewer_count INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### Users Table

```sql
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    is_banned BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Cassandra (Comment Storage)

#### Comments Table

```sql
CREATE TABLE comments (
    stream_id BIGINT,
    comment_id TIMEUUID,
    user_id BIGINT,
    message TEXT,
    is_deleted BOOLEAN,
    created_at TIMESTAMP,
    PRIMARY KEY (stream_id, comment_id)
) WITH CLUSTERING ORDER BY (comment_id DESC);
```

### Redis (Connection Registry)

#### Connection Mapping

```
Key: conn:user:{user_id}
Type: Hash
Fields:
  - stream_id: BIGINT
  - ws_server: "ws-server-42.example.com"
  - connected_at: TIMESTAMP
TTL: 2 hours

Operations:
HSET conn:user:123 stream_id 789 ws_server "ws-42"
HGET conn:user:123 ws_server  // Find server
HDEL conn:user:123  // Disconnect
```

#### Stream Presence (Viewer Count)

```
Key: stream:viewers:{stream_id}
Type: HyperLogLog

Operations:
PFADD stream:viewers:789 user_123  // Add viewer
PFCOUNT stream:viewers:789  // Get unique count
```

---

## Detailed Component Design

### WebSocket Connection Management

**Challenge**: Maintain 5M persistent WebSocket connections efficiently.

**Solution**: Connection sharding across 5,000 servers with connection registry.

#### Connection Establishment Flow

*See pseudocode.md::establish_websocket_connection() for implementation*

1. **Client Request**: User opens stream page
2. **API Gateway**: Routes to available WebSocket server
3. **Authentication**: Validate JWT token
4. **Connection Registry**: Store mapping in Redis
5. **Subscribe**: Server subscribes to Redis Pub/Sub channel
6. **Heartbeat**: Client pings every 30 seconds

**Connection Scaling**:

| Metric | Value | Notes |
|--------|-------|-------|
| **Connections per Server** | 1M connections | Optimized Go/Elixir |
| **Memory per Connection** | 4 KB | Minimal buffer |
| **Total Memory** | 20 GB | 5M × 4 KB across 5 servers |
| **CPU per Message** | 0.1ms | Efficient serialization |

**Optimization Techniques**:

- **Zero-Copy IO**: Use `sendfile()` syscall
- **Connection Pooling**: Reuse HTTP/2 connections
- **Batch Writes**: Buffer comments, flush every 10ms
- **Compression**: WebSocket permessage-deflate

### Comment Ingestion and Moderation

**Flow**:

1. **Client Sends Comment**: WebSocket message
2. **Rate Limiting**: Per-user limit (10/min)
3. **Validation**: Check length, format
4. **Publish to Kafka**: Write to topic (partition by stream_id)
5. **Async Moderation**: ML model processes
6. **Broadcast**: Immediately send to viewers
7. **Delete if Flagged**: Publish delete event if needed

**Why Async Moderation?**

- **Write Speed Priority**: Comments published instantly
- **Eventual Consistency**: Offensive content visible 1-2 seconds
- **User Experience**: No blocking wait for AI (100ms)

**Moderation Pipeline**:

*See pseudocode.md::moderate_comment() for implementation*

```
Stage 1: Rule-Based Filter (5ms)
  - Check blacklisted words
  
Stage 2: ML Model (50ms)
  - Toxicity score prediction
  - Reject if score > 0.9
  
Stage 3: User Reputation
  - Lower threshold for repeat offenders
```

**Moderation Actions**:

- **DELETE**: Remove from all viewers
- **SHADOW_BAN**: Show only to author
- **FLAG**: Manual review, remain visible

### Broadcast and Fanout

**Challenge**: Deliver 1 comment to 5M viewers in <500ms.

**Architecture**: Two-tier fanout (Kafka → Redis Pub/Sub → WebSocket servers)

#### Tier 1: Kafka to Broadcast Coordinator

```
Kafka Partition (stream_id=789)
      ↓
Broadcast Coordinator
      ↓
Redis Pub/Sub channel (stream:789:comments)
```

#### Tier 2: Redis Pub/Sub to WebSocket Servers

```
Redis Pub/Sub (stream:789:comments)
      ↓
5,000 WebSocket Servers (subscribed)
      ↓
5M WebSocket Connections (push)
```

**Why Redis Pub/Sub?**

- **Low Latency**: <1ms internal routing
- **Automatic Fan-out**: Redis duplicates to subscribers
- **Decoupling**: Servers don't know about each other

**Fanout Performance**:

- **Kafka → Coordinator**: 10ms
- **Coordinator → Redis**: 5ms
- **Redis → WebSocket**: 10ms
- **WebSocket → Clients**: 50ms
- **Total**: 75ms (p50), 150ms (p95)

### Adaptive Throttling

**Problem**: Peak rate (10k/sec) may overwhelm bandwidth.

**Solution**: Sample comments during extreme traffic.

*See pseudocode.md::adaptive_throttle() for implementation*

```
if comment_rate > 5000:
  // Sample 20% of comments
  if random() < 0.2:
    broadcast(comment)
  else:
    store_only(comment)  // Save but don't broadcast
```

**User Experience**:

- Show banner: "Chat moving fast! Showing sampled comments."
- Allow pause to read at own pace
- All comments stored (available in replay)

**Bandwidth Savings**:

- **Without Sampling**: 10 TB/sec
- **With Sampling (20%)**: 2 TB/sec
- **Reduction**: 80% savings

---

## Advanced Features

### Emoji Reactions

**Challenge**: Handle 100k reactions/sec (faster than text).

**Solution**: Aggregate reactions locally, flush every 500ms.

*See pseudocode.md::aggregate_reactions() for implementation*

```
Client sends: 🔥 reaction
  ↓
WebSocket Server buffers locally
  ↓
Every 500ms: Flush aggregated counts
  ↓
Redis: HINCRBY reactions:stream:789 fire 1250
  ↓
Broadcast: {"fire": 15000, "heart": 8000}
```

**Why Aggregation?**

- **Reduces Traffic**: 100k/sec → 2 updates/sec (50,000x reduction)
- **User Experience**: Running total (doesn't need per-reaction)
- **Bandwidth Savings**: 99.99% reduction

### Pinned Comments

**Flow**:

1. **Broadcaster Pins**: `POST /stream/789/pin (comment_id=123)`
2. **Store in Redis**: `SET stream:789:pinned 123`
3. **Broadcast Event**: `COMMENT_PINNED`
4. **Client UI**: Display pinned comment at top

### Slow Mode

**Purpose**: Limit comment rate during moderation-heavy streams.

**Implementation**:

```
slow_mode_interval = 10 seconds

function allow_comment(user_id, stream_id):
  key = f"rate:user:{user_id}:stream:{stream_id}"
  last_time = redis.GET(key)
  
  if last_time and (now() - last_time) < 10:
    return false, "Slow mode: wait 10 seconds"
  
  redis.SET(key, now(), EX=10)
  return true, ""
```

---

## Availability and Fault Tolerance

### Failure Scenarios

| Failure | Impact | Mitigation | Recovery Time |
|---------|--------|-----------|---------------|
| **Single Server Down** | 1k connections lost | Clients reconnect | $< 5$ seconds |
| **Redis Pub/Sub Down** | Broadcast stops | Fallback to Kafka direct | $< 10$ seconds |
| **Kafka Partition Down** | 1/10th streams affected | Replica promotion | $< 30$ seconds |
| **Connection Registry Down** | Can't route | Broadcast to all servers | Immediate |

**Circuit Breaker Pattern**:

- If Redis fails 50% → Open circuit
- Fallback: Direct Kafka broadcast
- Recovery: Half-open after 30 seconds

### Graceful Degradation

**Scenarios**:

1. **Moderation Down**: Skip moderation, allow all comments
2. **Cassandra Write Failure**: Buffer in Kafka (7-day retention)
3. **Bandwidth Saturation**: Enable adaptive throttling (10% sample)

---

## Bottlenecks and Optimizations

### WebSocket Server CPU Bottleneck

**Problem**: 1M connections × 10 messages/sec = 10M messages/sec per server.

**Solution**: Batch message delivery.

*See pseudocode.md::batch_message_delivery() for implementation*

```
Buffer messages for 10ms, send in batch:
  - Without batching: 10M syscalls/sec
  - With batching: 100k syscalls/sec (100x reduction)
  - Latency impact: +10ms (acceptable)
```

### Redis Pub/Sub Hot Key

**Problem**: Single Redis channel becomes hot key.

**Solution**: Shard Pub/Sub channels.

```
// Instead of single channel:
stream:789:comments

// Use 100 sharded channels:
stream:789:comments:shard0
stream:789:comments:shard1
...
stream:789:comments:shard99
```

**Result**: Distributes load across 100 keys (100x reduction).

### Network Bandwidth Bottleneck

**Problem**: 10 TB/sec exceeds data center capacity.

**Optimizations**:

1. **MessagePack**: 50% smaller than JSON
2. **WebSocket Compression**: 60% reduction
3. **Delta Encoding**: Send only changed fields
4. **Adaptive Sampling**: 80% reduction at peak

**Result**: 10 TB/sec → 0.5 TB/sec (20x reduction)

---

## Common Anti-Patterns

### ❌ **1. Polling for New Comments**

**Problem**:

- Client polls every second
- 5M clients × 1 req/sec = 5M QPS
- High latency (1-second delay)

**Solution**: Use WebSockets for push-based delivery.

---

### ❌ **2. Synchronous Moderation**

**Problem**:

- Block submission until moderation completes
- ML model takes 100ms → user waits
- Poor UX during high traffic

**Solution**: Async moderation with eventual consistency.

---

### ❌ **3. Single Redis for Connections**

**Problem**:

- 5M connections in single Redis
- Single point of failure
- Memory limit (64 GB)

**Solution**: Shard connection registry across 100 Redis nodes.

---

### ❌ **4. Not Implementing Adaptive Throttling**

**Problem**:

- Allow all 10k comments/sec
- 10 TB/sec → network saturation
- Service degradation

**Solution**: Sample comments during extreme traffic.

---

## Alternative Approaches

### HTTP/2 Server-Sent Events (SSE) vs WebSockets

**SSE Pros**:

- Simpler protocol
- Easier deployment
- Built-in reconnection

**SSE Cons**:

- No bidirectional communication
- Less efficient (HTTP headers)
- 6 connection limit per domain

**Decision**: WebSockets for bidirectional efficiency.

---

### GraphQL Subscriptions vs WebSockets

**GraphQL Pros**:

- Schema-driven (type safety)
- Flexible field selection

**GraphQL Cons**:

- GraphQL overhead (query parsing)
- Complex setup (Apollo Server)
- 10x higher latency

**Decision**: Raw WebSockets for maximum performance.

---

## Monitoring and Observability

### Key Metrics

| Metric | Target | Alert Threshold | Impact |
|--------|--------|----------------|--------|
| **Comment Latency (p95)** | $< 500$ ms | $> 1$ sec | User experience degradation |
| **WebSocket Connections** | Baseline | $> 2$x baseline | Potential DDoS |
| **Redis Pub/Sub Latency** | $< 10$ ms | $> 50$ ms | Broadcast delays |
| **Kafka Consumer Lag** | $< 1$ sec | $> 10$ sec | Delayed comments |
| **Connection Drop Rate** | $< 0.1$% | $> 1$% | Network issues |

### Distributed Tracing

**OpenTelemetry Trace**:

```
Span: Comment Ingestion
  ├─ Validate (10ms)
  ├─ Publish to Kafka (5ms)
  ├─ Broadcast Fanout (50ms)
  │  ├─ Redis Publish (10ms)
  │  └─ WebSocket Broadcast (40ms)
  └─ Persist to Cassandra (20ms)

Total: 85ms
```

### Alerting Rules

**Critical**:

- Comment latency p95 > 1 second → Page on-call
- Connection drop rate > 5% → Page on-call
- Kafka consumer lag > 60 seconds → Page on-call

**Warning**:

- Redis CPU > 80% → Scale cluster
- Bandwidth usage > 80% → Enable throttling

---

## Deployment and Scaling

### Horizontal Scaling

**WebSocket Cluster**:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: websocket-servers
spec:
  minReplicas: 1000
  maxReplicas: 10000
  metrics:
    - type: Pods
      pods:
        metric:
          name: active_connections
        target:
          type: AverageValue
          averageValue: "800000"
```

**Connection Migration**: New connections to new servers, existing remain on old servers (no disruption).

### Multi-Region Deployment

**Active-Active Architecture**:

```
Region: US-EAST
  ├─ Kafka Cluster (primary)
  ├─ Redis Cluster
  ├─ WebSocket Servers (2k)
  └─ Cassandra (4 nodes)

Region: EU-WEST
  ├─ Kafka Cluster (replica)
  ├─ Redis Cluster
  ├─ WebSocket Servers (2k)
  └─ Cassandra (4 nodes)
```

**Data Replication**:

- **Kafka**: MirrorMaker 2 (100ms lag)
- **Cassandra**: Multi-datacenter (eventual)
- **Redis**: No replication (local cache)

---

## Trade-offs Summary

| What You Gain | What You Sacrifice |
|---------------|-------------------|
| ✅ **Ultra-low latency** ($< 500$ ms) | ❌ Eventual consistency (out of order) |
| ✅ **Massive fanout** (50B pushes/sec) | ❌ High bandwidth cost ($\sim \$5$M/month) |
| ✅ **Real-time moderation** | ❌ Offensive content visible 1-2 sec |
| ✅ **Horizontal scalability** | ❌ Complex connection management |
| ✅ **Adaptive throttling** | ❌ Not all comments shown at peak |
| ✅ **Persistent connections** | ❌ Higher server resources |

---

## Real-World Examples

### Twitch

**Architecture**:

- **IRC-Based Protocol**: Modified IRC over WebSocket
- **TMI**: Custom WebSocket layer
- **Sharded Servers**: 1,000+ servers globally
- **Badge System**: Subscriber/moderator prioritization

**Scale**: 10M+ concurrent, 1M+ messages/minute

---

### YouTube Live

**Architecture**:

- **gRPC Streaming**: Bidirectional streaming
- **Spanner**: Global consistency
- **CDN Delivery**: Comments via Google CDN
- **Super Chat**: Paid message highlighting

**Scale**: 30M+ concurrent (FIFA World Cup)

---

### Facebook Live

**Architecture**:

- **MQTT over WebSocket**: Lightweight pub/sub
- **TAO**: Custom graph database
- **Edge Deployment**: Chat servers at CDN edge
- **Reaction Aggregation**: Real-time emoji counts

**Scale**: 50M+ peak concurrent

---

## Performance Optimization

### Connection Keep-Alive

**Optimization**: Server pings every 60 seconds (vs 30).

**Result**: 50 MB/sec saved (5M connections × 20 bytes/ping ÷ 2)

### Redis Pipelining

**Without Pipelining**: 1 RTT per SET (slow)

**With Pipelining**: Batch 100 SETs, 1 RTT total

**Result**: 100x faster connection registration

### Message Deduplication

**Solution**: Client-side deduplication with message IDs.

```
if seen_messages.contains(message_id):
  return  // Skip duplicate

seen_messages.add(message_id)
display_comment(message)
```

---

## Security Considerations

### Authentication

**Token-Based**:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

Validate:
1. Signature (public key)
2. Expiry (iat, exp)
3. Scopes (read:comments, write:comments)
```

### Rate Limiting

| Layer | Limit | Purpose |
|-------|-------|---------|
| **Per User** | 10 comments/min | Prevent spam |
| **Per IP** | 100 comments/min | Prevent bots |
| **Per Stream** | 10k comments/sec | Protect stream |
| **Global** | 1M comments/sec | Protect infrastructure |

### DDoS Protection

**Strategies**:

1. Connection limits: Max 5 per IP
2. Geographic filtering: Block low-traffic regions
3. Behavior analysis: Detect bot patterns
4. Cloudflare Shield: Edge protection

---

## Cost Breakdown

| Component | Monthly Cost | Percentage | Notes |
|-----------|--------------|-----------|-------|
| **WebSocket Servers** | $2.5M | 50% | 5k servers @ $500 |
| **Bandwidth** | $1.5M | 30% | 2.5 EB @ $0.06/GB |
| **Kafka** | $500k | 10% | 100 nodes @ $5k |
| **Redis** | $300k | 6% | 100 nodes @ $3k |
| **Cassandra** | $200k | 4% | 50 nodes @ $4k |
| **Total** | **$5M** | **100%** | Per month peak |

**Optimization Strategies**:

1. Spot instances (70% reduction)
2. Compression (50% bandwidth savings)
3. Adaptive sampling (80% at peaks)
4. Autoscaling (40% off-peak savings)

**Optimized Cost**: $\sim \$2.5$M/month

---

## Interview Discussion Points

### Common Questions

**Question**: "How would you handle a celebrity streamer with 10M concurrent viewers?"

**Answer**: Use adaptive throttling:

- Enable sampling (show 10% of comments)
- Increase WebSocket server count from 5k to 20k
- Use dedicated Kafka partition for celebrity streams
- Pre-warm connection pool (anticipate traffic spike)

**Key Metrics**:

- 10M connections → 20k servers (500k connections/server)
- Bandwidth: 10 TB/sec → 1 TB/sec (with 90% sampling)
- Cost: $\sim \$100$k for 2-hour event (acceptable for major events)

---

**Question**: "What happens if Redis Pub/Sub fails?"

**Answer**: Fallback to Kafka direct broadcast:

- Broadcast coordinator directly writes to Kafka topic `ws-messages`
- Each WebSocket server consumes from Kafka (inefficient but functional)
- Latency degrades from 75ms to 200ms (acceptable temporarily)
- Auto-recover when Redis restores

**Trade-off**: 3x higher latency during fallback, but system remains operational.

---

**Question**: "How do you prevent spam and bot attacks?"

**Answer**: Multi-layer defense:

1. **Rate Limiting**: 10 comments/min per user
2. **CAPTCHA**: Require for new accounts
3. **Phone Verification**: Verified users have higher limits
4. **Reputation Score**: Lower limits for spam history
5. **IP-Based Limiting**: 100 comments/min per IP

**Result**: 99.9% spam reduction without impacting legitimate users.

---

**Question**: "How would you scale from 5M to 50M concurrent viewers?"

**Answer**:

| Component | 5M Viewers | 50M Viewers (10x) | Strategy |
|-----------|-----------|-------------------|----------|
| **WebSocket Servers** | 5k | 50k | Horizontal scaling |
| **Redis Pub/Sub** | 100 nodes | 1000 nodes | Shard by stream_id |
| **Kafka** | 100 partitions | 1000 partitions | Partition by stream_id |
| **Bandwidth** | 1 TB/sec | 10 TB/sec | 90% sampling by default |
| **Cost** | $\sim \$5$M/month | $\sim \$50$M/month | Optimize compression |

**Key Insight**: Bandwidth cost dominates. At 50M scale, adaptive sampling is mandatory (not optional).

---

## Design Challenge Solution

### Problem Statement

Design a live commenting system for the Super Bowl with **30M concurrent viewers** and **50k comments/sec** during key
moments.

### Requirements

1. Deliver comments within 500ms
2. Handle 50k comments/sec burst for 10 minutes
3. Support emoji reactions (100k reactions/sec)
4. Implement adaptive throttling
5. Store comments for replay

### Solution Architecture

**WebSocket Cluster Sizing**:

```
30M connections ÷ 500k connections/server = 60k servers
With overprovisioning (2x): 120k servers
Cost: 120k × $0.50/hour = $60k/hour
Total event cost: $60k × 4 hours = $240k
```

**Adaptive Throttling Logic**:

```
function super_bowl_adaptive_throttle(comment):
  current_rate = get_comment_rate()
  
  if current_rate < 5000:
    broadcast_all(comment)  // Normal traffic
  elif current_rate < 20000:
    sample_rate = 0.5  // Show 50%
    if random() < sample_rate:
      broadcast(comment)
  elif current_rate < 50000:
    sample_rate = 0.1  // Show 10%
    if random() < sample_rate:
      broadcast(comment)
  else:
    sample_rate = 0.05  // Show 5%
    if random() < sample_rate:
      broadcast(comment)
```

**Redis Pub/Sub Sharding**:

```
// 1000 shards (vs 100 normal)
for shard_id in 0..999:
  redis.publish(f"superbowl:comments:shard{shard_id}", comment)

// Each WebSocket server subscribes to 1 shard
// 60k servers ÷ 1000 shards = 60 servers per shard
```

**Emoji Reaction Aggregation**:

```
// Local aggregation every 100ms (vs 500ms normal)
buffer = {"fire": 0, "heart": 0, "touchdown": 0}

every 100ms:
  redis.HINCRBY("reactions:superbowl", "fire", buffer["fire"])
  buffer.clear()

// Broadcast aggregated total every 1 second
broadcast({"reactions": {"fire": 50000, "heart": 30000}})
```

### Final Architecture

```
┌─────────────────────────────────────────┐
│  30M Viewers (WebSocket Connections)    │
└────────────┬────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────┐
│  120k WebSocket Servers                 │
│  - 60k active, 60k standby              │
│  - 250k connections each                │
└────────────┬────────────────────────────┘
             │ Redis Pub/Sub (1000 shards)
             ↓
┌─────────────────────────────────────────┐
│  Broadcast Coordinator (1000 instances) │
│  - Adaptive throttling enabled          │
│  - 5% sampling during 50k comments/sec  │
└────────────┬────────────────────────────┘
             │ Kafka (1000 partitions)
             ↓
┌─────────────────────────────────────────┐
│  Kafka Cluster                          │
│  - 200 brokers                          │
│  - 3x replication                       │
└────────────┬────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────┐
│  Cassandra (100 nodes)                  │
│  - Stores all comments (no sampling)    │
│  - 50k writes/sec capacity              │
└─────────────────────────────────────────┘
```

### Trade-offs Accepted

**Challenge 1: Bandwidth Cost During Sampling**

**Problem**: Even with 5% sampling, bandwidth cost is $\sim \$500$k/hour.

**Solution**:

- Pre-negotiate CDN contract for event (bulk discount)
- Enable Brotli compression (additional 30% savings)
- Use MessagePack serialization (50% smaller than JSON)

**Result**: Cost reduced to $\sim \$250$k/hour.

**Challenge 2: User Experience During Heavy Sampling**

**Problem**: Users see only 5% of comments (missing 95%).

**Solution**:

- Display banner: "Chat moving extremely fast! Showing 1 in 20 comments."
- Implement "pause chat" button (user can scroll at own pace)
- Highlight important comments (verified users, high engagement)

**Result**: Users understand why not all comments visible, acceptable UX.

**Challenge 3: Connection Stability with 60k Servers**

**Problem**: Managing 60k ephemeral servers (risk of misconfiguration).

**Solution**:

- Use Kubernetes StatefulSets (predictable naming)
- Health checks every 5 seconds (automatic pod replacement)
- Pre-warm servers 1 hour before event (detect issues early)

**Result**: 99.9% connection stability during event.

### Practical Considerations

**Real-World Architecture**:

- **Pre-Event Load Testing**: Simulate 50M connections 1 week before
- **Gradual Ramp-Up**: Open connections 30 minutes before kickoff
- **Regional Distribution**: 40% US, 30% EU, 20% Asia, 10% other
- **Fallback Strategy**: If sampling not enough, increase to 2% (1 in 50)

**Scaling Strategies**:

- **Horizontal Scaling**: Auto-scale WebSocket servers based on connection count
- **Vertical Scaling**: Use high-memory instances (768 GB RAM)
- **Geographic Distribution**: Deploy in 10 regions (reduce latency)

### Final Metrics

| Aspect | Decision | Reason |
|--------|----------|--------|
| **WebSocket Cluster** | 120k servers (2x overprovision) | Handle 30M connections with redundancy |
| **Fanout Strategy** | Hybrid (Redis Pub/Sub + Kafka) | Balance latency and reliability |
| **Adaptive Sampling** | 5% during 50k comments/sec peak | Prevent bandwidth saturation (95% reduction) |
| **Redis Sharding** | 1000 shards (vs 100 normal) | Distribute Pub/Sub load |
| **Reaction Aggregation** | 100ms local buffer | Reduce reaction traffic by 99.9% |
| **Trade-off Accepted** | Not all comments shown (95% sampled) | Bandwidth cost $500k/hour → $250k/hour |

**Performance Metrics**:

- **Latency**: 650ms p95 (vs 500ms target, acceptable during peak)
- **Bandwidth**: 15 TB/sec (vs 300 TB/sec without sampling)
- **Cost**: $\sim \$1$M for 4-hour event (acceptable for Super Bowl)
- **Uptime**: 99.9% (minor connection drops acceptable)

**Why NOT Full Broadcast (No Sampling):**

- Cost: $10M/hour (10x more expensive)
- Bandwidth: 300 TB/sec (exceeds data center capacity)
- User experience: System overload → stuttering, disconnects (worse than sampling)

**When to Reconsider:**

- Bandwidth cost drops below $0.01/GB (50x cheaper)
- WebSocket server cost drops to $0.05/hour (10x cheaper)
- Network infrastructure supports 300 TB/sec sustained (unlikely)

---

## References

- **[3.2.1 Twitter Timeline](../3.2.1-twitter-timeline/README.md)** - Fanout strategies
- **[3.3.1 Live Chat System](../3.3.1-live-chat-system/README.md)** - WebSocket architecture
- **[2.0.3 Real-Time Communication](../../02-components/2.0-communication/2.0.3-real-time-communication.md)** - WebSocket
  deep dive
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3-messaging-streaming/2.3.2-kafka-deep-dive.md)** - Event streaming
- **[2.1.9 Cassandra Deep Dive](../../02-components/2.1-databases/2.1.9-cassandra-deep-dive.md)** - Time-series storage

