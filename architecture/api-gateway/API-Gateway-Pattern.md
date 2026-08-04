# API Gateway Pattern

## 📋 Overview

The API Gateway is the single entry point for all client requests in a microservices architecture.

## 🎯 Responsibilities

```
Before API Gateway:
┌─────────────────────────────────────┐
│ Internet                            │
└─────────────────────────────────────┘
    │         │         │         │
    ▼         ▼         ▼         ▼
┌────────┬─────────┬─────────┬──────────┐
│Order   │ Payment │Inventory│Shipping  │
│Service │ Service │ Service │ Service  │
└────────┴─────────┴─────────┴──────────┘

Problems:
├── Each service handles auth separately
├── Each service handles rate limiting
├── Each service handles logging
├── Each service must be PCI compliant
├── Clients must know all service URLs
└── Cross-cutting concerns duplicated

After API Gateway:
┌─────────────────────────────────────┐
│ Internet                            │
└─────────────────────────────────────┘
            │
            ▼
    ┌──────────────────┐
    │  API Gateway     │
    │                  │
    │ ├─ Auth          │ (Centralized)
    │ ├─ Rate Limit    │ (Centralized)
    │ ├─ Logging       │ (Centralized)
    │ ├─ SSL/TLS       │ (Centralized)
    │ └─ Routing       │ (Centralized)
    └──────────────────┘
            │
    ┌───────┴────────┬────────────────┐
    ▼                ▼                ▼
Order Service   Payment Service   Inventory Service
(Internal only) (Internal only)   (Internal only)
```

## 🔌 Core Features

### 1. Request Routing

```
API Gateway receives request:
GET /api/products

Routing rules:
└── /api/products → Catalog Service:8000
└── /api/orders → Order Service:8001
└── /api/payments → Payment Service:8002
└── /api/inventory → Inventory Service:8003

Gateway forwards:
GET http://catalog-service:8000/products

Benefits:
├── Single URL for clients
├── Services behind firewall
├── Load balance across replicas
└── Service location transparent
```

### 2. Authentication & Authorization

```
Client Request:
GET /api/orders
Authorization: Bearer eyJhbGc...

API Gateway:
1. Extract token from header
2. Verify JWT signature
3. Check token expiration
4. Extract user ID and permissions
5. Add to request headers

Forwarded to Service:
GET /api/orders
Authorization: Bearer eyJhbGc...
X-User-ID: user-123
X-User-Role: customer
X-User-Permissions: read:orders,create:orders

Benefits:
├── Services don't validate tokens
├── Centralized auth logic
├── Services trust gateway headers
└── Easy to change auth strategy
```

### 3. Rate Limiting

```
per-User Rate Limiting:
Limit: 100 requests/hour per user

Request 1: ✓ (1/100)
Request 2: ✓ (2/100)
...
Request 100: ✓ (100/100)
Request 101: ✗ Rejected (Rate limited)

Response:
HTTP 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1628000000

per-API Endpoint Rate Limiting:
GET /api/search: 1000/sec
POST /api/orders: 100/sec
GET /api/products: 10000/sec

Benefits:
├── Prevent abuse
├── Fair resource allocation
├── Protect backend services
└── Monetization lever
```

### 4. Request/Response Transformation

```
Client Request (JavaScript):
{
  "productIds": [1, 2, 3],
  "quantity": 5
}

API Gateway Transform:
{
  "product_ids": [1, 2, 3],  # snake_case for backend
  "quantity": 5
}

Service Response (Internal):
{
  "order_id": "order-123",
  "total_amount": 99.99,
  "created_at": "2026-08-04T10:30:00Z"
}

API Gateway Transform:
{
  "orderId": "order-123",    # camelCase for frontend
  "totalAmount": 99.99,
  "createdAt": "2026-08-04T10:30:00Z"
}

Client Response (JavaScript):
{
  "orderId": "order-123",
  "totalAmount": 99.99,
  "createdAt": "2026-08-04T10:30:00Z"
}

Benefits:
├── Backend uses standard naming
├── Frontend uses preferred naming
├── Contract versioning
└── Field hiding/redacting
```

### 5. Response Aggregation

```
Client needs: Order + Customer + Shipping info

Microservices approach:
// Request 1: Get order
GET /orders/123 → Order Service
Response: { orderId, total, customerId }

// Request 2: Get customer
GET /customers/cust-456 → User Service
Response: { customerId, name, email, address }

// Request 3: Get shipping
GET /shipments/ship-789 → Shipping Service
Response: { shipmentId, trackingNumber, status }

Client must make 3 requests!

API Gateway approach:
// Single request
GET /orders/123 → API Gateway

// Gateway makes 3 parallel requests internally
POST /internal/composite-order/123
├─ Call Order Service
├─ Call User Service (with customerIdFromOrder)
└─ Call Shipping Service (with shipmentIdFromOrder)

// Aggregated response
{
  "order": { orderId, total, customerId },
  "customer": { customerId, name, email, address },
  "shipping": { shipmentId, trackingNumber, status }
}

Benefits:
├── Single request (fewer round trips)
├── Parallel internal calls
├── Optimized for client needs
└── Reduced latency
```

### 6. Protocol Translation

```
Client → API Gateway → Internal Services

Client speaks REST:
GET /api/products

Gateway translates to gRPC:
Service grpc_service {
  rpc GetProducts(GetProductsRequest) returns (GetProductsResponse)
}

Benefits:
├── Clients use REST (simple)
├── Services use gRPC (fast, binary)
├── Legacy system support
└── Protocol evolution
```

## 🏗️ API Gateway Architecture

### Single Gateway (Small Scale)

```
┌──────────────┐
│ Internet     │
└──────────────┘
       │
       ▼
┌──────────────┐
│ API Gateway  │
│ Single node  │
└──────────────┘
       │
   ┌───┴───┐
   ▼       ▼
Service   Service

Limitations:
├── Single point of failure
├── Limited throughput
├── No high availability
```

### Load-Balanced Gateways (Medium Scale)

```
┌─────────────┐
│ Internet    │
└─────────────┘
       │
       ▼
  ┌────────────┐
  │Load        │
  │Balancer    │
  │(DNS/ALB)   │
  └────────────┘
       │
   ┌───┼───┐
   ▼   ▼   ▼
 ┌─┐ ┌─┐ ┌─┐
 │G│ │G│ │G│ (API Gateway instances)
 └─┘ └─┘ └─┘
   │   │   │
   └───┴───┘
       │
   ┌───┴────────┬──────────┐
   ▼            ▼          ▼
Service A    Service B   Service C

Configuration:
├── 3 gateway instances
├── Active-active (all handle traffic)
├── Load balancer distributes
├── Shared cache (Redis)
└── Rate limit store (Redis)

Benefits:
├── High availability (one fails, others handle)
├── Better throughput (3x)
├── Health checks
└── Automatic failover
```

### Regional Gateways (Global Scale)

```
Client in US → US Gateway
Client in EU → EU Gateway
Client in APAC → APAC Gateway

┌─────────────────────────────────┐
│         Global Load Balancer    │
│         (Route 53 / Anycast)    │
└─────────────────────────────────┘
    │              │              │
    ▼              ▼              ▼
US Gateway      EU Gateway    APAC Gateway
│               │              │
├─ Auth         ├─ Auth       ├─ Auth
├─ Rate limit   ├─ Rate limit ├─ Rate limit
└─ Route        └─ Route      └─ Route
    │               │          │
    ▼               ▼          ▼
US Services    EU Services  APAC Services

Benefits:
├── Reduced latency (nearby gateway)
├── Regional rate limits
├── Data residency compliance
├── Regional failover
```

## 🔒 Security

### TLS Termination

```
Internet ────────────────────────> API Gateway
         (HTTPS, TLS 1.3)                 │
                                          ▼
                              Internal services
                                  (HTTP)

Benefits:
├── Clients use HTTPS (secure)
├── Gateway handles certificate
├── Services use simple HTTP
├── Certificate rotation centralized
└── Performance (TLS handshake expensive)
```

### API Keys

```
Client registers:
POST /auth/register
→ API Key generated: sk_live_abc123def456

All requests include key:
GET /api/products
X-API-Key: sk_live_abc123def456

Gateway validates:
1. Key exists
2. Key not revoked
3. Key rate limit not exceeded
4. Key has required permissions

Benefits:
├── Simple authentication
├── Per-app rate limiting
├── Easy key rotation
└── Audit trail
```

## 📊 Monitoring

### Key Metrics

```
Request Metrics:
├── Total requests/sec
├── Requests per endpoint
├── Request latency (p50, p95, p99)
├── Request size (bytes)
└── Response size (bytes)

Error Metrics:
├── 4xx errors (client errors)
├── 5xx errors (server errors)
├── Rate limit rejections
├── Auth failures
└── Timeout errors

Performance:
├── Gateway latency
├── Backend latency
├── Overhead (gateway + backend)
└── Cache hit rate

Throughput:
├── Requests/sec by endpoint
├── Bandwidth usage
├── Concurrent connections
└── Connection pool usage
```

### Common Issues

```
Issue 1: High Latency
Debug:
├── Is gateway slow? (check GW latency)
├── Is backend slow? (check backend latency)
├── Is network slow? (check p99 latency)
└── Is aggregation slow? (parallel calls?)

Solution:
├── Add more gateway instances
├── Optimize backend service
├── Implement response caching
└── Use connection pooling
```

## ✅ API Gateway Checklist

```
Functionality:
- [ ] Request routing configured
- [ ] Authentication working
- [ ] Rate limiting active
- [ ] Response caching enabled
- [ ] Error handling defined

Performance:
- [ ] Latency within SLA
- [ ] Throughput meets requirements
- [ ] Cache hit rate > 80%
- [ ] Connection pooling enabled
- [ ] Load balancing verified

Security:
- [ ] TLS/SSL configured (modern versions)
- [ ] Authentication enforced
- [ ] Authorization checked
- [ ] Rate limiting prevents abuse
- [ ] DDoS protection enabled
- [ ] Logging of all requests

Operations:
- [ ] Monitoring alerts configured
- [ ] Graceful shutdown implemented
- [ ] Health checks working
- [ ] Automatic failover tested
- [ ] Log aggregation working
- [ ] Documentation complete
```

---

**Key Principle:** "The API Gateway is both a feature enabler and a bottleneck. Design it for scale, security, and simplicity."
