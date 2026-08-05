# Cart Service Scenarios Implementation

## Scenario 1: Successful Shopping Session

### Setup
```typescript
const userId = 'user_alex_001';
const productIds = ['laptop_001', 'mouse_002', 'keyboard_003'];
```

### Flow
```typescript
// Step 1: Create cart
const createCommand = new CreateCartCommand(userId);
const cart = await createCartUseCase.execute(createCommand);
assert(cart.id !== null);
assert(cart.status === 'active');

// Step 2: Add laptop
const addLaptopCommand = new AddItemCommand(
  cart.id,
  'laptop_001',
  1
);
const cart1 = await addItemToCartUseCase.execute(addLaptopCommand);
assert(cart1.items.length === 1);
assert(cart1.items[0].quantity === 1);

// Step 3: Add mouse
const addMouseCommand = new AddItemCommand(
  cart.id,
  'mouse_002',
  2
);
const cart2 = await addItemToCartUseCase.execute(addMouseCommand);
assert(cart2.items.length === 2);

// Step 4: Apply coupon
const couponCommand = new ApplyCouponCommand(cart.id, 'SUMMER20');
const cart3 = await applyCouponUseCase.execute(couponCommand);
assert(cart3.appliedCoupons.length === 1);

// Step 5: Calculate totals
const totalsCommand = new CalculateTotalsCommand(
  cart.id,
  { zipCode: '10001', state: 'NY' }
);
const totals = await calculateCartTotalsUseCase.execute(totalsCommand);
assert(totals.total > 0);
assert(totals.discount > 0);
```

## Scenario 2: Duplicate Item Addition

### Setup
```typescript
const cartId = 'cart_dup_001';
const productId = 'laptop_001';
```

### Flow
```typescript
// Step 1: Add item first time
const command1 = new AddItemCommand(cartId, productId, 1);
const cart1 = await addItemToCartUseCase.execute(command1);
assert(cart1.items.length === 1);
assert(cart1.items[0].quantity === 1);

// Step 2: Add same item again
const command2 = new AddItemCommand(cartId, productId, 2);
const cart2 = await addItemToCartUseCase.execute(command2);

// Step 3: Verify merge
assert(cart2.items.length === 1); // Not 2 items, merged to 1
assert(cart2.items[0].quantity === 3); // 1 + 2 = 3
assert(cart2.items[0].productId === productId);
```

## Scenario 3: Stock Depletion During Shopping

### Setup
```typescript
const productId = 'limited_stock_001';
let availableStock = 5;
```

### Flow
```typescript
// Step 1: Add 3 items (stock: 5)
const command1 = new AddItemCommand(cartId, productId, 3);
await addItemToCartUseCase.execute(command1);
availableStock = 2;

// Step 2: Attempt to add 3 more (stock: 2) - should fail
const command2 = new AddItemCommand(cartId, productId, 3);
try {
  await addItemToCartUseCase.execute(command2);
  assert(false, 'Should throw InsufficientStockException');
} catch (error) {
  assert(error instanceof InsufficientStockException);
  assert(error.requested === 3);
  assert(error.available === 2);
}

// Step 3: Add 2 items instead (stock: 2) - should succeed
const command3 = new AddItemCommand(cartId, productId, 2);
const cart = await addItemToCartUseCase.execute(command3);
assert(cart.items.length === 1);
assert(cart.items[0].quantity === 5); // 3 + 2
```

## Scenario 4: Expired Coupon

### Setup
```typescript
const cartId = 'cart_expired_001';
const expiredCoupon = 'OLD_COUPON_2023';
```

### Flow
```typescript
// Step 1: Attempt to apply expired coupon
const command = new ApplyCouponCommand(cartId, expiredCoupon);
try {
  await applyCouponUseCase.execute(command);
  assert(false, 'Should throw CouponNotFoundException');
} catch (error) {
  assert(error instanceof CouponNotFoundException);
}

// Step 2: Verify coupon not applied
const cart = await cartRepository.findById(cartId);
assert(cart.appliedCoupons.length === 0);
```

## Scenario 5: Cart Abandonment Detection

### Setup
```typescript
// Create cart 7 days ago
const createdAt = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
const cart = new Cart('user_lazy_001');
cart.updatedAt = createdAt;
```

### Flow
```typescript
// Step 1: Verify cart is active
assert(cart.status === 'active');
assert(cart.isAbandoned() === true);

// Step 2: Run abandonment process
const count = await abandonInactiveCartsUseCase.execute();
assert(count >= 1);

// Step 3: Verify cart marked abandoned
const updatedCart = await cartRepository.findById(cart.id);
assert(updatedCart.status === 'abandoned');
```

## Scenario 6: Concurrent Cart Updates

### Setup
```typescript
const cartId = 'cart_concurrent_001';
const product1 = 'prod_001';
const product2 = 'prod_002';
```

### Flow
```typescript
// Simulate concurrent requests
const command1 = new AddItemCommand(cartId, product1, 1);
const command2 = new AddItemCommand(cartId, product2, 1);

// Execute in parallel
const [cart1, cart2] = await Promise.all([
  addItemToCartUseCase.execute(command1),
  addItemToCartUseCase.execute(command2)
]);

// Verify both items added
const finalCart = await cartRepository.findById(cartId);
assert(finalCart.items.length === 2);
assert(finalCart.items.map(i => i.productId).includes(product1));
assert(finalCart.items.map(i => i.productId).includes(product2));
```

## Scenario 7: Cart Capacity Exceeded

### Setup
```typescript
const cartId = 'cart_full_001';
const MAX_ITEMS = 100;
```

### Flow
```typescript
// Step 1: Add 100 items
for (let i = 0; i < MAX_ITEMS; i++) {
  const command = new AddItemCommand(cartId, `prod_${i}`, 1);
  await addItemToCartUseCase.execute(command);
}

// Step 2: Attempt to add 101st item
const command = new AddItemCommand(cartId, 'prod_100', 1);
try {
  await addItemToCartUseCase.execute(command);
  assert(false, 'Should throw CartCapacityExceededException');
} catch (error) {
  assert(error instanceof CartCapacityExceededException);
}

// Step 3: Verify cart has exactly 100 items
const cart = await cartRepository.findById(cartId);
assert(cart.items.length === MAX_ITEMS);
```

## Scenario 8: Remove Item and Recalculate

### Setup
```typescript
const cartId = 'cart_remove_001';
```

### Flow
```typescript
// Step 1: Add multiple items
await addItemToCartUseCase.execute(
  new AddItemCommand(cartId, 'prod_1', 2)
);
await addItemToCartUseCase.execute(
  new AddItemCommand(cartId, 'prod_2', 1)
);

// Step 2: Calculate initial totals
let totals = await calculateCartTotalsUseCase.execute(
  new CalculateTotalsCommand(cartId, {})
);
const initialTotal = totals.total;

// Step 3: Remove item
await removeItemFromCartUseCase.execute(
  new RemoveItemCommand(cartId, 'prod_2')
);

// Step 4: Recalculate totals
totals = await calculateCartTotalsUseCase.execute(
  new CalculateTotalsCommand(cartId, {})
);
assert(totals.total < initialTotal);
```
