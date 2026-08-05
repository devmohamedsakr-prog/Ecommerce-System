# Cart Service Infrastructure Examples

## Example 1: Database Initialization and Cart Persistence

### MongoDB Connection
```typescript
// Setup
const db = new MongoDatabase(process.env.MONGODB_URI);
await db.connect();

// Create cart
const cartData = {
  _id: 'cart_001',
  userId: 'user_john',
  items: [
    { productId: 'prod_001', quantity: 2, price: 49.99 }
  ],
  appliedCoupons: [],
  status: 'active'
};

await CartModel.insertOne(cartData);

// Query cart
const cart = await CartModel.findById('cart_001');
// Result: Cart document retrieved
```

## Example 2: Redis Caching

### Store and Retrieve
```typescript
const cacheService = new CacheService();

// Store cart
const cart = new Cart('user_jane');
await cacheService.setCart('cart_002', cart);

// Retrieve from cache
const cached = await cacheService.getCart('cart_002');
// Result: Cart returned from Redis (1 hour TTL)

// Invalidate on update
await cacheService.invalidateCart('cart_002');
```

## Example 3: Event Publishing

### Publish ItemAdded Event
```typescript
const eventBus = new EventBus();
await eventBus.initialize();

const event = new ItemAddedEvent({
  cartId: 'cart_001',
  productId: 'prod_001',
  quantity: 2,
  timestamp: new Date()
});

await eventBus.publish(event);

// Subscribe to event
await eventBus.subscribe('ItemAdded', async (event) => {
  console.log('Item added to cart:', event.cartId);
  // Trigger downstream processes
});
```

## Example 4: External Service Integration

### Fetch Product Information
```typescript
const productClient = new ProductServiceClient();

// Get product details
const product = await productClient.getProduct('prod_123');
// Result:
// {
//   id: 'prod_123',
//   name: 'Laptop',
//   price: 999.99,
//   description: '...'
// }

// Check inventory
const isAvailable = await productClient.checkStock('prod_123', 5);
// Result: true (5 units available)
```

## Example 5: Circuit Breaker Pattern

### Handling Service Failures
```typescript
class ResilientProductServiceClient {
  private circuitBreaker: CircuitBreaker;

  constructor() {
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,
      resetTimeout: 60000
    });
  }

  async getProduct(productId: string): Promise<Product> {
    try {
      return await this.circuitBreaker.execute(() =>
        this.httpClient.get(`/products/${productId}`)
      );
    } catch (error) {
      if (error instanceof CircuitBreakerOpenError) {
        throw new ProductServiceUnavailableException();
      }
      throw error;
    }
  }
}

// Usage
const client = new ResilientProductServiceClient();
try {
  const product = await client.getProduct('prod_123');
} catch (error) {
  console.log('Product service temporarily unavailable');
  // Use fallback or cached data
}
```

## Example 6: Dependency Injection Configuration

### Container Setup
```typescript
// Create container
const container = new Container();

// Register dependencies
container.register('mongoDatabase', () => {
  const db = new MongoDatabase(process.env.MONGODB_URI);
  db.connect();
  return db;
});

container.register('redisCache', () => {
  return new CacheService();
});

container.register('cartRepository', () => {
  return new CartRepository(
    container.resolve('mongoDatabase')
  );
});

container.register('eventBus', () => {
  return new EventBus(process.env.RABBITMQ_URL);
});

container.register('addItemToCartUseCase', () => {
  return new AddItemToCartUseCase(
    container.resolve('cartRepository'),
    new ProductServiceClient(),
    container.resolve('eventBus')
  );
});

// Resolve use case
const useCase = container.resolve('addItemToCartUseCase');
const result = await useCase.execute(command);
```

## Example 7: Connection Pool Management

### Database Connection Pool
```typescript
const pool = new ConnectionPool({
  min: 5,
  max: 20,
  acquireTimeoutMillis: 30000,
  idleTimeoutMillis: 30000
});

// Acquire connection
const connection = await pool.acquire();

try {
  const carts = await connection.query('SELECT * FROM carts WHERE userId = ?', [userId]);
  return carts;
} finally {
  await pool.release(connection);
}
```

## Example 8: Data Migration

### Schema Migration
```typescript
class Migration_AddCustomizationsField {
  async up(): Promise<void> {
    await CartModel.updateMany(
      { customizations: { $exists: false } },
      { $set: { customizations: {} } }
    );
  }

  async down(): Promise<void> {
    await CartModel.updateMany(
      {},
      { $unset: { customizations: 1 } }
    );
  }
}

// Run migration
const migration = new Migration_AddCustomizationsField();
await migration.up();
```
