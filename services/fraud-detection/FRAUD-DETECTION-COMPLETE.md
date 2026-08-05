# Fraud Detection Service - Complete Implementation Guide

**Scale:** 1M+ transactions/day | <100ms decision latency | 99.7% legitimate rate

---

## API Specification

```
POST /v1/fraud/score
  Score transaction risk
  Body: { userId, amount, payment_method, device_id, ip_address, order_items[], shipping_address, billing_address }
  Response: { score: 0-100, riskLevel: LOW|MEDIUM|HIGH|CRITICAL, recommendation: APPROVE|REVIEW|DECLINE }

POST /v1/fraud/verify
  Verify transaction (after decision)
  Body: { transactionId, approved, chargebackRaised }
  Response: { success, feedbackId }

GET /v1/fraud/rules
  Get active fraud rules
  Response: [{ ruleId, name, condition, action, enabled }]

POST /v1/fraud/report
  Report suspicious activity
  Body: { userId, transactionId, reason }
  Response: { reportId, escalated }
```

---

## Domain Models

```
Transaction {
  transactionId: UUID
  userId: UUID
  orderId: UUID
  amount: Money
  currency: String
  paymentMethod: PaymentMethod (credit_card, debit_card, wallet, bank_transfer)
  device: DeviceInfo { deviceId, os, browser, fingerprint }
  ipAddress: String
  geolocation: { country, state, city, lat, long }
  orderItems: OrderItem[]
  shippingAddress: Address
  billingAddress: Address
  velocity: VelocityData { transactionsIn1h, transactionsIn24h, amountIn24h }
  timestamp: DateTime
}

FraudScore {
  score: Integer (0-100)
  riskLevel: RiskLevel (LOW, MEDIUM, HIGH, CRITICAL)
  factors: FraudFactor[] ({ type, weight, value })
  recommendation: Recommendation (APPROVE, REVIEW, DECLINE)
  confidence: Decimal (0-1)
}

FraudFactor {
  type: String (velocity_too_high, address_mismatch, new_device, etc.)
  weight: Integer (0-100)
  value: Decimal
}

RiskLevel = LOW (0-20) | MEDIUM (21-50) | HIGH (51-80) | CRITICAL (81-100)
```

### Business Rules
1. **Score calculation:** ML model (1000+ features)
2. **Velocity checks:** Limit per hour/day
3. **Address mismatch:** Flag if billing != shipping
4. **Device fingerprinting:** Track by browser + OS combo
5. **Chargeback tracking:** 1 chargeback = higher friction
6. **VIP protection:** Trusted customers = lower scrutiny

---

## Use Cases

### UC-001: Real-time Fraud Scoring
**Trigger:** Payment initiated  
**Flow:**
1. Order Service sends transaction details
2. Fraud Service calls ML model
3. Analyze 1000+ features in <50ms
4. Return score + recommendation
5. Decision made: APPROVE/REVIEW/DECLINE

### UC-002: Velocity Checking
**Rules:**
- Max 5 transactions per hour per user
- Max $5K per day per user
- Max 10 new customers per IP per hour
- Escalate if exceeded

### UC-003: Device Fingerprinting
**Tracking:**
- Device ID (hardware + browser)
- OS + browser version
- Canvas fingerprinting (JavaScript)
- WebGL fingerprinting (GPU)
- Result: High-confidence device tracking

### UC-004: Address Verification
**Checks:**
- Billing vs shipping (flag if different)
- Known fraud addresses (blacklist)
- Address validation (USPS, Google Maps)
- Country mismatch with card

### UC-005: Chargeback Learning
**Flow:**
1. Customer disputes transaction
2. Chargeback raised (60 days later)
3. System records: transactionId, reason
4. Update ML model (weight transaction as fraud)
5. Future similar transactions: higher scrutiny

---

## Company Scenarios

### Amazon Fraud Detection
```
ML Model:
- Trained on 10B+ historical transactions
- XGBoost + neural networks
- 1000+ features
- Real-time scoring <20ms

Scoring factors:
- Velocity: Weight 15%
- Device: Weight 15%
- Address: Weight 10%
- Card: Weight 20%
- Patterns: Weight 20%
- User history: Weight 20%

Thresholds:
- Score <10: Auto-approve
- Score 10-30: Monitor
- Score 30-70: Request additional verification
- Score >70: Decline or call customer

Accuracy:
- True positive rate (catch fraud): 95%
- False positive rate (block legit): <1%
```

### eBay Fraud Prevention
```
Strategy: Seller + Buyer protection

Seller fraud:
- Account takeover: Detect unusual item posts
- Unauthorized selling: Same item multiple listings
- Shill bidding: Fake bids to inflate prices

Buyer fraud:
- Chargeback abuse: Repeat offenders
- Item Not Received (INR): Pattern detection
- Return fraud: Sending back different item

Managed Payments:
- 21-day hold on seller funds
- Automatic chargeback protection
- Seller accountability
```

### Shopify Fraud Detection
```
For merchants (especially new):
- Small average transaction: Lower score threshold
- First-time purchase: Require verification
- High-risk countries: Geo-based rules
- Bulk orders: Flag for manual review

Features:
- Shopify Fraud Analysis (free)
- Machine learning: Trained on all merchants
- Customizable rules per merchant
- Integration: Shopify Payments

Accuracy:
- 96% legitimate transactions
- 3% false positives (acceptable for SMEs)
- 1% actual fraud blocked
```

---

## ML Model Details

### Features (1000+)

**Velocity (100 features):**
- Transactions in 1h, 3h, 6h, 24h
- Amount in 1h, 3h, 6h, 24h
- Distinct cards used
- Distinct IPs used
- Distinct devices used

**User History (100 features):**
- Account age (days)
- Total transactions
- Average transaction value
- Success rate (%)
- Chargebacks (count)
- Returns (rate)

**Payment Method (50 features):**
- Card type (credit, debit, prepaid)
- Card issuer (bank)
- Card age (months)
- Card used before (yes/no)
- AVS match
- CVV match

**Device (100 features):**
- Device fingerprint
- OS version
- Browser version
- Screen resolution
- Canvas fingerprinting
- WebGL fingerprint

**Address (50 features):**
- Billing = shipping (yes/no)
- Known fraud address (yes/no)
- Address validation (valid, risky, invalid)
- City population
- Country risk score

**Behavioral (100 features):**
- Time of day (pattern)
- Device to country (mismatch)
- Card to country (mismatch)
- Shipping address to user location
- Mouse movements
- Typing speed

---

## Infrastructure

### ML Pipeline
```
Training:
- Data: 5 years of transactions
- Features: 1000+ engineered
- Algorithm: XGBoost + neural net ensemble
- Training time: 4 hours weekly
- Validation: 10% holdout set

Serving:
- Model: Deployed to microservice
- Framework: TensorFlow Serving
- Latency: <50ms p99
- Replicas: 10 (for redundancy)
- Cache: Last 1K scores (same input = same output)
```

### Real-time Data Processing
```
Event Stream (Kafka):
- Topic: transactions
- Partitions: 100 (by userId)
- Retention: 30 days

Processing:
- Velocity checks: Real-time windowing
- Device fingerprinting: Real-time database
- ML scoring: Inline, <50ms
- Results: Published to order-service
```

### Database Schema
```sql
CREATE TABLE fraud_scores (
  transaction_id UUID PRIMARY KEY,
  user_id UUID,
  score INTEGER,
  risk_level VARCHAR(20),
  recommendation VARCHAR(20),
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

CREATE TABLE chargebacks (
  chargeback_id UUID PRIMARY KEY,
  transaction_id UUID REFERENCES fraud_scores,
  user_id UUID,
  reason VARCHAR(255),
  raised_at TIMESTAMP,
  resolved_at TIMESTAMP,
  status VARCHAR(20)
);

CREATE TABLE fraud_rules (
  rule_id UUID PRIMARY KEY,
  name VARCHAR(255),
  condition JSON,
  action VARCHAR(50),
  enabled BOOLEAN
);

CREATE INDEX idx_user_id ON fraud_scores(user_id);
CREATE INDEX idx_score ON fraud_scores(score);
```

---

## Testing

### Unit Tests
- Velocity check: Transactions in 1h
- Address mismatch: Billing vs shipping
- Device fingerprinting: Consistent across sessions
- Chargeback learning: Model updated correctly

### Integration Tests
- Full scoring flow: Input → ML model → Score
- Verification: Feedback recorded for model retraining
- Chargeback handling: Correctly labeled for future scoring

### Load Tests
- 100K score requests/second
- <50ms latency p99
- ML model serving: 10K predictions/second per replica
- Zero failed predictions (graceful degradation)

---

## Monitoring

**Key Metrics:**
- Fraud detection rate (%)
- False positive rate (% of legit blocked)
- Chargeback rate (per 1000 transactions)
- ML model accuracy (%)
- Scoring latency (ms p50/p95/p99)
- Rule engine response time (ms)

---

