# Cart Service API

## Overview
The Cart Service API provides endpoints for managing shopping carts in the e-commerce system. It handles cart operations including adding/removing items, updating quantities, and calculating totals.

## API Endpoints

### Cart Management
- `GET /api/v1/carts/:cartId` - Retrieve cart details
- `POST /api/v1/carts` - Create new cart
- `PUT /api/v1/carts/:cartId` - Update cart
- `DELETE /api/v1/carts/:cartId` - Delete cart

### Cart Items
- `POST /api/v1/carts/:cartId/items` - Add item to cart
- `PUT /api/v1/carts/:cartId/items/:itemId` - Update cart item quantity
- `DELETE /api/v1/carts/:cartId/items/:itemId` - Remove item from cart

### Cart Calculations
- `GET /api/v1/carts/:cartId/totals` - Get cart totals with taxes and discounts
- `POST /api/v1/carts/:cartId/apply-coupon` - Apply coupon code
- `DELETE /api/v1/carts/:cartId/coupons/:couponId` - Remove coupon

## Request/Response Format
All requests and responses use JSON format with standardized error handling and pagination support.

## Authentication
All endpoints require JWT authentication token in Authorization header.

## Rate Limiting
- Standard tier: 1000 requests/hour
- Premium tier: 10000 requests/hour
