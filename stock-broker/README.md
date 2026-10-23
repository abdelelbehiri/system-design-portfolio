# Retail Stock Broker

[![Open the design board in Miro](https://img.shields.io/badge/Open%20the%20design%20board%20in-Miro-FFD02F?style=for-the-badge&logo=miro&logoColor=050038)](https://miro.com/app/board/uXjVH1rW-1M=/?share_link_id=903962015389)

**[View the original design board on Miro](https://miro.com/app/board/uXjVH1rW-1M=/?share_link_id=903962015389)** (view only): requirements, data model, API, architecture, dispatch idempotency and reconciliation.

The write-up below refines the original design; the main changes are listed in [section 6](#6-notes).

## 1. Product Requirements

### Functional requirements

- User can see live prices of stocks.
- User can manage orders of stocks.

### Non-functional requirements

- **High availability for price updates with eventual consistency.** The user should always be able to see stock prices and movements, but it is okay for prices to be stale for some seconds.
- **Strong consistency for order management.** It is okay for the user not to be able to place an order for some time, but an order must never be in an inconsistent state.
- **Low latency** for reflecting stock prices (under 200 ms) and for placing orders.
- **Scalability:** 10 million daily active users, 5 trades per day, 1000s of stocks.
- **Read-heavy system:** stock price updates >> trade orders.
- **Minimize active connections to the external exchange API.** Calls to the external exchange API are expensive.

### API

```text
GET    /subscribe?symbols=META,AAPL     → SSE stream: initial prices, then updates
GET    /stock/{symbol}                  → Stock
POST   /orders                          → 202 Accepted { orderId, state: "pending" }
       Idempotency-Key: <uuid>
       { side: "buy", symbol: "META", quantity: 5,
         type: "limit", limitPriceCents: 52210 }
DELETE /orders/{orderId}                → 202 Accepted { state: "pending_cancel" }
GET    /orders?cursor=…                 → Order[] (paginated, with executions)
```

- The user and account come from the authenticated token, never from the request body, so a client cannot act on another account.
- `POST /orders` returns `202 Accepted` because the order is accepted, not executed. The client learns about fills through `GET /orders` or order updates.
- The `Idempotency-Key` makes client retries safe: a retried request returns the original `orderId` instead of creating a second order.
- Prices and limits are integer cents, never floating point.

## 2. Clarifying Questions and Assumptions

### Scale and performance

- **10 million daily active users**, each placing about **5 orders per day**: about 50 million orders per day, roughly 600 orders per second on average. Trading clusters around market open and close, so the design targets peaks of at least 10x average.
- **Thousands of listed stocks**, each producing several price updates per second.
- Price updates should reach the client in **under 200 ms**. Order placement should be acknowledged quickly; execution time depends on the exchange.
- The system is heavily read-dominated: price updates far outnumber orders.
- Calls and connections to the exchange API are expensive and limited, so the design keeps few of them.

### Consistency and availability

- Prices favor availability and may be stale for a few seconds.
- Orders require strong consistency. It is acceptable for a user to be temporarily unable to place an order; it is not acceptable for an order to be duplicated, lost, or shown in the wrong state, or for a user to spend cash or shares they do not have.
- PostgreSQL is the source of truth for orders, executions, balances, and positions. The exchange is the source of truth for whether an order exists and what was executed.
### The exchange interface

The exchange's interface shapes most of the design, so it is pinned down first. The design assumes:

- **Order API:** place and cancel orders synchronously by request and response. A placed order returns the exchange's order ID, and the request accepts a client order ID that the exchange uses for deduplication and echoes back in later events.
- **Executions:** reported asynchronously by webhook, one event per execution with its own `execId`.
- **Prices:** a market data feed per symbol that the broker subscribes to.
- **Queries:** look up an order by client order ID, used by reconciliation.

Exchange-specific protocols (FIX, REST, WebSocket) sit behind adapters, so the rest of the system sees only this interface.

## 3. Data Model

```text
User {
  id
  name
}

Account {
  id
  user_id
  currency
}

Order {
  id                  // also the client order ID sent to the exchange
  account_id
  external_order_id   // exchange's ID; unique, indexed
  symbol
  side                // buy | sell
  type                // market | limit
  quantity
  limit_price_cents   // limit orders only
  filled_quantity
  idempotency_key     // from the client; unique per account
  state               // pending | dispatching | submitted | partially_filled | filled |
                      // pending_cancel | cancelling | cancelled | failed
  version
  created_at
  updated_at
}

Execution {
  exec_id             // exchange's execution ID; primary key
  order_id
  quantity
  price_cents
  executed_at
}

LedgerEntry {
  id
  account_id
  order_id
  asset               // EUR | symbol
  bucket              // available | reserved
  amount              // signed; entries of one movement sum to zero
  created_at
}

Balance {             // derived from LedgerEntry, kept for fast checks
  account_id
  asset
  available
  reserved
  version
}

OutboxEvent {
  id
  aggregate_id        // account_id; becomes the Kafka key
  type                // OrderCreated | OrderCancelRequested
  payload
  created_at
}

Stock {
  id
  symbol
  current_price_cents
}
```

Important constraints:

- Money is stored as integer cents (`price_cents`, `limit_price_cents`) to avoid floating-point rounding.
- `Execution.exec_id` is the primary key, so a duplicate webhook cannot apply the same fill twice.
- `Order.external_order_id` is unique and indexed. It replaces a separate key-value store for `externalOrderId → (orderId, userId)`, which would be a second write that can drift from the order row.
- `(account_id, idempotency_key)` is unique, so a retried `POST /orders` cannot create a second order.
- `Balance.available` has a check constraint `>= 0`. A reservation that would make it negative fails inside the transaction.
- State transitions only move forward. Every update is guarded by the expected current state.

## 4. Deep Dive

### Why the naive approach fails

A naive Orders Service inserts the order and then publishes to Kafka. If the process crashes between the two, the order is stored but never dispatched; if the publish succeeds and the commit fails, the dispatcher sends an order the database does not know about. A naive dispatcher that retries a timed-out exchange call can buy the same shares twice. And without a balance check, a user can place several orders that together spend more cash than they have.

### Reserving buying power before dispatch

Cash and shares are tracked in a double-entry ledger with two buckets per asset: `available` and `reserved`. Every movement is a set of entries that sum to zero, so balances are auditable and can be rebuilt from history.

- **Buy limit order:** reserve `quantity × limit_price_cents` plus estimated fees, moving it from `available` to `reserved` cash.
- **Buy market order:** reserve against the last price plus a safety buffer (a price collar), and send the order to the exchange with that collar as a protective limit.
- **Sell order:** reserve the shares from the position.

The reservation, the order insert, and the outbox row commit in one transaction:

```sql
BEGIN;

UPDATE balances
   SET available = available - :amount,
       reserved  = reserved  + :amount,
       version   = version + 1
 WHERE account_id = :account_id
   AND asset = 'EUR'
   AND available >= :amount;
-- 0 rows → insufficient buying power → ROLLBACK, return 422

INSERT INTO ledger_entries (...) VALUES (...);  -- available -amount, reserved +amount
INSERT INTO orders (id, account_id, ..., state) VALUES (:order_id, :account_id, ..., 'pending');
INSERT INTO outbox_events (aggregate_id, type, payload) VALUES (:account_id, 'OrderCreated', ...);

COMMIT;
```

On each execution, the filled amount moves from `reserved` cash to the position (or from reserved shares to cash for a sell). When the order reaches a final state (`filled`, `cancelled`, `failed`), any unused reservation is released back to `available`, for example the difference between the collar and the actual fill price.

Keeping the ledger in the same database as orders makes this a local transaction. If the ledger becomes its own service, the reservation becomes a saga step with `orderId` as its idempotency key and a compensating release.

### Publishing orders with a transactional outbox

The outbox row is written in the same transaction as the order, so either both exist or neither does. Debezium reads the outbox table from the PostgreSQL write-ahead log and publishes each row to Kafka. Delivery is at least once, so every consumer must be idempotent; the guarded claim below provides that for the dispatcher.

### Ordering: a cancel never overtakes its create

Outbox events use `account_id` as the Kafka key. Kafka preserves order within a partition, so the create and the cancel for an order reach the same dispatcher instance in the order they were committed. Keying by account rather than order also serializes all of an account's commands, which keeps reasoning about one user's orders simple; per-account volume is low enough that this does not create hot partitions.

If a cancel arrives while the order is still `pending`, the Orders Service cancels it directly with a guarded update (`pending → cancelled`), releases the reservation, and never calls the exchange.

### Dispatching exactly once to the exchange

Several dispatcher instances may see the same event (rebalances, redelivery). Each instance claims the order with a compare-and-set update before calling the exchange:

```sql
UPDATE orders
   SET state = 'dispatching', version = version + 1
 WHERE id = :order_id
   AND state = 'pending';
```

- 0 rows updated: another instance claimed it or it was already processed. Skip and commit the offset.
- 1 row updated: this instance owns the order and calls the exchange.

PostgreSQL's row lock makes the check and the write atomic. Cancels use the same pattern (`pending_cancel → cancelling`).

The exchange call carries `orderId` as the client order ID, so even if two calls slip through, the exchange rejects the duplicate. On acknowledgement, the dispatcher writes `submitted` and the `external_order_id`.

### Unknown outcomes and timeouts

An exchange timeout does not mean the order failed; the exchange may have accepted it. The dispatcher therefore distinguishes:

- **Definite rejection** (validation error, market closed, insufficient liquidity): move to `failed`, release the reservation, notify the user.
- **Timeout or connection error:** leave the order in `dispatching`. Do not retry blindly and do not mark it failed. The Reconciliation Job resolves it.

Only definite rejections and poison messages go to a dead-letter topic.

### Executions and partial fills

An order can be filled in several executions at different prices. Each webhook carries an `execId`, and the Order Updates Processor applies it in one transaction:

1. Verify the webhook signature (HMAC over the body with a shared secret, or mTLS) and that the caller is on the exchange's IP allow-list.
2. Insert the execution. A duplicate `exec_id` violates the primary key, so the webhook is acknowledged and nothing else happens.
3. Increase `filled_quantity`, write the ledger entries for the fill, and advance the state to `partially_filled` or `filled`, guarded by the current state and version so it only moves forward.
4. On a final state, release any remaining reservation.

The processor acknowledges the webhook only after the commit, so the exchange's retries cover crashes. Because executions are keyed by `exec_id`, out-of-order webhooks are harmless: the fill totals converge regardless of arrival order.

### Reconciliation

A periodic job selects orders that have been in `dispatching`, `submitted`, `partially_filled`, or `cancelling` for longer than a threshold and queries the exchange by client order ID.

For orders in `dispatching`:

- **Found at the exchange:** the send succeeded but the dispatcher did not record it. Store the `external_order_id`, the current state, and any missing executions.
- **Not found** after the exchange's acceptance window: the send never arrived. Move to `failed` and release the reservation, or re-dispatch if the product prefers that.

For orders in `cancelling`:

- **Cancelled at the exchange:** move to `cancelled` and release the remaining reservation.
- **Still open:** re-send the cancel (cancel by client order ID is idempotent at the exchange).
- **Filled before the cancel took effect:** record the missing executions and move to `filled` or `partially_filled`.

Missed webhooks are recovered the same way: executions reported by the exchange but absent locally are inserted through the same idempotent path. An end-of-day job also compares positions and cash with the exchange's and custodian's statements.

### Price fan-out

The Stock Price Processor keeps one connection per exchange feed regardless of user count, which satisfies the constraint on expensive exchange connections. Stocks Service instances subscribe to Redis channels on demand, so each price update is published once per symbol and delivered once per interested instance, not once per user. To protect clients and the network, instances conflate updates: if several ticks arrive for a symbol within a short window (for example 100 ms), only the latest is sent.

## 5. Trade-offs and Scaling

### Main trade-offs

- **Orders, executions, and ledger share one PostgreSQL database.** Reservation, order creation, and the outbox commit atomically without a distributed transaction, but the order path's write throughput is bounded by one primary until it is partitioned.
- **Transactional outbox with CDC.** No dual write and no lost orders, at the cost of operating Debezium and accepting a small dispatch delay and at-least-once delivery.
- **Kafka keyed by account.** Per-account ordering is guaranteed, but one slow account blocks its partition's later events, and repartitioning requires care.
- **Timeouts resolved by reconciliation.** No duplicate orders, but a user may see an order in `dispatching` until the job runs. The job interval trades exchange query cost against that delay.
- **Market orders reserved with a price collar.** Users cannot overspend, but some buying power is temporarily locked above the eventual fill price.
- **Redis Pub/Sub for prices.** Low latency and simple fan-out, but no delivery guarantee or replay; acceptable because prices are superseded continuously.
- **SSE instead of WebSockets.** Prices only flow server to client, so SSE is simpler and works over plain HTTP; order commands use REST.

### What is sacrificed for simplicity

- A single primary region for the order path; active-active trading across regions is not addressed.
- Settlement, custody, and corporate actions (splits, dividends) are out of scope, although the ledger is where they would be booked.
- Order updates reach the client by polling or push notification rather than a dedicated stream.
- One exchange; smart order routing across venues is not addressed.

### At 10x scale

- Scale Stocks Service instances horizontally by connection count; the price cache and Pub/Sub scale with symbols, not users.
- Add Kafka partitions and dispatcher instances; keep `account_id` as the key.
- Add read replicas for `GET /orders`, keeping reservations and state transitions on the primary.
- Batch or pool exchange connections in the dispatcher, within the exchange's rate limits.
- Monitor outbox lag, order age per state (especially `dispatching`), reconciliation corrections, webhook signature failures, reservation conflicts, and SSE fan-out latency.

### At 100x scale

- Partition the trading database by `account_id`. Orders, executions, and balances for an account stay on one shard, so the reservation transaction remains local.
- Split the price path by symbol range or region, with regional Pub/Sub clusters.
- Use regional cells to limit blast radius around market open.
- Move the ledger into its own service with saga-based reservations if independent scaling or team ownership requires it.

These are scaling directions, not capacity claims. They require measurements of traffic distribution, exchange limits, and peak behaviour.

### B2B multi-tenant variant

The same core can serve as broker infrastructure that other apps (banks, neobanks, fintechs) embed through an API, instead of a single consumer app. The changes:

- **Tenant ID everywhere:** on every account, order, ledger entry, and event; enforced in queries and in the API key or OAuth client.
- **Partition by tenant and account:** shards and Kafka keys include the tenant, so a large tenant can be isolated in its own cell.
- **Per-tenant rate limits and quotas** at the gateway, so one tenant's burst at market open does not starve others.
- **Outbound webhooks to tenants** for order and execution events, signed and retried with idempotency keys, built on the same outbox.
- **Audit trail:** an append-only log of every command and state transition, retained for regulatory reporting.
- **End-of-day reconciliation** against the custodian and the exchange per tenant, with breaks reported and resolved before the next trading day.

## 6. Notes

**What changed from the original Miro design?**
Added buying-power and position reservations with a double-entry ledger; replaced the DB-then-Kafka dual write with a transactional outbox and CDC; keyed Kafka by account; treated exchange timeouts as unknown outcomes resolved by reconciliation instead of dead-lettering them; added an executions table for partial fills; dropped the separate order metadata key-value store in favour of an indexed column; added webhook verification; and completed the state list with `dispatching` and `cancelling`.

**How do you prevent sending the same order twice?**
The dispatcher claims the order with a guarded update from `pending` to `dispatching`; only one instance can win. The exchange call carries `orderId` as the client order ID, so the exchange also rejects a duplicate.

**What happens if the exchange times out?**
The order stays in `dispatching`. The Reconciliation Job asks the exchange by client order ID and records what actually happened. A blind retry could double the position; marking it failed could hide a real execution.

**How do you stop a user from spending money they don't have?**
Cash or shares are moved from `available` to `reserved` in the same transaction that creates the order, guarded by `available >= amount` and a check constraint. Unused reservations are released when the order reaches a final state.

**Why an outbox instead of publishing to Kafka from the service?**
Writing to PostgreSQL and Kafka separately is not atomic. With an outbox, the event is part of the order's transaction and is published once the transaction commits.

**What if a webhook is delivered twice or out of order?**
Executions are keyed by `exec_id`, so duplicates are no-ops. Fill quantities are summed from stored executions and state only moves forward, so arrival order does not matter. Reconciliation recovers webhooks that never arrive.

**Why not let users cancel directly at the exchange?**
Cancels go through the same outbox and partition as creates, so a cancel cannot overtake its create. A cancel for a `pending` order never reaches the exchange at all.

**Why SSE and Redis Pub/Sub for prices?**
Prices only flow from server to client, and the next tick supersedes a lost one. SSE keeps connections simple; Pub/Sub fans out each update once per interested instance.

**Which signals would you monitor first?**
Price fan-out latency, SSE connection count, outbox and Kafka lag, order age per state, rate of timeouts and reconciliation corrections, reservation failures, webhook signature failures, and end-of-day reconciliation breaks.
