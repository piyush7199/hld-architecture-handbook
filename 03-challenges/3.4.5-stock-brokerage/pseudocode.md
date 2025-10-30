# Stock Brokerage Platform - Pseudocode Implementations

This document contains detailed algorithm implementations for the stock brokerage platform. The main challenge document
references these functions.

---

## Table of Contents

1. [Quote Handling](#quote-handling)
2. [Order Processing](#order-processing)
3. [WebSocket Management](#websocket-management)
4. [Search and Discovery](#search-and-discovery)
5. [Risk Management](#risk-management)
6. [FIX Protocol](#fix-protocol)
7. [Event Sourcing](#event-sourcing)
8. [Performance Optimization](#performance-optimization)

---

## Quote Handling

### update_quote()

**Purpose:** Update Redis cache with latest stock quote from exchange

**Parameters:**

- symbol (string): Stock ticker symbol (e.g., "AAPL")
- quote_data (object): Quote details {ltp, bid, ask, volume, timestamp}

**Returns:**

- boolean: Success status

**Algorithm:**

```
function update_quote(symbol, quote_data):
  // Validate input
  if not symbol or not quote_data:
    throw InvalidInputError("Symbol and quote_data required")
  
  // Build Redis key
  redis_key = "quote:" + symbol
  
  // Serialize quote data to JSON
  json_value = JSON.stringify({
    ltp: quote_data.ltp,
    bid: quote_data.bid,
    ask: quote_data.ask,
    volume: quote_data.volume,
    change: quote_data.change,
    change_percent: quote_data.change_percent,
    timestamp: quote_data.timestamp
  })
  
  try:
    // Write to Redis (no TTL - always keep latest)
    redis.SET(redis_key, json_value)
    
    // Update metrics
    metrics.increment("quotes_updated")
    
    return true
  catch RedisError as e:
    log.error("Failed to update quote", {symbol: symbol, error: e})
    metrics.increment("quotes_update_failed")
    return false
```

**Time Complexity:** O(1) - Redis SET operation

**Example Usage:**

```
quote_data = {
  ltp: 150.25,
  bid: 150.20,
  ask: 150.30,
  volume: 50000000,
  change: 1.50,
  change_percent: 1.01,
  timestamp: 1609459200000
}

success = update_quote("AAPL", quote_data)
if success:
  print("Quote updated successfully")
```

---

### update_quotes_batch()

**Purpose:** Batch update multiple quotes using Redis pipelining (20x faster than sequential)

**Parameters:**

- quotes (array): Array of {symbol, quote_data} objects

**Returns:**

- integer: Number of successfully updated quotes

**Algorithm:**

```
function update_quotes_batch(quotes):
  if not quotes or quotes.length == 0:
    return 0
  
  // Create Redis pipeline (batches commands)
  pipeline = redis.pipeline(transaction=false)
  
  // Queue all SET commands (no network calls yet)
  for each quote in quotes:
    redis_key = "quote:" + quote.symbol
    json_value = JSON.stringify(quote.quote_data)
    pipeline.SET(redis_key, json_value)
  
  try:
    // Execute all commands in single network round-trip
    results = pipeline.execute()
    
    // Count successes
    success_count = 0
    for each result in results:
      if result.status == "OK":
        success_count += 1
    
    metrics.increment("quotes_batch_updated", success_count)
    return success_count
    
  catch RedisError as e:
    log.error("Batch update failed", {error: e, batch_size: quotes.length})
    return 0
```

**Time Complexity:** O(n) where n = number of quotes, but only 1 network round-trip

**Performance Improvement:** 20x faster (100ms → 5ms for 100 quotes)

**Example Usage:**

```
quotes = [
  {symbol: "AAPL", quote_data: {ltp: 150.25, ...}},
  {symbol: "GOOGL", quote_data: {ltp: 2800.00, ...}},
  {symbol: "TSLA", quote_data: {ltp: 245.50, ...}}
]

updated_count = update_quotes_batch(quotes)
print(f"Updated {updated_count} out of {quotes.length} quotes")
```

---

### handle_market_data_update()

**Purpose:** Process market data message from Kafka and distribute to WebSocket subscribers

**Parameters:**

- kafka_message (object): Message from Kafka topic `market.quotes`

**Returns:**

- void

**Algorithm:**

```
function handle_market_data_update(kafka_message):
  // Parse message
  symbol = kafka_message.key  // Partition key
  quote_data = JSON.parse(kafka_message.value)
  
  // Update Redis cache (async, don't block)
  async_task(update_quote, symbol, quote_data)
  
  // Lookup subscribers on this WebSocket server
  subscribers = subscription_map.get(symbol, [])
  
  if subscribers.length == 0:
    // No subscribers on this server, skip
    return
  
  // Build WebSocket message
  ws_message = JSON.stringify({
    type: "quote_update",
    symbol: symbol,
    ltp: quote_data.ltp,
    change: quote_data.change,
    change_percent: quote_data.change_percent,
    timestamp: quote_data.timestamp
  })
  
  // Push to all subscribers (selective fanout)
  for each user_id in subscribers:
    ws_connection = websocket_connections.get(user_id)
    
    if ws_connection and ws_connection.is_open:
      try:
        ws_connection.send(ws_message)
        metrics.increment("quotes_pushed")
      catch WebSocketError as e:
        log.warning("Failed to push quote", {user_id: user_id, symbol: symbol})
        // Remove dead connection
        remove_subscription(user_id, symbol)
```

**Time Complexity:** O(n) where n = number of subscribers for this symbol on this server

**Example Usage:**

```
kafka_message = {
  key: "AAPL",
  value: '{"ltp": 150.30, "bid": 150.25, "ask": 150.35, ...}'
}

handle_market_data_update(kafka_message)
// Result: AAPL quote pushed to 2,000 subscribers on this server
```

---

## Order Processing

### place_order()

**Purpose:** Place a new order with ACID transaction (validate, deduct balance, insert order)

**Parameters:**

- user_id (integer): User placing the order
- symbol (string): Stock ticker
- side (enum): BUY or SELL
- quantity (integer): Number of shares
- price (decimal): Limit price (null for market order)
- order_type (enum): MARKET or LIMIT

**Returns:**

- object: {order_id, status, message}

**Algorithm:**

```
function place_order(user_id, symbol, side, quantity, price, order_type):
  // Input validation
  if quantity <= 0:
    return {error: "Quantity must be positive"}
  
  if order_type == LIMIT and price <= 0:
    return {error: "Limit price must be positive"}
  
  // Check market hours (9:15 AM - 3:30 PM IST)
  if not is_market_open():
    return {error: "Market is closed"}
  
  // Calculate order value
  if order_type == MARKET:
    // Use current market price for estimation
    current_quote = redis.GET("quote:" + symbol)
    estimated_price = current_quote.ltp
  else:
    estimated_price = price
  
  order_value = quantity * estimated_price
  
  // Start ACID transaction
  db.BEGIN_TRANSACTION()
  
  try:
    // Lock user's account row (prevents concurrent orders)
    account = db.query(
      "SELECT cash_balance, margin_used FROM accounts WHERE user_id = ? FOR UPDATE",
      [user_id]
    )
    
    if not account:
      db.ROLLBACK()
      return {error: "Account not found"}
    
    // Check sufficient balance
    available_balance = account.cash_balance - account.margin_used
    
    if side == BUY and available_balance < order_value:
      db.ROLLBACK()
      return {
        error: "INSUFFICIENT_BALANCE",
        required: order_value,
        available: available_balance,
        shortfall: order_value - available_balance
      }
    
    // Check position limits (max 25% in single stock)
    if not check_position_limit(user_id, symbol, order_value):
      db.ROLLBACK()
      return {error: "Position limit exceeded. Max 25% in single stock."}
    
    // Deduct balance (block margin for pending order)
    if side == BUY:
      db.execute(
        "UPDATE accounts SET margin_used = margin_used + ? WHERE user_id = ?",
        [order_value, user_id]
      )
    
    // Insert order record
    order_id = db.execute(
      "INSERT INTO orders (user_id, symbol, side, quantity, price, order_type, status, created_at) " +
      "VALUES (?, ?, ?, ?, ?, ?, 'PENDING', NOW()) RETURNING order_id",
      [user_id, symbol, side, quantity, price, order_type]
    )
    
    // Commit transaction (atomic)
    db.COMMIT()
    
    // Publish event to Kafka (async audit trail)
    kafka.publish("ledger.events", {
      type: "ORDER_PLACED",
      order_id: order_id,
      user_id: user_id,
      symbol: symbol,
      side: side,
      quantity: quantity,
      price: price,
      order_value: order_value,
      timestamp: now()
    })
    
    // Route order to exchange via FIX protocol
    async_task(route_order_to_exchange, order_id)
    
    return {
      order_id: order_id,
      status: "PENDING",
      message: "Order placed successfully. You will be notified when filled."
    }
    
  catch DatabaseError as e:
    db.ROLLBACK()
    log.error("Order placement failed", {user_id: user_id, error: e})
    return {error: "Order placement failed. Please try again."}
```

**Time Complexity:** O(1) - Database operations are indexed

**Example Usage:**

```
result = place_order(
  user_id=12345,
  symbol="RELIANCE",
  side=BUY,
  quantity=100,
  price=2450.00,
  order_type=LIMIT
)

if result.order_id:
  print(f"Order placed: {result.order_id}")
else:
  print(f"Error: {result.error}")
```

---

### async_place_order()

**Purpose:** Async wrapper for place_order (non-blocking API response)

**Parameters:**

- Same as place_order()

**Returns:**

- Immediate response with order_id, actual fill happens async

**Algorithm:**

```
function async_place_order(user_id, symbol, side, quantity, price, order_type):
  // Return immediately to user (don't wait for exchange fill)
  result = place_order(user_id, symbol, side, quantity, price, order_type)
  
  if result.order_id:
    // Success: Return order_id immediately
    return {
      order_id: result.order_id,
      status: "PENDING",
      estimated_time: "5-10 seconds",
      message: "Order placed. You'll receive notification when filled."
    }
  else:
    // Validation error: Return immediately
    return result

// Meanwhile (async, in background):
function route_order_to_exchange(order_id):
  // Fetch order details
  order = db.query("SELECT * FROM orders WHERE order_id = ?", [order_id])
  
  // Send to FIX engine
  fix_engine.send_order(
    order_id=order.order_id,
    symbol=order.symbol,
    side=order.side,
    quantity=order.quantity,
    price=order.price,
    order_type=order.order_type
  )
  
  // When exchange fills order (5-10 seconds later):
  // FIX engine publishes ORDER_FILLED event to Kafka
  // WebSocket server pushes notification to user
```

**Benefits:**

- ✅ User gets immediate feedback (<250ms)
- ✅ Server can handle 1,000 orders/sec (non-blocking)
- ✅ Graceful failure handling (no timeouts)

---

### check_position_limit()

**Purpose:** Validate user doesn't exceed 25% portfolio allocation to single stock

**Parameters:**

- user_id (integer): User ID
- symbol (string): Stock being purchased
- new_order_value (decimal): Value of new order

**Returns:**

- boolean: True if within limit, False if exceeds

**Algorithm:**

```
function check_position_limit(user_id, symbol, new_order_value):
  // Fetch current holdings
  holdings = db.query(
    "SELECT symbol, quantity, avg_price FROM holdings WHERE user_id = ?",
    [user_id]
  )
  
  // Calculate total portfolio value
  total_portfolio_value = 0
  current_position_value = 0
  
  for each holding in holdings:
    // Fetch current market price
    quote = redis.GET("quote:" + holding.symbol)
    current_price = quote.ltp
    
    holding_value = holding.quantity * current_price
    total_portfolio_value += holding_value
    
    // Track current position in this symbol
    if holding.symbol == symbol:
      current_position_value = holding_value
  
  // Add cash balance to total portfolio value
  account = db.query("SELECT cash_balance FROM accounts WHERE user_id = ?", [user_id])
  total_portfolio_value += account.cash_balance
  
  // Calculate new position value after order
  new_position_value = current_position_value + new_order_value
  
  // Check limit (25% max in single stock)
  max_allowed = total_portfolio_value * 0.25
  
  if new_position_value > max_allowed:
    log.warning("Position limit exceeded", {
      user_id: user_id,
      symbol: symbol,
      new_position_value: new_position_value,
      max_allowed: max_allowed,
      portfolio_value: total_portfolio_value
    })
    return false
  
  return true
```

**Time Complexity:** O(n) where n = number of holdings (typically <50)

**Example Usage:**

```
within_limit = check_position_limit(
  user_id=12345,
  symbol="RELIANCE",
  new_order_value=100000
)

if not within_limit:
  print("Cannot place order: Position limit exceeded (max 25% in single stock)")
```

---

## WebSocket Management

### handle_websocket_subscription()

**Purpose:** Handle user's subscribe message (add to subscription map)

**Parameters:**

- user_id (integer): User ID
- symbols (array): List of stock symbols to subscribe to
- websocket_connection (object): User's WebSocket connection

**Returns:**

- void

**Algorithm:**

```
function handle_websocket_subscription(user_id, symbols, websocket_connection):
  // Validate input
  if not symbols or symbols.length == 0:
    websocket_connection.send(JSON.stringify({
      type: "error",
      message: "No symbols provided"
    }))
    return
  
  // Limit subscriptions per user (max 50 symbols)
  if symbols.length > 50:
    symbols = symbols.slice(0, 50)
    websocket_connection.send(JSON.stringify({
      type: "warning",
      message: "Subscriptions limited to 50 symbols. Showing first 50."
    }))
  
  // Add user to subscription map for each symbol
  for each symbol in symbols:
    if not subscription_map.has(symbol):
      subscription_map.set(symbol, [])
    
    subscribers = subscription_map.get(symbol)
    
    // Add user if not already subscribed
    if user_id not in subscribers:
      subscribers.append(user_id)
  
  // Store WebSocket connection
  websocket_connections.set(user_id, websocket_connection)
  
  // Fetch initial snapshot (hydrate with current quotes)
  initial_quotes = []
  
  for each symbol in symbols:
    quote_data = redis.GET("quote:" + symbol)
    if quote_data:
      initial_quotes.append({
        symbol: symbol,
        ltp: quote_data.ltp,
        bid: quote_data.bid,
        ask: quote_data.ask,
        change: quote_data.change,
        volume: quote_data.volume
      })
  
  // Send initial snapshot to user
  websocket_connection.send(JSON.stringify({
    type: "snapshot",
    quotes: initial_quotes
  }))
  
  log.info("User subscribed", {
    user_id: user_id,
    symbols: symbols,
    server_id: SERVER_ID
  })
  
  metrics.increment("websocket_subscriptions", symbols.length)
```

**Time Complexity:** O(n) where n = number of symbols

**Example Usage:**

```
// User sends WebSocket message:
{
  "action": "subscribe",
  "symbols": ["AAPL", "GOOGL", "TSLA"]
}

// Server processes:
handle_websocket_subscription(
  user_id=12345,
  symbols=["AAPL", "GOOGL", "TSLA"],
  websocket_connection=ws_conn
)

// User receives:
{
  "type": "snapshot",
  "quotes": [
    {"symbol": "AAPL", "ltp": 150.25, ...},
    {"symbol": "GOOGL", "ltp": 2800.00, ...},
    {"symbol": "TSLA", "ltp": 245.50, ...}
  ]
}
```

---

### remove_subscription()

**Purpose:** Clean up subscription map when user disconnects

**Parameters:**

- user_id (integer): User ID
- symbol (string): Optional - remove specific symbol, or null for all

**Returns:**

- void

**Algorithm:**

```
function remove_subscription(user_id, symbol=null):
  if symbol:
    // Remove from specific symbol only
    if subscription_map.has(symbol):
      subscribers = subscription_map.get(symbol)
      subscribers.remove(user_id)
      
      // Clean up empty lists
      if subscribers.length == 0:
        subscription_map.delete(symbol)
  else:
    // Remove from ALL symbols (user disconnected)
    for each symbol in subscription_map.keys():
      subscribers = subscription_map.get(symbol)
      
      if user_id in subscribers:
        subscribers.remove(user_id)
        
        // Clean up empty lists
        if subscribers.length == 0:
          subscription_map.delete(symbol)
  
  // Remove WebSocket connection
  if not symbol:
    websocket_connections.delete(user_id)
  
  log.info("User unsubscribed", {user_id: user_id, symbol: symbol})
```

**Time Complexity:** O(n) where n = number of symbols in map (worst case, typically O(1))

**Example Usage:**

```
// User disconnects
on_websocket_close(user_id=12345):
  remove_subscription(user_id=12345, symbol=null)
  // Removes user from all subscription lists
```

---

## Search and Discovery

### search_and_hydrate()

**Purpose:** Search stocks by ticker/name and hydrate with live quotes

**Parameters:**

- query (string): Search query (e.g., "AP")
- limit (integer): Max results (default 10)

**Returns:**

- array: List of {symbol, name, ltp, change, ...}

**Algorithm:**

```
function search_and_hydrate(query, limit=10):
  // Input validation
  if not query or query.length < 2:
    return []
  
  // Check cache first (popular queries cached for 1 minute)
  cache_key = "search_cache:" + query
  cached_result = redis.GET(cache_key)
  
  if cached_result:
    metrics.increment("search_cache_hit")
    return cached_result
  
  // Cache miss - query Elasticsearch
  es_query = {
    "query": {
      "bool": {
        "should": [
          {"prefix": {"symbol": query}},  // "AP" → AAPL, APPL
          {"match": {"name": {"query": query, "fuzziness": "AUTO"}}}  // "Apple" → Apple Inc.
        ]
      }
    },
    "sort": [{"market_cap": "desc"}],  // Larger companies first
    "size": limit
  }
  
  es_results = elasticsearch.search(index="stocks", body=es_query)
  
  if es_results.length == 0:
    return []
  
  // Extract symbols
  symbols = []
  for each hit in es_results.hits:
    symbols.append(hit._source.symbol)
  
  // Hydrate with live quotes from Redis (batch fetch)
  redis_keys = []
  for each symbol in symbols:
    redis_keys.append("quote:" + symbol)
  
  quotes = redis.MGET(redis_keys)  // Batch fetch (fast)
  
  // Merge search results with quotes
  results = []
  for i = 0 to es_results.length - 1:
    result = {
      symbol: es_results[i].symbol,
      name: es_results[i].name,
      exchange: es_results[i].exchange,
      sector: es_results[i].sector,
      market_cap: es_results[i].market_cap
    }
    
    // Add live quote if available
    if quotes[i]:
      result.ltp = quotes[i].ltp
      result.change = quotes[i].change
      result.change_percent = quotes[i].change_percent
      result.volume = quotes[i].volume
    
    results.append(result)
  
  // Cache result for 1 minute
  redis.SETEX(cache_key, 60, results)
  
  metrics.increment("search_query")
  return results
```

**Time Complexity:** O(n) where n = number of results (typically 10)

**Example Usage:**

```
results = search_and_hydrate(query="AP", limit=10)

// Returns:
[
  {symbol: "AAPL", name: "Apple Inc.", ltp: 150.25, change: +1.5, ...},
  {symbol: "APPL", name: "AppLovin Corp.", ltp: 75.30, change: -0.8, ...},
  ...
]
```

---

## Risk Management

### check_margin_requirements()

**Purpose:** Monitor margin positions and trigger margin calls if equity drops below threshold

**Parameters:**

- user_id (integer): User to check

**Returns:**

- object: {equity_ratio, status, action_required}

**Algorithm:**

```
function check_margin_requirements(user_id):
  // Fetch user's holdings
  holdings = db.query(
    "SELECT symbol, quantity, avg_price FROM holdings WHERE user_id = ?",
    [user_id]
  )
  
  if holdings.length == 0:
    return {equity_ratio: 1.0, status: "OK", action_required: null}
  
  // Calculate current position value using live quotes
  position_value = 0
  
  for each holding in holdings:
    quote = redis.GET("quote:" + holding.symbol)
    current_price = quote.ltp
    position_value += holding.quantity * current_price
  
  // Fetch account details
  account = db.query(
    "SELECT cash_balance, margin_used FROM accounts WHERE user_id = ?",
    [user_id]
  )
  
  // Calculate equity
  debt = account.margin_used
  equity = position_value - debt
  equity_ratio = equity / position_value
  
  // Check thresholds
  INITIAL_MARGIN = 0.50  // 50% (user must have 50% equity when opening position)
  MAINTENANCE_MARGIN = 0.30  // 30% (triggers margin call)
  LIQUIDATION_MARGIN = 0.20  // 20% (auto-liquidate)
  
  if equity_ratio < LIQUIDATION_MARGIN:
    // Auto-liquidate immediately
    return {
      equity_ratio: equity_ratio,
      status: "CRITICAL",
      action_required: "LIQUIDATE",
      message: "Equity below 20%. Positions will be liquidated immediately."
    }
  
  else if equity_ratio < MAINTENANCE_MARGIN:
    // Trigger margin call
    margin_call_exists = db.query(
      "SELECT * FROM margin_calls WHERE user_id = ? AND status = 'ACTIVE'",
      [user_id]
    )
    
    if not margin_call_exists:
      // Create new margin call
      required_funds = (position_value * MAINTENANCE_MARGIN) - equity
      grace_period_end = now() + 1 hour
      
      db.execute(
        "INSERT INTO margin_calls (user_id, equity_ratio, threshold, required_funds, grace_period_end, status) " +
        "VALUES (?, ?, ?, ?, ?, 'ACTIVE')",
        [user_id, equity_ratio, MAINTENANCE_MARGIN, required_funds, grace_period_end]
      )
      
      // Send urgent notifications
      send_margin_call_alert(user_id, equity_ratio, required_funds)
    
    return {
      equity_ratio: equity_ratio,
      status: "WARNING",
      action_required: "ADD_FUNDS",
      message: f"Margin call! Add ${required_funds} within 1 hour or positions will be liquidated."
    }
  
  else:
    // Equity ratio OK
    return {
      equity_ratio: equity_ratio,
      status: "OK",
      action_required: null
    }
```

**Time Complexity:** O(n) where n = number of holdings

**Example Usage:**

```
// Run every 1 minute for all margin users
for each user_id in margin_users:
  result = check_margin_requirements(user_id)
  
  if result.action_required == "LIQUIDATE":
    auto_liquidate_positions(user_id)
  
  elif result.action_required == "ADD_FUNDS":
    print(f"Margin call for user {user_id}: {result.message}")
```

---

### detect_fraud()

**Purpose:** Detect suspicious trading patterns (wash trading, pump-and-dump)

**Parameters:**

- user_id (integer): User to analyze

**Returns:**

- array: List of detected fraud patterns

**Algorithm:**

```
function detect_fraud(user_id):
  fraud_patterns = []
  
  // Pattern 1: Wash Trading (buy/sell same stock >10 times in day)
  today_trades = db.query(
    "SELECT symbol, side, COUNT(*) as trade_count " +
    "FROM orders " +
    "WHERE user_id = ? AND status = 'FILLED' AND DATE(created_at) = CURRENT_DATE " +
    "GROUP BY symbol, side " +
    "HAVING COUNT(*) >= 10",
    [user_id]
  )
  
  for each trade in today_trades:
    // Check if user has both BUY and SELL (wash trading pattern)
    counterpart = today_trades.find(
      t => t.symbol == trade.symbol && t.side != trade.side && t.trade_count >= 10
    )
    
    if counterpart:
      fraud_patterns.append({
        type: "WASH_TRADING",
        symbol: trade.symbol,
        buy_count: trade.side == "BUY" ? trade.trade_count : counterpart.trade_count,
        sell_count: trade.side == "SELL" ? trade.trade_count : counterpart.trade_count,
        severity: "HIGH",
        action: "Block trades for 24 hours, report to exchange"
      })
  
  // Pattern 2: Pump and Dump (buy >5% of daily volume)
  large_trades = db.query(
    "SELECT symbol, SUM(quantity) as total_quantity " +
    "FROM orders " +
    "WHERE user_id = ? AND side = 'BUY' AND DATE(created_at) = CURRENT_DATE " +
    "GROUP BY symbol",
    [user_id]
  )
  
  for each trade in large_trades:
    quote = redis.GET("quote:" + trade.symbol)
    daily_volume = quote.volume
    
    user_volume_percent = (trade.total_quantity / daily_volume) * 100
    
    if user_volume_percent > 5:
      fraud_patterns.append({
        type: "PUMP_AND_DUMP",
        symbol: trade.symbol,
        user_volume: trade.total_quantity,
        daily_volume: daily_volume,
        percentage: user_volume_percent,
        severity: "CRITICAL",
        action: "Freeze account, manual review required"
      })
  
  // Pattern 3: Account Takeover (login from new country + immediate large withdrawal)
  recent_logins = db.query(
    "SELECT ip_address, country FROM login_audit " +
    "WHERE user_id = ? ORDER BY timestamp DESC LIMIT 10",
    [user_id]
  )
  
  if recent_logins.length >= 2:
    current_login = recent_logins[0]
    previous_logins = recent_logins.slice(1)
    
    // Check if current country is different from all previous
    country_mismatch = true
    for each prev_login in previous_logins:
      if prev_login.country == current_login.country:
        country_mismatch = false
        break
    
    if country_mismatch:
      // Check for recent large withdrawal
      recent_withdrawal = db.query(
        "SELECT amount FROM transactions " +
        "WHERE user_id = ? AND type = 'WITHDRAWAL' AND timestamp > NOW() - INTERVAL 1 HOUR " +
        "ORDER BY timestamp DESC LIMIT 1",
        [user_id]
      )
      
      if recent_withdrawal and recent_withdrawal.amount > 10000:
        fraud_patterns.append({
          type: "ACCOUNT_TAKEOVER",
          new_country: current_login.country,
          previous_country: previous_logins[0].country,
          withdrawal_amount: recent_withdrawal.amount,
          severity: "CRITICAL",
          action: "Block account immediately, require video KYC"
        })
  
  return fraud_patterns
```

**Time Complexity:** O(n) where n = number of trades today

**Example Usage:**

```
// Daily job: Check all active users
fraud_results = detect_fraud(user_id=12345)

if fraud_results.length > 0:
  for each pattern in fraud_results:
    log.critical("Fraud detected", {
      user_id: 12345,
      pattern: pattern.type,
      severity: pattern.severity
    })
    
    // Take action
    if pattern.severity == "CRITICAL":
      freeze_account(user_id=12345)
      notify_compliance_team(pattern)
```

---

## FIX Protocol

### send_fix_order()

**Purpose:** Encode and send FIX NewOrderSingle message to exchange

**Parameters:**

- order_id (integer): Internal order ID
- symbol (string): Stock ticker
- side (enum): BUY or SELL
- quantity (integer): Number of shares
- price (decimal): Limit price (null for market order)
- order_type (enum): MARKET or LIMIT

**Returns:**

- boolean: Success status

**Algorithm:**

```
function send_fix_order(order_id, symbol, side, quantity, price, order_type):
  // Increment sequence number
  session.last_sent_seqnum += 1
  
  // Build FIX message
  fix_message = create_fix_message()
  
  // Standard header
  fix_message.set(8, "FIX.4.2")  // BeginString
  fix_message.set(35, "D")  // MsgType: NewOrderSingle
  fix_message.set(49, SENDER_COMP_ID)  // SenderCompID: "BROKER_ID"
  fix_message.set(56, TARGET_COMP_ID)  // TargetCompID: "NSE"
  fix_message.set(34, session.last_sent_seqnum)  // MsgSeqNum
  fix_message.set(52, get_utc_timestamp())  // SendingTime: "20231101-10:30:00"
  
  // Order details
  fix_message.set(11, order_id)  // ClOrdID: Client Order ID
  fix_message.set(55, symbol)  // Symbol: "RELIANCE"
  fix_message.set(54, side == BUY ? 1 : 2)  // Side: 1=Buy, 2=Sell
  fix_message.set(38, quantity)  // OrderQty: 100
  
  if order_type == MARKET:
    fix_message.set(40, "1")  // OrdType: 1=Market
  else:
    fix_message.set(40, "2")  // OrdType: 2=Limit
    fix_message.set(44, price)  // Price: 2450.00
  
  fix_message.set(59, "0")  // TimeInForce: 0=Day
  
  // Calculate body length
  body_length = calculate_fix_body_length(fix_message)
  fix_message.set(9, body_length)  // BodyLength
  
  // Calculate checksum
  checksum = calculate_fix_checksum(fix_message)
  fix_message.set(10, checksum)  // CheckSum
  
  // Encode to FIX format (pipe-delimited)
  encoded_message = fix_message.encode()  // "8=FIX.4.2|9=120|35=D|..."
  
  try:
    // Send via TCP socket
    socket.send(encoded_message)
    
    // Persist sequence number (for recovery)
    db.execute(
      "UPDATE fix_sessions SET last_sent_seqnum = ? WHERE session_id = ?",
      [session.last_sent_seqnum, SESSION_ID]
    )
    
    log.info("FIX order sent", {
      order_id: order_id,
      symbol: symbol,
      seqnum: session.last_sent_seqnum
    })
    
    metrics.increment("fix_orders_sent")
    return true
    
  catch SocketError as e:
    log.error("Failed to send FIX order", {order_id: order_id, error: e})
    metrics.increment("fix_orders_failed")
    return false
```

**Time Complexity:** O(1)

**Example Usage:**

```
success = send_fix_order(
  order_id=999888,
  symbol="RELIANCE",
  side=BUY,
  quantity=100,
  price=2450.00,
  order_type=LIMIT
)

if success:
  print("Order sent to exchange via FIX protocol")
```

---

## Event Sourcing

### append_ledger_event()

**Purpose:** Append immutable event to Kafka ledger log

**Parameters:**

- event (object): Event details {type, user_id, amount, metadata}

**Returns:**

- boolean: Success status

**Algorithm:**

```
function append_ledger_event(event):
  // Validate event
  if not event.user_id or not event.type:
    throw InvalidEventError("user_id and type required")
  
  // Add timestamp and event ID
  event.event_id = generate_uuid()
  event.timestamp = now()
  
  // Serialize to JSON
  event_json = JSON.stringify(event)
  
  try:
    // Publish to Kafka (partitioned by user_id for ordering)
    kafka.produce(
      topic="ledger.events",
      key=event.user_id,  // Partition key (same user always same partition)
      value=event_json,
      callback=on_delivery
    )
    
    metrics.increment("ledger_events_appended")
    return true
    
  catch KafkaError as e:
    log.error("Failed to append event", {event: event, error: e})
    metrics.increment("ledger_events_failed")
    return false

function on_delivery(err, msg):
  if err:
    log.error("Kafka delivery failed", {error: err})
  else:
    log.debug("Event delivered", {partition: msg.partition, offset: msg.offset})
```

**Time Complexity:** O(1) - Kafka append is lock-free

**Example Usage:**

```
event = {
  type: "ORDER_FILLED",
  user_id: 12345,
  order_id: 999888,
  symbol: "RELIANCE",
  side: "BUY",
  quantity: 100,
  fill_price: 2450.00,
  amount: -245000  // Negative = deduction
}

append_ledger_event(event)
```

---

## Performance Optimization

### execute_vwap_order()

**Purpose:** Execute large order using VWAP (Volume Weighted Average Price) algorithm

**Parameters:**

- user_id (integer): User placing order
- symbol (string): Stock ticker
- total_quantity (integer): Total shares to buy/sell
- start_time (timestamp): Start of execution window
- end_time (timestamp): End of execution window

**Returns:**

- object: {avg_fill_price, total_quantity_filled, vwap_achieved}

**Algorithm:**

```
function execute_vwap_order(user_id, symbol, total_quantity, start_time, end_time):
  time_window = end_time - start_time  // e.g., 2 hours
  interval = 5 minutes
  num_intervals = time_window / interval  // e.g., 24 intervals
  
  filled_quantity = 0
  total_cost = 0
  
  // Get expected daily volume (historical average)
  historical_data = get_historical_volume(symbol, days=30)
  expected_daily_volume = average(historical_data)
  
  // Execute in intervals
  for t = start_time to end_time step interval:
    current_time = now()
    
    if current_time < t:
      sleep(t - current_time)
    
    // Calculate elapsed time percentage
    elapsed_time = current_time - start_time
    elapsed_pct = elapsed_time / time_window
    
    // Get current market volume
    quote = redis.GET("quote:" + symbol)
    current_volume = quote.volume
    
    // Calculate volume percentage traded so far today
    volume_pct = current_volume / expected_daily_volume
    
    // Target: Match market's volume distribution
    target_quantity = total_quantity * volume_pct
    quantity_to_place = target_quantity - filled_quantity
    
    // Place market order for this interval
    if quantity_to_place > 0:
      result = place_order(
        user_id=user_id,
        symbol=symbol,
        side=BUY,
        quantity=quantity_to_place,
        price=null,
        order_type=MARKET
      )
      
      // Wait for fill (typically <1 second)
      fill_price = wait_for_fill(result.order_id, timeout=10)
      
      if fill_price:
        filled_quantity += quantity_to_place
        total_cost += quantity_to_place * fill_price
      else:
        log.warning("Order not filled within timeout", {
          order_id: result.order_id,
          quantity: quantity_to_place
        })
  
  // Calculate achieved VWAP
  avg_fill_price = total_cost / filled_quantity
  
  // Calculate market VWAP for comparison
  market_vwap = calculate_market_vwap(symbol, start_time, end_time)
  
  vwap_achieved = avg_fill_price / market_vwap  // Ideally ~1.0
  
  return {
    avg_fill_price: avg_fill_price,
    total_quantity_filled: filled_quantity,
    vwap_achieved: vwap_achieved,
    slippage: avg_fill_price - market_vwap
  }
```

**Time Complexity:** O(n) where n = number of intervals

**Example Usage:**

```
result = execute_vwap_order(
  user_id=12345,
  symbol="RELIANCE",
  total_quantity=10000,
  start_time=timestamp("2023-11-01 09:15:00"),
  end_time=timestamp("2023-11-01 11:15:00")
)

print(f"Executed {result.total_quantity_filled} shares at avg price ${result.avg_fill_price}")
print(f"VWAP achieved: {result.vwap_achieved * 100}% (target: 100%)")
```
