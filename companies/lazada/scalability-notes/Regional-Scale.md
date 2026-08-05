# Lazada Scalability: Regional Multi-Country Operations

## Data Architecture
- Per-country databases (data residency)
- Replicas in each country (local redundancy)
- Central analytics database (read-only aggregation)
- Nightly sync: Country DB → Central analytics

## Payment Processing
- Local payment gateways per country
- Thailand: TrueMove, AIS, bank transfers
- Vietnam: Bank transfers, e-wallets
- Philippines: GCash, PayMaya
- Indonesia: OVO, Dana, bank transfers
- Malaysia: Online banking
- Singapore: Credit cards, PayNow

## Inventory Management
- Per-country inventory (regional separation)
- Seller SKU per country (allow different pricing)
- Warehouse: Per country (stock positioning)
- Automated reorder: When inventory low

## Search & Discovery
- Elasticsearch per country (local language)
- Recommendations: Local seller/product bias
- Trending: Per-country trending section
- Result: Relevant to regional preferences

## Infrastructure Scaling
- Servers per country: Scale independently
- Load balancing: Geographic-aware (route to closest)
- Peak capacity: Provision for local peak (11.11)
- Cost: ~$50M annually operational ($20M event spikes)

## Performance SLA
- Page load: <2 seconds (p95)
- Search: <500ms (p95)
- Payment: <2 seconds (p95)
- Delivery accuracy: 95%+ on-time

## Growth Capacity
- 2020: $1B GMV
- 2023: $10B GMV (10x)
- 2025: $15B+ GMV (1.5x)
- Infrastructure: Linear scaling maintained

