# Live Commenting - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made in designing the Live Commenting
system, including trade-offs, alternatives considered, and rationale behind each choice.

---

## Table of Contents

1. [WebSocket vs HTTP/2 Server-Sent Events vs HTTP Polling](#1-websocket-vs-http2-server-sent-events-vs-http-polling)
2. [Redis Pub/Sub vs Kafka Direct Broadcast vs Message Queue](#2-redis-pubsub-vs-kafka-direct-broadcast-vs-message-queue)
3. [Async Moderation vs Synchronous Moderation](#3-async-moderation-vs-synchronous-moderation)
4. [Adaptive Throttling vs Full Broadcast](#4-adaptive-throttling-vs-full-broadcast)
5. [Connection Registry vs Broadcast to All Servers](#5-connection-registry-vs-broadcast-to-all-servers)
6. [MessagePack vs JSON Serialization](#6-messagepack-vs-json-serialization)
7. [Cassandra vs PostgreSQL for Comment Storage](#7-cassandra-vs-postgresql-for-comment-storage)
8. [Two-Tier Fanout vs Single-Tier Fanout](#8-two-tier-fanout-vs-single-tier-fanout)
9. [Local Reaction Aggregation vs Instant Broadcast](#9-local-reaction-aggregation-vs-instant-broadcast)
10. [Fanout-on-Write vs Fanout-on-Read](#10-fanout-on-write-vs-fanout-on-read)

---

## 1. WebSocket vs HTTP/2 Server-Sent Events vs HTTP Polling

### The Problem

Need to deliver comments to millions of viewers in real-time. Three approaches exist:

- **WebSocket**: Bidirectional persistent connection
- **HTTP/2 Server-Sent Events (SSE)**: One-way server-to-client push
- **HTTP Polling**: Client repeatedly requests updates

### Options Considered

| Approach | Pros | Cons | Performance | Cost |
|----------|------|------|-------------|------|
| **WebSocket (This Over That)** | • Bidirectional<br/>• Low latency (50ms)<br/>• No connection limits<br/>• Efficient (no HTTP headers) | • Complex protocol<br/>• Connection management overhead | • Latency: 50ms<br/>• Overhead: 2 bytes/frame | • Moderate (5k servers) |
| **HTTP/2 SSE** | • Simpler protocol<br/>• Built-in reconnection<br/>• Easy to deploy | • One-way only (need separate POST)<br/>• 6 connection limit per domain<br/>• HTTP headers overhead | • Latency: 100ms<br/>• Overhead: 100 bytes/request | • Higher (10k servers) |
| **HTTP Polling** | • Simplest to implement<br/>• Works with any HTTP client | • High latency (1-5 seconds)<br/>• Wasted bandwidth (empty responses)<br/>• High server load | • Latency: 1-5 seconds<br/>• Overhead: 500 bytes/request | • Very high (50k servers) |

### Decision Made

**WebSocket** chosen for bidirectional efficiency and no connection limits.

### Rationale

1. **Bidirectional Communication**: Comments sent from client AND received from server on same connection
   - **SSE**: Requires separate POST endpoint for comments (2x complexity)
   - **Polling**: Requires separate API endpoints (more complex)

2. **No Connection Limits**: WebSocket has no per-domain connection limit
   - **SSE**: Browser limits to 6 connections per domain (would need subdomain sharding)
   - **Polling**: No limits but wasteful

3. **Low Latency**: 50ms vs 100ms (SSE) vs 1-5 seconds (polling)
   - **WebSocket**: Persistent connection, no handshake per message
   - **SSE**: HTTP/2 multiplexing helps but still slower
   - **Polling**: Round-trip latency on every request

4. **Efficiency**: 2 bytes overhead vs 100 bytes (SSE) vs 500 bytes (polling)
   - **WebSocket**: Binary frames, minimal overhead
   - **SSE**: HTTP headers on every message
   - **Polling**: Full HTTP request/response

5. **Real-World Validation**: Used by Twitch, YouTube Live, Facebook Live
   - Twitch: 10M+ concurrent WebSocket connections
   - YouTube: gRPC bidirectional streaming (similar to WebSocket)
   - Facebook: MQTT over WebSocket

### Implementation Details

**WebSocket Configuration**:

```javascript
// Client-side connection
const ws = new WebSocket('wss://chat.example.com/stream/789');

// Send comment
ws.send(JSON.stringify({type: 'comment', message: 'Great stream!'}));

// Receive comments
ws.onmessage = (event) => {
  const comment = JSON.parse(event.data);
  displayComment(comment);
};
```

**Connection Management**:

- **Heartbeat**: Ping/pong every 30 seconds
- **Reconnection**: Automatic exponential backoff
- **Compression**: Enable permessage-deflate (60% reduction)

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Low Latency**: 50ms vs 100ms (SSE) | ❌ **Complexity**: Need connection management, reconnection logic |
| ✅ **Efficiency**: 2 bytes overhead vs 100 bytes | ❌ **Protocol Overhead**: WebSocket handshake, frame management |
| ✅ **Bidirectional**: Single connection for send/receive | ❌ **Operational Overhead**: Monitor connection health, handle failures |
| ✅ **No Limits**: Unlimited connections | ❌ **Memory Cost**: 4 KB per connection (20 GB for 5M connections) |

### When to Reconsider

- **Browser Support**: If SSE becomes as efficient as WebSocket (unlikely)
- **Connection Limits Removed**: If browsers remove 6-connection limit for SSE
- **New Protocol**: If HTTP/3 QUIC introduces better bidirectional streaming

---

## 2. Redis Pub/Sub vs Kafka Direct Broadcast vs Message Queue

### The Problem

Need to deliver one comment to 5,000 WebSocket servers efficiently. Three approaches:

- **Redis Pub/Sub**: Lightweight pub/sub with automatic fanout
- **Kafka Direct Broadcast**: Each server consumes from Kafka directly
- **Message Queue (RabbitMQ)**: Traditional message queue with routing

### Options Considered

| Approach | Pros | Cons | Performance | Scalability |
|----------|------|------|-------------|-------------|
| **Redis Pub/Sub (This Over That)** | • Low latency (<10ms)<br/>• Automatic fanout<br/>• Simple API<br/>• Decouples servers | • Not persistent<br/>• Hot key risk (mitigated by sharding) | • Latency: 10ms<br/>• Throughput: 1M msgs/sec | • 100 shards handles 5k servers |
| **Kafka Direct Broadcast** | • Persistent<br/>• Exactly-once semantics<br/>• Replay capability | • Higher latency (50ms)<br/>• 5k consumers per partition (overwhelming)<br/>• Complex rebalancing | • Latency: 50ms<br/>• Throughput: 100k msgs/sec | • Kafka can't handle 5k consumers |
| **RabbitMQ** | • Durable queues<br/>• Routing flexibility | • Higher latency (20ms)<br/>• Complex routing setup<br/>• Lower throughput | • Latency: 20ms<br/>• Throughput: 50k msgs/sec | • Limited to 1k consumers |

### Decision Made

**Redis Pub/Sub** chosen for low latency and automatic fanout.

### Rationale

1. **Low Latency**: <10ms internal routing vs 50ms (Kafka) vs 20ms (RabbitMQ)
   - **Redis**: In-memory, optimized for pub/sub
   - **Kafka**: Disk-based, higher latency
   - **RabbitMQ**: Disk-based, queue overhead

2. **Automatic Fanout**: Redis handles message duplication automatically
   - **Redis**: One PUBLISH → all subscribers receive (no application logic)
   - **Kafka**: Need consumer groups, partition management
   - **RabbitMQ**: Need exchange routing, queue setup

3. **Decoupling**: WebSocket servers don't need to know about each other
   - **Redis**: Servers just subscribe to channel
   - **Kafka**: Servers need partition assignment, coordination
   - **RabbitMQ**: Servers need queue binding, routing keys

4. **Scalability**: 5,000 servers subscribing to one channel
   - **Redis**: Handles 1M messages/sec per channel (with sharding)
   - **Kafka**: Can't handle 5k consumers per partition (limit: 100)
   - **RabbitMQ**: Limited to 1k consumers per queue

5. **Simplicity**: Single PUBLISH command broadcasts to all
   ```redis
   PUBLISH stream:789:comments "comment_message"
   ```

### Implementation Details

**Redis Pub/Sub Sharding**:

```go
// Broadcast coordinator publishes to all shards
for shard_id := 0; shard_id < 100; shard_id++ {
    channel := fmt.Sprintf("stream:%d:comments:shard%d", stream_id, shard_id)
    redis.Publish(channel, comment)
}

// WebSocket server subscribes to one shard
shard_id := hash(server_id) % 100
channel := fmt.Sprintf("stream:%d:comments:shard%d", stream_id, shard_id)
redis.Subscribe(channel)
```

**Fallback Strategy**:

If Redis Pub/Sub fails, fallback to Kafka direct broadcast:
- Write to Kafka `ws-messages` topic
- Each WebSocket server consumes from Kafka
- Higher latency (200ms) but functional

### Trade-offs Accepted

| What We Gain | What You Sacrifice |
|--------------|-------------------|
| ✅ **Low Latency**: 10ms vs 50ms (Kafka) | ❌ **Persistence**: Messages lost if Redis crashes (mitigated by Kafka retention) |
| ✅ **Automatic Fanout**: No application logic | ❌ **Hot Key Risk**: Single channel can become bottleneck (mitigated by sharding) |
| ✅ **Simplicity**: Single PUBLISH command | ❌ **No Exactly-Once**: Messages may be duplicated (acceptable for comments) |
| ✅ **Scalability**: 5k servers easily | ❌ **Memory Cost**: Redis uses RAM (mitigated by sharding) |

### When to Reconsider

- **Persistence Required**: If we need guaranteed delivery (use Kafka with consumer groups)
- **Exactly-Once Semantics**: If duplicate comments are unacceptable (use Kafka)
- **Redis Limits**: If Redis Pub/Sub can't handle scale (use Kafka with partitions)

---

## 3. Async Moderation vs Synchronous Moderation

### The Problem

Comments need moderation (spam/hate speech detection) but must be delivered quickly. Two approaches:

- **Async Moderation**: Publish immediately, moderate in background, delete if needed
- **Synchronous Moderation**: Wait for moderation before publishing

### Options Considered

| Approach | Pros | Cons | User Experience | Performance |
|----------|------|------|-----------------|-------------|
| **Async Moderation (This Over That)** | • Instant user feedback<br/>• Scalable (horizontal workers)<br/>• Fault-tolerant (retries) | • Offensive content visible 1-2 seconds<br/>• Eventual consistency | • User sees success in 25ms<br/>• Comment visible immediately | • Publish: 25ms<br/>• Moderation: 50-100ms (background) |
| **Synchronous Moderation** | • No offensive content shown<br/>• Strong consistency | • User waits 100ms+<br/>• Poor UX<br/>• Scaling bottleneck | • User waits 100-200ms<br/>• Comment appears after moderation | • Publish: 100-200ms<br/>• Moderation: Blocks publish |

### Decision Made

**Async Moderation** chosen for user experience and scalability.

### Rationale

1. **User Experience**: User sees success in 25ms vs 100-200ms
   - **Async**: Optimistic UI, immediate feedback
   - **Sync**: User waits for AI model (poor UX)

2. **Scalability**: Moderation workers scale independently
   - **Async**: 100 workers process 10k comments/sec
   - **Sync**: API servers blocked during moderation (bottleneck)

3. **Fault Tolerance**: Failed moderation jobs can be retried
   - **Async**: Job re-queued automatically, user unaware
   - **Sync**: User sees error, must retry upload

4. **Acceptable Trade-off**: Offensive content visible 1-2 seconds
   - **Real-World**: 99% of comments are clean (low false positive rate)
   - **User Impact**: Minimal (deletion happens quickly)
   - **Business Impact**: Better UX outweighs brief visibility

5. **Industry Standard**: Used by Twitter, Facebook, YouTube
   - Twitter: Tweets visible immediately, moderation async
   - Facebook: Posts visible, moderation async
   - YouTube: Comments visible, moderation async

### Implementation Details

**Async Flow**:

```
Comment Published → Kafka → Immediate Broadcast
                    ↓
              Moderation Worker (async)
                    ↓
              ML Model (50-100ms)
                    ↓
              If REJECT → Publish Delete Event
```

**Delete Event**:

```json
{
  "type": "comment.deleted",
  "comment_id": 12345,
  "reason": "high_toxicity",
  "timestamp": 1699123456
}
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **User Experience**: 25ms vs 100-200ms | ❌ **Eventual Consistency**: Offensive content visible 1-2 seconds |
| ✅ **Scalability**: Independent worker scaling | ❌ **Complexity**: Need event sourcing for deletions |
| ✅ **Fault Tolerance**: Automatic retries | ❌ **False Negatives**: 1% of offensive content may slip through |
| ✅ **Performance**: 10k comments/sec | ❌ **Delete Latency**: 100-200ms before deletion |

### When to Reconsider

- **Zero Tolerance**: If any offensive content visible is unacceptable (use sync)
- **Regulatory Requirements**: If law requires pre-moderation (use sync)
- **High Toxicity Rate**: If >10% of comments are toxic (use sync)

---

## 4. Adaptive Throttling vs Full Broadcast

### The Problem

During peak traffic (10k comments/sec), full broadcast consumes 10 TB/sec bandwidth. Need to handle extreme scale.

### Options Considered

| Approach | Pros | Cons | Bandwidth | Cost |
|----------|------|------|-----------|------|
| **Adaptive Throttling (This Over That)** | • Prevents overload<br/>• 80-95% bandwidth savings<br/>• System stability<br/>• Users notified | • Not all comments shown<br/>• UX impact (minimal) | • Peak: 2 TB/sec (80% reduction)<br/>• Normal: 1 TB/sec | • $1.5M → $300k/month |
| **Full Broadcast** | • All comments shown<br/>• Simple logic | • System overload risk<br/>• 10x bandwidth cost<br/>• Service degradation | • Peak: 10 TB/sec<br/>• Normal: 1 TB/sec | • $1.5M → $15M/month |
| **Fixed Rate Limiting** | • Predictable bandwidth<br/>• Simple | • Unfair (drops later comments)<br/>• Poor UX | • Fixed: 5k comments/sec | • $1.5M/month |

### Decision Made

**Adaptive Throttling** chosen to prevent system overload while maintaining acceptable UX.

### Rationale

1. **System Stability**: Prevents bandwidth saturation and service degradation
   - **Adaptive**: System remains operational during peaks
   - **Full Broadcast**: Risk of network saturation, disconnects

2. **Cost Efficiency**: 80-95% bandwidth savings during peak
   - **Adaptive**: $300k/month vs $15M/month (50x cheaper)
   - **Full Broadcast**: Unsustainable at scale

3. **User Experience**: Transparent sampling with user notification
   - **Adaptive**: Users see banner "Chat moving fast! Showing 1 in 10 comments"
   - **Full Broadcast**: System stutters, disconnects (worse UX)

4. **Scalability**: Handles extreme events (Super Bowl: 30M viewers, 50k comments/sec)
   - **Adaptive**: System scales to 100x normal traffic
   - **Full Broadcast**: Requires 10x infrastructure (expensive)

5. **Industry Standard**: Used by Twitch, YouTube Live, Facebook Live
   - Twitch: Samples chat during extreme traffic
   - YouTube: Throttles comments during viral streams
   - Facebook: Adaptive sampling for live events

### Implementation Details

**Throttling Logic**:

```go
func adaptiveThrottle(commentRate int) float64 {
    switch {
    case commentRate < 5000:
        return 1.0  // 100% broadcast
    case commentRate < 20000:
        return 0.5  // 50% broadcast
    case commentRate < 50000:
        return 0.1  // 10% broadcast
    default:
        return 0.05  // 5% broadcast
    }
}
```

**User Notification**:

```json
{
  "type": "throttle_notification",
  "sample_rate": 0.1,
  "message": "Chat moving fast! Showing 1 in 10 comments. Pause to read all."
}
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **System Stability**: Prevents overload | ❌ **Incomplete Visibility**: Not all comments shown (95% sampled at peak) |
| ✅ **Cost Savings**: 80-95% bandwidth reduction | ❌ **UX Impact**: Users may miss comments (mitigated by pause feature) |
| ✅ **Scalability**: Handles 100x traffic spikes | ❌ **Complexity**: Need sampling logic, user notifications |
| ✅ **Transparency**: Users notified about sampling | ❌ **Perception**: Users may think system is "broken" (mitigated by clear messaging) |

### When to Reconsider

- **Bandwidth Cost Drops**: If bandwidth becomes free (use full broadcast)
- **Network Capacity Increases**: If infrastructure supports 10 TB/sec (use full broadcast)
- **User Complaints**: If users demand all comments (may need to accept higher cost)

---

## 5. Connection Registry vs Broadcast to All Servers

### The Problem

Need to route messages to specific users (bans, direct messages). Two approaches:

- **Connection Registry**: Track which server holds each user's connection
- **Broadcast to All**: Send message to all servers, let them filter

### Options Considered

| Approach | Pros | Cons | Performance | Efficiency |
|----------|------|------|-------------|------------|
| **Connection Registry (This Over That)** | • Targeted routing<br/>• Efficient (no broadcast)<br/>• Enables bans, DMs | • Lookup overhead (1ms)<br/>• Registry size (1 GB)<br/>• Cleanup complexity | • Latency: 75ms (targeted)<br/>• Overhead: 1ms lookup | • 1 message to 1 server |
| **Broadcast to All** | • Simple (no registry)<br/>• No lookup overhead | • Inefficient (5k servers receive)<br/>• Can't target specific users<br/>• Wasted bandwidth | • Latency: 150ms (broadcast)<br/>• Overhead: 0ms lookup | • 1 message to 5k servers |

### Decision Made

**Connection Registry** chosen for efficiency and targeted operations.

### Rationale

1. **Efficiency**: 1 message to 1 server vs 1 message to 5k servers
   - **Registry**: Direct routing, no waste
   - **Broadcast**: 5k servers receive message, 4,999 discard it

2. **Targeted Operations**: Enable bans, direct messages, moderation actions
   - **Registry**: Can disconnect specific user, send DM
   - **Broadcast**: Can't target specific users

3. **Bandwidth Savings**: 5,000x reduction in targeted messages
   - **Registry**: 1 message sent
   - **Broadcast**: 5,000 messages sent (99.98% waste)

4. **Scalability**: Registry lookup is fast (1ms Redis GET)
   - **Registry**: 1ms overhead acceptable
   - **Broadcast**: 150ms latency (unnecessary for targeted ops)

5. **Real-World Need**: Bans, moderation, direct messages are common
   - **Registry**: Supports all targeted operations
   - **Broadcast**: Can't support targeted operations

### Implementation Details

**Registry Structure**:

```redis
Key: conn:user:{user_id}
Type: Hash
Fields:
  stream_id: 789
  ws_server: "ws-server-42.example.com"
  connected_at: 1699123456
TTL: 2 hours
```

**Lookup and Route**:

```go
// Get user's server
server := redis.HGet("conn:user:123", "ws_server")

// Route message directly
wsServers[server].SendMessage(user_id, message)
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Efficiency**: 5,000x reduction in targeted messages | ❌ **Lookup Overhead**: 1ms Redis query per targeted operation |
| ✅ **Targeted Operations**: Bans, DMs, moderation | ❌ **Registry Size**: 1 GB for 5M connections (acceptable) |
| ✅ **Bandwidth Savings**: 99.98% reduction | ❌ **Complexity**: Need registry cleanup, connection migration |
| ✅ **Low Latency**: 75ms vs 150ms (broadcast) | ❌ **Registry Failure**: If Redis down, can't route (fallback to broadcast) |

### When to Reconsider

- **No Targeted Operations**: If we never need bans/DMs (use broadcast)
- **Registry Overhead High**: If Redis lookup becomes bottleneck (use local cache)
- **Simpler Architecture**: If operational complexity is too high (use broadcast)

---

## 6. MessagePack vs JSON Serialization

### The Problem

Comments need efficient serialization to reduce bandwidth. Two approaches:

- **MessagePack**: Binary serialization (50% smaller)
- **JSON**: Text-based serialization (human-readable)

### Options Considered

| Approach | Pros | Cons | Size | Parse Time |
|----------|------|------|------|------------|
| **MessagePack (This Over That)** | • 50% smaller<br/>• Faster parsing<br/>• Binary format | • Not human-readable<br/>• Requires library | • 20 bytes (50% reduction) | • 0.5ms |
| **JSON** | • Human-readable<br/>• Standard (no library)<br/>• Easy to debug | • Larger size<br/>• Slower parsing | • 40 bytes | • 1ms |
| **Protocol Buffers** | • 60% smaller<br/>• Type-safe<br/>• Fast parsing | • Schema required<br/>• Complex setup | • 16 bytes | • 0.3ms |

### Decision Made

**MessagePack** chosen for size and performance benefits.

### Rationale

1. **Bandwidth Savings**: 50% reduction in message size
   - **MessagePack**: 20 bytes per comment
   - **JSON**: 40 bytes per comment
   - **Savings**: 500 MB/sec at 10k comments/sec (significant)

2. **Parse Performance**: 2x faster parsing
   - **MessagePack**: 0.5ms parse time
   - **JSON**: 1ms parse time
   - **Impact**: 0.5ms saved per message (5 seconds saved at 10k/sec)

3. **Cost Impact**: 50% bandwidth reduction = $750k/month savings
   - **MessagePack**: $1.5M/month bandwidth
   - **JSON**: $3M/month bandwidth
   - **Savings**: $1.5M/month (significant)

4. **WebSocket Compatibility**: WebSocket supports binary frames
   - **MessagePack**: Native binary support
   - **JSON**: Text-based, less efficient

5. **Industry Standard**: Used by Slack, Discord, Twitch
   - Slack: MessagePack for real-time messaging
   - Discord: MessagePack for WebSocket
   - Twitch: MessagePack for chat

### Implementation Details

**MessagePack Encoding**:

```go
// Encode comment
comment := Comment{
    UserID:    123,
    Message:   "Great stream!",
    Timestamp: 1699123456,
}
data, _ := msgpack.Marshal(comment)
// Size: 20 bytes

// Decode comment
var comment Comment
msgpack.Unmarshal(data, &comment)
```

**JSON Equivalent**:

```json
{
  "user_id": 123,
  "message": "Great stream!",
  "timestamp": 1699123456
}
// Size: 40 bytes
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Bandwidth Savings**: 50% reduction ($1.5M/month) | ❌ **Human Readability**: Can't read messages in logs (use logging library) |
| ✅ **Parse Performance**: 2x faster (0.5ms vs 1ms) | ❌ **Library Dependency**: Need MessagePack library (minimal) |
| ✅ **Cost Efficiency**: $1.5M/month savings | ❌ **Debugging**: Harder to debug (use JSON for logs) |
| ✅ **WebSocket Efficiency**: Binary frames | ❌ **Compatibility**: Not all tools support MessagePack (acceptable) |

### When to Reconsider

- **Bandwidth Cost Drops**: If bandwidth becomes free (use JSON for readability)
- **Debugging Priority**: If human readability is critical (use JSON)
- **Tool Support**: If monitoring tools don't support MessagePack (use JSON for logs)

---

## 7. Cassandra vs PostgreSQL for Comment Storage

### The Problem

Need to store comment history (time-series data) with high write throughput. Two approaches:

- **Cassandra**: NoSQL database optimized for time-series writes
- **PostgreSQL**: SQL database with ACID guarantees

### Options Considered

| Database | Pros | Cons | Write Performance | Cost |
|----------|------|------|-------------------|------|
| **Cassandra (This Over That)** | • Excellent write performance<br/>• Time-series optimized<br/>• Horizontal scaling<br/>• Open-source | • Eventual consistency<br/>• No complex queries<br/>• Operational overhead | • 10k writes/sec per node<br/>• 60 nodes = 600k writes/sec | • $30k/month (60 nodes) |
| **PostgreSQL** | • ACID guarantees<br/>• Rich queries (JOINs)<br/>• Mature ecosystem | • Poor write performance<br/>• Vertical scaling limits<br/>• Complex sharding | • 1k writes/sec per node<br/>• 600 nodes = 600k writes/sec | • $150k/month (600 nodes) |
| **MongoDB** | • Flexible schema<br/>• Good write performance | • Less mature for time-series<br/>• Higher cost | • 5k writes/sec per node<br/>• 120 nodes = 600k writes/sec | • $60k/month (120 nodes) |

### Decision Made

**Cassandra** chosen for write performance and time-series optimization.

### Rationale

1. **Write Performance**: 10x better write throughput
   - **Cassandra**: 10k writes/sec per node (LSM tree architecture)
   - **PostgreSQL**: 1k writes/sec per node (B-tree, ACID overhead)
   - **Impact**: Need 60 nodes vs 600 nodes (10x cost difference)

2. **Time-Series Optimization**: Native support for time-ordered queries
   - **Cassandra**: CLUSTERING ORDER BY timestamp DESC (efficient)
   - **PostgreSQL**: Need indexes, less efficient for time-series
   - **Query**: `SELECT * FROM comments WHERE stream_id = 789 ORDER BY created_at DESC LIMIT 100` (fast in Cassandra)

3. **Cost Efficiency**: 5x cheaper than PostgreSQL
   - **Cassandra**: 60 nodes × $500/month = $30k/month
   - **PostgreSQL**: 600 nodes × $250/month = $150k/month
   - **Savings**: $120k/month (significant)

4. **Horizontal Scaling**: Easy to add nodes
   - **Cassandra**: Add nodes, data redistributes automatically
   - **PostgreSQL**: Complex sharding, manual rebalancing

5. **Eventual Consistency**: Acceptable for comments
   - **Comments**: Read-heavy, eventual consistency OK
   - **Metadata**: Use PostgreSQL for ACID requirements (streams, users)

### Implementation Details

**Cassandra Schema**:

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

**Query Pattern**:

```sql
-- Get recent comments (fast, time-ordered)
SELECT * FROM comments
WHERE stream_id = 789
LIMIT 100;
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Write Performance**: 10x better (10k vs 1k writes/sec) | ❌ **ACID Guarantees**: Eventual consistency (acceptable for comments) |
| ✅ **Cost Efficiency**: 5x cheaper ($30k vs $150k/month) | ❌ **Complex Queries**: No JOINs (use PostgreSQL for metadata) |
| ✅ **Time-Series Optimization**: Native support | ❌ **Operational Overhead**: Need Cassandra expertise |
| ✅ **Horizontal Scaling**: Easy node addition | ❌ **Consistency**: May see stale data (acceptable for comments) |

### When to Reconsider

- **ACID Required**: If comments need strong consistency (use PostgreSQL)
- **Complex Queries**: If need JOINs with users/streams (use PostgreSQL)
- **Low Write Rate**: If write rate < 1k/sec (PostgreSQL sufficient)

---

## 8. Two-Tier Fanout vs Single-Tier Fanout

### The Problem

Need to deliver one comment to 5,000 WebSocket servers. Two approaches:

- **Two-Tier**: Kafka → Redis Pub/Sub → WebSocket servers
- **Single-Tier**: Kafka → WebSocket servers directly

### Options Considered

| Approach | Pros | Cons | Latency | Complexity |
|----------|------|------|---------|------------|
| **Two-Tier (This Over That)** | • Low latency (10ms Redis)<br/>• Decoupling<br/>• Automatic fanout | • Two hops<br/>• More components | • Total: 75ms<br/>• Kafka: 10ms<br/>• Redis: 10ms<br/>• WS: 50ms | • Moderate (2 tiers) |
| **Single-Tier** | • Simpler (one hop)<br/>• Fewer components | • Higher latency (50ms Kafka)<br/>• Kafka overwhelmed<br/>• Complex consumer groups | • Total: 100ms<br/>• Kafka: 50ms<br/>• WS: 50ms | • High (Kafka complexity) |

### Decision Made

**Two-Tier Fanout** chosen for low latency and decoupling.

### Rationale

1. **Low Latency**: 75ms vs 100ms (25% faster)
   - **Two-Tier**: Kafka (10ms) + Redis (10ms) + WS (50ms) = 75ms
   - **Single-Tier**: Kafka (50ms) + WS (50ms) = 100ms
   - **Impact**: 25ms saved per comment (significant at scale)

2. **Kafka Scalability**: Kafka can't handle 5k consumers per partition
   - **Two-Tier**: Kafka → 100 coordinators → Redis → 5k servers (efficient)
   - **Single-Tier**: Kafka → 5k servers directly (overwhelming)
   - **Limit**: Kafka supports max 100 consumers per partition

3. **Decoupling**: WebSocket servers don't depend on Kafka
   - **Two-Tier**: Servers subscribe to Redis (simple)
   - **Single-Tier**: Servers must manage Kafka consumer groups (complex)

4. **Fault Isolation**: Redis failure doesn't affect Kafka
   - **Two-Tier**: Can fallback to Kafka direct (degraded)
   - **Single-Tier**: Kafka failure = complete outage

5. **Operational Simplicity**: Easier to scale and debug
   - **Two-Tier**: Scale Redis and Kafka independently
   - **Single-Tier**: Scale Kafka only (bottleneck)

### Implementation Details

**Two-Tier Architecture**:

```
Tier 1: Kafka → Broadcast Coordinator (100 instances)
  - Kafka partition: stream_id mod 100
  - Coordinator reads from Kafka
  - Publishes to Redis Pub/Sub

Tier 2: Redis Pub/Sub → WebSocket Servers (5k instances)
  - Redis channel: stream:789:comments
  - Servers subscribe to channel
  - Automatic fanout to all subscribers
```

**Single-Tier Alternative**:

```
Kafka → WebSocket Servers (5k instances)
  - Each server consumes from Kafka
  - Consumer group: websocket-servers
  - Problem: Kafka can't handle 5k consumers
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Low Latency**: 75ms vs 100ms (25% faster) | ❌ **Complexity**: Two tiers require coordination |
| ✅ **Scalability**: Kafka handles 100 consumers, not 5k | ❌ **Components**: Two systems to manage (Kafka + Redis) |
| ✅ **Decoupling**: Servers don't depend on Kafka | ❌ **Failure Points**: Two systems can fail (mitigated by fallback) |
| ✅ **Fault Isolation**: Redis failure doesn't affect Kafka | ❌ **Operational Overhead**: Monitor two systems |

### When to Reconsider

- **Kafka Scale**: If Kafka can handle 5k consumers (use single-tier)
- **Simpler Architecture**: If operational complexity too high (use single-tier)
- **Lower Latency Needed**: If 75ms is too slow (optimize Redis, not remove tier)

---

## 9. Local Reaction Aggregation vs Instant Broadcast

### The Problem

Emoji reactions arrive at 100k/sec. Need to handle efficiently. Two approaches:

- **Local Aggregation**: Buffer reactions locally, flush every 500ms
- **Instant Broadcast**: Broadcast each reaction immediately

### Options Considered

| Approach | Pros | Cons | Bandwidth | User Experience |
|----------|------|------|-----------|-----------------|
| **Local Aggregation (This Over That)** | • 99.99% bandwidth reduction<br/>• System stability<br/>• Running total (better UX) | • 500ms delay<br/>• Per-reaction timing lost | • 2 updates/sec (50k reduction) | • Running total (good) |
| **Instant Broadcast** | • Real-time updates<br/>• Simple logic | • 100k messages/sec<br/>• Bandwidth overload<br/>• Spam in UI | • 100k updates/sec | • Reaction spam (bad UX) |

### Decision Made

**Local Aggregation** chosen for bandwidth efficiency and better UX.

### Rationale

1. **Bandwidth Savings**: 99.99% reduction (100k/sec → 2/sec)
   - **Aggregation**: 2 updates/sec (aggregated counts)
   - **Instant**: 100k updates/sec (individual reactions)
   - **Savings**: 50,000x reduction (massive)

2. **User Experience**: Running total is better than spam
   - **Aggregation**: "🔥 15,000" (clean, readable)
   - **Instant**: Hundreds of reactions flashing (spam, unreadable)

3. **System Stability**: Prevents reaction overload
   - **Aggregation**: 2 messages/sec (manageable)
   - **Instant**: 100k messages/sec (overwhelming)

4. **Acceptable Delay**: 500ms delay is acceptable for reactions
   - **Aggregation**: 500ms before update (users don't notice)
   - **Instant**: 0ms (no benefit for reactions)

5. **Industry Standard**: Used by Facebook, YouTube, Twitch
   - Facebook: Aggregates reactions (running total)
   - YouTube: Aggregates likes (running total)
   - Twitch: Aggregates emote reactions (running total)

### Implementation Details

**Local Aggregation**:

```go
// Buffer reactions locally
buffer := map[string]int{
    "fire": 0,
    "heart": 0,
    "touchdown": 0,
}

// Every 500ms, flush to Redis
ticker := time.NewTicker(500 * time.Millisecond)
for range ticker.C {
    for reaction, count := range buffer {
        redis.HINCRBY(fmt.Sprintf("reactions:stream:%d", stream_id), reaction, count)
    }
    buffer = make(map[string]int)  // Clear buffer
}
```

**Broadcast Update**:

```json
{
  "type": "reactions_update",
  "reactions": {
    "fire": 15000,
    "heart": 8000,
    "touchdown": 5000
  }
}
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Bandwidth Savings**: 99.99% reduction (50k reduction) | ❌ **Latency**: 500ms delay before update (acceptable) |
| ✅ **User Experience**: Running total (better than spam) | ❌ **Granularity**: Lose per-reaction timing (not needed) |
| ✅ **System Stability**: Prevents overload | ❌ **Complexity**: Need aggregation logic, buffering |
| ✅ **Cost Efficiency**: Massive bandwidth savings | ❌ **Real-Time Feel**: Not instant (acceptable for reactions) |

### When to Reconsider

- **Real-Time Required**: If reactions must be instant (use instant broadcast)
- **Low Reaction Rate**: If < 100 reactions/sec (instant broadcast acceptable)
- **User Complaints**: If users want per-reaction updates (rare)

---

## 10. Fanout-on-Write vs Fanout-on-Read

### The Problem

When a comment is published, need to deliver to 5M viewers. Two approaches:

- **Fanout-on-Write**: Pre-compute and push to all viewers immediately
- **Fanout-on-Read**: Fetch comments when user requests feed

### Options Considered

| Approach | Pros | Cons | Latency | Write Amplification |
|----------|------|------|---------|---------------------|
| **Fanout-on-Write (This Over That)** | • Ultra-fast reads (50ms)<br/>• Pre-computed feeds<br/>• Guaranteed delivery | • Write amplification (1 → 5M)<br/>• High write load | • Read: 50ms<br/>• Write: 50ms | • 1 comment → 5M writes |
| **Fanout-on-Read** | • No write amplification<br/>• Always fresh<br/>• Simple writes | • Slow reads (500ms+)<br/>• High read load<br/>• Database bottleneck | • Read: 500ms+<br/>• Write: 20ms | • 1 comment → 1 write |

### Decision Made

**Fanout-on-Write** chosen for ultra-low latency.

### Rationale

1. **Ultra-Low Latency**: 50ms vs 500ms+ (10x faster)
   - **Fanout-on-Write**: Pre-computed, instant delivery
   - **Fanout-on-Read**: Query database, slow
   - **Impact**: 450ms saved (critical for real-time feel)

2. **User Experience**: Real-time feel requires instant delivery
   - **Fanout-on-Write**: Comment appears instantly (50ms)
   - **Fanout-on-Read**: Comment appears after query (500ms+)
   - **Perception**: 500ms feels "laggy" for live chat

3. **Database Load**: Write amplification acceptable vs read amplification
   - **Fanout-on-Write**: 5M writes (distributed across Redis)
   - **Fanout-on-Read**: 5M reads (database bottleneck)
   - **Redis**: Handles 5M writes/sec easily (distributed)
   - **Database**: Can't handle 5M reads/sec (single bottleneck)

4. **Guaranteed Delivery**: Pre-computed ensures all viewers see comment
   - **Fanout-on-Write**: Comment pushed to all viewers
   - **Fanout-on-Read**: Viewers may miss if query fails

5. **Industry Standard**: Used by all live chat systems
   - Twitch: Fanout-on-write for chat
   - YouTube: Fanout-on-write for live chat
   - Facebook: Fanout-on-write for live comments

### Implementation Details

**Fanout-on-Write Flow**:

```
Comment Published → Kafka
                    ↓
            Broadcast Coordinator
                    ↓
            Redis Pub/Sub (automatic fanout)
                    ↓
        5,000 WebSocket Servers (subscribe)
                    ↓
        5M WebSocket Connections (push)
```

**Write Amplification**:

```
1 comment → 1 Kafka write
          → 1 Redis Pub/Sub publish
          → 5,000 WebSocket server subscriptions
          → 5M WebSocket connections (push)
```

### Trade-offs Accepted

| What We Gain | What We Sacrifice |
|--------------|-------------------|
| ✅ **Ultra-Low Latency**: 50ms vs 500ms+ (10x faster) | ❌ **Write Amplification**: 1 comment → 5M writes (acceptable) |
| ✅ **User Experience**: Real-time feel (instant delivery) | ❌ **Write Load**: High write throughput (Redis handles it) |
| ✅ **Database Efficiency**: Writes distributed, not reads | ❌ **Complexity**: Need fanout orchestration |
| ✅ **Guaranteed Delivery**: All viewers see comment | ❌ **Resource Usage**: More memory for pre-computed feeds |

### When to Reconsider

- **Low Viewer Count**: If < 100 viewers (fanout-on-read acceptable)
- **High Write Rate**: If write rate > 100k/sec (may need hybrid)
- **Read-Heavy**: If reads are more important than writes (use fanout-on-read)

---

## Summary Comparison Table

| Decision | Chosen Approach | Alternative | Key Reason | Trade-off |
|----------|----------------|-------------|------------|-----------|
| **Real-Time Protocol** | WebSocket | HTTP/2 SSE | Bidirectional + no limits | Connection management complexity |
| **Fanout Mechanism** | Redis Pub/Sub | Kafka Direct | Low latency + automatic fanout | Not persistent (mitigated by Kafka) |
| **Moderation Strategy** | Async | Synchronous | User experience + scalability | Offensive content visible 1-2 sec |
| **Throttling Strategy** | Adaptive | Full Broadcast | System stability + cost | Not all comments shown |
| **Connection Routing** | Registry | Broadcast All | Efficiency + targeted ops | Lookup overhead |
| **Serialization** | MessagePack | JSON | 50% bandwidth savings | Not human-readable |
| **Comment Storage** | Cassandra | PostgreSQL | Write performance + cost | Eventual consistency |
| **Fanout Architecture** | Two-Tier | Single-Tier | Low latency + scalability | More components |
| **Reaction Delivery** | Aggregation | Instant | 99.99% bandwidth savings | 500ms delay |
| **Fanout Timing** | Fanout-on-Write | Fanout-on-Read | Ultra-low latency | Write amplification |

---

This document captures the architectural decision-making process with detailed analysis of alternatives, trade-offs, and
conditions under which decisions should be reconsidered.

