# Rakuten: Ecosystem Integration Architecture

## Single ID Model
- One Rakuten account across all services
- Single login (federated auth)
- Unified preferences and settings
- Shared payment methods

## Core Services Connected
1. **Rakuten Ichiba** (Marketplace) → Shopping
2. **Rakuten Bank** (Financial) → Account/loans
3. **Rakuten Card** (Credit) → Payments
4. **Rakuten Insurance** → Protection
5. **Rakuten Travel** → Bookings
6. **Rakuten Mobile** → Telecom
7. **Rakuten Securities** → Investments
8. **Rakuten Super Logistics** → Shipping

## Rakuten Points: Central Currency
- Earn across all services (5-15% earn rate)
- Accumulate in central account
- Redeem across all services
- Transfer: Between family members
- Expiration: 12 months (encourages use)

## Cross-Service Data Sharing
- Purchase history: All services visible
- Preferences: Personalization across services
- Payment method: Shared across ecosystem
- Identity verification: One-time across all

## Lock-In Mechanism
- Points accumulation: Customer invests time building balance
- Family members: Share points (family lock-in)
- Card rewards: Higher earn rate in ecosystem
- Switching cost: Lose points if leave
- Result: 90%+ retention (vs 70% single service)

## Infrastructure Integration
- Single auth service (central)
- Central points database
- Central customer profile
- Per-service databases (separate)
- Event bus: Service-to-service communication
- Result: Seamless experience, high switching cost

## Financial Impact
- Average customer LTV: 5x higher than single service
- Wallet share: 40-60% of spending in ecosystem
- Profitability: Points cost < customer LTV gain
- Result: Ecosystem model highly profitable

