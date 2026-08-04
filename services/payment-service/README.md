# Payment Service

Secure, scalable payment processing service handling all financial transactions with PCI DSS compliance.

## 📋 Overview

The Payment Service is critical for e-commerce success. It handles:

- Payment authorization and capture
- Multiple payment methods (cards, wallets, bank transfers)
- Transaction settlement
- Refunds and disputes
- PCI DSS compliance
- Fraud detection
- Payment reconciliation

## 🏛️ Core Responsibilities

### Payment Processing Pipeline

```
1. Payment Initiation
   ↓ Customer chooses payment method
   ↓
2. Tokenization
   ↓ Card data sent to payment provider (never to our servers)
   ↓
3. Authorization
   ↓ Request authorization from payment network
   ↓
4. Validation
   ↓ Fraud checks, 3D Secure (if required)
   ↓
5. Capture
   ↓ Confirm and settle transaction
   ↓
6. Recording
   ↓ Store transaction record (NO card data)
   ↓
7. Notification
   ↓ Notify order service, send receipts
```

## 💳 Payment Methods Supported

### 1. Credit/Debit Cards

**Flow:**
```
Customer enters card data (client-side)
    ↓
Stripe/PayPal/Adyen tokenizes card
    ↓
Server receives token (not card data)
    ↓
Server sends token to payment processor
    ↓
Processor authorizes charge
    ↓
Customer sees charge on statement
```

**Supported Brands:**
- Visa
- Mastercard
- American Express
- Discover

### 2. Digital Wallets

**Apple Pay**
```
Customer authenticates with Face ID/Touch ID
    ↓
Apple securely sends encrypted token to merchant
    ↓
Token sent to payment processor
    ↓
Processor handles authorization
```

**Google Pay**
```
Similar to Apple Pay, encrypted via Google
```

**PayPal**
```
Customer redirected to PayPal login
    ↓
Customer authorizes transaction on PayPal
    ↓
PayPal sends confirmation back to merchant
    ↓
Merchant completes transaction
```

### 3. Bank Transfers (ACH/Wire)

**ACH (Automated Clearing House) - US**
```
Customer provides bank account info
    ↓
Micro-deposits verify account (2-3 days)
    ↓
Large transaction scheduled (1-2 business days)
```

### 4. Buy Now, Pay Later (BNPL)

**Klarna/Affirm/Afterpay**
```
Customer checks approval at checkout
    ↓
Installment plan offered
    ↓
Merchant paid upfront
    ↓
Customer pays installments to BNPL provider
```

## 📊 Data Model

```javascript
// Transaction Schema
{
  id: "txn_123456",
  orderId: "order_789",
  customerId: "cust_456",
  amount: 99.99,
  currency: "USD",
  status: "completed",  // pending, authorized, captured, failed, refunded
  paymentMethod: "card",
  paymentToken: "tok_visa_4242",  // Never store card number
  cardLast4: "4242",
  cardBrand: "Visa",
  externalTransactionId: "ch_1234567890",  // Payment processor ID
  externalAuthorizationCode: "AUTH123456",
  riskScore: 25,  // 0-100 (higher = riskier)
  fraudStatus: "approved",  // approved, declined, review
  metadata: {
    customerIP: "192.168.1.1",
    userAgent: "Mozilla/5.0...",
    acceptLanguage: "en-US"
  },
  createdAt: "2026-08-04T10:30:00Z",
  authorizedAt: "2026-08-04T10:30:05Z",
  capturedAt: "2026-08-04T10:30:10Z",
  expiresAt: "2026-08-11T10:30:00Z",  // Hold expires after 7 days
  failureReason: null,
  failureCode: null,
  metadata: {
    attempts: 1,
    lastAttemptAt: "2026-08-04T10:30:00Z"
  }
}

// Refund Schema
{
  id: "refund_123",
  transactionId: "txn_123456",
  amount: 25.00,  // Partial refund possible
  reason: "partial_return",  // partial_return, customer_request, fraud, etc.
  status: "completed",  // pending, processing, completed, failed
  externalRefundId: "re_1234567890",
  createdAt: "2026-08-05T14:20:00Z",
  completedAt: "2026-08-06T10:00:00Z"
}
```

## 🔌 API Endpoints

### Create Payment Authorization

```
POST /api/v1/payments/authorize
Authorization: Bearer <token>

Request:
{
  "orderId": "order_789",
  "amount": 99.99,
  "currency": "USD",
  "paymentMethodToken": "tok_visa_4242",
  "billingAddress": {
    "line1": "123 Main St",
    "line2": "Apt 4B",
    "city": "New York",
    "state": "NY",
    "zip": "10001",
    "country": "US"
  },
  "metadata": {
    "userAgent": "Mozilla/5.0..."
  }
}

Response (201 Created):
{
  "transactionId": "txn_123456",
  "status": "authorized",
  "amount": 99.99,
  "externalTransactionId": "ch_1234567890",
  "authorizationCode": "AUTH123456",
  "riskScore": 25,
  "fraudStatus": "approved",
  "expiresAt": "2026-08-11T10:30:00Z"
}

Error Response (400 Bad Request):
{
  "error": "INSUFFICIENT_FUNDS",
  "message": "Card declined: insufficient funds",
  "code": 4000,
  "retryable": true
}
```

### Capture Payment

```
POST /api/v1/payments/:transactionId/capture
Authorization: Bearer <token>

Request:
{
  "amount": 99.99  // Can capture less than authorized (partial capture)
}

Response (200 OK):
{
  "transactionId": "txn_123456",
  "status": "captured",
  "amount": 99.99,
  "capturedAt": "2026-08-04T10:30:10Z"
}
```

### Process Refund

```
POST /api/v1/payments/:transactionId/refunds
Authorization: Bearer <token>

Request:
{
  "amount": 25.00,  // Can refund less than captured (partial refund)
  "reason": "customer_request"
}

Response (201 Created):
{
  "refundId": "refund_123",
  "transactionId": "txn_123456",
  "amount": 25.00,
  "status": "processing",
  "createdAt": "2026-08-05T14:20:00Z"
}
```

### Get Transaction Details

```
GET /api/v1/payments/:transactionId
Authorization: Bearer <token>

Response (200 OK):
{
  "transactionId": "txn_123456",
  "orderId": "order_789",
  "amount": 99.99,
  "status": "captured",
  "paymentMethod": "card",
  "cardLast4": "4242",
  "cardBrand": "Visa",
  "createdAt": "2026-08-04T10:30:00Z",
  "authorizedAt": "2026-08-04T10:30:05Z",
  "capturedAt": "2026-08-04T10:30:10Z"
}
```

## 🔒 Security & Compliance

### PCI DSS Level 1 Compliance

**DO:**
- ✅ Use payment processors (Stripe, PayPal, Adyen)
- ✅ Tokenize card data immediately
- ✅ Never store full card numbers
- ✅ Never store CVV codes
- ✅ Encrypt all data in transit (HTTPS)
- ✅ Log transactions (NO card data)
- ✅ Regular penetration testing

**DON'T:**
- ❌ Accept raw card data on your servers
- ❌ Store card numbers in database
- ❌ Store CVV codes
- ❌ Transmit card data unencrypted
- ❌ Log card numbers

### Tokenization Flow

```
Customer's browser (client-side):
1. Collects card data
2. Sends to Stripe/PayPal JavaScript library
3. Stripe tokenizes: tok_visa_4242
4. Sends token (NOT card) to server

Your server:
1. Receives token
2. Stores token (if recurring payments)
3. Sends token to payment processor
4. Processor handles authorization

Payment processor:
1. Detokenizes (only they have the key)
2. Authorizes with payment network
3. Returns success/failure
```

## 🛡️ Fraud Detection

### Real-Time Fraud Checks

```
Before processing payment:
├── Velocity checks
│   ├── Same card used 5+ times in 1 hour?
│   ├── Same customer 10+ transactions in 1 hour?
│   └── Large amount (>$1000) on new card?
├── Geolocation checks
│   ├── Customer in different country than usual?
│   ├── Impossible travel (2 countries in 1 hour)?
│   └── VPN/Proxy detected?
├── Device checks
│   ├── New device for customer?
│   ├── Device used for other accounts?
│   └── Jailbroken/rooted device?
└── Amount checks
    ├── Unusual amount (much higher than average)?
    ├── Unusual merchant category?
    └── Known fraud patterns?
```

### 3D Secure (Additional Customer Verification)

```
For high-risk transactions:

Payment authorization
    ↓
System detects high risk
    ↓
Initiate 3D Secure verification
    ↓
Customer's bank sends OTP to phone/email
    ↓
Customer enters OTP on bank's website
    ↓
Bank confirms identity
    ↓
Payment authorized with lower fraud liability
```

## 🔄 Payment State Machine

```
PENDING
  ↓
AUTHORIZED (Authorization placed on card)
  ├─→ CAPTURED (Money actually charged)
  │   ├─→ REFUNDED (Full or partial)
  │   └─→ COMPLETED
  ├─→ DECLINED (Authorization failed)
  │   ├─→ RETRY (Customer can retry)
  │   └─→ FAILED
  └─→ EXPIRED (7-day hold expires without capture)

WAITING_FOR_3D_SECURE
  ├─→ AUTHORIZED (After verification)
  └─→ FAILED (Verification failed)
```

## 🏢 Company-Specific Approaches

### Amazon Payment Strategy
- Multiple payment processor redundancy
- Automatic retry logic (3 attempts)
- Real-time fraud AI (trained on billions of transactions)
- 1-Click purchasing
- Cross-border payment optimization

### Stripe Payment Strategy
- Unified API for all payment methods
- Built-in fraud detection (Radar)
- 3D Secure automation
- Regional payment methods support
- Subscription billing

### PayPal Payment Strategy
- Buyer protection
- Seller protection
- Instant settlement option
- Global payment coverage
- Wallet integration

## 💾 Database Schema

```sql
CREATE TABLE transactions (
  id UUID PRIMARY KEY,
  order_id UUID NOT NULL REFERENCES orders(id),
  customer_id UUID NOT NULL REFERENCES users(id),
  amount DECIMAL(12,2) NOT NULL,
  currency VARCHAR(3) NOT NULL,
  status VARCHAR(50) NOT NULL,
  payment_method VARCHAR(50),
  payment_token TEXT,  -- Never full card number
  card_last4 VARCHAR(4),
  card_brand VARCHAR(20),
  external_transaction_id VARCHAR(255),
  external_auth_code VARCHAR(255),
  risk_score INT,
  fraud_status VARCHAR(50),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  authorized_at TIMESTAMP,
  captured_at TIMESTAMP,
  expires_at TIMESTAMP,
  
  INDEX idx_order_id (order_id),
  INDEX idx_customer_id (customer_id),
  INDEX idx_status (status),
  INDEX idx_created_at (created_at)
);

CREATE TABLE refunds (
  id UUID PRIMARY KEY,
  transaction_id UUID NOT NULL REFERENCES transactions(id),
  amount DECIMAL(12,2) NOT NULL,
  reason VARCHAR(100),
  status VARCHAR(50),
  external_refund_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  completed_at TIMESTAMP,
  
  INDEX idx_transaction_id (transaction_id),
  INDEX idx_status (status)
);
```

## 📈 Metrics to Track

```
Conversion metrics:
├── Authorization success rate (target: 98%+)
├── Capture success rate (target: 99%+)
├── Payment method breakdown
└── Currency breakdown

Risk metrics:
├── Fraud detection rate
├── False positive rate (target: <1%)
├── Dispute rate (target: <0.1%)
└── Chargeback rate

Performance metrics:
├── Authorization time (p99: <500ms)
├── Capture time (p99: <1s)
├── Refund processing time (24-48 hours)
└── Settlement time (1-2 days)

Financial metrics:
├── Transaction volume
├── Total payment value
├── Average order value
├── Refund percentage
```

## 🧪 Testing Strategy

- **Unit Tests:** Authorization logic, fraud checks
- **Integration Tests:** Payment processor integration
- **E2E Tests:** Full payment flow with test cards
- **Load Tests:** Peak transaction volume (1000+/sec)
- **Security Tests:** PCI compliance, encryption validation
- **Failure Tests:** Processor downtime, network failures

## 📚 Related Services

- **Order Service** - Creates payment requests
- **Inventory Service** - Confirms after payment
- **Notification Service** - Sends receipts
- **User Service** - Customer payment methods
- **Reporting Service** - Financial analytics

---

**Study These First:**
1. Use-cases (what payments need to do)
2. Scenarios (how top companies handle payments)
3. API specification (endpoint design)
4. Domain models (data structures)
