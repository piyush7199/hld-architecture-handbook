# Stock Brokerage Platform - High-Level Design

## Table of Contents

1. [System Architecture Overview](#system-architecture-overview)
2. [Multi-Level Pub/Sub Architecture](#multi-level-pubsub-architecture)
3. [Real-Time Market Data Flow](#real-time-market-data-flow)
4. [Order Entry and Execution Flow](#order-entry-and-execution-flow)
5. [WebSocket Server Topology](#websocket-server-topology)
6. [Event Sourcing Architecture](#event-sourcing-architecture)
7. [Search and Discovery Flow](#search-and-discovery-flow)
8. [Risk Management System](#risk-management-system)
9. [FIX Protocol Integration](#fix-protocol-integration)
10. [Multi-Region Deployment](#multi-region-deployment)
11. [Audit Logging Architecture](#audit-logging-architecture)
12. [Performance Optimization Pipeline](#performance-optimization-pipeline)
13. [Failure Scenarios and Recovery](#failure-scenarios-and-recovery)
14. [Monitoring and Observability Dashboard](#monitoring-and-observability-dashboard)

---

## System Architecture Overview

**Flow Explanation:**

This diagram shows the complete end-to-end architecture of the stock brokerage platform.

**Key Components:**

1. **Exchange (NSE/BSE):** Source of market data and order execution
2. **Market Data Ingestor:** Receives FIX protocol messages (100k updates/sec)
3. **Redis Quote Cache:** In-memory storage for latest quotes (<1ms reads)
4. **Kafka Message Bus:** Distributes data to 1,000 WebSocket servers
5. **WebSocket Broadcast Layer:** Pushes updates to 10M concurrent users
6. **Order Entry Gateway:** Validates and routes orders via FIX protocol
7. **PostgreSQL Ledger:** ACID-compliant storage for balances and holdings
8. **Elasticsearch:** Fast ticker search (<50ms)

**Performance:**

- Ingestion latency: <10ms
- Quote distribution: <100ms end-to-end
- Order execution: <1 second
- Search queries: <50ms

**Benefits:**

- Scalable: Handles 10M users, 200M WebSocket streams
- Low latency: <200ms quote updates
- Strong consistency: ACID transactions for financial data
- High availability: Multi-region, fault-tolerant

```mermaid
graph TB
    subgraph "External - Stock Exchange"
        Exchange[NSE/BSE Exchange<br/>Market Data + Order Execution]
    end

    subgraph "Ingestion Layer"
        FIXEngine[FIX Engine<br/>Protocol Handler<br/>Sequence Mgmt]
        Ingestor[Market Data Ingestor<br/>Parses FIX Messages<br/>100k updates/sec]
    end

    Exchange -->|FIX Protocol<br/>TCP 5001| FIXEngine
    Exchange -->|Market Data<br/>FIX 35 = W| Ingestor

    subgraph "Caching Layer"
        Redis[(Redis Cluster<br/>10 nodes<br/>Quote Cache<br/>Sub-ms reads)]
    end

    Ingestor -->|UPDATE quote:SYMBOL| Redis

    subgraph "Message Bus"
        Kafka[Kafka Cluster<br/>20 brokers<br/>100 partitions<br/>Topic: market.quotes]
    end

    Ingestor -->|Publish| Kafka

    subgraph "Broadcast Layer - 1000 WebSocket Servers"
        WS1[WS Server 1<br/>10k connections<br/>Subscription Map]
        WS2[WS Server 2<br/>10k connections]
        WS3[WS Server ...<br/>10k connections]
        WS1000[WS Server 1000<br/>10k connections]
    end

    Kafka -->|Consumer Group| WS1
    Kafka -->|Consumer Group| WS2
    Kafka -->|Consumer Group| WS3
    Kafka -->|Consumer Group| WS1000

    subgraph "Users - 10M Concurrent"
        User1[User 1<br/>Watches: AAPL, GOOGL]
        User2[User 2<br/>Watches: TSLA, AMZN]
        UserM[User 10M]
    end

    WS1 -.->|Push updates<br/>WebSocket| User1
    WS2 -.->|Push updates| User2
    WS1000 -.->|Push updates| UserM

    subgraph "User Actions"
        Client[Web/Mobile Client]
    end

    Client -->|Search stocks| SearchAPI[Search API]
    Client -->|Place order| OrderAPI[Order Entry API<br/>gRPC]
    Client -->|View portfolio| PortfolioAPI[Portfolio API<br/>REST]

    subgraph "Search Layer"
        ES[(Elasticsearch<br/>5 nodes<br/>Stock Index<br/>100k+ symbols)]
    end

    SearchAPI -->|Prefix query| ES
    SearchAPI -->|Fetch quotes| Redis

    subgraph "Order Processing"
        OrderGateway[Order Entry Gateway<br/>Validation<br/>Risk Checks]
    end

    OrderAPI -->|Validate| OrderGateway
    OrderGateway -->|Route order<br/>FIX 35 = D| FIXEngine
    FIXEngine -->|NewOrderSingle| Exchange

    subgraph "Ledger - ACID"
        PG[(PostgreSQL<br/>5 shards<br/>Sharded by user_id<br/>Accounts, Holdings, Orders)]
    end

    OrderGateway -->|BEGIN TRANSACTION<br/>Deduct balance| PG

    subgraph "Event Log"
        KafkaEvents[Kafka<br/>Topic: ledger.events<br/>Immutable Event Log]
    end

    OrderGateway -->|Append event<br/>ORDER_PLACED| KafkaEvents
    Exchange -->|ExecutionReport<br/>FIX 35 = 8| FIXEngine
    FIXEngine -->|ORDER_FILLED event| KafkaEvents

    subgraph "Ledger Service"
        LedgerSvc[Ledger Service<br/>Kafka Consumer<br/>Updates Holdings]
    end

    KafkaEvents -->|Consume| LedgerSvc
    LedgerSvc -->|UPDATE holdings<br/>Release margin| PG
    style Exchange fill: #ff9999
    style Redis fill: #99ccff
    style Kafka fill: #99ff99
    style PG fill: #ffcc99
    style ES fill: #cc99ff
```

---

## Multi-Level Pub/Sub Architecture

**Flow Explanation:**

This diagram illustrates the solution to the broadcast fanout problem (100k updates/sec → 200M WebSocket streams).

**Problem:**

- Naive approach: Each update triggers 2,000 user notifications (if 2,000 users watch each stock)
- Total: 100k × 2,000 = **200M messages/sec** (impossible for single server)

**Solution: Three-Level Hierarchy**

**Level 1: Centralized Ingestion (1 node)**

- Market Data Ingestor receives all 100k updates/sec
- Publishes to Kafka (partitioned by symbol)

**Level 2: Kafka Distribution (100 partitions)**

- Kafka distributes load across 1,000 WebSocket servers
- Each server consumes from 1 partition (~1,000 symbols)

**Level 3: Selective Fanout (1,000 servers)**

- Each WebSocket server maintains in-memory subscription map
- Pushes updates ONLY to subscribed users on that server
- Per-server fanout: 100 updates/sec × 2,000 users = 200k msg/sec (manageable)

**Performance:**

- Ingestion: <10ms
- Kafka propagation: <20ms
- WebSocket push: <50ms
- **End-to-end: <100ms** (within 200ms SLA)

**Benefits:**

- Scalable: Add more WebSocket servers horizontally
- Efficient: No wasted bandwidth (push only subscribed symbols)
- Fault-tolerant: Kafka consumer groups handle server failures

```mermaid
graph TB
    subgraph "Level 1 - Centralized Ingestion"
        Exchange[Stock Exchange<br/>100k updates/sec<br/>All symbols]
        Ingestor[Market Data Ingestor<br/>Single node<br/>Parses FIX messages]
    end

    Exchange -->|FIX Protocol<br/>100k updates/sec| Ingestor

    subgraph "Level 2 - Kafka Distribution"
        Kafka[Kafka Cluster<br/>20 brokers<br/>100 partitions<br/>Partitioned by symbol hash]
    end

    Ingestor -->|Publish to topic<br/>market . quotes<br/>Key: symbol| Kafka

    subgraph "Partition Assignment"
        P0[Partition 0<br/>AAPL, MSFT, ...<br/>1,000 symbols]
        P1[Partition 1<br/>GOOGL, AMZN, ...<br/>1,000 symbols]
        P2[Partition 2]
        P99[Partition 99<br/>TSLA, NVDA, ...]
    end

    Kafka --> P0
    Kafka --> P1
    Kafka --> P2
    Kafka --> P99

    subgraph "Level 3 - WebSocket Fanout (1,000 servers)"
        WS1[WS Server 1<br/>Consumes P0<br/>10,000 users<br/>Subscription Map in RAM]
        WS2[WS Server 2<br/>Consumes P1<br/>10,000 users]
        WS3[WS Server 3<br/>Consumes P2]
        WS1000[WS Server 1000<br/>Consumes P99]
    end

    P0 -->|1,000 symbols<br/>~1k updates/sec| WS1
    P1 -->|1,000 symbols| WS2
    P2 --> WS3
    P99 --> WS1000

    subgraph "Subscription Map Example (WS1)"
        SubMap["In-Memory Map:<br/>{<br/> AAPL: [user1, user5, user99, ...], // 2,000 users<br/> MSFT: [user3, user7, ...], // 1,500 users<br/> ...<br/>}"]
    end

    WS1 -.-> SubMap

    subgraph "Selective Fanout (WS1)"
        U1[User 1<br/>Watches: AAPL]
        U5[User 5<br/>Watches: AAPL]
        U99[User 99<br/>Watches: AAPL]
        U2k[User 2000<br/>Watches: AAPL]
    end

    WS1 -->|Push AAPL update<br/>Only to 2,000 subscribed users| U1
    WS1 --> U5
    WS1 --> U99
    WS1 --> U2k

    subgraph "Math: Fanout Reduction"
        Calc["Per-server fanout:<br/>100 updates/sec × 2,000 users = 200k msg/sec ✅<br/><br/>Total system fanout:<br/>1,000 servers × 200k = 200M msg/sec<br/>(Same total, but distributed across 1,000 servers)"]
    end

    style Exchange fill: #ff9999
    style Kafka fill: #99ff99
    style WS1 fill: #99ccff
    style SubMap fill: #ffff99
```

---

## Real-Time Market Data Flow

**Flow Explanation:**

This diagram shows the complete journey of a stock quote from the exchange to the user's screen.

**Steps:**

1. **Exchange publishes quote** (t=0ms): RELIANCE LTP=2450.75
2. **FIX message received** (t=5ms): Market Data Ingestor parses FIX 35=W
3. **Redis update** (t=10ms): SET quote:RELIANCE {"ltp": 2450.75, "bid": 2450.50, "ask": 2451.00}
4. **Kafka publish** (t=15ms): Produce to partition 42 (symbol hash)
5. **WebSocket consume** (t=35ms): WS Server 42 receives message
6. **Lookup subscribers** (t=38ms): subscriptionMap["RELIANCE"] → [user123, user456, ...]
7. **Push to users** (t=50ms): Send JSON message to 2,000 WebSocket connections
8. **User receives** (t=80ms): Client updates UI with new price

**Latency Breakdown:**

- Exchange → Ingestor: 5ms (network)
- Ingestor → Redis: 5ms (write)
- Ingestor → Kafka: 5ms (produce)
- Kafka → WebSocket: 20ms (consumer lag)
- WebSocket → User: 15ms (push)
- User network: 30ms (4G/WiFi)
- **Total: ~80ms** (well within 200ms SLA)

**Benefits:**

- Low latency: <100ms end-to-end
- Fault-tolerant: Kafka retains messages for replay
- Scalable: Add more WebSocket servers

```mermaid
sequenceDiagram
    participant Exchange as Exchange<br/>(NSE/BSE)
    participant Ingestor as Market Data<br/>Ingestor
    participant Redis as Redis<br/>Quote Cache
    participant Kafka as Kafka<br/>Topic
    participant WS as WebSocket<br/>Server
    participant User as User<br/>Client
    Note over Exchange: t=0ms<br/>Quote update triggered
    Exchange ->> Ingestor: FIX Message (35=W)<br/>RELIANCE LTP=2450.75<br/>Bid=2450.50, Ask=2451.00
    Note over Ingestor: t=5ms<br/>Parse FIX message

    par Update Redis Cache
        Ingestor ->> Redis: SET quote:RELIANCE<br/>{"ltp": 2450.75, "bid": 2450.50, "ask": 2451.00}
        Note over Redis: t=10ms<br/>In-memory write<br/><1ms latency
    and Publish to Kafka
        Ingestor ->> Kafka: Produce message<br/>Key: RELIANCE<br/>Partition: 42 (hash)
        Note over Kafka: t=15ms<br/>Append to log<br/>Replicated 3x
    end

    Note over Kafka: t=20ms<br/>Consumer lag: ~5ms
    Kafka ->> WS: Poll (batch of 100 messages)<br/>RELIANCE, TCS, INFY, ...
    Note over WS: t=35ms<br/>Received batch from Kafka
    Note over WS: Lookup subscription map<br/>subscriptionMap["RELIANCE"]<br/>→ [user123, user456, ..., user2000]
    WS ->> User: Push WebSocket message (2,000 users)<br/>{"symbol": "RELIANCE", "ltp": 2450.75}
    Note over User: t=80ms (total)<br/>Client receives update<br/>Updates UI
    Note over Exchange, User: End-to-end latency: ~80ms ✅<br/>(Target: <200ms)
```

---

## Order Entry and Execution Flow

**Flow Explanation:**

This diagram shows the complete flow of placing a stock order, from user click to ledger update.

**Steps:**

1. **User places order** (t=0ms): Buy 100 RELIANCE @ 2450
2. **API validation** (t=50ms): Check balance, risk limits, market hours
3. **ACID transaction** (t=150ms): Lock account, deduct balance, insert order
4. **FIX order routing** (t=200ms): Send NewOrderSingle to exchange
5. **Exchange matching** (t=205ms): Match with sell order (see 3.4.1 Stock Exchange)
6. **ExecutionReport** (t=210ms): FIX 35=8 (Order Filled)
7. **Kafka event** (t=220ms): Publish ORDER_FILLED event
8. **Ledger update** (t=10s, async): Update holdings, release margin
9. **User notification** (t=250ms): WebSocket push "Order filled"

**Latency:**

- User → Order filled notification: ~250ms
- User → Ledger updated: ~10 seconds (async)

**Benefits:**

- Fast: User sees confirmation in <1 second
- Reliable: ACID transaction ensures atomicity
- Auditable: Kafka event log for compliance

```mermaid
sequenceDiagram
    participant User as User<br/>Client
    participant API as Order Entry<br/>API (gRPC)
    participant PG as PostgreSQL<br/>Ledger
    participant FIX as FIX<br/>Engine
    participant Exchange as Exchange<br/>(NSE)
    participant Kafka as Kafka<br/>Event Log
    participant Ledger as Ledger<br/>Service
    participant WS as WebSocket<br/>Server
    Note over User: User clicks "Buy"<br/>100 RELIANCE @ 2450
    User ->> API: PlaceOrder(user_id=12345, symbol=RELIANCE,<br/>side=BUY, qty=100, price=2450)
    Note over API: Validation (50ms)<br/>- Check balance<br/>- Risk limits<br/>- Market hours
    API ->> PG: BEGIN TRANSACTION
    API ->> PG: SELECT cash_balance FROM accounts<br/>WHERE user_id=12345 FOR UPDATE
    PG -->> API: cash_balance = 50000
    Note over API: Check: 50000 >= 100 × 2450 = 245000?<br/>❌ Insufficient balance!<br/>(Would reject here if true)
    Note over API: Assume balance OK<br/>(50000 >= 10000 for 100 shares @ 100)
    API ->> PG: UPDATE accounts<br/>SET margin_used = margin_used + 10000<br/>WHERE user_id=12345
    API ->> PG: INSERT INTO orders<br/>(user_id, symbol, side, qty, price, status)<br/>VALUES (12345, RELIANCE, BUY, 100, 2450, PENDING)
    PG -->> API: order_id = 999888
    API ->> PG: COMMIT
    Note over API: t=150ms<br/>Transaction completed
    API ->> FIX: SendOrder(order_id=999888,<br/>symbol=RELIANCE, side=BUY, qty=100, price=2450)
    Note over FIX: Encode FIX message<br/>35=D (NewOrderSingle)<br/>11=999888 (ClOrdID)
    FIX ->> Exchange: FIX Message 35=D<br/>NewOrderSingle
    Note over Exchange: t=205ms<br/>Matching Engine processes<br/>(See 3.4.1 for details)<br/>Match found!
    Exchange ->> FIX: FIX Message 35=8<br/>ExecutionReport<br/>OrdStatus=FILLED (39=2)<br/>LastPx=2450, LastQty=100
    Note over FIX: t=210ms<br/>Order filled!
    FIX ->> API: ExecutionReport callback
    API ->> Kafka: Publish event<br/>{"type": "ORDER_FILLED",<br/>"order_id": 999888,<br/>"user_id": 12345,<br/>"symbol": "RELIANCE",<br/>"qty": 100,<br/>"price": 2450}
    Note over Kafka: t=220ms<br/>Event persisted<br/>(Immutable audit log)

    par User Notification
        API ->> WS: Send notification to user 12345
        WS ->> User: WebSocket push<br/>{"order_id": 999888, "status": "FILLED"}
        Note over User: t=250ms<br/>User sees "Order filled!"
    and Async Ledger Update
        Kafka ->> Ledger: Consume ORDER_FILLED event
        Note over Ledger: t=10 seconds (async)<br/>Process event
        Ledger ->> PG: UPDATE holdings<br/>SET quantity = quantity + 100,<br/>avg_price = recalculate()<br/>WHERE user_id=12345 AND symbol=RELIANCE
        Ledger ->> PG: UPDATE accounts<br/>SET margin_used = margin_used - 10000,<br/>cash_balance = cash_balance - 245000<br/>WHERE user_id=12345
    end

    Note over User, Ledger: Total latency:<br/>User click → Order filled notification: 250ms ✅<br/>User click → Ledger updated: ~10s (async)
```

---

## WebSocket Server Topology

**Flow Explanation:**

This diagram shows the WebSocket server cluster architecture and connection routing.

**Components:**

1. **Load Balancer (ALB):** Routes initial WebSocket handshake based on `user_id` hash
2. **1,000 WebSocket Servers:** Each handles 10,000 concurrent connections
3. **Sticky Sessions:** User always connects to same server (maintains subscription state)
4. **Health Checks:** ALB pings servers every 5 seconds

**Connection Flow:**

1. User initiates WebSocket connection
2. ALB computes: `user_id % 1000` → routes to specific server
3. Server accepts connection, stores in memory
4. User sends `subscribe` message with watchlist
5. Server updates subscription map
6. Kafka pushes quote updates → Server filters by subscription → Push to user

**Failover:**

1. Server crashes → ALB detects (15 seconds)
2. User's client detects disconnection
3. Client reconnects (exponential backoff)
4. ALB routes to different server
5. Client resubscribes to watchlist

**Memory per Server:**

- 10,000 connections × 50 KB = 500 MB
- Subscription map: ~10 MB
- **Total: ~600 MB** (fits in 8 GB RAM)

```mermaid
graph TB
    subgraph "Users - 10 Million"
        U1[User 1<br/>user_id=123]
        U2[User 2<br/>user_id=456]
        U3[User 3<br/>user_id=789]
        U10M[User 10M]
    end

    subgraph "Load Balancer Layer"
        ALB[AWS Application Load Balancer<br/>Layer 7<br/>WebSocket support<br/>Sticky sessions enabled]
    end

    U1 -->|WSS handshake| ALB
    U2 --> ALB
    U3 --> ALB
    U10M --> ALB

    subgraph "Routing Logic"
        Hash["Hash function:<br/>server_id = user_id mod 1000<br/><br/>User 123 → Server 123<br/>User 456 → Server 456<br/>User 789 → Server 789"]
    end

    ALB -.-> Hash

    subgraph "WebSocket Server Cluster - 1,000 Servers"
        WS1[WS Server 1<br/>10,000 connections<br/>c5.xlarge<br/>RAM: 8 GB]
        WS123[WS Server 123<br/>10,000 connections<br/>Handles user_id 123]
        WS456[WS Server 456<br/>10,000 connections<br/>Handles user_id 456]
        WS789[WS Server 789<br/>10,000 connections<br/>Handles user_id 789]
        WS1000[WS Server 1000<br/>10,000 connections]
    end

    ALB -->|Route user 123| WS123
    ALB -->|Route user 456| WS456
    ALB -->|Route user 789| WS789
    ALB -->|Route others| WS1
    ALB -->|Route others| WS1000

    subgraph "Server Internal (WS123)"
        Conn["Connection Pool<br/>10,000 active WebSockets<br/>Memory: 500 MB"]
        SubMap["Subscription Map (RAM):<br/>{<br/> AAPL: [user123, user234, ...],<br/> GOOGL: [user567, user123, ...],<br/> ...<br/>}<br/>Memory: 10 MB"]
        KafkaC[Kafka Consumer<br/>Partition: 42<br/>Poll interval: 100ms]
        RedisC[Redis Client<br/>Connection pool: 10]
    end

    WS123 --> Conn
    WS123 --> SubMap
    WS123 --> KafkaC
    WS123 --> RedisC

    subgraph "Data Sources"
        Kafka[Kafka<br/>market.quotes topic]
        Redis[(Redis<br/>Quote cache)]
    end

    KafkaC -->|Consume messages| Kafka
    RedisC -->|GET quote:SYMBOL| Redis

    subgraph "Health Check"
        HC[ALB Health Check<br/>Interval: 5 seconds<br/>Timeout: 3 seconds<br/>Unhealthy threshold: 3 failures]
    end

    HC -.->|HTTP GET /health| WS123
    HC -.->|HTTP GET /health| WS456

    subgraph "Failover Scenario"
        Failure["Server 123 crashes<br/>↓<br/>ALB marks unhealthy (15s)<br/>↓<br/>10,000 users disconnected<br/>↓<br/>Clients reconnect (exponential backoff)<br/>↓<br/>ALB routes to other servers<br/>↓<br/>Clients resubscribe to watchlist"]
    end

    WS123 -.->|X Crash| Failure
    style ALB fill: #ff9999
    style WS123 fill: #99ccff
    style Kafka fill: #99ff99
    style Redis fill: #ffcc99
```

---

## Event Sourcing Architecture

**Flow Explanation:**

This diagram shows the Event Sourcing pattern for the account ledger, solving the lock contention problem.

**Problem:**

- Traditional approach: `UPDATE accounts SET cash_balance = cash_balance - 1500 WHERE user_id=12345`
- Requires exclusive lock on row → Serializes all concurrent updates → Slow

**Solution: Event Sourcing**

**Concept:** Instead of updating balance directly, append immutable events to a log. Balance is derived by replaying
events.

**Components:**

1. **Kafka Event Log:** Immutable, append-only log (compacted topic)
2. **Event Producer:** Order Entry API appends events (ORDER_PLACED, ORDER_FILLED, DEPOSIT, WITHDRAWAL)
3. **Event Consumer:** Ledger Service replays events, updates materialized view
4. **Materialized View:** PostgreSQL snapshot (read-optimized, updated every 1 second)

**Benefits:**

- ✅ No locks: Kafka appending is lock-free (partition-level ordering)
- ✅ Audit trail: Complete history of all transactions (regulatory requirement)
- ✅ Replayable: Reconstruct balance at any point in time
- ✅ Scalable: Kafka handles 100k+ writes/sec

**Trade-offs:**

- ❌ Eventual consistency: Snapshot lags behind event log by ~1 second
- ❌ Complexity: Requires event replay logic

```mermaid
graph TB
    subgraph "Event Producers"
        OrderAPI[Order Entry API]
        DepositAPI[Deposit API]
        WithdrawAPI[Withdrawal API]
    end

    subgraph "Kafka Event Log - Immutable Append-Only"
        KafkaTopic[Kafka Topic: ledger.events<br/>Partitioned by user_id<br/>Compacted - retain latest per key<br/>Retention: 7 years]
    end

    OrderAPI -->|Append ORDER_PLACED event| KafkaTopic
    OrderAPI -->|Append ORDER_FILLED event| KafkaTopic
    DepositAPI -->|Append DEPOSIT event| KafkaTopic
    WithdrawAPI -->|Append WITHDRAWAL event| KafkaTopic

    subgraph "Event Log Example (user_id=12345)"
        E1["Event 1 (t=1000):<br/>{user_id: 12345, type: DEPOSIT, amount: 10000}"]
        E2["Event 2 (t=1010):<br/>{user_id: 12345, type: ORDER_PLACED, amount: -1500, order_id: 888}"]
        E3["Event 3 (t=1015):<br/>{user_id: 12345, type: ORDER_FILLED, amount: 0, order_id: 888}"]
        E4["Event 4 (t=1020):<br/>{user_id: 12345, type: HOLDING_ADDED, symbol: RELIANCE, qty: 100}"]
    end

    KafkaTopic --> E1
    E1 --> E2
    E2 --> E3
    E3 --> E4

    subgraph "Event Consumers"
        LedgerSvc[Ledger Service<br/>Kafka Consumer Group<br/>Replays events every 1 second]
    end

    KafkaTopic -->|Consume events<br/>Consumer offset tracking| LedgerSvc

    subgraph "Materialized View - PostgreSQL"
        PG[(PostgreSQL<br/>Read-Optimized Snapshot<br/>Updated every 1 second)]
        AcctTable["Table: accounts<br/>user_id | cash_balance | margin_used | updated_at<br/>12345 | 8500 | 0 | 2023-11-01 10:00:01"]
        HoldingTable["Table: holdings<br/>user_id | symbol | quantity | avg_price<br/>12345 | RELIANCE | 100 | 1500"]
    end

    LedgerSvc -->|UPDATE accounts| AcctTable
    LedgerSvc -->|UPDATE holdings| HoldingTable

    subgraph "Balance Calculation"
        Calc["Current balance (derived from events):<br/><br/>SUM(all DEPOSIT/WITHDRAWAL events)<br/>- SUM(margin_used from pending orders)<br/><br/>Event 1: +10000<br/>Event 2: -1500 (margin blocked)<br/>Event 3: Release margin (order filled)<br/><br/>Balance = 10000 - 1500 = 8500"]
    end

    LedgerSvc -.-> Calc

    subgraph "Read Path"
        API[Portfolio API]
        User[User requests balance]
    end

    User -->|GET /portfolio| API
    API -->|SELECT cash_balance FROM accounts<br/>WHERE user_id = 12345| AcctTable
    AcctTable -->|8500| API
    API -->|Response| User
    Note1["✅ Write Path Fast:<br/>Append to Kafka = Lock-free<br/>Latency under 10ms"]
    Note2["✅ Read Path Fast:<br/>Query PostgreSQL snapshot<br/>Latency under 50ms"]
    Note3["⚠️ Trade-off:<br/>Eventual consistency<br/>Snapshot lags by 1 second"]
    OrderAPI -.-> Note1
    API -.-> Note2
    LedgerSvc -.-> Note3
    style KafkaTopic fill: #99ff99
    style PG fill: #ffcc99
    style LedgerSvc fill: #99ccff
```

---

## Search and Discovery Flow

**Flow Explanation:**

This diagram shows the flow when a user searches for a stock ticker.

**Steps:**

1. **User types "AP"** in search box
2. **UI debounces** (wait 300ms after last keystroke)
3. **Search API** queries Elasticsearch with prefix query `"AP*"`
4. **Elasticsearch** returns top 10 matches: [AAPL, APPL, APHA, ...]
5. **Search API** fetches current quotes from Redis: MGET quote:AAPL quote:APPL ...
6. **UI displays** results with live prices
7. **User clicks AAPL** → WebSocket subscribes to AAPL stream

**Performance:**

- Elasticsearch query: <30ms
- Redis MGET (10 keys): <5ms
- **Total: <50ms** (within SLA)

**Optimization:**

- **Caching:** Cache popular queries (e.g., "AAPL") in Redis (1-minute TTL)
- **Debouncing:** Reduce load by waiting 300ms
- **Pagination:** Return top 10, fetch more on scroll

```mermaid
sequenceDiagram
    participant User as User<br/>Client
    participant UI as UI<br/>Search Box
    participant API as Search API
    participant Cache as Redis<br/>Cache
    participant ES as Elasticsearch<br/>Stock Index
    participant Redis as Redis<br/>Quote Cache
    participant WS as WebSocket<br/>Server
    Note over User: User types in search box
    User ->> UI: Keystroke: "A"
    Note over UI: Debounce timer: 300ms
    User ->> UI: Keystroke: "AP"
    Note over UI: Debounce timer reset: 300ms
    Note over UI: 300ms elapsed<br/>No more keystrokes<br/>Trigger search
    UI ->> API: GET /search?q=AP
    Note over API: Check cache first
    API ->> Cache: GET search:AP
    Cache -->> API: MISS (null)
    Note over API: Cache miss<br/>Query Elasticsearch
    API ->> ES: Search query:<br/>{"query": {"prefix": {"symbol": "AP"}},<br/>"size": 10,<br/>"sort": ["market_cap"]}
    Note over ES: Prefix search on inverted index<br/>Latency: ~30ms
    ES -->> API: Results: [<br/> {symbol: "AAPL", name: "Apple Inc.", market_cap: 2.8T},<br/> {symbol: "APPL", name: "AppLovin", market_cap: 30B},<br/> {symbol: "APHA", name: "Aphria Inc.", market_cap: 5B},<br/> ...<br/>]
    Note over API: Hydrate with live quotes
    API ->> Redis: MGET quote:AAPL quote:APPL quote:APHA ...
    Note over Redis: Fetch 10 keys<br/>Latency: <5ms
    Redis -->> API: [{ltp: 150.25}, {ltp: 75.30}, {ltp: 12.50}, ...]
    Note over API: Merge results
    API ->> Cache: SETEX search:AP 60<br/>[{symbol: "AAPL", ltp: 150.25}, ...]
    Note over Cache: Cache for 1 minute
    API -->> UI: Response (JSON):<br/>[<br/> {symbol: "AAPL", name: "Apple Inc.", ltp: 150.25},<br/> {symbol: "APPL", name: "AppLovin", ltp: 75.30},<br/> ...<br/>]
    Note over UI: Display results<br/>Total latency: <50ms ✅
    UI -->> User: Show search results with live prices
    Note over User: User clicks on "AAPL"
    User ->> UI: Click "AAPL"
    UI ->> WS: Subscribe to AAPL<br/>{"action": "subscribe", "symbols": ["AAPL"]}
    Note over WS: Add user to subscription map:<br/>subscriptionMap["AAPL"].push(user_id)
    WS -->> UI: Subscription confirmed
    Note over User: Now receives real-time<br/>AAPL price updates
```

---

## Risk Management System

**Flow Explanation:**

This diagram shows the real-time risk management system that monitors margin positions and triggers alerts.

**Components:**

1. **Margin Monitor:** Runs every 1 minute during market hours
2. **Quote Stream:** Live prices from Redis
3. **Holdings Database:** User positions from PostgreSQL
4. **Alert System:** Sends margin calls, auto-liquidates positions

**Monitoring Logic:**

1. For each user with margin positions:
2. Fetch current holdings from PostgreSQL
3. Calculate position value using live quotes from Redis
4. Calculate equity ratio: `equity / position_value`
5. If equity ratio < 30% → Trigger margin call
6. If grace period expired → Auto-liquidate positions

**Margin Call Example:**

```
User: 12345
Holdings: 100 RELIANCE @ avg buy price $200
Current price: $130 (30% drop)

Position value: 100 × $130 = $13,000
Margin borrowed: $10,000
Equity: $13,000 - $10,000 = $3,000
Equity ratio: $3,000 / $13,000 = 23% ❌ (below 30%)

Action: Trigger margin call
```

**Circuit Breaker:**

- Monitors index level (NIFTY) every 1 second
- If decline > 10%/15%/20% → Halt trading

```mermaid
graph TB
    subgraph "Risk Monitoring Service"
        Monitor[Margin Monitor<br/>Cron job: every 1 minute<br/>Runs during market hours]
    end

    Monitor -->|Start| Check[Check all users<br/>with margin positions]

    subgraph "Data Sources"
        PG[(PostgreSQL<br/>Holdings + Accounts)]
        Redis[(Redis<br/>Live Quotes)]
    end

    Check -->|Query| PG
    Check -->|Fetch quotes| Redis

    subgraph "Calculation Logic"
        Calc["FOR each user:<br/><br/>holdings = get_holdings(user_id)<br/>position_value = 0<br/><br/>FOR each holding:<br/> quote = redis.get(f'quote:{symbol}')<br/> position_value += holding.qty × quote.ltp<br/><br/>equity = position_value - margin_used<br/>equity_ratio = equity / position_value"]
    end

    Check --> Calc

    subgraph "Decision Tree"
        D1{equity_ratio < 30%?}
        D2{grace_period_expired?}
    end

    Calc --> D1
    D1 -->|Yes ❌| Trigger[Trigger Margin Call]
    D1 -->|No ✅| OK[Position OK<br/>No action]
    Trigger --> D2
    D2 -->|Yes| Liquidate[Auto-Liquidate<br/>Force sell positions]
    D2 -->|No| Grace[Grace Period<br/>1 hour to add funds]

    subgraph "Notification System"
        Email[Email Alert<br/>Subject: URGENT MARGIN CALL]
        SMS[SMS Alert<br/>Text message]
        Push[Push Notification<br/>Mobile app]
        WS[WebSocket Alert<br/>Red banner in UI]
    end

    Trigger --> Email
    Trigger --> SMS
    Trigger --> Push
    Trigger --> WS

    subgraph "Liquidation Process"
        L1[Create MARKET SELL orders<br/>for all holdings]
        L2[Route orders to exchange<br/>via FIX protocol]
        L3[Update ledger:<br/>Repay borrowed amount<br/>Return remaining equity]
    end

    Liquidate --> L1
    L1 --> L2
    L2 --> L3

    subgraph "Circuit Breaker Monitor"
        CBMonitor[Circuit Breaker Monitor<br/>Runs every 1 second]
    end

    CBMonitor -->|Fetch| IndexLevel[Get NIFTY level<br/>from Redis]

    subgraph "Circuit Breaker Logic"
        CB1["decline_pct = (prev_close - current) / prev_close × 100"]
        CB2{decline_pct >= 20%?}
        CB3{decline_pct >= 15%?}
        CB4{decline_pct >= 10%?}
    end

    IndexLevel --> CB1
    CB1 --> CB2
    CB2 -->|Yes| Halt3[Halt Level 3<br/>Trading halted for rest of day]
    CB2 -->|No| CB3
    CB3 -->|Yes| Halt2[Halt Level 2<br/>Trading halted for 1h 45min]
    CB3 -->|No| CB4
    CB4 -->|Yes| Halt1[Halt Level 1<br/>Trading halted for 45min]
    CB4 -->|No| Normal[Normal Trading]

    subgraph "Halt Actions"
        HA1[Reject all new orders<br/>with error message]
        HA2[Display banner:<br/>Trading halted due to circuit breaker]
        HA3[Send push notification<br/>to all users]
    end

    Halt1 --> HA1
    Halt2 --> HA1
    Halt3 --> HA1
    HA1 --> HA2
    HA2 --> HA3
    style Monitor fill: #ff9999
    style Trigger fill: #ffcc99
    style Liquidate fill: #ff6666
    style Halt3 fill: #cc0000
```

---

## FIX Protocol Integration

**Flow Explanation:**

This diagram shows the FIX protocol session lifecycle and message exchange with the stock exchange.

**FIX (Financial Information eXchange) Protocol:**

- Industry-standard protocol for order routing
- Session-based: Persistent TCP connection
- Sequence numbers: Every message has `MsgSeqNum` (prevents duplicates, detects gaps)
- Heartbeats: Keepalive messages every 30 seconds

**Session Lifecycle:**

1. **Logon (35=A):** Establish session, authenticate
2. **Heartbeats (35=0):** Keep session alive every 30 seconds
3. **Market Data (35=W):** Receive quote updates
4. **Order Flow (35=D, 35=8):** Place orders, receive fills
5. **Resend Requests (35=2):** Handle sequence number gaps
6. **Logout (35=5):** Gracefully close session

**Performance:**

- Order placement latency: ~100ms (end-to-end)
- Throughput: 1,000 orders/sec per session

```mermaid
stateDiagram-v2
    [*] --> Disconnected
    Disconnected --> LogonSent: TCP connect<br/>Send Logon (35=A)
    state "Logon Sent" as LogonSent
    LogonSent --> Active: Receive Logon (35=A)
    LogonSent --> Disconnected: Timeout (30s)
    state "Active Session" as Active {
[*] --> Idle

Idle --> SendingOrder: User places order
SendingOrder --> Idle: Send NewOrderSingle (35=D)

Idle --> ReceivingFill: Receive ExecutionReport (35=8)
ReceivingFill --> Idle: Process fill

Idle --> ReceivingMarketData: Receive MarketDataSnapshot (35=W)
ReceivingMarketData --> Idle: Update quote cache

Idle --> SendingHeartbeat: 30 seconds elapsed
SendingHeartbeat --> Idle: Send Heartbeat (35=0)

Idle --> ReceivingHeartbeat: Receive Heartbeat (35=0)
ReceivingHeartbeat --> Idle: Reset timer

Idle --> GapDetected: Sequence number gap detected
state "Gap Detected" as GapDetected
GapDetected --> SendingResendRequest : Send ResendRequest (35=2)
state "Sending Resend Request" as SendingResendRequest
SendingResendRequest --> Idle: Receive missing messages
}

Active --> LogoutSent: Initiate disconnect<br/>Send Logout (35=5)
state "Logout Sent" as LogoutSent
LogoutSent --> Disconnected: Receive Logout (35=5)

Active --> Disconnected : Heartbeat timeout (60s)<br/>Connection lost

Disconnected --> LogonSent : Reconnect

note right of Active
All messages include:
- MsgSeqNum (34)
- SendingTime (52)
- SenderCompID (49)
- TargetCompID (56)

Sequence number tracking:
- Sent: last_sent_seqnum++
- Received: last_rcvd_seqnum++
- Gap detection: expected != actual
end note
```

---

## Multi-Region Deployment

**Flow Explanation:**

This diagram shows the multi-region architecture for low-latency global access and data residency compliance.

**Regions:**

1. **Mumbai (Primary):** Nearest to NSE/BSE exchanges (Indian users)
2. **Singapore (Secondary):** Southeast Asia (Singapore users)
3. **London (Secondary):** Europe, Middle East (EU users)

**Data Residency Requirements:**

- **SEBI (India):** All Indian user data MUST be stored in India
- **GDPR (Europe):** European user data MUST stay in EU
- **Solution:** Shard database by region, replicate read-only data cross-region

**Cross-Region Replication:**

- **Quote Data:** Mumbai → Singapore/London via Kafka (~100ms latency)
- **User Data:** NO cross-region replication (compliance)

**Disaster Recovery:**

- **RTO:** 1 hour (recovery time objective)
- **RPO:** 15 minutes (recovery point objective - max data loss)

```mermaid
graph TB
    subgraph "Mumbai Region (Primary) - India"
        M_LB[Load Balancer<br/>Route53 DNS]
        M_WS[WebSocket Servers<br/>500 nodes]
        M_PG[(PostgreSQL<br/>Indian users<br/>5 shards)]
        M_Redis[(Redis<br/>Quote cache<br/>All symbols)]
        M_Kafka[Kafka Cluster<br/>20 brokers]
        M_FIX[FIX Engine<br/>NSE/BSE connection<br/>Active]
    end

    subgraph "NSE/BSE Exchange (Mumbai)"
        Exchange[Stock Exchange<br/>Market data source]
    end

    Exchange -->|FIX Protocol<br/>Dedicated line| M_FIX
    M_FIX -->|Quote stream| M_Kafka
    M_Kafka --> M_Redis
    M_Kafka --> M_WS
    M_WS --> M_LB

    subgraph "Indian Users (5M)"
        IN_Users[Indian Users<br/>Data stored in Mumbai<br/>SEBI compliance]
    end

    IN_Users <-->|Low latency<br/>~20ms| M_LB

    subgraph "Singapore Region (Secondary) - Southeast Asia"
        S_LB[Load Balancer<br/>Route53 DNS]
        S_WS[WebSocket Servers<br/>300 nodes]
        S_PG[(PostgreSQL<br/>Singapore users<br/>3 shards)]
        S_Redis[(Redis<br/>Quote cache<br/>Read replica)]
        S_Kafka[Kafka Mirror<br/>Consumer only]
        S_FIX[FIX Engine<br/>NSE connection<br/>Standby]
    end

    M_Kafka -->|Cross - region replication<br/>~100ms latency| S_Kafka
    S_Kafka --> S_Redis
    S_Kafka --> S_WS
    S_WS --> S_LB

    subgraph "Singapore Users (3M)"
        SG_Users[Singapore Users<br/>Data stored in Singapore<br/>Local data residency]
    end

    SG_Users <-->|Low latency<br/>~30ms| S_LB

    subgraph "London Region (Secondary) - Europe"
        L_LB[Load Balancer<br/>Route53 DNS]
        L_WS[WebSocket Servers<br/>200 nodes]
        L_PG[(PostgreSQL<br/>EU users<br/>2 shards)]
        L_Redis[(Redis<br/>Quote cache<br/>Read replica)]
        L_Kafka[Kafka Mirror<br/>Consumer only]
        L_FIX[FIX Engine<br/>NSE connection<br/>Standby]
    end

    M_Kafka -->|Cross - region replication<br/>~150ms latency| L_Kafka
    L_Kafka --> L_Redis
    L_Kafka --> L_WS
    L_WS --> L_LB

    subgraph "EU Users (2M)"
        EU_Users[EU Users<br/>Data stored in London<br/>GDPR compliance]
    end

    EU_Users <-->|Low latency<br/>~40ms| L_LB

    subgraph "Disaster Recovery Scenario"
        DR["Mumbai Datacenter Outage<br/><br/>Recovery Steps:<br/>1. DNS Failover (5 min)<br/> Route53: mumbai → singapore<br/>2. Activate Standby FIX (10 min)<br/> Singapore FIX engine connects to NSE<br/>3. DB Restoration (30 min)<br/> Restore from backup<br/> (Taken every 15 minutes)<br/>4. Resume Operations<br/><br/>Total downtime: 30-45 minutes<br/>Data loss: Up to 15 minutes"]
    end

    M_PG -.->|Backup every 15 min<br/>S3 Cross - region replication| DR
    S_FIX -.->|Standby connection<br/>Activated on failover| Exchange

    subgraph "Cross-Region Data Flow"
        Flow["Quote Data Flow:<br/>Mumbai (source) → Singapore/London (replicas)<br/>Latency: 100-150ms (acceptable for retail)<br/><br/>Order Data Flow:<br/>All orders route to Mumbai FIX engine<br/>(Direct exchange connection)<br/><br/>User Data Flow:<br/>NO cross-region replication<br/>(Compliance: Data residency laws)"]
    end

    M_Kafka -.-> Flow
    S_Kafka -.-> Flow
    L_Kafka -.-> Flow
    style Exchange fill: #ff9999
    style M_PG fill: #ffcc99
    style S_PG fill: #ffcc99
    style L_PG fill: #ffcc99
    style DR fill: #ff6666
```

---

## Audit Logging Architecture

**Flow Explanation:**

This diagram shows the complete audit logging system for regulatory compliance (7-year retention).

**Requirements:**

- **Immutable:** Logs cannot be modified (regulatory requirement)
- **Comprehensive:** Log every order, trade, deposit, withdrawal
- **Queryable:** Fast queries for compliance reporting
- **Long retention:** 7 years (SEBI/SEC requirement)

**Architecture:**

1. **Event Capture:** All critical events published to Kafka
2. **ClickHouse:** OLAP database optimized for time-series queries
3. **Tiered Storage:** Hot (6 months, SSD) → Warm (2 years, HDD) → Cold (7 years, S3 Glacier)

**Compliance Queries:**

- Reconstruct user's trading history
- Detect wash sales (buy/sell within 30 days)
- Best execution analysis (slippage measurement)
- GDPR compliance (pseudonymization)

```mermaid
graph TB
    subgraph "Event Sources"
        OrderAPI[Order Entry API]
        DepositAPI[Deposit API]
        LoginAPI[Login API]
        AdminAPI[Admin API<br/>Account freezes]
    end

    subgraph "Event Bus"
        Kafka[Kafka Topic: audit.events<br/>Retention: 7 days<br/>Buffer before ClickHouse]
    end

    OrderAPI -->|ORDER_PLACED<br/>ORDER_FILLED<br/>ORDER_CANCELLED| Kafka
    DepositAPI -->|DEPOSIT<br/>WITHDRAWAL| Kafka
    LoginAPI -->|LOGIN<br/>LOGOUT<br/>LOGIN_FAILED| Kafka
    AdminAPI -->|ACCOUNT_FROZEN<br/>MARGIN_CALL<br/>LIQUIDATION| Kafka

    subgraph "Snapshot Services"
        QuoteSnap[Quote Snapshot Service<br/>Captures market state<br/>at time of event]
        RiskSnap[Risk Snapshot Service<br/>Captures margin levels<br/>position limits]
    end

    Kafka --> QuoteSnap
    Kafka --> RiskSnap

    subgraph "Audit Database - ClickHouse"
        CH[(ClickHouse<br/>3-node cluster<br/>OLAP optimized<br/>Time-series partitioning)]
    end

    subgraph "ClickHouse Consumer"
        Consumer[Kafka Consumer<br/>Batch insert every 10 seconds<br/>Buffer: 10,000 events]
    end

    Kafka -->|Consume| Consumer
    QuoteSnap -->|Enrich events<br/>with market snapshot| Consumer
    RiskSnap -->|Enrich events<br/>with risk metrics| Consumer
    Consumer -->|Batch INSERT| CH

    subgraph "Tiered Storage Strategy"
        Hot["Hot Storage (6 months)<br/>SSD storage<br/>Query latency: <100ms<br/>Used for: Real-time compliance checks"]
        Warm["Warm Storage (2 years)<br/>HDD storage<br/>Query latency: ~1 second<br/>Used for: Monthly reports"]
        Cold["Cold Storage (7 years)<br/>S3 Glacier<br/>Query latency: ~hours<br/>Used for: Regulatory audits"]
    end

    CH --> Hot
    Hot -->|Move after 6 months| Warm
    Warm -->|Move after 2 years| Cold

    subgraph "Compliance Queries"
        Q1[Query 1: Trading History<br/>SELECT * FROM audit_log<br/>WHERE user_id=12345<br/>AND timestamp BETWEEN ...]
        Q2[Query 2: Wash Sales<br/>Detect buy/sell within 30 days<br/>JOIN trades on user+symbol]
        Q3[Query 3: Best Execution<br/>Compare execution price<br/>vs market price slippage]
    end

    Hot --> Q1
    Hot --> Q2
    Hot --> Q3

    subgraph "GDPR Compliance"
        GDPR["Right to be Forgotten:<br/><br/>1. Replace user_id with SHA256(user_id + salt)<br/>2. Store mapping: hash → user_id (encrypted)<br/>3. Audit log retains pseudonymized records<br/>4. Delete mapping after statute expires<br/><br/>Audit log remains immutable (regulatory)<br/>but cannot identify user (GDPR compliant)"]
    end

    CH -.->|Pseudonymization| GDPR

    subgraph "Monitoring"
        M1[Alert: ClickHouse lag > 1 minute]
        M2[Alert: Disk usage > 80 percent]
        M3[Alert: Query latency > 5 seconds]
    end

    CH -.-> M1
    Hot -.-> M2
    Q1 -.-> M3
    style Kafka fill: #99ff99
    style CH fill: #cc99ff
    style Cold fill: #ffcc99
    style GDPR fill: #ff9999
```

---

## Performance Optimization Pipeline

**Flow Explanation:**

This diagram shows the various performance optimization techniques applied at each layer of the stack.

**Optimizations:**

1. **WebSocket Layer:** Message compression (55 percent reduction), connection pooling
2. **Redis Layer:** Pipelining (20x faster), cluster sharding
3. **Kafka Layer:** Producer batching (100x fewer network calls), compression
4. **CDN Layer:** Cache historical charts (99.9999 percent hit rate)

**Impact:**

- Bandwidth: 6.8 GB/sec saved (compression)
- Latency: 100ms → 5ms (Redis pipelining)
- Network calls: 100k/sec → 1k/sec (Kafka batching)
- Cost: $500k/month → $50k/month (CDN caching)

```mermaid
graph LR
    subgraph "Optimization 1 - WebSocket Compression"
        WS_Before["Before:<br/>Uncompressed message:<br/>62 bytes<br/><br/>200M msg/sec × 62 bytes<br/>= 12.4 GB/sec bandwidth"]
        WS_Opt[Per-Message Deflate<br/>RFC 7692<br/>zlib compression level 1]
        WS_After["After:<br/>Compressed message:<br/>28 bytes<br/><br/>200M msg/sec × 28 bytes<br/>= 5.6 GB/sec bandwidth<br/><br/>✅ Savings: 6.8 GB/sec<br/>✅ Cost savings: $50k/month"]
    end

    WS_Before --> WS_Opt
    WS_Opt --> WS_After

    subgraph "Optimization 2 - Redis Pipelining"
        R_Before["Before:<br/>Sequential SET commands:<br/><br/>FOR 100 quotes:<br/> redis.SET<br/>(1ms per command)<br/><br/>Total: 100ms"]
        R_Opt[Redis Pipeline<br/>Batch 100 commands<br/>Single network round-trip]
        R_After["After:<br/>Batched execution:<br/><br/>pipeline.SET (100x)<br/>pipeline.execute<br/><br/>Total: 5ms<br/><br/>✅ 20x faster"]
    end

    R_Before --> R_Opt
    R_Opt --> R_After

    subgraph "Optimization 3 - Kafka Batching"
        K_Before["Before:<br/>Individual produces:<br/><br/>100k messages/sec<br/>= 100k network calls/sec"]
        K_Opt[Producer Config:<br/>linger.ms = 10<br/>batch.size = 100KB<br/>compression.type = snappy]
        K_After["After:<br/>Batched produces:<br/><br/>~1k batches/sec<br/>(100 messages per batch)<br/><br/>✅ 100x fewer network calls<br/>✅ 60 percent compression"]
    end

    K_Before --> K_Opt
    K_Opt --> K_After

    subgraph "Optimization 4 - CDN Caching"
        CDN_Before["Before:<br/>Historical chart API:<br/><br/>1M requests/hour<br/>= 1M backend queries/hour<br/><br/>Backend cost: $500k/month"]
        CDN_Opt[Cloudflare CDN<br/>Cache-Control: max-age=3600<br/>Edge caching]
        CDN_After["After:<br/>CDN serves from cache:<br/><br/>1M requests/hour<br/>= 1 backend query/hour<br/>(99.9999 percent cache hit)<br/><br/>CDN cost: $5k/month<br/><br/>✅ 100x reduction<br/>✅ $495k/month saved"]
    end

    CDN_Before --> CDN_Opt
    CDN_Opt --> CDN_After
    style WS_After fill: #99ff99
    style R_After fill: #99ff99
    style K_After fill: #99ff99
    style CDN_After fill: #99ff99
```

---

## Failure Scenarios and Recovery

**Flow Explanation:**

This diagram shows common failure scenarios and automated recovery procedures.

**Scenarios:**

1. **WebSocket Server Crash:** 10,000 users disconnected
2. **PostgreSQL Primary Failure:** Promote replica to primary
3. **Kafka Broker Failure:** Rebalance partitions
4. **Redis Node Failure:** Failover to replica

**Recovery:**

- **Automated:** Most failures handled automatically (health checks, failover)
- **Manual:** Complex scenarios require human intervention (data corruption)

```mermaid
stateDiagram-v2
    [*] --> Healthy
    state "Healthy State" as Healthy
    Healthy --> WSFailure: WebSocket server crashes
    Healthy --> PGFailure: PostgreSQL primary fails
    Healthy --> KafkaFailure: Kafka broker fails
    Healthy --> RedisFailure: Redis node fails
    state "WebSocket Server Failure" as WSFailure {
[*] --> Detected
Detected --> Disconnecting : ALB marks unhealthy<br/>(15 seconds)
Disconnecting --> Reconnecting: 10,000 users detect disconnect
Reconnecting --> Resubscribing: Clients reconnect<br/>(exponential backoff)
Resubscribing --> [*]: Clients resubscribe<br/>to watchlist
}

WSFailure --> Healthy: Recovery 5-10 seconds

state "PostgreSQL Primary Failure" as PGFailure {
[*] --> PGDetected
PGDetected --> Promoting : Health check fails<br/>(3 failed pings)
Promoting --> Updating : Promote replica to primary<br/>(30 seconds)
Updating --> [*]: Update connection strings<br/>in API servers
}

PGFailure --> Healthy: Recovery 1-2 minutes

state "Kafka Broker Failure" as KafkaFailure {
[*] --> KDetected
KDetected --> Rebalancing : Broker heartbeat timeout<br/>(10 seconds)
Rebalancing --> Reassigning: Partition leader election<br/>(5 seconds)
Reassigning --> [*] : Consumers rejoin<br/>consumer group
}

KafkaFailure --> Healthy: Recovery 15-20 seconds

state "Redis Node Failure" as RedisFailure {
[*] --> RDetected
RDetected --> RFailover : Sentinel detects failure<br/>(5 seconds)
RFailover --> [*]: Promote replica<br/>Update clients<br/>(10 seconds)
}

RedisFailure --> Healthy: Recovery 15 seconds

note right of Healthy
Monitoring:
- Health checks every 5 seconds
- Automated alerting (PagerDuty)
- Runbooks for manual procedures
end note
```

---

## Monitoring and Observability Dashboard

**Flow Explanation:**

This diagram shows the complete monitoring architecture with Prometheus, Grafana, and ClickHouse.

**Key Metrics:**

1. **Quote Latency (E2E):** Target <200ms, alert if >500ms
2. **Order Placement Latency:** Target <1s, alert if >3s
3. **WebSocket Connection Count:** Capacity planning
4. **Order Success Rate:** Detect exchange connectivity issues
5. **Kafka Consumer Lag:** Real-time pipeline health

**Alerting:**

- **Critical:** PagerDuty (24/7 oncall)
- **Warning:** Slack channel
- **Info:** Email digest

```mermaid
graph TB
    subgraph "Metrics Collection Layer"
        Prom[Prometheus<br/>3 instances<br/>Scrape interval: 15s]
    end

    subgraph "Application Metrics"
        API[Order Entry API<br/>/metrics endpoint]
        WS[WebSocket Servers<br/>/metrics endpoint]
        Kafka[Kafka Brokers<br/>JMX exporter]
        Redis[Redis<br/>redis_exporter]
        PG[PostgreSQL<br/>postgres_exporter]
    end

    Prom -->|Scrape| API
    Prom -->|Scrape| WS
    Prom -->|Scrape| Kafka
    Prom -->|Scrape| Redis
    Prom -->|Scrape| PG

    subgraph "Business Metrics"
        ClickHouse[(ClickHouse<br/>Analytics DB<br/>CTR, Conversion, Revenue)]
    end

    Kafka -.->|Stream user events| ClickHouse

    subgraph "Visualization - Grafana"
        Dashboard1["Dashboard 1 - Serving Health<br/>P99 Latency 45ms OK<br/>Cache Hit Rate 82 percent OK<br/>QPS 98k<br/>Error Rate 0.01 percent"]
        Dashboard2["Dashboard 2 - Infrastructure<br/>Redis Latency 2ms OK<br/>Redis QPS 1.1M<br/>Memory Usage 75 percent<br/>Kafka Lag 50ms"]
        Dashboard3["Dashboard 3 - Business<br/>Orders Placed 50k/day<br/>Orders Filled 49.5k/day<br/>Success Rate 99 percent<br/>Revenue $5M/day"]
    end

    Prom -->|Query metrics| Dashboard1
    Prom -->|Query metrics| Dashboard2
    ClickHouse -->|Query business metrics| Dashboard3

    subgraph "Alerting - Alertmanager"
        Alertmanager[Prometheus Alertmanager<br/>Rule engine<br/>Deduplication<br/>Routing]
        Rules["Alert Rules<br/>1. P99 Latency over 100ms<br/>2. Cache Hit Rate under 70 percent<br/>3. Order Success Rate under 95 percent<br/>4. Kafka Lag over 10 seconds<br/>5. Redis Down"]
    end

    Prom -->|Trigger alerts| Alertmanager
    Alertmanager -->|Load rules| Rules

    subgraph "Alert Channels"
        PagerDuty[PagerDuty<br/>Critical alerts<br/>24/7 oncall]
        Slack[Slack<br/>#alerts channel<br/>Warning alerts]
        Email[Email<br/>Daily digest<br/>Info alerts]
    end

    Alertmanager -->|Critical| PagerDuty
    Alertmanager -->|Warning| Slack
    Alertmanager -->|Info| Email
    style Prom fill: #ff9999
    style ClickHouse fill: #cc99ff
    style Dashboard1 fill: #99ff99
    style PagerDuty fill: #ff6666
```