# Cart Service Scenarios Examples

## Example Scenario 1: Complete Shopping Journey

### User Profile
- Name: John Doe
- ID: user_john_001
- Location: New York, NY

### Step-by-Step Flow

#### 1. Browse Products
```
Action: User opens e-commerce website
Result: User authenticated, session created
Time: 2024-01-15 10:00 AM
```

#### 2. Create Cart
```
Request: GET /carts/user_john_001
Response: 201 Created
Cart ID: cart_jd_001
Status: active
```

#### 3. Add Laptop to Cart
```
Request: POST /carts/cart_jd_001/items
Body: { productId: 'laptop_dell_001', quantity: 1 }
Response: 200 OK
Subtotal: $999.99
Items: 1
```

#### 4. Add Mouse to Cart
```
Request: POST /carts/cart_jd_001/items
Body: { productId: 'mouse_logitech_002', quantity: 2 }
Response: 200 OK
Subtotal: $1,059.97
Items: 2
```

#### 5. Apply Coupon
```
Request: POST /carts/cart_jd_001/apply-coupon
Body: { couponCode: 'SUMMER20' }
Response: 200 OK
Discount Applied: -$211.99 (20% off)
New Total: $847.98
```

#### 6. Review Totals
```
Request: GET /carts/cart_jd_001/totals
Response: {
  subtotal: 1059.97,
  tax: 84.80,
  shipping: 9.99,
  discount: -211.99,
  total: 942.77
}
```

#### 7. Proceed to Checkout
```
Status: Cart ready for payment
Items in cart: 2
Final Amount: $942.77
```

## Example Scenario 2: Out of Stock During Shopping

### User Profile
- Name: Jane Smith
- ID: user_jane_001

### Step-by-Step Flow

#### 1. Add Limited Item (5 in stock)
```
Product: Limited Edition Headphones
Stock: 5 units
Request: POST /carts/cart_js_001/items
Body: { productId: 'headphone_limited_001', quantity: 3 }
Response: 200 OK
Remaining Stock: 2
```

#### 2. Try to Add More (stock runs out)
```
Another customer buys last 2 units
Stock: 0 units
Jane attempts: { productId: 'headphone_limited_001', quantity: 2 }
Response: 400 Bad Request
Error: "Insufficient stock - Available: 0, Requested: 2"
```

#### 3. Notification
```
Email Sent to: jane@example.com
Subject: "Limited Edition Headphones - Stock Unavailable"
Body: "Sorry, the item is no longer in stock. 
       We'll notify you when it's available again."
```

## Example Scenario 3: Abandoned Cart Recovery

### User Profile
- Name: Mike Brown
- ID: user_mike_001

### Timeline

#### Day 1 - Shopping Session
```
Time: 2024-01-15 14:00 PM
Action: Mike adds items to cart
Cart Value: $500.00
Items: 3
Status: active
```

#### Day 7 - Abandonment Detection
```
Time: 2024-01-22 15:00 PM
System Check: Last updated 7 days ago
Status Changed: abandoned
Event: CartAbandonedEvent published
```

#### Day 7 - Recovery Email
```
Email Sent to: mike@example.com
Subject: "You left $500 in your cart!"
Body: "Complete your purchase now and get 15% off"
Link: https://ecommerce.com/cart/cart_mb_001
Offer Expires: 2024-01-29
```

#### Day 8 - Email Click
```
Time: 2024-01-23 10:30 AM
Action: Mike clicks email link
Result: Cart reopened
Status: active
Offer Applied: $75 discount
New Total: $425.00
```

## Example Scenario 4: Multiple Coupons Stacking

### User Profile
- Name: Sarah Lee
- ID: user_sarah_001

### Coupon Details
- SUMMER20: 20% off (valid: $100+ minimum)
- NEWUSER10: 10% off additional (valid: first purchase)
- FREESHIP: Free shipping (valid: no minimum)

### Flow

#### 1. Add Items
```
Item 1: Shoes - $120.00
Item 2: Jacket - $150.00
Subtotal: $270.00
```

#### 2. Apply First Coupon
```
Request: Apply SUMMER20
Discount Calculation: $270 * 20% = -$54.00
New Subtotal: $216.00
Coupons Applied: 1
```

#### 3. Apply Second Coupon
```
Request: Apply NEWUSER10
Discount Calculation: $216 * 10% = -$21.60
New Subtotal: $194.40
Coupons Applied: 2
```

#### 4. Apply Shipping Coupon
```
Request: Apply FREESHIP
Shipping Removed: -$9.99
Final Subtotal: $194.40
Shipping: $0.00
Tax: $15.55
Final Total: $209.95

Total Savings: $84.59 (31% off original price)
```

## Example Scenario 5: Cart Expiration

### Timeline

#### Creation
```
Date Created: 2024-01-15 10:00 AM
Expiration Date: 2024-02-14 10:00 AM (30 days)
```

#### Before Expiration (Day 25)
```
Date: 2024-02-09 10:00 AM
Request: GET /carts/cart_xyz_001
Response: 200 OK
Status: active
Items: 5
```

#### After Expiration (Day 31)
```
Date: 2024-02-15 10:00 AM
Request: GET /carts/cart_xyz_001
Response: 404 Not Found
Error: "Cart has expired and been archived"
Message: "Create a new cart to continue shopping"
```

## Example Scenario 6: Concurrent Operations

### Setup
```
Cart ID: cart_concurrent_001
User: user_alice_001
Time: 2024-01-15 10:00:00 AM
```

### Operations (Simultaneous)

#### Operation 1 (Browser Tab 1)
```
Time: 10:00:00.100
Action: Add Laptop
Product: prod_laptop_001
Quantity: 1
```

#### Operation 2 (Browser Tab 2)
```
Time: 10:00:00.105
Action: Add Monitor
Product: prod_monitor_001
Quantity: 2
```

#### Operation 3 (Mobile App)
```
Time: 10:00:00.110
Action: Apply Coupon
Coupon: SUMMER20
```

#### Final Result
```
Time: 10:00:01.500
Cart Contents:
- Laptop: 1 unit
- Monitor: 2 units
Coupon: SUMMER20 applied
Status: All operations succeeded
```

## Example Scenario 7: Error Recovery

### Scenario: Network Timeout

#### Initial State
```
User adding item to cart
Product: expensive_item
Quantity: 1
Network: Good connection
```

#### Timeout Occurs
```
Request sent at: 10:00:00
Timeout at: 10:00:05 (no response)
Response: 504 Gateway Timeout
```

#### Retry Logic
```
Attempt 1: Failed - Network error
Attempt 2: Failed - Timeout
Attempt 3: Success - Item added
Total Retries: 2 (exponential backoff: 1s, 2s)
```

#### User Notification
```
Message: "Adding to cart..."
Status Changes:
- Pending (3s)
- Success (with checkmark)
Confirmation: Item added to cart
```
