# Stock Exchange Matching Engine - Pseudocode Implementations

This document contains detailed algorithm implementations for the stock exchange matching engine system. The main challenge document references these functions.

---

## Table of Contents

1. [Order Book Operations](#1-order-book-operations)
2. [Matching Algorithm](#2-matching-algorithm)
3. [Ring Buffer (Lock-Free Queue)](#3-ring-buffer-lock-free-queue)
4. [Audit Log Management](#4-audit-log-management)
5. [Red-Black Tree Operations](#5-red-black-tree-operations)
6. [Price-Time Priority Queue](#6-price-time-priority-queue)
7. [Order Validation](#7-order-validation)
8. [Market Data Publishing](#8-market-data-publishing)
9. [Crash Recovery](#9-crash-recovery)
10. [Object Pool Memory Management](#10-object-pool-memory-management)

---

## 1. Order Book Operations

### insert_order()

**Purpose:** Insert a new limit order into the order book at the specified price level, maintaining price-time priority.

**Parameters:**
- order_book (OrderBook): The order book instance
- order (Order): The order to insert

**Returns:**
- bool: true if inserted successfully, false otherwise

**Algorithm:**

```
function insert_order(order_book, order):
  // Determine side (bid or ask)
  if order.side == BUY:
    price_tree = order_book.bids
    sort_order = DESCENDING  // Highest price first
  else:
    price_tree = order_book.asks
    sort_order = ASCENDING   // Lowest price first
  
  // Find or create price level
  price_level = price_tree.find(order.price)
  
  if price_level == null:
    // Create new price level
    price_level = PriceLevel()
    price_level.price = order.price
    price_level.orders = LinkedList()
    price_level.total_quantity = 0
    
    // Insert into Red-Black Tree (O(log N))
    price_tree.insert(order.price, price_level)
    
    // Update best bid/ask cache if necessary
    if order.side == BUY and order.price > order_book.best_bid:
      order_book.best_bid = order.price
    else if order.side == SELL and order.price < order_book.best_ask:
      order_book.best_ask = order.price
  
  // Append to end of linked list (FIFO time priority)
  price_level.orders.append(order)
  price_level.total_quantity += order.quantity
  
  // Add to hash map for fast cancellation lookup
  order_book.orders[order.order_id] = order
  
  return true
```

**Time Complexity:** O(log N) for tree insert + O(1) for linked list append = O(log N)

**Example Usage:**

```
order = Order(
  order_id = 12345,
  symbol = "AAPL",
  side = BUY,
  price = 10050,  // $100.50 in cents
  quantity = 100
)

success = insert_order(order_book, order)
```

---

### remove_order()

**Purpose:** Remove an order from the order book (for execution or cancellation).

**Parameters:**
- order_book (OrderBook): The order book instance
- order_id (u64): The ID of the order to remove

**Returns:**
- Order: The removed order, or null if not found

**Algorithm:**

```
function remove_order(order_book, order_id):
  // Lookup order in hash map (O(1))
  order = order_book.orders[order_id]
  if order == null:
    return null
  
  // Determine side
  if order.side == BUY:
    price_tree = order_book.bids
  else:
    price_tree = order_book.asks
  
  // Find price level
  price_level = price_tree.find(order.price)
  if price_level == null:
    return null  // Inconsistent state (should not happen)
  
  // Remove from linked list (O(1) if we have pointer)
  price_level.orders.remove(order)
  price_level.total_quantity -= order.quantity
  
  // If price level is now empty, remove from tree
  if price_level.orders.is_empty():
    price_tree.delete(order.price)
    
    // Update best bid/ask cache if necessary
    if order.side == BUY and order.price == order_book.best_bid:
      // Recalculate best bid (O(1) from tree)
      order_book.best_bid = price_tree.max_key()
    else if order.side == SELL and order.price == order_book.best_ask:
      // Recalculate best ask (O(1) from tree)
      order_book.best_ask = price_tree.min_key()
  
  // Remove from hash map
  delete order_book.orders[order_id]
  
  return order
```

**Time Complexity:** O(log N) for tree delete (if price level empty) + O(1) for linked list removal = O(log N)

**Example Usage:**

```
removed_order = remove_order(order_book, 12345)
if removed_order != null:
  print("Order removed:", removed_order.order_id)
```

---

## 2. Matching Algorithm

### match_order()

**Purpose:** Match an incoming order against the opposite side of the order book, generating execution events.

**Parameters:**
- order_book (OrderBook): The order book instance
- incoming_order (Order): The order to match

**Returns:**
- executions (List<Execution>): List of execution events generated

**Algorithm:**

```
function match_order(order_book, incoming_order):
  executions = []
  remaining_quantity = incoming_order.quantity
  
  // Determine opposite side
  if incoming_order.side == BUY:
    opposite_tree = order_book.asks  // Match against sells
    is_better_price = lambda resting, incoming: resting.price <= incoming.price
  else:
    opposite_tree = order_book.bids  // Match against buys
    is_better_price = lambda resting, incoming: resting.price >= incoming.price
  
  // Match loop: Walk the book until filled or no more matches
  while remaining_quantity > 0:
    // Get best opposite price (O(1) cached)
    if incoming_order.side == BUY:
      best_price = order_book.best_ask
    else:
      best_price = order_book.best_bid
    
    // Check if match is possible
    if best_price == null:
      break  // No more orders on opposite side
    
    if incoming_order.order_type == LIMIT:
      // Limit order: Only match if price crosses
      if not is_better_price(best_price, incoming_order.price):
        break  // No match possible
    
    // Get price level at best price
    price_level = opposite_tree.find(best_price)
    
    // Match against orders at this price level (FIFO)
    while remaining_quantity > 0 and not price_level.orders.is_empty():
      resting_order = price_level.orders.front()
      
      // Calculate execution quantity
      exec_quantity = min(remaining_quantity, resting_order.quantity)
      exec_price = resting_order.price  // Resting order's price (price priority)
      
      // Generate execution event
      execution = Execution(
        buyer_order_id = (incoming_order.side == BUY) ? incoming_order.order_id : resting_order.order_id,
        seller_order_id = (incoming_order.side == SELL) ? incoming_order.order_id : resting_order.order_id,
        price = exec_price,
        quantity = exec_quantity,
        timestamp = get_nanosecond_timestamp()
      )
      executions.append(execution)
      
      // Update quantities
      remaining_quantity -= exec_quantity
      resting_order.quantity -= exec_quantity
      resting_order.filled_quantity += exec_quantity
      
      incoming_order.filled_quantity += exec_quantity
      
      // Update price level total
      price_level.total_quantity -= exec_quantity
      
      // Remove resting order if fully filled
      if resting_order.quantity == 0:
        resting_order.status = FILLED
        price_level.orders.pop_front()
        delete order_book.orders[resting_order.order_id]
      else:
        resting_order.status = PARTIAL_FILL
    
    // If price level is now empty, remove from tree
    if price_level.orders.is_empty():
      opposite_tree.delete(best_price)
      
      // Update best bid/ask cache
      if incoming_order.side == BUY:
        order_book.best_ask = opposite_tree.min_key()
      else:
        order_book.best_bid = opposite_tree.max_key()
  
  // Update incoming order status
  if remaining_quantity == 0:
    incoming_order.status = FILLED
  else if incoming_order.filled_quantity > 0:
    incoming_order.status = PARTIAL_FILL
  else:
    incoming_order.status = PENDING
  
  // If incoming order is LIMIT and not fully filled, insert remainder
  if incoming_order.order_type == LIMIT and remaining_quantity > 0:
    incoming_order.quantity = remaining_quantity
    insert_order(order_book, incoming_order)
  
  // If incoming order is IOC and not fully filled, cancel remainder
  if incoming_order.time_in_force == IOC and remaining_quantity > 0:
    incoming_order.status = CANCELLED
    incoming_order.quantity = 0
  
  return executions
```

**Time Complexity:** O(K × log N) where K = number of orders matched, N = order book depth

**Example Usage:**

```
incoming_order = Order(
  order_id = 67890,
  symbol = "AAPL",
  side = SELL,
  order_type = MARKET,
  quantity = 500
)

executions = match_order(order_book, incoming_order)
for exec in executions:
  print("Execution:", exec.quantity, "@", exec.price)
```

---

## 3. Ring Buffer (Lock-Free Queue)

### ring_buffer_enqueue()

**Purpose:** Enqueue an order to the lock-free ring buffer for inter-process communication (gateway → matching engine).

**Parameters:**
- ring_buffer (RingBuffer): The ring buffer instance
- order (Order): The order to enqueue

**Returns:**
- bool: true if enqueued successfully, false if buffer is full

**Algorithm:**

```
function ring_buffer_enqueue(ring_buffer, order):
  // Load current write index (atomic)
  write_idx = atomic_fetch_add(ring_buffer.write_index, 1, memory_order_relaxed)
  
  // Calculate slot index (fast modulo using bitwise AND)
  // Assumes ring_buffer.size is power of 2 (e.g., 1M = 2^20)
  slot_idx = write_idx & (ring_buffer.size - 1)
  
  // Check if buffer is full (write caught up to read)
  read_idx = atomic_load(ring_buffer.read_index, memory_order_acquire)
  if write_idx - read_idx >= ring_buffer.size:
    // Buffer full! (backpressure)
    return false
  
  // Copy order to slot (no lock needed)
  ring_buffer.slots[slot_idx] = order
  
  return true
```

**Time Complexity:** O(1)

**Example Usage:**

```
ring_buffer = RingBuffer(size = 1000000)  // 1M slots

order = Order(order_id = 12345, ...)
success = ring_buffer_enqueue(ring_buffer, order)
if not success:
  // Handle backpressure (drop order or reject client)
  print("Ring buffer full! Dropping order.")
```

---

### ring_buffer_dequeue()

**Purpose:** Dequeue an order from the lock-free ring buffer (matching engine side).

**Parameters:**
- ring_buffer (RingBuffer): The ring buffer instance

**Returns:**
- Order: The dequeued order, or null if buffer is empty

**Algorithm:**

```
function ring_buffer_dequeue(ring_buffer):
  // Load current read index (atomic)
  read_idx = atomic_load(ring_buffer.read_index, memory_order_relaxed)
  
  // Check if buffer is empty (read caught up to write)
  write_idx = atomic_load(ring_buffer.write_index, memory_order_acquire)
  if read_idx == write_idx:
    return null  // Empty
  
  // Calculate slot index
  slot_idx = read_idx & (ring_buffer.size - 1)
  
  // Copy order from slot
  order = ring_buffer.slots[slot_idx]
  
  // Increment read index (atomic)
  atomic_store(ring_buffer.read_index, read_idx + 1, memory_order_release)
  
  return order
```

**Time Complexity:** O(1)

**Example Usage:**

```
while true:
  order = ring_buffer_dequeue(ring_buffer)
  if order == null:
    break  // No more orders
  
  // Process order
  executions = match_order(order_book, order)
```

---

## 4. Audit Log Management

### write_audit_log()

**Purpose:** Asynchronously write a log entry to the audit log (Write-Ahead Log for durability).

**Parameters:**
- audit_log (AuditLog): The audit log instance
- event_type (u8): 0=ORDER, 1=EXECUTION, 2=CANCEL
- payload (bytes): Serialized event data

**Returns:**
- bool: true if enqueued successfully

**Algorithm:**

```
function write_audit_log(audit_log, event_type, payload):
  // Create log entry
  log_entry = LogEntry()
  log_entry.sequence_number = atomic_fetch_add(audit_log.sequence, 1, memory_order_relaxed)
  log_entry.timestamp = get_nanosecond_timestamp()
  log_entry.event_type = event_type
  log_entry.payload = payload
  log_entry.checksum = crc32(payload)  // Integrity check
  
  // Enqueue to async ring buffer (non-blocking)
  success = ring_buffer_enqueue(audit_log.log_queue, log_entry)
  
  if not success:
    // Log queue full! (critical error)
    trigger_alert("Audit log queue full!")
    return false
  
  return true
```

**Time Complexity:** O(1) (non-blocking async write)

**Example Usage:**

```
// Log order received
order_payload = serialize(order)
write_audit_log(audit_log, event_type=ORDER, payload=order_payload)

// Log execution
exec_payload = serialize(execution)
write_audit_log(audit_log, event_type=EXECUTION, payload=exec_payload)
```

---

### audit_log_background_thread()

**Purpose:** Background thread that consumes the audit log queue and writes batches to disk.

**Parameters:**
- audit_log (AuditLog): The audit log instance

**Returns:**
- None (runs indefinitely)

**Algorithm:**

```
function audit_log_background_thread(audit_log):
  batch_buffer = []
  batch_size = 1000  // Batch 1000 entries
  
  while true:
    // Dequeue log entries
    while batch_buffer.length < batch_size:
      log_entry = ring_buffer_dequeue(audit_log.log_queue)
      if log_entry == null:
        break  // No more entries
      batch_buffer.append(log_entry)
    
    // If batch is empty, sleep briefly
    if batch_buffer.is_empty():
      sleep_microseconds(100)  // Poll every 100μs
      continue
    
    // Serialize batch (1000 entries × 272 bytes = 272 KB)
    batch_data = serialize_batch(batch_buffer)
    
    // Write to NVMe SSD (single syscall)
    write_to_disk(audit_log.file_descriptor, batch_data)
    
    // Flush to disk (fsync)
    fsync(audit_log.file_descriptor)
    
    // Clear batch buffer
    batch_buffer.clear()
```

**Time Complexity:** O(N) where N = batch size (1000 entries)

**Example Usage:**

```
// Start background thread at initialization
spawn_thread(audit_log_background_thread, audit_log)
```

---

## 5. Red-Black Tree Operations

### rb_tree_insert()

**Purpose:** Insert a key-value pair into the Red-Black Tree, maintaining balance.

**Parameters:**
- tree (RBTree): The Red-Black Tree instance
- key (Price): The price level
- value (PriceLevel): The price level data

**Returns:**
- Node: The inserted node

**Algorithm:**

```
function rb_tree_insert(tree, key, value):
  // Standard BST insert
  new_node = Node(key, value, color=RED)
  
  if tree.root == null:
    tree.root = new_node
    new_node.color = BLACK  // Root is always black
    return new_node
  
  // Find insertion point
  current = tree.root
  parent = null
  
  while current != null:
    parent = current
    if key < current.key:
      current = current.left
    else if key > current.key:
      current = current.right
    else:
      // Key already exists (update value)
      current.value = value
      return current
  
  // Insert as child of parent
  new_node.parent = parent
  if key < parent.key:
    parent.left = new_node
  else:
    parent.right = new_node
  
  // Fix Red-Black Tree properties
  rb_tree_fix_insert(tree, new_node)
  
  return new_node
```

**Time Complexity:** O(log N)

**Example Usage:**

```
tree = RBTree()
price_level = PriceLevel(price=10050, orders=[...])
rb_tree_insert(tree, key=10050, value=price_level)
```

---

### rb_tree_fix_insert()

**Purpose:** Fix Red-Black Tree violations after insertion (recoloring and rotations).

**Parameters:**
- tree (RBTree): The Red-Black Tree instance
- node (Node): The newly inserted node

**Returns:**
- None (modifies tree in-place)

**Algorithm:**

```
function rb_tree_fix_insert(tree, node):
  // Fix violations (red parent with red child)
  while node.parent != null and node.parent.color == RED:
    grandparent = node.parent.parent
    
    if node.parent == grandparent.left:
      uncle = grandparent.right
      
      if uncle != null and uncle.color == RED:
        // Case 1: Uncle is red (recolor)
        node.parent.color = BLACK
        uncle.color = BLACK
        grandparent.color = RED
        node = grandparent  // Move up
      else:
        // Case 2: Uncle is black
        if node == node.parent.right:
          // Left-Right case (rotate left)
          node = node.parent
          rb_tree_rotate_left(tree, node)
        
        // Left-Left case (rotate right)
        node.parent.color = BLACK
        grandparent.color = RED
        rb_tree_rotate_right(tree, grandparent)
    else:
      // Mirror case (parent is right child)
      uncle = grandparent.left
      
      if uncle != null and uncle.color == RED:
        node.parent.color = BLACK
        uncle.color = BLACK
        grandparent.color = RED
        node = grandparent
      else:
        if node == node.parent.left:
          node = node.parent
          rb_tree_rotate_right(tree, node)
        
        node.parent.color = BLACK
        grandparent.color = RED
        rb_tree_rotate_left(tree, grandparent)
  
  // Ensure root is black
  tree.root.color = BLACK
```

**Time Complexity:** O(log N)

---

## 6. Price-Time Priority Queue

### get_best_bid()

**Purpose:** Get the best bid price (highest buy price) in O(1) time.

**Parameters:**
- order_book (OrderBook): The order book instance

**Returns:**
- Price: The best bid price, or null if no bids

**Algorithm:**

```
function get_best_bid(order_book):
  // Best bid is cached at order book level
  return order_book.best_bid
```

**Time Complexity:** O(1)

---

### get_best_ask()

**Purpose:** Get the best ask price (lowest sell price) in O(1) time.

**Parameters:**
- order_book (OrderBook): The order book instance

**Returns:**
- Price: The best ask price, or null if no asks

**Algorithm:**

```
function get_best_ask(order_book):
  // Best ask is cached at order book level
  return order_book.best_ask
```

**Time Complexity:** O(1)

---

## 7. Order Validation

### validate_order()

**Purpose:** Validate an incoming order before processing (price limits, quantity limits, balance check).

**Parameters:**
- order (Order): The order to validate
- user_account (Account): The user's account

**Returns:**
- result (ValidationResult): success=true/false, error_message

**Algorithm:**

```
function validate_order(order, user_account):
  // Check order ID uniqueness
  if order.order_id in global_order_ids:
    return ValidationResult(success=false, error="Duplicate order ID")
  
  // Check symbol validity
  if order.symbol not in valid_symbols:
    return ValidationResult(success=false, error="Invalid symbol")
  
  // Check price collar (prevent fat-finger errors)
  reference_price = get_reference_price(order.symbol)
  max_price = reference_price * 1.10  // 10% collar
  min_price = reference_price * 0.90
  
  if order.order_type == LIMIT:
    if order.price > max_price or order.price < min_price:
      return ValidationResult(success=false, error="Price outside collar")
  
  // Check quantity limits
  min_quantity = 1
  max_quantity = 1000000  // 1M shares max
  
  if order.quantity < min_quantity or order.quantity > max_quantity:
    return ValidationResult(success=false, error="Invalid quantity")
  
  // Check account balance (buying power)
  if order.side == BUY:
    required_balance = order.price * order.quantity  // In cents
    if user_account.balance < required_balance:
      return ValidationResult(success=false, error="Insufficient balance")
  
  // Check account has shares (for sell orders)
  if order.side == SELL:
    if user_account.holdings[order.symbol] < order.quantity:
      return ValidationResult(success=false, error="Insufficient shares")
  
  // All checks passed
  return ValidationResult(success=true)
```

**Time Complexity:** O(1)

**Example Usage:**

```
order = Order(order_id=12345, ...)
user_account = get_account(user_id=67890)

result = validate_order(order, user_account)
if not result.success:
  reject_order(order, reason=result.error_message)
  return
```

---

## 8. Market Data Publishing

### publish_market_data()

**Purpose:** Publish market data update to Kafka for distribution to subscribers.

**Parameters:**
- kafka_producer (KafkaProducer): The Kafka producer instance
- symbol (string): The stock symbol
- event_type (string): "TRADE" or "TOP_OF_BOOK"
- data (object): The event data

**Returns:**
- bool: true if published successfully

**Algorithm:**

```
function publish_market_data(kafka_producer, symbol, event_type, data):
  // Create market data message
  message = MarketDataMessage()
  message.symbol = symbol
  message.event_type = event_type
  message.timestamp = get_nanosecond_timestamp()
  
  if event_type == "TRADE":
    message.price = data.price
    message.quantity = data.quantity
    message.buyer_order_id = data.buyer_order_id
    message.seller_order_id = data.seller_order_id
  else if event_type == "TOP_OF_BOOK":
    message.best_bid = data.best_bid
    message.best_bid_size = data.best_bid_size
    message.best_ask = data.best_ask
    message.best_ask_size = data.best_ask_size
  
  // Serialize to Protocol Buffers (compact format)
  payload = protobuf_serialize(message)
  
  // Publish to Kafka topic (partitioned by symbol)
  kafka_key = symbol  // Ensures all AAPL messages go to same partition
  kafka_topic = "market-data-feed"
  
  success = kafka_producer.send(topic=kafka_topic, key=kafka_key, value=payload)
  
  return success
```

**Time Complexity:** O(1) (async, non-blocking)

**Example Usage:**

```
// After execution, publish trade
for execution in executions:
  publish_market_data(
    kafka_producer,
    symbol = "AAPL",
    event_type = "TRADE",
    data = execution
  )

// After order book update, publish top-of-book
if best_bid_changed or best_ask_changed:
  publish_market_data(
    kafka_producer,
    symbol = "AAPL",
    event_type = "TOP_OF_BOOK",
    data = {
      best_bid: order_book.best_bid,
      best_bid_size: get_price_level_quantity(order_book.bids, order_book.best_bid),
      best_ask: order_book.best_ask,
      best_ask_size: get_price_level_quantity(order_book.asks, order_book.best_ask)
    }
  )
```

---

## 9. Crash Recovery

### replay_audit_log()

**Purpose:** Replay the audit log to rebuild the order book state after a crash.

**Parameters:**
- audit_log_file (string): Path to the audit log file
- order_book (OrderBook): The order book to rebuild

**Returns:**
- bool: true if replay successful

**Algorithm:**

```
function replay_audit_log(audit_log_file, order_book):
  // Open audit log file (sequential read)
  file = open(audit_log_file, mode=READ_SEQUENTIAL)
  
  entries_processed = 0
  start_time = get_timestamp()
  
  while not file.eof():
    // Read log entry (272 bytes)
    log_entry = read_log_entry(file)
    
    // Verify checksum (integrity)
    if crc32(log_entry.payload) != log_entry.checksum:
      print("Corrupted log entry at sequence:", log_entry.sequence_number)
      continue  // Skip corrupted entry
    
    // Deserialize payload
    event_type = log_entry.event_type
    
    if event_type == ORDER:
      order = deserialize_order(log_entry.payload)
      
      // Replay order (insert into order book)
      if order.status == PENDING or order.status == PARTIAL_FILL:
        insert_order(order_book, order)
    
    else if event_type == EXECUTION:
      execution = deserialize_execution(log_entry.payload)
      
      // Update order quantities (execution already happened)
      buyer_order = order_book.orders[execution.buyer_order_id]
      seller_order = order_book.orders[execution.seller_order_id]
      
      if buyer_order != null:
        buyer_order.filled_quantity += execution.quantity
        buyer_order.quantity -= execution.quantity
      
      if seller_order != null:
        seller_order.filled_quantity += execution.quantity
        seller_order.quantity -= execution.quantity
    
    else if event_type == CANCEL:
      cancel_event = deserialize_cancel(log_entry.payload)
      
      // Remove order from order book
      remove_order(order_book, cancel_event.order_id)
    
    entries_processed += 1
  
  // Close file
  file.close()
  
  // Log replay summary
  end_time = get_timestamp()
  elapsed_time = end_time - start_time
  
  print("Replay complete:")
  print("  Entries processed:", entries_processed)
  print("  Time elapsed:", elapsed_time, "ms")
  print("  Orders in book:", order_book.orders.size())
  
  return true
```

**Time Complexity:** O(N) where N = number of log entries

**Example Usage:**

```
// After crash, rebuild order book
order_book = OrderBook()
success = replay_audit_log("/var/log/audit.log", order_book)

if success:
  print("Order book rebuilt, resuming trading")
  start_matching_engine()
```

---

## 10. Object Pool Memory Management

### object_pool_allocate()

**Purpose:** Allocate an Order object from the pre-allocated object pool (avoid malloc overhead).

**Parameters:**
- pool (ObjectPool): The object pool instance

**Returns:**
- Order: A pointer to the allocated Order object, or null if pool is empty

**Algorithm:**

```
function object_pool_allocate(pool):
  // Check if free list is empty
  if pool.free_list == null:
    print("Object pool exhausted!")
    return null
  
  // Pop from free list (O(1))
  order = pool.free_list
  pool.free_list = order.next  // Move to next free object
  
  // Reset order fields (clear previous data)
  order.order_id = 0
  order.user_id = 0
  order.symbol = 0
  order.side = 0
  order.order_type = 0
  order.price = 0
  order.quantity = 0
  order.filled_quantity = 0
  order.timestamp = 0
  order.time_in_force = 0
  order.status = 0
  order.next = null
  
  pool.allocated_count += 1
  
  return order
```

**Time Complexity:** O(1)

**Example Usage:**

```
// Pre-allocate pool at startup
pool = ObjectPool(capacity=10000000)  // 10M orders

// Allocate order
order = object_pool_allocate(pool)
if order == null:
  // Handle exhaustion (reject order or expand pool)
  return
```

---

### object_pool_deallocate()

**Purpose:** Deallocate an Order object back to the object pool (return to free list).

**Parameters:**
- pool (ObjectPool): The object pool instance
- order (Order): The order to deallocate

**Returns:**
- None

**Algorithm:**

```
function object_pool_deallocate(pool, order):
  // Push to free list (O(1))
  order.next = pool.free_list
  pool.free_list = order
  
  pool.allocated_count -= 1
```

**Time Complexity:** O(1)

**Example Usage:**

```
// After order is fully filled or cancelled, deallocate
object_pool_deallocate(pool, order)
```

---

### object_pool_init()

**Purpose:** Initialize the object pool by pre-allocating all Order objects at startup.

**Parameters:**
- capacity (u32): The number of orders to pre-allocate

**Returns:**
- ObjectPool: The initialized object pool

**Algorithm:**

```
function object_pool_init(capacity):
  pool = ObjectPool()
  pool.capacity = capacity
  pool.allocated_count = 0
  
  // Allocate contiguous array (cache-friendly)
  pool.pool = allocate_array(Order, capacity)
  
  // Build free list (link all objects)
  for i in 0 to capacity - 1:
    order = &pool.pool[i]
    
    // Cache line alignment (64 bytes)
    assert((u64)order % 64 == 0, "Order not cache-aligned")
    
    // Link to next object
    if i < capacity - 1:
      order.next = &pool.pool[i + 1]
    else:
      order.next = null  // Last object
  
  // Set free list head
  pool.free_list = &pool.pool[0]
  
  print("Object pool initialized:")
  print("  Capacity:", capacity, "orders")
  print("  Memory:", capacity * sizeof(Order), "bytes")
  
  return pool
```

**Time Complexity:** O(N) where N = capacity (only run once at startup)

**Example Usage:**

```
// Initialize at matching engine startup
pool = object_pool_init(capacity=10000000)  // 10M orders
```

