# eBay Global Scalability

## Multi-Region Architecture
- North America: Virginia, Oregon
- Europe: Ireland, Frankfurt
- Asia: Tokyo, Singapore, Sydney
- Latency target: <200ms globally

## Database Sharding
- Primary: Seller ID sharding (30M shards)
- Secondary: By region (geo-sharding)
- Replication: Multi-region for disaster recovery

## Capacity Growth
- 2020: 10M listings
- 2023: 200M listings (20x growth)
- 2026: 350M listings (1.75x growth)
- Infrastructure: Scaled linearly with growth

## Peak Load Management
- Black Friday: 10x normal traffic
- Infrastructure provision: Pre-positioned servers
- Result: No downtime during peak shopping

## Cost Optimization
- Reserved capacity: For baseline
- Spot instances: For surge capacity
- Result: Cost increase <50% for 10x capacity

