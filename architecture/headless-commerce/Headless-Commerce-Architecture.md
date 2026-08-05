# Headless Commerce Architecture Pattern

**Status:** Enterprise Architecture Pattern | **Priority:** HIGH | **Benefit:** 10x faster feature launches

---

## Overview

Headless commerce decouples storefront (frontend) from commerce engine (backend). Frontend and backend ship independently on separate release cycles. Enables multiple storefronts (web, mobile, social) to share one commerce engine.

## Traditional vs Headless

### Traditional (Monolithic)
```
┌─────────────────────────────────────┐
│   E-Commerce Platform               │
│  (Shopify, Magento, WooCommerce)    │
├─────────────────────────────────────┤
│ ┌────────────────┐ ┌──────────────┐ │
│ │   Storefront   │ │   Catalog    │ │
│ │   (Frontend)   │ │   (Backend)  │ │
│ ├────────────────┤ ├──────────────┤ │
│ │   Cart         │ │   Inventory  │ │
│ │   Checkout     │ │   Orders     │ │
│ └────────────────┘ └──────────────┘ │
└─────────────────────────────────────┘

Problem: 
- Feature requires: 3-week release cycle
- Monolithic: All testing, deployment, rollback risk
- One team blocks another
- Scale: Frontend and backend must scale together
```

### Headless (Decoupled)
```
┌────────────────────┐              ┌─────────────────────────┐
│   Storefronts      │              │   Commerce Engine       │
├────────────────────┤              ├─────────────────────────┤
│ ┌──────────────┐   │              │ ┌─────────────────────┐ │
│ │ Web Store    │   │              │ │ Catalog Service     │ │
│ │ (React)      │   │              │ │ Order Service       │ │
│ │ Release: Any │───┼──────API─────┤ │ Payment Service     │ │
│ │ time         │   │              │ │ Inventory Service   │ │
│ └──────────────┘   │              │ │ Release: Any time   │ │
│                    │              │ └─────────────────────┘ │
│ ┌──────────────┐   │              │                         │
│ │ Mobile App   │   │              │                         │
│ │ (React Native)   │              │                         │
│ │ Release: Any │───┼──────API─────┤                         │
│ │ time         │   │              │                         │
│ └──────────────┘   │              │                         │
│                    │              │                         │
│ ┌──────────────┐   │              │                         │
│ │ Social Shop  │   │              │                         │
│ │ (Instagram)  │───┼──────API─────┤                         │
│ │ Release: Any │   │              │                         │
│ │ time         │   │              │                         │
│ └──────────────┘   │              │                         │
└────────────────────┘              └─────────────────────────┘

Benefits:
- Web shop updates: 1 hour cycle
- Mobile app: Update via App Store
- Social shop: Deploy whenever
- Each team independent
- Scale front/back separately
```

## Headless Benefits

### 1. Faster Feature Launches
```
Monolithic: Feature requires backend AND frontend
- Month 1: Design & development
- Week 1 Month 2: Testing
- Week 2 Month 2: Deployment (coordinated)
- Week 3 Month 2: Rollback on issues

Total: 6-8 weeks

Headless: Backend and frontend independent
- Backend feature: 2 weeks
- Deploy backend (no frontend impact)
- Frontend feature: 1 week
- Deploy frontend
- Both live after 3 weeks
- Can rollback independently
```

### 2. Multiple Storefronts (Same Backend)
```
Web Store (React)
- Optimized for desktop
- Complex filtering, recommendations

Mobile App (React Native)
- Optimized for mobile UX
- Checkout in 3 screens

Social Commerce (Instagram)
- Shoppable posts
- Direct to cart
- One-click checkout

All use same catalog, orders, inventory backend
Reduced development: 2 storefronts, not 3
```

### 3. Technology Independence
```
Current tech stack:
- Web: React v18
- Mobile: React Native
- Backend: Node.js
- Database: PostgreSQL

Future: Want to upgrade?
- Web: Switch to Vue (no backend changes)
- Mobile: Switch to Flutter (no backend changes)
- Backend: Switch to Python (no frontend changes)

Flexibility: Not locked into one technology
```

### 4. Scalability Independence
```
Black Friday: Traffic spike 10x
- Frontend servers: Scale to 100 instances
- Backend: Keep at 50 instances (handles load fine)
- Cost: $2K additional compute, not $10K

vs Monolithic:
- Everything together: Scale all or nothing
- Cost: $10K + wasted backend resources
```

## Headless Architecture Implementation

### Commerce APIs

**Product API**
```
GET /api/products
GET /api/products/{id}
GET /api/categories
GET /api/search
GET /api/recommendations

Features:
- Full product data (images, specs, reviews)
- Real-time inventory
- Pricing (including customer-specific)
- Recommendations
- Search with faceting
```

**Cart API**
```
POST /api/carts
POST /api/carts/{id}/items
PUT /api/carts/{id}/items/{item_id}
DELETE /api/carts/{id}/items/{item_id}
GET /api/carts/{id}

Features:
- Add/remove items
- Update quantities
- Apply coupon codes
- Persistent across devices
- Merge carts
```

**Checkout API**
```
POST /api/orders
GET /api/orders/{id}
PUT /api/orders/{id}/shipping
PUT /api/orders/{id}/payment
POST /api/orders/{id}/confirm

Features:
- Create order from cart
- Set shipping address
- Calculate tax & shipping
- Process payment
- Order confirmation
```

**Customer API**
```
POST /api/customers (register)
GET /api/customers/{id}
PUT /api/customers/{id}
GET /api/customers/{id}/orders
GET /api/customers/{id}/addresses

Features:
- User authentication
- Profile management
- Address book
- Order history
- Saved payment methods
```

### Frontend Examples

#### Web Storefront (React)

```jsx
import React from 'react';
import { useCart, useProducts } from '@commerce/hooks';

export function ProductPage({ productId }) {
  const { product, loading } = useProducts.getById(productId);
  const { addToCart } = useCart();

  if (loading) return <Spinner />;

  return (
    <div className="product">
      <img src={product.image} />
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <Price product={product} />
      <button onClick={() => addToCart(product)}>
        Add to Cart
      </button>
    </div>
  );
}

// Same component works for web, easy to adapt for mobile
```

#### Mobile App (React Native)

```jsx
import React from 'react';
import { View, Image, Text, TouchableOpacity } from 'react-native';
import { useCart, useProducts } from '@commerce/hooks';

export function ProductScreen({ route }) {
  const productId = route.params.id;
  const { product, loading } = useProducts.getById(productId);
  const { addToCart } = useCart();

  if (loading) return <Spinner />;

  return (
    <View className="product">
      <Image source={{ uri: product.image }} />
      <Text>{product.name}</Text>
      <Text>{product.description}</Text>
      <Price product={product} />
      <TouchableOpacity onPress={() => addToCart(product)}>
        <Text>Add to Cart</Text>
      </TouchableOpacity>
    </View>
  );
}

// Same API calls, different UI framework
```

#### Social Commerce (Instagram)

```jsx
// Shoppable post on Instagram
// No full storefront, just purchase flow

function ShoppablePost({ post }) {
  const items = post.products;
  const { addToCart } = useCart();

  return (
    <div className="post">
      <img src={post.image} />
      {items.map(item => (
        <ShoppableTag key={item.id} item={item} />
      ))}
    </div>
  );
}

function ShoppableTag({ item }) {
  const { addToCart } = useCart();

  return (
    <div className="tag" onClick={() => addToCart(item)}>
      <div className="card">
        <img src={item.image} width="100" />
        <span>{item.name}</span>
        <span>${item.price}</span>
        <button>Buy Now</button>
      </div>
    </div>
  );
}
```

## Headless Platforms

| Platform | Model | Scale | Complexity |
|----------|-------|-------|------------|
| **Shopify Headless** | API-first | Mid-Large | Medium |
| **BigCommerce** | API-first | Large | Medium |
| **Contentful** | CMS + Commerce | Small-Large | Low |
| **Sanity** | CMS + Commerce | Small-Large | Low |
| **Strapi** | OSS Headless | Any | High (setup) |
| **Custom Build** | Your APIs | Any | Very high |

## Migration Path: Monolithic → Headless

### Phase 1: API Layer (Month 1-2)
```
Keep monolith running
Build APIs for:
- Products
- Cart
- Orders

Monolith still serves web, but via API
```

### Phase 2: Frontend Framework (Month 2-3)
```
Build new web storefront:
- React/Vue app
- Calls new APIs
- Feature parity with old storefront
- Dark launch to 1% of users
- Test for 1 week
```

### Phase 3: Gradual Rollout (Month 3-4)
```
Route increasing % of traffic to new frontend:
- Week 1: 1% traffic
- Week 2: 5% traffic
- Week 3: 25% traffic
- Week 4: 100% traffic

Rollback easy if issues
```

### Phase 4: Decommission (Month 4+)
```
Old monolith can be sunset
Recompute cost: 50% infrastructure savings
Resources allocated to new features
```

## Monitoring Headless Systems

```
Track separately:
- Frontend metrics (JS errors, page load time)
- Backend metrics (API latency, errors)
- Business metrics (conversion, AOV)

Frontend dashboard:
- Error rate
- Page performance (Lighthouse)
- User engagement

Backend dashboard:
- API response times (p50, p99)
- Error rates by endpoint
- Throughput (requests/sec)

Alert on:
- Frontend error rate > 1%
- API latency p99 > 500ms
- Conversion rate drop > 5%
```

## Costs & Considerations

### Cost Comparison

**Monolithic** (Shopify)
- $300/month + 2.9% + $0.30 per transaction
- Team: 3 developers
- Customization: Limited

**Headless** (Custom)
- Infrastructure: $2K/month
- Frontend hosting: $500/month
- API server: $1500/month
- Team: 5 developers (frontend + backend)
- Customization: Unlimited

Break-even: $1M+ annual revenue or highly custom needs

### When to Go Headless

✓ **Go headless if:**
- Revenue > $1M
- Need multiple channels (web, mobile, social)
- Custom storefront critical
- Fast feature releases required
- Unique business logic

✗ **Stay monolithic if:**
- Revenue < $100K
- Single web storefront only
- Off-the-shelf solutions sufficient
- Small team
- Managed service preferred

---

**Pattern Version:** 1.0 | **Status:** Enterprise Pattern | **ROI:** 10x faster launches, 40% cost reduction at scale

