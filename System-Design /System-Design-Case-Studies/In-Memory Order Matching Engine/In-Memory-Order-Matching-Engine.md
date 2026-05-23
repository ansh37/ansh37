# System Design Deep Dive: In-Memory Order Matching Engine (Crypto / HFT Scale)

![Level](https://img.shields.io/badge/Level-Staff%20%2F%20L5-blue)
![Topic](https://img.shields.io/badge/Topic-System%20Design%20%7C%20HFT-success)

## 1. Problem Statement & Clarifications
Design a high-frequency Order Matching Engine capable of powering a cryptocurrency exchange like CoinDCX or Binance. The engine must accept limit and market orders, match them based on strict Price-Time priority, allow partial fills, and guarantee zero data loss in the event of catastrophic hardware failure.

**Critical Distinction:** This is a pure matching engine. It operates completely off-chain. Transactions here update internal ledger balances, not the cryptographic blockchain. 

## 2. Requirements

### Functional Requirements (FRs)
* **Order Placement:** Users can place Market Orders and Limit Orders.
* **Order Cancellation:** Users can cancel open orders in O(1) time.
* **Order Matching:** Continuous matching of Bids and Asks based on Price-Time Priority.
* **Market Data:** Emit real-time Order Book depth and trade execution events to downstream consumers.

### Non-Functional Requirements (NFRs)
* **Extreme Low Latency:** The critical path (Ingest -> Sequence -> Match -> Acknowledge) must be sub-millisecond (targeting < 50 microseconds).
* **Strict Determinism:** Orders must be processed exactly in the order received. No race conditions.
* **Absolute Durability:** 100% data durability. RAM is volatile; no order can be matched until it is safely persisted to highly available disk storage.

## 3. Back-of-the-Envelope (BoTE) Calculations
* **Throughput Target:** 100,000 Orders Per Second (OPS) per trading pair during high volatility (e.g., market crashes).
* **Latency Budget:** At 100,000 OPS, a single-threaded matching engine has exactly **10 microseconds** to process a single order.
* **Architecture Pivot:** Traditional databases (PostgreSQL, DynamoDB) average 2,000 to 10,000 microseconds (2-10ms) per write. We must bypass standard databases entirely for the critical matching path. The matching state must live exclusively in RAM.

## 4. Core Entities & Internal APIs

### In-Memory Entities
* `Order`: `order_id`, `user_id`, `symbol`, `side` (BUY/SELL), `type` (LIMIT/MARKET), `price`, `quantity`, `timestamp`.
* `Trade`: `trade_id`, `maker_order_id`, `taker_order_id`, `price`, `quantity`, `timestamp`.

### Critical Path APIs (gRPC / TCP)
* `RPC PlaceOrder(OrderRequest) returns (OrderResponse)`
* `RPC CancelOrder(CancelRequest) returns (CancelResponse)`

## 5. High-Level Design (HLD)

<img width="1757" height="792" alt="image" src="https://github.com/user-attachments/assets/226a2aad-cd15-455a-b60b-1db6432fc2bf" />


## 6. Detailed Data Flow (The LMAX Architecture)

   - Ingestion & Quorum: The API Gateway receives an order. It does not send it to the Matching Engine. It appends the order to a distributed Write-Ahead Log (the Sequencer). The Gateway waits until a quorum of disks (e.g., 2 out of 3 Kafka brokers) acknowledge the write.
   - Sequential Polling: The Matching Engine runs on a single thread. It aggressively polls the tail of the Sequencer log. Because it is single-threaded, there are zero race conditions. No locking is required.
   - Deterministic Execution: The engine pulls the order into RAM, traverses the Red-Black Tree, and mutates the Order Book state (matching SELL order, and create TRADE event).
   - Fan-Out (CQRS): The engine publishes a "Trade Event" event out to two asynchronous consumers (to an outbound Ring Buffer).
       1. The Ledger: Background threads (independent of the matching thread) consume this event to update the PostgreSQL settlement ledger and stream prices to the UI.
       2. Market Data: A background worker pushes the new price to WebSockets so all users see the chart update.
   
## 7. Deep Dives & Trade-offs

### Deep Dive 1: Data Structures for Microsecond Matching
- A standard array sort ($O(N \log N)$) is too slow. The Order Book must be engineered for $O(1)$ and $O(\log P)$ operations.
- We use a Balanced Binary Search Tree (Red-Black Tree) to maintain price levels. Bids are sorted descending; Asks are sorted ascending. Finding the best price is $O(\log P)$ where P is the number of distinct price levels.
- At each price level node, we maintain a Doubly Linked List of orders. This guarantees Price-Time priority.
- To support instant cancellations, we maintain a global Hash Map (Order_ID -> Memory Pointer). If a user cancels, we locate the pointer in $O(1)$ and remove it from the Linked List in $O(1)$.

### Deep Dive 2: Eliminating Mutex Locks (Single-Writer Principle)
- Multi-threading the core engine is an anti-pattern. Locking and unlocking memory (Mutexes) causes context switching and destroys the CPU cache, adding milliseconds of latency.
- The Solution: The engine is strictly Single-Threaded. Because only one thread mutates the Red-Black Tree, there are zero race conditions and zero locks required. We scale horizontally by sharding by trading pair (Node A handles BTC/USD, Node B handles ETH/USD).

### Deep Dive 3: Mechanical Sympathy & NUMA Architecture
- At the low level, hardware architecture dictates software design. To prevent CPU Cache Misses, the C++/Java application is designed with Mechanical Sympathy. Memory is pre-allocated at startup (preventing Garbage Collection pauses).
- The OS is configured to pin the matching thread to a specific physical CPU core. We disable Hyperthreading on that core, and ensure the memory allocated to the Order Book sits on the local NUMA (Non-Uniform Memory Access) node to prevent cross-motherboard memory fetching.Deep

### Dive 4: Catastrophic Recovery (Event Sourcing)
- If the server loses power, the RAM is wiped. Or A server reboot deletes millions of open orders. If we just read the entire Kafka log from the beginning of time (Event Sourcing replay), it will take 3 weeks to boot the engine.
- The Solution: Because every input was saved to the SSD Sequencer before execution, the new server simply replays the log from the beginning of time to rebuild the RAM state.
- The Optimization: Replaying 5 years of trades takes weeks. A background thread takes a serialized snapshot of the RAM state every 5 minutes and saves it to a fast local SSD (and replicates it to AWS S3). Upon crash recovery, the engine loads the latest 5-minute snapshot and only replays the Sequencer log for the final few minutes. 

### Dive 5: Strict Engine Determinism
- To guarantee that the snapshot and the replay perfectly match reality, the matching engine logic must be purely deterministic.
- The core engine is not allowed to make external network calls, read from a database, or even check the system clock (System.currentTimeMillis()).
- If the engine needs the time, the Gateway injects the timestamp into the payload before appending it to the Sequencer. This guarantees that during a crash replay, the engine evaluates the exact same timestamps it saw during the live run.

### Dive 6: Horizontal Scaling
- how do you scale the system when trade goes from 1M to 10M users? (For example, ELon tweet about Doge coins and sudden spike ).
- We cannot shard BTC/USD across multiple engines, because they must all match against the same book.
- Instead, we shard by Symbol. Engine Node 1 handles all BTC/USD trades. Engine Node 2 handles all ETH/USD trades. Since Bitcoin trades have zero impact on Ethereum trades, they can run entirely parallel on completely different physical servers.
