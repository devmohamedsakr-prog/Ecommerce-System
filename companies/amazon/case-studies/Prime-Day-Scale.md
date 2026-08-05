# Amazon Case Study: Prime Day at 1M+ TPS Scale

## Prime Day Overview
- Annual event: 48 hours (mid-July)
- Purpose: Drive Prime membership + seasonal sales
- Scale: $10B+ in 48 hours
- Growth: 30%+ YoY consistently

## 2026 Prime Day Metrics
- Orders: 500M+ (2-day total)
- Peak TPS: 1M+ transactions/second
- Data processed: 100TB+ (clickstreams, orders, inventory)
- Concurrent users: 500M+ active
- Flash deals: 10K+ throughout event

## Infrastructure Preparation (3 months)
1. **Capacity Planning:**
   - Historical data: 2020-2025 Prime Days
   - Growth forecast: 30%+ growth expected
   - Capacity target: 2x normal traffic
   - Cost: $500M+ infrastructure provisioning

2. **Load Testing:**
   - Synthetic load: 1M TPS test runs
   - Chaos engineering: Failure scenarios tested
   - Database stress: Query optimization
   - Result: Confidence in 1M+ TPS capacity

3. **Seller Onboarding:**
   - Inventory: Sellers stock up 2x-3x normal
   - Pricing: Sellers prepare deals
   - Marketing: Seller participation drives promotion
   - Result: 1M+ sellers participating

## Event Day Operations

### Hour 0-1: Soft Launch
- 00:00: Sale starts
- Prime member email: Early notification
- App notification: Push to 200M+ Prime members
- Website surge: 1B+ impressions in 1 minute

### Hour 1-12: Day One Peak
- Peak TPS: 700K-1M TPS sustained
- Flash deals: Staggered (prevent all sales at once)
- Auto-scaling: Add 1000+ servers per minute
- Monitoring: Real-time dashboard tracking

### Hour 12-24: Day Two
- Momentum: Continues from day 1
- New deals: Refresh inventory
- International: Global regions join
- Peak again: Similar to day 1

### Hour 24-48: Final Stretch
- Urgency: "Last chance" messaging
- Flash deals: Final clearance items
- Mobile surge: Last-minute mobile shopping
- Peak final 6 hours: 500K TPS

## Lessons Learned

### Scalability Victory
- 1M+ TPS achieved: Infrastructure held
- Auto-scaling: Dynamically added 2000+ servers
- Zero downtime: 99.99%+ availability maintained
- Performance: Page load <500ms p95

### Peak Management
- Flash deals: Staggered prevented gridlock
- Inventory: Real-time sync maintained (99.95% accuracy)
- Payment: All processed in real-time (no queue)
- Shipping: Orders routed to 200+ FCs immediately

### Financial Impact
- Revenue: $10B+ (2-day total)
- Profit: Likely $1-2B (10-20% margin)
- Prime conversion: 2M+ new members (estimated)
- LTV of new Prime: $500+ each = $1B+ lifetime

## Competitive Advantages
- Infrastructure: Amazon cloud-native (faster scaling)
- Logistics: 200+ FCs for same-day delivery
- Prime lock-in: Members already paid (friction-free)
- Scale: 10x larger than most competitors

