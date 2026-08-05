# Subscription Management & Recurring Billing Service

**Status:** Enterprise Service | **Priority:** HIGH | **Compliance:** PCI DSS, Fair Billing, GDPR

---

## 📋 Overview

Subscription Management Service handles recurring billing, subscription plans, payment retries (dunning), churn prevention, and subscription lifecycle management. Enables predictable recurring revenue models while reducing involuntary churn through intelligent payment recovery.

## 🎯 Business Problem

- Subscription revenue: **40-60% of DTC brand revenue**
- Involuntary churn from failed payments: **20-40% of cancellations**
- Payment failure recovery (dunning): **5-8% revenue recovery potential**
- Churn prediction critical for LTV optimization
- Complex billing logic (proration, plan changes, cancellations)

## 🏗️ Architecture

### Core Components

```
┌──────────────────────────────────────────┐
│  Subscription Management & Billing       │
├──────────────────────────────────────────┤
│                                           │
│  ┌──────────────┐  ┌──────────────┐    │
│  │ Subscription │  │ Recurring    │    │
│  │ Plans &      │  │ Billing &    │    │
│  │ Enrollment   │  │ Invoicing    │    │
│  └──────────────┘  └──────────────┘    │
│                                           │
│  ┌──────────────┐  ┌──────────────┐    │
│  │ Payment      │  │ Dunning &    │    │
│  │ Management   │  │ Retry Logic  │    │
│  └──────────────┘  └──────────────┘    │
│                                           │
│  ┌──────────────┐  ┌──────────────┐    │
│  │ Churn        │  │ Plan Changes │    │
│  │ Prediction   │  │ & Upgrades   │    │
│  └──────────────┘  └──────────────┘    │
│                                           │
└──────────────────────────────────────────┘
       ↓
   Payment Service (payment processing)
   User Service (customer data)
   Notification Service (communications)
   Analytics Service (metrics)
```

### Data Model

```
SUBSCRIPTION_PLAN
├── plan_id (UUID)
├── plan_name (string)
├── plan_type (enum: monthly, quarterly, annual, usage-based)
├── base_price (decimal)
├── billing_interval_days (number)
├── trial_period_days (number: 0-30)
├── trial_price (decimal, nullable)
├── currency (string)
├── features (array: string)
├── included_credits (object: type, amount)
├── status (enum: active, archived, beta)
├── created_date (timestamp)
└── setup_fee (decimal, nullable)

SUBSCRIPTION
├── subscription_id (UUID)
├── user_id (FK)
├── plan_id (FK)
├── subscription_status (enum: trial, active, paused, cancelled, failed)
├── start_date (timestamp)
├── trial_end_date (timestamp, nullable)
├── next_billing_date (timestamp)
├── current_period_start (timestamp)
├── current_period_end (timestamp)
├── cancellation_date (timestamp, nullable)
├── cancellation_reason (enum: switching, too-expensive, unused, other)
├── auto_renew (boolean)
├── payment_method_id (FK)
├── billing_cycle_count (number)
└── last_payment_date (timestamp, nullable)

SUBSCRIPTION_INVOICE
├── invoice_id (UUID)
├── subscription_id (FK)
├── user_id (FK)
├── invoice_date (timestamp)
├── amount (decimal)
├── status (enum: pending, sent, paid, failed, refunded)
├── billing_reason (enum: subscription-cycle, plan-change, refund)
├── items (array)
│   ├── description (string)
│   ├── quantity (number)
│   ├── unit_price (decimal)
│   └── subtotal (decimal)
├── subtotal (decimal)
├── tax_amount (decimal)
├── discount_amount (decimal)
├── total_amount (decimal)
├── due_date (timestamp)
├── payment_status (enum: unpaid, paid, partially-paid)
├── payment_date (timestamp, nullable)
└── pdf_location (URI)

PAYMENT_ATTEMPT
├── attempt_id (UUID)
├── invoice_id (FK)
├── subscription_id (FK)
├── payment_method_id (FK)
├── attempt_number (number)
├── attempt_date (timestamp)
├── amount (decimal)
├── status (enum: pending, success, failed)
├── failure_reason (enum: insufficient-funds, expired-card, fraud-declined, gateway-error, other)
├── processor_response (object)
├── retry_eligible (boolean)
└── next_retry_date (timestamp, nullable)

DUNNING_CAMPAIGN
├── campaign_id (UUID)
├── subscription_id (FK)
├── invoice_id (FK)
├── campaign_start_date (timestamp)
├── failed_payment_count (number)
├── current_stage (enum: stage-1, stage-2, stage-3, final)
├── status (enum: active, paused, successful, failed-recovery)
├── stages (array)
│   ├── stage_number (1-3)
│   ├── days_after_failure (number)
│   ├── retry_enabled (boolean)
│   ├── communication_sent (boolean)
│   ├── communication_type (enum: email, sms, push)
│   └── update_payment_required (boolean)
└── final_outcome (enum: recovered, cancelled, won)

PLAN_CHANGE
├── change_id (UUID)
├── subscription_id (FK)
├── user_id (FK)
├── from_plan_id (FK)
├── to_plan_id (FK)
├── change_date (timestamp)
├── change_type (enum: upgrade, downgrade, cross-grade)
├── effective_date (timestamp)
├── prorated_amount (decimal)
├── credit_applied (decimal)
├── new_billing_date (timestamp)
└── status (enum: completed, pending, rejected)

CHURN_PREDICTION
├── prediction_id (UUID)
├── subscription_id (FK)
├── user_id (FK)
├── churn_probability (decimal: 0-1.0)
├── churn_risk_factors (array)
├── prediction_date (timestamp)
├── next_prediction_date (timestamp)
├── intervention_triggered (boolean)
├── intervention_type (enum: discount-offer, feature-upgrade, support-outreach, none)
├── outcome (enum: retained, churned, pending)
└── prediction_accuracy (decimal: 0-1.0, after outcome known)

SUBSCRIPTION_ANALYTICS
├── analytics_id (UUID)
├── period (string: YYYY-MM)
├── new_subscriptions (number)
├── cancelled_subscriptions (number)
├── churn_rate (decimal: percentage)
├── mrr (decimal: monthly recurring revenue)
├── mrr_change (decimal: percentage from previous month)
├── average_subscription_lifetime_months (number)
├── ltv (decimal: lifetime value)
├── payment_success_rate (decimal: percentage)
├── dunning_recovery_rate (decimal: percentage)
├── customer_acquisition_cost (decimal)
└── payback_period_months (number)
```

## 📡 Core APIs

### Subscription Management

```
POST /v1/subscriptions
├── Create subscription
├── Request: user_id, plan_id, payment_method_id
└── Response: subscription_id, status=trial or active, next_billing_date

GET /v1/subscriptions/{subscription_id}
├── Get subscription details
└── Response: subscription record with plan details

GET /v1/subscriptions/user/{user_id}
├── Get user's subscriptions
└── Response: subscription list

PUT /v1/subscriptions/{subscription_id}/change-plan
├── Change subscription plan
├── Request: to_plan_id, effective_date
└── Response: plan_change_id, prorated_amount, new_next_billing_date

POST /v1/subscriptions/{subscription_id}/cancel
├── Cancel subscription
├── Request: cancellation_reason
└── Response: subscription_status=cancelled, cancellation_date

POST /v1/subscriptions/{subscription_id}/pause
├── Pause subscription (temporary hold)
├── Request: pause_duration_days
└── Response: subscription_status=paused, resume_date

POST /v1/subscriptions/{subscription_id}/resume
├── Resume paused subscription
├── Request: (none)
└── Response: subscription_status=active, next_billing_date
```

### Billing & Invoicing

```
POST /v1/invoices/generate
├── Generate invoice for subscription
├── Request: subscription_id, invoice_date
└── Response: invoice_id, status=pending, amount

GET /v1/invoices/{invoice_id}
├── Get invoice details
└── Response: invoice record with items

POST /v1/invoices/{invoice_id}/send
├── Send invoice to customer
├── Request: delivery_method (email, sms, portal)
└── Response: send_date, delivery_status

GET /v1/invoices/subscription/{subscription_id}
├── Get all invoices for subscription
├── Query: status, date_range
└── Response: paginated invoice list

POST /v1/invoices/{invoice_id}/record-payment
├── Record payment received
├── Request: payment_amount, payment_method, processor_reference
└── Response: invoice_status=paid, payment_recorded
```

### Payment & Dunning

```
POST /v1/payments/process
├── Process subscription payment
├── Request: subscription_id, payment_method_id, amount
└── Response: payment_id, status=success or failed, processor_response

GET /v1/payments/{payment_id}
├── Get payment details
└── Response: payment record with status

POST /v1/payments/retry
├── Manually retry failed payment
├── Request: invoice_id or payment_id
└── Response: payment_id, retry_initiated

GET /v1/dunning/campaigns
├── List active dunning campaigns
├── Query: status, subscription_id (optional)
└── Response: campaign list with recovery status

POST /v1/dunning/{subscription_id}/start
├── Initiate dunning campaign for failed payment
├── Request: invoice_id
└── Response: campaign_id, status=active, stage=1
```

### Churn Prevention

```
GET /v1/churn-prediction/at-risk
├── Get subscriptions at risk of churning
├── Query: churn_probability_threshold (default 0.5)
└── Response: at_risk_subscriptions with churn factors

POST /v1/churn-prevention/{subscription_id}/intervene
├── Trigger retention intervention
├── Request: intervention_type (discount, upgrade, support)
└── Response: intervention_id, offer_details

POST /v1/churn-prevention/{subscription_id}/offer-discount
├── Offer discount to at-risk subscription
├── Request: discount_percentage, duration_months
└── Response: offer_id, offer_expires_at

GET /v1/churn-analytics
├── Get churn analytics
├── Query: period (YYYY-MM)
└── Response: churn_rate, churn_factors, intervention_effectiveness
```

## 🔄 Workflows

### Subscription Lifecycle

```
1. Sign-up & Trial
   - Customer selects plan
   - Payment method collected
   - Subscription created: status = trial
   - Trial period starts (7-30 days free)

2. Trial Phase
   - Full access to plan features
   - No charge during trial
   - Reminders sent: 3 days, 1 day before trial ends

3. First Payment
   - Trial ends
   - First billing cycle begins
   - Invoice generated: amount = plan_price
   - Payment processed
   - If success: status = active
   - If failure: trigger dunning campaign

4. Active Subscription
   - Recurring billing each cycle
   - Automatic payment retry if failed
   - Plan changes allowed (upgrade, downgrade)
   - Customer support available

5. Cancellation
   - Customer initiates cancel
   - Effective date set (immediate or end of cycle)
   - Final invoice generated (prorated if mid-cycle)
   - Subscription status = cancelled
   - Exit surveys collected

6. Churn Prevention (Optional)
   - At-risk identified by model
   - Intervention offered (discount, upgrade)
   - Retention metrics tracked
```

### Payment Retry (Dunning) Workflow

```
1. Payment Attempt Failed
   - Invoice payment processing fails
   - Failure reason recorded (expired-card, NSF, fraud-declined, etc.)
   - Payment status = failed
   - Dunning campaign initiated

2. Stage 1: Immediate Retry
   - Immediate retry (same day)
   - If success: campaign ended, payment recorded
   - If failure: move to Stage 2

3. Stage 2: Customer Notification
   - 2-3 days after failure
   - Email/SMS sent to customer
   - Request to update payment method
   - Automatic retry on file

4. Stage 3: Escalation
   - 5-7 days after failure
   - Incentive offered: discount if paid immediately
   - More urgent communication
   - Final retry attempt

5. Final Outcome
   - If payment recovered: dunning ends, subscription continues
   - If payment fails: subscription paused or cancelled
   - Record outcome: recovered or lost

6. Win/Loss Analysis
   - Recovered payments tracked
   - Lost subscriptions analyzed
   - Failure patterns identified
   - Process improvements made
```

### Plan Change & Proration

```
1. Customer Initiates Plan Change
   - Selects new plan
   - Change type determined: upgrade, downgrade, cross-grade

2. Proration Calculation
   - Days remaining in current billing cycle calculated
   - Pro-rata amount for remaining days calculated
   - If upgrade: credit applied toward new higher price
   - If downgrade: customer receives credit or refund

3. Effective Date
   - Immediate: charge difference immediately, new billing date = today
   - End of cycle: change effective at next billing date
   - Customer selects preference

4. Billing Update
   - Invoice generated for difference (if immediate)
   - New billing amount set for future cycles
   - New next_billing_date calculated

5. Confirmation
   - Customer notified
   - New plan features activated
   - Old plan features disabled
```

## 🔐 Security & Compliance

### PCI DSS
- Payment methods tokenized (no full card stored)
- Payment processing via PCI-compliant gateway
- Audit logs for all payment operations

### Fair Billing
- Clear billing cycle definitions
- Pro-rata calculations accurate
- No hidden or surprise charges
- Easy cancellation process

### GDPR
- Right to cancel (must be easy)
- Right to data export
- Data deletion after retention period
- Transparent communications

## 📊 Reporting & Analytics

### Subscription Metrics
- New subscriptions per month
- Churn rate (% cancelled)
- Monthly Recurring Revenue (MRR)
- Average Lifetime Value (LTV)
- Customer Acquisition Cost (CAC)

### Billing Metrics
- Payment success rate
- Dunning recovery rate
- Failed payment reasons
- Invoice aging

### Churn Analytics
- Churn rate by plan
- Churn rate by cohort
- Churn factors analysis
- Intervention effectiveness

## 🔗 Integration Points

### Payment Service
- Tokenized payment processing
- Recurring payment support
- Payment retry logic

### User Service
- Customer account management
- Communication preferences

### Notification Service
- Invoice delivery
- Payment reminders
- Churn prevention offers

### Analytics Service
- Subscription metrics
- Churn prediction data
- Revenue reporting

## 📈 Key Metrics

| Metric | Target | Frequency |
|--------|--------|-----------|
| **Churn Rate** | <5% per month (SaaS), <10% (DTC) | Monthly |
| **Payment Success Rate** | 98%+ | Daily |
| **Dunning Recovery Rate** | 5-8% of failed payments | Monthly |
| **Plan Change Success** | 95%+ | Daily |
| **MRR Growth** | +5-15% per month (growing) | Monthly |

## 💻 Implementation Considerations

### Recurring Payment Processing
- Recurring payment gateway integration
- Automatic billing on schedule
- Timezone handling (customer billing date)
- Proration accuracy for plan changes

### Communication
- Invoice delivery (email, PDF, portal)
- Reminders (trial ending, payment due)
- Dunning escalation emails
- Churn prevention offers

### Data
- Subscription state tracking
- Billing history
- Payment history
- Churn prediction models

## 🚀 Example Use Cases

### Use Case 1: Subscription Sign-up
```
Input: Customer selects $19.99/month plan
Process:
  1. Subscription created with 14-day trial
  2. Payment method tokenized
  3. Trial starts: full access
  4. Day 11: "3 days left in trial" email
  5. Day 13: "Your trial ends tomorrow" reminder
  6. Day 14: First charge of $19.99
  7. Payment succeeds: subscription status = active
Output: Active subscriber, $19.99 MRR added
```

### Use Case 2: Payment Failure & Dunning
```
Input: Monthly billing date, card on file declined
Process:
  1. Payment attempt fails: insufficient funds
  2. Dunning campaign starts (Stage 1)
  3. Immediate retry: fails again
  4. Stage 2: email "Update payment method" (Day 2)
  5. Customer updates card
  6. Stage 3 automatic retry (Day 5): succeeds!
  7. Payment recorded, subscription continues
  8. Dunning campaign ends
Output: $19.99 recovered, subscription retained ($19.99 LTV value > intervention cost)
```

### Use Case 3: Plan Upgrade
```
Input: Customer upgrades from $19.99 to $49.99 plan
Process:
  1. Days remaining in cycle: 10 days
  2. Pro-rata credit: $6.66 (10/30 × $19.99)
  3. Upgrade cost: $49.99 × (10/30) = $16.66
  4. Net charge: $16.66 - $6.66 = $10.00
  5. Charge processed immediately
  6. New plan activated
  7. New billing date: 30 days from today
  8. Next billing: $49.99
Output: Customer upgraded, $10 charged, increased MRR by $30/month
```

## 📚 Related Services

- **Payment Service** - Payment processing
- **User Service** - Customer management
- **Notification Service** - Communications
- **Analytics Service** - Metrics

## 🔄 Future Enhancements

- Usage-based pricing (charge per API call, storage, etc.)
- Flexible billing (billing date varies by customer)
- Multi-currency subscriptions
- Team/seat-based subscriptions
- Self-serve plan customization

---

**Service Version:** 1.0  
**Last Updated:** August 2026  
**Status:** Enterprise High Priority  
**Compliance:** PCI DSS, Fair Billing Laws, GDPR

