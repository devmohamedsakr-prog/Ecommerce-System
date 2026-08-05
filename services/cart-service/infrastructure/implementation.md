# Cart Service Infrastructure Implementation

## Database Configuration

### MongoDB Connection
```typescript
import mongoose from 'mongoose';

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGODB_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('Connected to MongoDB');
  } catch (error) {
    console.error('MongoDB connection failed:', error);
    process.exit(1);
  }
};
```

### Cart Schema
```typescript
const cartSchema = new mongoose.Schema({
  _id: String,
  userId: { type: String, required: true, index: true },
  items: [{
    productId: String,
    quantity: Number,
    price: Number,
    customizations: mongoose.Schema.Types.Mixed
  }],
  appliedCoupons: [{
    code: String,
    type: String,
    value: Number
  }],
  status: { type: String, enum: ['active', 'abandoned', 'converted'] },
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
  expiresAt: { type: Date, index: { expireAfterSeconds: 0 } }
});

cartSchema.index({ userId: 1, status: 1 });
cartSchema.index({ updatedAt: 1 });

export const CartModel = mongoose.model('Cart', cartSchema);
```

## Repository Implementation

```typescript
class CartRepository {
  async findById(cartId: string): Promise<Cart | null> {
    const doc = await CartModel.findById(cartId);
    return doc ? this.toDomain(doc) : null;
  }

  async save(cart: Cart): Promise<void> {
    const doc = this.toPersistence(cart);
    await CartModel.updateOne(
      { _id: cart.id },
      doc,
      { upsert: true }
    );
  }

  async findByUserId(userId: string): Promise<Cart[]> {
    const docs = await CartModel.find({ userId, status: 'active' });
    return docs.map(doc => this.toDomain(doc));
  }

  async findAbandoned(days: number): Promise<Cart[]> {
    const cutoffDate = new Date(Date.now() - days * 24 * 60 * 60 * 1000);
    const docs = await CartModel.find({
      updatedAt: { $lt: cutoffDate },
      status: 'active'
    });
    return docs.map(doc => this.toDomain(doc));
  }

  private toDomain(doc: any): Cart {
    // Convert document to domain entity
  }

  private toPersistence(cart: Cart): any {
    // Convert domain entity to document
  }
}
```

## Redis Cache Implementation

```typescript
import redis from 'redis';

const redisClient = redis.createClient({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  db: 1
});

class CacheService {
  async getCart(cartId: string): Promise<Cart | null> {
    const cached = await redisClient.get(`cart:${cartId}`);
    return cached ? JSON.parse(cached) : null;
  }

  async setCart(cartId: string, cart: Cart): Promise<void> {
    await redisClient.setex(
      `cart:${cartId}`,
      3600, // 1 hour TTL
      JSON.stringify(cart)
    );
  }

  async invalidateCart(cartId: string): Promise<void> {
    await redisClient.del(`cart:${cartId}`);
  }

  async invalidateUserCarts(userId: string): Promise<void> {
    const keys = await redisClient.keys(`cart:*`);
    // Filter and delete carts for user
  }
}
```

## Event Bus Implementation

```typescript
import amqp from 'amqplib';

class EventBus {
  private connection: amqp.Connection;
  private channel: amqp.Channel;

  async initialize(): Promise<void> {
    this.connection = await amqp.connect(process.env.RABBITMQ_URL);
    this.channel = await this.connection.createChannel();
    
    // Declare exchanges
    await this.channel.assertExchange('cart-events', 'topic', { durable: true });
  }

  async publish(event: DomainEvent): Promise<void> {
    const message = JSON.stringify(event);
    this.channel.publish(
      'cart-events',
      event.eventType,
      Buffer.from(message),
      { persistent: true }
    );
  }

  async subscribe(eventType: string, handler: Function): Promise<void> {
    const queue = await this.channel.assertQueue('', { exclusive: true });
    await this.channel.bindQueue(queue.queue, 'cart-events', eventType);
    
    this.channel.consume(queue.queue, async (msg) => {
      if (msg) {
        const event = JSON.parse(msg.content.toString());
        await handler(event);
        this.channel.ack(msg);
      }
    });
  }
}
```

## External Service Integration

```typescript
class ProductServiceClient {
  private httpClient: AxiosInstance;

  constructor() {
    this.httpClient = axios.create({
      baseURL: process.env.PRODUCT_SERVICE_URL,
      timeout: 5000
    });
  }

  async getProduct(productId: string): Promise<Product | null> {
    try {
      const response = await this.httpClient.get(`/products/${productId}`);
      return response.data;
    } catch (error) {
      throw new ProductServiceException('Failed to fetch product');
    }
  }

  async checkStock(productId: string, quantity: number): Promise<boolean> {
    try {
      const response = await this.httpClient.post('/inventory/check', {
        productId,
        quantity
      });
      return response.data.available;
    } catch (error) {
      throw new InventoryServiceException('Failed to check stock');
    }
  }
}
```

## Dependency Injection

```typescript
// IoC Container
class Container {
  register(key: string, factory: Function): void {
    this.services.set(key, factory);
  }

  resolve(key: string): any {
    const factory = this.services.get(key);
    return factory();
  }
}

// Configuration
const container = new Container();

container.register('cartRepository', () => new CartRepository());
container.register('cacheService', () => new CacheService());
container.register('eventBus', () => new EventBus());
container.register('productServiceClient', () => new ProductServiceClient());

container.register('addItemToCartUseCase', () => 
  new AddItemToCartUseCase(
    container.resolve('cartRepository'),
    container.resolve('productServiceClient'),
    container.resolve('eventBus')
  )
);
```
