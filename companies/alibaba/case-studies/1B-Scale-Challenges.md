# Alibaba Case Study: Scaling to 1B+ Products

## Challenge 1: Search Infrastructure at 1B Scale
**Problem:** 1B+ products, <500ms search required

**Solution:**
- Elasticsearch: Distributed (1000+ nodes)
- Sharding: By product ID
- Caching: Hot products cached
- Result: <200ms p95 response time

**Key insight:** Linear scaling with node count (each node adds capacity)

## Challenge 2: Payment Processing at 1M TPS
**Problem:** Normal systems break at 100K TPS

**Solution:**
- Stateless services: No session affinity needed
- Sharding: By payment_id (divide load)
- Message queue: Decouple from processing
- Eventually consistent: Payment confirmed after 1-2 seconds
- Result: Handle 1M TPS without queues

**Key insight:** Async payment processing (not real-time)

## Challenge 3: Inventory Real-Time Sync
**Problem:** 500M+ SKUs, sales consuming inventory instantly

**Solution:**
- Optimistic locking: Reduce query coordination
- Event-driven: Sales events trigger inventory update
- Caching: Inventory cached, updated per sale
- Oversell handling: Accept 0.1% oversell (cheaper than preventing)
- Result: Inventory accurate to within 100 items (1B items ±0.01%)

**Key insight:** Perfection isn't needed, accuracy good enough wins

## Challenge 4: Global Distribution
**Problem:** 1000+ fulfillment centers, real-time routing

**Solution:**
- Regional warehouses: Stock positioned strategically
- Dynamic routing: Route to nearest FC (algorithm)
- Demand forecasting: Stock by region/product
- Cross-region rebalancing: Move stock preemptively
- Result: 2-day delivery to 95% of population

## Challenge 5: Logistics at Scale
**Problem:** 20M+ shipments daily, complex last-mile

**Solution:**
- In-house fleet: Cainiao handles 40% of volume
- Partner network: 100+ local couriers
- Pickup points: 100K+ self-service pickup locations
- Result: Cost $1.50 per shipment (vs $5-10 if outsourced)

## Financial Impact
- Scale benefits: $1.50 cost vs $5-10 competitors
- Profit margin: 2-3% higher than competitors
- Reinvestment: Infrastructure < profit gains
- Result: Sustainable competitive advantage

