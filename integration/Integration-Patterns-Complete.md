# Integration Patterns for Enterprise E-Commerce

**Status:** Integration Guide | **Priority:** CRITICAL | **Patterns:** 6 Enterprise Integrations

---

## Overview

Enterprise ecommerce requires seamless integration with ERP, CRM, PIM, payment gateways, shipping providers, and tax engines. Each integration follows similar patterns: API connectivity, data synchronization, error handling, audit trails.

---

## Pattern 1: ERP (Enterprise Resource Planning) Integration

**Problem:** Order data in ecommerce, inventory/GL in ERP. Must sync in real-time.

**Architecture:**
```
E-Commerce System          ERP System (SAP/NetSuite/Oracle)
├─ Order Service    ────── API ────── Order Module
├─ Inventory        ────── API ────── Inventory Module
├─ Accounting       ────── API ────── GL Module
└─ Returns          ────── API ────── RMA Module
```

**Data Flow:**

1. **Order Creation to ERP**
```
E-Commerce: Customer places order
1. Order created in order-service
2. Trigger: Order.created event
3. Integration service listens
4. Extract order data:
   - Customer ID, PO number
   - Line items, quantities, prices
   - Shipping address, method
5. Transform to ERP format
6. POST /api/sales-orders (ERP API)
7. ERP confirms: Order created, assigned SO#
8. Store SO# in ecommerce order record
9. Continue fulfillment

Real-time: Order appears in ERP immediately
Audit: All sync events logged
Error handling: Retry if ERP unavailable, alert ops
```

2. **Inventory Synchronization (Bidirectional)**
```
Scenario A: Inventory changes in ERP
- Warehouse receives stock
- ERP updates inventory levels
- ERP publishes: inventory.updated
- Ecommerce listens
- Update product availability
- Decrement/clear reservations if stock available

Scenario B: Inventory reserved in Ecommerce
- Customer places order, inventory reserved
- Ecommerce publishes: inventory.reserved
- ERP receives notification
- ERP reserves inventory in warehouse system
- Prevents double-sell

Sync frequency: Real-time (< 1 second)
Consistency: Both systems agree on inventory
```

3. **Financial Data (Revenue Recognition - ASC 606)**
```
Order placed: $1000
ERP records:
- AR (Accounts Receivable): +$1000
- Deferred Revenue: +$1000

Order shipped: 50%
- Recognize 50% revenue: $500
- Deferred Revenue: -$500
- Revenue: +$500

Order fulfilled: 100%
- Recognize remaining: $500
- Revenue: +$500

Ecommerce sends:
- Fulfillment status to ERP
- ERP calculates revenue recognition
- Financial statements accurate
```

**APIs:**
```
POST /api/sales-orders (create)
GET /api/sales-orders/{id} (retrieve)
GET /api/inventory/{sku} (check levels)
POST /api/inventory-adjustments (update stock)
GET /api/gl-accounts (chart of accounts)
POST /api/journal-entries (record transactions)
```

**Error Handling:**
```
If ERP unavailable:
- Queue orders in Redis
- Retry with exponential backoff
- Alert ops team
- Continue accepting orders
- Manual sync when ERP online

If data mismatch:
- Inventory discrepancy detected
- Log error with details
- Notify reconciliation team
- Block inventory operations until resolved
```

---

## Pattern 2: CRM (Customer Relationship Management) Integration

**Problem:** Customer data fragmented (ecommerce, CRM, support). Single source needed.

**Architecture:**
```
E-Commerce        CRM System (Salesforce/HubSpot)
├─ Customers ──── API ──── Accounts
├─ Orders    ──── API ──── Opportunities
└─ Support   ──── API ──── Cases
```

**Data Sync:**

1. **Customer Data Synchronization**
```
New customer signs up on ecommerce:
1. Customer created in ecommerce
2. Event: customer.created
3. Integration service:
   - Transform data
   - POST /crm/accounts (create account)
   - CRM returns: account_id
   - Store account_id in ecommerce
   - Bi-directional link established

Subsequent purchases:
- Link to CRM account
- CRM sees full history
- Sales team has context
```

2. **Order History**
```
Ecommerce publishes orders to CRM:
- Order ID, amount, date
- Products purchased
- Revenue by product
- Customer lifetime value

CRM benefits:
- Sales team sees purchase history
- Identifies upsell opportunities
- Tracks customer lifecycle
- Forecasting accuracy improves
```

3. **Sales Pipeline Integration**
```
CRM Opportunity → Ecommerce Discount

Sales rep creates opportunity:
- Company: Acme Corp
- Value: $50K
- Stage: Proposal

Ecommerce integration:
- Read opportunity
- Generate discount code: ACME50K
- Send code to contact
- Link to opportunity
- When code used: Mark opportunity won
- When expired: Mark opportunity lost
```

**APIs:**
```
POST /crm/accounts (create customer)
PUT /crm/accounts/{id} (update)
GET /crm/accounts/{id}
POST /crm/opportunities (create from orders)
GET /crm/contacts
POST /crm/activities (log interactions)
```

**Benefits:**
```
- 360-degree customer view
- Sales team informed
- Support team context
- Marketing personalization
- Churn prediction
```

---

## Pattern 3: PIM (Product Information Management) Integration

**Problem:** Product data in ecommerce, images/specs in PIM, search in search engine. Multiple sources of truth.

**Architecture:**
```
PIM System (Contentful/Salsify)
├─ Product Master Data
├─ Digital Assets (images, videos)
├─ Enriched Content (SEO, descriptions)
└─ Translations
         ↓
E-Commerce (pulls via API)
├─ Display to customers
├─ Catalog search
├─ Recommendations
```

**Data Flow:**

1. **Product Data Synchronization**
```
PIM is source of truth:
- Product name, description, specs
- Images, videos, PDFs
- Pricing rules (can vary by channel)
- Multilingual content
- Categories, tags, attributes

Ecommerce pulls on schedule:
- Frequency: Every 4 hours or on change
- API: GET /pim/products?updated_after=2026-08-05T10:00:00Z
- Transform: PIM format → ecommerce format
- Store in ecommerce database
- Invalidate caches
- Index in search

Benefits:
- Single product data source
- Consistent across channels
- Asset management centralized
```

2. **Catalog Expansion (Channel-Specific)**
```
PIM catalogs products for:
- B2C Web Store: 5000 products
- B2B Wholesale: 2000 products (subset)
- Marketplace (Amazon): 3000 products
- International: Regional subsets

Ecommerce uses:
- Only products for its channel
- Channel-specific pricing
- Channel-specific availability
- Channel-specific assets

Flexibility: Add new channel without product migration
```

3. **Multilingual Content**
```
PIM contains:
- English: "Red Widget"
- Spanish: "Widget Rojo"
- German: "Rotes Widget"
- Japanese: "赤いウィジェット"

Ecommerce:
- User in Spain: Shows Spanish content
- User in Japan: Shows Japanese content
- Content consistency: Single source
```

**APIs:**
```
GET /pim/products
GET /pim/products/{id}
GET /pim/assets
GET /pim/categories
GET /pim/attributes
GET /pim/translations/{product_id}/{language}
```

**Sync Strategy:**
```
Schedule 1: Full sync every 24 hours (reconciliation)
Schedule 2: Incremental sync every 4 hours (changes)
Event-based: Immediate sync on critical changes

Monitoring:
- Sync success rate
- Data discrepancies
- Missing fields
- Image availability
```

---

## Pattern 4: Payment Gateway Integration

**Problem:** Process payments via multiple gateways (Stripe, PayPal, local methods). Consistency and PCI compliance required.

**Architecture:**
```
Ecommerce Payment Service
├─ Tokenization (no card data stored)
├─ Multi-gateway routing
├─ Fraud detection
└─ PCI DSS compliance
         ↓
Payment Gateways
├─ Stripe (global)
├─ PayPal
├─ Alipay (China)
├─ WeChat Pay (China)
└─ Local methods per region
```

**Payment Flow:**

1. **Tokenized Payment (Secure)**
```
Customer enters card details:
1. Client-side encryption (Stripe.js)
2. Token created (never see card number)
3. Token sent to ecommerce
4. Ecommerce never handles card data
5. Store token in secure vault
6. Use token for future charges

Benefits:
- PCI DSS Level 1 compliance
- No card data exposure
- Automatic compliance updates
- Recurring billing support
```

2. **Payment Processing**
```
Cart total: $100

Payment selection:
- If USA: Route to Stripe
- If China: Route to Alipay
- If PayPal selected: Route to PayPal

Process:
1. POST /payment/charge
   {
     "amount": 10000,
     "currency": "USD",
     "token": "stripe_token_xyz",
     "gateway": "stripe"
   }
2. Gateway processes
3. Response:
   {
     "status": "succeeded",
     "transaction_id": "txn_123",
     "receipt_url": "..."
   }
4. Order confirmed
5. Fulfillment begins
```

3. **Recurring Billing**
```
Subscription charge $29.99/month:
1. Create subscription
   - Token stored
   - Amount: $29.99
   - Frequency: Monthly
   - Start: 2026-08-05
2. Gateway charges automatically
3. On success: Activate subscription
4. On failure: Retry (exponential backoff)
5. After 3 failed attempts:
   - Email customer
   - Hold subscription
   - Alert support team
6. Customer updates payment method
7. Resume subscription

Dunning (retry logic):
- Day 1: Charge fails
- Day 3: Retry #1
- Day 6: Retry #2
- Day 9: Retry #3
- Day 12: Suspend if all fail
```

4. **Multi-Currency Handling**
```
Customer in Germany:
1. Cart total: €92
2. Apply tax: €17.48
3. Final: €109.48
4. Send to Stripe:
   {
     "amount": 10948,  // cents
     "currency": "eur",
     "token": "..."
   }
5. Stripe processes in EUR
6. Receipt in EUR

Customer in Japan:
1. Cart total: ¥12,000
2. Final: ¥12,000 (no sales tax in Japan)
3. Send to appropriate gateway
4. Process in JPY

Currency conversion:
- Real-time rates from payment gateway
- Applied at checkout
- Locked in at payment

Multi-currency benefit:
- Lower decline rates (familiar currency)
- Better UX
- Regulatory compliance
```

**APIs:**
```
POST /payment/tokenize
POST /payment/charge
POST /payment/refund
GET /payment/transaction/{id}
POST /payment/subscriptions (recurring)
POST /payment/webhooks (receive updates)
```

**Compliance:**
```
PCI DSS Requirements:
1. No card data stored ✓ (tokenization)
2. Secure transmission ✓ (HTTPS)
3. Access controls ✓ (Token vault)
4. Audit trails ✓ (All payments logged)
5. Regular security testing ✓ (Annual)

Result: PCI Level 1 compliant
```

---

## Pattern 5: Shipping & Fulfillment Integration

**Problem:** Multiple carriers (FedEx, UPS, DHL), warehouse systems, tracking. Must coordinate.

**Architecture:**
```
Order Service
     ↓
Fulfillment Service
├─ Inventory allocation
├─ Picking/packing
├─ Carrier selection
     ↓
Carrier APIs (FedEx/UPS/DHL)
├─ Rate shopping
├─ Label generation
├─ Tracking
     ↓
WMS (Warehouse Management)
├─ Inventory location
├─ Pick list
├─ Verification
```

**Shipping Flow:**

1. **Rate Shopping**
```
Order placed: Ship to California
Need to ship: 2 units (5 lbs)
Carrier options:

Request rates:
POST /carriers/rates
{
  "origin": warehouse_address,
  "destination": "California",
  "weight": 5,
  "dimensions": [12, 10, 4]
}

Responses:
- FedEx Ground: $12.50 (3 days)
- FedEx Express: $35.00 (1 day)
- UPS Ground: $11.80 (2 days)
- DHL: $10.50 (4 days)

Selection logic:
- Preferred: Cheapest within SLA
- If 1-day needed: Express
- If 2-day OK: Ground (cheapest)

Add to order: $11.80 (UPS Ground)
```

2. **Label Generation & Tracking**
```
Order ready to ship:
POST /carriers/create-shipment
{
  "carrier": "ups",
  "order_id": "order_123",
  "items": [product_1, product_2],
  "weight": 5.2,
  "destination": address
}

Response:
{
  "tracking_number": "1Z999AA10123456784",
  "label_url": "https://...",
  "estimated_delivery": "2026-08-08"
}

Actions:
- Print label
- Attach to package
- Scan barcode to confirm
- Package leaves warehouse
- Tracking activated
```

3. **Tracking Updates**
```
Carrier provides webhooks:

1. Carrier: Package picked up
   Webhook: shipment.picked_up
   → Ecommerce updates order: "In Transit"
   → Customer notified: "Your order is on the way!"

2. Carrier: Out for delivery
   Webhook: shipment.out_for_delivery
   → Ecommerce updates order: "Out for Delivery"
   → Customer notified: "Your package will arrive today"

3. Carrier: Delivered
   Webhook: shipment.delivered
   → Ecommerce updates order: "Delivered"
   → Customer notified: "Your package has arrived"
   → Trigger: Review request (after 3 days)

Real-time tracking: Customers see updates without manual refresh
```

4. **Multi-Warehouse Fulfillment**
```
Inventory distributed:
- Warehouse A (East): 100 units
- Warehouse B (West): 80 units

Order placed in California:
1. Calculate closest warehouse: Warehouse B (West)
2. Reserve 1 unit from Warehouse B
3. Pick/pack from Warehouse B
4. Ship from Warehouse B
5. Cost: UPS Ground $5 (vs $12 from East)

Benefit: Cheaper shipping, faster delivery
```

**APIs:**
```
GET /carriers/rates
POST /carriers/create-shipment
GET /carriers/tracking/{tracking_number}
POST /carriers/webhooks (receive updates)
GET /wms/inventory/{warehouse_id}
POST /wms/pick-list
```

---

## Pattern 6: Tax Engine Integration

**Problem:** Tax rates vary by jurisdiction (50 US states, 197 countries, EU VAT). Manual calculation error-prone.

**Architecture:**
```
Order Service
     ↓
Tax Service (Avalara/TaxJar)
├─ Nexus determination
├─ Rate lookup
├─ Tax calculation
     ↓
ERP (For reporting & remittance)
```

**Tax Calculation Flow:**

1. **Real-Time Rate Lookup**
```
Customer in New York, buying $100:
1. POST /tax/calculate
   {
     "customer_address": "New York, NY",
     "line_items": [
       {
         "product_id": "widget-001",
         "price": 100,
         "category": "physical_good"
       }
     ]
   }

2. Tax service determines:
   - NY State: 4%
   - NYC Local: 4.5%
   - Total: 8.5%
   - Tax due: $8.50

3. Response:
   {
     "tax_rate": 0.085,
     "tax_amount": 8.50,
     "total": 108.50,
     "nexus": "NY",
     "breakdown": {
       "state": 4.00,
       "local": 4.50
     }
   }

4. Display to customer: $108.50 total
```

2. **International Tax (VAT)**
```
Customer in Germany (EU):
1. POST /tax/calculate
   {
     "customer_address": "Berlin, Germany",
     "line_items": [...]
   }

2. Tax service:
   - VAT Rate: 19% (Germany standard)
   - Tax amount: $19

3. Total: $119

4. Additional: Compliance data
   - Intra-EU: No VAT (B2B)
   - EU to non-EU: VAT applies
   - Non-EU to EU: Import duty applies
```

3. **Tax Remittance & Reporting**
```
Monthly tax report:
- Total sales: $100,000
- Total tax collected: $8,500
- By jurisdiction:
  - NY: $4,000
  - CA: $2,500
  - TX: $1,500
  - Other: $500

File returns:
- NY: File sales tax return
- CA: File sales tax return
- TX: File sales tax return

Remit taxes:
- NY: $4,000 due by 20th
- CA: $2,500 due by 15th
- TX: $1,500 due by 25th

Tax service integration:
- Calculate amounts
- Track by jurisdiction
- Generate reports
- Support compliance

ERP integration:
- Record tax liability
- Generate GL entries
- Balance sheet accurate
```

4. **Product-Category Tax Rules**
```
Different products taxed differently:

Food:
- Most food items: No tax
- Prepared food: Taxed
- Candy: May be taxed
- Dietary supplements: May be tax-exempt

Digital:
- E-books: May be tax-exempt
- Software: Varies by state
- Digital services: Different rules

Clothing:
- Regular clothing: Varies
- Some states: No tax
- Other states: Full tax
- Footwear: May differ

Tax service knows rules per jurisdiction
Ecommerce categorizes products correctly
Tax calculated automatically
```

**APIs:**
```
POST /tax/calculate
POST /tax/commit (finalize for reporting)
GET /tax/reporting/{period}
POST /tax/remittance
```

---

## Integration Monitoring & Error Handling

**Common Issues:**

```
1. API Rate Limiting
   - Implement backoff
   - Queue requests
   - Cache responses

2. Data Mismatch
   - Compare checksums
   - Alert on discrepancies
   - Manual reconciliation

3. Latency
   - Async where possible
   - Queue for later processing
   - Don't block checkout

4. Partial Failures
   - Retry logic
   - Dead-letter queues
   - Alert ops team

5. API Changes
   - Version compatibility
   - Monitor deprecation warnings
   - Test new versions in staging
```

**Monitoring Dashboard:**
```
Track per integration:
- Success rate (target: 99%+)
- Latency (target: < 1 second)
- Error rate (target: < 0.1%)
- Sync freshness (target: < 5 minutes)

Alert on:
- Success rate < 95%
- Latency > 5 seconds
- Service unavailable
- Data mismatch
```

---

**Guide Version:** 1.0 | **Status:** Production Integration Patterns | **Scope:** Enterprise-ready

