# Cart Service Use Cases Implementation

## Use Case 1: Add Item to Cart

```typescript
class AddItemToCartUseCase {
  constructor(
    private cartRepository: CartRepository,
    private productService: ProductService,
    private inventoryService: InventoryService,
    private eventBus: EventBus
  ) {}

  async execute(command: AddItemCommand): Promise<Cart> {
    // Validate product exists
    const product = await this.productService.getProduct(command.productId);
    if (!product) {
      throw new ProductNotFoundException(command.productId);
    }

    // Check inventory
    const available = await this.inventoryService.checkStock(
      command.productId,
      command.quantity
    );
    if (!available) {
      throw new InsufficientStockException(
        command.productId,
        command.quantity
      );
    }

    // Get or create cart
    let cart = await this.cartRepository.findById(command.cartId);
    if (!cart) {
      cart = new Cart(command.userId);
    }

    // Add item
    const item = new CartItem(
      command.productId,
      command.quantity,
      product.price
    );
    cart.addItem(item);

    // Save cart
    await this.cartRepository.save(cart);

    // Publish event
    this.eventBus.publish(new ItemAddedEvent(cart.id, command.productId));

    return cart;
  }
}
```

## Use Case 2: Apply Coupon to Cart

```typescript
class ApplyCouponUseCase {
  constructor(
    private cartRepository: CartRepository,
    private couponService: CouponService,
    private eventBus: EventBus
  ) {}

  async execute(command: ApplyCouponCommand): Promise<Cart> {
    // Get cart
    const cart = await this.cartRepository.findById(command.cartId);
    if (!cart) {
      throw new CartNotFoundException(command.cartId);
    }

    // Validate coupon
    const coupon = await this.couponService.validateCoupon(command.couponCode);
    if (!coupon) {
      throw new CouponNotFoundException(command.couponCode);
    }

    // Check coupon eligibility
    const cartValue = this.calculateCartValue(cart);
    if (coupon.minCartValue && cartValue < coupon.minCartValue) {
      throw new CouponEligibilityException(coupon.minCartValue, cartValue);
    }

    // Check if already applied
    if (cart.appliedCoupons.find(c => c.code === coupon.code)) {
      throw new CouponAlreadyAppliedException(coupon.code);
    }

    // Apply coupon
    cart.applyCoupon(coupon);
    await this.cartRepository.save(cart);

    // Publish event
    this.eventBus.publish(new CouponAppliedEvent(
      cart.id,
      coupon.code,
      this.calculateDiscount(cart, coupon)
    ));

    return cart;
  }

  private calculateCartValue(cart: Cart): number {
    return cart.items.reduce((sum, item) => 
      sum + (item.quantity * item.price), 0);
  }

  private calculateDiscount(cart: Cart, coupon: Coupon): number {
    return coupon.calculateDiscount(this.calculateCartValue(cart));
  }
}
```

## Use Case 3: Remove Item from Cart

```typescript
class RemoveItemFromCartUseCase {
  constructor(
    private cartRepository: CartRepository,
    private eventBus: EventBus
  ) {}

  async execute(command: RemoveItemCommand): Promise<Cart> {
    // Get cart
    const cart = await this.cartRepository.findById(command.cartId);
    if (!cart) {
      throw new CartNotFoundException(command.cartId);
    }

    // Remove item
    const itemExists = cart.items.some(
      i => i.productId === command.productId
    );
    if (!itemExists) {
      throw new ItemNotFoundException(command.productId);
    }

    cart.removeItem(command.productId);
    await this.cartRepository.save(cart);

    // Publish event
    this.eventBus.publish(new ItemRemovedEvent(cart.id, command.productId));

    return cart;
  }
}
```

## Use Case 4: Calculate Cart Totals

```typescript
class CalculateCartTotalsUseCase {
  constructor(
    private cartRepository: CartRepository,
    private taxService: TaxService,
    private shippingService: ShippingService
  ) {}

  async execute(command: CalculateTotalsCommand): Promise<CartTotals> {
    // Get cart
    const cart = await this.cartRepository.findById(command.cartId);
    if (!cart) {
      throw new CartNotFoundException(command.cartId);
    }

    // Calculate subtotal
    const subtotal = cart.items.reduce((sum, item) =>
      sum + (item.quantity * item.price), 0);

    // Calculate tax
    const tax = await this.taxService.calculateTax(
      subtotal,
      command.shippingAddress
    );

    // Calculate shipping
    const shipping = await this.shippingService.calculateShipping(
      cart.items,
      command.shippingAddress
    );

    // Calculate discount from coupons
    let discount = 0;
    for (const coupon of cart.appliedCoupons) {
      discount += coupon.calculateDiscount(subtotal);
    }

    // Calculate total
    const total = subtotal + tax + shipping - discount;

    return new CartTotals(
      subtotal,
      tax,
      shipping,
      discount,
      total
    );
  }
}
```

## Use Case 5: Abandon Inactive Carts

```typescript
class AbandonInactiveCartsUseCase {
  constructor(
    private cartRepository: CartRepository,
    private notificationService: NotificationService,
    private eventBus: EventBus
  ) {}

  async execute(): Promise<number> {
    // Find abandoned carts
    const abandonedCarts = await this.cartRepository.findAbandoned(7); // 7 days

    let count = 0;
    for (const cart of abandonedCarts) {
      // Update status
      cart.status = 'abandoned';
      await this.cartRepository.save(cart);

      // Send notification
      await this.notificationService.sendAbandonedCartEmail(cart);

      // Publish event
      this.eventBus.publish(new CartAbandonedEvent(cart.id));

      count++;
    }

    return count;
  }
}
```

## Use Case 6: Clear Cart

```typescript
class ClearCartUseCase {
  constructor(
    private cartRepository: CartRepository,
    private eventBus: EventBus
  ) {}

  async execute(command: ClearCartCommand): Promise<Cart> {
    // Get cart
    const cart = await this.cartRepository.findById(command.cartId);
    if (!cart) {
      throw new CartNotFoundException(command.cartId);
    }

    // Clear items
    const itemCount = cart.items.length;
    cart.clearCart();
    await this.cartRepository.save(cart);

    // Publish event
    this.eventBus.publish(new CartClearedEvent(cart.id, itemCount));

    return cart;
  }
}
```
