# Cart Service Domain Examples

## Example 1: Creating and Managing a Shopping Cart

### Scenario
Customer starts shopping and adds products to their cart.

### Domain Model Interaction
```typescript
// Create new cart
const cart = new Cart('user_john_123');

// Add first item
const item1 = new CartItem('prod_laptop_001', 1, 999.99);
cart.addItem(item1);

// Add second item
const item2 = new CartItem('prod_mouse_002', 2, 29.99);
cart.addItem(item2);

// Result
// cart.items = [
//   { productId: 'prod_laptop_001', quantity: 1, price: 999.99 },
//   { productId: 'prod_mouse_002', quantity: 2, price: 29.99 }
// ]
// Total items: 3
```

## Example 2: Applying Discount Coupons

### Scenario
Customer has a 20% off coupon and wants to apply it.

### Domain Model Interaction
```typescript
// Create coupon
const coupon = new Coupon('SAVE20', 'percentage', 20);

// Validate coupon
if (coupon.isValid()) {
  // Calculate subtotal
  const subtotal = cart.items.reduce((sum, item) => 
    sum + (item.quantity * item.price), 0
  ); // 1089.96

  // Calculate discount
  const discount = coupon.calculateDiscount(subtotal); // 217.99

  // Apply coupon
  cart.applyCoupon(coupon);
  coupon.markAsUsed();

  // Total after discount: 871.97
}
```

## Example 3: Cart Expiration and Abandonment

### Scenario
Monitoring cart lifecycle and status transitions.

### Domain Model Interaction
```typescript
// Cart created
const cart = new Cart('user_jane_456');
console.log(cart.status); // 'active'

// ... time passes, no updates for 8 days ...

if (cart.isAbandoned()) {
  cart.status = 'abandoned';
  // Trigger notification event
}

// ... time passes, 30 days total ...

if (cart.isExpired()) {
  // Archive cart
  cart.status = 'converted';
}
```

## Example 4: Handling Customizations

### Scenario
Customer adds a product with customizations (e.g., engraving, color).

### Domain Model Interaction
```typescript
// Create item with customizations
const customItem = new CartItem('prod_tshirt_001', 1, 25.99);
customItem.customizations = {
  color: 'blue',
  size: 'XL',
  engraving: 'JOHN'
};

cart.addItem(customItem);

// Apply customization fee
const customizationFee = 5.00;
const totalWithCustomization = customItem.getSubtotal() + customizationFee;
// 30.99
```

## Example 5: Multiple Coupons and Complex Discounts

### Scenario
Customer applies multiple coupons with different discount types.

### Domain Model Interaction
```typescript
// Create coupons
const percentageCoupon = new Coupon('PERCENT15', 'percentage', 15);
const fixedCoupon = new Coupon('FIXED10', 'fixed', 10);

// Subtotal: 1089.96

// Apply first coupon
cart.applyCoupon(percentageCoupon);
const discount1 = percentageCoupon.calculateDiscount(1089.96); // 163.49

// Apply second coupon
cart.applyCoupon(fixedCoupon);
const discount2 = fixedCoupon.calculateDiscount(1089.96); // 10.00

// Total discount: 173.49
// Final price: 916.47
```

## Example 6: Money Calculations with Currency

### Scenario
System calculates totals with proper currency handling.

### Domain Model Interaction
```typescript
const subtotal = new Money(1089.96, 'USD');
const tax = new Money(87.20, 'USD');
const discount = new Money(50.00, 'USD');

const total = subtotal.add(tax).subtract(discount);
// Result: Money { amount: 1127.16, currency: 'USD' }

// Error handling
try {
  const euroAmount = new Money(100, 'EUR');
  const usdAmount = new Money(100, 'USD');
  subtotal.add(euroAmount); // Throws error
} catch (e) {
  console.log('Cannot add different currencies');
}
```

## Example 7: Cart Cleanup and Reset

### Scenario
Customer decides to clear their cart and start over.

### Domain Model Interaction
```typescript
// Before
// cart.items.length = 5
// cart.appliedCoupons.length = 2

// Clear cart
cart.clearCart();

// After
// cart.items = []
// cart.appliedCoupons = []
// cart.status = 'active'
// cart.updatedAt = new Date() // Current time
```
