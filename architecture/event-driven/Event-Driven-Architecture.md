# Event-Driven Architecture

## 📋 Overview

Complete guide to building scalable, responsive e-commerce systems using event-driven patterns.

## 🎯 Core Concepts

### Events vs Commands

```
Command (Request-Response):
"Please authorize this payment"
    ↓ (Synchronous)
Response: "Authorization successful"

Event (Fact):
"Payment was authorized"
    ↓ (Asynchronous - multiple subscribers)
├── Notify customer
├── Update inventory
├── Generate receipt
└── Update analytics
```

### Publish-Subscribe Pattern

```
Publisher (Order Service):
├── Payment Authorized
├── Inventory Reserved
├── Order Confirmed
└── Shipment Scheduled

Events published to Kafka topic: "orders"

Subscribers:
├── Notification Service → sends email
├── Inventory Service → decrements stock
├── Analytics Service → records metrics
├── Recommendation Service → updates model
└── Email Service → generates receipt

Key Feature: Publisher doesn't know subscribers!
```

## 🔄 Event Flow Architecture

### Order Processing with Events

```
Traditional Synchronous:

Order Service (blocks while calling other services):
1. Call Inventory Service (wait for response)
2. Call Payment Service (wait for response)
3. Call Shipping Service (wait for response)
4. Return result

Time: Sum of all calls (100+100+100 = 300ms)

Event-Driven Asynchronous:

Order Service (publishes events immediately):
1. Publish: OrderCreated
2. Return immediately (10ms)
   │
   ├─→ Inventory Service consumes → processes async
   ├─→ Payment Service consumes → processes async
   ├─→ Shipping Service consumes → processes async
   └─→ Others can also consume

Time: 10ms (initial) + async processing

Benefits:
✅ Much faster response (10ms vs 300ms)
✅ Services independent (can be down)
✅ Easy to add new subscribers
✅ Better scalability
```

## 🏗️ Event Sourcing

### Event Store (Source of Truth)

```
Traditional Database:
┌──────────────────┐
│ Order            │
├──────────────────┤
│ id: order-123    │
│ status: shipped  │
│ total: $99.99    │
└──────────────────┘

Problem: Can't answer "how did we get here?"

Event Store:
┌─────────────────────────────────────────┐
│ All events that ever happened:          │
├─────────────────────────────────────────┤
│ 1. OrderCreated (2026-08-04 10:00)      │
│ 2. PaymentAuthorized (10:01)            │
│ 3. PaymentCaptured (10:02)              │
│ 4. InventoryReserved (10:03)            │
│ 5. ShipmentCreated (10:04)              │
│ 6. ShipmentDispatched (10:05)           │
│ 7. ShipmentInTransit (10:30)            │
│ 8. ShipmentDelivered (10:45)            │
└─────────────────────────────────────────┘

Benefits:
✅ Complete audit trail
✅ Can replay events (debugging)
✅ Can time-travel (what was status at 10:25?)
✅ Can change business logic retroactively
✅ CQRS ready (separate read model)
```

### Event Replay

```
Current State:
Order status: shipped, total: $99.99

Question: What happened between 10:00-10:05?

Replay events 1-5:
1. OrderCreated: Order created
2. PaymentAuthorized: Payment verified
3. PaymentCaptured: $99.99 charged
4. InventoryReserved: Item reserved from warehouse
5. ShipmentCreated: Prepared for shipment

Conclusion: Between 10:00-10:05, order went from created to ready to ship
```

### CQRS (Command Query Responsibility Segregation)

```
Traditional Model:
One database (normalized)
├── Write: UPDATE orders SET status = 'shipped'
└── Read: SELECT * FROM orders (must JOIN multiple tables)

CQRS Model:
Command Store (Event Sourced):
├── Events: OrderCreated, PaymentAuthorized, Shipped
└── Written only (append-only)

Read Model (Denormalized):
├── Orders table
│   ├── id
│   ├── status
│   ├── total
│   └── shipped_date
└── Read-optimized (single table, no joins)

Flow:
1. User executes command: "Ship order"
2. Command handler validates and publishes event: "OrderShipped"
3. Event handler updates read model
4. User queries read model (fast!)

Benefits:
✅ Read and write optimized independently
✅ Read model can be different (NoSQL, graph, etc.)
✅ Easy to add new read models
✅ Better performance (no joins)

Example:
Write: POST /orders/123/ship (command)
  └─ Publishes: OrderShipped event
  └─ Updates: Read model

Read: GET /orders/123 (query)
  └─ Reads from: Denormalized read model (fast!)
```

## 🎯 Event Patterns

### Event Notification

```
Publisher broadcasts: "Something happened"
Subscribers: "Got it, I'll handle it"

Example:
Order Service: "OrderCreated event published"
├── Notification Service subscribes
│   └─ Action: Send order confirmation email
├── Inventory Service subscribes
│   └─ Action: Reserve items
└── Analytics Service subscribes
    └─ Action: Record order metric

Latency: 50-500ms (async processing)
```

### Event-Carried State Transfer

```
Event includes all relevant data:

Traditional:
├── Event: "OrderShipped"
├── Data: Just order ID
└── Subscriber must look up order details → Extra database call

Better:
├── Event: "OrderShipped"
├── Data: Full order details (no lookup needed)
│   ├── OrderId
│   ├── CustomerId
│   ├── Items
│   ├── Total
│   ├── ShippingAddress
│   └── TrackingNumber
└── Subscriber has everything needed (no database call)

Benefit:
✅ Faster (no database lookup)
✅ Resilient (works if order service down)
❌ More data per event (storage)
```

### Event Correlation

```
Trace Request Through System:

User Action: Add item to cart
  ├─ CartItemAdded event
  │  └─ correlationId: req-123-abc
  │     (unique ID for this request)
  │
  ├─ Inventory Service consumes
  │  └─ Publishes: StockReserved
  │     correlationId: req-123-abc (SAME!)
  │
  ├─ Notification Service consumes
  │  └─ Publishes: NotificationQueued
  │     correlationId: req-123-abc (SAME!)
  │
  └─ All events linked by correlationId
      (can trace full request path)

Debugging:
grep correlationId:req-123-abc logs
  → See all services involved in this request
  → Understand latency breakdown
  → Find performance bottleneck
```

## 📊 Event Storage (Kafka)

### Kafka Topic Structure

```
Topics organize events:
├── orders (Order-related events)
│   ├── OrderCreated
│   ├── OrderConfirmed
│   ├── OrderShipped
│   └── OrderDelivered
│
├── payments (Payment-related events)
│   ├── PaymentAuthorized
│   ├── PaymentCaptured
│   └── PaymentRefunded
│
├── inventory (Inventory-related events)
│   ├── StockReserved
│   ├── StockReleased
│   └── StockAdjusted
│
└── shipping (Shipping-related events)
    ├── ShipmentCreated
    ├── ShipmentDispatched
    ├── ShipmentInTransit
    └── ShipmentDelivered

Partitioning (Order Service example):
└── orders topic
    ├── Partition 0: Orders with ID hash % 3 = 0
    ├── Partition 1: Orders with ID hash % 3 = 1
    └── Partition 2: Orders with ID hash % 3 = 2

Benefit: Parallel processing (3 partitions = 3x throughput)
```

### Consumer Groups

```
Multiple subscribers to same topic:

orders topic
├── Consumer Group 1 (Notification Service)
│   └─ Consumes all events
│      └─ Sends emails
│
├── Consumer Group 2 (Inventory Service)
│   └─ Consumes all events
│      └─ Updates stock
│
└── Consumer Group 3 (Analytics Service)
    └─ Consumes all events
        └─ Records metrics

Key Feature:
├── Each consumer group has independent offset
├── One group doesn't affect others
├── Can replay from specific offset
└── Can pause/resume independently
```

## 🔄 Handling Failures

### Idempotency

```
Problem: What if same event processed twice?

Without Idempotency:
Event: PaymentCaptured
├── First processing: $99.99 charged ✓
└── Duplicate: $99.99 charged again ✗ (Wrong!)

With Idempotency:
Event: PaymentCaptured (idempotencyKey: "abc123")
├── First processing:
│   ├─ Check: idempotencyKey exists? No
│   ├─ Charge: $99.99
│   └─ Store: idempotencyKey "abc123" → processed
│
└── Duplicate:
    ├─ Check: idempotencyKey exists? Yes
    ├─ Return cached result
    └─ Don't charge again

Implementation:
Database table:
┌──────────────────┐
│ IdempotencyKey   │
├──────────────────┤
│ abc123: charged  │
│ def456: shipped  │
└──────────────────┘

All event handlers check table before processing
```

### Dead Letter Queue (DLQ)

```
Normal Flow:
Event published
  ├─ Processing succeeds
  └─ Marked as processed

Error Flow:
Event published
  ├─ Processing fails (retry 3 times)
  ├─ All retries failed
  └─ Moved to DLQ

Dead Letter Queue:
├── Contains failed events
├── Requires manual intervention
├── Alerting triggered
├── Operator investigates
├── Fix applied
└── Event reprocessed

Example:
Event: OrderShipped
Error: Shipping service not found
  ├─ Retry 1: Still not found
  ├─ Retry 2: Still not found
  ├─ Retry 3: Still not found
  └─ Move to DLQ

Operator:
  ├─ Notices alert
  ├─ Shipping service back up
  ├─ Manually republish event
  └─ Event reprocessed successfully
```

## ✅ Event-Driven Checklist

```
Design:
- [ ] Events clearly defined (what happened?)
- [ ] Event schema versioning
- [ ] Event ordering requirements clear
- [ ] Idempotency implemented
- [ ] Error handling strategy defined

Implementation:
- [ ] Event publisher/subscriber established
- [ ] Kafka/RabbitMQ cluster running
- [ ] Event serialization (JSON/Avro/Protobuf)
- [ ] Event versioning handled
- [ ] Monitoring of event flow

Operations:
- [ ] Dead letter queue monitored
- [ ] Consumer lag tracked
- [ ] Event retention configured
- [ ] Scaling strategy for peak load
- [ ] Disaster recovery plan

Monitoring:
- [ ] Event count per topic
- [ ] Consumer lag
- [ ] Processing latency
- [ ] Error rate per consumer
- [ ] DLQ size
```

---

**Key Principle:** "Events are the foundation of scalable systems. They decouple services, enable real-time processing, and provide a complete audit trail."
