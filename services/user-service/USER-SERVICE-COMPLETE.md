# User Service - Complete Implementation Guide

**Scale:** 500M+ users | OAuth2 + JWT | 99.99% availability

---

## API Specification

```
POST /v1/users/register
  Create new account
  Body: { email, password, name, country }
  Response: { userId, email, createdAt, verificationRequired }

POST /v1/auth/login
  Authenticate user
  Body: { email, password }
  Response: { accessToken, refreshToken, expiresIn }

POST /v1/auth/social-login
  Login via Google/Facebook/Apple
  Body: { provider, socialToken }
  Response: { accessToken, refreshToken, user }

GET /v1/users/{userId}
  Get user profile
  Response: { userId, email, name, addresses[], preferences, loyaltyPoints }

PATCH /v1/users/{userId}
  Update profile
  Body: { name, email, phone, preferences }
  Response: { success }

POST /v1/users/{userId}/addresses
  Add shipping address
  Body: { name, street, city, state, zip, country, isDefault }
  Response: { addressId }

POST /v1/auth/logout
  Logout user
  Response: { success }

POST /v1/auth/refresh
  Refresh access token
  Body: { refreshToken }
  Response: { accessToken, expiresIn }
```

---

## Domain Models

```
User {
  userId: UUID
  email: String (unique, verified)
  passwordHash: String (bcrypt)
  name: String
  phone: String (optional)
  country: String
  addresses: Address[]
  preferences: UserPreferences
  loyaltyPoints: Integer
  status: UserStatus (ACTIVE, SUSPENDED, DELETED)
  createdAt: DateTime
  lastLoginAt: DateTime
  twoFactorEnabled: Boolean
}

UserPreferences {
  newsletter: Boolean
  smsNotifications: Boolean
  emailNotifications: Boolean
  language: String (en, es, fr)
  currency: String (USD, EUR)
  timezone: String
}

Address {
  addressId: UUID
  userId: UUID
  type: String (shipping, billing, both)
  name: String (recipient name)
  street: String
  city: String
  state: String
  zip: String
  country: String
  phone: String
  isDefault: Boolean
}

UserStatus = ACTIVE | SUSPENDED | DELETED
```

### Business Rules
1. **Email unique:** One email per account
2. **Password strength:** Min 8 chars, 1 upper, 1 number, 1 special
3. **Two-factor:** Optional, recommended for VIP users
4. **Session timeout:** 24 hours inactivity = logout
5. **Privacy:** PII encrypted at rest
6. **GDPR:** Right to deletion (anonymize data)

---

## Use Cases

### UC-001: Registration
**Flow:**
1. User enters email/password
2. System validates (email format, password strength)
3. Check email not registered
4. Create user (password hashed with bcrypt)
5. Send verification email
6. User clicks link, email verified
7. Account activated

### UC-002: Login
**Flow:**
1. User enters credentials
2. Look up user by email
3. Verify password (bcrypt compare)
4. Check account not suspended
5. Generate JWT tokens (access + refresh)
6. Update lastLoginAt
7. Return tokens

### UC-003: Social Login
**Flow:**
1. User clicks "Login with Google"
2. Redirect to Google OAuth
3. User approves
4. Google returns socialToken
5. Call Google API to verify token
6. Check if user exists
7. If new: Create user (auto-verified)
8. If exists: Log in
9. Return JWT tokens

### UC-004: Manage Addresses
**Flow:**
1. User clicks "Add Address"
2. Enter address details
3. Validate address (format, country)
4. Save to database
5. Mark as default (if first address)
6. Show in checkout dropdown

### UC-005: Two-Factor Authentication
**Flow:**
1. User enables 2FA in settings
2. System generates TOTP secret (Google Authenticator)
3. User scans QR code
4. User verifies by entering code
5. On login: After password, prompt for TOTP code
6. Verify code is current (±1 window)
7. Grant access if valid

---

## Company Scenarios

### Amazon User Management
```
Features:
- Multiple logins (email + phone)
- Linked accounts (family sharing)
- Subscription management (Prime)
- Payment methods saved
- Address book (50+ addresses)
- One-click checkout (saved payment + address)

Scale:
- 500M+ active users
- 300M+ verified addresses
- 400M+ saved payment methods

Auth:
- OAuth2 + custom JWT
- Refresh token rotation (7 days)
- Session binding (device + IP)
- Logout all devices option
```

### Shopify Multi-tenant
```
By merchant (tenant):
- Merchant admin login (staff accounts)
- Customer login (B2C buyers)
- B2B buyer login (with approval)

Features:
- Email verification required
- Password reset flow
- Session management (per merchant)
- API tokens (for apps, webhooks)

Security:
- Rate limiting (10 login attempts/hour)
- Account lockout (5 failures = 1 hour)
- Session timeout (30 min inactivity)
```

### Alibaba/Taobao
```
Features:
- Alipay-based login (one-click)
- Real name verification (required for sellers)
- Seller accounts (separate from buyer)
- Buyer levels (based on reputation)
- Phone verification (SMS)

Scale:
- 900M+ registered accounts
- 700M+ verified phone numbers
- 50M+ verified seller accounts
```

---

## Infrastructure

### Authentication Flow (OAuth2)
```
Authorization Code Flow:
1. User clicks "Login"
2. Redirect to /authorize?client_id=...&redirect_uri=...
3. User logs in (credential check)
4. Ask user: "Allow this app to access your profile?"
5. User approves
6. Redirect to redirect_uri?code=...
7. Frontend exchanges code for tokens
8. User logged in

Token structure:
- Access Token: JWT (expires 1 hour)
  - userId, email, roles, permissions
  - Signed with RS256 (public key verification)
  
- Refresh Token: Opaque (expires 7 days)
  - Stored in secure httpOnly cookie
  - Used to get new access token
```

### Password Hashing
```
Algorithm: bcrypt
- Cost factor: 12 (takes ~250ms per hash)
- Salt: Included in hash
- Verification: Compare provided password with stored hash

Never store: Plain password
```

### Session Management
```
Token-based (stateless):
- JWT validated via public key signature
- No session storage needed
- Scalable to any number of servers

Logout:
- Option 1: Client discards token (simple)
- Option 2: Blacklist token (if revocation needed)
  - Redis blacklist set
  - Checked on each request
```

### Database Schema
```sql
CREATE TABLE users (
  user_id UUID PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  name VARCHAR(255),
  phone VARCHAR(20),
  country VARCHAR(50),
  status VARCHAR(20),
  loyalty_points INTEGER DEFAULT 0,
  two_factor_enabled BOOLEAN DEFAULT false,
  two_factor_secret VARCHAR(255),
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  last_login_at TIMESTAMP
);

CREATE TABLE addresses (
  address_id UUID PRIMARY KEY,
  user_id UUID REFERENCES users,
  type VARCHAR(20),
  name VARCHAR(255),
  street VARCHAR(255),
  city VARCHAR(100),
  state VARCHAR(50),
  zip VARCHAR(20),
  country VARCHAR(50),
  is_default BOOLEAN,
  created_at TIMESTAMP
);

CREATE TABLE user_preferences (
  user_id UUID PRIMARY KEY REFERENCES users,
  newsletter BOOLEAN DEFAULT true,
  sms_notifications BOOLEAN DEFAULT true,
  email_notifications BOOLEAN DEFAULT true,
  language VARCHAR(10),
  currency VARCHAR(10),
  timezone VARCHAR(50)
);

CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_country ON users(country);
```

---

## Testing

### Unit Tests
- Password validation (strength rules)
- Email validation (format)
- Address validation (completeness)
- TOTP verification (time window)
- JWT token generation and validation

### Integration Tests
- Full registration flow (email verification)
- Login flow (password check, tokens issued)
- Social login (OAuth callback)
- Address management (CRUD)
- Two-factor flow (enable, disable, verify)
- Logout (token invalidation if needed)

### Load Tests
- 100K logins/second
- <100ms latency p99
- JWT validation <5ms
- Password hashing <300ms (acceptable)
- OAuth callback <500ms

---

## Monitoring

**Key Metrics:**
- Registration rate (new users/day)
- Login success rate (%)
- Failed login attempts (rate)
- Social login adoption (%)
- 2FA adoption (%)
- Session timeout rate (%)
- JWT validation latency (ms)

---

