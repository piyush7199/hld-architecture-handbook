# Live Chat System - Quick Overview

> 📚 **Quick Revision Guide**  
> This is a condensed overview for quick revision. For comprehensive details, see **[README.md](./README.md)**.

---

## Core Concept

A real-time messaging system like WhatsApp or Slack that supports billions of users sending messages with sub-500ms latency. The system maintains persistent connections for instant message delivery, ensures messages are never lost even during network failures, preserves strict message ordering within conversations, and provides real-time presence updates (online/offline status). The solution handles both one-on-one and group chats while scaling to 100 million concurrent connections globally.

**Key Characteristics:**
- **Real-time delivery** - Sub-500ms latency via WebSocket persistent connections
- **Zero message loss** - Durable storage with Kafka replication
- **Strict ordering** - Messages within conversations delivered in correct sequence
- **Presence tracking** - Real-time online/offline status for all users
- **Massive scale** - 100M concurrent connections, 25B messages/day, 45 PB storage over 5 years
- **Group chat support** - Up to 256 members per group
- **Read receipts** - Track message status: sent, delivered, read

---

## Requirements & Scale

### Functional Requirements

| Requirement | Description | Priority |
|-------------|-------------|----------|
| **Send/Receive Messages** | Messages delivered near-instantly to online users (push notification if offline) | Must Have |
| **Message Persistence** | All messages reliably stored and retrievable (message history) | Must Have |
| **Strict Ordering** | Messages within a conversation delivered in correct sequence | Must Have |
| **Presence Service** | Real-time online/offline status for all users | Must Have |
| **Read Receipts** | Track message status: sent, delivered, read | Must Have |
| **Group Chat** | Support group conversations (up to 256 members) | Must Have |
| **Message History** | Users can retrieve past messages (years of history) | Should Have |
| **File Sharing** | Support images, videos, documents | Should Have |
| **End-to-End Encryption** | Optional E2E encryption (client-side) | Nice to Have |

### Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Low Latency** | < 500 ms (p99) | Real-time feel for users |
| **Fault Tolerance** | Zero message loss | Messages must be durable |
| **High Availability** | 99.99% uptime | Users expect always-on service |
| **Concurrent Connections** | 100M persistent connections | 10% of 1B MAU online simultaneously |
| **Message Throughput** | 10K messages/sec | Peak traffic handling |
| **Scalability** | Horizontal scaling | Add servers as users grow |

### Scale Estimation

| Metric | Assumption | Calculation | Result |
|--------|-----------|-------------|--------|
| **Total Users (MAU)** | 1 Billion monthly active users | - | 1B MAU |
| **Daily Active Users** | 50% of MAU | 1B × 0.5 | 500M DAU |
| **Concurrent Connections** | 10% of MAU online | 1B × 0.1 | 100M persistent connections |
| **Messages per Day** | 50 messages per DAU | 500M × 50 | 25 Billion messages/day |
| **Peak QPS** | 3× average | 25B / 86400 × 3 | ~870K messages/sec (peak) |
| **Average QPS** | 25B messages over 24 hours | 25B / 86400 | ~290K messages/sec |
| **Storage per Message** | 1 KB average (text + metadata) | - | 1 KB/message |
| **Daily Storage** | 25B messages × 1 KB | 25B × 1 KB | 25 TB/day |
| **5-Year Storage** | 25 TB × 365 × 5 | 25 × 365 × 5 | ~45 PB (45,000 TB) |
| **Bandwidth (Outgoing)** | 290K msg/sec × 1 KB × 2 (fanout) | 290K × 1 KB × 2 | ~580 MB/sec |

**Key Insights:**
- **100M concurrent WebSocket connections** require massive connection management infrastructure
- **45 PB storage** over 5 years requires distributed storage (Cassandra, S3)
- **870K peak QPS** requires horizontal scaling across multiple data centers
- **Strict ordering** within conversations is critical for user experience

---

## Key Components

| Component | Responsibility | Technology | Scalability |
|-----------|---------------|------------|-------------|
| **WebSocket Servers** | Maintain persistent connections, push messages to clients | Node.js, Go, Netty | Horizontal (1000 servers for 100M connections) |
| **Load Balancer** | Route clients to sticky WebSocket server | NGINX, HAProxy | Horizontal + DNS |
| **Message Router** | Validate, sequence, route messages to Kafka | Go, Java service | Horizontal (stateless) |
| **Kafka Cluster** | Ordered, durable message log (source of truth) | Apache Kafka | Horizontal (partitioned) |
| **History Workers** | Persist messages from Kafka to Cassandra | Kafka consumers | Horizontal |
| **Delivery Workers** | Read from Kafka, push to online users | Kafka consumers | Horizontal |
| **Cassandra Cluster** | Store message history (billions of messages) | Apache Cassandra | Horizontal (sharded) |
| **Redis Cluster** | Store presence data (online/offline status) | Redis | Horizontal (sharded) |
| **Sequence Manager** | Generate globally unique, ordered message IDs | ZooKeeper, etcd | HA with leader election |

---

## Architecture Flow

### Send Message Flow

**Steps:**
1. **Client sends message** via WebSocket to WS Server
2. **WS Server** forwards to Message Router
3. **Message Router:**
   - Validates sender authentication
   - Requests sequence ID from Sequence Manager
   - Wraps message with metadata (sender_id, chat_id, seq_id, timestamp)
4. **Publishes to Kafka** topic partitioned by `chat_id`
5. **Kafka persists** message (durable, ordered log)
6. **Two consumers process in parallel:**
   - **History Worker:** Writes to Cassandra for persistence
   - **Delivery Worker:** Reads and pushes to recipient's WS Server
7. **WS Server pushes** message over recipient's WebSocket
8. **Client receives** and sends ACK back

**Why Kafka?**
- ✅ **Strict Ordering:** Kafka partitions guarantee order per `chat_id`
- ✅ **Durability:** Messages replicated across brokers
- ✅ **Replayability:** History and Delivery workers can restart from any offset
- ✅ **Decoupling:** Write and read paths independent

---

## WebSocket Connection Management

**Challenge:** Maintain 100M persistent connections with sticky routing.

**Solution:**
- Each WebSocket server holds ~100K connections (OS limit: ~65K file descriptors)
- Required servers: 100M / 100K = **1000 servers**
- Sticky sessions: Load balancer routes same `user_id` to same server (hash-based)
- Heartbeat: Every 30 seconds to detect dead connections

**Connection State:**
```
user:12345:connection → {
    ws_server_id: "ws-server-42",
    socket_id: "socket-abc123",
    last_heartbeat: 1704067200,
    online: true
}
```

---

## Message Ordering & Sequence IDs

**Problem:** Ensure messages within a chat arrive in correct order globally.

**Solution: Centralized Sequence Manager**

**How It Works:**
- Sequence Manager maintains counter per `chat_id`
- Uses ZooKeeper or etcd for distributed coordination
- Generates strictly increasing sequence IDs

**Sequence ID Format (64-bit):**
```
┌─────────────────────────────────────────────────────────┐
│  41 bits: Timestamp (ms)  │  10 bits: Node ID │ 12 bits: Sequence │
└─────────────────────────────────────────────────────────┘
```

**Benefits:**
- Globally unique across all servers
- Time-ordered (sortable)
- Distributed generation (multiple Sequence Manager nodes)

**Alternative: Kafka Offset as Sequence**
- Use Kafka partition offset as sequence number
- Simpler (no external service needed)
- Trade-off: Tightly couples to Kafka implementation

---

## Presence Service (Online/Offline Status)

**Challenge:** Track online status for 1B users in real-time.

**Solution: Redis with TTL**

**Data Model:**
```
user:12345:presence → {
    status: "online",
    last_seen: 1704067200,
    ws_server: "ws-server-42"
}
TTL: 60 seconds (auto-expire if no heartbeat)
```

**How It Works:**
1. On WebSocket connect: `SET user:12345:presence "online" EX 60`
2. Heartbeat every 30s: `EXPIRE user:12345:presence 60` (refresh TTL)
3. On disconnect: `DEL user:12345:presence`
4. On TTL expire: User automatically marked offline

**Query Presence:**
```
GET user:12345:presence → "online" or NULL (offline)
```

**Optimization: Batch Presence Queries**
- Use `MGET` to fetch presence for multiple users at once
- Example: Group chat with 50 members → Single `MGET` call

---

## Read Receipts & Message Status

**Message States:**
1. **Sent:** Message sent by sender (stored in Kafka)
2. **Delivered:** Message delivered to recipient's device
3. **Read:** Recipient opened chat and viewed message

**Implementation:**

**Sent Status:**
- Automatically tracked when message written to Kafka
- Sender receives ACK from Message Router

**Delivered Status:**
- Recipient's client sends ACK after receiving message via WebSocket
- ACK published to Kafka, processed by Read Receipt Worker
- Stored in Cassandra: `message_status` table

**Read Status:**
- Recipient's client sends READ event when user views message
- Published to Kafka, processed by Read Receipt Worker
- Updated in Cassandra

**Data Model:**
```
message_status:
  message_id: uuid
  chat_id: uuid
  sender_id: uuid
  recipient_id: uuid
  sent_at: timestamp
  delivered_at: timestamp (nullable)
  read_at: timestamp (nullable)
```

---

## Group Chat vs 1-1 Chat

**1-1 Chat:**
- Simple: 2 participants
- Kafka partition key: `hash(user1_id, user2_id)`
- Direct message delivery

**Group Chat (up to 256 members):**
- Complex: N participants (fanout problem)
- Kafka partition key: `group_id`
- Fanout: Delivery Worker must push to all N members

**Fanout Strategy:**

**Small Groups (< 50 members):**
- Push to all members synchronously
- Acceptable latency

**Large Groups (50-256 members):**
- Async fanout using worker queue
- Members receive message with slight delay (100-500ms)

**Optimization: Read Receipt Aggregation**
- Store only "last read message ID" per user
- Don't store individual read receipts for large groups (too expensive)

---

## Message History Storage (Cassandra)

**Why Cassandra?**
- ✅ **Write-Heavy:** 25 TB/day of new messages
- ✅ **Horizontal Scaling:** Add nodes as data grows (45 PB over 5 years)
- ✅ **Time-Series:** Optimized for time-ordered reads (`chat_id`, `timestamp`)
- ✅ **High Availability:** Replication factor 3, no single point of failure

**Data Model:**
```
CREATE TABLE messages (
    chat_id UUID,
    timestamp TIMESTAMP,
    message_id UUID,
    sender_id UUID,
    content TEXT,
    message_type TEXT,  -- 'text', 'image', 'video', 'file'
    file_url TEXT,      -- for media messages
    PRIMARY KEY (chat_id, timestamp, message_id)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

**Query Pattern:**
```
SELECT * FROM messages
WHERE chat_id = ?
  AND timestamp >= ?
  AND timestamp <= ?
ORDER BY timestamp DESC
LIMIT 100;
```

**Benefits:**
- Partition key: `chat_id` → All messages for a chat on same node
- Clustering key: `timestamp` → Messages sorted by time
- Efficient range queries for history fetching

**Optimization: Hot Chat Cache**
- Cache last 100 messages of active chats in Redis
- 90%+ of reads served from cache (recent messages)
- Fallback to Cassandra for older history

---

## Bottlenecks & Solutions

### Bottleneck 1: WebSocket Server Connection Limits

**Problem:** Each server limited to ~100K connections due to OS file descriptor limits.

**Mitigation:**
- **Horizontal Scaling:** Add more servers (1000 servers for 100M connections)
- **OS Tuning:** Increase file descriptor limit (`ulimit -n 1000000`)
- **Use Efficient Runtime:** Go or Node.js with non-blocking I/O

**Calculation:**
```
Max connections per server: 100,000
Required connections: 100,000,000
Servers needed: 100M / 100K = 1,000 servers
```

### Bottleneck 2: Sequence Manager (Single Point)

**Problem:** Centralized Sequence Manager becomes bottleneck at high QPS.

**Mitigation:**

**Option 1: Pre-allocate ID Ranges**
- Sequence Manager pre-allocates blocks of 1000 IDs to each Message Router
- Message Router uses local IDs without calling Sequence Manager
- 99% reduction in Sequence Manager calls

**Option 2: Use Kafka Offset**
- Use Kafka partition offset as sequence number
- Eliminates need for separate Sequence Manager
- Trade-off: Couples to Kafka implementation

### Bottleneck 3: Message Delivery to Offline Users

**Problem:** Recipient offline when message sent. How to deliver later?

**Solution: Push Notifications + Message Queue**

**When Recipient Offline:**
1. Delivery Worker detects recipient offline (checks Presence Service)
2. Stores message in offline queue: `offline_messages:user:12345` (Redis List)
3. Sends push notification to recipient's device (via FCM/APNS)

**When Recipient Comes Online:**
1. Client connects via WebSocket
2. Client queries offline queue: `LRANGE offline_messages:user:12345 0 -1`
3. Server pushes all pending messages
4. Clear queue: `DEL offline_messages:user:12345`

**Optimization: Batch Notifications**
- Don't send push for every message if user receives 100 messages while offline
- Batch: "You have 100 new messages from Alice"

### Bottleneck 4: Multi-Region Consistency

**Problem:** Users in different continents. How to ensure global message ordering?

**Solution: Multi-Region Kafka with Global Sequence Manager**

**Architecture:**
```
Region US          Region EU          Region Asia
  |                  |                   |
  └────────────┬─────┴─────────┬─────────┘
            Global Sequence Manager
            (Single source of truth)
                     │
            Kafka Multi-Region Replication
```

**Trade-offs:**
- ✅ **Strict Ordering:** Single Sequence Manager ensures global order
- ❌ **Higher Latency:** Cross-region calls add 100-200ms
- ❌ **Complexity:** Kafka MirrorMaker for replication

**Alternative: Regional Ordering Only**
- Each region has independent Sequence Manager
- Messages ordered within region, but not globally
- Trade-off: Acceptable for most users (95%+ same-region)

---

## Common Anti-Patterns to Avoid

### 1. Storing Messages in Redis

❌ **Bad:** Using Redis as message database
- Memory Cost: 45 PB in Redis = $millions/year
- Volatile: Redis data lost if cluster fails (need RDB/AOF)
- Limited Query: Can't efficiently query by timestamp range

✅ **Good:** Use Cassandra for storage, Redis for hot cache
- Cassandra for durable storage (45 PB)
- Redis for caching recent messages (90%+ cache hit rate)

### 2. No Message Ordering

❌ **Bad:** Using UUID as message ID (no ordering)
```
Message 1: "Hello" (UUID: abc123)
Message 2: "How are you?" (UUID: def456)

Client receives: "How are you?" then "Hello" ❌
```

✅ **Good:** Use timestamp-based sequence IDs
- Sequence Manager generates globally ordered IDs
- Messages sorted by sequence ID, not timestamp

### 3. Synchronous Database Writes

❌ **Bad:** Writing to Cassandra synchronously before ACK
- High Latency: Cassandra write = 10-50ms
- Blocks User: Sender waits for persistence before seeing "sent"

✅ **Good:** Write to Kafka first (5ms), then async to Cassandra
- Fast user experience (5ms vs 50ms)
- Durability guaranteed by Kafka replication

---

## Monitoring & Observability

### Key Metrics

| Metric | Target | Alert Threshold | Description |
|--------|--------|-----------------|-------------|
| **Message Latency (P99)** | < 500 ms | > 1 second | End-to-end delivery time |
| **WebSocket Connections** | 100M | < 80M (capacity) | Total active connections |
| **Kafka Lag** | < 1000 messages | > 10K messages | Consumer lag behind producers |
| **Message Loss Rate** | 0% | > 0.001% | Messages not delivered |
| **Presence Update Latency** | < 100 ms | > 500 ms | Online/offline status update |
| **Cassandra Write Latency** | < 50 ms | > 200 ms | Message persistence time |

### Dashboards

**Dashboard 1: Real-Time Health**
- Active WebSocket connections per server
- Message throughput (msg/sec)
- Kafka lag per consumer group
- Presence update rate

**Dashboard 2: Message Delivery**
- Message latency distribution (P50, P99, P999)
- Delivery success rate
- Read receipt delivery time
- Offline message queue size

**Critical Alerts:**
- Message latency > 1s for > 1 minute → Check Kafka lag, network issues
- Kafka lag > 10K messages → Scale consumer workers
- WebSocket connections < 80M → Check server health, load balancer
- Message loss rate > 0.001% → Critical: Check Kafka replication, Cassandra writes

---

## Trade-offs Summary

| Decision | Choice | Alternative | Why Chosen | Trade-off |
|----------|--------|-------------|------------|-----------|
| **Connection Protocol** | WebSocket | Long Polling | Full-duplex, low latency | Stateful servers, sticky sessions |
| **Message Broker** | Kafka | RabbitMQ | Strict ordering, durability | Higher complexity |
| **Storage** | Cassandra | PostgreSQL | Horizontal scaling, 45 PB | Eventual consistency |
| **Presence** | Redis | Database | Sub-millisecond lookups | Volatile (TTL-based) |
| **Ordering** | Centralized Sequence Manager | Distributed Timestamps | Global ordering | Single point of failure risk |
| **Failure Mode** | Fail-Open | Fail-Close | API availability | Temporary message loss possible |

### What We Gained

✅ **Real-Time:** Sub-500ms message delivery via WebSockets
✅ **Ordered:** Strict message ordering within conversations via Kafka
✅ **Durable:** Zero message loss via Kafka replication
✅ **Scalable:** 100M concurrent connections, 45 PB storage
✅ **Available:** 99.99% uptime via distributed architecture

### What We Sacrificed

❌ **Complexity:** Kafka, Cassandra, Redis all required (high ops burden)
❌ **Cost:** 1000 WebSocket servers + Kafka cluster + Cassandra cluster = expensive
❌ **Eventual Consistency:** Read receipts, presence may lag slightly
❌ **Stateful Servers:** WebSocket servers require sticky sessions
❌ **Multi-Region Latency:** Global ordering requires cross-region coordination (100-200ms)

---

## Real-World Implementations

### WhatsApp
- **Architecture:** Erlang servers for WebSocket connections (2M connections per server)
- **Message Broker:** Custom XMPP-based protocol + Kafka-like log
- **Storage:** Cassandra for message history
- **Scale:** 2B users, 100B messages/day
- **Key Insight:** End-to-end encryption implemented client-side (Signal Protocol)

### Slack
- **Architecture:** Node.js WebSocket servers
- **Message Broker:** Kafka for message log
- **Storage:** MySQL (sharded) for messages, Redis for presence
- **Scale:** 10M+ DAU, billions of messages
- **Key Insight:** Heavy use of Redis for caching recent messages (90%+ cache hit rate)

### Discord
- **Architecture:** Elixir/Erlang for WebSocket connections
- **Message Broker:** Custom message queue (Go-based)
- **Storage:** Cassandra for history, Redis for hot data
- **Scale:** 150M MAU, 4B messages/day
- **Key Insight:** Voice chat requires different optimization (UDP vs TCP)

---

## Key Takeaways

1. **WebSockets** are essential for sub-500ms real-time messaging (vs long polling)
2. **Kafka** provides strict ordering per chat via partitioning
3. **Cassandra** scales horizontally to 45 PB for message history
4. **Redis** enables sub-millisecond presence lookups (TTL-based)
5. **Sequence Manager** ensures globally ordered message IDs
6. **Sticky sessions** required for WebSocket connection routing
7. **Fanout strategy** differs for small vs large groups
8. **Offline delivery** via push notifications + message queue
9. **Hot chat cache** in Redis reduces Cassandra load (90%+ cache hit rate)
10. **Fail-open policy** ensures API availability even if rate limiter fails

---

## Recommended Stack

- **WebSocket Servers:** Node.js, Go, or Netty (non-blocking I/O)
- **Load Balancer:** NGINX or HAProxy (sticky sessions)
- **Message Broker:** Apache Kafka (ordered, durable log)
- **Storage:** Apache Cassandra (horizontal scaling, 45 PB)
- **Presence:** Redis Cluster (sub-millisecond lookups)
- **Sequence Manager:** ZooKeeper or etcd (distributed coordination)
- **Monitoring:** Prometheus + Grafana (metrics, dashboards)

---

## See Also

- **[README.md](./README.md)** - Comprehensive design document (full details)
- **[hld-diagram.md](./hld-diagram.md)** - Architecture diagrams (system, component design, data flow)
- **[sequence-diagrams.md](./sequence-diagrams.md)** - Detailed interaction flows (send message, receive message, presence updates)
- **[this-over-that.md](./this-over-that.md)** - In-depth design decisions (why this over that)
- **[pseudocode.md](./pseudocode.md)** - Algorithm implementations (WebSocket management, message ordering, presence)

