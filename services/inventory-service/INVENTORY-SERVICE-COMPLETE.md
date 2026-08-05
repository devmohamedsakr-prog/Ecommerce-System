# Inventory Service - Complete Implementation Guide

**Scale:** 1B+ SKUs | Real-time synchronization | 99.95% accuracy

---

## API Specification

```
POST /v1/inventory/reserve
  Reserve stock (before payment)
  Body: { productId, quantity, orderId, timeoutMinutes }
  Response: { reservationId, expiresAt }
  Idempotent: same orderId = same result

POST /v1/inventory/confirm
  Confirm reservation (after payment)
  Body: { reservationId }
  Response: { success }

POST /v1/inventory/release
  Release reservation (order cancelled)
  Body: { reservationId }
  Response: { success }

GET /v1/inventory/{productId}
  Check stock level
  Response: { productId, quantity, reserved, available, warehouseLocations[] }

PATCH /v1/inventory/{productId}
  Update stock (manual adjustment)
  Body: { delta, reason, reference }
  Response: { newQuantity }

GET /v1/inventory/events
  Stream inventory changes (real-time)
  Response: Event stream (stock updated, reserved, confirmed)
```

---

## Domain Models

```
Inventory {
  productId: UUID
  totalQuantity: Integer
  reservedQuantity: Integer
  availableQuantity: Integer (calculated: total - reserved)
  warehouseLocations: WarehouseLocation[]
  lastUpdated: DateTime
  version: Integer (for optimistic locking)
}

WarehouseLocation {
  warehouseId: UUID
  quantity: Integer
  reservedQuantity: Integer
  locationCode: String (e.g., "A-23-45")
}

Reservation {
  reservationId: UUID
  productId: UUID
  orderId: UUID
  quantity: Integer
  createdAt: DateTime
  expiresAt: DateTime (5 minutes typical)
  status: ReservationStatus (PENDING, CONFIRMED, EXPIRED, RELEASED)
}

ReservationStatus = PENDING | CONFIRMED | EXPIRED | RELEASED
```

### Business Rules
1. **Reserve first:** Stock reserved before payment processes
2. **Expiration:** Reservation expires if not confirmed (5 min default)
3. **Oversell Prevention:** reserved + available >= 0 (no negative)
4. **Warehouse allocation:** Pick closest warehouse to customer
5. **Reorder automation:** When stock < min_level, auto-trigger reorder
6. **Real-time sync:** Update across all services within 1 second

---

## Use Cases

### UC-001: Reserve Stock Before Payment
**Flow:**
1. Customer proceeds to checkout
2. System calls inventory.reserve()
3. Stock locked for 5 minutes
4. Customer completes payment
5. If payment succeeds: confirm reservation
6. If fails/timeout: auto-release

### UC-002: Confirm After Payment
**Flow:**
1. Payment successful
2. System calls inventory.confirm()
3. Reserved → committed stock
4. Fulfillment notified

### UC-003: Oversell Prevention
**Rule:**
- Available = Total - Reserved - Damage
- Never allow: Available < 0
- On order: Reduce immediately (don't wait for fulfillment)

### UC-004: Real-time Stock Sync
**Trigger:** Stock level changes  
**Flow:**
1. Any service updates inventory
2. Event published to Kafka topic
3. All consumers updated (cart, search, warehouse)
4. Cache invalidated
5. <1 second propagation

---

## Company Scenarios

### Amazon Inventory
```
Strategy: Distributed per region + central
- 200+ FCs, each maintains stock
- Central DynamoDB (source of truth)
- Real-time sync via SQS
- Oversell allowed (1-2%, acceptable loss)
- Auto-reorder based on ML forecast

Reservation:
- 15 minutes timeout (vs 5 min standard)
- Multiple items: all-or-nothing
- Priority: Prime orders prioritized
```

### Alibaba Taobao Inventory
```
Strategy: Seller-managed with platform verification
- Seller updates stock directly
- Platform audit: Spot check 1% of sellers
- Fraud detection: Sudden drops flagged
- Flash sale: Pre-allocate stock
- Reservation: 3 minutes (aggressive)

Real-time sync:
- Event-driven (Kafka)
- Sub-second latency
- Consistency acceptable (eventual)
```

### Walmart Omnichannel
```
Strategy: Unified store + warehouse
- Store inventory real-time
- Warehouse inventory real-time
- Ship-from-store: Pull from stores first
- BOPIS: Reserve at nearest store
- Allocation algorithm: Minimize shipping cost

Allocation priority:
1. Nearest store (BOPIS)
2. Nearest FC (2-day shipping)
3. Regional hub (3+ days)
```

### Shopify Multi-tenant
```
Strategy: Per-merchant inventory
- Merchant controls stock
- Platform provides tools
- Integration: Shopify Fulfillment Network (SFN)
- Channels: Shopify + Amazon + eBay sync
- Reservation: 10 minutes (startup friendly)
```

---

## Infrastructure

### Database Schema
```sql
CREATE TABLE inventory (
  product_id UUID PRIMARY KEY,
  total_quantity INTEGER,
  reserved_quantity INTEGER,
  last_updated TIMESTAMP,
  version INTEGER
);

CREATE TABLE warehouse_locations (
  location_id UUID PRIMARY KEY,
  product_id UUID REFERENCES inventory,
  warehouse_id UUID,
  quantity INTEGER,
  location_code VARCHAR(50)
);

CREATE TABLE reservations (
  reservation_id UUID PRIMARY KEY,
  product_id UUID REFERENCES inventory,
  order_id UUID,
  quantity INTEGER,
  created_at TIMESTAMP,
  expires_at TIMESTAMP,
  status VARCHAR(20)
);

CREATE INDEX idx_order_id ON reservations(order_id);
CREATE INDEX idx_expires_at ON reservations(expires_at);
```

### Sharding Strategy
```
Shard by product_id (1000+ shards)
- Shard key: hash(product_id) % 1000
- Each shard: Separate database (PostgreSQL)
- Replication: 3 replicas per shard (HA)
- Hot products: Dedicated cache (Redis)

Scaling:
- Add shard: Re-shard when single shard >50GB
- Vertical: Increase replica capacity
- Horizontal: Add more products → more shards
```

### Kafka Topic
```
Topic: inventory-events
Partitions: 100 (by product_id)
Retention: 7 days

Messages:
{
  "eventType": "reserved|confirmed|released|updated",
  "productId": UUID,
  "quantity": Integer,
  "timestamp": DateTime,
  "source": "order-service|warehouse-service"
}
```

### Caching (Redis)
```
Pattern:
  inventory:{productId} → JSON (ttl: 1 minute)
  inventory:hot → Set of productIds (most accessed)

Invalidation:
  - Any inventory change → invalidate cache
  - Auto-expiry: 1 minute (cheap to rebuild)
  - Hot products: 5 minute TTL (higher value)
```

---

## Testing

### Unit Tests
- Reserve when stock available → success
- Reserve when stock low → partial reject
- Confirm expired reservation → error
- Release released reservation → idempotent
- Calculate available = total - reserved

### Integration Tests
- Full flow: Reserve → Confirm → Fulfillment notified
- Expiration: Unreserved items after timeout
- Oversell prevention: Never go negative
- Real-time sync: Kafka event published and consumed

### Load Tests
- 1M concurrent reserves/second
- <100ms latency p99
- Zero lost updates (compare-and-set)
- Consistent across 1000+ shards

---

## Monitoring

**Key Metrics:**
- Stock level per product
- Reservation rate (reserved/total)
- Reservation timeout rate (%)
- Real-time sync latency (seconds)
- Oversell rate (%)
- Cache hit rate (%)

---

