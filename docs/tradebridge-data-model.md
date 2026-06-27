# TradeBridge — Data Model & Domain Reference

This document defines every entity that flows through the TradeBridge integration platform. Every RAML spec, every DataWeave transformation, and every flow built in this project traces back to the contracts defined here. Commit this as the first file in the repository — it is the source of truth for the whole build.

## Domain entities

### Product
Represents a single sellable item in inventory. Owned by the **Inventory System API**.

| Field | Type | Notes |
|---|---|---|
| productId | string | e.g. `"PRD-1001"` |
| sku | string | warehouse stock keeping unit |
| name | string | e.g. `"Wireless Mouse"` |
| description | string | |
| category | string | e.g. `"Electronics"` |
| unitPrice | number | in ZAR, two decimal places |
| stockQuantity | number | current available stock |

### Customer
Represents whoever is placing an order — either a retail end-customer or a B2B partner. Kept simple and shared across both Experience APIs.

| Field | Type | Notes |
|---|---|---|
| customerId | string | e.g. `"CUST-3301"` |
| name | string | |
| email | string | used for notifications |
| accountType | enum | `RETAIL` or `PARTNER` |

### Order
The central transactional entity. Owned by the **Orders System API**, created and orchestrated by the **Order Processing Process API**.

| Field | Type | Notes |
|---|---|---|
| orderId | string | generated server-side, e.g. `"ORD-88421"` |
| customerId | string | foreign key to Customer |
| orderDate | datetime | ISO 8601 |
| status | enum | `PENDING`, `CONFIRMED`, `FULFILLED`, `CANCELLED` |
| items | array of OrderItem | see below |
| totalAmount | number | sum of all line totals |

### OrderItem
A single line within an order. Always nested inside an Order — never a standalone resource.

| Field | Type | Notes |
|---|---|---|
| productId | string | foreign key to Product |
| quantity | number | |
| unitPrice | number | price at time of order (snapshot, not live-looked-up) |
| lineTotal | number | quantity × unitPrice |

### Notification
Represents an outbound message triggered by order events. Owned by the **Notifications System API**.

| Field | Type | Notes |
|---|---|---|
| notificationType | enum | `ORDER_CONFIRMATION`, `LOW_STOCK_ALERT`, `ORDER_FAILED` |
| recipientEmail | string | |
| subject | string | |
| body | string | |
| timestamp | datetime | |

## Canonical JSON contracts

These are the exact payload shapes used internally between the Process and System layers. Both Experience APIs (retail JSON, partner XML) transform their inbound requests into this shape before calling the Process API — this is the canonical internal contract.

### Product (response shape, from Inventory System API)
```json
{
  "productId": "PRD-1001",
  "sku": "WM-BLK-01",
  "name": "Wireless Mouse",
  "category": "Electronics",
  "unitPrice": 249.99,
  "stockQuantity": 84
}
```

### Order creation request — canonical internal shape
This is what the Retail Experience API produces directly from its JSON input, and what the Partner Experience API produces after transforming inbound XML. The Process API only ever sees this shape — it has no awareness of where the request originated.
```json
{
  "customerId": "CUST-3301",
  "items": [
    { "productId": "PRD-1001", "quantity": 2 },
    { "productId": "PRD-1042", "quantity": 1 }
  ]
}
```

### Partner Experience API — inbound XML (before transformation)
This is the format a B2B partner system sends. It must be transformed into the canonical shape above before reaching the Process API.
```xml
<purchaseOrder>
  <partnerCustomerId>CUST-3301</partnerCustomerId>
  <lineItems>
    <lineItem>
      <sku>WM-BLK-01</sku>
      <qty>2</qty>
    </lineItem>
  </lineItems>
</purchaseOrder>
```
Note the deliberate mismatch with the canonical shape: `partnerCustomerId` vs `customerId`, `sku` vs `productId`, `qty` vs `quantity`, and a nested `lineItems` wrapper that doesn't exist internally. This mismatch is intentional — it's what gives you a real DataWeave transformation to build and show off, not a trivial pass-through.

### Order — response shape (returned to caller after creation)
```json
{
  "orderId": "ORD-88421",
  "customerId": "CUST-3301",
  "orderDate": "2026-06-20T10:15:00Z",
  "status": "CONFIRMED",
  "items": [
    {
      "productId": "PRD-1001",
      "quantity": 2,
      "unitPrice": 249.99,
      "lineTotal": 499.98
    }
  ],
  "totalAmount": 499.98
}
```

### Error response shape (used by every API in the platform)
A consistent error contract across all layers — this is what your error handlers (Try scope, On Error Continue/Propagate) ultimately produce.
```json
{
  "status": "error",
  "errorCode": "INSUFFICIENT_STOCK",
  "message": "Requested quantity exceeds available stock for PRD-1001",
  "timestamp": "2026-06-20T10:15:00Z"
}
```

## H2 database schema sketch (System API layer)

Rough shape of the tables the Inventory and Orders System APIs will read/write. Exact DDL gets finalized when we build those flows in Phase 3.

```sql
CREATE TABLE products (
  product_id    VARCHAR(20) PRIMARY KEY,
  sku           VARCHAR(30),
  name          VARCHAR(100),
  category      VARCHAR(50),
  unit_price    DECIMAL(10,2),
  stock_quantity INT
);

CREATE TABLE orders (
  order_id      VARCHAR(20) PRIMARY KEY,
  customer_id   VARCHAR(20),
  order_date    TIMESTAMP,
  status        VARCHAR(20),
  total_amount  DECIMAL(10,2)
);

CREATE TABLE order_items (
  order_id      VARCHAR(20),
  product_id    VARCHAR(20),
  quantity      INT,
  unit_price    DECIMAL(10,2),
  line_total    DECIMAL(10,2),
  FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
```

## What this document drives

Every RAML spec written in Phase 2 declares request/response bodies matching the JSON contracts above. Every DataWeave transformation built in Phase 3–5 maps between the XML/JSON shapes shown here. The H2 schema above becomes the Database connector queries in the System APIs. Nothing in later phases should require inventing a new field — if it does, this document gets updated first, and the change is reflected everywhere downstream.
