# Database Sharding - Enterprise Scale Guide

## 📋 Overview

When a single database can't handle your data volume or request rate, sharding divides data across multiple databases for horizontal scaling.

## 🎯 When to Shard

### Signs You Need Sharding

```
Single Database Limits:

Data Size:
├── Database reaches 1TB+ (too large for backups)
├── Queries slow down (full table scans inefficient)
├── Replication lag > 1 second (too much data)
└── Upgrade paths exhausted

Throughput:
├── Database at 80%+ CPU utilization
├── Connections at max limits
├── Query queue building up
├── p99 latency > 500ms

Scale Timeline:
├── Starting: 100k users (single DB fine)
├── Growing: 1M users (replicas help)
├── Scaling: 10M users (sharding needed)
└── Enterprise: 100M+ users (multiple sharding strategies)
```

### What NOT to Shard (Yet)

```
Don't shard when:
├── Can optimize queries (add indexes)
├── Can cache more (Redis hits)
├── Can buy bigger hardware (vertical scaling)
├── Database upgrade available
└── Read replicas solve problem
```

## 🔑 Choosing a Shard Key

### Shard Key Requirements

```
Good Shard Key:
✅ Evenly distributes data (no hot shards)
✅ Never changes (no resharding needed)
✅ Frequently used in queries
✅ Has enough cardinality (millions of values)
✅ Enables consistent hashing

Bad Shard Key:
❌ Changes over time (user moves to new country)
❌ Uneven distribution (100 shards, one has 90% data)
❌ Not queryable without shard info
❌ Too few values (status: active/inactive only)
❌ Enables hot sharding (one shard overloaded)
```

### Shard Key Options

**Option 1: User ID (RECOMMENDED)**

```
Shard Selection:
shard_id = hash(user_id) % total_shards

Example:
user_id = 12345
hash(12345) = 98765432
98765432 % 10 = 2
→ Store in Shard 2

Advantages:
✅ Even distribution (hash function)
✅ Never changes
✅ Most queries start with user_id
✅ Easy to reshrd (migrate user data)
✅ Used by: Amazon, Google, Facebook

Disadvantages:
❌ Cross-user queries hard (analytics)
❌ Migration requires moving all user data
```

**Option 2: Customer ID**

```
Similar to user ID for B2B systems

shard_id = hash(customer_id) % total_shards

Good for:
├── Multi-tenant SaaS (Shopify)
├── B2B platforms
├── Enterprise systems
```

**Option 3: Geographic Region**

```
shard_id = geographic_mapping(country)

Example mapping:
├── Shard 1: United States
├── Shard 2: Europe
├── Shard 3: Asia
└── Shard 4: Rest of World

Advantages:
✅ Data residency compliance (GDPR)
✅ Low latency (data near users)
✅ Simple sharding logic

Disadvantages:
❌ Uneven data distribution
❌ Hot shards during peak hours
❌ User relocation causes resharding
```

**Option 4: Timestamp Range (Time-Series)**

```
shard_id = date_range_mapping(timestamp)

Example:
├── Shard 1: January 2026 data
├── Shard 2: February 2026 data
├── Shard 3: March 2026 data
└── Archive: Older data

Good for:
├── Log data (ELK Stack)
├── Time-series data (metrics)
├── Event streams
├── Analytics events

Advantages:
✅ Easy to archive old data
✅ Query optimization by time
✅ Predictable growth

Disadvantages:
❌ Hot shards at current date
❌ Cross-time queries hard
```

## 🏗️ Sharding Architecture

### Single-Shard Database

```
Before Sharding:

┌────────────────────────┐
│   Master Database      │
│   All Users            │
│   (100M users)         │
└────────────────────────┘
        ↓
    Bottleneck!
├── CPU maxed out
├── Disk I/O high
├── Replication lag
└── Slow queries
```

### Multi-Shard Database

```
After Sharding (10 shards):

┌─────────────────────────────────────────────────────┐
│         Shard Router / Middleware                    │
│  (Routes queries to correct shard)                  │
└─────────────────────────────────────────────────────┘
              ↓
┌──────────┬──────────┬──────────┬─────────┐
│ Shard 1  │ Shard 2  │ Shard 3  │ Shard N │
├──────────┼──────────┼──────────┼─────────┤
│10M users │10M users │10M users │10M user │
│          │          │          │         │
│Orders    │Orders    │Orders    │Orders   │
│Products  │Products  │Products  │Products │
│Inventory │Inventory │Inventory │Inventory│
└──────────┴──────────┴──────────┴─────────┘
```

## 🔄 Sharding Implementation

### Shard Router

```javascript
// Router determines which shard
class ShardRouter {
  constructor(shardCount) {
    this.shardCount = shardCount;
  }

  getShardId(shardKey) {
    // Consistent hashing
    const hash = this.hashFunction(shardKey);
    const shardId = hash % this.shardCount;
    return shardId;
  }

  hashFunction(key) {
    // Use MurmurHash or XXHash
    // for better distribution
    const crypto = require('crypto');
    return crypto
      .createHash('md5')
      .update(key.toString())
      .digest()
      .readUInt32BE();
  }

  getShardAddress(shardId) {
    // Map shard ID to database address
    const shards = {
      0: 'shard-1.db.internal:5432',
      1: 'shard-2.db.internal:5432',
      2: 'shard-3.db.internal:5432',
      // ...
    };
    return shards[shardId];
  }
}

// Usage
const router = new ShardRouter(10);
const userId = 12345;
const shardId = router.getShardId(userId);
const dbAddress = router.getShardAddress(shardId);
// Connect to appropriate database
```

### Query Execution

```javascript
// Query routed to correct shard
async function getUserOrders(userId) {
  // Step 1: Determine shard
  const shardId = router.getShardId(userId);
  const db = connections[shardId];
  
  // Step 2: Query only that shard
  const orders = await db.query(
    'SELECT * FROM orders WHERE user_id = $1',
    [userId]
  );
  
  return orders;
}

// Cross-shard query (must check all shards!)
async function getOrdersByStatus(status) {
  const promises = [];
  
  // Query all shards
  for (let shardId = 0; shardId < shardCount; shardId++) {
    const db = connections[shardId];
    promises.push(
      db.query(
        'SELECT * FROM orders WHERE status = $1',
        [status]
      )
    );
  }
  
  // Combine results
  const results = await Promise.all(promises);
  return results.flat();
}
```

## 📊 Data Distribution

### Ideal Distribution

```
Each shard has roughly equal data:

Shard 1: 10.1% of total data
Shard 2: 9.9% of total data
Shard 3: 10.2% of total data
...
Shard 10: 9.8% of total data

Variance: < 5% (even distribution)
```

### Hot Shard Problem

```
Uneven Distribution:

Shard 1: 50% of total data (HOT!)
Shard 2: 5% of total data
Shard 3: 5% of total data
...
Shard 10: 4% of total data

Causes:
├── Bad shard key choice
├── Natural data clustering
├── Time-based sharding (all current data in one shard)
└── Geographic sharding (one region dominant)

Solution:
├── Better shard key distribution
├── Sub-sharding (divide hot shard further)
├── Range-based sharding (multiple ranges in hot shard)
└── Consistent hashing (better distribution)
```

## 🔄 Resharding (The Hard Part)

### Why Resharding Needed

```
Initial Setup (10 shards):
├── 10M users per shard
├── 100M total users
└── Good performance

6 Months Later (100M users):
├── 10M users per shard... wait that's same ratio
├── Actually 100M total users now
├── Each shard still has 10M users
└── Still good, but approaching limits

1 Year Later (1B users):
├── Need to double to 20 shards
├── 1B / 20 = 50M users per shard
├── Performance degrading
└── Time to reshrd!
```

### Resharding Strategy: Expand (Add Shards)

```
Original Setup (10 shards):
hash(key) % 10 = shard_id

Example: hash(user_123) = 555
555 % 10 = 5 (Shard 5)

Resharding to 20 shards:
hash(key) % 20 = new_shard_id

Same user: hash(user_123) = 555
555 % 20 = 15 (Shard 15) ← Different!

Migration Process:

Phase 1: Prepare (no downtime)
├── Create 10 new empty shards (11-20)
├── Set up replication from old shards
└── Test migration process

Phase 2: Dual-Write (no downtime)
├── New writes go to both old and new shards
├── Reads from old shards (migration ongoing)
├── Verify consistency
└── Typically lasts 1-2 weeks

Phase 3: Migration (rolling)
├── Batch process all data in old shards
├── Rehash keys, move to new shards
├── Verify row counts match
├── Shard by shard (don't do all at once)

Phase 4: Cutover (minimal downtime)
├── Stop writes temporarily (5 minutes)
├── Verify no new data in old shards
├── Switch reads to new shards
├── Resume writes to new shards
└── Keep old shards as backup 1 week

Phase 5: Cleanup
├── Verify no issues for 1 week
├── Decomission old shards
├── Document process for next resharding
```

### Resharding Strategy: Consistent Hashing

```
Alternative: Use Consistent Hashing
(reduces resharding overhead)

Concept:
├── Map users and shards to a ring (0-2^32)
├── User goes to nearest shard (clockwise)
├── Add/remove shards: only nearby users affected

Example Ring:
           Shard B
             |
        Shard C ← Users 200-300
             |     (only these move)
User 100 → Shard A
User 250 → Shard B
User 350 → Shard C

Add new shard:
           Shard B
             |
        Shard D (new)
             |
        Shard C
             |
User 100 → Shard A
User 150 → Shard D (moved from A)
User 200 → Shard D (moved from B)
User 250 → Shard B (unchanged)
User 350 → Shard C (unchanged)

Benefit:
├── Only 10% of data needs to move
├── Instead of reshashing all data
└── Can add shards incrementally
```

## ⚠️ Challenges

### Challenge 1: Cross-Shard Joins

```
❌ HARD: Join across shards

SELECT o.id, o.total, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'completed'

If orders and customers sharded by user_id:
├── Easy (both on same shard)

But if differently sharded:
├── Must query all shards
├── Fetch results
├── Perform join in application
└── Slow and expensive

Solution:
├── Ensure related data on same shard
├── Denormalize data to avoid joins
├── Pre-compute results
└── Accept eventual consistency
```

### Challenge 2: Global Secondary Indexes

```
❌ HARD: Search by non-shard key

Sharded by: user_id
Search by: email

Solution 1: Query all shards
├── Send query to all 10 shards
├── Combine results
├── Slow (100-500ms)

Solution 2: Global secondary index
├── Maintain index of email → user_id
├── Lookup email → get user_id
├── Shard by user_id
├── Query that shard
└── Fast (10-50ms)

Index Structure:
{
  "email": "user@example.com",
  "user_id": "12345"
}

Stored separately (or in dedicated shard)
```

### Challenge 3: Distributed Transactions

```
❌ HARD: ACID transactions across shards

All single-shard:
BEGIN
  UPDATE account SET balance = balance - 100 WHERE id = 1
  UPDATE account SET balance = balance + 100 WHERE id = 2
COMMIT

Cross-shard (if accounts on different shards):
├── No ACID guarantee
├── Must implement 2-phase commit (complex)
├── Or use compensating transactions
└── Or denormalize to avoid cross-shard updates

Solution: Keep related data on same shard
├── If both accounts belong to same customer
├── Shard by customer_id
└── Both accounts on same shard → single transaction
```

## ✅ Best Practices

```
DO:
✅ Choose stable shard key (never changes)
✅ Aim for even distribution (<5% variance)
✅ Plan resharding strategy beforehand
✅ Use consistent hashing (easier resharding)
✅ Keep related data on same shard
✅ Monitor shard sizes and traffic
✅ Document sharding logic
✅ Automate resharding process
✅ Test resharding in staging first
✅ Have rollback plan ready

DON'T:
❌ Reshard without testing
❌ Shard by rapidly changing fields
❌ Create very small shards (overhead)
❌ Create very large shards (bottleneck)
❌ Forget about disaster recovery
❌ Ignore cross-shard queries (performance impact)
❌ Reshrd during peak traffic
❌ Skip monitoring post-resharding
```

## 📈 Monitoring

### Metrics to Track

```
Per-Shard Monitoring:
├── Data size per shard (should be equal)
├── Queries per second per shard
├── Query latency per shard
├── CPU usage per shard
├── Connection count per shard
└── Replication lag per shard

Alerts:
├── Shard size imbalance > 10%
├── Hot shard queries > 50k/sec
├── Shard latency > 100ms
├── Replication lag > 5 seconds
└── Connection limit near threshold
```

---

**When to Shard:** 1M+ users or 1TB+ data
**How to Choose Key:** Hash-based user ID
**Expect Complexity:** 2-3x operational overhead
**Plan Ahead:** Know your resharding strategy before launch
