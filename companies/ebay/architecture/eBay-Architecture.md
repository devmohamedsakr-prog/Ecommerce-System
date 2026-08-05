# eBay System Architecture

**Scale:** 30M+ sellers | 350M+ listings | 2B+ searches/month

## Core Services
- Auction Engine (real-time bidding, proxy logic)
- Inventory Management (seller stock tracking)
- Payment Processing (Managed Payments)
- Seller Trust System (feedback scoring)
- Search (Elasticsearch 350M+ documents)
- Recommendation Engine (personalization)

## Database Strategy
- Sharding by seller_id (30M+ shards)
- Feedback scores (cached, updated real-time)
- Auction data (time-based partitioning)
- Archive: 2+ year old data

## Caching
- Product cache: 95%+ hit rate
- Seller reputation: Redis (updated per transaction)
- Search queries: Memcached (popular searches)

## Load Balancing
- Auction ending surge: Auto-scale 10x
- Peak: 1M+ bid/second
- Geographic: Multi-region (US, EU, APAC)

