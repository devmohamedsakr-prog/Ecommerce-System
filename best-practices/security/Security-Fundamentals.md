# Security Fundamentals for E-Commerce Systems

## 🔒 Overview

Security is non-negotiable in e-commerce. This guide covers critical security practices for protecting customer data, payments, and systems.

## 📋 Security Layers

```
Layer 1: Infrastructure Security
├── Network security
├── SSL/TLS encryption
├── DDoS protection
└── WAF (Web Application Firewall)

Layer 2: Application Security
├── Authentication & Authorization
├── Input validation
├── Output encoding
└── Secure dependencies

Layer 3: Data Security
├── Encryption at rest
├── Encryption in transit
├── Key management
└── Data masking

Layer 4: Compliance
├── PCI DSS (Payment Card Industry)
├── GDPR (EU data protection)
├── HIPAA (if health-related)
└── Local regulations
```

## 🔐 PCI DSS Compliance (Critical for Payments)

### Level 1: Highest Security Requirement
- Processing over 6 million card transactions annually
- Mandatory annual third-party assessment
- Maximum fine: $100,000+ per incident

### What PCI DSS Requires

**Requirement 1-6: Infrastructure**
```
1. Firewall configuration standards
2. No default passwords
3. Protect cardholder data
4. Encryption for cardholder data
5. Malware protection
6. Secure development and patching
```

**Requirement 7-10: Access & Monitoring**
```
7. Restrict access by business need
8. Unique IDs for all staff
9. Logging and monitoring
10. Regular security testing
```

### PCI DSS Best Practices for E-Commerce

**DO:**
- ✅ Never store full card numbers
- ✅ Use tokens from payment processors
- ✅ Encrypt all cardholder data
- ✅ Validate cards with networks
- ✅ Regular penetration testing
- ✅ Keep all systems patched
- ✅ Use secure APIs only

**DON'T:**
- ❌ Never store CVV codes
- ❌ Never transmit card data unencrypted
- ❌ Never store card data in logs
- ❌ Never use unsecured connections
- ❌ Never trust unvalidated input
- ❌ Never hardcode credentials

### PCI DSS Compliant Payment Flow

```
Customer enters card data
    ↓
JavaScript tokenization (client-side)
    ↓
Send token (NOT card data) to server
    ↓
Server processes with payment processor
    ↓
Payment processor validates card
    ↓
Processor returns: success/failure
    ↓
Server records transaction (NO card data)
    ↓
Customer gets confirmation
```

### Tokenization Example

```json
{
  "creditCard": {
    "number": "4532-XXXX-XXXX-9123",
    "expiry": "12/25"
  }
}

↓ Tokenization

{
  "paymentToken": "tok_1234567890abcdef",
  "cardLast4": "9123",
  "cardBrand": "Visa"
}

↓ This is all you store/transmit
```

---

## 🔑 Authentication & Authorization

### Authentication (Who Are You?)

**Method 1: Basic Authentication (Legacy)**
```
Authorization: Basic base64(username:password)

❌ Avoid: Not secure without HTTPS
```

**Method 2: API Keys (Simple)**
```
Authorization: Bearer api_key_12345

✅ Use for: Server-to-server, internal APIs
❌ Not for: Public APIs, user authentication
```

**Method 3: JWT (JSON Web Tokens) - Recommended**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

Structure:
[Header].[Payload].[Signature]

Header:
{
  "alg": "HS256",
  "typ": "JWT"
}

Payload:
{
  "sub": "user123",
  "email": "user@example.com",
  "iat": 1626003661,
  "exp": 1626007261
}

Signature: HMAC(Header + Payload, secret_key)
```

**JWT Best Practices:**
```
✅ DO:
- Set short expiration (15-60 minutes)
- Use refresh tokens for renewal
- Sign with strong algorithm (HS256+)
- Transmit only over HTTPS
- Include required claims (exp, iat, sub)

❌ DON'T:
- Use weak algorithms (HS128)
- Never store sensitive data in payload
- No long expiration times
- No cleartext storage
```

**Method 4: OAuth 2.0 (For Third-Party Apps)**
```
User wants to authorize app
    ↓
App redirects to: https://oauth-provider.com/authorize?client_id=...
    ↓
User logs in and approves
    ↓
Provider redirects back with authorization code
    ↓
App exchanges code for access token
    ↓
App uses access token to access user data
```

### Authorization (What Can You Do?)

**Role-Based Access Control (RBAC)**
```
User Role → Permissions

Admin Role:
├── create_users
├── delete_users
├── view_reports
└── manage_settings

Seller Role:
├── create_listings
├── view_own_sales
└── process_own_orders

Customer Role:
├── view_catalog
├── create_orders
└── view_own_orders
```

**Implementation:**
```javascript
// Check permissions before action
function canDeleteOrder(user, order) {
  if (user.role === 'admin') return true;
  if (user.role === 'support' && order.status === 'pending') return true;
  if (user.id === order.customerId && !order.shipped) return true;
  return false;
}
```

---

## 🔒 Data Encryption

### Encryption at Rest (Data in Database)

**Option 1: Database-Level Encryption**
```sql
-- PostgreSQL
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email TEXT,
  payment_info TEXT ENCRYPTED WITH (encryption_key='key')
);
```

**Option 2: Application-Level Encryption**
```javascript
const crypto = require('crypto');

function encryptPaymentInfo(data) {
  const cipher = crypto.createCipher('aes-256-cbc', process.env.ENCRYPTION_KEY);
  let encrypted = cipher.update(data, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  return encrypted;
}

function decryptPaymentInfo(encrypted) {
  const decipher = crypto.createDecipher('aes-256-cbc', process.env.ENCRYPTION_KEY);
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

### Encryption in Transit (Data Over Network)

**HTTPS/TLS**
```
Client wants to connect to server
    ↓
Server sends SSL certificate
    ↓
Client validates certificate
    ↓
TLS handshake (exchange keys)
    ↓
Encrypted channel established
    ↓
All data encrypted in both directions
```

**Implementation:**
```javascript
// Always redirect HTTP to HTTPS
app.use((req, res, next) => {
  if (req.header('x-forwarded-proto') !== 'https') {
    res.redirect(`https://${req.header('host')}${req.url}`);
  } else {
    next();
  }
});

// Set security headers
app.use((req, res, next) => {
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  next();
});
```

---

## 🛡️ Input Validation & Output Encoding

### Input Validation (Never Trust User Input)

**Principle: Whitelist, Don't Blacklist**

```javascript
// ❌ BAD: Blacklist approach
function validateEmail(email) {
  if (!email.includes('..')) return true;  // Incomplete!
  return false;
}

// ✅ GOOD: Whitelist approach
function validateEmail(email) {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return emailRegex.test(email) && email.length <= 255;
}
```

### SQL Injection Prevention

```javascript
// ❌ VULNERABLE
const query = `SELECT * FROM users WHERE email = '${email}'`;
// If email = "'; DROP TABLE users; --"
// Query becomes: SELECT * FROM users WHERE email = ''; DROP TABLE users; --'

// ✅ SAFE: Parameterized queries
const query = 'SELECT * FROM users WHERE email = $1';
db.query(query, [email]);
```

### XSS (Cross-Site Scripting) Prevention

```javascript
// ❌ VULNERABLE: Direct HTML insertion
function displayReview(review) {
  return `<div>${review.text}</div>`;
}

// If review.text = "<img src=x onerror='alert(\"XSS\")'>"
// Will execute JavaScript!

// ✅ SAFE: HTML escape
function escapeHtml(text) {
  const div = document.createElement('div');
  div.textContent = text;
  return div.innerHTML;
}

function displayReview(review) {
  return `<div>${escapeHtml(review.text)}</div>`;
}
```

---

## 🔑 Key Management

### Key Storage

**Where to Store Encryption Keys:**

```
❌ DON'T:
- Hardcoded in code
- In version control
- In config files
- In logs
- Publicly accessible

✅ DO:
- Environment variables (for dev)
- AWS KMS, Google Cloud KMS, Azure Key Vault (production)
- HashiCorp Vault
- Encrypted key management system
```

### Key Rotation

```
Current Key (Active)
    ↓ (Every 90 days)
New Key Generated
    ↓
Both keys work for period
    ↓
Old data encrypted with old key
New data encrypted with new key
    ↓ (After 30 days)
Old key deactivated
```

---

## 🔍 Security Monitoring & Incident Response

### What to Monitor

```
Security Events:
├── Failed login attempts (>5 in 5 min = block)
├── Unusual access patterns
├── Large data exports
├── Permission changes
├── Configuration changes
├── Payment errors
└── Database access anomalies
```

### Alert Thresholds

```
CRITICAL (Immediate action):
- SQL injection attempt detected
- Multiple failed authentication
- Unauthorized admin access
- Payment processing failure

HIGH (Within 1 hour):
- Unusual traffic spike
- Suspicious API calls
- Data access anomalies

MEDIUM (Within 24 hours):
- Failed backup
- Certificate expiring soon
- Deprecated library usage
```

---

## ✅ Security Checklist

### Before Launch
- [ ] All passwords hashed (bcrypt, scrypt, Argon2)
- [ ] All secrets in environment variables
- [ ] HTTPS/TLS enabled
- [ ] SQL injection protected (parameterized queries)
- [ ] XSS protection (output encoding)
- [ ] CSRF protection (tokens on forms)
- [ ] Rate limiting enabled
- [ ] Logging configured (not logging sensitive data)
- [ ] Backups encrypted and tested
- [ ] Security headers set
- [ ] Dependencies scanned for vulnerabilities
- [ ] Penetration testing completed

### Ongoing
- [ ] Security patches applied within 24 hours
- [ ] Monthly security reviews
- [ ] Quarterly penetration testing
- [ ] Annual third-party assessment
- [ ] Incident response plan documented
- [ ] Employee security training
- [ ] Access reviews (quarterly)
- [ ] Encryption key rotation (quarterly)

---

## 📚 Related Files

- Authentication.md (detailed auth patterns)
- Authorization.md (access control details)
- Data-Encryption.md (encryption techniques)
- Security-Checklist.md (pre-launch review)
- Incident-Response.md (handling breaches)

---

**Next:** Review Authentication.md for detailed implementation patterns.
