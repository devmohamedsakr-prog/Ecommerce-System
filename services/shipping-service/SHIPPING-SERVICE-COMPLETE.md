# Shipping Service - Complete Implementation Guide

**Scale:** 50M+ shipments annually | Multi-carrier integration | Global coverage

---

## API Specification

```
POST /v1/shipping/rates
  Get shipping quotes
  Body: { origin, destination, weight, dimensions, items[] }
  Response: [{ carrierId, method, cost, days, trackingCapable }]

POST /v1/shipping/labels
  Generate shipping label
  Body: { orderId, carrierCode, service, recipient, sender, parcels[] }
  Response: { labelId, trackingNumber, label_pdf_url, cost }

GET /v1/shipping/{trackingNumber}
  Track shipment
  Response: { status, location, lastUpdate, estimatedDelivery, events[] }

POST /v1/shipping/returns
  Create return label
  Body: { orderId, reason, itemIds[] }
  Response: { returnLabelId, trackingNumber, instructions }

GET /v1/shipping/carriers
  List available carriers by region
  Response: [{ carrierId, name, methods[], coverage_regions[] }]

PATCH /v1/shipping/{shipmentId}/cancel
  Cancel shipment
  Response: { success, refundAmount }
```

---

## Domain Models

```
Shipment {
  shipmentId: UUID
  orderId: UUID
  carrier: CarrierCode (USPS, UPS, FedEx, DHL, local)
  service: ShippingService (standard, express, overnight)
  origin: Address
  destination: Address
  weight: Decimal (kg)
  dimensions: Dimensions { length, width, height (cm) }
  cost: Money
  trackingNumber: String
  status: ShipmentStatus
  createdAt: DateTime
  estimatedDelivery: DateTime
  actualDelivery: DateTime
}

ShipmentStatus = CREATED | LABEL_GENERATED | PICKED_UP | IN_TRANSIT | DELIVERED | FAILED | RETURNED

CarrierCode = USPS | UPS | FedEx | DHL | Amazon_Logistics | Local

ShippingService = STANDARD | EXPRESS | OVERNIGHT | INTERNATIONAL
```

### Business Rules
1. **Rate shopping:** Get 3+ carrier quotes, pick cheapest/fastest
2. **Regional carriers:** Use local carriers where possible (cheaper)
3. **Consolidation:** Combine multiple items from same order
4. **Flat rate:** USPS flat-rate boxes (cheaper for small items)
5. **Tracking:** Provide tracking within 1 hour of label creation
6. **Returns:** Pre-paid return labels (customer satisfaction)

---

## Use Cases

### UC-001: Get Shipping Quotes
**Flow:**
1. Order ready for shipment
2. System calls shipping.rates()
3. Query all carriers (parallel API calls)
4. Get quotes: cost + delivery time
5. Return ranked by cost/speed
6. Order fulfillment chooses carrier

### UC-002: Generate Label & Ship
**Flow:**
1. Fulfillment scans package
2. System calls shipping.labels()
3. Carrier API generates label
4. Label printed at warehouse
5. Package scanned by carrier
6. Tracking number provided to customer

### UC-003: Track Shipment
**Flow:**
1. Customer clicks tracking link
2. System queries carrier API
3. Get latest status + location
4. Return to customer UI
5. Cache result (updated hourly)

### UC-004: Manage Returns
**Flow:**
1. Customer initiates return
2. System calls shipping.returns()
3. Generate pre-paid return label
4. Email label to customer
5. Customer drops at carrier location
6. Warehouse receives, processes refund

---

## Company Scenarios

### Amazon Logistics
```
Strategy: Own + carrier mix
- Amazon Logistics: 40% (own fleet, last-mile)
- UPS/USPS/FedEx: 60% (partner carriers)
- Negotiated rates: Volume discounts
- Regional optimization: Route to closest FC

Features:
- Same-day delivery (major cities)
- Next-day delivery (most US)
- Real-time tracking (every scan)
- Delivery photo confirmation

Consolidation:
- Combine multiple orders (same customer, same day)
- Reduce shipping volume 10-20%
```

### Lazada Southeast Asia
```
Strategy: Regional logistics partnerships
- LazLog (in-house): 30% of volume
- Local couriers: 70% (country-specific)

By country:
- Thailand: Kerry Express, NinjaVan
- Vietnam: Giao Hang Tiet Tron, J&T
- Philippines: LBC, 2GO
- Indonesia: JNE, Pos Indonesia

Cost per package:
- Urban (Bangkok, Jakarta): $1-2
- Regional: $2-4
- Islands: $5-10 (complex logistics)

Consolidation:
- Group by destination
- Hub-and-spoke model
```

### Shopify Fulfillment Network
```
Strategy: Per-merchant + SFN hybrid
- Merchant fulfillment: Direct from store (40%)
- Shopify FN: National network (60%)
- Integration: Automated inventory sync

Merchant control:
- Can use own carrier
- Can use SFN (cheaper via volume)
- Per-order selection

Rates:
- SFN: Negotiated bulk rates (lower than individual)
- Partner carriers: Pass-through rates
```

### Walmart Omnichannel
```
Strategy: Store + DC + regional optimization
- BOPIS: From 4500 stores (same-day)
- Ship-from-store: From nearby store (2-day)
- DC: From fulfillment center (3-5 days)

Carrier:
- Walmart Logistics (in-house)
- UPS/FedEx (partner)
- Regional couriers (last-mile)

Regional variation:
- Urban: 2-day standard
- Rural: 5-7 day standard
- Islands: Premium pricing
```

---

## Infrastructure

### Carrier Integration (Example: UPS)
```
API calls per day:
- 100K rate requests
- 50K label generations
- 5M tracking queries (cached)

Caching:
- Rates: Cache 1 hour (stable throughout day)
- Tracking: Cache 3 hours (refreshed hourly)

Error handling:
- Carrier API down: Use fallback carrier
- Rate request timeout: Show last known rates
- Label generation fail: Retry with alternate service
```

### Database Schema
```sql
CREATE TABLE shipments (
  shipment_id UUID PRIMARY KEY,
  order_id UUID,
  carrier VARCHAR(50),
  service VARCHAR(50),
  origin_address_id UUID,
  destination_address_id UUID,
  weight DECIMAL(5,2),
  cost DECIMAL(8,2),
  tracking_number VARCHAR(50),
  status VARCHAR(20),
  created_at TIMESTAMP,
  estimated_delivery TIMESTAMP,
  actual_delivery TIMESTAMP
);

CREATE TABLE return_labels (
  return_id UUID PRIMARY KEY,
  shipment_id UUID,
  tracking_number VARCHAR(50),
  reason VARCHAR(255),
  created_at TIMESTAMP
);

CREATE INDEX idx_order_id ON shipments(order_id);
CREATE INDEX idx_tracking ON shipments(tracking_number);
```

### Microservices Coordination
```
Shipping Service dependencies:
- Inventory: Stock location (choose warehouse)
- Order: Shipment details + address
- Fulfillment: Actual picked items (weight)

Events published:
- shipment.created → Notification Service
- shipment.label_generated → Fulfillment
- shipment.delivered → Analytics

Events consumed:
- order.confirmed → Request rates
- fulfillment.ready → Generate label
```

---

## Testing

### Unit Tests
- Calculate shipping cost (weight-based, flat-rate)
- Select carrier (cheapest, fastest)
- Validate address format
- Generate tracking number

### Integration Tests
- Full flow: Get rates → Generate label → Track
- Carrier fallback: Primary down → use secondary
- Return label: Generate, email, track
- Multi-item: Consolidate into single shipment

### Load Tests
- 100K rate requests/second
- 10K label generations/second
- 1M tracking queries/second (cached)
- <500ms latency p99 for all

---

## Monitoring

**Key Metrics:**
- Shipping cost per order ($ average)
- Carrier utilization (% by carrier)
- Delivery success rate (%)
- On-time delivery rate (%)
- Tracking accuracy (%)
- Carrier API availability (%)
- Label generation time (seconds)

---

