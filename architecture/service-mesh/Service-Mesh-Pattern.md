# Service Mesh Pattern

**Status:** Architecture Pattern | **Priority:** HIGH | **Complexity:** Advanced

---

## Overview

Service Mesh manages service-to-service communication at infrastructure level, not application code. Handles retries, timeouts, circuit breakers, load balancing, security policies, observability without changing service code. Typical: Istio or Linkerd.

## Problem Solved

Without service mesh: Each service implements:
- Retry logic
- Timeout handling
- Circuit breaker pattern
- Load balancing
- Mutual TLS
- Distributed tracing
- Rate limiting

Result: Boilerplate everywhere, inconsistent implementations, hard to test

With service mesh: Infrastructure handles all of this uniformly

## Architecture

```
┌────────────────────────────────────────────┐
│          Kubernetes Cluster                │
├────────────────────────────────────────────┤
│                                            │
│  ┌──────────────┐       ┌──────────────┐ │
│  │ Order        │       │ Payment      │ │
│  │ Service Pod  │───────│ Service Pod  │ │
│  │ + Sidecar    │  ↓ ↓  │ + Sidecar    │ │
│  └──────────────┘       └──────────────┘ │
│       ↓                         ↓         │
│  Sidecar proxies               │         │
│  (Envoy)                       │         │
│  Handle all networking         │         │
│                                ↓         │
│  ┌─────────────────────────────────────┐ │
│  │   Service Mesh Control Plane        │ │
│  │   (Istio Pilot / Linkerd Controller)│ │
│  │   - Route policies                  │ │
│  │   - Circuit breakers                │ │
│  │   - Load balancing rules            │ │
│  └─────────────────────────────────────┘ │
│                                            │
└────────────────────────────────────────────┘
```

## Key Capabilities

### 1. Traffic Management
```
Route by hostname:
- api.example.com → api-service
- www.example.com → web-service

Route by header:
- X-User-Type: premium → premium-service
- X-User-Type: free → free-service

Canary deployments:
- 10% traffic → new version
- 90% traffic → stable version
- Monitor metrics, gradually increase
```

### 2. Resilience

**Retry Policy**
```
If request fails:
  Retry up to 3 times
  Exponential backoff: 100ms, 200ms, 400ms
  Total timeout: 10 seconds
```

**Circuit Breaker**
```
If service fails:
  Fail 5 consecutive requests → Open circuit
  Stop sending requests (fail fast)
  After 30 seconds, try one request (half-open)
  If succeeds, close circuit and resume traffic
```

**Timeout Management**
```
All service calls have timeout:
  - Connect timeout: 5s
  - Request timeout: 30s
  - If exceeded, fail and retry

Prevents: One slow service hanging all requests
```

### 3. Security (mTLS)

```
Every service-to-service call encrypted:
1. Service A initiates TLS handshake
2. Sidecar proxy handles TLS
3. Service B sidecar validates cert
4. Encrypted tunnel established
5. Application layer unaware of TLS

Benefit: Zero-trust networking, no plaintext communication
Automatic: No code changes needed
```

### 4. Observability

**Distributed Tracing**
```
Request flows through:
- Order Service → Payment Service → Inventory Service
Each hop recorded with latency
Entire trace visible in dashboard

Problem: Which service is slow?
Solution: Trace shows payment-service taking 2s (bottleneck)
```

**Metrics Collection**
```
For every request:
- Success rate (%)
- Latency (p50, p99)
- Error rate (%)
- Throughput (req/sec)

Automatic collection, no instrumentation needed
```

**Service Graph**
```
Visualize dependencies:
- Order Service → Payment Service
- Order Service → Inventory Service
- Payment Service → Bank (external)

Shows: Topology, error rates, latencies
Instant view: What depends on what
```

## Typical Policies

### VirtualService
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - match:
    - uri:
      prefix: "/api/v2"
    route:
    - destination:
        host: payment-service
        port:
          number: 3000
        subset: v2
      weight: 10  # 10% canary
    - destination:
        host: payment-service
        subset: v1
      weight: 90  # 90% stable
```

### DestinationRule
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
```

## Comparison: Istio vs Linkerd

| Feature | Istio | Linkerd |
|---------|-------|---------|
| **Complexity** | High | Low |
| **Learning curve** | Steep | Gentle |
| **Resource usage** | 4GB+ | 100MB |
| **Features** | Comprehensive | Core |
| **Best for** | Large enterprises | Smaller teams |
| **Cost** | $0 (OSS) | $0 (OSS) |

## Implementation Approach

### Phase 1: Observe (Non-Intrusive)
```
Deploy service mesh in observe-only mode
Collect metrics, no traffic changes
Verify nothing breaks
Baseline: Current behavior vs mesh behavior
Duration: 1-2 weeks
```

### Phase 2: Inject (Gradual)
```
Enable circuit breaker on non-critical service
Monitor for issues
Expand to more services
Track improvements
Duration: 2-4 weeks
```

### Phase 3: Enforce (Full)
```
Enable all policies
mTLS enforcement
Rate limiting
Full traffic management
Duration: 4-8 weeks
```

## Monitoring

```
Key Metrics:
- Request success rate (target: 99%+)
- Latency p99 (target: < 100ms)
- Circuit breaker open events (should be rare)
- Retry rate (target: < 2%)

Dashboard should show:
- Service dependency graph
- Error rates by service
- Latencies by percentile
- Traffic distribution (canary weight)
```

## When NOT to Use Service Mesh

```
- Small team (< 10 engineers): Too complex
- Simple architecture (< 10 services): Overhead not justified
- Real-time requirements (< 10ms latency): Proxy adds latency
- Resource-constrained (embedded, IoT): Uses too much memory

Start with: Observability, then add capabilities gradually
```

---

**Pattern Version:** 1.0 | **Status:** Production Pattern | **Typical Benefit:** 40% reduction in operational complexity

