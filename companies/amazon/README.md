# Amazon - E-Commerce Architecture

## 🏢 Company Profile

**Amazon** is the world's largest e-commerce company by revenue. Key facts:

- **Revenue**: $575B+ annually
- **GMV**: $200B+
- **Daily Orders**: 7-10 million
- **Regions**: 24+ countries
- **Fulfillment Centers**: 500+ globally
- **AWS Availability**: 30+ regions

## 🏗️ Architecture Overview

Amazon's infrastructure is built on principles of:
- Extreme scalability
- Fault tolerance and resilience
- Microsecond-level latency optimization
- Global distribution
- AWS cloud-native

## 📁 Folder Structure

```
amazon/
├── architecture/              # System design documents
│   ├── order-processing.md   # How Amazon processes orders
│   ├── fulfillment.md        # FC network and logistics
│   ├── payment.md            # Payment processing at scale
│   ├── catalog.md            # Product catalog (500M+)
│   └── infrastructure.md      # AWS infrastructure
├── workflows/                 # Business process flows
│   ├── prime-workflow.md     # Prime member experience
│   ├── marketplace-flow.md   # 3rd party seller flow
│   ├── return-process.md     # Return & refund handling
│   └── flash-deals.md        # Lightning deals & sales
├── case-studies/              # Real implementation examples
│   ├── prime-day.md          # Handling Black Friday/Prime Day
│   ├── prime-same-day.md     # Same-day delivery
│   ├── alexa-integration.md  # Voice shopping
│   └── aws-benefits.md       # Leveraging AWS advantages
├── challenges/                # Technical challenges solved
│   ├── inventory-management.md
│   ├── cart-abandonment.md
│   ├── fraud-prevention.md
│   ├── international-expansion.md
│   └── vendor-management.md
└── scalability-notes/         # How Amazon scales
    ├── billion-qps.md        # Handling 1B+ requests/second
    ├── global-distribution.md # Multi-region architecture
    ├── machine-learning.md   # Personalization at scale
    └── infrastructure-investment.md
```

## 🎯 Key Characteristics

### 1. Two-Sided Marketplace

Amazon operates two key marketplaces:
- **1P (First-Party)** - Amazon's own inventory
- **3P (Third-Party)** - Seller marketplace

This requires:
- Separate fulfillment paths
- Different commission models
- Vendor management systems
- Competitive pricing algorithms

### 2. Fulfillment by Amazon (FBA)

Amazon's logistics:
- 500+ fulfillment centers
- Multi-tier fulfillment (standard/expedited/prime)
- Reverse logistics for returns
- Last-mile delivery optimization

### 3. Prime Ecosystem

Premium subscription model:
- 200M+ members globally
- Same-day/next-day delivery
- Free shipping
- Additional benefits (Video, Music, etc.)
- Loyalty driver

### 4. Global Scale

Operating across multiple regions requires:
- Region-specific catalog
- Currency and tax handling
- Local payment methods
- Language localization
- Regulatory compliance

## 🏛️ Architecture Patterns Used

### Microservices

```
┌─────────────────────────────────────────┐
│         API Gateway / Load Balancer      │
└─────────────────────────────────────────┘
         ↓ Routes to services
┌──────────────────────────────────────────────────────────┐
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Order     │  │   Payment   │  │ Inventory   │      │
│  │   Service   │  │   Service   │  │ Service     │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │  Shipping   │  │  Catalog    │  │  User       │      │
│  │  Service    │  │  Service    │  │  Service    │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

### Event-Driven Architecture

Events drive state changes:
- OrderCreated
- PaymentAuthorized
- InventoryReserved
- ShipmentDispatched
- DeliveryConfirmed

Events → EventBridge/SNS/SQS → Processing

### Data Layer Strategy

- **DynamoDB** for high-frequency access (orders, carts)
- **RDS Aurora** for transactional data
- **Redshift** for analytics
- **S3** for unstructured data (images, reviews)
- **ElastiCache** for caching

## 💾 Database Strategy

### DynamoDB (Key-Value Store)

Used for:
- Active orders (fast lookups)
- Shopping carts
- User sessions
- Product pages (cache)

Pattern:
```
Table: Orders
Partition Key: CustomerID
Sort Key: OrderTimestamp
GSI: OrderStatus

Table: ShoppingCarts
Partition Key: CartID
TTL: 30 days
```

### RDS Aurora (Relational)

Used for:
- User accounts
- Product catalog
- Prices and inventory
- Reports and analytics

```
Schema:
- Users
- Products
- Orders (Order history)
- Order_Items
- Inventory
- Vendors
```

### Redshift (Data Warehouse)

Used for:
- Historical analytics
- Business intelligence
- Recommendations
- Trending analysis

## 🔄 Order Processing Flow

```
1. Customer Places Order
   ↓
2. Order Service validates
   - Items exist in catalog
   - Inventory available
   - Customer valid
   ↓
3. Payment Service authorizes
   - Credit card / payment method validated
   - Amount authorized
   ↓
4. Inventory Service reserves
   - Items marked as reserved
   - Expiration set (15 minutes)
   ↓
5. Notification Service
   - Confirmation email sent
   - SMS if subscribed
   ↓
6. Fulfillment Service
   - Route to closest FC
   - Pick list generated
   ↓
7. Shipping Service
   - Carrier assigned
   - Label printed
   - Item scanned
   ↓
8. Customer Notified
   - Tracking number
   - Estimated delivery
   ↓
9. Delivery & Completion
   - Item delivered
   - Order marked complete
   - Review request
```

## 🌍 Global Distribution

Amazon operates in 24+ countries with:

```
Region Strategy:
├─ North America (US, Canada, Mexico)
├─ Europe (UK, Germany, France, Spain, Italy)
├─ Asia-Pacific (Japan, Australia, Singapore, India)
├─ China (Special deal with local partner)
└─ Latin America (Brazil)

Each region has:
- Local fulfillment centers
- Regional database replica
- Local payment methods
- Currency handling
- Tax compliance
- Seller fulfillment center
```

## 📊 Scale Numbers

### Traffic
- 1 billion+ requests/second peak
- 7-10 million orders/day
- 500 million+ product pages viewed daily

### Data
- 300+ million products indexed
- 200+ million customer profiles
- Petabytes of historical data
- Real-time search with sub-100ms latency

### Infrastructure
- 30+ AWS regions
- 500+ fulfillment centers
- 5+ million sellers
- 1 billion+ daily active users

## 🔒 Security & Compliance

- **PCI DSS Level 1** - Highest payment security
- **GDPR** - EU data protection
- **Fraud Detection** - ML-based real-time detection
- **Encryption** - TLS in transit, AES at rest
- **Access Control** - IAM, role-based access
- **Audit Logging** - All operations logged

## 🎯 Key Innovations

### 1. Personalization
- Recommendation Engine - ML-based suggestions
- Search Relevance - ML ranking
- Dynamic Pricing - Supply/demand pricing

### 2. Logistics
- Prime Same-Day/Next-Day - Logistic optimization
- FBA - Seller fulfillment service
- AWS Logistics - Owned delivery fleet

### 3. Customer Experience
- 1-Click Checkout - UX simplification
- Alexa Integration - Voice shopping
- A/B Testing - Continuous optimization

### 4. Marketplace
- Seller Central - Vendor management
- Vendor Management System - Supply chain
- Sponsored Products - Advertising platform

## 🚀 Performance Optimization

- **CDN**: CloudFront for global content delivery
- **Caching**: Multiple caching layers (edge, application, database)
- **Compression**: GZIP for API responses
- **Async Processing**: Non-blocking operations
- **Batch Operations**: Bulk inventory updates

## 💡 Lessons for Your System

### For Startups (Day 1-6 months)
1. Start with monolith on AWS
2. Use managed services (RDS, SQS)
3. Focus on customer experience
4. Implement basics of tracking

### For Scale-ups (6 months - 2 years)
1. Migrate to microservices
2. Implement event-driven architecture
3. Multi-region for resilience
4. Advanced analytics

### For Enterprise (2+ years)
1. Global distribution
2. Custom infrastructure
3. ML-based personalization
4. Proprietary logistics

## 📚 Key Technologies

- **Compute**: EC2, Lambda, ECS
- **Storage**: S3, DynamoDB, RDS, Redshift
- **Messaging**: SQS, SNS, EventBridge
- **Analytics**: Kinesis, Redshift
- **Monitoring**: CloudWatch, X-Ray
- **CI/CD**: CodePipeline, CodeDeploy

---

**Study This For:**
- Building billion-scale systems
- Understanding marketplace models
- Global distribution strategies
- Logistics integration
- Customer experience at scale
