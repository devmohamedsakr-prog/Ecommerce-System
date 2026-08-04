# Production Monitoring & Observability

## 📋 Overview

Complete monitoring strategy for e-commerce systems with 1B+ requests/day.

## 🏗️ Three Pillars of Observability

### 1. Metrics (Quantitative)

```
Application Metrics:
├── Request rate (requests/second)
├── Response time (p50, p95, p99)
├── Error rate (5xx, 4xx)
├── Database latency
├── Cache hit rate
└── Business metrics (orders/min, revenue)

Infrastructure Metrics:
├── CPU utilization
├── Memory usage
├── Disk I/O
├── Network throughput
├── Connections
└── Thread count

Tools: Prometheus, Grafana, DataDog
```

### 2. Logs (Detailed Events)

```
Application Logs:
├── INFO: Normal operations
├── WARN: Unusual but handled
├── ERROR: Errors (retryable)
└── FATAL: Critical errors (unrecoverable)

Example:
{
  "timestamp": "2026-08-04T10:30:00Z",
  "level": "ERROR",
  "service": "order-service",
  "traceId": "abc123def456",
  "message": "Failed to authorize payment",
  "error": "Insufficient funds",
  "orderId": "order-789",
  "customerId": "cust-456",
  "amount": 99.99
}

Tools: ELK Stack, Splunk, Datadog
```

### 3. Traces (Request Flow)

```
Distributed Tracing:

User Request
  ├── API Gateway (50ms)
  │   └── auth check (5ms)
  ├── Order Service (120ms)
  │   ├── create order (30ms)
  │   ├── Inventory Service call (40ms)
  │   │   ├── reserve stock (15ms)
  │   │   ├── database (20ms)
  │   │   └── cache update (5ms)
  │   └── Payment Service call (50ms)
  │       ├── fraud check (20ms)
  │       ├── authorize (25ms)
  │       └── log transaction (5ms)
  └── Response (10ms)

Total: 180ms

Tools: Jaeger, Zipkin, DataDog
```

## 📊 Key Metrics to Monitor

### Frontend Metrics

```
Web Vitals:
├── Largest Contentful Paint (LCP): < 2.5s
├── First Input Delay (FID): < 100ms
├── Cumulative Layout Shift (CLS): < 0.1

Page Load:
├── Time to First Byte (TTFB): < 600ms
├── First Contentful Paint (FCP): < 1.8s
├── Time to Interactive (TTI): < 3.8s

User Actions:
├── Add to cart latency: < 500ms
├── Checkout latency: < 1000ms
├── Payment latency: < 2000ms
```

### Backend Metrics

```
API Performance:
├── Request count: 
├── Response time:
│   ├── p50: < 50ms
│   ├── p95: < 200ms
│   ├── p99: < 500ms
├── Error rate: < 0.1%
└── Throughput: requests/sec

Database:
├── Query time:
│   ├── p50: < 10ms
│   ├── p95: < 50ms
│   ├── p99: < 100ms
├── Replication lag: < 100ms
├── Connection pool:
│   ├── Available: > 20%
│   ├── In use: < 80%
│   └── Waiting: 0

Cache:
├── Hit rate: > 95%
├── Latency: < 2ms
├── Memory usage: < 80%
└── Evictions: 0
```

### Business Metrics

```
Real-Time Dashboard:
├── Orders/minute: Current rate
├── Conversion rate: % completing checkout
├── Cart abandonment: % not purchasing
├── Revenue/minute: $ per minute
├── Average Order Value: $
├── Payment success rate: %
├── Return rate: %
└── Refund rate: %
```

## 🔔 Alerting Strategy

### Severity Levels

```
CRITICAL (Page on-call immediately):
├── Payment processing down
├── Database unreachable
├── API error rate > 5%
├── Response time p99 > 2000ms
└── Revenue dropped > 50%

HIGH (Notify team within 15 min):
├── API error rate > 1%
├── Response time p99 > 500ms
├── Cache hit rate < 90%
├── CPU > 80%
├── Memory > 85%

MEDIUM (Log ticket):
├── Response time p95 > 300ms
├── CPU 60-80%
├── Cache hit rate 90-95%
├── Replication lag > 100ms

INFO (Dashboard only):
├── Response time p50 trending up
├── Memory trending up
├── New deployment status
```

### Alert Rules (Prometheus)

```yaml
# prometheus.yml
groups:
  - name: payment
    interval: 30s
    rules:
      - alert: PaymentServiceDown
        expr: up{job="payment"} == 0
        for: 1m
        severity: critical
        annotations:
          summary: "Payment service is down"

      - alert: PaymentErrorRateHigh
        expr: rate(payment_errors_total[5m]) > 0.05
        for: 5m
        severity: critical
        annotations:
          summary: "Payment error rate {{ $value }}"

  - name: database
    interval: 30s
    rules:
      - alert: DatabaseLatencyHigh
        expr: histogram_quantile(0.99, db_query_duration) > 0.1
        for: 5m
        severity: high
        annotations:
          summary: "Database p99 latency is {{ $value }}s"

      - alert: DatabaseConnectionPoolExhausted
        expr: db_connections_in_use / db_connections_max > 0.95
        for: 2m
        severity: high
        annotations:
          summary: "Connection pool almost exhausted"
```

## 📈 Monitoring Tools

### Prometheus (Metrics Collection)

```
Architecture:
┌──────────────┐
│ Application  │─────────┐
│ exports      │         │
│ metrics      │         │
└──────────────┘         ▼
                    ┌─────────────┐
                    │ Prometheus  │
                    │ Scrapes     │
                    │ every 15s   │
                    └─────────────┘
                         │
            ┌────────────┴────────────┐
            ▼                         ▼
       ┌─────────┐            ┌──────────┐
       │ Grafana │            │ Alert    │
       │Dashboards│           │Manager   │
       └─────────┘            └──────────┘
```

### ELK Stack (Logs)

```
Architecture:
┌──────────────┐
│ Application  │─────────┐
│ writes logs  │         │
└──────────────┘         │
                         ▼
                    ┌─────────────┐
                    │ Filebeat    │
                    │ Collects    │
                    │ logs        │
                    └─────────────┘
                         │
                         ▼
                    ┌─────────────┐
                    │ Logstash    │
                    │ Parses &    │
                    │ enriches    │
                    └─────────────┘
                         │
                         ▼
                    ┌─────────────┐
                    │ Elasticsearch
                    │ Stores logs │
                    │ searchable  │
                    └─────────────┘
                         │
                         ▼
                    ┌─────────────┐
                    │ Kibana      │
                    │ Visualize   │
                    │ logs        │
                    └─────────────┘
```

### Jaeger (Distributed Tracing)

```
Trace Collection:
┌──────────────┐
│ Service A    │
│ spans        │─────────┐
└──────────────┘         │
                         ▼
┌──────────────┐    ┌─────────────┐
│ Service B    │───>│ Jaeger      │
│ spans        │    │ Collector   │
└──────────────┘    └─────────────┘
                         │
┌──────────────┐         │
│ Service C    │─────────┘
│ spans        │    ┌─────────────┐
└──────────────┘    │ Jaeger Query│
                    │ Reconstruct │
                    │ traces      │
                    └─────────────┘
                         │
                         ▼
                    ┌─────────────┐
                    │ UI          │
                    │ Visualize   │
                    │ traces      │
                    └─────────────┘
```

## 🎯 Dashboards

### Real-Time Operations Dashboard

```
┌──────────────────────────────────────────────┐
│  E-COMMERCE SYSTEM - LIVE OPERATIONS         │
├──────────────────────────────────────────────┤
│                                              │
│ ORDERS: 2,450/min ↑ 5% | REVENUE: $410k/min│
│                                              │
├─────────────────────────┬────────────────────┤
│ API Response Time       │ Database Status    │
│ P50: 85ms               │ Latency p99: 45ms │
│ P95: 220ms              │ Connections: 85/100
│ P99: 450ms ↑ from 420ms │ Replication lag: 2ms
│                         │                    │
├─────────────────────────┼────────────────────┤
│ Error Rate              │ Cache Performance  │
│ 0.08% (GOOD) ↓          │ Hit rate: 98.5%    │
│ Last hour: 0.12%        │ Evictions: 0/min   │
│ Errors per endpoint:    │ Memory: 12.5GB/16GB
│ - Payment: 0.05%        │                    │
│ - Order: 0.10%          │                    │
│ - Search: 0.02%         │                    │
│                         │                    │
├─────────────────────────┼────────────────────┤
│ Server Health           │ Traffic Source     │
│ CPU: 65%                │ Web: 60%           │
│ Memory: 72%             │ Mobile: 35%        │
│ Disk I/O: 40%           │ API: 5%            │
│ Network: 55%            │                    │
│                         │                    │
├──────────────────────────────────────────────┤
│ RECENT ALERTS:                               │
│ ⚠️  09:42 - Response time elevated (p95 > 300) 
│ ✅ 08:15 - Deployment successful             │
│ ℹ️  07:50 - Cache eviction rate normal        │
└──────────────────────────────────────────────┘
```

### Business Metrics Dashboard

```
┌──────────────────────────────────────────────┐
│  BUSINESS PERFORMANCE                        │
├──────────────────────────────────────────────┤
│                                              │
│ TODAY'S METRICS:                             │
│ Orders: 125,450 ↑ 12% vs yesterday           │
│ Revenue: $4.2M ↑ 8% vs yesterday             │
│ AOV: $33.50 (avg order value)                │
│ Conversion Rate: 3.2% ↑ from 3.0%            │
│ Cart Abandonment: 68% ↓ from 72%             │
│                                              │
│ PAYMENT SUCCESS:                             │
│ Authorizations: 98.5%                        │
│ Captures: 99.2%                              │
│ Refunds: 1.2% of orders                      │
│ Chargebacks: 0.03%                           │
│                                              │
│ CUSTOMER SATISFACTION:                       │
│ Average Rating: 4.7/5.0                      │
│ Return Rate: 2.3%                            │
│ Support Tickets: 145 (avg response: 2.4hrs)  │
│                                              │
│ TOP PRODUCTS (by revenue):                   │
│ 1. Blue T-Shirt: $45,280                     │
│ 2. Wireless Earbuds: $38,920                 │
│ 3. Phone Case: $32,150                       │
│                                              │
└──────────────────────────────────────────────┘
```

## 🚨 Incident Response

### On-Call Workflow

```
Alert Triggered
    ↓
Page on-call engineer
    ↓
Engineer logs in (2 min)
    ↓
Review dashboard
    ├─ Current metrics
    ├─ Related alerts
    ├─ Recent changes
    └─ Error logs
    ↓
Identify root cause
    ├─ Database issue?
    ├─ Service down?
    ├─ Traffic spike?
    └─ Bad deployment?
    ↓
Implement fix
    ├─ Rollback deployment
    ├─ Scale up
    ├─ Restart service
    ├─ Query optimization
    └─ Clear cache
    ↓
Verify recovery
    ├─ Metrics normalize
    ├─ Error rate drops
    ├─ Alert clears
    └─ Customers unaffected
    ↓
Post-mortem (next day)
    ├─ What happened
    ├─ Why it happened
    ├─ How to prevent
    └─ Action items
```

## ✅ Monitoring Checklist

```
Setup:
- [ ] Prometheus configured
- [ ] Grafana dashboards created
- [ ] ELK stack running
- [ ] Jaeger tracing enabled
- [ ] Alert rules defined
- [ ] On-call rotation set up

Ongoing:
- [ ] All dashboards displayed in office
- [ ] Alert response < 5 minutes
- [ ] Incident post-mortems completed
- [ ] Metrics retention: 30 days min
- [ ] Log retention: 7 days min
- [ ] Traces retention: 48 hours min
- [ ] Regular capacity planning
- [ ] Monthly review of metrics
```

---

**Key Principle:** "If you can't measure it, you can't manage it. Monitor everything, alert intelligently, respond quickly."
