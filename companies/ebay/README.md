# eBay - Global C2C Marketplace Leader

**Market Position:** $70B+ GMV | **Employees:** 10K+ | **Countries:** 32 | **Active Sellers:** 30M+

---

## Business Model

**C2C (Consumer-to-Consumer) + B2C Hybrid**
- Auction-based (traditional eBay differentiator)
- Fixed-price listings (eBay Now, Stores)
- Business sellers (30% of GMV)
- Individual collectors/resellers (70% of GMV)

**Revenue Model:**
- Insertion fees ($0.30-$2.00 per listing)
- Final value fees (12.9% on most categories)
- Managed Payments (2.35% + $0.30)
- Premium subscriptions (Store tiers: $27.95-$299.95/month)

**Scale:**
- 350M+ active listings globally
- 20M+ new listings daily
- 2B+ product searches monthly

---

## Key Differentiators

### 1. Auction System (Original, Still Core)
```
Traditional eBay advantage:
- Collector market (rare items, vintage)
- Price discovery through bidding
- Seller trust through feedback (since 1998)
- Competitive bidding drives prices
- User engagement (emotional attachment to winning bids)

Modern strategy:
- Declining but stable (20% of GMV)
- Still attracts niche/rare items
- Maintained for customer loyalty
```

### 2. Seller Trust & Rating System
```
Feedback score (reputation):
- Positive feedback: +1
- Negative feedback: -1
- Neutral feedback: 0

Seller tiers:
- New Seller (0-50 feedback)
- Established Seller (50-1000)
- Top Rated Seller (4.8+ rating, <2% defect rate)

Consequences:
- Low ratings = Delisting
- Negative feedback = Seller suspension
- Trust is currency on eBay
```

### 3. Managed Payments (Payment Consolidation)
```
Unified payment system (launched 2021):
- Single transaction fee: 2.35% + $0.30
- Automatic seller payout (next day)
- Buyer protection: Automatic refunds
- No manual payment processing

Migration from:
- PayPal (external)
- Direct checkout
- Credit cards

Result: Higher margins for eBay
```

### 4. eBay Plus (Loyalty Program)
```
Premium membership ($10/month):
- Free shipping on eligible items
- Exclusive deals
- Priority customer service
- Price guarantee

Goal: Increase repeat purchases, LTV
```

---

## Architecture Patterns

### Catalog & Search
```
350M+ listings present scale challenge:
- Real-time indexing (new listings instantly searchable)
- Category taxonomy (19 top-level categories, 500+ subcategories)
- Search ranking (relevance, price, seller rating)
- Faceted search (condition, price range, location)

Technical:
- Elasticsearch for search (10B+ documents)
- Redis for cache (popular searches)
- ML ranking model (personalized results per user)
```

### Auction Engine
```
Real-time bidding logic:
1. Seller sets: Starting bid, reserve price, auction duration
2. Bidder places: Bid (automatic bidding up to max)
3. System logic:
   - Outbid: Notifies previous bidder
   - Increment logic: $0.05-$100 based on current bid
   - Proxy bidding: Automatic bid increments
4. Auction end: Auto-close, winner/loser notified
5. Payment: 24-48 hour payment deadline

Scale challenges:
- 50M+ concurrent auctions
- 1M+ bid/second peak load
- Accuracy critical (trust system)
```

### Seller Onboarding
```
New seller workflow:
1. Register account
2. Connect payment method
3. Verify ID (document upload)
4. Set seller policies
5. Tax info (if applicable)
6. Listing limits: 10 items, $500 total value (week 1)
7. Increase limits based on performance

Risk management:
- Fraud detection (payment history, shipping address)
- Account review (manual for high risk)
- Suspension if suspicious activity
```

---

## Global Operations

### Regional Data Centers
```
North America: Virginia, Oregon
Europe: Ireland, Netherlands
Asia-Pacific: Singapore, Tokyo, Sydney

Latency targets: < 200ms from any user
Failover: Multi-region redundancy
```

### Payment Methods (By Region)
```
USA: Credit card (70%), PayPal (20%), Managed Payments (10%)
Europe: Bank transfer (50%), Credit card (40%), PayPal (10%)
Asia: Local methods (Alipay, WeChat, bank transfer)
Latin America: Cash on delivery (40%), bank transfer (30%)
```

### Currency & Tax
```
Support: 30+ currencies
Tax calculation: By jurisdiction
B2B sellers: B2B billing (invoice-based)
```

---

## Challenges & Solutions

### Challenge 1: Fraud at Scale
```
Problem: 30M+ sellers, 350M+ listings = fraud opportunity
- Counterfeit items
- Item not as described (INAD)
- Non-delivery (seller takes payment, doesn't ship)

Solution:
- AI detection: Flag suspicious patterns
- Buyer protection: Full refund if issues
- Managed Payments: Money held 21 days
- Seller suspension: Repeat offenders immediately removed
- Partnership: Brand protection team (IP holders)
```

### Challenge 2: Auction Ending Spike
```
Problem: All bids in last 60 seconds
- Server load spike (1M bids/sec)
- Race condition (multiple winners possible)
- User timeout (server unavailable)

Solution:
- Auto-scaling: Launch extra servers 5 min before end
- Proxy bidding: Reduce last-second bids
- Load balancing: Distribute auction endings
- Queue: Hold bids if overload, process in order
```

### Challenge 3: International Shipping
```
Problem: Shipping complexity across 32 countries
- Customs, duties, tariffs
- Tracking (different carriers per region)
- Returns (reshipping internationally expensive)

Solution:
- Global Shipping Program: eBay handles international logistics
- Seller ships to distribution center (USA)
- eBay ships internationally
- Seller not liable for international issues
```

### Challenge 4: Mobile Transformation
```
Problem: Desktop-first platform, mobile world
- 60% of traffic now mobile
- Auction model complex on mobile
- App adoption slow initially

Solution:
- Native iOS/Android apps (100M+ downloads)
- Mobile-optimized listing creation
- One-tap bidding
- Push notifications (outbid alerts)
```

---

## Scalability Approach

```
Year 1: Single database, monolithic app
Year 2: Database replication (read/write split)
Year 3: Microservices (search, auction, payment separate)
Year 4: Global sharding (data by region)
Year 5: Multi-cloud (AWS, Google Cloud, on-prem)

Current: 1000+ microservices, 50+ data centers
Auction engine: Dedicated service, separate scaling
```

---

## Key Technologies

- **Search:** Elasticsearch, Solr
- **Database:** Oracle (legacy), PostgreSQL, Cassandra (new)
- **Cache:** Memcached, Redis
- **Messaging:** Kafka, RabbitMQ
- **APIs:** REST, gRPC
- **Mobile:** iOS, Android (native)
- **Cloud:** AWS primary, multi-cloud strategy

---

## Lessons for Startups

1. **Trust is currency:** Feedback score matters more than anything else
2. **Auction creates engagement:** Even if declining, users love winning
3. **Global means local:** Payment methods, language, currency critical
4. **Fraud is constant:** Need real-time detection, manual review team
5. **Mobile is table-stakes:** Desktop-first platforms struggle (learned hard)
6. **Seller tools = retention:** Stores, analytics, automation keep sellers

---

**eBay Status:** Mature, stable, focus on seller tools and international expansion

