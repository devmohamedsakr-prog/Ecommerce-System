# Shopify: Multi-Tenant SaaS Architecture

## Platform Design
- Merchants: 1M+ independent stores
- Infrastructure: Shared (cost efficient)
- Isolation: Per-merchant database sharding
- Scale: From $10K to $10M+ annual merchants

## Multi-Tenancy Model
- Shared compute: Servers pool
- Per-merchant database: Isolated (Postgres shards)
- Per-merchant storage: S3 buckets
- Per-merchant API quota: Rate limited

## Data Isolation Strategy
- Database sharding: By merchant_id
- 1000+ shards (distributed globally)
- Each shard: Dedicated database server
- Replication: Multi-region per shard
- Result: Isolation + high availability

## Scalability Approach
1. **Horizontal scaling:**
   - Add new shards as merchants grow
   - Load balancer routes to shard
   - No reshuffling needed

2. **Vertical scaling:**
   - Add compute resources per merchant
   - If merchant grows 10x, allocate more servers
   - Seamless to customer

3. **Resource pooling:**
   - Underutilized merchants: Share servers
   - High-volume merchants: Dedicated servers
   - Dynamic allocation: Based on actual usage

## Performance
- Merchant store load: <500ms p95
- API latency: <200ms p95
- Search queries: <500ms p95
- Payment processing: <2 seconds p95

## Infrastructure
- Compute: Kubernetes (auto-scaling)
- Database: PostgreSQL (sharded)
- Cache: Redis (per-shard)
- CDN: Cloudflare (global)
- Storage: S3 (per-merchant)

## Cost Model
- Fixed cost: $29-299/month (base)
- Transaction fee: 2.9% + $0.30 per sale
- App store: 30% commission on apps
- Result: Scales with merchant success

## Reliability
- SLA: 99.99% uptime
- Failover: Automatic (multi-region)
- Backup: Daily snapshots per merchant
- Recovery: <1 hour RTO

