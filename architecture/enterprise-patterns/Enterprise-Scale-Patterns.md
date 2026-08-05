# Enterprise Scale Patterns

**Status:** Enterprise Architecture Pattern | **Priority:** HIGH | **Scope:** SaaS, Marketplace, Global

---

## Pattern 1: Multi-Tenancy Architecture

**What is Multi-Tenancy?**
One software instance serves multiple customers (tenants). Each tenant has isolated data but shares infrastructure. Used by SaaS ecommerce platforms (Shopify, BigCommerce, WooCommerce).

**Example:** Shopify platform:
- Tenant 1: Alice's Boutique store
- Tenant 2: Bob's Electronics store
- Shared infrastructure: 1 database, 1 code base, 1 API

**Isolation Models:**

### 1. Database-per-Tenant (Most Secure)
```
Each tenant has separate database:
- Alice: ecommerce_alice_db
- Bob: ecommerce_bob_db
- Charlie: ecommerce_charlie_db

Pros:
- Maximum isolation (hacked tenant doesn't affect others)
- Scaling: Scale each tenant independently
- Compliance: Easier GDPR/CCPA compliance (data separation)

Cons:
- Cost: More database instances
- Complexity: Database management overhead
- Disaster recovery: More backups to manage

Best for: Enterprise SaaS, compliance-heavy
```

### 2. Schema-per-Tenant (Balanced)
```
One database, separate schemas:
- Database: ecommerce_main
  - Schema alice: All Alice's tables
  - Schema bob: All Bob's tables
  - Schema charlie: All Charlie's tables

Pros:
- Lower cost than DB-per-tenant
- Reasonable isolation
- Management easier than DB-per-tenant

Cons:
- Single database is single point of failure
- Database changes affect all tenants

Best for: Mid-market SaaS
```

### 3. Row-Level Isolation (Cost-Optimized)
```
Single database, single schema, tenant_id column:

orders table:
- order_id, tenant_id, customer_id, total
- 1001, alice_tenant, cust_1, $100
- 1002, bob_tenant, cust_1, $200
- All queries: WHERE tenant_id = 'alice_tenant'

Pros:
- Lowest cost (single DB instance)
- Simple database operations
- Easiest scaling (most queries the same)

Cons:
- Lowest isolation (single query bug exposes all tenants)
- Performance: Tenants compete for database resources
- Compliance: Harder GDPR compliance

Best for: High-volume, low-risk SaaS
```

**Choosing Isolation Model:**
```
Trust risk: HIGH (enterprise data) → Database-per-Tenant
Compliance needs: GDPR/HIPAA → Database-per-Tenant
Budget: Limited → Row-Level Isolation
Scale: 1000+ tenants → Row-Level Isolation
Risk tolerance: Low → Database-per-Tenant
```

**Multi-Tenant Architecture Diagram:**

```
┌───────────────────────────────────────────────────────────┐
│              Shared API Gateway                           │
│           (Routing by tenant_id header)                   │
└────────────┬────────────────────────────────┬─────────────┘
             │                                │
      ┌──────▼──────┐               ┌────────▼────────┐
      │ Tenant 1    │               │ Tenant 2        │
      │ Alice's     │               │ Bob's           │
      │ Boutique    │               │ Electronics     │
      └──────┬──────┘               └────────┬────────┘
             │                                │
    ┌────────▼────────────────────────────────▼─────┐
    │      Application Layer (Shared code)           │
    │    - Same services for all tenants            │
    │    - Tenant context passed in requests        │
    └────────┬────────────────────────────┬─────────┘
             │                            │
    ┌────────▼──────┐          ┌──────────▼────────┐
    │ Alice's DB    │          │ Bob's DB          │
    │ (Isolated)    │          │ (Isolated)        │
    └───────────────┘          └───────────────────┘
```

**Managing Tenant Configuration:**

```
Each tenant has configuration:
- Store name: "Alice's Boutique"
- Logo/branding
- Payment gateway integration
- Shipping partners
- Tax rules
- Email configuration
- Custom domain
- Permissions/roles
- Subscription tier (Basic, Pro, Enterprise)

Stored in:
- Tenant metadata table
- Redis cache (fast access)
- Invalidate on change

Per-request:
- API includes: X-Tenant-ID: alice_tenant
- Middleware extracts and validates
- Tenant config loaded from cache
- Available to all services
```

---

## Pattern 2: Marketplace Governance

**What is a Marketplace?**
Multiple sellers on one platform. Platform handles transactions, trust, disputes.
Examples: eBay, Amazon Marketplace, Etsy.

**Seller Lifecycle:**

```
1. Seller Registration
   - Apply to marketplace
   - Verification: Government ID, bank account
   - KYC (Know Your Customer) checks
   - Background check (for financial platforms)
   - Manual approval by marketplace team

2. Seller Onboarding
   - Setup store profile
   - Add products
   - Configure shipping
   - Set up payment (direct deposit)
   - Training videos
   - Initial transaction limits (low)

3. Active Selling
   - Listings live
   - Customers can purchase
   - Transactions normal

4. Performance Monitoring
   - Track metrics:
     - Response time to buyers
     - Defect rate (returns, complaints)
     - Shipping on-time rate
     - Rating/reviews
   - Ratings: 1-5 stars
   - Penalties if below threshold (e.g., < 4.0 stars)

5. Compliance & Trust
   - Seller must maintain standards
   - Reviews/ratings transparent
   - Complaints handled by platform
   - Appeal process if seller disputes

6. Termination (if needed)
   - Policy violation
   - Consistent poor performance
   - Legal issues
   - Seller can appeal
```

**Seller Tiers & Incentives:**

```
Tier 1: New Seller
- Transaction limits: 10/month, $1000/month
- Commission: 15%
- Dispute resolution: 90 days
- No payment hold
- Goals: Build reputation

Tier 2: Established (100+ sales, 4.5+ rating)
- Transaction limits: Unlimited
- Commission: 12%
- Dispute resolution: 60 days
- Payment: Next day
- Benefits: Featured listings, marketing

Tier 3: Top Seller (1000+ sales, 4.8+ rating)
- Commission: 8%
- Dispute resolution: 30 days
- Payment: Same day
- Benefits: Priority support, featured placement
- Eligibility: Invite-only

Benefits motivate sellers to provide excellent service
```

**Dispute Resolution:**

```
Customer: "Item not as described"
1. Customer opens case in marketplace
2. Seller gets 48 hours to respond
3. Communication tools provided
4. If no agreement:
   - Marketplace reviews evidence
   - Customer provides: Photos, descriptions of problem
   - Seller provides: Evidence of item description
   - Marketplace decides: Refund, partial refund, or seller wins
5. Enforcement: Platform collects from seller if needed

Process protects:
- Customers: Recourse if seller fraud
- Sellers: Protection from false claims
- Platform: Trust enables transactions
```

**Quality Control:**

```
Automated monitoring:
- Reviews: Flag if customer reports item not as described
- Metrics: Monitor seller performance daily
- Red flags:
  - High return rate (> 10%)
  - Low ratings (< 3 stars)
  - Slow responses (> 2 days)
  - Multiple complaints
  - Policy violations

Manual review:
- Team investigates flagged sellers
- Education: Coach on best practices
- Penalties: Warnings, commission increase, suspension
- Severe: Termination
```

**Commission Management:**

```
Seller lists item: $100
Platform commission: 15% = $15
Seller gets: $85

Marketplace:
- Collects payment from customer
- Deposits $85 to seller (after hold period)
- Keeps $15
- Handles disputes (platform money at risk)
- Pays payment processors fees
```

---

## Pattern 3: Data Residency & Sovereignty

**What is Data Residency?**
Data must be stored in specific geographic locations per local laws.

**Examples:**
```
GDPR (EU): Customer data must stay in EU
- No data transfer outside EU without legal basis
- Fine: €20M or 4% revenue

China Data Laws: Customer data must stay in China
- Local servers required
- Government may request data
- Foreign companies struggle

India Data Laws: Certain data must stay in India
- Payment information
- Personal information
- Government ID data

Brazil (LGPD): Similar to GDPR, data in South America
```

**Multi-Region Architecture:**

```
Global Ecommerce Platform:
┌─────────────────────────────────┐
│ US Customer                     │
│ Data → US East (Virginia)       │
│ Processing: US Region           │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│ EU Customer                     │
│ Data → EU West (Ireland)        │
│ Processing: EU Region           │
│ GDPR compliant                  │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│ China Customer                  │
│ Data → China (Beijing/Shanghai) │
│ Processing: China Region        │
│ Separate infrastructure needed  │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│ India Customer                  │
│ Data → India (Mumbai)           │
│ Processing: India Region        │
└─────────────────────────────────┘
```

**Implementation:**

```
Data Classification:
1. Customer personal data (name, address)
   - Residency: Strict (must stay in customer country)
   - Replication: Not allowed across borders

2. Transaction data (order, payment)
   - Residency: Strict (must stay in customer country)
   - Audit: May need to replicate to headquarters

3. Analytics data (aggregated, anonymized)
   - Residency: Flexible (can move for analysis)
   - Compliance: Anonymized, so different rules

4. Backup data
   - Residency: Flexible (but must encrypt)
   - Recovery: Must be able to restore locally

Customer in France (GDPR):
- Personal data: EU servers only
- Analytics: Can aggregate to US (anonymized)
- Backup: EU or encrypted elsewhere
- Audit: Annual residency review
```

**Compliance & Monitoring:**

```
Monitoring:
- Data flow logs: Track where data moves
- Geographic enforcement: Prevent data outside region
- Automated checks: Daily residency verification
- Audit reports: Quarterly for regulators

Penalties for violation:
- GDPR: €20M or 4% revenue
- China: License revocation
- India: Fines + block access
- Brazil (LGPD): Similar to GDPR

Prevention:
- Data classification: Know what's where
- Automation: Prevent manual transfers
- Testing: Quarterly disaster recovery drills in each region
- Documentation: Prove compliance
```

**Challenges:**

```
Complexity: Multiple databases, data centers
Cost: Separate infrastructure per region
Latency: Data may be far from users
Disaster recovery: Failover harder across regions
Compliance: Different rules per region
Operations: More moving parts
```

**When to Implement:**
```
✓ IF: Large customer in regulated market (EU, China, India)
✓ IF: Customer contracts require residency
✓ IF: Business model depends on trust
✓ IF: Revenue > $10M (can afford complexity)

✗ IF: Mostly US-based customers
✗ IF: Budget-constrained startup
✗ IF: Operations team < 20 people
```

---

**Guide Version:** 1.0 | **Status:** Enterprise Pattern | **Scope:** SaaS, Marketplace, Global Scale

