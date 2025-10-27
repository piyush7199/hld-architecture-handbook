# Live Chat System - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made in the Live Chat System, including alternatives considered, trade-offs, and conditions that would change these decisions.

---

## Table of Contents

1. [WebSockets vs Long Polling vs Server-Sent Events](#1-websockets-vs-long-polling-vs-server-sent-events)
2. [Kafka vs Database for Message Ordering](#2-kafka-vs-database-for-message-ordering)
3. [Cassandra vs PostgreSQL vs MongoDB for History](#3-cassandra-vs-postgresql-vs-mongodb-for-history)
4. [Redis vs Memcached vs Database for Presence](#4-redis-vs-memcached-vs-database-for-presence)
5. [Centralized vs Distributed Sequence Manager](#5-centralized-vs-distributed-sequence-manager)
6. [Fanout-on-Write vs Fanout-on-Read for Group Chat](#6-fanout-on-write-vs-fanout-on-read-for-group-chat)
7. [Single-Region vs Multi-Region Architecture](#7-single-region-vs-multi-region-architecture)

---

## 1. WebSockets vs Long Polling vs Server-Sent Events

### The Problem

Need to deliver messages to clients in real-time (< 500ms) with bidirectional communication. Must support 100M concurrent connections efficiently.

**Constraints:**
- Sub-500ms latency requirement
- Both client→server and server→client communication needed
- 100M persistent connections
- Mobile clients (battery efficiency matters)

### Options Considered

| Approach | Latency | Bidirectional | Overhead | Battery Impact | Complexity |
|----------|---------|---------------|----------|----------------|------------|
| **WebSockets** | <50ms | ✅ Yes | Low | Low | Medium |
| **Long Polling** | 1-5s | ✅ Yes | High | High | Low |
| **Server-Sent Events** | <100ms | ❌ No | Medium | Medium | Low |
| **HTTP/2 Server Push** | <100ms | ❌ No | Medium | Medium | Medium |
| **WebRTC Data Channel** | <50ms | ✅ Yes | Low | Low | High |

#### Option A: WebSockets ✅ (Chosen)

**How It Works:**
- Single persistent TCP connection upgraded from HTTP
- Full-duplex: Both client and server can send messages anytime
- Binary or text frames with minimal overhead

**Pros:**
- ✅ **Real-Time:** <50ms latency (no polling delay)
- ✅ **Bidirectional:** Both directions over single connection
- ✅ **Efficient:** No HTTP headers per message (saves bandwidth)
- ✅ **Low Overhead:** ~2 bytes per frame vs 200+ bytes HTTP headers
- ✅ **Battery Friendly:** Single connection, no constant polling

**Cons:**
- ❌ **Stateful:** Requires sticky sessions (connection tied to server)
- ❌ **Scale Complexity:** Managing 100M connections requires 1000 servers
- ❌ **Firewall Issues:** Some corporate firewalls block WebSocket
- ❌ **No HTTP Caching:** Can't leverage CDN/HTTP cache

**Implementation:**
```
Client initiates HTTP request with Upgrade header
Server responds with 101 Switching Protocols
Connection upgraded to WebSocket (ws:// or wss://)
Persistent connection maintained with heartbeats every 30s
```

#### Option B: Long Polling (Not Chosen)

**How It Works:**
- Client sends HTTP request
- Server holds request open until new message arrives (or timeout)
- Server responds with message
- Client immediately opens new request

**Pros:**
- ✅ **Simple:** Standard HTTP, works everywhere
- ✅ **Firewall Friendly:** Uses port 80/443

**Cons:**
- ❌ **High Latency:** 1-5 second delay between messages
- ❌ **Inefficient:** Constant connection open/close cycles
- ❌ **Server Load:** 100M clients polling = massive overhead
- ❌ **Battery Drain:** Constant network activity kills mobile battery
- ❌ **HTTP Overhead:** 200+ bytes headers on every poll

**Why Not Chosen:** Cannot meet <500ms latency requirement. 100M clients polling every second = 100M req/sec even with no messages.

#### Option C: Server-Sent Events (SSE) (Not Chosen)

**How It Works:**
- Client opens HTTP connection
- Server can push messages to client
- Client must use separate HTTP POST for uploads

**Pros:**
- ✅ **Simple:** Built into HTML5, easy to use
- ✅ **Server Push:** Low latency for server→client messages
- ✅ **Automatic Reconnect:** Browser handles reconnection

**Cons:**
- ❌ **One-Way Only:** Server can push, but client must POST separately
- ❌ **Browser Limits:** Max 6 SSE connections per domain
- ❌ **No Binary:** Text only (JSON encoding overhead)
- ❌ **HTTP Overhead:** Still sends HTTP headers

**Why Not Chosen:** Not bidirectional. Would need separate HTTP POST for client→server, doubling connection count.

### Decision Made

**✅ WebSockets with sticky session load balancing**

**Rationale:**
1. **Latency:** <50ms meets requirement (<500ms)
2. **Efficiency:** Minimal overhead (2 bytes vs 200+ bytes HTTP)
3. **Battery:** Single connection vs constant polling
4. **Bidirectional:** Both directions over one connection
5. **Industry Standard:** WhatsApp, Slack, Discord all use WebSockets

**Configuration:**
```
Load Balancer: NGINX with sticky sessions (hash by user_id)
WebSocket Servers: Go/Erlang (handle 100K connections each)
Heartbeat: Every 30 seconds (detect dead connections)
Reconnect: Exponential backoff (1s, 2s, 4s, 8s)
```

### Trade-offs Accepted

| Trade-off | Impact | Mitigation |
|-----------|--------|------------|
| **Stateful Servers** | Must route same user to same server | Sticky sessions via consistent hashing |
| **Firewall Issues** | Some corporate networks block WebSocket | Fallback to Long Polling |
| **Connection Management** | Need 1000 servers for 100M connections | Horizontal scaling, OS tuning |

### When to Reconsider

**Switch to Long Polling if:**
- Firewall/proxy issues affect >10% of users
- Real-time latency requirement relaxes to >5 seconds
- Connection count drops below 1M (simpler architecture)

**Switch to WebRTC if:**
- Need peer-to-peer communication (voice/video)
- Want to bypass server for direct client connections
- Have expertise in WebRTC complexity

---

## 2. Kafka vs Database for Message Ordering

### The Problem

Must guarantee strict message ordering within each conversation. Messages must never be lost, even during failures. Need to handle 290K messages/sec average, 870K peak.

**Constraints:**
- Strict ordering per chat_id
- Zero message loss (durability)
- High throughput (870K msg/sec peak)
- Decoupled read/write paths

### Options Considered

| System | Ordering | Durability | Throughput | Replayability | Complexity |
|--------|----------|------------|------------|---------------|------------|
| **Kafka** | ✅ Per partition | ✅ Replicated log | ✅ 1M+ msg/sec | ✅ Yes | High |
| **PostgreSQL** | ⚠️ Best effort | ✅ WAL | ❌ 10K/sec | ❌ No | Low |
| **MongoDB** | ⚠️ Best effort | ✅ Oplog | ⚠️ 50K/sec | ⚠️ Limited | Medium |
| **RabbitMQ** | ⚠️ Best effort | ✅ Queues | ⚠️ 20K/sec | ❌ No | Medium |
| **Redis Streams** | ✅ Per stream | ⚠️ AOF | ⚠️ 100K/sec | ✅ Yes | Low |

#### Option A: Kafka ✅ (Chosen)

**How It Works:**
- Messages partitioned by chat_id
- Each partition is ordered, append-only log
- Replication factor 3 (message on 3 brokers)
- Consumers track offset (position in log)

**Ordering Guarantee:**
```
Messages with same chat_id always go to same partition
Partition guarantees sequential ordering
→ Messages for chat_id=123 always ordered ✅
```

**Pros:**
- ✅ **Strict Ordering:** Guaranteed per partition (per chat)
- ✅ **High Throughput:** 1M+ messages/sec
- ✅ **Durability:** Replicated across 3 brokers
- ✅ **Replayability:** Can reprocess from any offset
- ✅ **Decoupling:** Write and read paths independent
- ✅ **Scale:** Horizontal scaling via partitions

**Cons:**
- ❌ **Complexity:** Requires Kafka expertise to operate
- ❌ **Operational Overhead:** 50+ broker cluster
- ❌ **Consumer Management:** Must track offsets carefully
- ❌ **Storage Cost:** Retain messages for replay (7 days = 175TB)

**Configuration:**
```
Topics: chat-messages (1000 partitions)
Replication: 3 replicas
Retention: 7 days (replayability window)
Producer: acks=all (wait for all replicas)
Consumer Groups: history-writers, delivery-workers
```

#### Option B: PostgreSQL (Not Chosen)

**How It Works:**
- Messages written to `messages` table
- Ordered by timestamp + auto-increment ID
- Replication via streaming replication

**Pros:**
- ✅ **Simple:** Familiar SQL database
- ✅ **ACID:** Strong consistency guarantees
- ✅ **Transactions:** Atomic multi-row operations

**Cons:**
- ❌ **No Ordering Guarantee:** Concurrent inserts can arrive out-of-order
  - Example: Transaction 1 starts first but commits after Transaction 2
- ❌ **Low Throughput:** ~10K writes/sec (vs 290K required)
- ❌ **Write Bottleneck:** Single master, can't shard easily
- ❌ **No Replayability:** Once read, message consumed
- ❌ **Lock Contention:** High concurrency causes lock waits

**Why Not Chosen:** Cannot handle 290K writes/sec. No ordering guarantee for concurrent writes.

#### Option C: RabbitMQ (Not Chosen)

**How It Works:**
- Messages sent to queues
- Consumers pull from queues
- Acknowledgment-based delivery

**Pros:**
- ✅ **Mature:** Battle-tested message queue
- ✅ **Flexible Routing:** Exchanges, routing keys
- ✅ **Dead Letter Queues:** Handle failures

**Cons:**
- ❌ **No Strict Ordering:** Messages can be processed out-of-order
- ❌ **Lower Throughput:** ~20K msg/sec per queue
- ❌ **No Replayability:** Messages deleted after ACK
- ❌ **Stateful:** Queues store state (vs Kafka's stateless brokers)

**Why Not Chosen:** No strict ordering guarantee. Cannot replay messages.

### Decision Made

**✅ Kafka as distributed message log**

**Rationale:**
1. **Ordering:** Guarantees strict order per partition (per chat)
2. **Throughput:** Handles 870K peak messages/sec
3. **Durability:** 3× replication, zero message loss
4. **Replayability:** Can recover from failures by replaying log
5. **Decoupling:** History and delivery workers independent

**Architecture:**
```
Producer (Message Router)
   ↓
Kafka (1000 partitions by chat_id)
   ↓
Consumer Group 1: History Workers → Cassandra
Consumer Group 2: Delivery Workers → WebSocket Servers
```

### Trade-offs Accepted

| Trade-off | Impact | Mitigation |
|-----------|--------|------------|
| **Operational Complexity** | Need Kafka expertise | Managed service (Confluent Cloud) or dedicated team |
| **Storage Cost** | 7 days retention = 175 TB | Compress messages (70% reduction) |
| **Consumer Lag** | If consumer falls behind, latency increases | Monitor lag, alert if >1000 messages |

### When to Reconsider

**Switch to PostgreSQL if:**
- Message volume drops below 10K/sec
- Strict ordering not required
- Team has no Kafka expertise

**Switch to Redis Streams if:**
- Message volume <100K/sec
- Cost is primary concern
- Willing to sacrifice some durability

---

## 3. Cassandra vs PostgreSQL vs MongoDB for History

### The Problem

Store 45 PB of message history (5 years). Support 25 TB/day writes. Query pattern: "Fetch last 100 messages for chat_id, ordered by timestamp."

**Constraints:**
- Write-heavy: 25 TB/day new messages
- Time-series data (chat_id + timestamp)
- Horizontal scaling required (45 PB)
- High availability (99.99%)

### Options Considered

| Database | Write Throughput | Scaling | Time-Series | Cost (45 PB) |
|----------|------------------|---------|-------------|--------------|
| **Cassandra** | ✅ Excellent | ✅ Horizontal | ✅ Optimized | $981K/year |
| **PostgreSQL** | ❌ Limited | ❌ Vertical | ⚠️ OK | $2M+/year |
| **MongoDB** | ⚠️ Good | ✅ Horizontal | ⚠️ OK | $1.5M/year |
| **DynamoDB** | ✅ Excellent | ✅ Automatic | ⚠️ OK | $3M+/year |
| **S3 + Parquet** | ✅ Unlimited | ✅ Infinite | ❌ Poor | $180K/year |

#### Option A: Cassandra ✅ (Chosen)

**Data Model:**
```sql
CREATE TABLE messages (
    chat_id UUID,
    timestamp TIMESTAMP,
    message_id UUID,
    sender_id UUID,
    content TEXT,
    PRIMARY KEY (chat_id, timestamp, message_id)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

**Pros:**
- ✅ **Write-Optimized:** Append-only LSM tree (no disk seeks)
- ✅ **Horizontal Scaling:** Add nodes linearly (100 nodes for 45 PB)
- ✅ **Time-Series:** Optimized for timestamp-based queries
- ✅ **High Availability:** RF=3, no single point of failure
- ✅ **Fast Reads:** Partition by chat_id, cluster by timestamp

**Cons:**
- ❌ **No Joins:** Can't query across multiple tables
- ❌ **Limited Queries:** Must query by partition key (chat_id)
- ❌ **Eventual Consistency:** Tunable, but not strong by default
- ❌ **Ops Complexity:** Requires expertise (compaction, repair)

**Query Performance:**
```
Query: Last 100 messages for chat_id
SELECT * FROM messages WHERE chat_id = ? ORDER BY timestamp DESC LIMIT 100
Latency: 10-50ms (single partition, no joins)
```

**Why Chosen:** Perfect fit for time-series, write-heavy workload. Scales horizontally to 45 PB.

#### Option B: PostgreSQL (Not Chosen)

**Data Model:**
```sql
CREATE TABLE messages (
    message_id UUID PRIMARY KEY,
    chat_id UUID,
    timestamp TIMESTAMP,
    sender_id UUID,
    content TEXT
);
CREATE INDEX idx_chat_timestamp ON messages(chat_id, timestamp DESC);
```

**Pros:**
- ✅ **Rich Queries:** Joins, complex WHERE clauses
- ✅ **Strong Consistency:** ACID transactions
- ✅ **Mature:** Well-understood, large ecosystem

**Cons:**
- ❌ **Write Bottleneck:** ~10K writes/sec per instance
- ❌ **Vertical Scaling:** Limited to single instance size
- ❌ **Sharding Hard:** Manual sharding complex
- ❌ **Cost:** 45 PB would require 100+ instances + Citus ($2M+/year)

**Why Not Chosen:** Cannot handle 290K writes/sec. Scaling to 45 PB too expensive and complex.

#### Option C: MongoDB (Not Chosen)

**Data Model:**
```javascript
{
  _id: ObjectId(),
  chat_id: UUID,
  timestamp: ISODate(),
  sender_id: UUID,
  content: String
}
Index: { chat_id: 1, timestamp: -1 }
```

**Pros:**
- ✅ **Flexible Schema:** Can store nested objects, attachments
- ✅ **Horizontal Scaling:** Sharding built-in
- ✅ **Rich Queries:** Aggregation framework

**Cons:**
- ❌ **Write Performance:** 50K writes/sec (vs 290K needed)
- ❌ **Storage Overhead:** BSON encoding + indexes = 2× size
- ❌ **Cost:** $1.5M/year for 45 PB (vs $981K Cassandra)
- ❌ **Time-Series:** Not optimized (Cassandra better for this)

**Why Not Chosen:** Lower write throughput, higher cost than Cassandra.

### Decision Made

**✅ Cassandra with 100-node cluster**

**Rationale:**
1. **Write Throughput:** Handles 290K writes/sec easily
2. **Horizontal Scaling:** Linear scaling to 45 PB
3. **Time-Series:** Optimized for chat_id + timestamp queries
4. **Cost:** $981K/year (vs $2M+ PostgreSQL, $1.5M MongoDB)
5. **Availability:** RF=3, survives 2 node failures

**Configuration:**
```
Cluster: 100 nodes (i3.2xlarge, 1.9 TB SSD each)
Replication Factor: 3
Consistency Level: QUORUM (2 out of 3)
Compaction: Time-Window (optimize for time-series)
TTL: None (keep all messages, archive to S3 after 1 year)
```

### Trade-offs Accepted

| Trade-off | Impact | Mitigation |
|-----------|--------|------------|
| **No Joins** | Can't query across tables | Denormalize data (store sender name in message) |
| **Eventual Consistency** | Reads may lag writes slightly | Use QUORUM consistency for critical reads |
| **Ops Complexity** | Requires Cassandra expertise | Managed service (Astra DB) or dedicated team |

### When to Reconsider

**Switch to PostgreSQL if:**
- Message volume <1 TB total
- Complex relational queries needed
- Strong consistency critical

**Switch to DynamoDB if:**
- Want fully managed solution
- Cost not a concern ($3M/year acceptable)
- AWS-only deployment

---

## 4. Redis vs Memcached vs Database for Presence

### The Problem

Track online/offline status for 1B users. Must support sub-100ms lookups. Presence changes frequently (every 30s heartbeat). Support batch queries (group chat with 50 members).

**Constraints:**
- Sub-100ms read latency
- 100M concurrent users online
- Frequent updates (heartbeat every 30s)
- Batch queries (get presence for 50 users)

### Options Considered

| System | Read Latency | Write Latency | TTL Support | Batch Queries | Cost (100M users) |
|--------|--------------|---------------|-------------|---------------|-------------------|
| **Redis** | <1ms | <1ms | ✅ Yes | ✅ MGET | $43.5K/year |
| **Memcached** | <1ms | <1ms | ✅ Yes | ❌ No | $30K/year |
| **PostgreSQL** | 10-50ms | 10-50ms | ❌ No | ✅ Yes | $100K/year |
| **MongoDB** | 5-10ms | 5-10ms | ✅ Yes | ✅ Yes | $80K/year |
| **DynamoDB** | 5-10ms | 5-10ms | ✅ TTL | ⚠️ BatchGet | $200K/year |

#### Option A: Redis ✅ (Chosen)

**Data Model:**
```
Key: user:{user_id}:presence
Value: "online"
TTL: 60 seconds (auto-expire if no heartbeat)
```

**Pros:**
- ✅ **Sub-Millisecond:** <1ms reads and writes
- ✅ **TTL Support:** Auto-expire offline users
- ✅ **Batch Queries:** MGET for multiple users at once
- ✅ **Data Structures:** Can store complex presence (last_seen, device)
- ✅ **Pub/Sub:** Can broadcast presence changes

**Cons:**
- ❌ **Memory Cost:** All data in RAM ($43.5K/year)
- ❌ **Persistence:** Data lost if cluster fails (need AOF/RDB)
- ❌ **Single-Threaded:** CPU-bound on single core

**Operations:**
```
Set presence: SET user:123:presence "online" EX 60
Get presence: GET user:123:presence
Batch get: MGET user:123:presence user:456:presence user:789:presence
Heartbeat: EXPIRE user:123:presence 60 (refresh TTL)
```

**Why Chosen:** Fastest option (<1ms), supports TTL and batch queries.

#### Option B: Memcached (Not Chosen)

**Pros:**
- ✅ **Fast:** <1ms latency
- ✅ **Simple:** Minimal features, easy to operate
- ✅ **Cheaper:** $30K/year (vs $43.5K Redis)

**Cons:**
- ❌ **No Batch Queries:** Must GET each key individually
  - Fetching 50 users = 50 network roundtrips (vs 1 for Redis MGET)
- ❌ **No Pub/Sub:** Can't broadcast presence changes
- ❌ **Limited Data Types:** Only strings

**Why Not Chosen:** No MGET (batch queries critical for group chat).

#### Option C: PostgreSQL (Not Chosen)

**Data Model:**
```sql
CREATE TABLE presence (
    user_id UUID PRIMARY KEY,
    status VARCHAR(10),
    last_seen TIMESTAMP
);
CREATE INDEX idx_status ON presence(status);
```

**Pros:**
- ✅ **Persistent:** Data survives restarts
- ✅ **Rich Queries:** Can query by status, last_seen

**Cons:**
- ❌ **Slow:** 10-50ms vs <1ms Redis
- ❌ **No TTL:** Must manually delete stale records
- ❌ **Write Load:** 100M heartbeats every 30s = 3.3M writes/sec (impossible)

**Why Not Chosen:** 100× slower than Redis. Cannot handle 3.3M heartbeat writes/sec.

### Decision Made

**✅ Redis Cluster (20 nodes)**

**Rationale:**
1. **Speed:** <1ms meets sub-100ms requirement
2. **TTL:** Auto-expire offline users (no manual cleanup)
3. **Batch Queries:** MGET for group chat (50 users, 1 roundtrip)
4. **Pub/Sub:** Broadcast presence changes to online users
5. **Cost:** $43.5K/year reasonable

**Configuration:**
```
Cluster: 20 nodes (r5.xlarge, 32 GB RAM each)
Sharding: By user_id (consistent hashing)
Persistence: AOF every second
Eviction: allkeys-lru (if memory full, evict least used)
```

### Trade-offs Accepted

| Trade-off | Impact | Mitigation |
|-----------|--------|------------|
| **Memory Cost** | $43.5K/year for 100M users | Acceptable for critical feature |
| **Volatile** | Data lost if all nodes fail | AOF persistence, Kafka has source of truth |
| **CPU-Bound** | Single-threaded per core | Shard across 20 nodes |

### When to Reconsider

**Switch to Memcached if:**
- Batch queries not needed
- Want to save $13.5K/year

**Switch to PostgreSQL if:**
- Latency requirement relaxes to >10ms
- Need complex queries on presence data

---

## 5. Centralized vs Distributed Sequence Manager

### The Problem

Generate strictly increasing sequence IDs for messages to ensure ordering. Must handle 870K messages/sec peak. IDs must be globally unique and time-ordered.

### Options Considered

| Approach | Ordering | Throughput | Complexity | SPOF |
|----------|----------|------------|------------|------|
| **Centralized (ZooKeeper)** | ✅ Strict | ⚠️ 100K/sec | Low | ⚠️ Yes (mitigated) |
| **Snowflake (Distributed)** | ✅ Strict | ✅ 1M+/sec | Medium | ✅ No |
| **UUID v4** | ❌ No order | ✅ Unlimited | Low | ✅ No |
| **Kafka Offset** | ✅ Per partition | ✅ 1M+/sec | Low | ✅ No |

#### Option A: Snowflake-style Distributed IDs ✅ (Chosen)

**ID Format (64-bit):**
```
[41 bits: Timestamp] [10 bits: Node ID] [13 bits: Sequence]
```

**How It Works:**
- Each Message Router has unique Node ID (0-1023)
- Timestamp: milliseconds since epoch
- Sequence: increments for IDs within same millisecond

**Pros:**
- ✅ **No SPOF:** Each router generates IDs independently
- ✅ **High Throughput:** 1M+ IDs/sec
- ✅ **Time-Ordered:** Sort by ID = sort by time
- ✅ **Globally Unique:** Node ID ensures no collisions

**Cons:**
- ❌ **Clock Skew:** Different servers have different clocks
- ❌ **Node ID Management:** Must assign unique IDs to each router

**Mitigation (Clock Skew):**
- NTP sync every 5 minutes
- Alert if drift > 1 second
- Use sequence bits to handle burst within millisecond

**Why Chosen:** No single point of failure, handles 1M+ req/sec.

#### Option B: Centralized (ZooKeeper) (Not Chosen for Default)

**How It Works:**
- ZooKeeper maintains counter per chat_id
- Message Router requests next ID from ZooKeeper
- ZooKeeper increments and returns

**Pros:**
- ✅ **Perfect Ordering:** Central counter guarantees strict sequence
- ✅ **Simple:** No clock sync issues

**Cons:**
- ❌ **Bottleneck:** ZooKeeper limited to ~100K req/sec
- ❌ **Latency:** Network roundtrip to ZooKeeper adds 5-10ms
- ❌ **SPOF:** If ZooKeeper unavailable, no new messages

**Mitigation (Pre-allocated Ranges):**
```
Router requests block of 1000 IDs from ZooKeeper
Router uses local IDs (no ZooKeeper call)
When block exhausted, request new block
→ 99% reduction in ZooKeeper calls
```

**When to Use:** Small deployments (<10K msg/sec) where simplicity matters.

#### Option C: Kafka Offset (Alternative)

**How It Works:**
- Use Kafka partition offset as sequence number
- Kafka guarantees strictly increasing offsets per partition

**Pros:**
- ✅ **Perfect Ordering:** Kafka partition offset already ordered
- ✅ **No Extra Service:** Leverage existing Kafka infrastructure
- ✅ **Simple:** No Sequence Manager needed

**Cons:**
- ❌ **Coupling:** Tightly coupled to Kafka implementation
- ❌ **No Timestamp:** Offset doesn't encode time
- ❌ **Can't Pre-generate:** Must write to Kafka first to get offset

**When to Use:** Want to eliminate Sequence Manager entirely.

### Decision Made

**✅ Snowflake-style distributed IDs**

**Rationale:**
1. **No SPOF:** Each router independent
2. **High Throughput:** Handles 870K peak
3. **Time-Ordered:** Sort by ID = chronological
4. **Decoupled:** Not tied to Kafka

**Alternative:** Use Kafka offset (simpler, but couples to Kafka).

### Trade-offs Accepted

| Trade-off | Impact | Mitigation |
|-----------|--------|------------|
| **Clock Skew** | Messages may be slightly out-of-order | NTP sync, sequence bits handle bursts |
| **Node ID Management** | Must assign unique IDs | Automated via service registry |

### When to Reconsider

**Switch to Centralized if:**
- Perfect ordering more important than throughput
- Message volume <100K/sec
- Willing to accept bottleneck

**Switch to Kafka Offset if:**
- Want to eliminate Sequence Manager
- Don't need timestamp in ID

---

## 6. Fanout-on-Write vs Fanout-on-Read for Group Chat

### The Problem

User sends message to group with 256 members. Must deliver to all members efficiently. Trade-off: write amplification vs read amplification.

### Options Considered

| Strategy | Write Cost | Read Cost | Latency | Storage |
|----------|------------|-----------|---------|---------|
| **Fanout-on-Write (Push)** | High (256 writes) | Low (1 read) | Low for reads | High (256 copies) |
| **Fanout-on-Read (Pull)** | Low (1 write) | High (256 reads) | High for reads | Low (1 copy) |
| **Hybrid** | Medium | Medium | Medium | Medium |

#### Option A: Fanout-on-Write (Push) (Chosen for Small Groups)

**How It Works:**
- When message sent, immediately push to all online members
- Store in each member's inbox

**Pros:**
- ✅ **Fast Reads:** User opens chat, messages already there
- ✅ **Pre-computed:** No aggregation needed at read time
- ✅ **Notification:** Know which users to notify

**Cons:**
- ❌ **Write Amplification:** 1 message → 256 writes
- ❌ **Storage:** 256 copies of same message
- ❌ **Slow Writes:** Must wait for all pushes to complete

**Use For:** Small groups (< 50 members)

#### Option B: Fanout-on-Read (Pull) (Chosen for Large Groups)

**How It Works:**
- Store message once in group inbox
- On read, fetch all messages from group

**Pros:**
- ✅ **Fast Writes:** Single write, immediate ACK
- ✅ **Storage Efficient:** Only 1 copy
- ✅ **Scalable:** Works for 1000+ member groups

**Cons:**
- ❌ **Slow Reads:** Must aggregate from group inbox
- ❌ **Read Amplification:** Every member reads same data

**Use For:** Large groups (50-256 members)

### Decision Made

**✅ Hybrid: Push for small groups (< 50), Pull for large groups (50-256)**

**Rationale:**
- 95% of groups have < 50 members (fast reads critical)
- 5% large groups (write efficiency critical)

**Configuration:**
```
Small groups (< 50): Push to all members (100-200ms)
Large groups (50-256): Pull from group inbox (500ms-1s)
```

### When to Reconsider

**Switch to Pull-Only if:**
- Most groups are large (> 100 members)
- Write throughput is bottleneck

---

## 7. Single-Region vs Multi-Region Architecture

### The Problem

Users are global. Need low latency worldwide. Must ensure message ordering across regions.

### Options Considered

| Architecture | Latency (US) | Latency (EU) | Latency (Asia) | Consistency | Complexity | Cost |
|--------------|--------------|--------------|----------------|-------------|------------|------|
| **Single-Region (US)** | 10ms | 150ms | 250ms | ✅ Strong | Low | 1× |
| **Multi-Region (Independent)** | 10ms | 10ms | 10ms | ❌ Eventual | Medium | 3× |
| **Multi-Region (Global Sequence)** | 10ms | 10ms | 10ms | ✅ Strong | High | 3× + latency |

#### Option A: Single-Region (US-East) (Chosen for MVP)

**Pros:**
- ✅ **Simplicity:** Single Kafka cluster, single Cassandra
- ✅ **Strong Consistency:** Global message ordering guaranteed
- ✅ **Lower Cost:** 1× infrastructure

**Cons:**
- ❌ **High Latency:** EU/Asia users experience 150-250ms

**Use For:** MVP, initial launch (optimize later).

#### Option B: Multi-Region with Global Sequence Manager (Future)

**Architecture:**
```
Region US          Region EU          Region Asia
  ↓                  ↓                   ↓
Global Sequence Manager (Raft consensus)
  ↓
Kafka Multi-Region Replication
```

**Pros:**
- ✅ **Low Latency:** Users connect to nearest region
- ✅ **Strong Consistency:** Global ordering maintained

**Cons:**
- ❌ **Complexity:** Raft consensus, multi-region Kafka
- ❌ **Higher Latency:** +100ms for cross-region coordination
- ❌ **3× Cost:** Infrastructure in 3 regions

**When to Use:** After 100M+ users globally.

### Decision Made

**✅ Start single-region, migrate to multi-region when needed**

**Rationale:**
- Simplicity for MVP
- 80% of initial users likely in one region
- Can migrate later (Kafka replication, regional Sequence Managers)

---

## Summary: All Decisions

| Component | Choice | Alternative | Key Reason |
|-----------|--------|-------------|------------|
| **Transport** | WebSockets | Long Polling, SSE | Real-time, bidirectional, efficient |
| **Message Log** | Kafka | PostgreSQL, RabbitMQ | Strict ordering, high throughput |
| **History Storage** | Cassandra | PostgreSQL, MongoDB | Write-heavy, time-series, 45 PB |
| **Presence** | Redis | Memcached, DB | Sub-ms, TTL, batch queries |
| **Sequence IDs** | Snowflake (distributed) | ZooKeeper, UUID | No SPOF, time-ordered |
| **Group Fanout** | Hybrid (push/pull) | Push-only, Pull-only | Balance writes/reads |
| **Regions** | Single → Multi | Multi from start | Simplicity for MVP |

**All decisions prioritize:**
1. **Real-time:** Sub-500ms latency
2. **Ordering:** Strict within conversations
3. **Durability:** Zero message loss
4. **Scale:** 100M connections, 45 PB storage
5. **Simplicity:** Start simple, optimize later
