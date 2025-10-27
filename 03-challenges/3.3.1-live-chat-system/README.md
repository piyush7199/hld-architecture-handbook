# 3.3.1 Design a Live Chat System (WhatsApp/Slack)

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

Design a real-time messaging system like WhatsApp or Slack that supports billions of users sending messages with
sub-500ms latency. The system must maintain persistent connections for instant message delivery, ensure messages are
never lost even during network failures, preserve strict message ordering within conversations, and provide real-time
presence updates (online/offline status). The solution must handle both one-on-one and group chats while scaling to
100 million concurrent connections globally.

---

## 1. Requirements and Scale Estimation

### Functional Requirements (FRs)

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

### Non-Functional Requirements (NFRs)

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
| **Daily Active Users** | 50% of MAU | $1\text{B} \times 0.5$ | 500M DAU |
| **Concurrent Connections** | 10% of MAU online | $1\text{B} \times 0.1$ | 100M persistent connections |
| **Messages per Day** | 50 messages per DAU | $500\text{M} \times 50$ | 25 Billion messages/day |
| **Peak QPS** | 3× average | $25\text{B} / 86400 \times 3$ | ~870K messages/sec (peak) |
| **Average QPS** | 25B messages over 24 hours | $25\text{B} / 86400$ | ~290K messages/sec |
| **Storage per Message** | 1 KB average (text + metadata) | - | 1 KB/message |
| **Daily Storage** | 25B messages × 1 KB | $25\text{B} \times 1\text{KB}$ | 25 TB/day |
| **5-Year Storage** | 25 TB × 365 × 5 | $25 \times 365 \times 5$ | ~45 PB (45,000 TB) |
| **Bandwidth (Outgoing)** | 290K msg/sec × 1 KB × 2 (fanout) | $290\text{K} \times 1\text{KB} \times 2$ | ~580 MB/sec |

**Key Insights:**
- **100M concurrent WebSocket connections** require massive connection management infrastructure
- **45 PB storage** over 5 years requires distributed storage (Cassandra, S3)
- **870K peak QPS** requires horizontal scaling across multiple data centers
- **Strict ordering** within conversations is critical for user experience

---

## 2. High-Level Architecture

> 📊 **See detailed architecture diagrams:** [HLD Diagrams](./hld-diagram.md)

### System Overview

```
                    ┌──────────────────────────────────┐
                    │    Client Applications            │
                    │  (Mobile, Web, Desktop)           │
                    └────────────┬─────────────────────┘
                                 │ WebSocket
                    ┌────────────▼─────────────────────┐
                    │      Load Balancer               │
                    │  (Sticky Session by user_id)     │
                    └────────────┬─────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
   ┌──────────▼─────────┐ ┌─────▼──────────┐ ┌───▼──────────┐
   │ WebSocket Server 1 │ │ WS Server 2    │ │ WS Server N  │
   │ (100K connections) │ │ (100K conns)   │ │ (100K conns) │
   └──────────┬─────────┘ └────────┬───────┘ └───┬──────────┘
              │                     │              │
              └─────────────────────┼──────────────┘
                                    │
                    ┌───────────────▼──────────────┐
                    │   Message Router / Gateway   │
                    │  - Validate user             │
                    │  - Assign sequence ID        │
                    │  - Route to Kafka            │
                    └───────────────┬──────────────┘
                                    │
                    ┌───────────────▼──────────────┐
                    │     Kafka Cluster            │
                    │  (Message Log / Event Store) │
                    │  - Partitioned by chat_id    │
                    │  - Guarantees ordering       │
                    └─────┬────────────────┬───────┘
                          │                │
           ┌──────────────▼──┐      ┌─────▼──────────────┐
           │ History Worker  │      │ Delivery Worker    │
           │ (Kafka Consumer)│      │ (Kafka Consumer)   │
           └──────────┬──────┘      └─────┬──────────────┘
                      │                    │
           ┌──────────▼──────┐      ┌─────▼──────────────┐
           │  Cassandra      │      │ Back to WS Servers │
           │  (Message DB)   │      │ (Push to receiver) │
           │  - chat_id      │      └────────────────────┘
           │  - timestamp    │
           └─────────────────┘
                      │
           ┌──────────▼──────┐
           │  Redis          │
           │  (Presence)     │
           │  user:123:online│
           └─────────────────┘
```

### Key Components

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

## 3. Detailed Component Design

### 3.1 WebSocket Connection Management

**Challenge:** Maintain 100M persistent connections with sticky routing.

**Solution:**
- Each WebSocket server holds ~100K connections (OS limit: ~65K file descriptors)
- Required servers: $100\text{M} / 100\text{K} = 1000$ servers
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

*See [pseudocode.md::maintain_websocket_connection()](pseudocode.md) for implementation.*

---

### 3.2 Message Flow Architecture

#### Send Message Flow

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

> 📊 **See detailed flow:** [Sequence Diagrams](./sequence-diagrams.md)

**Why Kafka?**
- ✅ **Strict Ordering:** Kafka partitions guarantee order per `chat_id`
- ✅ **Durability:** Messages replicated across brokers
- ✅ **Replayability:** History and Delivery workers can restart from any offset
- ✅ **Decoupling:** Write and read paths independent

*See [pseudocode.md::send_message_flow()](pseudocode.md) for implementation.*

---

### 3.3 Kafka Partitioning Strategy

**Goal:** Ensure strict message ordering within each conversation while enabling parallel processing.

**Strategy:** Partition messages by `chat_id`

```
partition_number = hash(chat_id) % NUM_PARTITIONS
```

**Benefits:**
- All messages for chat "ABC" go to partition 7
- Partition 7 guarantees sequential ordering
- Different chats processed in parallel across partitions
- Recommended: **1000 partitions** for 870K QPS = ~870 msg/sec per partition

**Kafka Configuration:**
- Partitions: 1000
- Replication Factor: 3 (durability)
- Retention: 7 days (replayability)

*See [this-over-that.md](this-over-that.md) for comparison with alternatives.*

---

### 3.4 Data Model

#### Messages Table (Cassandra)

Primary table for storing all message history.

```sql
CREATE TABLE messages (
    chat_id UUID,
    seq_id BIGINT,
    sender_id UUID,
    recipient_id UUID,
    content TEXT,
    message_type VARCHAR(20),  -- 'text', 'image', 'video'
    timestamp TIMESTAMP,
    deleted BOOLEAN DEFAULT false,
    deleted_at TIMESTAMP,
    edited_at TIMESTAMP,
    PRIMARY KEY ((chat_id), seq_id)
) WITH CLUSTERING ORDER BY (seq_id DESC);
```

**Why Cassandra over PostgreSQL?**

| Factor | Cassandra | PostgreSQL |
|--------|-----------|------------|
| **Write Throughput** | 290K writes/sec ✅ | ~10K writes/sec ❌ |
| **Storage** | 45 PB easily ✅ | Sharding complex for TB+ ❌ |
| **Horizontal Scaling** | Add nodes easily ✅ | Manual sharding ❌ |
| **Time-Series** | Optimized ✅ | Suboptimal ❌ |
| **Cost** | $2-3/GB/month | $10-15/GB/month ❌ |

*See [this-over-that.md: Cassandra vs PostgreSQL](this-over-that.md) for detailed analysis.*

#### Message Status Table (Cassandra)

Track delivery and read receipts.

```sql
CREATE TABLE message_status (
    message_id BIGINT,
    chat_id UUID,
    user_id UUID,
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    read_at TIMESTAMP,
    PRIMARY KEY ((message_id), user_id)
);
```

#### User Presence (Redis)

Store online/offline status with automatic expiration.

```
Key: user:{user_id}:presence
Value: "online"
TTL: 60 seconds
```

**Presence Logic:**
- Set key to "online" with 60-second TTL on connection
- Heartbeat every 30 seconds refreshes TTL
- If no heartbeat, key expires → user automatically offline
- Query: `GET user:12345:presence` returns "online" or NULL

*See [pseudocode.md::presence_service()](pseudocode.md) for implementation.*

---

### 3.5 Sequence ID Generation

**Goal:** Generate globally unique, time-ordered message IDs for strict ordering.

**Algorithm:** Snowflake-style 64-bit IDs

```
[41 bits: Timestamp] [10 bits: Node ID] [13 bits: Sequence]
```

**Bit Breakdown:**
- **41 bits timestamp:** Milliseconds since epoch (69 years range)
- **10 bits node ID:** Supports 1024 nodes (0-1023)
- **13 bits sequence:** 8192 IDs per millisecond per node

**Throughput per node:**
- 8192 IDs per millisecond
- 8.192 million IDs per second per node
- 1000 nodes = 8.192 billion IDs/second

**Generation Logic:**
```
function generate_sequence_id():
    timestamp = current_timestamp_milliseconds()
    node_id = THIS_NODE_ID  // Assigned by ZooKeeper
    sequence = increment_sequence_for_current_millisecond()
    
    if sequence > 8191:
        wait_for_next_millisecond()
        sequence = 0
    
    id = (timestamp << 23) | (node_id << 13) | sequence
    return id
```

**Properties:**
- IDs are roughly sortable by time
- Globally unique (no collisions)
- No coordination needed (except node ID assignment)

*See [pseudocode.md::generate_sequence_id()](pseudocode.md) for full implementation.*

---

### 3.6 Presence Service

**Challenge:** Track online/offline status for 1B users with sub-millisecond lookup latency.

**Solution: Redis with TTL-based Presence**

**Data Model:**
```
Key: user:{user_id}:presence
Value: "online"
TTL: 60 seconds
```

**Operations:**

1. **Set Online (on connection):**
   ```
   SET user:12345:presence "online" EX 60
   ```

2. **Heartbeat (every 30 seconds):**
   ```
   EXPIRE user:12345:presence 60
   ```

3. **Check Presence:**
   ```
   GET user:12345:presence
   → "online" (if present) or NULL (if offline)
   ```

4. **Batch Check (for group chat):**
   ```
   MGET user:12345:presence user:67890:presence user:99999:presence
   → ["online", NULL, "online"]
   ```

**Why Redis over Database?**

| Factor | Redis | PostgreSQL |
|--------|-------|------------|
| **Latency** | <1ms ✅ | 5-20ms ❌ |
| **Throughput** | 1M ops/sec ✅ | 10K ops/sec ❌ |
| **TTL Support** | Native ✅ | Manual cleanup ❌ |
| **Sharding** | Easy (20 shards) ✅ | Complex ❌ |

**Scaling:**
- 100M online users × 100 bytes/user = 10 GB data
- 20 Redis shards = 500 MB per shard
- Shard key: `hash(user_id) % 20`

*See [pseudocode.md::presence_operations()](pseudocode.md) for implementation.*

---

### 3.7 Read Receipts

**Message States:**

1. **SENT ✓:** Message written to Kafka (sender confirmation)
2. **DELIVERED ✓✓:** Message pushed to recipient's device (gray checkmark)
3. **READ ✓✓:** Recipient viewed message (blue checkmark)

**Implementation Flow:**

1. **Sender sends message:**
   - Message written to Kafka
   - Sender immediately sees "SENT ✓" status

2. **Recipient receives message:**
   - Delivery worker pushes to recipient's WebSocket
   - Recipient's app sends ACK
   - Update `message_status.delivered_at = now()`
   - Notify sender: "DELIVERED ✓✓"

3. **Recipient views message:**
   - Client detects message in viewport for >2 seconds
   - Client sends READ event via WebSocket
   - Publish READ event to Kafka
   - Read Receipt Worker updates `message_status.read_at = now()`
   - Notify sender: "READ ✓✓" (blue checkmark)

**Group Chat Optimization:**

For groups > 50 members, tracking individual read receipts is expensive.

**Solution:**
- Track only "last read message ID" per user
- Query: "Show me unread count" = `last_message_id - user_last_read_id`

*See [sequence-diagrams.md: Read Receipt Flow](sequence-diagrams.md) for detailed flow.*

---

### 3.8 Group Chat

**Challenge:** Deliver message to 256 members efficiently (fanout problem).

**Fanout Strategies:**

#### Small Groups (< 50 members): Synchronous Fanout

**Flow:**
1. Message published to Kafka (single write)
2. Delivery worker reads message
3. Lookup all 50 members
4. Check presence for each (batch query)
5. Push to all online members in parallel (10 goroutines)
6. Queue offline members with push notification

**Latency:** ~100-200ms for all members

#### Large Groups (50-256 members): Asynchronous Fanout

**Flow:**
1. Message published to Kafka
2. Sender gets immediate "SENT" ACK
3. Background fanout workers process in batches
4. Push to online members over 500ms-1s

**Trade-off:**
- Faster sender experience (25ms ACK)
- Slight delay for recipients in large groups (acceptable)

**Read Receipt Handling:**
- Small groups: Track all read receipts
- Large groups: Track only "last read message ID" per user (too expensive to track 256 receipts per message)

*See [pseudocode.md::fanout_group_message()](pseudocode.md) for implementation.*

---

### 3.9 Offline Message Handling

**Challenge:** User offline when message arrives.

**Solution: Offline Queue + Push Notification**

**Flow:**

1. **Delivery worker checks presence:**
   ```
   GET user:67890:presence → NULL (offline)
   ```

2. **Add to offline queue (Redis List):**
   ```
   LPUSH offline_messages:67890 {message_json}
   EXPIRE offline_messages:67890 604800  // 7 days
   ```

3. **Send push notification (FCM/APNS):**
   - Single message: "Alice: Hello!"
   - Multiple messages (>10): "You have 47 new messages"
   - Avoid spamming with 47 separate notifications

4. **User comes online:**
   ```
   LRANGE offline_messages:67890 0 -1  // Fetch all
   → [message1, message2, ..., message47]
   ```

5. **Push all messages via WebSocket**

6. **Clear queue:**
   ```
   DEL offline_messages:67890
   ```

**Edge Cases:**
- Queue > 10K messages: Fetch in batches of 100
- User never comes back online: 7-day TTL auto-cleanup
- Multiple devices: Each fetches and clears queue (idempotent)

*See [pseudocode.md::offline_message_queue()](pseudocode.md) for implementation.*

---

## 4. Why This Over That?

> 📊 **Full analysis:** [Design Decisions (This Over That)](./this-over-that.md)

### Decision 1: WebSockets vs Long Polling vs SSE

**Chosen:** WebSockets

**Why:**
- ✅ **Full-duplex:** Bidirectional communication (send + receive)
- ✅ **Low latency:** <50ms (persistent connection)
- ✅ **Efficient:** No HTTP overhead per message
- ✅ **Real-time:** Instant push to client

**Alternatives:**
- ❌ **Long Polling:** 1-5s latency, inefficient (new request per message)
- ❌ **SSE (Server-Sent Events):** One-way only (server → client), browser connection limits

### Decision 2: Kafka vs Database for Ordering

**Chosen:** Kafka as Message Log

**Why:**
- ✅ **Strict Ordering:** Partitions guarantee sequential order
- ✅ **Durability:** Replicated across brokers (zero message loss)
- ✅ **Decoupling:** Write path (sender) and read path (receiver) independent
- ✅ **Replayability:** Can reprocess from any offset (recovery)
- ✅ **High Throughput:** 290K writes/sec easily

**Alternatives:**
- ❌ **PostgreSQL:** Can't handle 290K writes/sec, no ordering guarantee
- ❌ **Cassandra:** No built-in change log, no ordering within partition

### Decision 3: Cassandra vs PostgreSQL vs MongoDB

**Chosen:** Cassandra for Message Storage

**Why:**
- ✅ **Write-heavy:** 290K writes/sec, 25 TB/day
- ✅ **Horizontal Scaling:** Add nodes for 45 PB over 5 years
- ✅ **Time-series Optimized:** Query by `(chat_id, timestamp)`
- ✅ **Cost-effective:** $2-3/GB/month vs $10-15 for RDBMS

**Alternatives:**
- ❌ **PostgreSQL:** Manual sharding for TB-scale, write bottleneck
- ❌ **MongoDB:** Not optimized for time-series, higher cost

### Decision 4: Redis vs Memcached for Presence

**Chosen:** Redis

**Why:**
- ✅ **TTL Support:** Native expiration (automatic offline)
- ✅ **Data Structures:** HASH, LIST, SET for complex presence
- ✅ **Persistence:** AOF/RDB for recovery
- ✅ **Pub/Sub:** For typing indicators

**Alternatives:**
- ❌ **Memcached:** No TTL per key, no persistence, simpler data types

---

## 5. Bottlenecks and Future Scaling

### Bottleneck 1: Connection Management (100M Connections)

**Problem:**
- Each server limited to ~100K connections (OS file descriptors)
- Sticky sessions create uneven load

**Solutions:**
1. **Increase servers:** 1000 WebSocket servers for 100M connections
2. **OS tuning:** Increase `ulimit -n 1000000` (file descriptors)
3. **Consistent hashing:** Evenly distribute users across servers
4. **Connection pooling:** Reuse connections for multiplexing

### Bottleneck 2: Sequence Manager (Single Point of Failure)

**Problem:**
- All message requests hit Sequence Manager
- 290K QPS → bottleneck
- Single point of failure

**Solutions:**
1. **Pre-allocate ID ranges:** Each server gets batch of 1000 IDs
   - Reduces Sequence Manager calls by 99%
   - 290K QPS → 290 QPS to Sequence Manager
2. **Alternative: Use Kafka offset as sequence ID**
   - No Sequence Manager needed
   - Trade-off: Couples to Kafka implementation

*See [pseudocode.md::preallocate_id_range()](pseudocode.md) for implementation.*

### Bottleneck 3: Hot Chat Rooms

**Problem:**
- Celebrity group chat with 1M messages/day
- Single Kafka partition can't handle load

**Solutions:**
1. **Split by time:** Partition by `(chat_id, date)`
   - Each day gets new partition
2. **Caching:** Hot messages cached in Redis
3. **Read replicas:** Multiple consumers for same partition

### Bottleneck 4: Cross-Region Latency

**Problem:**
- US user → EU user = +100-200ms latency (across Atlantic)
- Single Sequence Manager in US = +50ms for EU

**Solutions:**
1. **Regional Kafka clusters:** Each region has local Kafka
2. **Cross-region replication:** Async replication between regions
3. **Global Sequence Manager:** Single source of truth (accept +50ms)
4. **Alternative: Regional IDs:** Each region generates IDs (no global ordering)

---

## 6. Common Anti-Patterns

### ❌ Anti-Pattern 1: Using Redis as Primary Message Database

**Why Bad:**
- 45 PB in Redis = $$millions per year
- Redis is volatile (lost on crash without persistence)
- Memory-only storage not cost-effective for archives

**✅ Best Practice:**
- Cassandra for persistent storage ($2-3/GB/month)
- Redis for hot cache (last 100 messages per chat)

### ❌ Anti-Pattern 2: No Message Ordering Guarantee

**Why Bad:**
- UUIDs have no time order
- Client receives: "I'm here" before "Where are you?"
- Terrible UX

**✅ Best Practice:**
- Snowflake IDs (timestamp-based)
- Kafka partitioning by `chat_id`

### ❌ Anti-Pattern 3: Synchronous Database Writes on Send

**Why Bad:**
- Sender waits 10-50ms for Cassandra write
- Blocks fast ACK to user

**✅ Best Practice:**
- Write to Kafka first (5ms)
- Immediate ACK to sender
- Async workers write to Cassandra

### ❌ Anti-Pattern 4: No Message Deduplication

**Why Bad:**
- Network retries cause duplicate messages
- User sees "Hello!" twice

**✅ Best Practice:**
- Client generates `request_id` (UUID)
- Server caches processed `request_id` for 5 minutes
- Duplicate requests return cached response

*See [pseudocode.md::deduplicate_message()](pseudocode.md) for implementation.*

---

## 7. Alternative Approaches

### Alternative 1: Fanout-on-Read (Pull Model)

**Architecture:**
- Don't push messages to users
- User pulls messages on demand (polling or WebSocket query)

**When to Use:**
- Very large groups (>1000 members)
- Broadcast channels (1M subscribers)
- Example: Twitter celebrity tweets

**Pros:**
- ✅ No fanout cost on write
- ✅ Scales to millions of followers

**Cons:**
- ❌ Higher read latency (must query on demand)
- ❌ More complex client logic

### Alternative 2: Hybrid Fanout

**Architecture:**
- Small chats: Fanout-on-Write (push)
- Large groups: Fanout-on-Read (pull)

**When to Use:**
- Mix of small chats and large groups
- Example: Slack (channels + DMs)

---

## 8. Monitoring and Observability

### Critical Metrics

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| **Message Latency (P99)** | < 500ms | > 1000ms 🔴 |
| **Active Connections** | 100M | < 95M or > 105M 🟡 |
| **Kafka Consumer Lag** | < 1000 msgs | > 10K msgs 🔴 |
| **Message Loss Rate** | 0% | > 0.01% 🔴 |
| **WebSocket Server Health** | 99.99% | Server down 🔴 |
| **Cassandra Write Latency** | < 50ms | > 100ms 🟡 |

### Dashboards

1. **Real-Time Overview:**
   - Active connections (line chart)
   - Messages/sec (line chart)
   - Latency distribution (histogram)

2. **Kafka Health:**
   - Consumer lag per partition
   - Broker health
   - Replication lag

3. **Storage Health:**
   - Cassandra write/read latency
   - Disk usage (45 PB capacity)
   - Redis memory usage

### Alerts

**🔴 Critical (Page On-Call):**
- Kafka consumer lag > 10K messages
- WebSocket server down (100K users affected)
- Message loss detected
- Sequence Manager unavailable

**🟡 Warning (Investigate Next Day):**
- Cassandra latency > 100ms
- Redis memory > 90%
- Connection count > 105M (capacity planning)

*See [hld-diagram.md: Monitoring Dashboard](hld-diagram.md) for visualization.*

---

## 9. Trade-offs Summary

### What We Gained ✅

| Benefit | Explanation |
|---------|-------------|
| **Real-time Delivery** | <500ms end-to-end via WebSockets |
| **Strict Ordering** | Kafka partitions guarantee sequence |
| **Zero Message Loss** | Kafka durability + Cassandra persistence |
| **Horizontal Scaling** | Add servers/nodes as users grow |
| **High Availability** | 99.99% uptime with multi-AZ |

### What We Sacrificed ❌

| Trade-off | Impact |
|-----------|--------|
| **Complexity** | Kafka + Cassandra + Redis + WS servers |
| **Cost** | 1000 servers + massive clusters = $$ |
| **Eventual Consistency** | Read receipts delayed by 100-500ms |
| **Stateful Servers** | Sticky sessions complicate load balancing |
| **Multi-Region Latency** | +100-200ms for cross-continent messages |

---

## 10. Real-World Implementations

### WhatsApp

**Architecture:**
- **Erlang** for WebSocket servers (2M connections per server!)
- **Kafka-like** message log
- **Cassandra** for message storage
- **2 billion users**, **100 billion messages/day**

**Key Insight:** Erlang's lightweight processes enable 2M connections per server vs 100K with Go/Node.js.

### Slack

**Architecture:**
- **Node.js** WebSocket servers
- **Kafka** for message log
- **MySQL (sharded)** + **Vitess** for storage
- **Redis** for presence and caching
- **10M+ DAU**, billions of messages

**Key Insight:** MySQL with Vitess (sharding layer) instead of Cassandra for strong consistency.

### Discord

**Architecture:**
- **Elixir/Erlang** for real-time
- **Custom Go-based** message queue (not Kafka)
- **Cassandra** for message storage
- **ScyllaDB** (Cassandra-compatible) for performance
- **150M MAU**, **4 billion messages/day**

**Key Insight:** Custom message queue optimized for their specific use case (gaming-focused features).

---

## 11. Interview Discussion Points

### Q1: Why Kafka Over Direct Database Writes?

**Answer:**
- **Ordering:** Kafka partitions guarantee sequential order per `chat_id`
- **Decoupling:** Write path (sender) and read path (receiver) are independent
- **Durability:** Replicated across brokers (zero message loss)
- **Throughput:** Can handle 290K writes/sec easily
- **Replayability:** Can reprocess messages from any offset (disaster recovery)

### Q2: How to Handle 100 Million Concurrent Connections?

**Answer:**
1. **1000 WebSocket servers** (100K connections each)
2. **Sticky sessions:** Load balancer routes same `user_id` to same server
3. **OS tuning:** Increase file descriptors (`ulimit -n 1000000`)
4. **Heartbeat:** 30-second intervals to detect dead connections
5. **Horizontal scaling:** Add servers as needed

### Q3: What If User Is in Group with 1000 Members?

**Answer:**
- **Fanout-on-Read** instead of Fanout-on-Write
- User queries: "Give me last 100 messages for group_id"
- No push to 1000 members on every message
- Trade-off: Slightly higher latency (query on demand)
- Example: Slack public channels, Twitter

### Q4: How to Ensure Messages Aren't Lost?

**Answer:**
1. **Kafka Durability:** Replication factor 3 (message on 3 brokers)
2. **Cassandra Replication:** RF=3 across nodes
3. **Client-Side Retry:** If no ACK in 5 seconds, retry with same `request_id`
4. **Deduplication:** Server ignores duplicate `request_id`
5. **Monitoring:** Alert on message loss (should be 0%)

### Q5: Multi-Tenancy: How to Isolate Large Enterprise Customers?

**Answer:**
1. **Dedicated Kafka partitions:** Enterprise customers get separate partitions
2. **Dedicated Cassandra keyspace:** Logical isolation
3. **Rate limiting per tenant:** Prevent one customer from overwhelming system
4. **Resource quotas:** CPU/memory limits per tenant
5. **Separate clusters:** Very large enterprises (e.g., Fortune 500) get dedicated clusters

---

## 12. Cost Analysis (AWS Example)

### Infrastructure Costs

| Component | Specification | Monthly Cost |
|-----------|---------------|--------------|
| **WebSocket Servers** | 1000 × c5.2xlarge (8 vCPU, 16 GB) | $200K |
| **Kafka Cluster** | 50 brokers × r5.4xlarge | $120K |
| **Cassandra Cluster** | 100 nodes × i3.4xlarge (16 TB SSD each) | $500K |
| **Redis Cluster** | 20 nodes × r5.2xlarge | $40K |
| **Load Balancers** | 10 × Network LB | $2K |
| **Bandwidth** | 580 MB/sec × 2.5 PB/month | $225K |
| **Total** | | **~$1.1M/month** |

**Annual Cost:** ~$13M/year for 1B MAU

**Cost per User:** $13/year or $1.08/month per MAU

### Cost Optimization Strategies

1. **Reserved Instances:** Save 30-50% on compute
2. **Spot Instances:** Use for non-critical workers (History Workers)
3. **S3 Archive:** Move messages >1 year old to S3 Glacier ($0.004/GB)
4. **Compression:** Gzip messages (50% reduction)
5. **Multi-Region:** Only deploy in high-user-density regions

---

## Summary

**Live Chat System** is a highly complex, real-time distributed system requiring:

- **100M WebSocket connections** across 1000 servers
- **Kafka** for ordered, durable message log (290K writes/sec)
- **Cassandra** for massive message storage (45 PB over 5 years)
- **Redis** for presence and hot data caching
- **Snowflake IDs** for globally unique, time-ordered messages
- **Fanout strategies** for efficient group chat delivery
- **Offline queues** for guaranteed message delivery
- **Multi-region deployment** for global low latency

**Key Challenges Solved:**
✅ Real-time delivery (<500ms)
✅ Strict message ordering
✅ Zero message loss
✅ Horizontal scalability
✅ High availability (99.99%)

**Trade-offs Accepted:**
❌ High complexity (multiple technologies)
❌ High cost (~$13M/year)
❌ Eventual consistency (read receipts)
❌ Stateful infrastructure (sticky sessions)

---

## References

- [WebSockets](../../02-components/2.0.3-real-time-communication.md)
- [Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)
- [Cassandra (Specialized Databases)](../../02-components/2.1.3-specialized-databases.md)
- [Consistent Hashing](../../02-components/2.2.2-consistent-hashing.md)
- [Distributed Locking](../../02-components/2.5.3-distributed-locking.md)

---

For **visual diagrams**, see [hld-diagram.md](hld-diagram.md) and [sequence-diagrams.md](sequence-diagrams.md).

For **detailed implementations**, see [pseudocode.md](pseudocode.md).

For **design decision analysis**, see [this-over-that.md](this-over-that.md).

For **complete details**, see the **[Full Design Document](3.3.1-design-live-chat-system.md)**.
