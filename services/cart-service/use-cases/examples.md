# Cart Service Use Cases Examples

## Example 1: Complete Shopping Flow

### Step 1: Customer Creates Cart
```typescript
const command = new CreateCartCommand('user_john_123');
const cart = await createCartUseCase.execute(command);
// Result: Cart with ID cart_001
```

### Step 2: Customer Adds First Item
```typescript
const addItemCommand = new AddItemCommand(
  'cart_001',
  'prod_laptop_001',
  1
);
const cart = await addItemToCartUseCase.execute(addItemCommand);
// Result: Cart with 1 item, subtotal: $999.99
```

### Step 3: Customer Adds Second Item
```typescript
const addItemCommand = new AddItemCommand(
  'cart_001',
  'prod_mouse_002',
  2
);
const cart = await addItemToCartUseCase.execute(addItemCommand);
// Result: Cart with 2 items, subtotal: $1,059.97
```

### Step 4: Customer Applies Coupon
```typescript
const couponCommand = new ApplyCouponCommand(
  'cart_001',
  'SAVE20'
);
const cart = await applyCouponUseCase.execute(couponCommand);
// Result: Discount applied: -$211.99, Total: $847.98
```

### Step 5: Customer Reviews Totals
```typescript
const totalsCommand = new CalculateTotalsCommand(
  'cart_001',
  { city: 'New York', state: 'NY' }
);
const totals = await calculateCartTotalsUseCase.execute(totalsCommand);
// Result: 
// - Subtotal: $1,059.97
// - Tax: $84.80
// - Shipping: $9.99
// - Discount: -$211.99
// - Total: $942.77
```

## Example 2: Quantity Adjustment

### Step 1: Add Quantity
```typescript
const updateCommand = new UpdateItemQuantityCommand(
  'cart_001',
  'prod_laptop_001',
  2
);
const cart = await updateItemQuantityUseCase.execute(updateCommand);
// Result: Quantity updated from 1 to 2
```

### Step 2: Calculate New Total
```typescript
const totals = await calculateCartTotalsUseCase.execute(totalsCommand);
// Result: Total updated to reflect new quantity
```

## Example 3: Remove Item

### Step 1: Remove Item from Cart
```typescript
const removeCommand = new RemoveItemCommand(
  'cart_001',
  'prod_mouse_002'
);
const cart = await removeItemFromCartUseCase.execute(removeCommand);
// Result: Mouse removed, subtotal now $1,999.98
```

### Step 2: Verify Coupon Still Applied
```typescript
const totals = await calculateCartTotalsUseCase.execute(totalsCommand);
// Result: Discount recalculated on new subtotal
```

## Example 4: Multiple Coupons

### Step 1: Apply First Coupon
```typescript
let command = new ApplyCouponCommand('cart_001', 'SAVE20');
await applyCouponUseCase.execute(command);
// Discount: -$211.99
```

### Step 2: Apply Second Coupon (if allowed)
```typescript
command = new ApplyCouponCommand('cart_001', 'FREESHIPPING');
await applyCouponUseCase.execute(command);
// Both coupons applied
```

## Example 5: Cart Abandonment Workflow

### Step 1: User Leaves Cart
```typescript
// Cart created: 2024-01-15 10:00 AM
// Last updated: 2024-01-15 10:00 AM
```

### Step 2: System Detects Abandonment
```typescript
// Current time: 2024-01-22 (7 days later)
const abandonedCarts = await findAbandonedCartsUseCase.execute();
// Result: Cart detected as abandoned
```

### Step 3: System Sends Email & Archives
```typescript
await abandonInactiveCartsUseCase.execute();
// Email sent to user_john_123@email.com
// Cart status changed to 'abandoned'
```

## Example 6: Error Handling

### Insufficient Stock
```typescript
const command = new AddItemCommand(
  'cart_001',
  'prod_limited_001',
  100 // Only 10 in stock
);
try {
  await addItemToCartUseCase.execute(command);
} catch (error) {
  console.log('Error: Insufficient Stock');
  // System returns: Available: 10, Requested: 100
}
```

### Invalid Coupon
```typescript
const command = new ApplyCouponCommand('cart_001', 'EXPIRED20');
try {
  await applyCouponUseCase.execute(command);
} catch (error) {
  console.log('Error: Coupon Not Found or Expired');
}
```

### Cart Not Found
```typescript
const command = new CalculateTotalsCommand('cart_invalid', {});
try {
  await calculateCartTotalsUseCase.execute(command);
} catch (error) {
  console.log('Error: Cart Not Found');
}
```
