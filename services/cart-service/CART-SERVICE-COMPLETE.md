# Cart Service - Complete Implementation Guide

**Scale:** 500M+ active carts | 10M+ concurrent users | <100ms latency required

---

## API Specification

### Core Endpoints

```
POST /v1/carts
  Create new cart
  Body: { userId, currency, countryCode }
  Response: { cartId, createdAt, status: "ACTIVE" }

POST /v1/carts/{cartId}/items
  Add item to cart
  Body: { productId, quantity, price, variantId }
  Response: { itemId, cartTotal, itemCount }
  Errors: { 409: "Item already exists", 400: "Invalid quantity" }

DELETE /v1/carts/{cartId}/items/{itemId}
  Remove item from cart
  Response: { success, cartTotal, itemCount }

PATCH /v1/carts/{cartId}/items/{itemId}
  Update item quantity
  Body: { quantity }
  Response: { itemId, quantity, cartTotal }

GET /v1/carts/{cartId}
  Get cart details
  Response: { cartId, userId, items[], subtotal, tax, shipping, total, state }

POST /v1/carts/{cartId}/checkout
  Proceed to checkout
  Body: { shippingAddress, billingAddress, shippingMethod }
  Response: { checkoutId, redirectUrl }

POST /v1/carts/{cartId}/abandon
  Mark cart abandoned (for recovery)
  Response: { success }

GET /v1/carts/abandoned
  List abandoned carts (for analytics)
  Response: [{ cartId, userId, lastActivity, value, itemCount }]
```

### Error Codes

- 400: Invalid input
- 404: Cart not found
- 409: Conflict (item exists, out of stock)
- 429: Rate limited
- 500: Internal error

---

## Domain Models

### Cart Aggregate
```
Cart {
  cartId: UUID
  userId: UUID (nullable for guest)
  items: CartItem[] (0-1000)
  subtotal: Money
  tax: Money
  shipping: Money
  total: Money
  state: CartState (ACTIVE, MERGED, ABANDONED, CHECKED_OUT)
  createdAt: DateTime
  updatedAt: DateTime
  expiresAt: DateTime (30 days)
  couponCode: string (optional)
  shippingMethod: ShippingMethod (optional)
}

CartItem {
  itemId: UUID
  productId: UUID
  variantId: UUID (optional)
  quantity: Integer (1-999)
  price: Money
  discount: Money (optional)
  total: Money (calculated)
}

CartState = ACTIVE | MERGED | ABANDONED | CHECKED_OUT

ShippingMethod {
  methodId: string
  name: string (e.g., "Standard 5-7 days", "Express 2-3 days")
  cost: Money
  estimatedDays: Integer
}
```

### Business Rules
1. **Item Merging:** Same productId + variantId = merged (quantity added)
2. **Quantity Limits:** 1-999 per item
3. **Expiration:** Cart expires 30 days without activity
4. **Checkout Conversion:** Cart → CheckedOut after payment
5. **Guest to Registered:** Merge guest cart with registered cart on login
6. **Out of Stock:** Remove item or reduce quantity automatically
7. **Tax Calculation:** Based on shipping address + items

---

## Use Cases

### UC-001: Add Item to Cart
**Actor:** Customer  
**Flow:**
1. Customer browses product
2. Customer clicks "Add to Cart"
3. System checks stock (inventory service)
4. System adds to cart or increases quantity
5. Cart total recalculated
6. Cart updated in cache (Redis)
7. Confirmation message shown

**Scenarios:**
- Happy path: Item added, user sees "+1"
- Item exists: Quantity increased
- Out of stock: Error message shown
- Cart expired: New cart created

### UC-002: Merge Carts (Guest to Registered)
**Actor:** System  
**Trigger:** User logs in with active cart
**Flow:**
1. Detect guest cart + registered cart
2. Merge items (combining quantities)
3. Use registered cart as primary
4. Mark guest cart as MERGED
5. Keep order history intact

### UC-003: Apply Coupon
**Actor:** Customer  
**Flow:**
1. Customer enters coupon code
2. Validate coupon (active, not expired, usage limits)
3. Check eligibility (min purchase, category restrictions)
4. Apply discount to cart
5. Recalculate total
6. Store coupon in cart

### UC-004: Abandon & Recovery
**Actor:** Customer / System  
**Flow:**
1. Cart inactive 3+ days
2. System marks ABANDONED
3. Send email with recovery link
4. If link clicked within 7 days: reactivate cart
5. If not clicked: mark for analytics

### UC-005: Checkout
**Actor:** Customer  
**Flow:**
1. Customer clicks Checkout
2. System creates checkout session
3. Validate items (still in stock)
4. Calculate shipping
5. Redirect to payment
6. On success: convert cart to CHECKED_OUT

---

## Company Scenarios

### Amazon Cart Implementation
```
Features:
- 1-click add to cart (saved payment methods)
- Frequently bought together (recommendations)
- Save for later (secondary list)
- Cart persistence across devices
- Abandoned cart recovery (email 1h, 6h, 24h)
- Subscribe & Save (recurring)

Scale:
- Peak: 50M+ concurrent carts
- Data: DynamoDB with global tables
- Cache: ElastiCache (99% hit rate)
- Latency: <50ms p99
```

### Alibaba Taobao Cart Implementation
```
Features:
- Group buying (combine orders from multiple sellers)
- Shared carts (copy to friend)
- Coupon management (platform + seller coupons)
- Flash sale integration (limited stock, countdown)
- Pre-order items (mixed with normal items)

Scale:
- 100M+ concurrent carts
- Alipay integration (instant payment)
- Real-time stock sync
```

### Shopify Cart Implementation
```
Features:
- Headless cart API (decoupled frontend)
- Persistent carts (no session loss)
- Bundled items (combo products)
- Subscription items (recurring + one-time)
- Custom attributes (gift message, engraving)

Scale:
- Multi-tenant architecture
- Per-merchant sharding
- <200ms latency SLA
```

### Startup (MVP) Implementation
```
Features:
- Basic add/remove items
- Persistent carts (cookies for guest, DB for registered)
- Simple checkout (email required)
- Minimal validation

Tech:
- Redis for caching
- PostgreSQL for persistence
- REST API only
- <500ms latency acceptable
```

---

## Infrastructure

### Docker
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/
EXPOSE 3000
CMD ["node", "src/index.js"]
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart-service
spec:
  replicas: 10
  selector:
    matchLabels:
      app: cart-service
  template:
    metadata:
      labels:
        app: cart-service
    spec:
      containers:
      - name: cart-service
        image: cart-service:latest
        ports:
        - containerPort: 3000
        env:
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: cart-config
              key: redis-url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
```

### Database Schema
```sql
CREATE TABLE carts (
  cart_id UUID PRIMARY KEY,
  user_id UUID,
  state VARCHAR(20),
  subtotal DECIMAL(10,2),
  tax DECIMAL(10,2),
  shipping DECIMAL(10,2),
  total DECIMAL(10,2),
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  expires_at TIMESTAMP
);

CREATE TABLE cart_items (
  item_id UUID PRIMARY KEY,
  cart_id UUID REFERENCES carts,
  product_id UUID,
  variant_id UUID,
  quantity INTEGER,
  price DECIMAL(10,2),
  discount DECIMAL(10,2),
  created_at TIMESTAMP
);

CREATE INDEX idx_user_cart ON carts(user_id);
CREATE INDEX idx_state ON carts(state);
CREATE INDEX idx_expires_at ON carts(expires_at);
```

### Caching Strategy
```
Redis Structure:
  cart:{cartId} → JSON (ttl: 24 hours)
  user:{userId}:cart → cartId (ttl: 24 hours)

Cache invalidation:
  - Add/remove item → invalidate cart:{cartId}
  - Merge carts → invalidate both carts
  - Checkout → delete cache
```

### Scaling Notes
- Horizontal: Add cart-service replicas (stateless)
- Vertical: Increase Redis cluster size
- Database: Shard by user_id (1000+ shards for 500M users)
- Peak capacity: 10M concurrent carts on 10 replicas = 1M per replica

---

## Testing

### Unit Tests
```
Test: addItem()
- Add new item → quantity 1
- Add duplicate → quantity increases
- Add with invalid quantity → error
- Add with negative price → error

Test: applyDiscount()
- Valid coupon → discount applied
- Invalid coupon → error
- Expired coupon → error
- Usage limit exceeded → error

Test: calculateTotal()
- Subtotal correct
- Tax applied (by country)
- Shipping added
- Discount subtracted
```

### Integration Tests
```
Test: Full checkout flow
- Create cart
- Add items (verify inventory service called)
- Apply coupon
- Calculate tax/shipping
- Create checkout (verify payment service ready)
- Verify cart state changed

Test: Inventory sync
- Add item to cart
- Inventory decreases elsewhere
- Item removed from cart (out of stock)

Test: Guest to registered merge
- Create guest cart (add items)
- User logs in
- Carts merged correctly
- Total accurate
```

### Load Tests
```
Scenario: Peak holiday shopping
- 10M concurrent carts
- 100K add/remove operations per second
- <100ms latency p99
- Zero errors

Tool: Apache JMeter or Gatling
```

---

## Monitoring

**Key Metrics:**
- Active carts count
- Add-to-cart success rate (%)
- Average cart value ($)
- Cart abandonment rate (%)
- Checkout conversion rate (%)
- Latency p50/p95/p99
- Cache hit rate (%)
- Error rate (%)

---

