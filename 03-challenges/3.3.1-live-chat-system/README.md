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

## 1. Problem Statement

Design a real-time messaging system like WhatsApp or Slack that supports billions of users sending messages with
sub-500ms latency. The system must maintain persistent connections for instant message delivery, ensure messages are
never lost even during network failures, preserve strict message ordering within conversations, and provide real-time
presence updates (online/offline status). The solution must handle both one-on-one and group chats while scaling to
100 million concurrent connections globally.

---

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

| Requirement               | Description                                                                      | Priority     |
|---------------------------|----------------------------------------------------------------------------------|--------------|
| **Send/Receive Messages** | Messages delivered near-instantly to online users (push notification if offline) | Must Have    |
| **Message Persistence**   | All messages reliably stored and retrievable (message history)                   | Must Have    |
| **Strict Ordering**       | Messages within a conversation delivered in correct sequence                     | Must Have    |
| **Presence Service**      | Real-time online/offline status for all users                                    | Must Have    |
| **Read Receipts**         | Track message status: sent, delivered, read                                      | Must Have    |
| **Group Chat**            | Support group conversations (up to 256 members)                                  | Must Have    |
| **Message History**       | Users can retrieve past messages (years of history)                              | Should Have  |
| **File Sharing**          | Support images, videos, documents                                                | Should Have  |
| **End-to-End Encryption** | Optional E2E encryption (client-side)                                            | Nice to Have |

### Non-Functional Requirements (NFRs)

| Requirement                | Target                      | Rationale                           |
|----------------------------|-----------------------------|-------------------------------------|
| **Low Latency**            | < 500 ms (p99)              | Real-time feel for users            |
| **Fault Tolerance**        | Zero message loss           | Messages must be durable            |
| **High Availability**      | 99.99% uptime               | Users expect always-on service      |
| **Concurrent Connections** | 100M persistent connections | 10% of 1B MAU online simultaneously |
| **Message Throughput**     | 10K messages/sec            | Peak traffic handling               |
| **Scalability**            | Horizontal scaling          | Add servers as users grow           |

### Scale Estimation

| Metric                     | Assumption                       | Calculation                              | Result                      |
|----------------------------|----------------------------------|------------------------------------------|-----------------------------|
| **Total Users (MAU)**      | 1 Billion monthly active users   | -                                        | 1B MAU                      |
| **Daily Active Users**     | 50% of MAU                       | $1\text{B} \times 0.5$                   | 500M DAU                    |
| **Concurrent Connections** | 10% of MAU online                | $1\text{B} \times 0.1$                   | 100M persistent connections |
| **Messages per Day**       | 50 messages per DAU              | $500\text{M} \times 50$                  | 25 Billion messages/day     |
| **Peak QPS**               | 3× average                       | $25\text{B} / 86400 \times 3$            | ~870K messages/sec (peak)   |
| **Average QPS**            | 25B messages over 24 hours       | $25\text{B} / 86400$                     | ~290K messages/sec          |
| **Storage per Message**    | 1 KB average (text + metadata)   | -                                        | 1 KB/message                |
| **Daily Storage**          | 25B messages × 1 KB              | $25\text{B} \times 1\text{KB}$           | 25 TB/day                   |
| **5-Year Storage**         | 25 TB × 365 × 5                  | $25 \times 365 \times 5$                 | ~45 PB (45,000 TB)          |
| **Bandwidth (Outgoing)**   | 290K msg/sec × 1 KB × 2 (fanout) | $290\text{K} \times 1\text{KB} \times 2$ | ~580 MB/sec                 |

**Key Insights:**

- **100M concurrent WebSocket connections** require massive connection management infrastructure
- **45 PB storage** over 5 years requires distributed storage (Cassandra, S3)
- **870K peak QPS** requires horizontal scaling across multiple data centers
- **Strict ordering** within conversations is critical for user experience

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

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

| Component             | Responsibility                                            | Technology         | Scalability                                    |
|-----------------------|-----------------------------------------------------------|--------------------|------------------------------------------------|
| **WebSocket Servers** | Maintain persistent connections, push messages to clients | Node.js, Go, Netty | Horizontal (1000 servers for 100M connections) |
| **Load Balancer**     | Route clients to sticky WebSocket server                  | NGINX, HAProxy     | Horizontal + DNS                               |
| **Message Router**    | Validate, sequence, route messages to Kafka               | Go, Java service   | Horizontal (stateless)                         |
| **Kafka Cluster**     | Ordered, durable message log (source of truth)            | Apache Kafka       | Horizontal (partitioned)                       |
| **History Workers**   | Persist messages from Kafka to Cassandra                  | Kafka consumers    | Horizontal                                     |
| **Delivery Workers**  | Read from Kafka, push to online users                     | Kafka consumers    | Horizontal                                     |
| **Cassandra Cluster** | Store message history (billions of messages)              | Apache Cassandra   | Horizontal (sharded)                           |
| **Redis Cluster**     | Store presence data (online/offline status)               | Redis              | Horizontal (sharded)                           |
| **Sequence Manager**  | Generate globally unique, ordered message IDs             | ZooKeeper, etcd    | HA with leader election                        |

---

## 4. Detailed Component Design

### 4.1 WebSocket Connection Management

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

### 4.2 Message Flow Architecture

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

**Why Kafka?**

- ✅ **Strict Ordering:** Kafka partitions guarantee order per `chat_id`
- ✅ **Durability:** Messages replicated across brokers
- ✅ **Replayability:** History and Delivery workers can restart from any offset
- ✅ **Decoupling:** Write and read paths independent

*See [pseudocode.md::send_message_flow()](pseudocode.md) for implementation.*

---

### 4.3 Message Ordering & Sequence IDs

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

*See [pseudocode.md::generate_sequence_id()](pseudocode.md) for implementation.*

---

### 4.4 Presence Service (Online/Offline Status)

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

*See [pseudocode.md::update_presence()](pseudocode.md) for implementation.*

---

### 4.5 Read Receipts & Message Status

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

*See [pseudocode.md::handle_read_receipt()](pseudocode.md) for implementation.*

---

### 4.6 Group Chat vs 1-1 Chat

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

*See [pseudocode.md::fanout_group_message()](pseudocode.md) for implementation.*

---

### 4.7 Message History Storage (Cassandra)

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

*See [pseudocode.md::store_message_cassandra()](pseudocode.md) for implementation.*

---

## 5. Why This Over That?

### Decision 1: WebSockets vs Long Polling vs Server-Sent Events

**Chosen:** WebSockets

**Why WebSockets?**

- ✅ **Full-Duplex:** Both client and server can push messages anytime
- ✅ **Low Latency:** Single persistent connection (no connection overhead)
- ✅ **Efficient:** No HTTP headers on every message (saves bandwidth)
- ✅ **Real-Time:** True push notifications (no polling delay)

**Why Not Long Polling?**

- ❌ **High Overhead:** Client repeatedly opens/closes HTTP connections
- ❌ **Latency:** Delay between polling requests (1-5 seconds)
- ❌ **Server Load:** 100M clients polling every second = massive load

**Why Not Server-Sent Events (SSE)?**

- ❌ **One-Way Only:** Server can push, but client must use separate HTTP requests for uploads
- ❌ **Browser Limits:** Max 6 SSE connections per domain

*See [this-over-that.md](this-over-that.md) for detailed analysis.*

---

### Decision 2: Kafka vs Database for Message Ordering

**Chosen:** Kafka as message log

**Why Kafka?**

- ✅ **Ordering Guarantee:** Messages in same partition are strictly ordered
- ✅ **Durable:** Replicated log that survives failures
- ✅ **Decoupling:** Write and read paths are independent
- ✅ **Replayability:** Can reprocess messages from any point in time

**Why Not PostgreSQL/MySQL?**

- ❌ **Ordering Hard:** Concurrent inserts can arrive out-of-order
- ❌ **Write Bottleneck:** 290K writes/sec too high for single RDBMS
- ❌ **No Replayability:** Once read, message is consumed (no log concept)

**Why Not Cassandra Directly?**

- ❌ **No Ordering Guarantee:** Writes across nodes can arrive out-of-order
- ❌ **No Change Log:** Can't stream changes to consumers

*See [this-over-that.md](this-over-that.md) for detailed analysis.*

---

## 6. Bottlenecks and Future Scaling

### 6.1 Bottleneck: WebSocket Server Connection Limits

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

---

### 6.2 Bottleneck: Sequence Manager (Single Point)

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

*See [pseudocode.md::preallocate_id_range()](pseudocode.md) for implementation.*

---

### 6.3 Bottleneck: Message Delivery to Offline Users

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

*See [pseudocode.md::deliver_offline_messages()](pseudocode.md) for implementation.*

---

### 6.4 Bottleneck: Multi-Region Consistency

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

## 7. Common Anti-Patterns

### Anti-Pattern 1: Storing Messages in Redis

❌ **Using Redis as message database:**

**Why It's Bad:**

- ❌ **Memory Cost:** $45 \text{PB}$ in Redis = $\$$millions/year
- ❌ **Volatile:** Redis data lost if cluster fails (need RDB/AOF)
- ❌ **Limited Query:** Can't efficiently query by timestamp range

✅ **Better: Use Cassandra for storage, Redis for hot cache**

---

### Anti-Pattern 2: No Message Ordering

❌ **Using UUID as message ID (no ordering):**

**Why It's Bad:**

```
Message 1: "Hello" (UUID: abc123)
Message 2: "How are you?" (UUID: def456)

Client receives: "How are you?" then "Hello" ❌
```

✅ **Better: Use timestamp-based sequence IDs**

*See [pseudocode.md::generate_sequence_id()](pseudocode.md) for implementation.*

---

### Anti-Pattern 3: Synchronous Database Writes

❌ **Writing to Cassandra synchronously before ACK:**

**Why It's Bad:**

- ❌ **High Latency:** Cassandra write = 10-50ms
- ❌ **Blocks User:** Sender waits for persistence before seeing "sent"

✅ **Better: Write to Kafka first (5ms), then async to Cassandra**

---

## 8. Alternative Approaches

### Approach A: HTTP Polling (Not Chosen)

**Architecture:** Client polls server every 1 second for new messages

**Pros:**

- ✅ **Simple:** No persistent connections
- ✅ **Firewall-Friendly:** Works through proxies

**Cons:**

- ❌ **High Latency:** 1-5 second delay
- ❌ **Inefficient:** 100M clients × 1 poll/sec = 100M req/sec (even if no messages)
- ❌ **Battery Drain:** Constant polling kills mobile battery

**Why Not Chosen:** Latency and efficiency requirements not met.

---

### Approach B: P2P (Peer-to-Peer)

**Architecture:** Clients connect directly to each other (like BitTorrent)

**Pros:**

- ✅ **No Server:** Decentralized
- ✅ **Low Cost:** No infrastructure

**Cons:**

- ❌ **NAT Traversal:** Firewall issues
- ❌ **No History:** Messages lost if both clients offline
- ❌ **No Ordering:** No central coordinator

**Why Not Chosen:** Reliability and ordering requirements not met.

---

## 9. Monitoring and Observability

### Key Metrics

| Metric                      | Target          | Alert Threshold  | Description                   |
|-----------------------------|-----------------|------------------|-------------------------------|
| **Message Latency (P99)**   | < 500 ms        | > 1 second       | End-to-end delivery time      |
| **WebSocket Connections**   | 100M            | < 80M (capacity) | Total active connections      |
| **Kafka Lag**               | < 1000 messages | > 10K messages   | Consumer lag behind producers |
| **Message Loss Rate**       | 0%              | > 0.001%         | Messages not delivered        |
| **Presence Update Latency** | < 100 ms        | > 500 ms         | Online/offline status update  |
| **Cassandra Write Latency** | < 50 ms         | > 200 ms         | Message persistence time      |

### Dashboards

**Dashboard 1: Real-Time Health**

- Active WebSocket connections per server
- Message throughput (msg/sec)
- Kafka lag per consumer group
- Redis hit rate (presence cache)

**Dashboard 2: User Experience**

- Message delivery latency (P50, P99, P999)
- Connection success rate
- Offline message queue depth
- Read receipt processing time

### Critical Alerts

- 🔴 **Kafka consumer lag > 10K:** Messages delayed, users not receiving messages
- 🔴 **WebSocket server down:** 100K users disconnected
- 🔴 **Sequence Manager unavailable:** No new messages can be sent
- 🟡 **Cassandra write latency > 200ms:** History storage slow
- 🟡 **Redis presence cache miss rate > 50%:** Presence queries slow

---

## 10. Trade-offs Summary

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

## 11. Real-World Implementations

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

## 12. Deep Dive: Handling Edge Cases

### 12.1 Message Deduplication

**Problem:** Network retries can cause duplicate messages.

**Example:**

```
Client sends: "Hello"
Network timeout (no ACK received)
Client retries: "Hello" (duplicate!)
```

**Solution: Idempotency with Message ID**

Each message has unique ID (generated client-side):

```
message_id: UUID v4 (client-generated)
```

**Server-side deduplication:**

1. Check if `message_id` already processed
2. Store processed IDs in Redis (TTL: 24 hours)
3. If duplicate, return success (idempotent)

**Implementation:**

```
Redis key: processed_message:{message_id}
TTL: 86400 seconds (24 hours)

On receive:
  if EXISTS processed_message:{message_id}:
    return SUCCESS (already processed)
  else:
    process message
    SET processed_message:{message_id} 1 EX 86400
```

*See [pseudocode.md::handle_duplicate_message()](pseudocode.md) for implementation.*

---

### 12.2 Clock Skew & Time Synchronization

**Problem:** Different servers have different clocks → messages may arrive out-of-order.

**Example:**

```
Server 1 (clock: 10:00:00) sends message A (timestamp: 10:00:00)
Server 2 (clock: 09:59:55) sends message B (timestamp: 09:59:55)

Recipient receives: B, then A (wrong order!)
```

**Solution: Logical Clocks (Sequence IDs)**

Don't rely on timestamps alone. Use strictly increasing sequence IDs from centralized Sequence Manager:

```
Message A: seq_id = 1001, timestamp = 10:00:00
Message B: seq_id = 1002, timestamp = 09:59:55

Order by seq_id, not timestamp → A before B ✅
```

**NTP Synchronization:**

- All servers sync with NTP every 5 minutes
- Alert if clock drift > 1 second
- Use sequence IDs as primary ordering mechanism

---

### 12.3 Split Brain (Network Partition)

**Problem:** Two data centers can't communicate. Each thinks the other is down.

**Scenario:**

```
DC1 (US-East)          DC2 (US-West)
    |                       |
    X  Network partition   X
    |                       |
Both accept writes independently
→ Conflicting message orders!
```

**Solution: Quorum-Based Writes**

**Approach 1: Single-Region Write Master**

- Only one region (e.g., US-East) can accept writes
- Other regions read-only
- Trade-off: Higher latency for users in other regions

**Approach 2: Conflict-Free Replicated Data Types (CRDTs)**

- Messages are commutative (order doesn't affect final state)
- Use vector clocks to detect conflicts
- Trade-off: Complex implementation

**Recommendation:** Single write master for chat (simpler, acceptable latency).

---

### 12.4 Message Editing & Deletion

**Problem:** User edits/deletes message. How to sync across all recipients?

**Edit Flow:**

1. Client sends EDIT event (message_id, new_content)
2. Router publishes to Kafka
3. Update in Cassandra: `message_id → content = new_content, edited_at = now()`
4. Broadcast to all online recipients
5. Offline recipients see edited version when they come online

**Delete Flow:**

1. Client sends DELETE event (message_id)
2. Soft delete in Cassandra: `deleted = true, deleted_at = now()`
3. Content not actually deleted (compliance, legal)
4. Broadcast tombstone to recipients
5. Clients hide message locally

**Data Model:**

```sql
CREATE TABLE messages (
    ...
    deleted BOOLEAN DEFAULT false,
    deleted_at TIMESTAMP,
    edited BOOLEAN DEFAULT false,
    edited_at TIMESTAMP,
    edit_history LIST<TEXT>  -- Store previous versions
);
```

*See [pseudocode.md::handle_message_edit()](pseudocode.md) for implementation.*

---

### 12.5 Rate Limiting per User

**Problem:** Spam/abuse. User sends 1000 messages/second.

**Solution: Multi-Level Rate Limiting**

**Level 1: Per-User Rate Limit (Redis)**

```
rate_limit:user:{user_id}:messages → count
TTL: 60 seconds

If count > 100 messages/minute:
  Return 429 Too Many Requests
  Temporarily block user
```

**Level 2: Per-Chat Rate Limit**

```
rate_limit:chat:{chat_id}:messages → count
TTL: 60 seconds

If count > 1000 messages/minute (all users combined):
  Slow down message processing
  Alert moderators
```

**Level 3: IP-Based Rate Limit**

```
rate_limit:ip:{ip_address}:messages → count
TTL: 60 seconds

If count > 500 messages/minute:
  Block IP temporarily (DDoS protection)
```

*See [pseudocode.md::check_rate_limit_user()](pseudocode.md) for implementation.*

---

### 12.6 Graceful Degradation Strategies

**When Kafka is Down:**

- Buffer messages in WebSocket server memory (max 1000 messages)
- Return "Sending..." status to client
- Retry publishing to Kafka every 5 seconds
- After 5 minutes, fail with error (inform user)

**When Cassandra is Down:**

- Messages still delivered (via Kafka)
- History queries fail → Return cached messages only
- Display warning: "Some history unavailable"

**When Redis (Presence) is Down:**

- Assume all users online (fail safe)
- No presence information displayed
- Message delivery continues normally

**Priority: Message delivery > History > Presence**

---

### 12.7 Connection Stickiness & Load Balancing

**Problem:** User's WebSocket connection must stay on same server.

**Solution: Consistent Hashing Load Balancer**

**Load Balancer Configuration:**

```
upstream websocket_servers {
    hash $user_id consistent;  # Sticky by user_id
    server ws-server-1:8080;
    server ws-server-2:8080;
    server ws-server-3:8080;
}
```

**Benefits:**

- Same user always routes to same server
- Connection state preserved
- No need for shared session store

**Handling Server Failure:**

- If server dies, load balancer re-hashes to different server
- Client automatically reconnects
- Fetch offline messages from queue

---

## 13. Interview Discussion Points

### 13.1 Why Kafka Over Direct Database Writes?

**Interviewer:** "Why not just write messages directly to Cassandra?"

**Answer:**

**Without Kafka (Direct Write):**

```
Client → WebSocket Server → Cassandra → Respond to client
Latency: 50-100ms (Cassandra write)
```

**With Kafka:**

```
Client → WebSocket Server → Kafka → Respond to client
Latency: 5-10ms (Kafka write)

Background workers:
  Kafka → Cassandra (async)
  Kafka → Delivery Worker → Recipient
```

**Benefits:**

1. **Lower Latency:** Sender gets ACK faster (5ms vs 50ms)
2. **Ordering:** Kafka guarantees order per partition
3. **Decoupling:** Write and read paths independent
4. **Replayability:** Can reprocess messages from any offset
5. **Reliability:** Kafka handles temporary Cassandra outages

**Trade-off:** More complexity (need to operate Kafka cluster).

---

### 13.2 How to Handle 1 Million Concurrent Connections?

**Interviewer:** "100M connections seems impossible on commodity hardware."

**Answer:**

**Math:**

```
Connections per server: 100,000 (OS limit ~65K file descriptors, tuned to 100K)
Required connections: 100,000,000
Servers needed: 100M / 100K = 1,000 servers
```

**Server Specs (each):**

- CPU: 8-16 cores (handle I/O multiplexing)
- RAM: 32-64 GB (100K connections × 64 KB buffers = 6.4 GB minimum)
- Network: 10 Gbps NIC
- OS: Linux with tuned kernel parameters

**OS Tuning:**

```bash
ulimit -n 1000000          # File descriptors
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=8192
```

**Technology Choices:**

- **Go:** Goroutines scale to 1M+ connections
- **Erlang/Elixir:** Lightweight processes (WhatsApp uses this)
- **Node.js:** Event loop, but single-threaded (harder to scale)
- **Java/Netty:** NIO, excellent performance

---

### 13.3 What if User is in Group with 1000 Members?

**Interviewer:** "How do you fanout to 1000 members efficiently?"

**Answer:**

**Problem:**

```
User sends message to group with 1000 members
→ Need to push to 1000 connections
→ If done synchronously: 1000 × 10ms = 10 seconds!
```

**Solution: Async Fanout with Worker Pool**

**Architecture:**

```
Message → Kafka (single write, fast)
         ↓
    Fanout Worker Pool (100 workers)
         ↓
    Each worker handles 10 users
    → 1000 users / 100 workers = 10 users/worker
    → Parallel fanout: ~100ms total
```

**Optimizations:**

1. **Batch WebSocket writes:** Send to multiple connections in parallel
2. **Prioritize online users:** Deliver to online users first, queue for offline
3. **Compression:** Compress message once, send to all (save bandwidth)
4. **Smart routing:** Group members in same region get faster delivery

**Trade-off:** Small delay (100-500ms) for large groups vs instant for 1-1 chat.

---

### 13.4 How to Ensure Messages Aren't Lost?

**Interviewer:** "What guarantees zero message loss?"

**Answer:**

**Multi-Level Durability:**

**Level 1: Client ACKs**

```
Client sends message
   ↓
Server ACKs (message in Kafka, replicated)
   ↓
Client shows "Sent ✓"
```

**Level 2: Kafka Replication**

```
Producer: acks=all (wait for all replicas)
Replication factor: 3 (message on 3 brokers)
Min in-sync replicas: 2 (can lose 1 broker safely)
```

**Level 3: Cassandra Durability**

```
Consistency level: QUORUM (2 out of 3 nodes)
Commit log: Append-only, fsync on write
Replication factor: 3
```

**Level 4: Offline Message Queue**

```
If recipient offline:
  Store in Redis List (persistence: AOF)
  Send push notification
  Deliver when user comes online
```

**Failure Scenarios:**

- **Kafka broker dies:** Other 2 replicas have message ✅
- **Cassandra node dies:** QUORUM still satisfied ✅
- **Redis dies:** Kafka has message, can replay ✅
- **WebSocket server dies:** Client reconnects, fetches from Kafka offset ✅

**Only way to lose message:** All 3 Kafka replicas + all 3 Cassandra replicas fail simultaneously (extremely unlikely).

---

### 13.5 Multi-Tenancy: How to Isolate Large Enterprise Customers?

**Interviewer:** "What if one customer sends 1B messages/day?"

**Answer:**

**Problem:** One customer (e.g., Slack workspace with 100K users) can overwhelm shared infrastructure.

**Solution: Tenant-Based Sharding**

**Kafka Partitioning:**

```
Small customers: chat_id-based partitions (shared)
Large customers: dedicated partitions

tenant:acme-corp → Kafka topic: chat-acme-corp
  → Dedicated consumer groups
  → Isolated from other customers
```

**WebSocket Server Pools:**

```
Pool 1: General users (shared)
Pool 2: Enterprise customer "acme-corp" (dedicated)
Pool 3: Enterprise customer "globex" (dedicated)
```

**Benefits:**

- **Isolation:** Large customer doesn't affect others
- **Custom SLA:** Enterprise customers get higher guarantees
- **Billing:** Easy to track per-customer usage

**Trade-off:** Higher operational complexity (more clusters to manage).

---

## 14. References

### Related System Design Components

- **[2.0.3 Real-Time Communication](../../02-components/2.0.3-real-time-communication.md)** - WebSocket protocol,
  alternatives
- **[2.3.2 Kafka Deep Dive](../../02-components/2.3.2-kafka-deep-dive.md)** - Partitions, ordering, consumer groups
- **[2.1.3 Specialized Databases](../../02-components/2.1.3-specialized-databases.md)** - Wide-column storage (
  Cassandra)
- **[2.1.9 Cassandra Deep Dive](../../02-components/2.1.9-cassandra-deep-dive.md)** - Time-series data, horizontal
  scaling
- **[2.1.11 Redis Deep Dive](../../02-components/2.1.11-redis-deep-dive.md)** - In-memory storage, presence tracking
- **[2.5.3 Distributed Locking](../../02-components/2.5.3-distributed-locking.md)** - Sequence Manager coordination
- **[1.2.3 API Gateway & Service Mesh](../../01-principles/1.2.3-api-gateway-servicemesh.md)** - WebSocket routing
- **[2.5.1 Rate Limiting Algorithms](../../02-components/2.5.1-rate-limiting-algorithms.md)** - Spam prevention

### Related Design Challenges

- **[3.2.1 Twitter Timeline](../3.2.1-twitter-timeline/)** - Fanout strategies for group chats
- **[3.2.2 Notification Service](../3.2.2-notification-service/)** - Push notifications, offline message delivery
- **[3.3.2 Uber Ride Matching](../3.3.2-uber-ride-matching/)** - Real-time location updates, WebSocket patterns

### External Resources

- **WebSocket Protocol:** [RFC 6455](https://tools.ietf.org/html/rfc6455) - WebSocket specification
- **Kafka Documentation:** [Apache Kafka](https://kafka.apache.org/documentation/) - Message ordering, partitions
- **Cassandra Documentation:** [Apache Cassandra](https://cassandra.apache.org/doc/latest/) - Wide-column database
- **WhatsApp Architecture:** [WhatsApp Engineering Blog](https://engineering.fb.com/category/core-data/) - Real-world
  implementation
- **Slack Architecture:** [Slack Engineering Blog](https://slack.engineering/) - Message delivery patterns

### Books

- *Designing Data-Intensive Applications* by Martin Kleppmann - Chapters on message queues, ordering, and real-time
  systems
- *High Performance Browser Networking* by Ilya Grigorik - WebSocket optimization and real-time communication