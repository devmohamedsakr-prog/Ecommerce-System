# Performance Optimization for E-Commerce

## 📋 Overview

Complete guide to optimizing e-commerce systems for speed, handling peak traffic, and improving user experience.

## 🎯 Performance Targets

### Response Time SLAs

```
Page Load: < 3 seconds (p95)
├── Recommendation: < 2 seconds
└── Critical: > 5 seconds = unacceptable

API Response: < 200ms (p95)
├── Checkout: < 100ms (critical)
├── Search: < 200ms
└── Product page: < 300ms

Payment Processing: < 500ms (p99)
├── Authorization: < 200ms
├── Capture: < 100ms
└── Refund: < 200ms

Search: < 1 second (p95)
├── < 100ms (p50)
└── < 2 seconds (p99)
```

### User Perception

```
0-100ms: Feels instant
100-300ms: Perceptibly fast
300-1000ms: Noticeable delay
1000ms+: Significant lag
```

## 🏗️ Optimization Layers

### Layer 1: Database Optimization

**Query Optimization**

```sql
❌ SLOW: Full table scan
SELECT * FROM orders WHERE customer_id = 123 AND status = 'pending'
-- Scans all 100M orders

✅ FAST: Indexed lookup
CREATE INDEX idx_orders_customer_status 
ON orders(customer_id, status);

SELECT * FROM orders WHERE customer_id = 123 AND status = 'pending'
-- Uses index: < 1ms
```

**Monitoring Slow Queries**

```
Enable slow query log:
├── Log queries > 100ms
├── Analyze patterns
├── Find missing indexes
└── Optimize hot queries

Example:
Query: SELECT * FROM products WHERE category = 'electronics'
Time: 2500ms (SLOW!)

Solution:
CREATE INDEX idx_products_category ON products(category);
Time: 15ms (166x faster!)
```

**Connection Pooling**

```javascript
// ❌ Bad: New connection per request
app.get('/api/products', (req, res) => {
  const db = mysql.createConnection(config);  // New connection!
  db.query('SELECT * FROM products');
  db.end();  // Close immediately
});

// ✅ Good: Connection pool
const pool = mysql.createPool({
  connectionLimit: 100,
  host: 'localhost',
  user: 'root',
  password: 'password'
});

app.get('/api/products', (req, res) => {
  pool.query('SELECT * FROM products', (err, results) => {
    res.json(results);  // Connection returned to pool
  });
});

Benefits:
├── Reuse connections
├── No connection overhead
├── Handles 100+ concurrent requests
└── < 1ms per query (vs 50ms per new connection)
```

### Layer 2: Caching Strategy

**Redis Caching Pattern**

```javascript
async function getProduct(productId) {
  // Check cache first
  const cached = await redis.get(`product:${productId}`);
  if (cached) return JSON.parse(cached);
  
  // Cache miss: query database
  const product = await db.query(
    'SELECT * FROM products WHERE id = ?',
    [productId]
  );
  
  // Store in cache (1 hour TTL)
  await redis.set(
    `product:${productId}`,
    JSON.stringify(product),
    'EX',
    3600  // 3600 seconds = 1 hour
  );
  
  return product;
}

Performance:
├── Cache hit: 1-2ms (Redis)
├── Cache miss: 50-100ms (Database)
├── Hit rate: 99% (after warm-up)
├── Average latency: 2-3ms (mostly cache hits)
```

**Cache Invalidation**

```
TTL-Based (Lazy Invalidation):
├── Products: 1 hour
├── Categories: 24 hours
├── User sessions: 30 minutes
└── Cart: 7 days

Event-Based (Eager Invalidation):
Product updated:
  ├── DELETE cache entry
  ├── Send event to all servers
  ├── Serve fresh from database
  └── Repopulate cache

Hybrid:
├── Short TTL (5 minutes)
├── Event-based refresh
└── Probabilistic refresh (small % of requests refresh)
```

### Layer 3: CDN & Static Asset Optimization

**Content Delivery Network**

```
Static Content (CSS, JS, images):
├── Users in US → Serve from US CDN node
├── Users in EU → Serve from EU CDN node
├── Users in Asia → Serve from Asia CDN node
└── Latency reduced: 100-200ms → 10-20ms

Configuration:
├── Cache duration: 24 hours
├── Gzip compression: 70% reduction
├── Brotli compression: 80% reduction (better)
├── Minification: Remove comments, whitespace
└── Image optimization: WebP format
```

**Image Optimization**

```
Original JPG: 2.5 MB
├── Resize to 1200px width: 800 KB
├── Compress quality: 85%: 200 KB
├── Convert to WebP: 80 KB
├── Serve different sizes per device:
│   ├── Mobile (400px): 30 KB
│   ├── Tablet (800px): 60 KB
│   ├── Desktop (1200px): 100 KB
│   └── Retina (1200px @ 2x): 150 KB
└── Lazy load images below fold
```

### Layer 4: Database Query Optimization

**N+1 Query Problem**

```javascript
❌ SLOW: N+1 queries
const orders = await db.query('SELECT * FROM orders');
for (const order of orders) {
  order.customer = await db.query(
    'SELECT * FROM customers WHERE id = ?',
    [order.customer_id]
  );  // 100 orders = 101 queries!
}

✅ FAST: Single query with JOIN
const orders = await db.query(
  `SELECT o.*, c.* FROM orders o
   JOIN customers c ON o.customer_id = c.id`
);

✅ FAST: Batch query
const orders = await db.query('SELECT * FROM orders');
const customerIds = orders.map(o => o.customer_id);
const customers = await db.query(
  'SELECT * FROM customers WHERE id IN (?)',
  [customerIds]
);
// 2 queries instead of 101!
```

**Denormalization for Speed**

```
Normalized (slow):
SELECT o.total, c.name, c.country
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.id = 123;
-- 1 join = 50ms

Denormalized (fast):
CREATE TABLE orders (
  id UUID,
  total DECIMAL,
  customer_name TEXT,  -- Copied from customers
  customer_country TEXT  -- Copied from customers
);

SELECT total, customer_name, customer_country
FROM orders
WHERE id = 123;
-- No join = 5ms

Trade-off:
✅ Fast reads
❌ Slow updates (must update both tables)
└── Use for data that changes rarely
```

### Layer 5: Application-Level Optimization

**Lazy Loading**

```javascript
// ❌ Load everything immediately
app.get('/products', async (req, res) => {
  const products = await db.query(`
    SELECT * FROM products p
    LEFT JOIN reviews r ON p.id = r.product_id
    LEFT JOIN ratings r2 ON p.id = r2.product_id
  `);  // Large result set: 500ms
  res.json(products);
});

// ✅ Load on demand
app.get('/products/:id', async (req, res) => {
  const product = await db.query(
    'SELECT * FROM products WHERE id = ?',
    [req.params.id]
  );  // Fast: 50ms
  
  res.json(product);
});

app.get('/products/:id/reviews', async (req, res) => {
  // Reviews loaded separately when needed
  const reviews = await db.query(
    'SELECT * FROM reviews WHERE product_id = ? LIMIT 10',
    [req.params.id]
  );
  res.json(reviews);
});
```

**Pagination**

```javascript
// ❌ Load all 1M products
app.get('/products', async (req, res) => {
  const products = await db.query('SELECT * FROM products');
  res.json(products);  // 1M records = 500MB response
});

// ✅ Paginate
app.get('/products', async (req, res) => {
  const limit = req.query.limit || 20;
  const offset = req.query.offset || 0;
  
  const products = await db.query(
    'SELECT * FROM products LIMIT ? OFFSET ?',
    [limit, offset]
  );
  
  res.json({
    data: products,
    limit,
    offset,
    hasMore: products.length === limit
  });
});
```

### Layer 6: Frontend Optimization

**Code Splitting**

```javascript
// ❌ Single bundle: 500 KB
// Initial page load: 5 seconds

// ✅ Code splitting: Multiple bundles
import React from 'react';
const ProductPage = React.lazy(() => import('./ProductPage'));
const CheckoutPage = React.lazy(() => import('./CheckoutPage'));

// Initial bundle: 50 KB
// ProductPage loaded on demand: 100 KB
// CheckoutPage loaded on demand: 80 KB

// Benefits:
// ├── Initial load: 500ms (vs 5s)
// └── Additional pages: < 200ms (cached)
```

**Minification & Compression**

```
Original bundle: 500 KB
├── Minify JS: 150 KB (70% reduction)
├── Minify CSS: 40 KB (80% reduction)
├── Gzip compression: 50 KB (66% reduction)
└── Download: 500 KB → 50 KB (10x smaller)

Transfer time:
├── 4G (10 Mbps): 400ms → 40ms
├── 3G (3 Mbps): 1.3s → 130ms
└── Satellite (1 Mbps): 4s → 400ms
```

## 📊 Load Testing

### Apache JMeter Test Plan

```
Test Setup:
├── 1000 concurrent users
├── Ramp-up: 5 minutes (200 users/min)
├── Duration: 30 minutes
├── Requests: 100k total

Endpoints Tested:
├── Homepage: 20%
├── Product search: 30%
├── Product detail: 25%
├── Add to cart: 15%
├── Checkout: 10%

Results Target:
├── Response time p95: < 500ms
├── Response time p99: < 1000ms
├── Error rate: < 0.1%
├── Throughput: > 100 requests/sec
```

### Load Test Results Analysis

```
Good Results:
├── Response time p95: 250ms ✅
├── Response time p99: 400ms ✅
├── Error rate: 0.05% ✅
├── Throughput: 150 req/sec ✅

Action: Deploy to production

Poor Results:
├── Response time p95: 2000ms ❌
├── Error rate: 5% ❌
├── CPU: 95% ❌

Investigation:
1. Find bottleneck (query? cache? disk I/O?)
2. Optimize or scale
3. Retest
4. Repeat
```

## 🎯 Real-Time Monitoring

### Key Metrics to Monitor

```
Frontend:
├── Page Load Time (p50, p95, p99)
├── Time to First Interactive
├── Cumulative Layout Shift
├── Core Web Vitals

Backend:
├── API response time
├── Error rate (4xx, 5xx)
├── Database query time
├── Cache hit rate

Infrastructure:
├── CPU utilization
├── Memory utilization
├── Disk I/O
├── Network throughput

Business:
├── Conversion rate
├── Cart abandonment
├── Revenue per user
├── Orders per minute
```

### Alerting Rules

```
Alert if:
├── Response time p95 > 500ms → Page load slow
├── Error rate > 1% → Something broken
├── CPU > 80% → Need to scale
├── Cache hit rate < 90% → Cache misconfigured
├── Database latency > 100ms → Need indexes
└── Memory > 85% → Memory leak or need more RAM
```

## 💡 Performance Optimization Checklist

```
Database:
- [ ] Indexes on frequently queried columns
- [ ] Query analysis (EXPLAIN plan)
- [ ] Connection pooling enabled
- [ ] Replication for read distribution
- [ ] Sharding for massive scale

Caching:
- [ ] Redis for hot data
- [ ] CDN for static assets
- [ ] Browser cache (long TTL)
- [ ] Cache invalidation strategy
- [ ] Cache hit rate > 95%

Frontend:
- [ ] Code splitting
- [ ] Lazy loading
- [ ] Image optimization
- [ ] Minification & compression
- [ ] Async JavaScript loading

Infrastructure:
- [ ] Load balancing
- [ ] Auto-scaling
- [ ] Regional deployment
- [ ] Multiple availability zones
- [ ] Disaster recovery

Monitoring:
- [ ] Real-time dashboards
- [ ] Alerting rules
- [ ] Log aggregation
- [ ] Distributed tracing
- [ ] Performance budget
```

---

**Key Rule:** "Cache everything you can, invalidate intelligently, monitor obsessively."
