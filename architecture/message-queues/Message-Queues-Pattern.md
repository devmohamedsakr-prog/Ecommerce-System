# Message Queues Pattern

**Status:** Architecture Pattern | **Priority:** CRITICAL | **Throughput:** 100K-1M messages/sec

---

## 📋 Overview

Message queues decouple services through asynchronous communication. Instead of Service A waiting for Service B to respond (blocking), Service A sends message to queue, continues immediately. Service B processes message whenever ready. Enables reliability, scalability, and fault tolerance.

## 🎯 Business Problem

- Synchronous calls cause cascading failures (one slow service blocks everything)
- Retry logic complex to implement
- Services tightly coupled (change one, break others)
- High-traffic events overwhelm downstream services
- Example: Order placed → Send email → Update analytics → Inventory update
  - If email service slow, entire order flow blocks
  - If email fails, order fails (bad UX)

## 🏗️ Architecture

```
┌─────────────────────────────────────┐
│   Service A (Order Service)          │
│   Receives order request             │
└────────────┬────────────────────────┘
             │
             │ Publish: order.created
             ↓
    ┌────────────────────┐
    │  Message Queue     │
    │  (Kafka/RabbitMQ)  │
    │  order.created msg │
    └────────┬───────────┘
             │
    ┌────────┴───────────────────┐
    │                            │
    ↓                            ↓
┌─────────────┐      ┌─────────────────┐
│Email Service│      │Analytics Service│
│(Subscribe)  │      │(Subscribe)      │
│Sends email  │      │Logs event       │
└─────────────┘      └─────────────────┘
```

## Message Queue Types

### 1. Point-to-Point (Queue)
```
One producer, one consumer
Message consumed by single service

Order placed → order-queue → Email service (only consumer)

Best for: Task delegation, load balancing
Guarantees: Exactly-once delivery per message
```

### 2. Publish-Subscribe (Topic)
```
One producer, many consumers
Message delivered to all subscribers

Order placed → orders-topic → Email, Analytics, Inventory (all receive)

Best for: Event broadcasting
Guarantees: Broadcast to all subscribers
```

## 📡 Message Queue Systems

### Apache Kafka (Distributed, High-Throughput)
```
Architecture:
- Topics (event streams)
- Partitions (parallel processing)
- Brokers (distributed cluster)
- Consumer groups (scale horizontally)

Characteristics:
- 1M+ messages/sec throughput
- Persistent (messages stored for days)
- Distributed (survives broker failures)
- Ordered per partition (not globally)

Best for: High-volume event streaming
Examples: Order events, user actions, product updates
```

### RabbitMQ (Reliable Message Broker)
```
Architecture:
- Queues (message storage)
- Exchanges (routing logic)
- Bindings (route queue to exchange)
- Acknowledgments (reliability)

Characteristics:
- 50K messages/sec throughput
- Reliable delivery (ACKs)
- Multiple routing patterns
- AMQP protocol

Best for: Critical workflows
Examples: Payment processing, email sending
```

## 🔄 Message Flow Patterns

### Fire and Forget (Unreliable)
```
Service A sends message
Service A doesn't wait for confirmation
Message might be lost

Used for: Analytics, non-critical logging
Risk: Message loss
```

### Synchronous ACK (Reliable)
```
Service A sends message
Waits for broker confirmation
Service A continues only after ACK

Used for: Critical transactions
Risk: Slower (wait for ACK)
Benefit: Guaranteed delivery
```

### Async Processing
```
1. Client sends request to Service A
2. Service A returns immediately
3. Service A queues task asynchronously
4. Consumer processes task
5. Sends result via callback/notification

Used for: Long-running tasks (email, reports, data processing)
User experience: Immediate response, background processing
```

## 💾 Message Structure

```
MESSAGE
├── message_id (UUID: unique identifier)
├── timestamp (when sent)
├── topic (string: "orders", "users", "payments")
├── partition_key (string: customer_id for ordering)
├── headers (object: correlation_id, user_id, etc)
├── body (JSON: message content)
│   ├── event_type (enum)
│   ├── entity_id (string)
│   ├── payload (object: event details)
│   └── timestamp (when event occurred)
├── source_service (string: "order-service")
├── ttl_seconds (number: message expiry)
└── retry_count (number: delivery attempts)

CONSUMER_OFFSET
├── consumer_group (string)
├── topic (string)
├── partition (number)
├── offset (number: which message to consume next)
└── last_updated (timestamp)
```

## 📊 Ordering Guarantees

### No Ordering
```
Messages processed in any order
Message 1: Update price to $100
Message 2: Update price to $50
Result: Could be $100 or $50

Best for: Independent events (likes, views)
```

### Partition-Level Ordering (Kafka)
```
Messages with same partition_key go to same partition
Partition processed sequentially (in order)

partition_key = customer_id
Messages for customer 1 always in same partition
Messages for customer 2 always in different partition
Different customers can process in parallel

Best for: Order events, user activity
Guarantees: FIFO per customer
```

### Global Ordering
```
All messages processed strictly in order
Implementation: Single partition or single consumer

Best for: Critical transactions (financial ledger)
Trade-off: Can't parallelize (slower)
```

## 🔧 Example: Kafka Order Processing

```python
# Producer (Order Service)
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Publish order event
order = {
    'order_id': '12345',
    'customer_id': 'cust_789',
    'total': 99.99,
    'timestamp': '2026-08-05T10:00:00Z'
}

producer.send(
    'orders',
    value=order,
    key=str(order['customer_id']).encode()  # partition by customer
)

# Consumer (Email Service)
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='email-service',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    order = message.value
    # Send email
    send_order_confirmation_email(order)
    # Commit offset (marks as processed)
    consumer.commit()
```

## 🎯 Dead Letter Queue (DLQ)

```
Problem: Consumer crashes, message can't be processed

Solution: Dead Letter Queue
1. Message fails processing after 3 retries
2. Message moved to DLQ (special queue)
3. Alert sent to ops team
4. Ops investigates and fixes issue
5. Message manually replayed

Guarantees: No message loss, critical errors surfaced
```

## 🔗 Typical Ecommerce Events

```
Order Events:
- order.created (→ email, inventory, analytics)
- order.payment_confirmed (→ fulfillment, accounting)
- order.shipped (→ customer notification, tracking)
- order.delivered (→ review request, loyalty points)

User Events:
- user.signed_up (→ welcome email, analytics)
- user.logged_in (→ session, security audit)

Product Events:
- product.created (→ search index, catalog)
- product.price_changed (→ caching, analytics)

Payment Events:
- payment.failed (→ retry queue, customer notification)
- payment.succeeded (→ order processing)
```

## 📈 Scalability

### Horizontal Scaling (Kafka)
```
Initial: 1 broker, 1 consumer
- Throughput: 50K messages/sec
- Latency: 100ms

Scale to: 3 brokers, 6 consumers (2 per partition)
- Throughput: 500K messages/sec
- Latency: 50ms
- Cost: 3x but 10x throughput

Partitions = parallelism
Topics = independent streams
```

## 🔗 Integration Points

- **Order Service** - Publishes order events
- **Email Service** - Subscribes to send emails
- **Analytics Service** - Subscribes to log events
- **Inventory Service** - Subscribes to update stock
- **Notification Service** - Subscribes for all notifications
- **Accounting Service** - Subscribes for revenue recognition

## ⚠️ Common Pitfalls

1. **Message Loss** - Network failure, broker crash loses messages
   - Solution: Persistent queues (Kafka), ACKs, replication

2. **Duplicate Processing** - Consumer processes same message twice
   - Solution: Idempotent operations, unique message IDs

3. **Ordering Issues** - Messages processed out of order
   - Solution: Partition keys (Kafka), single consumer

4. **Backlog Growth** - Consumers can't keep up with producers
   - Solution: Scale consumers horizontally, monitor lag

5. **Message Explosion** - Too many message types, system becomes complex
   - Solution: Domain events, canonical event format

---

**Pattern Version:** 1.0 | **Status:** Production Pattern | **Typical Benefit:** 10-100x throughput improvement

