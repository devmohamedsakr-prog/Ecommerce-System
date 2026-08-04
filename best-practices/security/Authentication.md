# Authentication Implementation Guide

## 🔐 Overview

Complete guide to implementing secure authentication for e-commerce systems.

## 1. Password-Based Authentication

### Password Storage (DO NOT STORE PLAINTEXT)

**Hashing Algorithms (Ranked):**

| Algorithm | Iterations | Speed | Security | Use Case |
|-----------|-----------|-------|----------|----------|
| **Argon2** | 2-3 | Slow (200-500ms) | ⭐⭐⭐⭐⭐ | New projects (RECOMMENDED) |
| **bcrypt** | 10-12 | Slow (100-200ms) | ⭐⭐⭐⭐ | Industry standard |
| **scrypt** | 14+ | Slow (100-200ms) | ⭐⭐⭐⭐ | High security needs |
| **PBKDF2** | 100k+ | Medium (50-100ms) | ⭐⭐⭐ | Legacy systems |
| **SHA-256** | None | Fast (1ms) | ⭐ | ❌ DO NOT USE |
| **MD5** | None | Fast (1ms) | ⭐ | ❌ DO NOT USE |

### Implementation Example (Node.js with Argon2)

```javascript
const argon2 = require('argon2');
const crypto = require('crypto');

// Registration: Hash and store password
async function registerUser(email, password) {
  // Validate password strength
  if (password.length < 12) {
    throw new Error('Password must be at least 12 characters');
  }
  
  if (!/[A-Z]/.test(password) || !/[0-9]/.test(password)) {
    throw new Error('Password must contain uppercase letter and number');
  }

  try {
    // Hash password with Argon2
    const hashedPassword = await argon2.hash(password, {
      type: argon2.argon2id,
      memoryCost: 65540,
      timeCost: 3,
      parallelism: 4
    });

    // Store in database
    const user = await db.users.create({
      email,
      password_hash: hashedPassword,
      created_at: new Date()
    });

    return user.id;
  } catch (error) {
    throw new Error('Password hashing failed');
  }
}

// Login: Verify password
async function login(email, password) {
  const user = await db.users.findOne({ email });
  
  if (!user) {
    // Don't reveal if email exists (timing attack prevention)
    await simulateHashTime();
    throw new Error('Invalid credentials');
  }

  try {
    // Verify password against hash
    const isValid = await argon2.verify(user.password_hash, password);
    
    if (!isValid) {
      // Log failed attempt
      await logFailedLogin(email);
      
      // Rate limiting check
      const recentFailures = await getRecentFailures(email);
      if (recentFailures > 5) {
        await lockAccount(email, 15 * 60); // Lock 15 minutes
        throw new Error('Account locked. Try again later.');
      }
      
      throw new Error('Invalid credentials');
    }

    // Clear failed attempts on successful login
    await clearFailedLogins(email);
    
    // Generate session/token
    const token = generateJWT(user);
    return token;
  } catch (error) {
    throw new Error('Login failed');
  }
}
```

### Password Requirements

**Minimum Standards:**
```
Length:     12+ characters (16+ better)
Uppercase:  At least one A-Z
Lowercase:  At least one a-z
Numbers:    At least one 0-9
Symbols:    At least one !@#$%^&*
No common:  Not in top 10k most used passwords
No personal: Not email, username, or variations
```

**Strength Meter:**
```
Weak:     < 8 characters, all same type (letters only)
Fair:     8-11 characters, mixed types
Good:     12-15 characters, mixed types, symbols
Strong:   16+ characters, all types, symbols, random
```

---

## 2. JWT (JSON Web Token) Authentication

### JWT Structure

```
Header.Payload.Signature

Example:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiI1ZmE2NjE0YzhhODQwYjAwMWM5N2ZkZGEiLCJlbWFpbCI6InVzZXJAZXhhbXBsZS5jb20iLCJpYXQiOjE2MjYwMDM2NjEsImV4cCI6MTYyNjAwNzI2MX0.
_U5sLYY5Z-LF9WKvdZMl_t5xRFrQkqBbkTJSPyTlLrE

Decoded:
Header:
{
  "alg": "HS256",
  "typ": "JWT"
}

Payload:
{
  "sub": "user-id-123",
  "email": "user@example.com",
  "role": "customer",
  "iat": 1626003661,     // Issued at
  "exp": 1626007261,     // Expires in
  "nbf": 1626003661      // Not before
}

Signature:
HMAC-SHA256(Header + Payload, secret_key)
```

### JWT Implementation

```javascript
const jwt = require('jsonwebtoken');
const crypto = require('crypto');

// Configuration
const JWT_SECRET = process.env.JWT_SECRET;
const JWT_EXPIRY = '15m';        // Access token
const REFRESH_EXPIRY = '7d';     // Refresh token

// Generate tokens
function generateTokens(user) {
  // Access token (short-lived)
  const accessToken = jwt.sign(
    {
      sub: user.id,
      email: user.email,
      role: user.role
    },
    JWT_SECRET,
    { 
      algorithm: 'HS256',
      expiresIn: JWT_EXPIRY
    }
  );

  // Refresh token (long-lived, stored in database)
  const refreshToken = jwt.sign(
    { 
      sub: user.id,
      tokenVersion: user.tokenVersion // Invalidate old tokens on logout
    },
    JWT_SECRET,
    { 
      algorithm: 'HS256',
      expiresIn: REFRESH_EXPIRY
    }
  );

  // Store refresh token in database
  db.refreshTokens.create({
    userId: user.id,
    token: crypto.createHash('sha256').update(refreshToken).digest('hex'),
    expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  });

  return { accessToken, refreshToken };
}

// Verify token middleware
function verifyToken(req, res, next) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader) {
    return res.status(401).json({ error: 'No authorization header' });
  }

  const [scheme, token] = authHeader.split(' ');
  
  if (scheme !== 'Bearer') {
    return res.status(401).json({ error: 'Invalid authentication scheme' });
  }

  try {
    const decoded = jwt.verify(token, JWT_SECRET, {
      algorithms: ['HS256']
    });
    
    req.user = decoded;
    next();
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired', code: 'TOKEN_EXPIRED' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}

// Refresh token endpoint
app.post('/auth/refresh', async (req, res) => {
  const { refreshToken } = req.body;
  
  if (!refreshToken) {
    return res.status(401).json({ error: 'No refresh token provided' });
  }

  try {
    // Verify refresh token
    const decoded = jwt.verify(refreshToken, JWT_SECRET);
    
    // Check if token exists in database and not revoked
    const storedToken = await db.refreshTokens.findOne({
      userId: decoded.sub,
      token: crypto.createHash('sha256').update(refreshToken).digest('hex')
    });

    if (!storedToken || new Date() > storedToken.expiresAt) {
      return res.status(401).json({ error: 'Invalid or expired refresh token' });
    }

    // Generate new access token
    const user = await db.users.findById(decoded.sub);
    const newAccessToken = jwt.sign(
      {
        sub: user.id,
        email: user.email,
        role: user.role
      },
      JWT_SECRET,
      { expiresIn: JWT_EXPIRY }
    );

    res.json({ accessToken: newAccessToken });
  } catch (error) {
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});

// Logout endpoint
app.post('/auth/logout', verifyToken, async (req, res) => {
  // Delete all refresh tokens for this user
  await db.refreshTokens.deleteMany({ userId: req.user.sub });
  
  res.json({ message: 'Logged out successfully' });
});
```

---

## 3. Multi-Factor Authentication (MFA)

### Two-Factor Authentication (2FA)

**Option 1: TOTP (Time-based One-Time Password)**

```javascript
const speakeasy = require('speakeasy');
const QRCode = require('qrcode');

// Generate 2FA secret
async function setupTOTP(user) {
  const secret = speakeasy.generateSecret({
    name: `ECommerce (${user.email})`,
    issuer: 'ECommerce',
    length: 32
  });

  // Generate QR code
  const qrCode = await QRCode.toDataURL(secret.otpauth_url);

  return {
    secret: secret.base32,
    qrCode: qrCode
  };
}

// Verify TOTP code
function verifyTOTP(secret, token) {
  const verified = speakeasy.totp.verify({
    secret: secret,
    encoding: 'base32',
    token: token,
    window: 2  // Allow ±2 time windows for clock skew
  });

  return verified;
}

// Enable 2FA flow
app.post('/auth/2fa/setup', verifyToken, async (req, res) => {
  const { secret, qrCode } = await setupTOTP(req.user);

  // Store secret in temporary cache (not yet enabled)
  await cache.set(`2fa:temp:${req.user.sub}`, secret, 300); // 5 min expiry

  res.json({ 
    qrCode,
    secret: secret  // For manual entry if QR doesn't work
  });
});

app.post('/auth/2fa/verify', verifyToken, async (req, res) => {
  const { code } = req.body;
  
  // Get temp secret
  const secret = await cache.get(`2fa:temp:${req.user.sub}`);
  
  if (!secret) {
    return res.status(400).json({ error: '2FA setup expired' });
  }

  if (!verifyTOTP(secret, code)) {
    return res.status(400).json({ error: 'Invalid code' });
  }

  // Store secret permanently
  await db.users.update(
    { id: req.user.sub },
    { 
      totp_secret: secret,
      two_fa_enabled: true
    }
  );

  // Clear temporary secret
  await cache.delete(`2fa:temp:${req.user.sub}`);

  res.json({ message: '2FA enabled successfully' });
});

// Login with 2FA
app.post('/auth/verify-2fa', async (req, res) => {
  const { email, password, totpCode } = req.body;

  const user = await db.users.findOne({ email });
  
  if (!user || !await argon2.verify(user.password_hash, password)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  if (!user.two_fa_enabled) {
    // 2FA not enabled, issue token
    const token = generateJWT(user);
    return res.json({ accessToken: token });
  }

  // 2FA enabled, verify code
  if (!verifyTOTP(user.totp_secret, totpCode)) {
    return res.status(401).json({ error: 'Invalid 2FA code' });
  }

  const token = generateJWT(user);
  res.json({ accessToken: token });
});
```

**Option 2: SMS-Based 2FA**

```javascript
const twilio = require('twilio');

const twilioClient = twilio(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);

// Send SMS verification code
async function sendSMS2FACode(phoneNumber, userId) {
  const code = Math.floor(100000 + Math.random() * 900000).toString();
  
  // Store code with expiry
  await cache.set(`2fa:sms:${userId}`, code, 300); // 5 min expiry

  // Send SMS
  await twilioClient.messages.create({
    body: `Your verification code is: ${code}`,
    from: process.env.TWILIO_PHONE_NUMBER,
    to: phoneNumber
  });
}

// Verify SMS code
async function verifySMS2FACode(userId, code) {
  const storedCode = await cache.get(`2fa:sms:${userId}`);
  
  if (!storedCode || storedCode !== code) {
    return false;
  }

  await cache.delete(`2fa:sms:${userId}`);
  return true;
}
```

---

## 4. OAuth 2.0 Integration (Third-Party Login)

### OAuth 2.0 Authorization Code Flow

```
User clicks "Login with Google"
    ↓
Redirect to: https://accounts.google.com/o/oauth2/v2/auth?
  client_id=...&
  redirect_uri=https://myapp.com/callback&
  scope=openid+email+profile&
  state=random_string&
  response_type=code
    ↓
User logs in to Google
    ↓
User approves permissions
    ↓
Google redirects to: https://myapp.com/callback?
  code=auth_code_xyz&
  state=random_string
    ↓
Server validates state (CSRF protection)
    ↓
Server exchanges code for token:
  POST https://oauth2.googleapis.com/token
  client_id=...&
  client_secret=...&
  code=auth_code_xyz&
  grant_type=authorization_code
    ↓
Google returns: { access_token, id_token, ... }
    ↓
Server gets user info:
  GET https://www.googleapis.com/oauth2/v2/userinfo
  Authorization: Bearer access_token
    ↓
Create or update user in database
    ↓
Issue application JWT
    ↓
Redirect to: https://myapp.com/dashboard?token=jwt_token
```

### OAuth Implementation (Google Example)

```javascript
const { OAuth2Client } = require('google-auth-library');

const oauth2Client = new OAuth2Client(
  process.env.GOOGLE_CLIENT_ID,
  process.env.GOOGLE_CLIENT_SECRET,
  'http://localhost:3000/auth/google/callback'
);

// Step 1: Redirect to Google
app.get('/auth/google', (req, res) => {
  const state = crypto.randomBytes(16).toString('hex');
  
  // Store state in session for CSRF validation
  req.session.oauth_state = state;

  const authUrl = oauth2Client.generateAuthUrl({
    access_type: 'offline',
    scope: ['openid', 'email', 'profile'],
    state: state
  });

  res.redirect(authUrl);
});

// Step 2: Handle callback
app.get('/auth/google/callback', async (req, res) => {
  const { code, state } = req.query;

  // CSRF validation
  if (state !== req.session.oauth_state) {
    return res.status(400).json({ error: 'CSRF validation failed' });
  }

  try {
    // Exchange code for tokens
    const { tokens } = await oauth2Client.getToken(code);
    oauth2Client.setCredentials(tokens);

    // Get user info
    const oauth2 = google.oauth2({ version: 'v2', auth: oauth2Client });
    const userInfo = await oauth2.userinfo.get();

    // Find or create user
    let user = await db.users.findOne({ email: userInfo.data.email });
    
    if (!user) {
      user = await db.users.create({
        email: userInfo.data.email,
        name: userInfo.data.name,
        googleId: userInfo.data.id,
        avatar: userInfo.data.picture
      });
    } else {
      // Update Google ID if not set
      if (!user.googleId) {
        await db.users.update({ id: user.id }, { googleId: userInfo.data.id });
      }
    }

    // Issue JWT
    const token = generateJWT(user);

    // Redirect with token
    res.redirect(`http://localhost:3000/dashboard?token=${token}`);
  } catch (error) {
    res.status(500).json({ error: 'Authentication failed' });
  }
});
```

---

## ✅ Authentication Security Checklist

- [ ] Passwords hashed with Argon2/bcrypt
- [ ] Passwords minimum 12 characters
- [ ] JWT expiry set (15-60 minutes)
- [ ] Refresh tokens stored securely
- [ ] HTTPS only (no HTTP)
- [ ] Rate limiting on login attempts
- [ ] Account lockout after N failed attempts
- [ ] 2FA enabled for admin accounts
- [ ] Password reset via email with token
- [ ] Session timeout configured
- [ ] Credentials never logged
- [ ] Dependencies updated monthly

---

**Next:** Review Authorization.md for access control patterns.
