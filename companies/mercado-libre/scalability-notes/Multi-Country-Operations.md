# Mercado Libre Scalability: Multi-Country Operations

## Data Architecture
- Per-country databases (regulatory requirement)
- Brazil: Separate DB (LGPD compliance)
- Mexico: Separate DB (local data centers)
- Colombia, Chile, Peru: Regional DB (cost optimization)
- Central analytics: Aggregated (read-only)

## Multi-Currency Support
- 5 major currencies: BRL, ARS, MXN, COP, CLP
- Real-time exchange rates (updated hourly)
- Buyer sees local currency (USD in Venezuela)
- Seller paid in local currency (option for USD hold)
- Central accounting: All converted to BRL for reporting

## Payment Processing
- Per-country payment gateways
- Local methods priority (not card-first)
- Pix in Brazil: 50%+ of transactions
- Bank transfer standard: All countries
- Cash payment: Mexico (OXXO)
- Digital wallets: Growing adoption

## Seller Support
- Local teams per country
- Language-specific help (Spanish, Portuguese)
- Payment method education
- Regulatory compliance training
- Regional incentive programs

## Infrastructure Scale
- Servers: Regional deployment (Brazil largest)
- CDN: Local content caching per country
- Database: Sharded by seller_id within country
- Peak capacity: Holiday season (November-December)

## Regulatory Compliance
- LGPD (Brazil): Strict privacy requirements
- GDPR-light (some countries following Brazil)
- Tax compliance: Different rates per country
- Anti-fraud: Regional customization
- AML/KYC: Enhanced for financial products

## Performance SLA
- Page load: <2 seconds (p95)
- Search: <500ms (p95)
- Payment: <3 seconds (p95)
- Delivery: 95%+ on-time (regional goal)

## Growth Trajectory
- Users: 50M (2020) → 80M (2026)
- GMV: $2B (2020) → $8B+ (2026)
- Infrastructure: Scaled linearly with growth
- Cost optimization: Achieved 20% reduction per transaction

