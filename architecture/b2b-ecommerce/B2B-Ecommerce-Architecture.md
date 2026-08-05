# B2B E-Commerce Architecture Pattern

**Status:** Enterprise Architecture Pattern | **Priority:** CRITICAL | **Market:** $8T by 2027

---

## Overview

B2B ecommerce differs fundamentally from B2C. Complex pricing (contracts, volume discounts, tiered), approval workflows, long sales cycles, integration with ERP/CRM. Research: Shopify shows B2B businesses need connected systems handling customer-specific pricing and complex workflows.

## B2B vs B2C Key Differences

| Aspect | B2C | B2B |
|--------|-----|-----|
| **Buying Process** | Impulse (5 min) | Complex (weeks/months) |
| **Pricing** | Fixed for all | Per customer, per contract |
| **Quantities** | 1-10 | 100-10,000 |
| **Payment** | Credit card | Net-30, Net-60, Wire transfer |
| **Approval** | None | Multiple levels (manager, CFO) |
| **Integration** | Minimal | ERP, CRM, custom integrations |
| **Relationships** | Transactional | Long-term partnerships |

## B2B-Specific Services (Not in Core 21)

### 1. Contract Management Service
```
Manages customer-specific contracts, pricing, terms

Features:
- Store contract terms (volume discounts, payment terms)
- Pricing tiers: 1-10 units = $50, 11-100 = $45, 100+ = $40
- Contract expiration and renewal
- Early payment discounts
- Minimum order quantities

APIs:
GET /contracts/{customer_id}
POST /contracts (create new contract)
GET /pricing/{customer_id}/{product_id}
```

### 2. Approval Workflow Service
```
Multi-level purchase approvals based on amount/department

Workflow:
- Order < $1000: Auto-approved
- Order $1000-$10K: Manager approval
- Order $10K-$100K: Director approval
- Order > $100K: VP approval

Track:
- Approval chain
- Time to approval (SLA: 24 hours)
- Auto-escalation if pending too long

APIs:
POST /approvals/{order_id}/approve
POST /approvals/{order_id}/reject
GET /approvals/pending (for approver)
```

### 3. Customer-Specific Pricing Engine
```
Calculate real-time prices based on:
- Customer contract terms
- Volume purchased today + historical
- Seasonal discounts
- Payment method (wire vs credit gets discount)
- Loyalty tier

Example:
- Base price: $100
- Volume discount (100+ units): -10%
- Contract annual commitment: -5%
- Early payment discount: -2%
- Final: $83
```

### 4. Purchase Requisition Service
```
Corporate buyer workflow:
- Employee creates PO in internal system
- Exports to requisition
- Submits to procurement
- Procurement uploads to B2B portal
- Order auto-created from requisition
- Fulfillment processes
- Invoice matches PO quantity

Handles:
- Bulk uploads (Excel, CSV)
- PO matching to orders
- Variance reporting
- Exception handling
```

### 5. Credit Management Service
```
B2B buyers have credit limits

Features:
- Set customer credit limit ($100K)
- Track current balance ($65K used)
- Available credit ($35K)
- Credit hold if limit exceeded
- Automatic payment tracking
- Credit line reviews (annual)

APIs:
GET /credit/{customer_id}
POST /credit-hold (place hold)
POST /credit-release (release hold)
```

### 6. B2B Catalog Service Enhancement
```
Specific to B2B needs:
- Catalogs per customer (different products per tier)
- Volume-based specs (bulk packaging)
- Technical specifications (PDF downloads)
- Compliance certifications (ISO, safety)
- Customization options (color, configuration)
- Lead times (custom orders = 6 weeks)
```

## B2B Order Workflow

```
1. Buyer Creates Purchase Order
   - Selects products
   - Applies company-specific pricing
   - Adds custom PO number

2. Approval Process
   - Amount checked
   - Routes to appropriate approver
   - Approval workflow tracked
   - Auto-escalation if needed

3. Order Confirmation
   - Special terms applied
   - Lead time calculated
   - Commitment tracking
   - Order acknowledged to ERP

4. Fulfillment
   - Bulk picking/packing
   - Customization if needed
   - Mixed pallet for multiple SKUs
   - Shipping per customer schedule

5. Invoice & Payment
   - Net-30, Net-60 terms
   - Recurring billing support
   - Payment method options
   - Credit applied automatically

6. Loyalty & Account Management
   - Account review quarterly
   - Performance metrics
   - Growth opportunities
   - Renewal negotiations
```

## B2B Catalog Strategy

### Tiered Catalogs
```
Tier 1 (Large volumes): 500+ SKUs
Tier 2 (Medium volumes): 1000 SKUs
Tier 3 (Small businesses): 2000 SKUs

- Different pricing per tier
- Different minimum order quantities
- Different lead times
- Different support levels
```

### Technical Specifications
```
Product details:
- Dimensions & weight
- Certifications (ISO, FDA)
- Compliance info (RoHS, REACH)
- Testing reports
- Safety data sheets

Downloadable PDFs:
- Datasheets
- Installation guides
- Warranty documentation
- Technical support contacts
```

## Integration with ERP Systems

### Order to Cash Flow
```
1. Order placed in B2B portal
2. Order exported to ERP
3. ERP confirms inventory
4. ERP creates picking list
5. Fulfillment completes
6. ERP generates invoice
7. Invoice sent to customer
8. ERP tracks payment
9. Revenue recognized (ASC 606)

Key: Real-time data sync, no manual entry
```

### CRM Integration
```
Sales team uses CRM (Salesforce, HubSpot)
Customer info synced to B2B portal
- Company details
- Contacts
- Purchase history
- Previous quotes
- Account manager assigned

Portal shows:
- Account manager contact
- Company-specific pricing
- Historical orders
```

## B2B Pricing Model

### Tiered Volume Pricing
```
Product: Widgets
- 1-50: $10 each
- 51-200: $8 each
- 201-1000: $6 each
- 1000+: $4 each + quantity discount

Customer buys 100:
- Regular price: 100 × $10 = $1000
- With discount: 100 × $8 = $800
- Savings: $200
```

### Contract-Based Pricing
```
Annual contract: 50,000 units
- Q1: Buy 10,000 @ $4/unit
- Q2: Buy 15,000 @ $4/unit
- Q3: Buy 12,000 @ $4/unit
- Q4: Buy 13,000 @ $4/unit

Locked rate: $4/unit regardless of market
Commitment discount: -20% vs list
```

## Wholesale Account Management

### Customer Tiers
```
Gold ($1M+ annual): 
- 5% volume discount
- Net-60 terms
- Dedicated account manager
- Quarterly business reviews

Silver ($500K-$1M):
- 3% volume discount
- Net-30 terms
- Shared account manager

Bronze (< $500K):
- 2% volume discount
- Net-15 terms
- Self-service support
```

## API Patterns for B2B

```
GET /b2b/customers/{customer_id}/pricing
Response:
{
  "customer_id": "acme123",
  "pricing_model": "contract",
  "tier": "gold",
  "discount_percent": 15,
  "minimum_order_qty": 100,
  "payment_terms": "net-60",
  "products": [
    {
      "product_id": "widget-001",
      "unit_price": 85.00,
      "discount_applied": 15.00
    }
  ]
}

POST /b2b/orders
Request:
{
  "customer_id": "acme123",
  "po_number": "PO-2026-0001",
  "items": [
    { "product_id": "widget-001", "quantity": 500 }
  ]
}
Response:
{
  "order_id": "order_abc",
  "total": 42500,
  "requires_approval": true,
  "estimated_delivery": "2026-08-15"
}
```

## Metrics for B2B Success

| Metric | B2C | B2B | Target |
|--------|-----|-----|--------|
| **Conversion Rate** | 2-3% | 5-10% | Longer cycle, higher intent |
| **AOV (Avg Order Value)** | $50-100 | $5K-50K | Much larger orders |
| **LTV (Lifetime Value)** | $500-2000 | $50K-500K | Long-term relationships |
| **Repeat Purchase Rate** | 30-50% | 70-90% | Strong loyalty |
| **Customer Concentration** | Distributed | Concentrated | Top 10 customers = 40% |

---

**Pattern Version:** 1.0 | **Status:** Enterprise Pattern | **Market Size:** $8T global by 2027

