# REST API Design Best Practices

## 📋 Overview

Guidelines for designing scalable, maintainable REST APIs for e-commerce systems.

## 🎯 Core Principles

### 1. Resource-Oriented Design

Think in terms of resources, not actions.

**❌ Bad (RPC-style)**
```
GET /getOrders
POST /createOrder
PUT /updateOrder
DELETE /removeOrder
```

**✅ Good (REST)**
```
GET    /api/v1/orders              # List orders
POST   /api/v1/orders              # Create order
GET    /api/v1/orders/:id          # Get specific order
PATCH  /api/v1/orders/:id          # Partial update
DELETE /api/v1/orders/:id          # Delete order
```

### 2. HTTP Methods

```
GET     - Retrieve resource (safe, idempotent)
POST    - Create resource
PUT     - Replace entire resource (idempotent)
PATCH   - Partial update
DELETE  - Remove resource (idempotent)
HEAD    - Like GET but no body
OPTIONS - Describe communication options
```

### 3. Stateless Operations

Each request must contain all information needed.

**❌ Bad (Stateful)**
```
POST /login → Session created
GET /orders → Relies on session
```

**✅ Good (Stateless)**
```
POST /login → JWT token returned
GET /orders → Authorization: Bearer <token>
```

## 📊 URL Structure

### Base URL
```
https://api.example.com/api/v1
```

### Resource URLs
```
/orders
/orders/:id
/orders/:id/items
/orders/:id/items/:itemId
/customers/:customerId/orders
```

### Query Parameters
```
GET /orders?status=pending&limit=20&offset=0
GET /products?category=electronics&sort=-price&filter[brand]=samsung
```

## 📝 Response Format

### Success Response (200 OK)

```json
{
  "success": true,
  "data": {
    "id": "order-123",
    "customerId": "cust-456",
    "status": "confirmed",
    "total": 299.99,
    "items": [
      {
        "id": "item-1",
        "productId": "prod-789",
        "quantity": 2,
        "price": 149.99
      }
    ],
    "createdAt": "2026-08-04T10:30:00Z",
    "updatedAt": "2026-08-04T10:30:00Z"
  }
}
```

### List Response with Pagination

```json
{
  "success": true,
  "data": [
    { "id": "order-123", "status": "confirmed" },
    { "id": "order-124", "status": "shipped" }
  ],
  "pagination": {
    "total": 1000,
    "limit": 20,
    "offset": 0,
    "hasMore": true
  }
}
```

### Error Response (4xx/5xx)

```json
{
  "success": false,
  "error": {
    "code": "PAYMENT_FAILED",
    "message": "Payment authorization declined",
    "details": {
      "reason": "insufficient_funds",
      "retryable": true
    }
  }
}
```

## 🔐 Authentication & Authorization

### Header-Based (JWT)

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Refresh Token Pattern

```
POST /auth/refresh
Body: { "refreshToken": "..." }
Response: { "accessToken": "...", "expiresIn": 3600 }
```

### Scope-Based Authorization

```
Headers:
- X-Request-ID: unique-id-123 (for tracing)
- Authorization: Bearer <token>
```

## 🚀 Versioning Strategy

### URL-Based Versioning (Recommended)

```
GET /api/v1/orders        # Version 1
GET /api/v2/orders        # Version 2
```

### Header-Based Versioning

```
Accept: application/vnd.example.v1+json
Accept: application/vnd.example.v2+json
```

## ⏱️ Rate Limiting

Headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 987
X-RateLimit-Reset: 1628000000
```

Implementation:
```
- Per IP: 1000 req/hour
- Per User: 10000 req/hour
- Per API Key: 100000 req/hour
```

## 🔄 Idempotency

For POST requests that should be retryable:

Request:
```
POST /api/v1/orders
Idempotency-Key: unique-key-123
Content-Type: application/json

{
  "items": [...]
}
```

Response:
```
HTTP 201 Created
Idempotency-Key: unique-key-123
{
  "id": "order-123"
}
```

Server ensures same Idempotency-Key returns same response.

## ❌ Error Codes

Standard HTTP codes:

| Code | Meaning | Example |
|------|---------|---------|
| 200 | OK | Successful GET/PUT/PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid parameters |
| 401 | Unauthorized | Missing/invalid token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate order ID |
| 422 | Unprocessable Entity | Invalid data format |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Server Error | Unexpected error |
| 503 | Service Unavailable | Maintenance |

## 📦 Pagination

### Offset/Limit Pattern

```
GET /api/v1/orders?limit=20&offset=40

Response:
{
  "data": [...],
  "pagination": {
    "limit": 20,
    "offset": 40,
    "total": 1000,
    "hasMore": true
  }
}
```

### Cursor-Based Pagination (Better for scale)

```
GET /api/v1/orders?limit=20&cursor=abc123

Response:
{
  "data": [...],
  "pagination": {
    "cursor": "xyz789",
    "hasMore": true
  }
}
```

## 🔗 Relationships

### Embedded Resources

```json
{
  "id": "order-123",
  "customer": {
    "id": "cust-456",
    "name": "John Doe"
  }
}
```

### Related Links (HATEOAS)

```json
{
  "id": "order-123",
  "status": "confirmed",
  "_links": {
    "self": { "href": "/orders/order-123" },
    "customer": { "href": "/customers/cust-456" },
    "items": { "href": "/orders/order-123/items" }
  }
}
```

## 🔍 Filtering & Sorting

### Filtering

```
GET /products?category=electronics&price_min=100&price_max=500

GET /orders?status=pending,confirmed&created_after=2026-01-01
```

### Sorting

```
GET /products?sort=price                    # Ascending
GET /products?sort=-price                   # Descending
GET /products?sort=-rating,price            # Multiple fields
```

## 📐 API Metrics

Track and monitor:

```
- Response time (p50, p95, p99)
- Error rate by endpoint
- Success rate
- Requests per second
- Unique users
- Cache hit rate
```

## 🛡️ Security Headers

Always include:

```
Content-Security-Policy: default-src 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000
```

## 📚 API Documentation

Use OpenAPI/Swagger:

```yaml
openapi: 3.0.0
info:
  title: E-Commerce API
  version: 1.0.0

paths:
  /orders:
    get:
      summary: List orders
      parameters:
        - name: status
          in: query
          schema:
            type: string
      responses:
        '200':
          description: Orders retrieved
```

## ✅ Checklist

- [ ] Resources, not actions
- [ ] Proper HTTP methods
- [ ] Consistent URL structure
- [ ] Standard error responses
- [ ] Authentication/authorization
- [ ] Rate limiting headers
- [ ] Idempotency for POST
- [ ] Pagination for lists
- [ ] Versioning strategy
- [ ] Comprehensive documentation
- [ ] Security headers
- [ ] Monitoring and logging

---

**Apply These:**
1. Start with v1 versioning
2. Use JWT for auth
3. Implement rate limiting early
4. Monitor all endpoints
5. Version before breaking changes
