# Cart Service Use Cases

## Overview
Use cases define the primary business workflows and interactions in the cart service, implementing the application layer between APIs and domain models.

## Core Use Cases

### 1. Add Item to Cart
- Validate product exists
- Check inventory availability
- Add or merge items
- Update cart timestamps
- Publish ItemAdded event

### 2. Remove Item from Cart
- Locate item in cart
- Remove item
- Update cart
- Publish ItemRemoved event

### 3. Apply Coupon
- Validate coupon
- Check eligibility
- Apply to cart
- Calculate discount
- Publish CouponApplied event

### 4. Remove Coupon
- Locate coupon in cart
- Remove coupon
- Recalculate totals
- Publish CouponRemoved event

### 5. Calculate Cart Totals
- Sum item prices
- Calculate tax
- Add shipping cost
- Apply discount
- Return total

### 6. Update Item Quantity
- Validate quantity
- Check stock
- Update item
- Publish ItemUpdated event

### 7. Clear Cart
- Remove all items
- Clear coupons
- Reset cart
- Publish CartCleared event

### 8. Manage Cart Lifecycle
- Track cart creation
- Mark abandoned carts
- Handle expiration
- Archive completed carts
