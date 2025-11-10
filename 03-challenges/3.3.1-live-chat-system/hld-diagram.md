# Live Chat System - High-Level Design

This document contains Mermaid diagrams illustrating the system architecture, component design, data flow, and scaling
strategies for the Live Chat System.

---

## Table of Contents

1. [Complete System Architecture](#1-complete-system-architecture)
2. [WebSocket Connection Management](#2-websocket-connection-management)
3. [Message Send Flow](#3-message-send-flow)
4. [Message Receive Flow](#4-message-receive-flow)
5. [Kafka Partitioning Strategy](#5-kafka-partitioning-strategy)
6. [Presence Service Architecture](#6-presence-service-architecture)
7. [Read Receipt Flow](#7-read-receipt-flow)
8. [Group Chat Fanout](#8-group-chat-fanout)
9. [Offline Message Handling](#9-offline-message-handling)
10. [Multi-Region Deployment](#10-multi-region-deployment)
11. [Sequence ID Generation](#11-sequence-id-generation)
12. [Monitoring Dashboard](#12-monitoring-dashboard)

---

## 1. Complete System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        C1[Mobile App]
        C2[Web Browser]
        C3[Desktop App]
    end

    subgraph "Load Balancing"
        LB[Load Balancer<br/>Sticky Sessions<br/>Hash by user_id]
    end

    subgraph "Connection Layer - 1000 Servers"
        WS1[WebSocket Server 1<br/>100K connections]
        WS2[WebSocket Server 2<br/>100K connections]
        WSN[WebSocket Server N<br/>100K connections]
    end

    subgraph "Application Layer"
        MR[Message Router<br/>Validate, Sequence, Route]
        SM[Sequence Manager<br/>ZooKeeper/etcd<br/>Generate IDs]
    end

    subgraph "Message Broker"
        K1[Kafka Partition 1<br/>chat_id: 0-999]
        K2[Kafka Partition 2<br/>chat_id: 1000-1999]
        KN[Kafka Partition N<br/>chat_id: ...]
    end

    subgraph "Consumer Workers"
        HW[History Workers<br/>Persist to DB]
        DW[Delivery Workers<br/>Push to users]
    end

    subgraph "Storage Layer"
        CASS[Cassandra Cluster<br/>100 nodes<br/>45 PB storage]
        REDIS[Redis Cluster<br/>20 nodes<br/>Presence + Cache]
    end

    C1 --> LB
    C2 --> LB
    C3 --> LB
    LB --> WS1
    LB --> WS2
    LB --> WSN
    WS1 --> MR
    WS2 --> MR
    WSN --> MR
    MR --> SM
    MR --> K1
    MR --> K2
    MR --> KN
    K1 --> HW
    K2 --> HW
    KN --> HW
    K1 --> DW
    K2 --> DW
    KN --> DW
    HW --> CASS
    DW --> WS1
    DW --> WS2
    DW --> WSN
    WS1 -. Presence Updates .-> REDIS
    WS2 -. Presence Updates .-> REDIS
    WSN -. Presence Updates .-> REDIS
    style LB fill: #e1f5ff
    style MR fill: #ffe1e1
    style CASS fill: #e1ffe1
    style REDIS fill: #fff4e1
```

**Flow Explanation:**

This diagram shows the complete end-to-end architecture for the Live Chat System handling 100M concurrent connections
and 870K messages/sec.

**Key Components:**

1. **Client Layer:** Mobile, web, desktop apps connect via WebSocket
2. **Load Balancer:** Routes connections with sticky sessions (same user → same server)
3. **WebSocket Servers:** 1000 servers, each handling 100K persistent connections
4. **Message Router:** Validates messages, requests sequence IDs, publishes to Kafka
5. **Sequence Manager:** Generates globally unique, time-ordered message IDs
6. **Kafka Cluster:** Ordered, durable message log (1000 partitions by chat_id)
7. **History Workers:** Consume from Kafka, persist to Cassandra (25 TB/day)
8. **Delivery Workers:** Consume from Kafka, push to recipient's WebSocket server
9. **Cassandra:** 100-node cluster storing 45 PB of message history (5 years)
10. **Redis:** 20-node cluster for presence data and hot message cache

**Flow:**

1. Client sends message via WebSocket
2. Load balancer routes to sticky server
3. WebSocket server forwards to Message Router
4. Router gets sequence ID from Sequence Manager
5. Router publishes to Kafka (partitioned by chat_id)
6. Two consumer groups process in parallel:
    - History Workers → Write to Cassandra
    - Delivery Workers → Push to recipient
7. Recipient receives via WebSocket push

**Performance:**

- Message latency: < 500ms end-to-end
- Write throughput: 870K messages/sec peak
- Connection capacity: 100M concurrent connections
- Storage: 45 PB (5-year retention)

---

## 2. WebSocket Connection Management

```mermaid
flowchart TD
    Start([Client Connects]) --> DNS[DNS Resolves to Load Balancer]
    DNS --> LB{Load Balancer<br/>Calculate Hash}
    LB -->|hash user_id mod 1000| Server[Route to WebSocket Server X]
    Server --> Upgrade[HTTP Upgrade to WebSocket]
    Upgrade --> Auth[Authenticate User JWT Token]
    Auth -->|Valid| Register[Register Connection<br/>user_id to ws_server_id mapping]
    Auth -->|Invalid| Reject[Return 401 Unauthorized]
    Register --> Presence[Update Presence Service<br/>SET user presence online TTL 60s]
    Presence --> Heartbeat[Start Heartbeat Timer<br/>Every 30 seconds]
    Heartbeat --> Active{Connection Active?}
    Active -->|Yes| Refresh[EXPIRE user presence 60<br/>Refresh TTL]
    Active -->|No| Timeout[Connection Timeout<br/>or Client Disconnect]
    Refresh --> Heartbeat
    Timeout --> Cleanup[Cleanup:<br/>1. Remove from connection pool<br/>2. Delete presence<br/>3. Close socket]
    Cleanup --> Done([End])
    Reject --> Done
    style Register fill: #e1ffe1
    style Active fill: #fff4e1
    style Cleanup fill: #ffe1e1
```

**Flow Explanation:**

Shows the complete WebSocket connection lifecycle from establishment to cleanup.

**Steps:**

1. **DNS Resolution** (0ms): Client resolves load balancer IP
2. **Load Balancing** (1ms): Hash user_id to determine target server (sticky session)
3. **HTTP Upgrade** (10ms): Standard HTTP request upgraded to WebSocket protocol
4. **Authentication** (5ms): Validate JWT token, verify user identity
5. **Registration** (2ms): Store connection mapping in memory (user_id → ws_server_id)
6. **Presence Update** (1ms): Set Redis key `user:{user_id}:presence = "online"` with TTL 60s
7. **Heartbeat Loop** (ongoing): Every 30 seconds, refresh presence TTL
8. **Cleanup** (on disconnect): Remove connection, delete presence, close socket

**Connection State:**

```
{
  user_id: "12345",
  ws_server_id: "ws-server-42",
  socket_fd: 1234,
  connected_at: timestamp,
  last_heartbeat: timestamp
}
```

**Benefits:**

- **Sticky Sessions:** Same user always connects to same server (state preserved)
- **Automatic Cleanup:** TTL ensures presence accurate (no manual deletion)
- **Fast Reconnect:** Client retries with exponential backoff (1s, 2s, 4s, 8s)

**Performance:**

- Connection establishment: ~50ms
- Heartbeat overhead: 1 byte every 30s (minimal bandwidth)
- Server capacity: 100K connections per server

---

## 3. Message Send Flow

```mermaid
sequenceDiagram
    participant Client
    participant WS as WebSocket Server
    participant Router as Message Router
    participant SeqMgr as Sequence Manager
    participant Kafka
    participant History as History Worker
    participant Delivery as Delivery Worker
    participant Cassandra
    participant Recipient
    Client ->> WS: Send Message<br/>chat_id, content
    WS ->> Router: Forward Message<br/>+ sender_id, timestamp
    Router ->> Router: Validate:<br/>- User authenticated?<br/>- Content valid?<br/>- Rate limit OK?
    Router ->> SeqMgr: Request Sequence ID<br/>for chat_id
    SeqMgr -->> Router: seq_id = 100542
    Router ->> Router: Wrap Message:<br/>{chat_id, seq_id,<br/>sender_id, content,<br/>timestamp}
    Router ->> Kafka: Publish to Partition<br/>partition = hash(chat_id)
    Kafka -->> Router: ACK (message persisted)
    Router -->> WS: ACK (message sent)
    WS -->> Client: Status: SENT ✓

    par Parallel Processing
        Kafka ->> History: Consumer reads message
        History ->> Cassandra: INSERT INTO messages
        Cassandra -->> History: OK
    and
        Kafka ->> Delivery: Consumer reads message
        Delivery ->> Delivery: Lookup recipient<br/>ws_server_id
        Delivery ->> Recipient: Push via WebSocket
        Recipient -->> Delivery: ACK (delivered)
    end

    Note over Client, Recipient: Total latency: < 500ms
```

**Flow Explanation:**

Shows the complete path of a message from sender to recipient with all intermediate steps.

**Steps:**

1. **Client Send** (0ms): User types message, client sends via WebSocket
2. **Forward to Router** (5ms): WebSocket server forwards to Message Router
3. **Validation** (2ms): Check authentication, content validity, rate limits
4. **Request Sequence ID** (3ms): Get globally unique, time-ordered ID from Sequence Manager
5. **Wrap Message** (1ms): Add metadata (seq_id, sender_id, timestamp)
6. **Publish to Kafka** (5ms): Write to partition based on chat_id hash
7. **Kafka ACK** (5ms): Message persisted, replicated to 3 brokers
8. **Client ACK** (10ms): Sender receives "SENT ✓" status (doesn't wait for delivery)
9. **Parallel Processing:**
    - **History Worker** (50-100ms): Reads from Kafka, writes to Cassandra for persistence
    - **Delivery Worker** (100-200ms): Reads from Kafka, pushes to recipient's WebSocket
10. **Recipient Receives** (300-500ms total): Message appears in recipient's chat

**Performance:**

- Sender ACK: ~50ms (fast feedback)
- Recipient delivery: ~300-500ms (total end-to-end)
- Throughput: 870K messages/sec peak

**Key Benefits:**

- **Low Latency:** Sender gets ACK in 50ms (doesn't wait for Cassandra)
- **Decoupled:** History and delivery workers independent
- **Durable:** Kafka replication ensures zero message loss
- **Ordered:** Kafka partition guarantees sequence within chat_id

---

## 4. Message Receive Flow

```mermaid
flowchart TD
    Start([Delivery Worker Reads from Kafka]) --> Parse[Parse Message:<br/>recipient_id, content]
    Parse --> CheckPresence{Check Presence Service<br/>Is recipient online?}
    CheckPresence -->|Online| LookupConn[Lookup Connection:<br/>Which ws_server has<br/>recipient connection?]
    CheckPresence -->|Offline| QueueOffline[Add to Offline Queue<br/>LPUSH offline_messages<br/>Send Push Notification]
    LookupConn --> Route[Route to WebSocket Server<br/>via internal RPC/gRPC]
    Route --> Push[Push Message via WebSocket<br/>to recipient]
    Push --> Ack{Recipient ACKs?}
    Ack -->|Yes| UpdateStatus[Update Message Status:<br/>delivered_at = now]
    Ack -->|No after 5s| Retry{Retry Count < 3?}
    Retry -->|Yes| Push
    Retry -->|No| QueueOffline
    UpdateStatus --> Done([End])
    QueueOffline --> Done
    style Push fill: #e1ffe1
    style QueueOffline fill: #fff4e1
    style UpdateStatus fill: #e1f5ff
```

**Flow Explanation:**

Shows how messages are delivered to recipients, handling both online and offline cases.

**Steps:**

1. **Delivery Worker Reads** (0ms): Consumer pulls message from Kafka
2. **Parse Message** (1ms): Extract recipient_id, content, metadata
3. **Check Presence** (1ms): Query Redis: `GET user:{recipient_id}:presence`
4. **Online Path:**
    - **Lookup Connection** (2ms): Find which WebSocket server has recipient
    - **Route** (5ms): Internal RPC call to target WebSocket server
    - **Push** (10ms): WebSocket server pushes to client
    - **ACK** (10ms): Client sends delivery acknowledgment
    - **Update Status** (5ms): Write to Cassandra: `delivered_at = now()`
5. **Offline Path:**
    - **Queue** (2ms): `LPUSH offline_messages:{recipient_id} {message}`
    - **Push Notification** (100ms): Send via FCM/APNS to wake app
    - **Delivered Later:** When user comes online, fetch from offline queue

**Retry Logic:**

```
Attempt 1: Immediate
Attempt 2: After 1 second
Attempt 3: After 2 seconds
After 3 failures: Mark offline, queue message
```

**Performance:**

- Online delivery: ~50ms
- Offline queuing: ~10ms
- Push notification: ~100-500ms

---

## 5. Kafka Partitioning Strategy

```mermaid
graph TB
    subgraph "Message Router"
        MR[Message Router<br/>Receives message with chat_id]
    end

    subgraph "Partition Assignment"
        HASH[Partition = hash chat_id mod 1000]
    end

    subgraph "Kafka Cluster - 1000 Partitions"
        P0[Partition 0<br/>chat_id: hash 0]
        P1[Partition 1<br/>chat_id: hash 1]
        P2[Partition 2<br/>chat_id: hash 2]
        P999[Partition 999<br/>chat_id: hash 999]
    end

    subgraph "Consumer Groups"
        CG1[Consumer Group:<br/>History Workers<br/>10 consumers]
        CG2[Consumer Group:<br/>Delivery Workers<br/>100 consumers]
    end

    subgraph "Benefits"
        B1[Strict Ordering:<br/>Same chat_id always<br/>same partition]
        B2[Parallel Processing:<br/>Different chats processed<br/>in parallel]
        B3[Load Distribution:<br/>Even distribution<br/>across partitions]
    end

    MR --> HASH
    HASH -->|hash = 0| P0
    HASH -->|hash = 1| P1
    HASH -->|hash = 2| P2
    HASH -->|hash = 999| P999
    P0 --> CG1
    P1 --> CG1
    P2 --> CG1
    P999 --> CG1
    P0 --> CG2
    P1 --> CG2
    P2 --> CG2
    P999 --> CG2
    CG1 -.-> B1
    CG2 -.-> B2
    P0 -.-> B3
    style HASH fill: #e1ffe1
    style B1 fill: #fff4e1
    style B2 fill: #e1f5ff
```

**Flow Explanation:**

Shows how Kafka partitioning ensures message ordering and enables parallel processing.

**Partitioning Logic:**

```
partition_id = hash(chat_id) % 1000

Example:
  chat_id: "abc-123" → hash: 742 → partition: 742
  chat_id: "def-456" → hash: 18  → partition: 18
```

**Key Guarantees:**

1. **Ordering per Chat:**
    - All messages for chat_id "abc-123" go to partition 742
    - Partition 742 guarantees sequential ordering
    - Messages delivered in exact send order ✅

2. **Parallel Processing:**
    - Chat A (partition 0) and Chat B (partition 1) processed independently
    - 1000 partitions = 1000 parallel processing streams
    - No cross-chat blocking

3. **Consumer Distribution:**
    - History Workers: 10 consumers, each handles ~100 partitions
    - Delivery Workers: 100 consumers, each handles ~10 partitions
    - If consumer dies, partitions reassigned automatically

**Benefits:**

- **Strict Ordering:** Guaranteed within each conversation
- **High Throughput:** 870K msg/sec across 1000 partitions = 870 msg/sec/partition
- **Scalability:** Add more partitions as traffic grows
- **Fault Tolerance:** Replication factor 3 (message on 3 brokers)

**Performance:**

- Write latency: ~5ms (with replication)
- Consumer lag: < 1000 messages (target)
- Throughput per partition: ~1000 msg/sec

---

## 6. Presence Service Architecture

```mermaid
graph TB
    subgraph "WebSocket Servers"
        WS1[WS Server 1]
        WS2[WS Server 2]
        WSN[WS Server N]
    end

    subgraph "Redis Cluster - Sharded by user_id"
        R1[Redis Shard 1<br/>user_id: 0-49M]
        R2[Redis Shard 2<br/>user_id: 50M-99M]
        R3[Redis Shard 3<br/>user_id: 100M-149M]
        R20[Redis Shard 20<br/>user_id: ...]
    end

    subgraph "Operations"
        OP1[Set Online:<br/>SET user presence online<br/>EX 60]
        OP2[Heartbeat:<br/>EXPIRE user presence 60<br/>Every 30s]
        OP3[Get Status:<br/>GET user presence<br/>Returns online or NULL]
        OP4[Batch Query:<br/>MGET user1 user2 user3<br/>For group chat]
    end

    WS1 -->|On Connect| OP1
    WS1 -->|Periodic| OP2
    WS2 -->|Query| OP3
    WSN -->|Group Query| OP4
    OP1 --> R1
    OP2 --> R2
    OP3 --> R3
    OP4 --> R20
    R1 -. TTL Expire .-> Auto[Automatic Offline:<br/>Key expires after 60s<br/>No manual deletion]
    style OP1 fill: #e1ffe1
    style OP2 fill: #fff4e1
    style Auto fill: #ffe1e1
```

**Flow Explanation:**

Shows how presence tracking works using Redis with automatic TTL expiration.

**Data Model:**

```
Key: user:{user_id}:presence
Value: "online"
TTL: 60 seconds

Example:
  user:12345:presence = "online" (expires in 60s)
```

**Operations:**

1. **Set Online** (on WebSocket connect):
   ```
   SET user:12345:presence "online" EX 60
   ```
    - Atomic operation
    - TTL set to 60 seconds
    - Auto-expires if no heartbeat

2. **Heartbeat** (every 30 seconds):
   ```
   EXPIRE user:12345:presence 60
   ```
    - Refresh TTL to 60 seconds
    - Keeps user online
    - If missed, user automatically offline after 60s

3. **Get Status** (check if online):
   ```
   GET user:12345:presence
   ```
    - Returns "online" if present
    - Returns NULL if offline (key expired)
    - Sub-millisecond latency

4. **Batch Query** (for group chat):
   ```
   MGET user:12345:presence user:67890:presence user:99999:presence
   ```
    - Single roundtrip for multiple users
    - Example: 50-member group = 1 query (vs 50 separate queries)

**Benefits:**

- **Automatic Cleanup:** TTL handles offline users (no manual deletion)
- **Sub-Millisecond:** <1ms query latency
- **Scalable:** 20 Redis shards handle 100M users
- **Accurate:** 60-second TTL ensures recent status

**Sharding:**

```
shard_id = hash(user_id) % 20

100M users / 20 shards = 5M users per shard
Memory per shard: 5M × 100 bytes = 500 MB
```

---

## 7. Read Receipt Flow

```mermaid
sequenceDiagram
    participant Sender
    participant Recipient
    participant WS as WebSocket Server
    participant Kafka
    participant Worker as Read Receipt Worker
    participant Cassandra
    Note over Sender, Recipient: Message already delivered
    Recipient ->> Recipient: User opens chat<br/>Views message
    Recipient ->> WS: Send READ event<br/>{message_id, chat_id}
    WS ->> Kafka: Publish READ event<br/>topic: read-receipts
    Kafka -->> WS: ACK
    WS -->> Recipient: READ ACK
    Kafka ->> Worker: Consumer reads event
    Worker ->> Cassandra: UPDATE message_status<br/>SET read_at = now()<br/>WHERE message_id = ?
    Cassandra -->> Worker: OK
    Worker ->> Worker: Lookup sender ws_server
    Worker ->> Sender: Push READ notification<br/>via WebSocket
    Sender ->> Sender: Update UI:<br/>Show double checkmark ✓✓
    Note over Sender, Recipient: Sender sees read receipt
```

**Flow Explanation:**

Shows how read receipts are tracked and delivered to the sender.

**Message States:**

1. **SENT:** Message written to Kafka (sender sees single checkmark ✓)
2. **DELIVERED:** Message delivered to recipient's device (double checkmark ✓✓)
3. **READ:** Recipient viewed message (blue checkmark ✓✓)

**Steps:**

1. **User Views Message** (0ms): Recipient opens chat, scrolls to message
2. **Client Sends READ Event** (10ms): `{message_id: "xyz", chat_id: "abc"}`
3. **Publish to Kafka** (5ms): Separate topic for read receipts
4. **Worker Processes** (50ms): Read Receipt Worker consumes event
5. **Update Database** (20ms): `UPDATE message_status SET read_at = now()`
6. **Notify Sender** (100ms): Push notification to sender's WebSocket
7. **UI Update** (0ms): Sender's UI shows blue checkmark

**Data Model:**

```sql
CREATE TABLE message_status (
    message_id UUID PRIMARY KEY,
    chat_id UUID,
    sender_id UUID,
    recipient_id UUID,
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    read_at TIMESTAMP
);
```

**Optimization for Group Chat:**

- **Small groups (< 10):** Track individual read receipts
- **Large groups (> 50):** Only track "last read message ID" per user
    - Too expensive to track 256 read receipts per message

**Performance:**

- READ event latency: ~100ms (sender notified)
- Throughput: ~100K read events/sec

---

## 8. Group Chat Fanout

```mermaid
flowchart TD
    Start([User Sends to Group]) --> Store[Store Message in Kafka<br/>Single write, fast]
    Store --> Count{Group Size?}
    Count -->|Small < 50 members| SyncFanout[Synchronous Fanout<br/>Push to all online members<br/>in parallel]
    Count -->|Large 50 - 256 members| AsyncFanout[Async Fanout<br/>Add to worker queue<br/>Return immediately]
    SyncFanout --> Parallel[Parallel Push:<br/>10 goroutines<br/>5 members each]
    Parallel --> CheckOnline{For each member:<br/>Online?}
    CheckOnline -->|Online| Push[Push via WebSocket<br/>to member]
    CheckOnline -->|Offline| Queue[Add to offline queue<br/>LPUSH offline_messages]
    AsyncFanout --> WorkerPool[Fanout Worker Pool<br/>100 workers]
    WorkerPool --> BatchPush[Each worker handles<br/>10-20 members<br/>Parallel push]
    Push --> UpdateDelivered[Update delivery status:<br/>delivered_at = now]
    Queue --> Notification[Send push notification<br/>via FCM/APNS]
    BatchPush --> CheckOnline
    UpdateDelivered --> Done([End])
    Notification --> Done
    style SyncFanout fill: #e1ffe1
    style AsyncFanout fill: #fff4e1
    style WorkerPool fill: #e1f5ff
```

**Flow Explanation:**

Shows how group messages are distributed to all members efficiently.

**Fanout Strategies:**

**Small Groups (< 50 members):**

- Synchronous fanout
- Push to all online members in parallel (10 goroutines)
- Latency: ~100-200ms for all members
- Acceptable for small groups

**Large Groups (50-256 members):**

- Asynchronous fanout via worker queue
- User gets "SENT" ACK immediately
- Workers push in background over 500ms-1s
- Trade-off: Slight delay for better write performance

**Parallel Fanout Implementation:**

```
Message to 50 members:
  - 10 worker goroutines
  - Each handles 5 members
  - All push in parallel
  → Total time: max(individual push times) ≈ 100ms
```

**Optimization:**

1. **Online Priority:** Push to online members first, queue offline for later
2. **Batch WebSocket Writes:** Send to multiple connections simultaneously
3. **Compression:** Compress message once, send to all (save bandwidth)
4. **Smart Routing:** Members in same region get faster delivery

**Performance:**

- Small group (10 members): ~100ms all delivered
- Medium group (50 members): ~200ms all delivered
- Large group (256 members): ~500ms-1s all delivered

**Read Receipt Handling:**

- Small groups: Track individual read receipts
- Large groups: Only track "last read message ID" (too expensive otherwise)

---

## 9. Offline Message Handling

```mermaid
graph TB
    subgraph "Message Arrives for Offline User"
        MSG[Delivery Worker:<br/>Message for user 12345]
        CHECK{Check Presence:<br/>User online?}
    end

    subgraph "Offline Queue (Redis)"
        QUEUE[LPUSH offline_messages 12345<br/>Store message in list<br/>TTL: 7 days]
        COUNT[Check queue depth:<br/>LLEN offline_messages 12345]
    end

    subgraph "Push Notification"
        PUSH[Send Push Notification:<br/>FCM for Android<br/>APNS for iOS]
        BATCH{Queue depth > 10?}
    end

    subgraph "User Comes Online"
        CONNECT[User reconnects<br/>WebSocket established]
        FETCH[LRANGE offline_messages 12345<br/>Fetch all pending messages]
        DELIVER[Push all messages<br/>via WebSocket]
        CLEAR[DEL offline_messages 12345<br/>Clear queue]
    end

    MSG --> CHECK
    CHECK -->|Offline| QUEUE
    QUEUE --> COUNT
    COUNT --> BATCH
    BATCH -->|< = 10 messages| PUSH
    BATCH -->|> 10 messages| BATCH_PUSH[Batch notification:<br/>You have 47 new messages<br/>from Alice, Bob, Carol]
    PUSH --> DONE1([Notification Sent])
    BATCH_PUSH --> DONE1
    CONNECT --> FETCH
    FETCH --> DELIVER
    DELIVER --> CLEAR
    CLEAR --> DONE2([User Caught Up])
    style QUEUE fill: #fff4e1
    style PUSH fill: #e1f5ff
    style DELIVER fill: #e1ffe1
```

**Flow Explanation:**

Shows how messages are queued and delivered when recipient is offline.

**Offline Flow:**

1. **Delivery Worker Checks Presence** (1ms):
   ```
   GET user:12345:presence → NULL (offline)
   ```

2. **Add to Offline Queue** (2ms):
   ```
   LPUSH offline_messages:12345 {message_json}
   EXPIRE offline_messages:12345 604800  // 7 days TTL
   ```

3. **Check Queue Depth** (1ms):
   ```
   LLEN offline_messages:12345 → 47 messages
   ```

4. **Send Push Notification** (100-500ms):
    - **Single message:** "Alice: Hey, how are you?"
    - **Multiple messages (> 10):** "You have 47 new messages from Alice, Bob, Carol"
    - **Batching:** Avoid spamming user with 47 separate notifications

**Online Flow (Reconnection):**

1. **User Reconnects** (0ms): WebSocket connection established

2. **Fetch Offline Messages** (5ms):
   ```
   LRANGE offline_messages:12345 0 -1
   → Returns all 47 messages
   ```

3. **Deliver via WebSocket** (50ms): Push all messages to client

4. **Clear Queue** (2ms):
   ```
   DEL offline_messages:12345
   ```

5. **Update UI:** Client displays all missed messages in chronological order

**Data Structure:**

```
Key: offline_messages:{user_id}
Type: LIST (ordered by arrival time)
Value: [message1_json, message2_json, ..., messageN_json]
TTL: 7 days (auto-cleanup old messages)
```

**Benefits:**

- **No Message Loss:** All messages queued until delivered
- **Ordered Delivery:** LIST maintains arrival order
- **Efficient:** Single query fetches all pending messages
- **Auto-Cleanup:** 7-day TTL prevents indefinite growth

**Edge Cases:**

- **Queue too large (>1000 messages):** Fetch in batches of 100
- **User never comes online:** Messages expire after 7 days
- **Multiple devices:** Each device fetches and clears queue (idempotent)

---

## 10. Multi-Region Deployment

```mermaid
graph TB
    subgraph "US-East Region"
        US_CLIENT[US Clients]
        US_WS[WebSocket Servers]
        US_KAFKA[Kafka Cluster]
        US_CASS[Cassandra]
        US_REDIS[Redis]
    end

    subgraph "EU-West Region"
        EU_CLIENT[EU Clients]
        EU_WS[WebSocket Servers]
        EU_KAFKA[Kafka Cluster]
        EU_CASS[Cassandra]
        EU_REDIS[Redis]
    end

    subgraph "AP-South Region"
        AP_CLIENT[AP Clients]
        AP_WS[WebSocket Servers]
        AP_KAFKA[Kafka Cluster]
        AP_CASS[Cassandra]
        AP_REDIS[Redis]
    end

    subgraph "Global Services"
        GLOBAL_SEQ[Global Sequence Manager<br/>Raft Consensus<br/>Single source of truth]
        GLOBAL_DNS[Global DNS<br/>GeoDNS Routing<br/>Route to nearest region]
    end

    US_CLIENT --> GLOBAL_DNS
    EU_CLIENT --> GLOBAL_DNS
    AP_CLIENT --> GLOBAL_DNS
    GLOBAL_DNS -->|Route| US_WS
    GLOBAL_DNS -->|Route| EU_WS
    GLOBAL_DNS -->|Route| AP_WS
    US_WS --> US_KAFKA
    EU_WS --> EU_KAFKA
    AP_WS --> AP_KAFKA
    US_KAFKA -->|Cross - Region Replication| EU_KAFKA
    EU_KAFKA -->|Cross - Region Replication| AP_KAFKA
    AP_KAFKA -->|Cross - Region Replication| US_KAFKA
    US_WS -. Request Seq ID .-> GLOBAL_SEQ
    EU_WS -. Request Seq ID .-> GLOBAL_SEQ
    AP_WS -. Request Seq ID .-> GLOBAL_SEQ
    US_KAFKA --> US_CASS
    EU_KAFKA --> EU_CASS
    AP_KAFKA --> AP_CASS
    US_CASS -. Async Replication .-> EU_CASS
    EU_CASS -. Async Replication .-> AP_CASS
    style GLOBAL_SEQ fill: #ffe1e1
    style GLOBAL_DNS fill: #e1f5ff
```

**Flow Explanation:**

Shows multi-region architecture for global low-latency access.

**Architecture:**

1. **Regional Independence:** Each region has complete stack (WebSocket + Kafka + Cassandra + Redis)
2. **GeoDNS Routing:** Users automatically routed to nearest region
3. **Local Processing:** Messages processed locally (low latency)
4. **Global Sequence Manager:** Single source of truth for message ordering (Raft consensus)
5. **Cross-Region Replication:** Kafka messages replicated across regions (eventual consistency)

**Latency by Region:**

| User Location | Target Region | Latency |
|---------------|---------------|---------|
| US            | US-East       | ~10ms   |
| EU            | EU-West       | ~10ms   |
| Asia          | AP-South      | ~10ms   |
| US → EU       | Cross-region  | +100ms  |

**Message Flow (Cross-Region):**

**Scenario:** US user sends message to EU user

1. US user → US-East WebSocket (10ms)
2. US-East → Global Sequence Manager (50ms cross-region)
3. US-East → US-East Kafka (5ms)
4. Kafka replication US → EU (100ms async)
5. EU Delivery Worker → EU user WebSocket (10ms)
6. **Total:** ~175ms (still < 500ms target ✅)

**Trade-offs:**

**Benefits:**

- ✅ **Low Latency:** Users connect to nearest region
- ✅ **High Availability:** Region failure doesn't affect others
- ✅ **Compliance:** Data residency requirements met

**Costs:**

- ❌ **Complexity:** Multi-region coordination
- ❌ **Higher Latency:** Cross-region messages +100ms
- ❌ **3× Cost:** Infrastructure replicated 3 times

**When to Use:**

- After reaching 100M+ globally distributed users
- When regional latency becomes user complaint
- When compliance requires data residency

---

## 11. Sequence ID Generation

```mermaid
graph TB
    subgraph "Snowflake-Style 64-bit ID"
        BITS[Bit Allocation:<br/>41 bits Timestamp<br/>10 bits Node ID<br/>13 bits Sequence]
    end

    subgraph "ID Components"
        TS[Timestamp 41 bits:<br/>Milliseconds since epoch<br/>Range: 69 years]
        NODE[Node ID 10 bits:<br/>Unique server ID<br/>Range: 0-1023 servers]
        SEQ[Sequence 13 bits:<br/>Counter per millisecond<br/>Range: 0-8191 IDs/ms]
    end

    subgraph "Generation Process"
        REQ[Message Router<br/>Requests ID]
        CURR[Get current timestamp:<br/>now ms]
        CHECK{Same millisecond?}
        INCR[Increment sequence<br/>counter]
        WAIT[Wait 1ms<br/>Reset counter]
        BUILD[Build ID:<br/>timestamp node_id sequence]
    end

    subgraph "Example IDs"
        ID1[ID: 8796093022208000<br/>Time: 2024-01-01 10:00:00<br/>Node: 42<br/>Seq: 0]
        ID2[ID: 8796093022208001<br/>Time: 2024-01-01 10:00:00<br/>Node: 42<br/>Seq: 1]
    end

    BITS --> TS
    BITS --> NODE
    BITS --> SEQ
    REQ --> CURR
    CURR --> CHECK
    CHECK -->|Yes| INCR
    CHECK -->|No| BUILD
    INCR -->|< 8191| BUILD
    INCR -->|= 8191| WAIT
    WAIT --> CURR
    BUILD --> ID1
    BUILD --> ID2
    style BITS fill: #e1ffe1
    style BUILD fill: #fff4e1
```

**Flow Explanation:**

Shows Snowflake-style distributed ID generation that ensures global uniqueness and time-ordering.

**ID Format (64 bits):**

```
[41 bits: Timestamp] [10 bits: Node ID] [13 bits: Sequence]

Example:
  0000000001110101100110100000000000000 0000101010 0000000000001
  |--- 41 bits timestamp ----------------|  |10 bits|  |-13 bits-|
  
  Timestamp: 1704067200000 (2024-01-01 10:00:00 UTC)
  Node ID: 42 (this server)
  Sequence: 1 (first ID this millisecond)
```

**Generation Algorithm:**

```
function generate_sequence_id():
    current_timestamp = now_milliseconds()
    
    if current_timestamp == last_timestamp:
        // Same millisecond: increment sequence
        sequence = (sequence + 1) & 8191  // Mask to 13 bits
        
        if sequence == 0:
            // Sequence exhausted, wait for next millisecond
            while now_milliseconds() <= last_timestamp:
                wait(1ms)
            current_timestamp = now_milliseconds()
    else:
        // New millisecond: reset sequence
        sequence = 0
    
    last_timestamp = current_timestamp
    
    // Build 64-bit ID
    id = (current_timestamp << 23) | (node_id << 13) | sequence
    
    return id
```

**Properties:**

1. **Globally Unique:**
    - Node ID ensures no collision between servers
    - Sequence ensures no collision within same millisecond
    - Timestamp ensures no collision across time

2. **Time-Ordered:**
    - Sort by ID = sort by time (roughly)
    - Useful for "show latest messages" queries

3. **Distributed:**
    - Each server generates IDs independently
    - No coordination needed (fast)
    - No single point of failure

**Performance:**

- Generation: ~0.1 microseconds (CPU-bound)
- Throughput: 8192 IDs per millisecond per node
- With 1000 nodes: 8.192 million IDs per millisecond = 8 billion IDs/second

**Clock Skew Handling:**

- NTP sync every 5 minutes
- Alert if drift > 1 second
- Sequence bits handle bursts within millisecond

**Alternative: Kafka Offset**

- Use Kafka partition offset as sequence number
- Simpler (no Sequence Manager)
- Trade-off: Couples to Kafka, no timestamp encoding

---

## 12. Monitoring Dashboard

```mermaid
graph TB
    subgraph "Real-Time Metrics"
        M1[Active Connections:<br/>99.8M / 100M<br/>Status: Healthy]
        M2[Message Throughput:<br/>745K msg/sec<br/>Target: < 870K]
        M3[Delivery Latency P99:<br/>385ms<br/>Target: < 500ms]
    end

    subgraph "Kafka Health"
        K1[Consumer Lag:<br/>History: 342 msgs<br/>Delivery: 127 msgs<br/>Status: Good]
        K2[Partition Lag:<br/>Max: 890 msgs<br/>Avg: 245 msgs<br/>Status: Good]
        K3[Broker Health:<br/>50 brokers online<br/>Replication: OK]
    end

    subgraph "Storage Health"
        S1[Cassandra Write Latency:<br/>P99: 42ms<br/>Target: < 50ms]
        S2[Cassandra Storage:<br/>38 PB / 45 PB<br/>Usage: 84 percent]
        S3[Redis Hit Rate:<br/>Presence: 98.7 percent<br/>Message Cache: 92.3 percent]
    end

    subgraph "Alerts Active"
        A1[WARNING:<br/>Cassandra shard 42<br/>CPU at 85 percent]
        A2[INFO:<br/>New WebSocket server<br/>added ws-1001]
    end

    subgraph "System Health Score"
        SCORE[Overall: 98.5 / 100<br/>Status: Excellent]
    end

    M1 --> SCORE
    M2 --> SCORE
    M3 --> SCORE
    K1 --> SCORE
    S1 --> SCORE
    A1 -. Impacts .-> SCORE
    style M3 fill: #e1ffe1
    style K1 fill: #e1ffe1
    style A1 fill: #fff4e1
    style SCORE fill: #e1f5ff
```

**Flow Explanation:**

Shows key metrics and alerts for monitoring Live Chat System health.

**Critical Metrics:**

**Real-Time Performance:**

- **Active Connections:** 99.8M / 100M capacity (99.8% utilization)
- **Message Throughput:** 745K msg/sec current (vs 870K peak capacity)
- **Delivery Latency:** P99 = 385ms (target < 500ms) ✅

**Kafka Health:**

- **Consumer Lag:** < 1000 messages (good)
    - History Workers: 342 messages behind
    - Delivery Workers: 127 messages behind
- **Partition Health:** All 1000 partitions healthy
- **Broker Health:** All 50 brokers online, replication OK

**Storage Health:**

- **Cassandra:**
    - Write latency P99: 42ms (target < 50ms) ✅
    - Storage: 38 PB / 45 PB (84% full, plan expansion)
    - Node health: 98/100 nodes healthy
- **Redis:**
    - Presence hit rate: 98.7% (excellent)
    - Message cache hit rate: 92.3% (good)
    - Memory: 560 GB / 640 GB (87% used)

**Alerts:**

- 🟡 **WARNING:** Cassandra shard 42 CPU at 85% (threshold: 80%)
    - Action: Add more nodes or rebalance load
- 🟢 **INFO:** New WebSocket server ws-1001 added (auto-scaling)

**Dashboards:**

1. **Overview Dashboard:**
    - System health score
    - Active connections timeline
    - Message throughput graph
    - Delivery latency heatmap

2. **Kafka Dashboard:**
    - Consumer lag per group
    - Partition distribution
    - Broker CPU/memory
    - Replication lag

3. **Storage Dashboard:**
    - Cassandra write/read latency
    - Storage usage trends
    - Redis hit rates
    - Presence query latency

4. **Alerts Dashboard:**
    - Active alerts by severity
    - Alert history
    - On-call escalation status

**Critical Alerts:**

- 🔴 **CRITICAL:** Kafka consumer lag > 10K messages
- 🔴 **CRITICAL:** Delivery latency P99 > 1 second
- 🔴 **CRITICAL:** WebSocket server down (100K users affected)
- 🟡 **WARNING:** Cassandra write latency > 100ms
- 🟡 **WARNING:** Redis hit rate < 90%