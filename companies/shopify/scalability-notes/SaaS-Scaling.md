# Shopify Scalability: SaaS Horizontal & Vertical Scaling

## Horizontal Scaling: Add Merchants
- Multi-tenancy: Shared infrastructure
- Database sharding: By merchant_id
- Load balancing: Route to appropriate shard
- Capacity: Add shards as needed (linear)
- Result: 1M+ merchants without redesign

## Vertical Scaling: Merchant Growth
- Merchant growing 10x: Allocates more resources
- Resource reallocation: Dedicated servers if needed
- Performance: Maintains <500ms page load
- Cost structure: Merchant pays more (higher plan)

## Merchant Scale Examples

### Small Merchant ($10K/year revenue)
- Plan: $29/month
- Infrastructure: Shared servers
- Database: Shared shard
- Performance: <500ms load time
- Cost to Shopify: ~$5/month per merchant

### Medium Merchant ($500K/year revenue)
- Plan: $79/month
- Infrastructure: Dedicated shard
- Database: Private database
- Performance: <300ms load time
- Cost to Shopify: ~$50/month per merchant

### Large Merchant ($10M/year revenue)
- Plan: $299+/month (custom pricing)
- Infrastructure: Multiple dedicated servers
- Database: Dedicated cluster
- Performance: <100ms load time
- Cost to Shopify: ~$500+/month per merchant

## Profitability Per Merchant
| Merchant Size | Revenue | Cost | Margin |
|---|---|---|---|
| $10K | $29/mo | $5 | $24 (83%) |
| $500K | $79/mo | $50 | $29 (37%) |
| $10M | $300/mo | $500 | -$200 (-67%) |

Note: Large merchants receive custom rates (net positive)

## Infrastructure Efficiency
- Underutilized: Small merchants over-provisioned (acceptable loss leader)
- Optimized: Medium merchants (highest margin)
- Premium: Large merchants (custom rate negotiation)
- Result: Portfolio optimization

## Global Distribution
- Data centers: Multi-region (USA, EU, APAC)
- Replication: Per-merchant database replicated
- Latency: <200ms globally (local DC)
- Compliance: GDPR, local data residency

## Cost Structure
- Fixed costs: $500M+ annually (infrastructure, R&D)
- Variable costs: ~$20-100 per merchant annually
- Gross margin: 70%+ at scale
- Operating margin: Improving (approaching 20%+)

## Auto-Scaling Technology
- Kubernetes: Orchestrate containers
- Auto-scaling: Based on CPU/memory/requests
- Predictive: Machine learning forecasts peaks
- Cost: Optimize cloud spend (~30% of revenue)

## Future Scalability
- 2M+ merchants: Likely by 2030
- 1T+ GMV: Platform goal
- Edge computing: Reduce latency further
- AI: Merchant support automation
- Result: Sustainable competitive advantage

