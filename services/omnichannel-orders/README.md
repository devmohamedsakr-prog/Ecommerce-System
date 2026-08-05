# Omnichannel Order Management Service

**Status:** Enterprise Service | **Priority:** HIGH | **Compliance:** GDPR

---

## 📋 Overview

Omnichannel Service unifies orders from multiple sales channels (website, marketplaces, retail, social commerce), manages inventory across channels, and routes orders to optimal fulfillment locations. Provides single source of truth for operations across channels.

## 🎯 Business Problem

- 50%+ of sales cross-channel or marketplace
- Inventory fragmentation wastes 20-30% in excess stock
- No unified customer view across channels
- Fulfillment inefficiencies from siloed operations
- Manual channel management error-prone

## 🏗️ Architecture

```
┌────────────────────────────────────────┐
│  Omnichannel Order Management          │
├────────────────────────────────────────┤
│                                         │
│  Website │ Marketplace │ Retail│Social │
│    ↓         ↓            ↓       ↓    │
│  ┌──────────────────────────────┐    │
│  │ Order Aggregation & Routing  │    │
│  └──────────────────────────────┘    │
│           ↓                            │
│  ┌──────────────────────────────┐    │
│  │ Unified Inventory View       │    │
│  └──────────────────────────────┘    │
│           ↓                            │
│  ┌──────────────────────────────┐    │
│  │ Optimal Fulfillment Routing  │    │
│  └──────────────────────────────┘    │
│                                        │
└────────────────────────────────────────┘
```

### Data Model

```
CHANNEL
├── channel_id (UUID)
├── channel_name (string: Amazon, eBay, Shopify, Instagram, retail-store)
├── channel_type (enum: marketplace, direct-ecom, social, retail, b2b)
├── status (enum: active, inactive)
├── api_credential (encrypted)
├── sync_frequency (enum: real-time, hourly, daily)
└── enabled (boolean)

CHANNEL_ORDER
├── channel_order_id (string)
├── channel_id (FK)
├── unified_order_id (FK)
├── order_date (timestamp)
├── status (enum: new, processing, shipped, delivered)
├── items (array)
├── total (decimal)
└── fulfillment_location (warehouse_id, nullable)

UNIFIED_ORDER
├── unified_order_id (UUID)
├── customer_id (FK)
├── channel_orders (array: channel_order_id)
├── fulfillment_location (warehouse_id)
├── fulfillment_method (enum: drop-ship, warehouse, retail-pickup)
├── status (enum: pending, allocated, shipped, delivered)
├── created_date (timestamp)
└── ship_date (timestamp, nullable)

CHANNEL_INVENTORY
├── inventory_id (UUID)
├── product_id (FK)
├── channel_id (FK)
├── warehouse_id (FK)
├── quantity_available (number)
├── quantity_allocated (number)
├── last_sync (timestamp)
└── sync_status (enum: in-sync, out-of-sync, pending-sync)

UNIFIED_INVENTORY
├── unified_inventory_id (UUID)
├── product_id (FK)
├── total_available (number: sum across all locations)
├── total_allocated (number)
├── warehouse_distribution (object: warehouse_id → quantity)
├── last_updated (timestamp)
└── sync_status (enum: in-sync, syncing)

FULFILLMENT_ROUTING
├── routing_id (UUID)
├── order_id (FK)
├── algorithm (enum: closest-warehouse, cheapest-shipping, fastest-delivery)
├── selected_warehouse (warehouse_id)
├── estimated_delivery_date (timestamp)
├── shipping_cost (decimal)
└── optimization_score (0-100)

CHANNEL_SYNC_LOG
├── sync_id (UUID)
├── channel_id (FK)
├── sync_type (enum: order-sync, inventory-sync, full-sync)
├── sync_date (timestamp)
├── status (enum: success, partial, failed)
├── records_synced (number)
├── errors (array, nullable)
└── next_sync_date (timestamp)
```

## 📡 Core APIs

```
POST /v1/omnichannel/orders/sync
├── Sync orders from all channels
├── Request: channel_id (optional, for single channel)
└── Response: orders_synced, sync_status

GET /v1/omnichannel/orders/{unified_order_id}
├── Get unified order view across all channels
└── Response: order_record, all_channel_orders, fulfillment_status

GET /v1/omnichannel/inventory
├── Get unified inventory view
└── Response: product_availability, warehouse_distribution, total_available

POST /v1/omnichannel/route-order
├── Route order to optimal fulfillment location
├── Request: order_id, optimization_criterion (closest, cheapest, fastest)
└── Response: fulfillment_location, shipping_cost, estimated_delivery

GET /v1/omnichannel/channel-inventory/{channel_id}
├── Get inventory for specific channel
└── Response: inventory_sync_status, last_sync_date

POST /v1/omnichannel/reallocate-inventory
├── Reallocate inventory across channels/warehouses
├── Request: product_id, new_allocation
└── Response: allocation_updated, sync_initiated
```

## 🔄 Workflows

### Order Aggregation
```
1. Customer places order on Amazon
2. Order synced to unified system (real-time or batch)
3. Unified order created
4. Linked to customer record
5. Fulfillment routing decision made
6. Order sent to selected warehouse
```

### Inventory Synchronization
```
1. Real-time event: item sold on eBay
2. Inventory decremented immediately
3. Unified inventory updated
4. Other channels' inventory levels updated
5. Prevent oversell across channels
```

### Fulfillment Routing
```
1. Order received: 50-pack of item
2. Warehouse 1: 30 units (2-day delivery, $5 shipping)
3. Warehouse 2: 25 units (5-day delivery, $2 shipping)
4. Algorithm: closest warehouse = Warehouse 1
5. Route order to Warehouse 1
6. Optimized cost + speed
```

## 📊 Key Metrics

| Metric | Target | Impact |
|--------|--------|--------|
| **Inventory Fragmentation** | < 10% waste | 20-30% savings |
| **Fulfillment Speed** | 30-40% faster | Unified routing |
| **Order Sync Latency** | < 1 hour | Real-time data |
| **Omnichannel Penetration** | 50%+ of orders | Scalability |

## 🔗 Integration Points

- **Order Service** - Order management
- **Inventory Service** - Stock levels
- **Shipping Service** - Carrier routing
- **Analytics Service** - Channel performance

---

**Service Version:** 1.0 | **Status:** Enterprise High Priority | **Compliance:** GDPR

