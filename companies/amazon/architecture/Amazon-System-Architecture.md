# Amazon System Architecture - Deep Dive

**Scale:** 500M+ SKUs | $600B+ GMV | 100+ AWS Regions | 200K+ Employees

---

## Microservices Breakdown (100+ services)

### Core Commerce Services
```
Order Service → Inventory Service → Fulfillment Service
     ↓                                    ↓
Payment Service ← Notification Service ← Analytics
```

**Key services:**
- Order Management (createOrder, cancelOrder, trackOrder)
- Inventory Management (reserve, allocate, restock)
- Fulfillment (pick, pack, ship, deliver)
- Catalog (product data, search, recommendations)
- Payment Processing (Visa, Mastercard, Amazon Pay, local methods)
- Returns Management (RMA, refunds, reverse logistics)
- Pricing Engine (dynamic pricing, regional pricing)
- Fraud Detection (real-time scoring)
- Recommendation Engine (personalization at scale)
- Analytics (user behavior, product trends, conversions)

**Infrastructure:**
- 1000+ microservices total
- 200+ API Gateway instances
- Independent scaling per service
- Separate databases (polyglot persistence)

---

## Data Architecture

### Multi-Region Database Strategy
```
Primary regions:
- US East (Virginia) - North America hub
- EU West (Ireland) - Europe hub
- Asia Pacific (Tokyo, Singapore) - Asia hubs
- Multiple availability zones per region

Replication:
- Synchronous replication (critical data)
- Asynchronous replication (analytics data)
- Cross-region failover (disaster recovery)
```

### Sharding Strategy
```
By Customer:
- Shard key: customer_id
- 1000+ shards across data centers
- Customer data stays in shard
- Prevents hot spots (one customer isolated)

By Product:
- Shard key: product_id
- Catalog distribution
- Local caching per shard

By Time:
- Order data sharded by date
- Old orders archived
- Performance: Recent orders fast, archived slower
```

### Caching Layers
```
Layer 1: CDN (CloudFront)
- Static assets cached globally
- TTL: 1 day - 1 year (versioned files)
- 400+ edge locations

Layer 2: ElastiCache (Redis/Memcached)
- Product data (hot items)
- User sessions
- Recommendations (personalization)
- TTL: 5 minutes - 24 hours

Layer 3: Application cache
- In-memory (Java/Python objects)
- Process-level caching
- TTL: Seconds

Result: 99%+ cache hit rate for popular products
```

---

## Event-Driven Architecture

```
Order placed event:
1. OrderService publishes: order.created
2. Kafka topic: orders (1000+ partitions)
3. Consumers:
   - Inventory: Reserve stock
   - Fulfillment: Create pick list
   - Payment: Process payment
   - Analytics: Log event
   - Recommendation: Update user profile
   - Notification: Send confirmation

All asynchronous (order returns immediately)
Parallel processing (multiple services act independently)
Failure handling: Dead letter queue + manual reconciliation
```

---

## Search & Discovery

### Elasticsearch Infrastructure
```
10B+ product documents indexed
- Product name, description, keywords
- Reviews, ratings, sentiment
- Inventory status (in stock, low stock, out)
- Price (dynamic, regional)

Ranking factors:
1. Relevance (TF-IDF, BM25)
2. Customer reviews (4.5+ rated higher)
3. Sales velocity (trending boosts ranking)
4. Personalization (based on browsing history)
5. Ad placement (sponsored products)

Performance:
- Query response: < 200ms p99
- 1M+ queries/second peak
- Auto-scaling: Add nodes on load
```

---

## Payment Processing

### Multi-Gateway Architecture
```
Visa/Mastercard (60%):
- Stripe/processor routing
- PCI compliance
- 3D Secure validation

Amazon Pay (20%):
- One-click checkout
- Stored payment method
- Lower friction

Local methods (20%):
- UPI (India)
- WeChat Pay (China)
- Alipay (Asia)
- Bank transfer (Europe)
- Local wallets
```

### Risk & Fraud Management
```
Real-time scoring:
- Velocity checks (# orders per minute/hour)
- Device fingerprinting
- Address verification
- AVS (Address Verification System)
- ML model (trained on millions of transactions)

Action:
- Score < 10: Approve
- Score 10-50: Approve with monitoring
- Score 50-80: Request additional verification
- Score > 80: Decline or manual review
```

---

## Fulfillment Network

### Distribution Centers (Logistics)
```
500+ fulfillment centers globally:
- US: 150+ centers
- Europe: 100+ centers
- Asia: 150+ centers
- India: 50+ centers

Placement strategy:
- Cover 95% of population within 2 days
- Regional clustering (cheaper labor, lower costs)
- Redundancy (multiple centers per major city)

Capacity: 500M+ daily throughput potential
```

### Fulfillment Flow
```
1. Order placed
2. Route to nearest FC with stock
3. Pick from bin (RFID, conveyor systems)
4. Pack with instructions
5. Label with carrier (UPS, FedEx, Amazon Logistics)
6. Ship
7. Real-time tracking
8. Delivery (next day, same day options)
9. Return processing (if needed)
```

---

## Scalability Patterns

### Auto-Scaling
```
Metrics monitored:
- Requests per second
- CPU utilization (target: 70%)
- Memory usage
- Queue depth

Scaling decisions:
- Scale up: If metric > threshold for 2 minutes
- Scale down: If metric < threshold for 10 minutes
- Cooldown: 60 seconds before next scale

Result: Handle 10x traffic spikes without manual intervention
```

### Database Scaling
```
As data grows:
- Add more shards (horizontal scaling)
- Increase replica count
- Archive old data (cold storage)
- Use read replicas for analytics queries

Performance: Linear scaling possible (O(1) response time)
```

---

## Key Technologies

- **Language:** Java (primary), Python, Go
- **Database:** DynamoDB (NoSQL), RDS (SQL), Redshift (Analytics)
- **Cache:** ElastiCache (Redis, Memcached)
- **Search:** Elasticsearch, Solr
- **Messaging:** SQS, SNS, Kafka
- **APIs:** REST, GraphQL
- **Infrastructure:** AWS (EC2, Lambda, ECS, Kubernetes)
- **CDN:** CloudFront

---

**Architecture Philosophy:** Decoupled, independently scalable, fault-tolerant, multi-region ready

