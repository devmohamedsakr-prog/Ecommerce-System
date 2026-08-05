# Cart Service API Examples

## Use Case 1: Create Cart and Add Items

### Request: Create Cart
```bash
POST /api/v1/carts
Authorization: Bearer {token}
Content-Type: application/json

{
  "userId": "user123"
}
```

### Response
```json
{
  "cartId": "cart_abc123",
  "userId": "user123",
  "items": [],
  "subtotal": 0,
  "tax": 0,
  "discount": 0,
  "total": 0,
  "appliedCoupons": [],
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

### Request: Add Item to Cart
```bash
POST /api/v1/carts/cart_abc123/items
Authorization: Bearer {token}
Content-Type: application/json

{
  "productId": "prod_xyz789",
  "quantity": 2
}
```

### Response
```json
{
  "cartId": "cart_abc123",
  "userId": "user123",
  "items": [
    {
      "productId": "prod_xyz789",
      "quantity": 2,
      "price": 49.99
    }
  ],
  "subtotal": 99.98,
  "tax": 7.99,
  "discount": 0,
  "total": 107.97,
  "appliedCoupons": [],
  "updatedAt": "2024-01-15T10:35:00Z"
}
```

## Use Case 2: Apply Coupon Discount

### Request: Apply Coupon
```bash
POST /api/v1/carts/cart_abc123/apply-coupon
Authorization: Bearer {token}
Content-Type: application/json

{
  "couponCode": "SAVE20"
}
```

### Response
```json
{
  "cartId": "cart_abc123",
  "userId": "user123",
  "items": [
    {
      "productId": "prod_xyz789",
      "quantity": 2,
      "price": 49.99
    }
  ],
  "subtotal": 99.98,
  "tax": 7.99,
  "discount": 19.99,
  "total": 87.98,
  "appliedCoupons": [
    {
      "couponId": "SAVE20",
      "type": "percentage",
      "value": 20
    }
  ],
  "updatedAt": "2024-01-15T10:40:00Z"
}
```

## Use Case 3: Update Item Quantity

### Request: Update Quantity
```bash
PUT /api/v1/carts/cart_abc123/items/prod_xyz789
Authorization: Bearer {token}
Content-Type: application/json

{
  "quantity": 5
}
```

### Response
```json
{
  "cartId": "cart_abc123",
  "userId": "user123",
  "items": [
    {
      "productId": "prod_xyz789",
      "quantity": 5,
      "price": 49.99
    }
  ],
  "subtotal": 249.95,
  "tax": 19.99,
  "discount": 49.99,
  "total": 219.95,
  "appliedCoupons": [
    {
      "couponId": "SAVE20",
      "type": "percentage",
      "value": 20
    }
  ],
  "updatedAt": "2024-01-15T10:45:00Z"
}
```

## Use Case 4: Remove Item from Cart

### Request: Remove Item
```bash
DELETE /api/v1/carts/cart_abc123/items/prod_xyz789
Authorization: Bearer {token}
```

### Response
```json
{
  "cartId": "cart_abc123",
  "userId": "user123",
  "items": [],
  "subtotal": 0,
  "tax": 0,
  "discount": 0,
  "total": 0,
  "appliedCoupons": [],
  "updatedAt": "2024-01-15T10:50:00Z"
}
```

## Error Examples

### 400 Bad Request - Insufficient Stock
```json
{
  "error": "Insufficient stock",
  "statusCode": 400,
  "details": {
    "requested": 100,
    "available": 25
  }
}
```

### 404 Not Found
```json
{
  "error": "Cart not found",
  "statusCode": 404,
  "cartId": "cart_invalid"
}
```

### 401 Unauthorized
```json
{
  "error": "Unauthorized - Invalid token",
  "statusCode": 401
}
```
