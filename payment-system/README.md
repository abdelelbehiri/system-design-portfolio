# Payment System

> **Status: work in progress.** The requirements and data model are written up. The deep dives and trade-offs are still in progress.

![Original Excalidraw design for the payment system](architecture-original.svg)

## 1. Product Requirements

### Functional requirements

- Merchants can initiate a payment request.
- Customers can pay for products with a credit or debit card.
- The system provides payment status updates to the merchant.

### Non-functional requirements

- **Highly secure.** The system must verify that merchants are who they say they are and protect customers' card and personal data.
- **Transaction safety and financial integrity.** A payment must never be charged twice or left in an inconsistent state.
- **Strong durability.** Transactions must never be lost.
- **Scale** to 10,000 transactions per second (TPS).

## 2. Data Model

```text
Merchant {
  id
  name
  apiKey
  metadata
}

PaymentIntent {
  id
  merchantId
  customerId
  status        // created | ...
  amount
  metadata
}

Transaction {
  id
  paymentId
  amount
  type
  status
  network
  metadata
}
```
