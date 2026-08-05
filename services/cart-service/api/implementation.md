# Cart Service API Implementation

## Technology Stack
- Framework: Express.js/Node.js
- Language: TypeScript
- Database: MongoDB
- Cache: Redis
- Message Queue: RabbitMQ

## API Architecture

### Request/Response Models

```typescript
interface CartRequest {
  userId: string;
  items: CartItem[];
  couponCode?: string;
}

interface CartItem {
  productId: string;
  quantity: number;
  price: number;
  customizations?: Record<string, any>;
}

interface CartResponse {
  cartId: string;
  userId: string;
  items: CartItem[];
  subtotal: number;
  tax: number;
  discount: number;
  total: number;
  appliedCoupons: Coupon[];
  createdAt: Date;
  updatedAt: Date;
}
```

### Core Endpoints Implementation

#### POST /api/v1/carts
Creates a new shopping cart for a user.

```typescript
router.post('/carts', authenticateToken, async (req, res) => {
  try {
    const { userId } = req.body;
    
    const cart = new Cart({
      userId,
      items: [],
      status: 'active',
      createdAt: new Date()
    });
    
    await cart.save();
    res.status(201).json(cart);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

#### GET /api/v1/carts/:cartId
Retrieves cart details with caching.

```typescript
router.get('/carts/:cartId', authenticateToken, async (req, res) => {
  try {
    const cacheKey = `cart:${req.params.cartId}`;
    const cached = await redis.get(cacheKey);
    
    if (cached) {
      return res.json(JSON.parse(cached));
    }
    
    const cart = await Cart.findById(req.params.cartId)
      .populate('items.productId');
    
    if (!cart) {
      return res.status(404).json({ error: 'Cart not found' });
    }
    
    await redis.setex(cacheKey, 3600, JSON.stringify(cart));
    res.json(cart);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

#### POST /api/v1/carts/:cartId/items
Adds item to cart with inventory validation.

```typescript
router.post('/carts/:cartId/items', authenticateToken, async (req, res) => {
  try {
    const { productId, quantity } = req.body;
    
    // Validate inventory
    const product = await Product.findById(productId);
    if (!product || product.stock < quantity) {
      return res.status(400).json({ error: 'Insufficient stock' });
    }
    
    const cart = await Cart.findById(req.params.cartId);
    const existingItem = cart.items.find(i => i.productId === productId);
    
    if (existingItem) {
      existingItem.quantity += quantity;
    } else {
      cart.items.push({
        productId,
        quantity,
        price: product.price
      });
    }
    
    await cart.save();
    
    // Invalidate cache
    await redis.del(`cart:${req.params.cartId}`);
    
    res.json(cart);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

#### GET /api/v1/carts/:cartId/totals
Calculates cart totals with tax and discounts.

```typescript
router.get('/carts/:cartId/totals', authenticateToken, async (req, res) => {
  try {
    const cart = await Cart.findById(req.params.cartId)
      .populate('items.productId');
    
    const subtotal = cart.items.reduce((sum, item) => 
      sum + (item.quantity * item.price), 0);
    
    const tax = subtotal * 0.08; // 8% tax
    let discount = 0;
    
    // Apply coupon discounts
    for (const coupon of cart.appliedCoupons) {
      if (coupon.type === 'percentage') {
        discount += subtotal * (coupon.value / 100);
      } else {
        discount += coupon.value;
      }
    }
    
    const total = subtotal + tax - discount;
    
    res.json({
      subtotal,
      tax,
      discount,
      total
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

## Error Handling
- 400 Bad Request: Invalid input
- 401 Unauthorized: Missing/invalid token
- 404 Not Found: Cart/item not found
- 409 Conflict: Inventory conflict
- 500 Server Error: Internal error

## Caching Strategy
- Cart data cached in Redis for 1 hour
- Cache invalidated on cart modifications
- TTL: 3600 seconds

## Rate Limiting Implementation
Uses express-rate-limit middleware with token bucket algorithm.
