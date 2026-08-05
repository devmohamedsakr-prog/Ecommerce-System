# Data Layer Pattern

**Status:** Architecture Pattern | **Priority:** CRITICAL | **Scale:** Petabytes, 100K+ QPS

---

## Overview

Data layer architecture determines system capacity. Choices: SQL vs NoSQL, sharding strategy, replication, consistency model. Wrong choice early = impossible to scale later.

## Core Strategies

### 1. Database Selection

**PostgreSQL/MySQL (Relational)**
- Best for: Structured data, transactions, relationships
- Scale: 10K-50K QPS single instance
- Strength: ACID, complex queries, data integrity
- Trade-off: Harder to shard, vertical scaling limit

**MongoDB/DynamoDB (NoSQL)**
- Best for: Unstructured data, horizontal scaling, high throughput
- Scale: 100K-1M QPS with sharding
- Strength: Easy sharding, flexible schema, massive scale
- Trade-off: Eventual consistency, limited transaction support

**Elasticsearch (Search)**
- Best for: Full-text search, analytics, logs
- Scale: Real-time search across billions of docs
- Strength: Fast text search, aggregations
- Trade-off: Not for transactional data

### 2. Read/Write Splitting

```
Write Path:
1. Client writes to Primary
2. Primary writes to disk
3. Primary replicates to Replicas
4. Return success

Read Path:
1. Client reads from Replica
2. Replica responds (low latency)
3. Primary unaffected

Benefit: 10x read throughput
Requirement: Tolerate temporary stale reads
```

### 3. Sharding Strategies

**Range Sharding**
```
Shard by customer_id range:
- Shard 1: cust_id 1-100K
- Shard 2: cust_id 100K-200K
- Shard 3: cust_id 200K-300K

Pro: Easy range queries
Con: Uneven distribution if data skewed
```

**Hash Sharding**
```
Shard by hash(customer_id):
- hash(123) % 4 = Shard 0
- hash(456) % 4 = Shard 2

Pro: Even distribution
Con: Range queries require scatter-gather
```

**Directory-Based Sharding**
```
Lookup table: customer_id → shard_id
- 100 → Shard 2
- 200 → Shard 1

Pro: Flexible redistribution
Con: Lookup overhead
```

### 4. Replication

**Primary-Replica (Master-Slave)**
```
1 Primary, N Replicas
- Writes to Primary only
- Reads from Replicas
- Primary replicates to Replicas

Pro: Consistent writes
Con: Single point of failure for writes
```

**Multi-Master**
```
All nodes accept writes
Nodes replicate to each other
Conflict resolution needed

Pro: High availability, no SPOF
Con: Complexity, potential conflicts
```

### 5. Consistency Models

**Strong Consistency (ACID)**
```
Write succeeds → all readers see new value immediately

Example: Banking (must see current balance)
Trade-off: Slower (coordination overhead)
```

**Eventual Consistency**
```
Write succeeds → replicas sync within seconds
Brief window: readers see old value

Example: Social likes (stale data acceptable)
Trade-off: Fast, but temporary inconsistency
```

## Data Models by Service

### Orders (Write-Intensive, High-Consistency)
```
Storage: PostgreSQL + primary-replica
Sharding: By customer_id
Consistency: Strong (ACID)
Read replicas: Yes (analytics)
Archival: Orders > 1 year to cold storage

Rationale: Financial data, must be correct
```

### Product Catalog (Read-Heavy, Mutable)
```
Storage: PostgreSQL + cache (Redis)
Sharding: By category or range
Consistency: Eventual (cache lag OK)
Cache: 95% of reads hit cache
Search: Elasticsearch for full-text

Rationale: Read 1000x more than write
```

### User Sessions (High-Throughput, Volatile)
```
Storage: Redis (not durable database)
TTL: 24-48 hours
Consistency: Eventual (session loss acceptable)
No replication needed (ephemeral)

Rationale: Temporary state, massive throughput
```

### Events/Logs (Write-Intensive, Immutable)
```
Storage: Kafka (streaming) + time-series (InfluxDB)
Retention: 30 days in Kafka, 1 year in InfluxDB
Sharding: By timestamp partition
Consistency: None required (immutable)

Rationale: Append-only, archive old data
```

## Scalability Patterns

### Vertical Scaling (Buy Bigger Server)
```
DB Server: 8 cores, 64GB RAM → 16 cores, 256GB RAM
Throughput: 10K QPS → 20K QPS
Cost: 5x infrastructure cost
Limit: Hardware ceiling (~100K QPS max)
```

### Horizontal Scaling (Add More Servers)
```
1 DB → 4 DB shards
Throughput: 10K QPS → 40K QPS
Cost: 4x server cost
Limit: No theoretical limit
Implementation: Requires application sharding logic
```

### Time-Series Data Optimization
```
Problem: Events table grows 1GB/day
1 year = 365GB, queries slow

Solution: Partitioning by date
- 2026-01-events (Jan)
- 2026-02-events (Feb)
...

Queries on recent data only touch small partition
Archive old partitions
Queries: 100x faster
```

## Query Optimization

```
1. Indexes on frequently queried columns
   SELECT * FROM orders WHERE customer_id = ?
   → Add INDEX on customer_id

2. Denormalization for read-heavy queries
   Store frequently-accessed calculated values

3. CQRS (Command Query Responsibility Segregation)
   - Write normalized (ACID)
   - Read denormalized (fast aggregations)

4. Query caching
   Cache query results, invalidate on write

5. Connection pooling
   Reuse DB connections instead of creating new
```

## Backup & Disaster Recovery

```
Backup Strategy:
1. Daily snapshot backups
2. Continuous replication to standby
3. Weekly offline backups (cold storage)

Recovery:
- Point-in-time recovery from snapshots
- Hot standby (seconds to failover)
- Cold restore from offline backups (hours)

RTO: Recovery Time Objective (target: < 5 minutes)
RPO: Recovery Point Objective (target: < 1 minute)
```

## Monitoring

```
Metrics:
- QPS (queries per second)
- Query latency (p50, p99)
- Connection count
- Replication lag
- Disk usage, memory usage
- Slow queries

Alerts:
- QPS > 80% capacity
- Query latency p99 > 100ms
- Replication lag > 10s
- Disk > 85% full
```

---

**Pattern Version:** 1.0 | **Status:** Production Pattern | **Scalability:** 1K → 100M+ QPS possible

