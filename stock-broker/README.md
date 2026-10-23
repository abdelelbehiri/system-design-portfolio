# Stocks Broker

[![Open the design board in Miro](https://img.shields.io/badge/Open%20the%20design%20board%20in-Miro-FFD02F?style=for-the-badge&logo=miro&logoColor=050038)](https://miro.com/app/board/uXjVH1rW-1M=/?share_link_id=903962015389)

**[View the design board on Miro](https://miro.com/app/board/uXjVH1rW-1M=/?share_link_id=903962015389)** (view only): requirements, data model, API, architecture, dispatch idempotency and reconciliation.

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
GET /stock/{symbol}
  → Stock

POST /order
{
  position: "buy",
  symbol: "META",
  shares: 5,
  type: LIMIT | MARKET,
  priceInCents: 52210   // limit order
}
  → 202 Accepted

DELETE /order/{orderId}
  → 202 Accepted

GET /orders
  → Order[] (paginated)
```

## 2. Data Model

```text
User {
  userId
  name
}

Order {
  orderId
  userId
  externalOrderId
  stock
  shares
  price
  state   // pending | pending_cancel | dispatching | cancelling | submitted |
          // partially_filled | filled | cancelled | failed
}

Stock {
  stockId
  symbol
  currentPrice
}
```

## 3. Deep Dive

### Order Dispatch Service

**Orders:**

1. Idempotency check: if the order is already processed, do nothing and proceed.
2. Send the order to the exchange with the order ID.
3. Write the `submitted` status to the Orders DB.
4. Write `externalOrderId → (orderId, userId)` in the Order Metadata DB.

If sending to the exchange fails, move the order to a dead letter.

**Cancellation:**

1. Idempotency check: if the cancellation is already processed, do nothing and proceed.
2. Send the cancellation of the order to the exchange.
3. Write `cancelled` in the Orders DB.

### Dispatch idempotency

Two instances of the Order Dispatch Service must never submit the same order twice to the exchange API. The `orderId` is the idempotency key, and the service performs an idempotency check before calling the exchange: an atomic write with a strict guard condition.

```sql
UPDATE orders SET state = 'dispatching'
WHERE order_id = ? AND state = 'pending';
```

Examine the affected row count:

- **0 rows updated:** already claimed or processed elsewhere. Skip and commit the offset.
- **1 row updated:** this consumer owns the order and proceeds to call the exchange.

Same pattern for cancel: `pending_cancel → cancelling`.

**Why it works:** the row-level lock makes check + write atomic, so two consumers cannot acquire the same order and process it.

**Also:** the exchange call carries the order ID as the client / idempotency key in the client metadata, so even a duplicate call is handled or ignored on the exchange side.

### Reconciliation job

**Orders in `pending` or `dispatching`:** the job queries the exchange with the idempotency key.

1. **The job finds the order.** The request reached the exchange, but the Order Dispatch Service failed to update the DB. Fill in the current status (`submitted`, `filled` or `partially_filled`) and the `externalOrderId`, and update the metadata store if necessary.
2. **The job cannot find the order.** Update the DB with the status `failed`.

**Orders in `pending_cancel` or `cancelling`:** the job queries the exchange with the idempotency key.

1. **The order is cancelled:** update the local DB to `cancelled`.
2. **The order is still active:** ensure the cancellation and set the status to `cancelled`.
3. **The order was filled:** if it was executed before the cancellation could be processed, update the DB to `filled` or `partially_filled`.
