# Lazada: LazMall Flash Sale Infrastructure

## Scale Challenge
- 11.11 Single Day Sale: $5B+ in 24 hours
- Normal traffic: 10K transactions/sec
- Peak traffic: 1M transactions/sec (100x)
- 6 countries, simultaneous sales

## Pre-Event Infrastructure (1 month before)
1. Capacity planning (traffic forecast)
2. Server provisioning (500+ servers per country)
3. Database scaling (add replicas)
4. CDN cache warming
5. Seller coordination (inventory prep)

## Event Day Infrastructure
1. **T-1 hour:** Bring all servers online
2. **T-0:** Auto-scaling enabled
3. **T+0 to T+24:** Real-time scaling based on traffic
4. **T+24:** Scale down over 4 hours

## Technical Stack
- Load balancing: By product category
- Database: Sharded by seller_id + time
- Cache: 99%+ hit rate for popular items
- Message queue: Decouple payment from fulfillment

## Cost Optimization
- Normal infrastructure: $5M/month
- Flash sale extra cost: $20M for 24-hour event
- Amortized: Critical revenue ($5B) justifies cost

## Results
- No downtime during peak
- 99.99% uptime maintained
- Avg transaction time: <500ms
- All orders processed successfully

