# Stock Brokerage Platform - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made for the stock brokerage platform,
explaining why certain technologies and patterns were chosen over alternatives.

---

## Table of Contents

1. [WebSockets (Push) vs HTTP Polling (Pull)](#1-websockets-push-vs-http-polling-pull)
2. [Redis (In-Memory) vs PostgreSQL for Quote Storage](#2-redis-in-memory-vs-postgresql-for-quote-storage)
3. [Event Sourcing vs Direct Database Updates for Ledger](#3-event-sourcing-vs-direct-database-updates-for-ledger)
4. [Multi-Level Pub/Sub vs Direct Fanout for Quote Distribution](#4-multi-level-pubsub-vs-direct-fanout-for-quote-distribution)
5. [FIX Protocol vs REST API for Exchange Integration](#5-fix-protocol-vs-rest-api-for-exchange-integration)
6. [PostgreSQL (ACID) vs Cassandra (NoSQL) for Ledger](#6-postgresql-acid-vs-cassandra-nosql-for-ledger)
7. [Elasticsearch vs PostgreSQL Full-Text Search](#7-elasticsearch-vs-postgresql-full-text-search)
8. [Async Order Processing vs Synchronous Blocking](#8-async-order-processing-vs-synchronous-blocking)

---

## 1. WebSockets (Push) vs HTTP Polling (Pull)

### The Problem

Users need real-time stock quotes with minimal latency (<200ms). How do we deliver 100,000 quote updates per second to
10 million concurrent users watching 200 million total symbol subscriptions?

### Options Considered

| Aspect                  | WebSockets (Push)                             | HTTP Polling (Pull)                                    | Server-Sent Events (SSE) |
|-------------------------|-----------------------------------------------|--------------------------------------------------------|--------------------------|
| **Latency**             | <100ms (server pushes immediately)            | 1-5 seconds (polling interval)                         | <100ms (server pushes)   |
| **Bandwidth**           | Minimal (only changed quotes)                 | High (polls even if no change)                         | Moderate (HTTP overhead) |
| **Server Load**         | 100k updates/sec                              | 200M requests/sec (10M users × 20 stocks × 1 poll/sec) | 100k updates/sec         |
| **Scalability**         | Horizontal (1,000 servers)                    | Poor (crushes servers)                                 | Horizontal               |
| **Bidirectional**       | ✅ Yes (client can send subscribe/unsubscribe) | ✅ Yes (separate HTTP requests)                         | ❌ No (one-way only)      |
| **Connection Overhead** | Low (persistent connection)                   | High (new connection per poll)                         | Low (persistent)         |
| **Browser Support**     | ✅ All modern browsers                         | ✅ Universal                                            | ⚠️ Limited (no IE)       |
| **Implementation**      | Complex (stateful servers)                    | Simple (stateless)                                     | Moderate                 |

### Decision Made

**✅ WebSockets (Push Model)**

### Rationale

1. **Latency Requirement:** <200ms end-to-end mandates push model. Polling introduces minimum 1-second lag.

2. **Bandwidth Efficiency:** With 100k quote updates/sec and 10M users:
    - **WebSockets:** Only push changed quotes to subscribed users = ~100k messages/sec (after selective fanout)
    - **HTTP Polling:** 10M users × 20 stocks × 1 poll/sec = **200M requests/sec** (2000x more load)

3. **Cost Savings:** Reduced bandwidth = $50k/month saved (vs polling infrastructure)

4. **Bidirectional Communication:** Clients can dynamically subscribe/unsubscribe without separate HTTP requests

5. **Real-World Precedent:** All major brokers (Robinhood, ETRADE, Zerodha) use WebSockets for live quotes

### Implementation Details

**WebSocket Server Architecture:**

```
1,000 servers × 10,000 connections each = 10M concurrent users
Memory per server: ~600 MB (10k connections + subscription map)
Technology: Node.js (event-driven, single-threaded, efficient)
Protocol: WSS (WebSocket Secure over TLS)
```

**Subscription Management:**

```javascript
// In-memory subscription map per server
subscriptionMap = {
  "AAPL": [user123, user456, ...],  // 2,000 users on this server
  "GOOGL": [user789, user234, ...]
}

// When quote arrives from Kafka
function onQuoteUpdate(symbol, quoteData) {
  const subscribers = subscriptionMap[symbol] || [];
  subscribers.forEach(userId => {
    websocketConnections[userId].send(JSON.stringify({
      symbol: symbol,
      ltp: quoteData.ltp,
      change: quoteData.change
    }));
  });
}
```

### Trade-offs Accepted

| What We Sacrifice                                                    | What We Gain                                              |
|----------------------------------------------------------------------|-----------------------------------------------------------|
| ❌ **Stateful servers** (harder to scale, maintain session state)     | ✅ **2000x lower server load** (100k vs 200M requests/sec) |
| ❌ **Complex failover** (clients must reconnect, resubscribe)         | ✅ **Sub-100ms latency** (vs 1-5s polling lag)             |
| ❌ **Memory overhead** (10k connections × 50KB = 500MB per server)    | ✅ **55% bandwidth reduction** (with compression)          |
| ❌ **Load balancer sticky sessions** (user pinned to specific server) | ✅ **Bidirectional** (subscribe/unsubscribe without HTTP)  |

### When to Reconsider

- **If user base < 10,000:** HTTP polling might be simpler (lower complexity)
- **If latency requirement relaxed to 5+ seconds:** Polling becomes viable
- **If corporate firewall blocks WebSockets:** Fall back to SSE or long-polling

---

## 2. Redis (In-Memory) vs PostgreSQL for Quote Storage

### The Problem

Store latest quote (LTP, Bid, Ask) for 100,000+ stock symbols. Must support:

- 100,000 writes/sec (quote updates from exchange)
- 200,000,000 reads/sec (200M WebSocket subscriptions polling cache)
- Sub-millisecond read latency (<1ms)

### Options Considered

| Aspect               | Redis (In-Memory)              | PostgreSQL (Disk)             | DynamoDB (Cloud)             |
|----------------------|--------------------------------|-------------------------------|------------------------------|
| **Read Latency**     | <1ms (RAM)                     | 10-50ms (SSD)                 | 5-10ms (network + disk)      |
| **Write Throughput** | 100k+ TPS                      | 10k TPS (bottleneck)          | 50k TPS (costs scale)        |
| **Data Structure**   | Key-Value (optimal for quotes) | Relational (overkill)         | Key-Value                    |
| **Persistence**      | Optional (RDB/AOF snapshots)   | ✅ Durable                     | ✅ Durable                    |
| **Cost (10-node)**   | $18k/month                     | $27k/month (over-provisioned) | $30k/month (pay-per-request) |
| **Scalability**      | Horizontal (sharding)          | Vertical (limited)            | Auto-scaling                 |
| **Complexity**       | Moderate (cluster management)  | Low (mature tooling)          | Low (managed)                |

### Decision Made

**✅ Redis (In-Memory Key-Value Store)**

### Rationale

1. **Latency is Critical:** <1ms reads are **non-negotiable** for 200M concurrent streams. PostgreSQL's 10-50ms disk
   reads would violate SLA.

2. **Write Throughput:** 100k quote updates/sec exceed PostgreSQL's 10k TPS limit. Redis handles 100k+ TPS easily.

3. **Data Model Simplicity:** Quotes are simple key-value pairs (`quote:AAPL` → `{"ltp": 150.25, "bid": 150.20}`).
   Relational schema is unnecessary overhead.

4. **Cost-Effective:** $18k/month for 10-node Redis cluster vs $27k for over-provisioned PostgreSQL.

5. **Real-World Success:** NYSE, NASDAQ, and all major HFT firms use in-memory caches for market data.

### Implementation Details

**Redis Cluster Configuration:**

```
10 nodes (r6g.2xlarge)
- 3 master nodes (sharded by symbol hash)
- 7 replica nodes (2-3 replicas per master)
- Total RAM: 10 × 64 GB = 640 GB
- Quote storage: 100k symbols × 500 bytes = 50 MB (plenty of headroom)
```

**Data Schema:**

```
Key: quote:{symbol}
Value: {"ltp": 150.25, "bid": 150.20, "ask": 150.30, "volume": 50000000, "timestamp": 1609459200000}
TTL: None (always keep latest)

Key: ohlc:{symbol}:{date}
Value: {"open": 148.50, "high": 151.00, "low": 147.00, "close": 150.25}
TTL: 7 days (rolling window)
```

**Persistence Strategy:**

```
RDB snapshots: Every 15 minutes
AOF log: fsync every second (balance durability vs performance)
Trade-off: Max 1 second of data loss on crash (acceptable for quotes, which are re-streamed)
```

### Trade-offs Accepted

| What We Sacrifice                                              | What We Gain                                       |
|----------------------------------------------------------------|----------------------------------------------------|
| ❌ **Data loss risk** (max 1 second on crash, quotes re-stream) | ✅ **<1ms read latency** (200M reads/sec supported) |
| ❌ **No complex queries** (key-value only, no SQL joins)        | ✅ **100k+ writes/sec** (10x PostgreSQL throughput) |
| ❌ **RAM cost** ($18k/month for 640GB)                          | ✅ **Simple data model** (no schema migrations)     |
| ❌ **Cluster management** (replication, sharding, failover)     | ✅ **Cost-effective** ($18k vs $27k PostgreSQL)     |

### When to Reconsider

- **If quotes must be durable:** Use PostgreSQL or DynamoDB (stricter persistence guarantees)
- **If budget < $10k/month:** Use single-node Redis (sacrifices HA)
- **If complex analytics needed:** Supplement with ClickHouse for historical queries

---

## 3. Event Sourcing vs Direct Database Updates for Ledger

### The Problem

Account ledger (cash balance, holdings) experiences high contention:

- 1,000 orders/sec during peak hours
- Many users place multiple orders simultaneously
- Traditional `UPDATE accounts SET cash_balance = cash_balance - amount` requires exclusive locks
- Locks serialize all updates, causing latency spikes (5-10 seconds)

### Options Considered

| Aspect            | Event Sourcing (Kafka Log)            | Direct Updates (PostgreSQL)       | Optimistic Locking                  |
|-------------------|---------------------------------------|-----------------------------------|-------------------------------------|
| **Write Latency** | <10ms (append to log)                 | 50-200ms (lock + update)          | 20-100ms (retry on conflict)        |
| **Throughput**    | 1M+ writes/sec (Kafka)                | 6k TPS (row-level locks)          | 10k TPS (retries reduce throughput) |
| **Consistency**   | Eventual (~1 second lag)              | Strong (ACID)                     | Strong (ACID)                       |
| **Audit Trail**   | ✅ Complete (immutable log)            | ❌ No history (only current state) | ❌ No history                        |
| **Complexity**    | High (event replay, snapshots)        | Low (standard SQL)                | Moderate (retry logic)              |
| **Replayability** | ✅ Yes (reconstruct state at any time) | ❌ No (lost history)               | ❌ No                                |
| **Compliance**    | ✅ 7-year retention (regulatory)       | ⚠️ Separate audit log needed      | ⚠️ Separate audit log needed        |

### Decision Made

**✅ Hybrid Approach: Event Sourcing (Kafka) + Materialized View (PostgreSQL)**

### Rationale

1. **Eliminate Lock Contention:** Appending to Kafka is lock-free (partition-level ordering). No more 5-10 second
   latency spikes.

2. **Audit Trail Built-In:** Every deposit, withdrawal, order, and trade is an immutable event in Kafka. Satisfies
   7-year regulatory retention automatically.

3. **Replayability:** Can reconstruct user's balance at any point in time by replaying events (critical for dispute
   resolution).

4. **High Throughput:** Kafka handles 1M+ writes/sec (100x improvement over PostgreSQL locks).

5. **Fast Reads:** Materialized view in PostgreSQL provides <50ms reads for displaying user's current balance (best of
   both worlds).

### Implementation Details

**Event Log (Kafka Compacted Topic):**

```
Topic: ledger.events
Partitions: 100 (partitioned by user_id)
Retention: 7 years (compliance)
Compaction: Keep latest event per key (balance snapshots)

Event Schema:
{
  "event_id": "uuid",
  "user_id": 12345,
  "type": "ORDER_PLACED" | "ORDER_FILLED" | "DEPOSIT" | "WITHDRAWAL",
  "amount": -1500,  // Negative = deduction
  "order_id": 999888,
  "timestamp": 1609459200000,
  "metadata": {...}
}
```

**Materialized View (PostgreSQL):**

```sql
-- Read-optimized snapshot (updated every 1 second)
CREATE TABLE accounts (
  user_id BIGINT PRIMARY KEY,
  cash_balance DECIMAL(15,2),
  margin_used DECIMAL(15,2),
  updated_at TIMESTAMP
);

-- Ledger Service (Kafka Consumer) updates this table
UPDATE accounts SET cash_balance = derived_from_events WHERE user_id = 12345;
```

**Balance Calculation:**

```
Current balance = SUM(all DEPOSIT/WITHDRAWAL events) - SUM(margin_used from pending orders)

Example:
Event 1: DEPOSIT +10000
Event 2: ORDER_PLACED -1500 (margin blocked)
Event 3: ORDER_FILLED (margin released, actual deduction)
Event 4: DEPOSIT +5000

Balance = 10000 - 1500 (if order pending) OR 10000 - actual_fill_price + 5000 (if filled)
```

### Trade-offs Accepted

| What We Sacrifice                                           | What We Gain                                       |
|-------------------------------------------------------------|----------------------------------------------------|
| ❌ **Eventual consistency** (snapshot lags by ~1 second)     | ✅ **100x throughput** (1M vs 10k writes/sec)       |
| ❌ **Complexity** (event replay logic, snapshot maintenance) | ✅ **Zero lock contention** (append-only log)       |
| ❌ **Storage cost** (7-year Kafka retention = $50k/year)     | ✅ **Complete audit trail** (regulatory compliance) |
| ❌ **Operational overhead** (Kafka cluster management)       | ✅ **Replayable** (reconstruct balance at any time) |

### When to Reconsider

- **If strong real-time consistency required:** Use distributed transactions (2PC) across PostgreSQL shards (trades off
  throughput for consistency)
- **If budget < $20k/month:** Use PostgreSQL with row-level locks (accept lower throughput)
- **If team lacks Kafka expertise:** Stick with PostgreSQL + separate audit log table

---

## 4. Multi-Level Pub/Sub vs Direct Fanout for Quote Distribution

### The Problem

Broadcast 100,000 quote updates/sec to 200 million WebSocket streams (10M users × 20 watched stocks each). Naive direct
fanout is impossible.

### Options Considered

| Aspect              | Multi-Level Pub/Sub (Kafka + WebSocket)        | Direct Fanout (Ingestor → WebSocket)           | Managed Pub/Sub (AWS IoT Core) |
|---------------------|------------------------------------------------|------------------------------------------------|--------------------------------|
| **Fanout Capacity** | 200M streams (distributed)                     | 10k streams per server (single bottleneck)     | 1B+ streams (cloud-scale)      |
| **Latency**         | <100ms (2 hops: Kafka → WS)                    | <50ms (1 hop: direct)                          | <200ms (3 hops: cloud)         |
| **Cost**            | $14k/month (Kafka) + $72k (WS servers) = $86k  | $5k/month (single ingestor) + $72k (WS) = $77k | $200k/month (AWS IoT pricing)  |
| **Scalability**     | Horizontal (add Kafka partitions + WS servers) | Vertical (limited by single ingestor)          | Auto-scaling (managed)         |
| **Fault Tolerance** | ✅ High (Kafka replication, WS server failover) | ❌ Single point of failure (ingestor)           | ✅ High (cloud SLA 99.99%)      |
| **Operational**     | Moderate (manage Kafka + WS clusters)          | Simple (one ingestor node)                     | Low (fully managed)            |

### Decision Made

**✅ Multi-Level Pub/Sub (Kafka as Distribution Layer + WebSocket as Edge)**

### Rationale

1. **Scalability:** Direct fanout from single ingestor cannot support 200M streams. Kafka distributes load across 100
   partitions, consumed by 1,000 WebSocket servers.

2. **Fault Tolerance:** Kafka's replication (factor 3) ensures no data loss if ingestor crashes. WebSocket servers can
   fail independently without affecting others.

3. **Selective Fanout:** WebSocket servers maintain subscription maps, pushing only subscribed symbols to users. This
   reduces 200M potential streams to ~100k actual pushes/sec (per user watches 1-2 symbols actively).

4. **Cost-Effective:** $86k/month (self-hosted) vs $200k/month (AWS IoT Core). At 10M users, economies of scale favor
   self-hosted.

5. **Proven Architecture:** Used by Netflix, Uber, LinkedIn for large-scale real-time systems.

### Implementation Details

**Architecture Layers:**

**Level 1: Centralized Ingestion (1 node)**

```
Market Data Ingestor
  - Receives 100k updates/sec from Exchange (FIX protocol)
  - Publishes to Kafka topic: market.quotes
  - Partition key: symbol hash (ensures ordering per symbol)
```

**Level 2: Kafka Distribution (100 partitions)**

```
Kafka Cluster (20 brokers, 100 partitions)
  - Each partition: ~1,000 symbols
  - Replication factor: 3 (durability)
  - Retention: 7 days (replay capability)
  - Consumer group: websocket-consumers (1,000 consumers)
```

**Level 3: Selective Fanout (1,000 WebSocket servers)**

```
Each WebSocket Server:
  - Consumes from 1 Kafka partition (~1,000 symbols)
  - Maintains in-memory subscription map: {symbol: [user1, user2, ...]}
  - Receives ~100 updates/sec from Kafka
  - Pushes to ~2,000 subscribed users per update
  - Per-server fanout: 100 updates × 2,000 users = 200k msg/sec (manageable)
```

### Trade-offs Accepted

| What We Sacrifice                                 | What We Gain                                        |
|---------------------------------------------------|-----------------------------------------------------|
| ❌ **Added latency** (Kafka adds ~20ms vs direct)  | ✅ **200M stream capacity** (vs 10k direct fanout)   |
| ❌ **Kafka operational cost** ($14k/month cluster) | ✅ **Fault tolerance** (3x replication, no SPOF)     |
| ❌ **Architecture complexity** (3 layers vs 2)     | ✅ **Horizontal scaling** (add partitions + servers) |
| ❌ **Storage overhead** (7-day Kafka retention)    | ✅ **Replay capability** (recover from failures)     |

### When to Reconsider

- **If user base < 100k:** Direct fanout is simpler (no Kafka needed)
- **If budget > $200k/month:** Use managed service (AWS IoT Core, Pusher) for zero ops overhead
- **If latency must be <50ms:** Direct fanout (accept lower scale, single point of failure)

---

## 5. FIX Protocol vs REST API for Exchange Integration

### The Problem

Route user orders to stock exchange (NSE/BSE) for execution. Exchange provides two options:

1. FIX Protocol (industry standard, binary, session-based)
2. REST API (modern, JSON, stateless)

### Options Considered

| Aspect                 | FIX Protocol                                      | REST API                                | gRPC                        |
|------------------------|---------------------------------------------------|-----------------------------------------|-----------------------------|
| **Latency**            | <100ms (persistent TCP, binary)                   | 200-500ms (HTTP overhead, JSON parsing) | <150ms (binary, HTTP/2)     |
| **Throughput**         | 10k orders/sec per session                        | 1k orders/sec (rate limited)            | 5k orders/sec               |
| **Reliability**        | ✅ Sequence numbers (no duplicates, gap detection) | ⚠️ Manual retry logic                   | ⚠️ Manual retry logic       |
| **Industry Standard**  | ✅ Yes (all exchanges support)                     | ⚠️ Limited (modern exchanges only)      | ❌ No (not supported)        |
| **Complexity**         | High (sequence management, heartbeats)            | Low (standard HTTP)                     | Moderate (protobuf schemas) |
| **Real-Time Feed**     | ✅ Market data (35=W messages)                     | ❌ No (separate WebSocket)               | ✅ Server streaming          |
| **Session Management** | ✅ Built-in (logon, heartbeat, logout)             | ❌ Manual token refresh                  | ❌ Manual auth               |
| **Compatibility**      | ✅ Works with ALL exchanges globally               | ⚠️ Limited to modern platforms          | ❌ Not widely supported      |

### Decision Made

**✅ FIX Protocol (Financial Information eXchange)**

### Rationale

1. **Industry Standard:** **100% of stock exchanges** (NYSE, NASDAQ, NSE, BSE, LSE) support FIX. REST APIs are rare and
   limited to modern platforms only.

2. **Latency:** Sub-100ms order routing is critical. FIX's persistent TCP connection + binary encoding beats REST's HTTP
   overhead + JSON parsing (200-500ms).

3. **Reliability:** FIX's sequence numbers ensure:
    - **No duplicate orders** (each message has unique `MsgSeqNum`)
    - **Gap detection** (if msg 50 arrives but expected 48, request resend)
    - **Audit trail** (every message is numbered and logged)

4. **Market Data Integration:** FIX provides **both** order routing (35=D) **and** market data (35=W) in single
   protocol. REST requires separate WebSocket for quotes.

5. **Production-Grade:** All tier-1 brokers (Interactive Brokers, Charles Schwab, ETRADE) use FIX exclusively.

### Implementation Details

**FIX Session Lifecycle:**

```
1. TCP Connect: socket.connect(exchange_host, 5001)
2. Logon (35=A): Send credentials, establish session
3. Heartbeat (35=0): Every 30 seconds (keep session alive)
4. Order Flow:
   - NewOrderSingle (35=D): Place order
   - ExecutionReport (35=8): Order status update (New, Filled, Cancelled)
5. Market Data:
   - MarketDataRequest (35=V): Subscribe to symbols
   - MarketDataSnapshot (35=W): Receive quote updates
6. Logout (35=5): Gracefully close session
```

**Sequence Number Management:**

```python
class FIXSession:
    def __init__(self):
        self.last_sent_seqnum = 0
        self.last_received_seqnum = 0
    
    def send_message(self, msg):
        self.last_sent_seqnum += 1
        msg.set(34, self.last_sent_seqnum)  # MsgSeqNum
        self.persist_seqnum()  # Save to DB (recovery)
        self.socket.send(msg.encode())
    
    def on_receive(self, msg):
        received_seqnum = msg.get(34)
        expected_seqnum = self.last_received_seqnum + 1
        
        if received_seqnum > expected_seqnum:
            # Gap detected! Request resend
            self.send_resend_request(expected_seqnum, received_seqnum - 1)
        
        self.last_received_seqnum = received_seqnum
        self.persist_seqnum()
```

**Error Handling:**

```
1. Heartbeat Timeout (60 seconds):
   - No heartbeat received → Assume disconnection
   - Close socket, wait 5s, reconnect, logon with last seq nums

2. Order Rejection (35=8, OrdStatus=8):
   - Exchange rejects order (insufficient funds, invalid symbol)
   - Parse rejection reason (103=OrdRejReason)
   - Notify user, release margin

3. Sequence Number Reset (35=4):
   - Exchange lost session state (rare, during maintenance)
   - Reset sequence numbers to 1
   - Re-establish session
```

### Trade-offs Accepted

| What We Sacrifice                                             | What We Gain                                             |
|---------------------------------------------------------------|----------------------------------------------------------|
| ❌ **High complexity** (sequence management, heartbeats, gaps) | ✅ **<100ms latency** (vs 200-500ms REST)                 |
| ❌ **Binary protocol** (harder to debug than JSON)             | ✅ **Universal compatibility** (works with ALL exchanges) |
| ❌ **Stateful sessions** (requires persistent TCP connection)  | ✅ **Reliability** (no duplicates, gap detection)         |
| ❌ **Learning curve** (FIX spec is 1,000+ pages)               | ✅ **Market data + orders** (single protocol)             |

### When to Reconsider

- **If exchange only provides REST API:** Modern platforms (Alpaca, Robinhood internal) may only support REST
- **If low-frequency trading:** <10 orders/day doesn't justify FIX complexity
- **If prototyping/MVP:** REST is faster to implement (production should migrate to FIX)

---

## 6. PostgreSQL (ACID) vs Cassandra (NoSQL) for Ledger

### The Problem

Store user account ledger (cash balance, holdings, orders). Requirements:

- **Strong consistency:** User's balance must always be accurate (no double spending)
- **ACID transactions:** Order placement + balance deduction must be atomic
- **High availability:** Users must access balances 24/7

### Options Considered

| Aspect                    | PostgreSQL (ACID)          | Cassandra (NoSQL)             | DynamoDB (Cloud NoSQL)             |
|---------------------------|----------------------------|-------------------------------|------------------------------------|
| **Consistency**           | Strong (ACID)              | Eventual (tunable)            | Eventual (or strong with cost)     |
| **Transactions**          | ✅ Full ACID (BEGIN/COMMIT) | ❌ No multi-row transactions   | ⚠️ Limited (single-partition only) |
| **Write Throughput**      | 10k TPS (locks limit)      | 100k+ TPS (no locks)          | 50k TPS (pay-per-request)          |
| **Availability**          | 99.95% (replication)       | 99.99% (multi-datacenter)     | 99.99% (managed)                   |
| **Double Spending Risk**  | ✅ None (locks prevent)     | ❌ High (eventual consistency) | ⚠️ Low (conditional writes)        |
| **Regulatory Compliance** | ✅ ACID required by law     | ❌ Unacceptable for finance    | ⚠️ Limited (no SQL audits)         |
| **Cost**                  | $27k/month (5 shards)      | $40k/month (multi-DC)         | $35k/month (provisioned)           |

### Decision Made

**✅ PostgreSQL (ACID-Compliant Relational Database)**

### Rationale

1. **Financial Integrity is Non-Negotiable:** Eventual consistency is **unacceptable** for money. Users must never see
   incorrect balances or experience double spending.

2. **ACID Transactions Required:** Order placement logic:
   ```sql
   BEGIN TRANSACTION;
     -- Lock user's account
     SELECT cash_balance FROM accounts WHERE user_id = 12345 FOR UPDATE;
     
     -- Check balance
     IF cash_balance < order_amount THEN ROLLBACK;
     
     -- Deduct balance
     UPDATE accounts SET margin_used = margin_used + order_amount WHERE user_id = 12345;
     
     -- Insert order
     INSERT INTO orders (...) VALUES (...);
   COMMIT;
   ```
   Cassandra cannot provide this atomicity across multiple rows.

3. **Regulatory Requirement:** SEBI/SEC audits require **ACID-compliant ledger**. Cassandra's eventual consistency
   violates compliance standards.

4. **Real-World Precedent:** **Every major broker** (Robinhood, ETRADE, Schwab, Zerodha) uses ACID databases (
   PostgreSQL, MySQL, Oracle) for ledger. **None use Cassandra for financial data.**

5. **Risk of Eventual Consistency:**
   ```
   Scenario: User has $1,000 balance
   
   With Eventual Consistency (Cassandra):
   - 10:00:00 - User places order for $1,000 (replica 1 updated)
   - 10:00:01 - User places ANOTHER order for $1,000 (replica 2 not synced yet)
   - Result: $2,000 spent with only $1,000 balance ❌ DOUBLE SPENDING
   
   With ACID (PostgreSQL):
   - 10:00:00 - User places order for $1,000 (row locked)
   - 10:00:01 - User places ANOTHER order for $1,000 (waits for lock)
   - Second order fails: "Insufficient balance" ✅ PREVENTED
   ```

### Implementation Details

**Sharding Strategy:**

```sql
-- Shard by user_id (co-locate user's data)
Shard 1: user_id % 5 = 0  (2M users)
Shard 2: user_id % 5 = 1  (2M users)
Shard 3: user_id % 5 = 2  (2M users)
Shard 4: user_id % 5 = 3  (2M users)
Shard 5: user_id % 5 = 4  (2M users)

Each shard: Primary + 2 replicas (read scaling)
```

**High Availability:**

```
Primary-Replica Replication:
- Synchronous replication to 1 replica (strong consistency)
- Asynchronous replication to 2nd replica (read scaling)
- Automatic failover (promote replica to primary on failure)
- Downtime: <1 minute (Patroni/Stolon orchestration)
```

**Performance Optimization:**

```
1. Connection Pooling: PgBouncer (1,000 connections per shard)
2. Read Replicas: 2 per shard (split read traffic 80/20)
3. Partitioning: Partition orders table by date (1M rows per partition)
4. Indexing: B-tree on user_id, created_at (fast lookups)
```

### Trade-offs Accepted

| What We Sacrifice                                          | What We Gain                                  |
|------------------------------------------------------------|-----------------------------------------------|
| ❌ **Lower write throughput** (10k vs 100k TPS)             | ✅ **Zero double spending** (ACID prevents)    |
| ❌ **Vertical scaling limits** (shard rebalancing hard)     | ✅ **Regulatory compliance** (ACID required)   |
| ❌ **Single-region constraint** (multi-DC Postgres complex) | ✅ **SQL queries** (analytics, reporting easy) |
| ❌ **Downtime during failover** (~1 minute)                 | ✅ **Data integrity** (transactions atomic)    |

### When to Reconsider

- **For non-critical data only:** Use Cassandra for activity logs, click streams, analytics (eventual consistency
  acceptable)
- **Never for ledger:** Financial data **must** be ACID-compliant
- **If throughput > 100k TPS:** Supplement with Event Sourcing (Kafka) to reduce Postgres write load

---

## 7. Elasticsearch vs PostgreSQL Full-Text Search

### The Problem

Enable users to search for stock tickers with autocomplete (prefix search). Requirements:

- <50ms query latency
- Support fuzzy search (typos: "GOGLE" → GOOGL)
- Rank by relevance (Apple Inc. > AppLovin for "AP")

### Options Considered

| Aspect            | Elasticsearch                     | PostgreSQL (pg_trgm)          | Algolia (Managed)         |
|-------------------|-----------------------------------|-------------------------------|---------------------------|
| **Query Latency** | <30ms (optimized inverted index)  | 100-500ms (sequential scan)   | <20ms (edge locations)    |
| **Fuzzy Search**  | ✅ Built-in (Levenshtein distance) | ⚠️ Manual (pg_trgm extension) | ✅ Built-in                |
| **Ranking**       | ✅ BM25 relevance scoring          | ❌ Basic (LIKE, no scoring)    | ✅ Machine learning        |
| **Scalability**   | Horizontal (sharding)             | Vertical (limited)            | Auto-scaling (managed)    |
| **Cost**          | $9k/month (5-node cluster)        | $0 (included in Postgres)     | $15k/month (SaaS pricing) |
| **Index Size**    | 100k symbols × 1KB = 100MB        | 100k symbols × 2KB = 200MB    | 100MB (managed)           |
| **Complexity**    | Moderate (cluster management)     | Low (single SQL query)        | Low (fully managed)       |

### Decision Made

**✅ Elasticsearch (Purpose-Built Search Engine)**

### Rationale

1. **Latency Requirement:** <50ms is critical for instant autocomplete. Elasticsearch's inverted index achieves <30ms.
   PostgreSQL's `LIKE '%AP%'` requires full table scan (100-500ms).

2. **Fuzzy Search:** Built-in Levenshtein distance handles typos ("GOGLE" → GOOGL with 1 character difference).
   PostgreSQL requires manual implementation.

3. **Relevance Ranking:** Elasticsearch's BM25 algorithm ranks by:
    - Term frequency (how often "AP" appears in document)
    - Inverse document frequency (rare terms ranked higher)
    - Field boosting (symbol name weighted 2x vs description)

   PostgreSQL LIKE has no relevance concept (alphabetical order only).

4. **Real-World Success:** Google Search, Amazon Product Search, GitHub Code Search all use Elasticsearch/Solr (
   Lucene-based).

### Implementation Details

**Index Schema:**

```json
{
  "mappings": {
    "properties": {
      "symbol": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {
            "type": "keyword"
          },
          // Exact match
          "prefix": {
            "type": "text",
            "analyzer": "edge_ngram"
          }
          // Autocomplete
        }
      },
      "name": {
        "type": "text",
        "analyzer": "standard",
        "boost": 2.0
        // Name is 2x more important than description
      },
      "exchange": {
        "type": "keyword"
      },
      "market_cap": {
        "type": "long"
      },
      "sector": {
        "type": "keyword"
      }
    }
  }
}
```

**Autocomplete Query:**

```json
{
  "query": {
    "bool": {
      "should": [
        {
          "prefix": {
            "symbol": "AP"
          }
        },
        // Matches "AAPL", "APPL"
        {
          "match": {
            "name": {
              "query": "AP",
              "fuzziness": "AUTO"
            }
          }
        }
        // Matches "Apple", "AppLovin"
      ]
    }
  },
  "sort": [
    {
      "market_cap": "desc"
    }
    // Larger companies ranked higher
  ],
  "size": 10
}
```

**Performance Optimization:**

```
1. Edge N-grams: Pre-compute all prefixes (A, AP, APP, APPL) for instant autocomplete
2. Caching: Cache popular queries (e.g., "AAPL") in Redis (1-minute TTL)
3. Sharding: Split 100k symbols across 3 shards (33k each)
4. Replicas: 2 replicas per shard (read scaling, high availability)
```

### Trade-offs Accepted

| What We Sacrifice                                                  | What We Gain                                     |
|--------------------------------------------------------------------|--------------------------------------------------|
| ❌ **Eventual consistency** (index lags behind Postgres by seconds) | ✅ **<30ms queries** (vs 100-500ms Postgres LIKE) |
| ❌ **Operational cost** ($9k/month cluster)                         | ✅ **Fuzzy search** (handles typos automatically) |
| ❌ **Cluster management** (sharding, replicas, upgrades)            | ✅ **Relevance ranking** (BM25 scoring)           |
| ❌ **Storage overhead** (inverted index = 2x raw data size)         | ✅ **Autocomplete** (instant as-you-type results) |

### When to Reconsider

- **If budget < $5k/month:** Use PostgreSQL with `pg_trgm` (accept slower queries)
- **If user base < 10k:** PostgreSQL LIKE queries are fast enough
- **If using Algolia budget:** Managed search service ($15k/month, zero ops)

---

## 8. Async Order Processing vs Synchronous Blocking

### The Problem

User places an order. How long should the API wait before responding?

**Option 1:** Wait for exchange fill confirmation (5-10 seconds)  
**Option 2:** Return immediately, notify user when filled (250ms)

### Options Considered

| Aspect                | Async (Event-Driven)                   | Synchronous (Blocking)                |
|-----------------------|----------------------------------------|---------------------------------------|
| **API Response Time** | <250ms (immediate)                     | 5-10 seconds (wait for fill)          |
| **User Experience**   | ✅ Instant feedback ("Order placed")    | ❌ UI frozen for 10 seconds            |
| **Server Resources**  | ✅ Non-blocking (handle other requests) | ❌ Blocked (thread/connection tied up) |
| **Order Status**      | Pushed via WebSocket ("Order filled")  | Returned in HTTP response             |
| **Failure Handling**  | ✅ Graceful (user notified async)       | ❌ Timeout errors (user confused)      |
| **Complexity**        | Moderate (event bus, notifications)    | Low (simple request-response)         |

### Decision Made

**✅ Async Event-Driven Processing**

### Rationale

1. **User Experience:** 10-second API response is **unacceptable**. Users expect instant confirmation. Async model
   provides <250ms feedback.

2. **Server Scalability:** Synchronous blocking ties up threads for 10 seconds each. At 1,000 orders/sec, this requires
   10,000 concurrent threads (impossible). Async model handles 1,000 orders/sec with 100 threads.

3. **Failure Resilience:** If exchange is slow (30 seconds), synchronous API times out (bad UX). Async model continues
   processing, notifies user when done.

4. **Real-World Standard:** All modern brokers (Robinhood, ETRADE, Zerodha) use async order processing.

### Implementation Details

**Async Flow:**

```
1. User clicks "Buy" (t=0ms)
2. API validates order (balance, limits) (t=50ms)
3. API writes to Postgres (ACID transaction) (t=150ms)
4. API publishes event to Kafka (t=160ms)
5. API returns 200 OK to user (t=200ms)
   Response: {"order_id": 999888, "status": "PENDING", "message": "Order placed successfully"}
6. User sees "Order placed!" (instant feedback)

Meanwhile (async):
7. FIX Engine consumes Kafka event (t=500ms)
8. FIX Engine sends order to exchange (t=1s)
9. Exchange fills order (t=5s)
10. FIX Engine publishes ORDER_FILLED event (t=5.1s)
11. WebSocket server pushes notification to user (t=5.2s)
12. User sees "Order filled at $2450!" (5 seconds later)
```

**WebSocket Notification:**

```javascript
// Client-side
ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  if (msg.type === 'order_update') {
    showNotification(`Order ${msg.order_id} ${msg.status} at $${msg.price}`);
    refreshPortfolio();  // Update UI
  }
};
```

### Trade-offs Accepted

| What We Sacrifice                                           | What We Gain                                        |
|-------------------------------------------------------------|-----------------------------------------------------|
| ❌ **Complex architecture** (Kafka, WebSocket notifications) | ✅ **<250ms API response** (vs 5-10s blocking)       |
| ❌ **User must wait for notification** (5s async)            | ✅ **Non-blocking server** (10k orders/sec capacity) |
| ❌ **State management** (track pending orders)               | ✅ **Failure resilience** (no timeouts)              |

### When to Reconsider

- **If low-frequency trading:** <10 orders/day, synchronous is simpler
- **If no WebSocket infrastructure:** Async notifications require real-time channel

---

## Summary: Key Decision Matrix

| Decision                                   | Primary Driver                    | Main Trade-off                   |
|--------------------------------------------|-----------------------------------|----------------------------------|
| **WebSockets over Polling**                | Latency (<200ms)                  | Complexity (stateful servers)    |
| **Redis over PostgreSQL (quotes)**         | Throughput (100k writes/sec)      | Durability (1s data loss risk)   |
| **Event Sourcing over Direct Updates**     | Audit trail (7-year retention)    | Eventual consistency (~1s lag)   |
| **Multi-Level Pub/Sub over Direct Fanout** | Scale (200M streams)              | Latency (+20ms Kafka hop)        |
| **FIX over REST**                          | Industry standard (universal)     | Complexity (sequence management) |
| **PostgreSQL over Cassandra (ledger)**     | ACID compliance (regulatory)      | Throughput (10k vs 100k TPS)     |
| **Elasticsearch over PostgreSQL (search)** | Query latency (<30ms)             | Cost ($9k/month cluster)         |
| **Async over Sync**                        | User experience (<250ms response) | Architecture complexity (events) |
