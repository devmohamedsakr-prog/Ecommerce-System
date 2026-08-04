# E-Commerce System - Service Integration Guide

## 📋 Overview

Complete guide to how all 10 microservices work together to process orders end-to-end.

## 🔄 Complete Order Processing Flow (Integrated)

### Step 1: Customer Browses & Adds to Cart

```
SERVICES INVOLVED:
├── Catalog Service (product information)
├── Inventory Service (stock availability)
├── Cart Service (persistence)
└── Recommendation Service (suggestions)

SEQUENCE:

Customer visits store
    ↓ [Catalog Service]
Fetch product details
    {
      "productId": "prod-123",
      "name": "Blue T-Shirt",
      "price": 29.99,
      "images": [...],
      "description": "..."
    }
    ↓ [Inventory Service]
Check stock levels
    {
      "productId": "prod-123",
      "inStock": true,
      "quantity": 150,
      "warehouse": "NY-1"
    }
    ↓ [Catalog Service → Recommendation Service]
Show similar products
    {
      "recommendations": [
        { "productId": "prod-124", "reason": "often_bought_together" },
        { "productId": "prod-125", "reason": "customers_also_viewed" }
      ]
    }
    ↓ [Cart Service]
Add to cart
    {
      "cartId": "cart-abc123",
      "items": [
        {
          "productId": "prod-123",
          "quantity": 2,
          "price": 29.99
        }
      ]
    }
    Cart stored in Redis (TTL: 30 days)
```

### Step 2: Checkout Initiation

```
SERVICES INVOLVED:
├── Cart Service (get cart contents)
├── User Service (get customer info)
├── Inventory Service (reserve stock)
└── Order Service (create order record)

SEQUENCE:

Customer clicks checkout
    ↓ [Cart Service]
Retrieve cart contents
    {
      "cartId": "cart-abc123",
      "items": [...],
      "subtotal": 59.98
    }
    ↓ [User Service]
Validate customer & get addresses
    {
      "customerId": "cust-456",
      "email": "customer@example.com",
      "savedAddresses": [...]
    }
    ↓ [Inventory Service]
Reserve inventory (15-min hold)
    {
      "productId": "prod-123",
      "quantity": 2,
      "reservationId": "rsv-xyz789",
      "expiresAt": "2026-08-04T11:30:00Z"
    }
    Stock decremented: 150 → 148
    ↓ [Order Service]
Create order with reserved items
    {
      "orderId": "order-789",
      "customerId": "cust-456",
      "items": [
        {
          "productId": "prod-123",
          "quantity": 2,
          "price": 29.99,
          "reservationId": "rsv-xyz789"
        }
      ],
      "status": "pending_payment",
      "expiresAt": "2026-08-04T18:30:00Z"
    }
```

### Step 3: Payment Processing

```
SERVICES INVOLVED:
├── Payment Service (authorization)
├── Order Service (updates order status)
└── Inventory Service (confirms reserve)

SEQUENCE:

Customer enters payment info
    ↓ [Payment Service]
Authorize payment
    POST /payments/authorize
    {
      "orderId": "order-789",
      "amount": 59.98,
      "currency": "USD",
      "paymentToken": "tok_visa_4242",
      "billingAddress": {...},
      "shippingAddress": {...}
    }
    
    Response:
    {
      "transactionId": "txn-123",
      "status": "authorized",
      "authCode": "AUTH123456",
      "expiresAt": "2026-08-11T10:30:00Z"
    }
    ↓ [Order Service]
Update order status
    {
      "orderId": "order-789",
      "status": "payment_authorized",
      "transactionId": "txn-123",
      "updatedAt": "2026-08-04T10:30:05Z"
    }
    ↓ [Inventory Service]
Confirm reservation (convert to committed)
    {
      "reservationId": "rsv-xyz789",
      "status": "committed",
      "committedAt": "2026-08-04T10:30:05Z"
    }
```

### Step 4: Order Confirmation & Capture

```
SERVICES INVOLVED:
├── Payment Service (capture funds)
├── Order Service (finalize order)
├── Inventory Service (decrement stock)
└── Notification Service (send confirmation)

SEQUENCE:

Payment successfully authorized
    ↓ [Payment Service]
Capture payment
    POST /payments/txn-123/capture
    {
      "amount": 59.98
    }
    
    Response:
    {
      "transactionId": "txn-123",
      "status": "captured",
      "capturedAt": "2026-08-04T10:30:10Z"
    }
    ↓ [Order Service]
Publish event: OrderConfirmed
    {
      "event": "OrderConfirmed",
      "orderId": "order-789",
      "customerId": "cust-456",
      "amount": 59.98,
      "items": [...]
    }
    Event published to Kafka
    
    Order status updated:
    {
      "orderId": "order-789",
      "status": "confirmed",
      "confirmedAt": "2026-08-04T10:30:10Z"
    }
    ↓ [Inventory Service]
Decrement actual inventory
    {
      "productId": "prod-123",
      "quantity": -2,
      "action": "order_confirmed",
      "orderId": "order-789"
    }
    Stock updated: 148 → 146
    ↓ [Notification Service]
Send confirmation email
    Email address: customer@example.com
    Subject: "Order #order-789 Confirmed"
    Content: Order summary, items, total
    ↓ [Cart Service]
Clear cart
    {
      "cartId": "cart-abc123",
      "status": "cleared"
    }
```

### Step 5: Fulfillment

```
SERVICES INVOLVED:
├── Order Service (get order details)
├── Inventory Service (reserve from warehouse)
├── Shipping Service (coordinate logistics)
└── Notification Service (send tracking)

SEQUENCE:

Order confirmed → Fulfillment triggered
    ↓ [Order Service]
Create fulfillment request
    {
      "orderId": "order-789",
      "items": [
        {
          "productId": "prod-123",
          "quantity": 2,
          "warehouse": "NY-1"
        }
      ],
      "shippingAddress": {...},
      "shippingMethod": "standard"
    }
    ↓ [Inventory Service]
Reserve for fulfillment
    {
      "orderId": "order-789",
      "productId": "prod-123",
      "quantity": 2,
      "warehouse": "NY-1",
      "status": "reserved_for_fulfillment"
    }
    ↓ [Shipping Service]
Request shipping quote
    {
      "orderId": "order-789",
      "weight": 0.5,  # kg
      "dimensions": { "length": 30, "width": 20, "height": 10 },
      "destination": {...}
    }
    
    Response (carrier quotes):
    {
      "methods": [
        {
          "id": "standard",
          "carrier": "FedEx",
          "cost": 5.99,
          "estimatedDays": 5
        },
        {
          "id": "express",
          "carrier": "FedEx",
          "cost": 12.99,
          "estimatedDays": 2
        }
      ]
    }
    ↓ [Order Service]
Update fulfillment status
    {
      "orderId": "order-789",
      "status": "fulfillment_in_progress",
      "warehouse": "NY-1",
      "carrier": "FedEx",
      "trackingNumber": "1Z999AA10123456784"
    }
    ↓ [Notification Service]
Send shipping notification
    Email: customer@example.com
    Subject: "Your order has shipped"
    Content: Tracking number, estimated delivery
```

### Step 6: Delivery & Completion

```
SERVICES INVOLVED:
├── Shipping Service (tracking updates)
├── Order Service (status updates)
├── Review Service (enable reviews)
└── Notification Service (delivery confirmation)

SEQUENCE:

Package in transit
    ↓ [Shipping Service]
Poll carrier for tracking updates (daily)
    GET FedEx API for tracking
    Response: "In Transit"
    ↓ [Notification Service]
Send update to customer
    Email: "Your package is on the way"
    SMS: "Tracking: [link]"
    ↓ [Order Service]
Update status
    {
      "orderId": "order-789",
      "status": "shipped",
      "lastUpdate": "2026-08-05T14:20:00Z"
    }

Package delivered
    ↓ [Shipping Service]
Carrier confirms delivery
    Response: "Delivered - Signature: John D"
    ↓ [Order Service]
Update order to complete
    {
      "orderId": "order-789",
      "status": "delivered",
      "deliveredAt": "2026-08-08T14:20:00Z",
      "reviewWindow": {
        "opensAt": "2026-08-08T14:20:00Z",
        "closesAt": "2026-09-08T14:20:00Z"  # 30 days
      }
    }
    ↓ [Review Service]
Enable reviews for this order
    {
      "orderId": "order-789",
      "customerId": "cust-456",
      "items": [
        {
          "productId": "prod-123",
          "allowReview": true
        }
      ]
    }
    ↓ [Notification Service]
Send delivery confirmation + review request
    Email: customer@example.com
    Subject: "Your order has arrived - Leave a review"
    Content: Review link, product images
```

### Step 7: Post-Delivery (Returns, Reviews, etc.)

```
SERVICES INVOLVED:
├── Review Service (collect feedback)
├── Order Service (handle returns)
├── Inventory Service (restock)
├── Payment Service (process refunds)
└── Recommendation Service (update ratings)

SCENARIO A: Customer Leaves Review

Customer clicks review
    ↓ [Review Service]
Submit review
    {
      "orderId": "order-789",
      "productId": "prod-123",
      "rating": 5,
      "title": "Great shirt!",
      "comment": "Fits perfectly, great quality",
      "verifiedPurchase": true
    }
    ↓ [Recommendation Service]
Update product rating
    {
      "productId": "prod-123",
      "newRating": 4.7,
      "reviewCount": 1523
    }
    Recommendation model updates
    ↓ [Catalog Service]
Update product page
    Display new rating in search results

SCENARIO B: Customer Returns Item

Customer initiates return
    ↓ [Order Service]
Create return request
    {
      "orderId": "order-789",
      "productId": "prod-123",
      "quantity": 1,
      "reason": "quality_issue",
      "refundAmount": 29.99
    }
    ↓ [Shipping Service]
Generate return label
    Return carrier: UPS
    Return address: Warehouse address
    Label: [PDF]
    ↓ [Notification Service]
Send return instructions
    Email: customer@example.com
    Subject: "Return instructions for order #order-789"
    Content: Shipping label, return address, warehouse address
    
    Customer ships package back

Return package arrives
    ↓ [Inventory Service]
Receive returned item
    {
      "orderId": "order-789",
      "productId": "prod-123",
      "quantity": 1,
      "condition": "returned"
    }
    Stock updated: 146 → 147
    ↓ [Order Service]
Approve return
    {
      "returnId": "ret-123",
      "orderId": "order-789",
      "status": "approved",
      "refundAmount": 29.99
    }
    ↓ [Payment Service]
Process refund
    POST /payments/txn-123/refund
    {
      "amount": 29.99,
      "reason": "customer_return"
    }
    
    Response:
    {
      "refundId": "refund-456",
      "status": "processing",
      "estimatedAt": "2026-08-15"
    }
    ↓ [Notification Service]
Send refund confirmation
    Email: customer@example.com
    Subject: "Return approved - Refund processed"
    Content: Refund amount, estimated arrival date
```

## 📊 Service Communication Patterns

### Synchronous (Request-Response)

```
Used for: Critical operations, immediate feedback

Service A → Service B (REST/gRPC)
Wait for response before proceeding

Examples:
├── Order Service → Inventory Service (check stock)
├── Order Service → User Service (get customer)
└── Payment Service → Fraud Detection Service (verify payment)

Advantages:
✅ Immediate feedback
✅ Strong consistency
✅ Error handling clear

Disadvantages:
❌ Slower (waits for response)
❌ Tight coupling
❌ Cascading failures (if Service B down)
```

### Asynchronous (Event-Driven)

```
Used for: Non-blocking operations, notifications

Service A publishes event → Event broker (Kafka) → Service B consumes

Examples:
├── Order confirmed → Inventory service updates stock
├── Payment captured → Notification service sends email
├── Review submitted → Recommendation service updates ratings

Advantages:
✅ Fast (fire and forget)
✅ Loose coupling
✅ Fault tolerant (retry on failure)
✅ Easy to add new subscribers

Disadvantages:
❌ Eventual consistency
❌ Harder to debug
❌ Retry logic needed
```

### Hybrid Approach

```
Critical path: Synchronous
├── Order Service → Inventory Service (must check stock immediately)
├── Payment Service → Fraud Detection (must check fraud immediately)
└── Order Service confirms with Payment Service (must wait for auth)

Non-critical path: Asynchronous
├── Order confirmed → Notification Service (can be delayed 1-2 sec)
├── Order confirmed → Analytics Service (can be delayed minutes)
└── Payment captured → Reporting Service (can be delayed hours)
```

## 🔌 API Contracts

### Service-to-Service APIs

**Order Service → Inventory Service**

```json
Request:
POST /api/v1/inventory/reserve

{
  "items": [
    {
      "productId": "prod-123",
      "quantity": 2
    }
  ],
  "orderId": "order-789"
}

Response:
{
  "reservationId": "rsv-xyz789",
  "items": [
    {
      "productId": "prod-123",
      "reserved": 2,
      "available": 148
    }
  ],
  "expiresAt": "2026-08-04T11:30:00Z"
}

Error Response:
{
  "error": "INSUFFICIENT_STOCK",
  "items": [
    {
      "productId": "prod-123",
      "requested": 2,
      "available": 1
    }
  ]
}
```

**Order Service → Payment Service**

```json
Request:
POST /api/v1/payments/authorize

{
  "orderId": "order-789",
  "amount": 59.98,
  "currency": "USD",
  "paymentToken": "tok_visa_4242",
  "customerId": "cust-456",
  "metadata": {
    "shippingAddress": {...},
    "billingAddress": {...}
  }
}

Response:
{
  "transactionId": "txn-123",
  "status": "authorized",
  "authCode": "AUTH123456",
  "expiresAt": "2026-08-11T10:30:00Z"
}

Error Response:
{
  "error": "PAYMENT_DECLINED",
  "code": "card_declined",
  "message": "Card was declined",
  "retryable": true
}
```

## 🛡️ Error Handling & Recovery

### Service Failure Scenarios

**Scenario 1: Inventory Service Down**

```
Customer clicks checkout
    ↓ [Order Service]
Try to reserve inventory
    Inventory Service not responding (timeout)
    
    Options:
    1. Fail immediately (tell customer "Try again")
    2. Retry with exponential backoff (retry 3 times)
    3. Use cached data (inventory from 5 min ago)
    4. Degrade gracefully (allow checkout, reserve later)

    Recommended: Retry once, then degrade
    ├── First retry: Immediate (maybe just slow)
    ├── Second retry: Wait 1 second
    ├── If still failing: Allow checkout
    ├── Mark order for manual verification
    └── Notify Inventory Team

Inventory comes back up:
    Batch process orders needing verification
    ├── Check inventory
    ├── If insufficient, cancel order + refund
    └── If sufficient, finalize fulfillment
```

**Scenario 2: Payment Processor Down**

```
Customer submits payment
    ↓ [Payment Service]
Try to authorize with Stripe/PayPal
    Payment processor API down
    
    Response: 503 Service Unavailable
    
    Options:
    1. Retry (Stripe usually recovers within 30 seconds)
    2. Fallback to backup processor
    3. Queue for retry (ask customer to retry in 5 min)
    
    Recommended: Retry + Queue
    ├── Immediate retry (1-2 times)
    ├── If failing: Store in queue
    ├── Attempt retry every 1 minute (for 1 hour)
    ├── Notify customer: "Payment processing delayed"
    ├── Send email with retry link
    └── If succeeds within 1 hour: Process order
    └── If fails: Ask customer to try different card
```

**Scenario 3: Notification Service Down**

```
Order confirmed but emails not sending
    ↓ [Notification Service]
Emails queued in persistent queue
    
    If email service down > 1 hour:
    ├── Send SMS (if phone number available)
    ├── Show message in customer dashboard
    ├── Page admin team
    ├── Manually retry emails when service recovers
    └── Log all failed emails for audit
```

## ✅ Integration Checklist

```
Before Going to Production:

Service Dependencies:
- [ ] All service discovery working
- [ ] All APIs versioned
- [ ] All APIs documented (Swagger/OpenAPI)
- [ ] Rate limiting configured
- [ ] Circuit breakers implemented

Testing:
- [ ] Integration tests for full order flow
- [ ] Failure scenario tests (service down)
- [ ] Load tests (peak traffic)
- [ ] Chaos engineering tests
- [ ] End-to-end tests (real customer journey)

Monitoring:
- [ ] All service latencies monitored
- [ ] All error rates monitored
- [ ] Distributed tracing enabled (Jaeger)
- [ ] Alerts for failures
- [ ] Dashboard for critical metrics

Operations:
- [ ] Runbooks for failures
- [ ] On-call rotation
- [ ] Incident response process
- [ ] Logging centralized (ELK)
- [ ] Security scanning (OWASP, dependency scanning)
```

---

**Key Principle:** Services should be loosely coupled but tightly coordinated. Use synchronous calls for critical paths, asynchronous events for everything else.
