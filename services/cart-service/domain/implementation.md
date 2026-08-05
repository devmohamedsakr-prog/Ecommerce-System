# Cart Service Domain Implementation

## Entity Models

### Cart Entity
```typescript
class Cart {
  id: string;
  userId: string;
  items: CartItem[];
  appliedCoupons: Coupon[];
  status: 'active' | 'abandoned' | 'converted';
  createdAt: Date;
  updatedAt: Date;
  expiresAt: Date;

  constructor(userId: string) {
    this.id = generateId();
    this.userId = userId;
    this.items = [];
    this.appliedCoupons = [];
    this.status = 'active';
    this.createdAt = new Date();
    this.updatedAt = new Date();
    this.expiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000); // 30 days
  }

  addItem(item: CartItem): void {
    if (this.items.length >= 100) {
      throw new Error('Cart capacity exceeded');
    }
    
    const existing = this.items.find(i => i.productId === item.productId);
    if (existing) {
      existing.quantity += item.quantity;
    } else {
      this.items.push(item);
    }
    
    this.updatedAt = new Date();
  }

  removeItem(productId: string): void {
    this.items = this.items.filter(i => i.productId !== productId);
    this.updatedAt = new Date();
  }

  updateItemQuantity(productId: string, quantity: number): void {
    const item = this.items.find(i => i.productId === productId);
    if (!item) throw new Error('Item not found');
    if (quantity <= 0) {
      this.removeItem(productId);
    } else {
      item.quantity = quantity;
    }
    this.updatedAt = new Date();
  }

  applyCoupon(coupon: Coupon): void {
    if (this.appliedCoupons.find(c => c.code === coupon.code)) {
      throw new Error('Coupon already applied');
    }
    this.appliedCoupons.push(coupon);
    this.updatedAt = new Date();
  }

  removeCoupon(couponCode: string): void {
    this.appliedCoupons = this.appliedCoupons.filter(c => c.code !== couponCode);
    this.updatedAt = new Date();
  }

  clearCart(): void {
    this.items = [];
    this.appliedCoupons = [];
    this.updatedAt = new Date();
  }

  isExpired(): boolean {
    return new Date() > this.expiresAt;
  }

  isAbandoned(): boolean {
    const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
    return this.updatedAt < sevenDaysAgo;
  }
}
```

### CartItem Entity
```typescript
class CartItem {
  id: string;
  productId: string;
  quantity: number;
  price: number;
  customizations?: Record<string, any>;
  addedAt: Date;

  constructor(productId: string, quantity: number, price: number) {
    this.id = generateId();
    this.productId = productId;
    this.quantity = quantity;
    this.price = price;
    this.addedAt = new Date();
  }

  getSubtotal(): number {
    return this.quantity * this.price;
  }

  canUpdate(newQuantity: number, maxStock: number): boolean {
    return newQuantity > 0 && newQuantity <= maxStock;
  }
}
```

### Coupon Entity
```typescript
class Coupon {
  id: string;
  code: string;
  type: 'percentage' | 'fixed';
  value: number;
  validFrom: Date;
  validUntil: Date;
  maxUsage: number;
  currentUsage: number;
  minCartValue?: number;

  constructor(code: string, type: 'percentage' | 'fixed', value: number) {
    this.id = generateId();
    this.code = code;
    this.type = type;
    this.value = value;
    this.validFrom = new Date();
    this.validUntil = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000); // 30 days
    this.maxUsage = 1000;
    this.currentUsage = 0;
  }

  isValid(): boolean {
    const now = new Date();
    return now >= this.validFrom && now <= this.validUntil && 
           this.currentUsage < this.maxUsage;
  }

  calculateDiscount(cartValue: number): number {
    if (!this.isValid()) return 0;
    if (this.minCartValue && cartValue < this.minCartValue) return 0;

    if (this.type === 'percentage') {
      return cartValue * (this.value / 100);
    } else {
      return Math.min(this.value, cartValue);
    }
  }

  markAsUsed(): void {
    this.currentUsage++;
  }
}
```

## Value Objects

### Money
```typescript
class Money {
  amount: number;
  currency: string = 'USD';

  constructor(amount: number, currency: string = 'USD') {
    this.amount = amount;
    this.currency = currency;
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error('Cannot add different currencies');
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  subtract(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error('Cannot subtract different currencies');
    }
    return new Money(this.amount - other.amount, this.currency);
  }
}
```

## Domain Events
- `CartCreated` - When cart is initialized
- `ItemAdded` - When item added to cart
- `ItemRemoved` - When item removed from cart
- `CouponApplied` - When coupon applied
- `CartAbandoned` - When cart inactive for 7 days
