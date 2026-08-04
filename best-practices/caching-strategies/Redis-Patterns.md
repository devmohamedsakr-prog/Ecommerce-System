# Redis Caching Patterns for E-Commerce

## 📋 Overview

Redis is the de-facto standard for high-performance caching in e-commerce systems. This guide covers patterns for 10B+ requests/day systems.

## 🎯 Common Redis Patterns

### Pattern 1: Cache-Aside (Lazy Loading)

```javascript
async function getProduct(productId) {
  // Step 1: Check cache
  const cacheKey = `product:${productId}`;
  let product = await redis.get(cacheKey);
  
  if (product) {
    return JSON.parse(product);  // Cache hit: fast!
  }
  
  // Step 2: Cache miss - fetch from database
  product = await db.query('SELECT * FROM products WHERE id = ?', [productId]);
  
  // Step 3: Store in cache
  await redis.set(cacheKey, JSON.stringify(product), 'EX', 3600);  // 1 hour
  
  return product;
}

Performance:
├── Cache hit: 1ms (Redis)
├── Cache miss (cold): 100ms (DB query + Redis write)
├── Hit rate: 95% (after warm-up)
├── Average: 6ms (0.95 * 1 + 0.05 * 100)
```

### Pattern 2: Write-Through

```javascript
async function updateProduct(productId, data) {
  // Step 1: Update cache
  const cacheKey = `product:${productId}`;
  await redis.set(cacheKey, JSON.stringify(data), 'EX', 3600);
  
  // Step 2: Update database
  await db.query('UPDATE products SET ? WHERE id = ?', [data, productId]);
  
  return data;
}

Benefits:
├── Data always in cache
├── No stale data
├── Simple logic

Risks:
├── Slow writes (wait for cache + DB)
├── Failed DB write leaves stale cache
```

### Pattern 3: Write-Behind (Write-Back)

```javascript
async function updateProduct(productId, data) {
  // Step 1: Update cache immediately (fast)
  const cacheKey = `product:${productId}`;
  await redis.set(cacheKey, JSON.stringify(data), 'EX', 3600);
  
  // Step 2: Queue database update (async)
  await queue.add('updateProduct', { productId, data });
  
  return { status: 'queued' };
}

// Background worker processes queue
queue.process('updateProduct', async (job) => {
  const { productId, data } = job.data;
  await db.query('UPDATE products SET ? WHERE id = ?', [data, productId]);
});

Benefits:
├── Fast writes (cache only, then queue)
├── Database batch updates
├── High throughput

Risks:
├── Data loss if crash before DB write
├── Complex debugging
├── Must handle failures
```

## 🔑 Key Patterns for E-Commerce

### Session Storage

```javascript
// Store user session in Redis
async function createSession(userId) {
  const sessionId = crypto.randomUUID();
  const sessionData = {
    userId,
    loginTime: new Date(),
    lastActivity: new Date(),
    cart: []
  };
  
  // TTL: 30 days (or until logout)
  await redis.set(
    `session:${sessionId}`,
    JSON.stringify(sessionData),
    'EX',
    30 * 24 * 60 * 60  // 30 days in seconds
  );
  
  return sessionId;
}

// Update last activity
async function updateSession(sessionId) {
  const data = await redis.get(`session:${sessionId}`);
  const session = JSON.parse(data);
  session.lastActivity = new Date();
  
  // Reset TTL on every update
  await redis.set(
    `session:${sessionId}`,
    JSON.stringify(session),
    'EX',
    30 * 24 * 60 * 60
  );
}

Performance:
├── Session lookup: < 1ms
├── Session update: < 1ms
├── Supports 1M+ concurrent sessions
```

### Shopping Cart

```javascript
async function addToCart(userId, productId, quantity) {
  const cartKey = `cart:${userId}`;
  
  // Get current cart
  let cart = await redis.get(cartKey);
  cart = cart ? JSON.parse(cart) : { items: [] };
  
  // Add/update item
  const item = cart.items.find(i => i.productId === productId);
  if (item) {
    item.quantity += quantity;
  } else {
    cart.items.push({ productId, quantity });
  }
  
  // Save cart (TTL: 30 days)
  await redis.set(
    cartKey,
    JSON.stringify(cart),
    'EX',
    30 * 24 * 60 * 60
  );
  
  return cart;
}

// Persistent storage (for recovery)
async function savePersistentCart(userId, cart) {
  // Save to database (async)
  await db.query(
    'INSERT INTO carts (user_id, data) VALUES (?, ?) ON DUPLICATE KEY UPDATE data = ?',
    [userId, JSON.stringify(cart), JSON.stringify(cart)]
  );
}
```

### Rate Limiting

```javascript
async function checkRateLimit(userId) {
  const key = `ratelimit:${userId}`;
  const limit = 100;  // 100 requests
  const window = 3600;  // per 1 hour
  
  const current = await redis.incr(key);
  
  if (current === 1) {
    // First request in window
    await redis.expire(key, window);
  }
  
  if (current > limit) {
    throw new Error('Rate limit exceeded');
  }
  
  return {
    remaining: limit - current,
    resetAt: await redis.ttl(key)
  };
}
```

### Leaderboard (Sorted Set)

```javascript
// Add score for seller
async function updateSellerRating(sellerId, rating) {
  // Add to sorted set (scored by rating)
  await redis.zadd('seller_ratings', rating, sellerId);
}

// Get top 10 sellers
async function getTopSellers() {
  return await redis.zrevrange('seller_ratings', 0, 9, 'WITHSCORES');
}

// Get seller rank
async function getSellerRank(sellerId) {
  return await redis.zrevrank('seller_ratings', sellerId);
}

// Result:
// 1. Seller A: 4.9
// 2. Seller B: 4.8
// 3. Seller C: 4.7
```

### Pub/Sub for Real-Time Updates

```javascript
// Publisher: Order service publishes order events
async function publishOrderEvent(orderId, status) {
  const message = JSON.stringify({ orderId, status, timestamp: new Date() });
  await redis.publish('orders', message);
}

// Subscriber 1: Notification service (send emails)
const subscriber1 = redis.createClient();
subscriber1.subscribe('orders', (message) => {
  const { orderId, status } = JSON.parse(message);
  if (status === 'delivered') {
    sendDeliveryEmail(orderId);
  }
});

// Subscriber 2: Analytics service (track metrics)
const subscriber2 = redis.createClient();
subscriber2.subscribe('orders', (message) => {
  const { orderId, status, timestamp } = JSON.parse(message);
  recordMetric('orders', status, timestamp);
});

Benefits:
├── Real-time event distribution
├── Decoupled services
├── Millisecond latency
```

## 🏗️ Redis Deployment Strategies

### Single Node (Small Scale)

```
Use for: < 10 GB data, < 10k req/sec

┌─────────────┐
│  Redis      │
│  Node       │
│  16GB RAM   │
└─────────────┘

Limitations:
├── Single point of failure
├── Can't scale beyond single server
├── Data loss if crash
└── No redundancy
```

### Master-Slave Replication

```
Use for: 10-100 GB data, 10k-100k req/sec

┌──────────────┐         ┌──────────────┐
│  Master      │────────>│  Slave       │
│  Write ops   │ Replicate│ Read ops    │
│  16GB RAM    │         │ 16GB RAM     │
└──────────────┘         └──────────────┘

Setup:
master.conf:
  port 6379
  save 900 1

slave.conf:
  port 6380
  slaveof 127.0.0.1 6379

Benefits:
├── Read scaling (read from slaves)
├── Redundancy (switch to slave if master fails)
└── Backup (slave can be backed up)

Limitations:
├── Manual failover
├── All writes still go to master
```

### Redis Sentinel (High Availability)

```
Use for: 10-100 GB data, critical systems

┌────────────┐          ┌────────────┐
│ Sentinel 1 │          │ Sentinel 2 │
│ Monitors   │────╱────>│ Monitors   │
│            │         │            │
└────────────┘          └────────────┘
      │                       │
      │ Quorum: 2/3 needed   │
      └───────────┬──────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
    ┌────────┐          ┌────────┐
    │Master  │─────────>│ Slave  │
    │        │Replicate │        │
    └────────┘          └────────┘

Automatic Failover:
1. Master dies
2. Sentinels detect (3 checks)
3. Quorum reached (2/3 sentinels agree)
4. Slave promoted to master
5. Apps auto-redirect to new master
6. Old master demoted to slave when recovered

Configuration:
sentinel.conf:
  sentinel monitor mymaster 127.0.0.1 6379 2
  sentinel down-after-milliseconds mymaster 5000
  sentinel parallel-syncs mymaster 1
```

### Redis Cluster (Massive Scale)

```
Use for: > 100 GB data, > 100k req/sec

┌─────────┐  ┌─────────┐  ┌─────────┐
│Shard 1  │  │Shard 2  │  │Shard 3  │
│Keys 0-5 │  │Keys 5-10│  │Keys 10+ │
│3 nodes  │  │3 nodes  │  │3 nodes  │
└─────────┘  └─────────┘  └─────────┘

Sharding key: hash(key) % 16384 (Redis hash slots)

Cluster Topology:
├── 6 nodes (3 masters + 3 slaves)
├── Each master: 1 slave
├── Auto-rebalancing
└── Automatic failover

Advantages:
├── Horizontal scaling (add more nodes)
├── High availability (master + slave per shard)
├── No single point of failure
└── Can handle 1TB+ data

Setup:
redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6380 ...

Client Code:
const redis = require('redis-cluster');
const client = redis.createClient({ nodes: [...] });
await client.set('key', 'value');  // Auto-routes to correct shard
```

## 📊 Monitoring Redis

### Key Metrics

```
Memory Usage:
├── used_memory: Current RAM usage
├── used_memory_peak: Peak usage
├── evicted_keys: Keys removed due to memory limit
└── Alert if: used_memory > 80% of max_memory

Performance:
├── ops_per_sec: Commands per second
├── hits: Cache hits
├── misses: Cache misses
├── hit_rate: hits / (hits + misses)
└── Alert if: hit_rate < 90%

Replication:
├── connected_slaves: Slave count
├── master_repl_offset: Replication position
├── slave_repl_offset: Slave position
└── Alert if: Offset divergence > 1MB

Connections:
├── connected_clients: Client count
├── rejected_connections: Rejected due to limit
└── Alert if: connected_clients > 80% of max_clients

Persistence:
├── rdb_last_save_time: Last RDB save
├── rdb_last_bgsave_status: Background save status
└── Alert if: Last save > 1 hour
```

### Slow Query Log

```
CONFIG SET slowlog-log-slower-than 10000  # 10ms

Get slow queries:
SLOWLOG GET 10

Output:
1) 1) ID
   2) Timestamp
   3) Duration (microseconds)
   4) Command
   5) Client address

Example:
SLOWLOG GET 5
Shows last 5 slow queries for analysis
```

## ✅ Redis Best Practices

```
DO:
✅ Use appropriate data types (String, Hash, List, Set, Sorted Set)
✅ Set TTL on all keys (prevent memory leak)
✅ Use connection pooling
✅ Monitor memory usage
✅ Replicate for redundancy
✅ Use Sentinel or Cluster for HA
✅ Compress large values
✅ Use pipelines for batch operations
✅ Monitor hit rate
✅ Clean up expired keys

DON'T:
❌ Store uncompressed large objects
❌ Use without TTL (memory leak)
❌ Make synchronous blocking calls
❌ Ignore memory warnings
❌ Run on same server as app
❌ Use production data in dev
❌ Store unencrypted sensitive data
❌ Ignore replication lag
❌ Forget disaster recovery
```

---

**Key Principle:** "Redis is fast, but only if you use it correctly. Cache strategically, monitor obsessively, scale proactively."
