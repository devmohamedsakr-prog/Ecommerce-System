# Cart Service Infrastructure

## Overview
Infrastructure layer handles data persistence, external service integrations, and technical implementation details.

## Technology Stack
- Database: MongoDB
- Cache: Redis
- Message Queue: RabbitMQ
- ORM: Mongoose
- HTTP Client: Axios

## Components

### Repository Pattern
- CartRepository: Handles cart persistence
- CartItemRepository: Manages cart items
- CouponRepository: Coupon data access

### External Service Integrations
- ProductService: Product information
- InventoryService: Stock management
- TaxService: Tax calculations
- ShippingService: Shipping costs
- NotificationService: Email/SMS notifications

### Event Publishing
- EventBus: Publishes domain events
- RabbitMQ configuration
- Event handlers

### Caching Strategy
- Redis for cart data (TTL: 1 hour)
- Cache invalidation on updates
- Cache warming strategies
