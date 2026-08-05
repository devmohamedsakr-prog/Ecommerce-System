# Cart Service Scenarios

## Overview
Scenarios document real-world business situations and how the cart service handles them, including happy paths, edge cases, and error handling.

## Scenario Categories

### Happy Path Scenarios
1. Customer successfully creates cart and adds items
2. Customer applies valid coupon successfully
3. Customer checks out from populated cart
4. Customer browses and adds multiple items

### Edge Case Scenarios
1. Customer adds duplicate items to cart
2. Customer applies expired coupon
3. Concurrent updates to same cart
4. Stock level decreases while item in cart
5. Coupon becomes invalid before checkout

### Error Scenarios
1. Product not found when adding to cart
2. Insufficient stock for quantity requested
3. Cart exceeds maximum item limit
4. Database connection failure
5. External service timeout
6. Coupon application fails validation

### Business Rule Scenarios
1. Cart automatically abandoned after 7 days of inactivity
2. Cart expires after 30 days
3. Multiple coupons stacking rules
4. Tax calculation based on location
5. Shipping cost determination

## Performance Scenarios
1. Large cart with 100+ items
2. Concurrent add operations
3. Cache hit vs miss comparison
4. Bulk cart operations
