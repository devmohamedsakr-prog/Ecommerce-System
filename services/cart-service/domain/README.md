# Cart Service Domain Model

## Overview
The domain layer defines the core business entities and logic for shopping cart management, including cart structure, item management, and pricing calculations.

## Core Entities

### Cart
- Unique identifier (cartId)
- User association (userId)
- Status (active, abandoned, converted)
- Items collection
- Applied coupons
- Timestamps (createdAt, updatedAt, expiresAt)

### Cart Item
- Product reference (productId)
- Quantity
- Unit price
- Customizations/options
- Added timestamp

### Coupon/Discount
- Coupon code
- Discount type (percentage, fixed amount)
- Discount value
- Validity period
- Applied date

## Business Rules
1. Cart persists for 30 days after last modification
2. Maximum 100 items per cart
3. Quantity cannot exceed available stock
4. Coupons must be valid and not expired
5. Multiple coupons can be combined
6. Cart is abandoned if inactive for 7 days

## Relationships
- One User has many Carts
- One Cart has many Items
- One Cart can have many Coupons
- One Product can appear in many Carts
