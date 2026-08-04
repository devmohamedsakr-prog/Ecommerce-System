# Testing Strategy for E-Commerce Systems

## 📋 Overview

Complete testing pyramid for production e-commerce systems handling 1B+ requests/day.

## 🔺 Testing Pyramid

```
                  ▲
               E2E (5%)
             /        \
            /  Manual  \
           /  Testing  \
          ─────────────
         Integration  (15%)
        /              \
       / Service to    \
      / Service Tests  \
     ─────────────────
    Unit Tests (80%)
   /                 \
  / Logic Tests      \
 / Edge Cases        \
─────────────────────

Time: Unit (fast) → Integration (medium) → E2E (slow)
Coverage: Unit (high) → Integration (medium) → E2E (low)
```

## 🧪 Unit Tests (Fast, High Coverage)

### Payment Calculation Logic

```javascript
// payment.test.js
const { calculateTotal } = require('./payment');

describe('Payment Calculation', () => {
  test('calculates subtotal correctly', () => {
    const items = [
      { price: 10, quantity: 2 },
      { price: 20, quantity: 1 }
    ];
    const subtotal = calculateTotal(items);
    expect(subtotal).toBe(40);  // (10*2) + (20*1) = 40
  });

  test('applies discount correctly', () => {
    const items = [{ price: 100, quantity: 1 }];
    const total = calculateTotal(items, { discountPercent: 10 });
    expect(total).toBe(90);  // 100 - 10% = 90
  });

  test('calculates tax correctly', () => {
    const items = [{ price: 100, quantity: 1 }];
    const total = calculateTotal(items, { taxRate: 0.08 });
    expect(total).toBe(108);  // 100 + 8% tax = 108
  });

  test('handles multiple discounts and tax', () => {
    const items = [{ price: 100, quantity: 1 }];
    const total = calculateTotal(items, {
      discountPercent: 10,
      taxRate: 0.08
    });
    // (100 - 10) + tax = 90 * 1.08 = 97.2
    expect(total).toBeCloseTo(97.2);
  });

  test('rejects negative quantities', () => {
    const items = [{ price: 10, quantity: -1 }];
    expect(() => calculateTotal(items)).toThrow('Invalid quantity');
  });
});

// Run tests
npm test

// Output:
// PASS  ./payment.test.js
//   Payment Calculation
//     ✓ calculates subtotal correctly (2ms)
//     ✓ applies discount correctly (1ms)
//     ✓ calculates tax correctly (1ms)
//     ✓ handles multiple discounts and tax (1ms)
//     ✓ rejects negative quantities (1ms)
//
// Test Suites: 1 passed, 1 total
// Tests: 5 passed, 5 total
// Time: 0.234s
```

### Inventory Management Logic

```javascript
describe('Inventory Management', () => {
  test('reserves stock correctly', () => {
    const inventory = new Inventory();
    inventory.add('prod-123', 100);
    
    const reserved = inventory.reserve('prod-123', 10);
    
    expect(reserved).toBe(true);
    expect(inventory.available('prod-123')).toBe(90);
  });

  test('rejects reservation if insufficient stock', () => {
    const inventory = new Inventory();
    inventory.add('prod-123', 5);
    
    const reserved = inventory.reserve('prod-123', 10);
    
    expect(reserved).toBe(false);
    expect(inventory.available('prod-123')).toBe(5);  // Unchanged
  });

  test('releases reserved stock', () => {
    const inventory = new Inventory();
    inventory.add('prod-123', 100);
    inventory.reserve('prod-123', 10);
    
    inventory.release('prod-123', 10);
    
    expect(inventory.available('prod-123')).toBe(100);
  });

  test('handles concurrent reservations correctly', () => {
    const inventory = new Inventory();
    inventory.add('prod-123', 100);
    
    // Simulate 10 concurrent requests each reserving 15
    const promises = [];
    for (let i = 0; i < 10; i++) {
      promises.push(inventory.reserve('prod-123', 15));
    }
    
    const results = Promise.all(promises);
    // Only first 6 should succeed (6 * 15 = 90 <= 100)
    // 7th and beyond should fail
  });
});
```

## 🔗 Integration Tests (Service to Service)

### Order Service → Inventory Service

```javascript
describe('Order Service Integration', () => {
  beforeEach(async () => {
    // Start services
    await inventoryService.start();
    await orderService.start();
    
    // Seed test data
    await inventoryService.addProduct('prod-123', { quantity: 100 });
  });

  afterEach(async () => {
    await inventoryService.stop();
    await orderService.stop();
  });

  test('reserves inventory when order created', async () => {
    const order = await orderService.createOrder({
      items: [{ productId: 'prod-123', quantity: 10 }]
    });
    
    const inventory = await inventoryService.getInventory('prod-123');
    expect(inventory.available).toBe(90);  // 100 - 10 = 90
  });

  test('releases inventory if order cancelled', async () => {
    const order = await orderService.createOrder({
      items: [{ productId: 'prod-123', quantity: 10 }]
    });
    
    await orderService.cancelOrder(order.id);
    
    const inventory = await inventoryService.getInventory('prod-123');
    expect(inventory.available).toBe(100);  // Back to original
  });

  test('handles inventory service timeout gracefully', async () => {
    await inventoryService.stop();
    
    const order = await orderService.createOrder({
      items: [{ productId: 'prod-123', quantity: 10 }],
      options: { retryTimeout: 1000 }
    });
    
    // Should fail after timeout
    expect(order.error).toBe('Inventory service unavailable');
  });
});
```

### Order Service → Payment Service

```javascript
describe('Order + Payment Integration', () => {
  beforeEach(async () => {
    await paymentService.start();
    await orderService.start();
  });

  test('authorizes payment when order created', async () => {
    const order = await orderService.createOrder({
      items: [{ productId: 'prod-123', quantity: 1, price: 99.99 }],
      paymentToken: 'tok_visa_4242'
    });
    
    expect(order.status).toBe('payment_authorized');
    
    const payment = await paymentService.getTransaction(order.transactionId);
    expect(payment.status).toBe('authorized');
  });

  test('captures payment on order confirmation', async () => {
    const order = await orderService.createOrder({
      items: [{ productId: 'prod-123', quantity: 1, price: 99.99 }],
      paymentToken: 'tok_visa_4242'
    });
    
    await orderService.confirmOrder(order.id);
    
    const payment = await paymentService.getTransaction(order.transactionId);
    expect(payment.status).toBe('captured');
  });

  test('refunds payment if order cancelled', async () => {
    const order = await orderService.createOrder({...});
    await orderService.confirmOrder(order.id);
    
    await orderService.cancelOrder(order.id);
    
    const payment = await paymentService.getTransaction(order.transactionId);
    expect(payment.status).toBe('refunded');
  });
});
```

## 🌐 End-to-End (E2E) Tests

### Complete User Journey

```javascript
// cypress/e2e/checkout.spec.js
describe('Complete Checkout Flow', () => {
  beforeEach(() => {
    cy.visit('http://localhost:3000');
    cy.login('customer@example.com', 'password');
  });

  it('completes purchase from product page to confirmation', () => {
    // Navigate to product
    cy.get('[data-cy=search-input]').type('Blue T-Shirt');
    cy.get('[data-cy=search-button]').click();
    cy.get('[data-cy=product-result]').first().click();
    
    // Verify product details
    cy.get('[data-cy=product-title]').should('contain', 'Blue T-Shirt');
    cy.get('[data-cy=product-price]').should('contain', '$29.99');
    
    // Add to cart
    cy.get('[data-cy=quantity-input]').clear().type('2');
    cy.get('[data-cy=add-to-cart-button]').click();
    cy.get('[data-cy=cart-count]').should('contain', '2');
    
    // Go to cart
    cy.get('[data-cy=cart-icon]').click();
    cy.get('[data-cy=cart-item]').should('have.length', 1);
    
    // Checkout
    cy.get('[data-cy=checkout-button]').click();
    
    // Verify shipping address
    cy.get('[data-cy=shipping-address]').should('be.visible');
    cy.get('[data-cy=shipping-method]').select('standard');
    
    // Enter payment
    cy.get('[data-cy=card-number]').type('4242424242424242');
    cy.get('[data-cy=card-expiry]').type('12/25');
    cy.get('[data-cy=card-cvc]').type('123');
    
    // Place order
    cy.get('[data-cy=place-order-button]').click();
    
    // Verify confirmation
    cy.get('[data-cy=order-confirmation]').should('be.visible');
    cy.get('[data-cy=order-number]').should('match', /^Order #\d+$/);
    cy.get('[data-cy=order-total]').should('contain', '$59.98');
  });

  it('handles payment failure gracefully', () => {
    // Add to cart, go to checkout...
    
    // Enter declined card
    cy.get('[data-cy=card-number]').type('4000000000000002');  // Known declined card
    cy.get('[data-cy=place-order-button]').click();
    
    // Verify error message
    cy.get('[data-cy=error-message]').should('contain', 'Card was declined');
    cy.get('[data-cy=retry-button]').should('be.visible');
  });

  it('allows cart abandonment and recovery', () => {
    // Add to cart and abandon
    cy.addToCart('prod-123');
    cy.reload();  // Simulate leaving
    
    // Check abandoned cart recovery email
    cy.login('admin@example.com', 'password');
    cy.visitEmail('customer@example.com');
    cy.get('[data-cy=email-subject]').should('contain', 'You left something behind');
    
    // Click recovery link
    cy.get('[data-cy=email-link]').click();
    cy.get('[data-cy=cart-item]').should('have.length', 1);
  });
});
```

## 📊 Test Coverage

### Coverage Targets

```
Unit Tests:
├── Line coverage: 80%+
├── Branch coverage: 75%+
├── Function coverage: 80%+
└── Statement coverage: 80%+

Integration Tests:
├── Critical paths: 100%
├── Happy path: 95%+
├── Error paths: 80%+
└── Edge cases: 70%+

E2E Tests:
├── User journeys: 100%
├── Common workflows: 95%+
├── Error scenarios: 50%
└── Localization: 20%+
```

### Coverage Report

```bash
npm run test:coverage

# Output:
# ─────────────────────────────────────
# │ File        │ % Stmts │ % Branch │
# ─────────────────────────────────────
# │ All files   │  82.5   │  78.2    │
# │ payment.js  │  95.3   │  92.1    │
# │ order.js    │  88.2   │  85.5    │
# │ inventory.js│  75.3   │  70.1    │  ← Below target!
# └──────────────────────────────────────

# Action: Improve inventory.js tests
```

## ⚡ Performance Testing

### Load Test with k6

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp-up
    { duration: '5m', target: 100 },   // Sustain
    { duration: '2m', target: 0 },     // Ramp-down
  ],
};

export default function() {
  // Get product
  const res1 = http.get('http://localhost:3000/api/products/123');
  check(res1, {
    'product status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });

  // Add to cart
  const res2 = http.post('http://localhost:3000/api/cart/add', {
    productId: '123',
    quantity: 1
  });
  check(res2, {
    'add cart status is 200': (r) => r.status === 200,
  });

  sleep(1);
}

// Run test:
k6 run load-test.js

// Results:
// execution: local
// scenarios: (1 xall default)
// samples.................: 2100
// http_reqs..................: 2100 27.3/s
// http_req_duration........: avg=85ms min=12ms med=75ms max=450ms p(95)=156ms p(99)=298ms
// http_req_failed.........: 0.00%
```

## ✅ Test Automation

### Pre-Commit Hooks

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "Running unit tests..."
npm run test:unit
if [ $? -ne 0 ]; then
  echo "❌ Unit tests failed"
  exit 1
fi

echo "Running linter..."
npm run lint
if [ $? -ne 0 ]; then
  echo "❌ Linting failed"
  exit 1
fi

echo "✅ All checks passed"
```

### CI/CD Pipeline

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Node
        uses: actions/setup-node@v2
        with:
          node-version: '16'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Run integration tests
        run: npm run test:integration
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload coverage
        run: npm run test:coverage
      
      - name: Comment coverage on PR
        uses: romeovs/lcov-reporter-action@v0.3.1
        if: github.event_name == 'pull_request'
```

## 🎯 Testing Checklist

```
Before Launch:
- [ ] Unit test coverage > 80%
- [ ] Integration tests for all critical paths
- [ ] E2E tests for happy path
- [ ] Load testing at 2x expected peak
- [ ] Security testing (OWASP Top 10)
- [ ] Payment testing with test cards
- [ ] Email testing (deliverability)
- [ ] Mobile testing (responsive)
- [ ] Browser testing (all supported)
- [ ] Performance testing (p99 latency)
- [ ] Failure scenario testing
- [ ] Chaos engineering testing

Ongoing:
- [ ] Regression testing before releases
- [ ] New features tested before merge
- [ ] Performance benchmarking
- [ ] Synthetic monitoring
- [ ] User acceptance testing
```

---

**Key Rule:** "Write tests as documentation. Good tests tell the story of how the system works."
