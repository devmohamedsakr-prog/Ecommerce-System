# Microservices Architecture for E-Commerce

## 📋 Overview

Complete guide to designing, deploying, and operating microservices at scale for e-commerce systems.

## 🎯 When to Use Microservices

### The Right Time

```
Monolith → Growing Pains → Microservices

Timeline:
├── 0-6 months: Single monolith (1-2 engineers)
│   └── Fast development, no complexity overhead
├── 6-18 months: Monolith hitting limits (5-10 engineers)
│   ├── Hard to deploy (one bug breaks everything)
│   ├── Hard to scale (must scale whole app)
│   ├── Hard to maintain (large codebase)
│   └── Hard to test (integrated dependencies)
└── 18+ months: Microservices (20+ engineers)
    ├── Independent deployments
    ├── Independent scaling
    ├── Multiple teams (one per service)
    └── Separate databases per service
```

### Anti-Patterns (When NOT to Use)

```
❌ Too Early:
├── Team too small (< 10 engineers)
├── Services poorly understood (domain unclear)
├── Requirements changing rapidly
└── Not enough monitoring/infrastructure

❌ Too Late:
├── Monolith too tightly coupled
├── Database too centralized
├── Team lacking DevOps skills
└── Infrastructure immature
```

## 🏗️ Microservices Design

### Service Decomposition

```
By Business Capability (Recommended):

┌─────────────────────────────────────┐
│         E-Commerce Platform         │
├─────────────────────────────────────┤
│                                     │
│  ┌──────────────┐  ┌────────────┐  │
│  │Order Team    │  │Payment Team│  │
│  ├──────────────┤  ├────────────┤  │
│  │Order Service │  │Payment Svc │  │
│  ├──────────────┤  ├────────────┤  │
│  │Order DB      │  │Payment DB  │  │
│  └──────────────┘  └────────────┘  │
│                                     │
│  ┌──────────────┐  ┌────────────┐  │
│  │Inventory Team│ │Shipping Team│ │
│  ├──────────────┤  ├────────────┤  │
│  │Inventory Svc │  │Shipping Svc│  │
│  ├──────────────┤  ├────────────┤  │
│  │Inventory DB  │  │Shipping DB │  │
│  └──────────────┘  └────────────┘  │
│                                     │
└─────────────────────────────────────┘

Team Structure Mirrors Service Structure!
```

### Service Boundaries

```
Good Boundaries:
├── Order Service owns:
│   ├── Order creation
│   ├── Order status
│   ├── Order history
│   └── Order cancellation
│
├── Payment Service owns:
│   ├── Authorization
│   ├── Capture
│   ├── Refunds
│   └── Transactions
│
└── Inventory Service owns:
    ├── Stock levels
    ├── Reservations
    ├── Fulfillment
    └── Returns

Bad Boundaries:
├── ❌ Order Service owns both Order AND Payment
│   (Too many responsibilities)
├── ❌ Split Order Service by function (Create, Update, Delete)
│   (Hard to maintain, poor cohesion)
└── ❌ Order Service owns Order + Cart + Wishlist
    (Unrelated concepts)
```

## 🔌 Service Communication

### Synchronous (REST/gRPC)

**REST API Example:**

```
Order Service → Inventory Service

Request:
POST /api/v1/inventory/reserve
{
  "orderId": "order-123",
  "items": [
    { "productId": "prod-456", "quantity": 2 }
  ]
}

Response:
{
  "reservationId": "rsv-789",
  "items": [
    {
      "productId": "prod-456",
      "reserved": 2,
      "available": 150
    }
  ],
  "expiresAt": "2026-08-04T11:30:00Z"
}

Characteristics:
✅ Immediate feedback
✅ Strong consistency
✅ Simple debugging
❌ Tight coupling
❌ Cascading failures
❌ Network dependency
```

**gRPC (Binary Protocol):**

```protobuf
// inventory.proto
service InventoryService {
  rpc ReserveInventory(ReserveRequest) returns (ReserveResponse);
}

message ReserveRequest {
  string order_id = 1;
  repeated Item items = 2;
}

message Item {
  string product_id = 1;
  int32 quantity = 2;
}

message ReserveResponse {
  string reservation_id = 1;
  repeated ItemStatus items = 2;
  string expires_at = 3;
}

message ItemStatus {
  string product_id = 1;
  int32 reserved = 2;
  int32 available = 3;
}

Characteristics:
✅ Faster than REST (binary encoding)
✅ Strongly typed
✅ Better for internal APIs
❌ Not browser-friendly
❌ Not human-readable
```

### Asynchronous (Event-Driven)

**Kafka Event Example:**

```
Order Service publishes:
OrderConfirmed event
  ├── Event: OrderConfirmed
  ├── OrderId: order-123
  ├── CustomerId: cust-456
  ├── Amount: $99.99
  └── Timestamp: 2026-08-04T10:30:00Z

Subscribers:
├── Inventory Service consumes → Decrements stock
├── Notification Service consumes → Sends email
├── Analytics Service consumes → Updates metrics
└── Recommendation Service consumes → Updates model

Characteristics:
✅ Loose coupling (don't know subscribers)
✅ Fast (fire and forget)
✅ Fault tolerant (retry on failure)
✅ Scales easily (add subscribers)
❌ Eventual consistency
❌ Harder to debug (async flow)
❌ Potential data loss (if queue lost)
```

### Hybrid Approach (Recommended)

```
Decision Tree:
┌─ Synchronous if:
│  ├─ Need immediate result
│  ├─ Critical path (must succeed)
│  ├─ Request-response pattern
│  └─ Data consistency required
│
└─ Asynchronous if:
   ├─ Can handle eventual consistency
   ├─ Non-critical side effects
   ├─ High throughput needed
   ├─ Can retry failures
   └─ Can handle out-of-order events

Example:
Order creation (Synchronous):
  Order Service → Inventory Service (check stock)
  Order Service → Payment Service (authorize)
  ↓ If both succeed
  
Order confirmed (Asynchronous):
  Order Service publishes OrderConfirmed
  ↓ Subscribers process independently
  ├── Inventory Service: Decrement stock
  ├── Notification Service: Send email
  ├── Analytics Service: Update metrics
  └── Recommendation Service: Update model
```

## 📊 Data Management in Microservices

### Database per Service Pattern

```
Traditional Monolith:
┌────────────────────────┐
│     All Services       │
└────────────────────────┘
            │
            ▼
    ┌──────────────┐
    │ Single DB    │
    │ All tables   │
    └──────────────┘

Microservices:
┌─────────────────┐
│ Order Service   │
└─────────────────┘
        │
        ▼
    ┌──────────┐
    │ Order DB │
    └──────────┘

┌─────────────────┐
│ Payment Service │
└─────────────────┘
        │
        ▼
    ┌───────────┐
    │ Payment DB│
    └───────────┘

Benefits:
✅ Services independent (no DB conflicts)
✅ Technology choice per service
✅ Scaling independent
✅ Failure isolation

Challenges:
❌ Cross-service queries hard
❌ Transactions across services hard
❌ Data duplication
❌ Consistency challenges
```

### Distributed Transactions (Saga Pattern)

```
Scenario: Money transfer between accounts

Monolith (ACID transaction):
BEGIN TRANSACTION
  UPDATE accounts SET balance = balance - 100 WHERE id = 1
  UPDATE accounts SET balance = balance + 100 WHERE id = 2
COMMIT

Microservices (Saga pattern - Compensating Transactions):

┌─────────────────────────────────────────────────────┐
│ Step 1: Account A Service                           │
│ Debit 100 from Account A                            │
│ → Success                                           │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ Step 2: Account B Service                           │
│ Credit 100 to Account B                             │
│ → FAILS (Account B not found)                       │
└─────────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────────┐
│ Compensation: Account A Service                     │
│ Credit 100 back to Account A (Rollback)             │
│ → Success                                           │
└─────────────────────────────────────────────────────┘

Result: Both accounts unchanged (consistency maintained)
```

## 🚀 Deployment Patterns

### Independent Deployment

```
Monolith:
Deploy → All services restart → All users affected

Microservices:
Deploy Order Service → Only Order Service restarts
Deploy Payment Service → Only Payment Service restarts

Benefits:
✅ Faster deployments (just one service)
✅ Easier rollbacks (don't affect others)
✅ Reduce deployment risk
✅ Multiple deployments per day
```

### Blue-Green Deployment

```
Current (Blue):
┌─────────────┐
│ Order Svc v1│ ← All traffic
│ (blue)      │
└─────────────┘

New Version (Green):
┌─────────────┐
│ Order Svc v2│ ← No traffic
│ (green)     │
└─────────────┘

Testing:
├── Run tests against green
├── Health checks pass
├── Performance acceptable
└── Ready for traffic

Switch:
    Load Balancer
    
Before: ──→ Blue  (v1)
After:  ──→ Green (v2)

Rollback (if needed):
After:  ──→ Blue  (v1)
```

### Canary Deployment

```
Current Traffic: 100% → Old Service (v1)

Deployment:
Phase 1: Send 5% traffic to v2
├── Monitor error rate
├── Monitor response time
├── Check business metrics
└── If good → Phase 2

Phase 2: Send 25% traffic to v2
├── Monitor error rate
├── Monitor response time
├── Check business metrics
└── If good → Phase 3

Phase 3: Send 50% traffic to v2
├── Monitor error rate
├── Monitor response time
├── Check business metrics
└── If good → Phase 4

Phase 4: Send 100% traffic to v2
├── v1 kept running (fallback)
├── Monitor metrics
└── After 1 hour: Decommission v1

Benefits:
✅ Gradual rollout (catch issues early)
✅ Fast rollback (switch to v1)
✅ Real production traffic tested
✅ Low risk
```

## 🔒 Service-to-Service Security

### API Gateway + Internal APIs

```
Public Internet:
    ↓
┌──────────────────┐
│  API Gateway     │
│  (Public facing) │
└──────────────────┘
    │
    ├─→ Order Service (internal)
    ├─→ Catalog Service (internal)
    └─→ User Service (internal)

Firewall rules:
├── Only API Gateway accepts external traffic
├── Services only accept from API Gateway
├── Services can call each other (internal)
└── No service directly accessible from internet

API Gateway responsibilities:
├── Authentication (verify JWT)
├── Rate limiting
├── Request validation
├── Response aggregation
├── Logging
└── Routing to services
```

### Service-to-Service Authentication (mTLS)

```
Certificate-Based:

Service A → Service B
   │
   ├─ Present Certificate (Service A)
   ├─ Verify Certificate (Service B)
   ├─ Mutual trust established
   └─ Encrypted communication

Configuration:
┌────────────────────────┐
│ Kubernetes Mesh        │
│ (Istio/Linkerd)        │
└────────────────────────┘
   │
   ├─ Automatic mTLS
   ├─ Certificate rotation
   ├─ Policy enforcement
   └─ Audit logging
```

## 📈 Monitoring & Observability

### Service Mesh Observability

```
Service Mesh (Istio):

Request Flow:
Client → Envoy Proxy → Service A
              ↓
         Istio Control Plane
              │
         ┌────┴────┐
         ▼         ▼
      Metrics   Tracing
         │
    ┌────┴────┐
    ▼         ▼
Prometheus Jaeger
    │         │
    ▼         ▼
Grafana    Trace UI

Automatic Metrics:
├── Request count per service
├── Response time (p50, p95, p99)
├── Error rate
├── Request size/response size
├── Connection count
└── TLS certificate status

Automatic Traces:
├── Request latency
├── Service dependencies
├── Error propagation
├── Timeout locations
└── Performance bottlenecks
```

## ✅ Microservices Checklist

```
Design:
- [ ] Clear service boundaries (by business capability)
- [ ] Each service has single responsibility
- [ ] Dependencies documented
- [ ] Data ownership clear
- [ ] API contracts defined (OpenAPI)

Development:
- [ ] Local development setup (Docker)
- [ ] Unit tests for each service
- [ ] Integration tests with dependencies
- [ ] API versioning strategy
- [ ] Error handling consistent

Deployment:
- [ ] CI/CD pipeline per service
- [ ] Independent deployments
- [ ] Health checks implemented
- [ ] Graceful shutdown
- [ ] Configuration externalized

Operations:
- [ ] Centralized logging
- [ ] Distributed tracing
- [ ] Metrics collection
- [ ] Alerting rules
- [ ] Runbooks documented
- [ ] On-call rotation
- [ ] Incident response plan

Scaling:
- [ ] Horizontal scaling possible
- [ ] Load balancing configured
- [ ] Connection pooling
- [ ] Rate limiting
- [ ] Circuit breakers
- [ ] Bulkhead isolation
```

---

**Key Principle:** "Microservices are not free. You pay for independence with operational complexity. Use them when benefits outweigh costs."
