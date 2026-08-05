# Caching Layer Pattern

**Status:** Architecture Pattern | **Priority:** CRITICAL | **Scale:** 10K-1M requests/sec

---

## 📋 Overview

Caching layer reduces database load by 10-100x through intelligent caching strategies. Typical response time improvement: 50ms → 5ms. Redis or Memcached sits between application and database.

## 🎯 Business Problem

- Database queries become bottleneck at scale
- Hot data accessed repeatedly (products, inventory, user sessions)
- Without caching: 100K requests/sec overwhelms any database
- With caching: Same database handles 1M+ requests/sec
- Cost impact: 10-100x reduction in database resources needed

## 🏗️ Architecture

```
┌─────────────────────────────────────┐
│     Client Requests                 │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│     Application Layer               │
│  (Node.js, Python, Java)            │
└──────────────┬──────────────────────┘
               ↓
    ┌──────────────────────┐
    │  Caching Layer       │
    │  (Redis/Memcached)   │
    │  Cache Hit: 5ms      │
    └──────┬────────┬──────┘
           │        │
     Cache │        │ Cache Miss
     Hit   │        ↓
     (95%) │    ┌──────────────────┐
           │    │  Database Layer  │
           │    │  (PostgreSQL)    │
           │    │  Query: 50ms     │
           └────┤  + Cache Update  │
                └──────────────────┘
```

## Data Model: Cache Strategy

```
CACHE_ENTRY
├── cache_key (string: "product:123", "user:456:session")
├── value (JSON: product data, session data)
├── ttl_seconds (number: 300, 3600, 86400)
├── created_timestamp (timestamp)
├── last_accessed (timestamp)
├── access_count (number)
├── size_bytes (number)
├── cache_level (enum: L1-memory, L2-redis, L3-disk)
└── expiry_timestamp (timestamp)

CACHE_POLICY
├── policy_id (UUID)
├── resource_type (enum: product, user, cart, order, inventory)
├── ttl_seconds (number)
├── max_size_mb (number)
├── eviction_policy (enum: LRU, LFU, FIFO)
├── cache_warming (boolean)
├── staleness_tolerance_seconds (number)
└── fallback_behavior (enum: database, error, stale-data)

CACHE_STATS
├── stats_id (UUID)
├── period (YYYY-MM-DD HH:00)
├── cache_hits (number)
├── cache_misses (number)
├── hit_rate (decimal: percentage)
├── avg_response_time_ms (number)
├── memory_used_mb (number)
├── evictions (number)
└── error_rate (decimal: percentage)
```

## 🔄 Caching Strategies

### 1. Cache-Aside (Lazy Loading)
```
Application checks cache:
1. Request comes in
2. Check cache (5ms if hit)
3. If miss, query database (50ms)
4. Write to cache
5. Return result

Best for: Non-critical data, infrequent writes
Hit rate: 80-95%
Downside: First request always slow
```

### 2. Write-Through (Synchronous)
```
Application writes to both cache and database:
1. Write request arrives
2. Write to cache (immediate)
3. Write to database (50ms)
4. Return success

Best for: Critical data (orders, payments)
Consistency: Guaranteed
Downside: Writes slower, double writes
```

### 3. Write-Behind (Asynchronous)
```
Application writes to cache only:
1. Write request arrives
2. Write to cache (5ms)
3. Queue write to database (async)
4. Return success immediately

Best for: High-volume writes, tolerable delay
Performance: 10x faster writes
Downside: Temporary inconsistency if crash
```

## 📡 Cache Patterns by Data Type

### Session Cache (TTL: 1-24 hours)
```
Key: user:{user_id}:session
Value: {
  user_id, session_token, 
  permissions, last_activity
}
Pattern: Cache-aside
Hit rate target: 95%+
```

### Product Cache (TTL: 1-6 hours)
```
Key: product:{product_id}
Value: {
  name, description, price, 
  inventory, images, ratings
}
Pattern: Cache-aside (warm on startup)
Hit rate target: 98%+
Update: On product catalog changes
```

### Inventory Cache (TTL: 5-30 minutes)
```
Key: inventory:{product_id}:{warehouse_id}
Value: {quantity_available, reserved, pending}
Pattern: Write-through (consistency critical)
Hit rate target: 90%+
Update: Real-time on orders/returns
```

### Cart Cache (TTL: 30 minutes-7 days)
```
Key: cart:{user_id}
Value: {items[], total, last_updated}
Pattern: Write-through
Hit rate target: 85%+
Fallback: Database if lost (non-critical)
```

### Leaderboard/Rankings (TTL: 1-24 hours)
```
Key: leaderboard:top-products:category:{category}
Value: sorted list of products
Pattern: Cache-aside (rebuilt periodically)
Hit rate target: 99%+
Update: Batch rebuild off-peak
```

## 🎯 Cache Invalidation Strategies

### 1. TTL-Based (Time-To-Live)
```
Cache entries expire after X seconds
Implementation: Redis EXPIRE key 3600
Best for: Data that changes infrequently
Simplest approach
```

### 2. Event-Based
```
Cache invalidated when data changes
1. Product updated → publish event
2. Cache listener receives event
3. Cache key deleted immediately
4. Next request misses cache, reloads

Best for: Critical consistency
Real-time invalidation
```

### 3. Active Invalidation
```
Application explicitly clears cache:
DELETE cache_key on data update
Pattern: Immediate after mutation
Best for: Writes that must invalidate
```

### 4. Lazy Invalidation
```
Cache serves stale data until TTL expires
Application detects staleness via versioning
Best for: Tolerable temporary staleness
Improves hit rate
```

## 📊 Key Metrics

| Metric | Target | Method |
|--------|--------|--------|
| **Cache Hit Rate** | 85-99% | Count hits vs misses |
| **Response Time (hit)** | < 10ms | Monitor latency |
| **Response Time (miss)** | 40-60ms | Track database queries |
| **Memory Efficiency** | < 80% utilization | Monitor memory usage |
| **Eviction Rate** | < 5% | Track evictions |
| **Cost Reduction** | 30-70% vs no cache | Compare infrastructure |

## 🔧 Implementation Example (Redis)

```javascript
// Cache-aside pattern
async function getProduct(productId) {
  const cacheKey = `product:${productId}`;
  
  // Check cache
  let product = await redis.get(cacheKey);
  if (product) return JSON.parse(product);
  
  // Cache miss - fetch from database
  product = await db.query(
    'SELECT * FROM products WHERE id = ?', 
    [productId]
  );
  
  // Write to cache (1 hour TTL)
  await redis.setex(cacheKey, 3600, JSON.stringify(product));
  
  return product;
}

// Write-through pattern
async function updateProduct(productId, updates) {
  const cacheKey = `product:${productId}`;
  
  // Update database
  const product = await db.query(
    'UPDATE products SET ? WHERE id = ?',
    [updates, productId]
  );
  
  // Update cache
  await redis.setex(cacheKey, 3600, JSON.stringify(product));
  
  return product;
}

// Invalidate cache
async function invalidateProductCache(productId) {
  const cacheKey = `product:${productId}`;
  await redis.del(cacheKey);
}
```

## 🔗 Integration Points

- **Database Layer** - Reduce query load
- **Session Service** - Fast user session lookups
- **Product Catalog** - Cache product data
- **Inventory Service** - Real-time inventory caching
- **Order Service** - Cache recent orders
- **Notification Service** - Cache templates

## ⚠️ Common Pitfalls

1. **Cache Stampede** - Many requests miss cache simultaneously, all hit database at once
   - Solution: Probabilistic early expiration or lock-based refresh
   
2. **Cache Corruption** - Stale or invalid data cached
   - Solution: Versioning, validation on retrieve
   
3. **Memory Pressure** - Cache grows too large
   - Solution: Set max memory, use LRU eviction
   
4. **Inconsistency** - Cache diverges from database
   - Solution: Write-through or event-based invalidation

---

**Pattern Version:** 1.0 | **Status:** Production Pattern | **Typical Improvement:** 10-100x performance lift

