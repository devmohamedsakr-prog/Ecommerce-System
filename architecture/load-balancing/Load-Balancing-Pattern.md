# Load Balancing Pattern

**Status:** Architecture Pattern | **Priority:** CRITICAL | **Scale:** Distribute 100K-1M requests/sec

---

## 📋 Overview

Load balancing distributes traffic across multiple servers to prevent single points of failure and maximize throughput. Without load balancing: one server handles all traffic and fails catastrophically. With load balancing: traffic distributed across 10+ servers, any server can fail without impact.

## 🎯 Business Problem

- Single server can't handle production traffic (bottleneck)
- Server failures cause complete outages (downtime = $9,000/second)
- Uneven traffic distribution causes some servers overloaded, others idle
- Need automatic failover when servers go down
- Cost: 10% improvement in efficiency saves millions at scale

## 🏗️ Architecture

```
        ┌─────────────────────────┐
        │   Client Requests       │
        │   (100K reqs/sec)       │
        └────────────┬────────────┘
                     ↓
        ┌─────────────────────────┐
        │    Load Balancer        │
        │  (Health checks, routing)
        └────────┬────────────────┘
                 │
    ┌────────────┼────────────┬──────────┐
    ↓            ↓            ↓          ↓
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│Server 1│  │Server 2│  │Server 3│  │Server 4│
│25K r/s │  │25K r/s │  │25K r/s │  │25K r/s │
└────────┘  └────────┘  └────────┘  └────────┘
    ✓           ✓           ✓           ✓
   Healthy    Healthy     Healthy    Healthy
```

## Load Balancing Strategies

### 1. Round-Robin (Simple)
```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 4
Request 5 → Server 1 (cycle)

Best for: Homogeneous servers, equal load
Pros: Simple, no overhead
Cons: Doesn't account for server capacity
Hit rate: 70-80%
```

### 2. Least Connections (Smart)
```
Route to server with fewest active connections
- Server 1: 45 connections
- Server 2: 12 connections ← ROUTE HERE
- Server 3: 38 connections

Best for: Long-lived connections
Pros: Balances real load
Cons: Requires connection tracking
Hit rate: 85-95%
```

### 3. Weighted Round-Robin
```
Assign weights to servers:
- Server 1 (new): weight 1
- Server 2 (new): weight 1
- Server 3 (powerful): weight 4

Route 6 requests: 1→S1, 1→S2, 4→S3
Best for: Heterogeneous hardware
Pros: Utilizes different capacities
```

### 4. IP Hash (Session Affinity)
```
Hash client IP → same server
Client 192.168.1.10 → hash → Server 2 (always)
Client 192.168.1.11 → hash → Server 4 (always)

Best for: Session state
Pros: Session persistence
Cons: Poor distribution if uneven IPs
```

### 5. Least Response Time
```
Route to server with fastest average response time
- Server 1: avg 45ms
- Server 2: avg 12ms ← ROUTE HERE
- Server 3: avg 38ms

Best for: Variable server performance
Pros: Optimizes user experience
Cons: Requires response time tracking
```

## 📡 Health Checking

### Active Health Checks
```
Load balancer polls servers periodically:
Every 5 seconds:
  GET /health
  If 200 OK → server healthy
  If timeout/error → mark unhealthy
  Remove from rotation until recovery

Types:
1. HTTP health endpoint (fast, accurate)
2. TCP connection (basic)
3. Custom protocol (detailed metrics)
```

### Failure Detection
```
Threshold: 3 failed health checks
Action: Remove server from rotation
Auto-recovery: Once healthy again, re-add to rotation

Prevents: Cascade failures from faulty servers
```

## 🔄 Failover Workflow

```
1. Client sends request
2. Load balancer routes to Server 1
3. Server 1 connection times out
4. Load balancer tries Server 2
5. Server 2 responds successfully
6. Next request goes to healthy servers
7. Server 1 health check fails
8. Server 1 marked unhealthy
9. Taken out of rotation
10. When Server 1 recovers:
    - Health check passes
    - Added back to rotation
```

## 📊 Load Balancing Algorithms Comparison

| Algorithm | Best For | Hit Rate | Setup |
|-----------|----------|----------|-------|
| **Round-Robin** | Simple, equal servers | 70% | 1 hour |
| **Least Connections** | Long-lived connections | 85% | 2 hours |
| **Weighted** | Heterogeneous hardware | 90% | 3 hours |
| **IP Hash** | Session persistence | 75% | 1 hour |
| **Least Response Time** | Variable performance | 95% | 4 hours |

## 🏢 Load Balancing Tiers

### Layer 4 (Transport Layer)
```
Operates on IP/TCP level
- High performance (millions of requests/sec)
- Lower latency (minimal processing)
- Less aware of application logic
- Examples: HAProxy (TCP mode), NGINX (TCP)

Best for: High-throughput, latency-sensitive
```

### Layer 7 (Application Layer)
```
Operates on HTTP/HTTPS level
- Can inspect request content
- Route based on URL path, hostname, headers
- Lower throughput (more processing)
- Examples: NGINX (HTTP mode), HAProxy (HTTP mode), AWS ALB

Best for: Complex routing logic, microservices
```

## 🔧 Routing Rules Example (Layer 7)

```nginx
# Route by URL path
/api/users → user-service:3001
/api/orders → order-service:3002
/api/products → product-service:3003

# Route by hostname
api.example.com → API servers
www.example.com → Web servers
admin.example.com → Admin servers

# Route by header
X-Client-Type: mobile → mobile-optimized servers
X-Client-Type: desktop → desktop servers

# Route by cookie
session_server=1 → Server 1 (sticky sessions)
```

## 🎯 Stickiness & Sessions

### Session Persistence (Sticky Sessions)
```
Problem: User logs in on Server 1
User sent to Server 2 on next request
Server 2 doesn't have session data
User appears logged out

Solution: Sticky sessions
User 1 always routes to Server 1
User 2 always routes to Server 2

Implementation:
1. Cookie-based: Store server ID in cookie
2. IP-hash: Hash client IP to server
3. App-managed: App tells LB which server to use

Trade-off: Reduces load distribution but maintains sessions
```

## 💡 Typical Implementation: NGINX

```nginx
upstream backend {
    # Round-robin with health checks
    server backend1.example.com max_fails=3 fail_timeout=10s;
    server backend2.example.com max_fails=3 fail_timeout=10s;
    server backend3.example.com max_fails=3 fail_timeout=10s;
    
    # Least connections algorithm
    least_conn;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        # Route to backend
        proxy_pass http://backend;
        
        # Sticky sessions
        proxy_cookie_path / "/";
        proxy_cookie_flags ~ secure httponly;
        
        # Health check
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

## 📈 Scaling Strategies

### Horizontal Scaling
```
As traffic grows:
5K reqs/sec → 1 server
25K reqs/sec → 4 servers
100K reqs/sec → 16 servers
500K reqs/sec → 64 servers

Each server handles constant load
Load balancer distributes evenly
```

### Auto-Scaling
```
1. Monitor server load
2. If load > threshold (80%):
   - Launch new server instance
   - Add to load balancer pool
3. If load < threshold (20%):
   - Remove server from rotation
   - Terminate instance (save cost)

Tools: Kubernetes, AWS ECS, Nomad
```

## 🔗 Integration Points

- **API Gateway** - Routes incoming requests
- **Microservices** - Distribute across service instances
- **Cache Layer** - Load balance cache nodes
- **Database** - Read replicas behind load balancer
- **Monitoring** - Track health and distribution

## ⚠️ Common Pitfalls

1. **Uneven Distribution** - Some servers overloaded, others idle
   - Solution: Use least-connections or response-time algorithms

2. **Connection Limits** - Too many connections to one server
   - Solution: Connection pooling, queue management

3. **Session Loss** - Sessions not persisted across servers
   - Solution: Use sticky sessions or centralized session store

4. **Health Check Failures** - Healthy servers marked unhealthy
   - Solution: Tune timeouts, use multiple health check methods

---

**Pattern Version:** 1.0 | **Status:** Production Pattern | **Typical Gain:** 4-16x capacity at same cost

