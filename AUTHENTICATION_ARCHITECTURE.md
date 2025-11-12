# Freelance402 Authentication Architecture

## 🔐 Overview

A comprehensive multi-modal authentication system supporting:
- **Email/Password** - Traditional authentication
- **Web3 Wallet** - Ethereum, Solana, and X402-compatible wallets
- **OAuth 2.0** - Google, GitHub, LinkedIn
- **Magic Links** - Passwordless email authentication
- **2FA/MFA** - Time-based OTP and SMS verification

**Key Features**:
- Unified user identity across auth methods
- X402 wallet integration for payments
- JWT-based session management
- Refresh token rotation
- Device tracking and management
- Social account linking

---

## 🏗️ Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                        Client Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │   Web App    │  │  Mobile App  │  │  Desktop App │        │
│  │   (React)    │  │(React Native)│  │  (Electron)  │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
└────────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌────────────────────────────────────────────────────────────────┐
│                     Auth Middleware                            │
│              (Express.js + Passport.js)                        │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  JWT Verification │ Rate Limiting │ Device Fingerprinting│ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌────────────────────────────────────────────────────────────────┐
│                   Auth Service Layer                           │
│                                                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │  Email/Password │  │  Wallet Auth    │  │  OAuth 2.0   │ │
│  │  • Registration │  │  • Sign Message │  │  • Google    │ │
│  │  • Login        │  │  • Verify Sig   │  │  • GitHub    │ │
│  │  • Password Rst │  │  • Link Wallet  │  │  • LinkedIn  │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
│                                                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │  Magic Link     │  │  2FA/MFA        │  │  Session Mgmt│ │
│  │  • Send Link    │  │  • TOTP         │  │  • JWT       │ │
│  │  • Verify Token │  │  • SMS          │  │  • Refresh   │ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└────────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌────────────────────────────────────────────────────────────────┐
│                    Integration Layer                           │
│                                                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │  X402 Protocol  │  │  User Service   │  │  Subscription│ │
│  │  • Wallet Link  │  │  • Profile      │  │  Service     │ │
│  │  • Payment Auth │  │  • Permissions  │  │  • Tier Check│ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└────────────────────────────────────────────────────────────────┘
                            │
                            ↓
┌────────────────────────────────────────────────────────────────┐
│                      Data Layer                                │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │    Users     │  │   Sessions   │  │  AuthMethods │       │
│  │  (MongoDB)   │  │   (Redis)    │  │  (MongoDB)   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└────────────────────────────────────────────────────────────────┘
```

---

## 📊 Database Schema

### 1. **users** Collection (Enhanced)

```javascript
{
  _id: ObjectId,

  // Primary identifier
  email: String (unique, indexed, sparse), // Sparse for wallet-only users
  emailVerified: Boolean,
  emailVerifiedAt: Date,

  // Traditional auth
  passwordHash: String, // Nullable for wallet-only users
  passwordLastChanged: Date,
  passwordResetToken: String,
  passwordResetExpires: Date,

  // Profile
  profile: {
    firstName: String,
    lastName: String,
    displayName: String,
    avatar: String,
    bio: String,
    location: {
      country: String,
      city: String,
      timezone: String
    },
    phone: String,
    phoneVerified: Boolean
  },

  // Role and permissions
  role: Enum ['client', 'contractor', 'admin'],
  permissions: [String],

  // Account status
  status: Enum ['active', 'suspended', 'deleted', 'pending_verification'],
  suspendedReason: String,
  suspendedUntil: Date,

  // Multi-factor authentication
  twoFactorEnabled: Boolean,
  twoFactorSecret: String, // Encrypted TOTP secret
  twoFactorBackupCodes: [String], // Hashed backup codes
  twoFactorMethod: Enum ['totp', 'sms', 'email'],

  // Security
  security: {
    loginAttempts: Number,
    lockoutUntil: Date,
    lastPasswordChange: Date,
    passwordHistory: [String], // Last 5 password hashes
    securityQuestions: [{
      question: String,
      answerHash: String
    }]
  },

  // Account linking
  linkedAccounts: {
    google: {
      id: String,
      email: String,
      linkedAt: Date
    },
    github: {
      id: String,
      username: String,
      linkedAt: Date
    },
    linkedin: {
      id: String,
      email: String,
      linkedAt: Date
    }
  },

  // Wallet authentication
  wallets: [{
    address: String (indexed),
    chain: Enum ['ethereum', 'solana', 'polygon', 'x402'],
    isPrimary: Boolean,
    isVerified: Boolean,
    verifiedAt: Date,
    nonce: String, // For signature verification
    lastUsed: Date,
    label: String // User-defined label
  }],

  // X402 integration
  x402Wallet: {
    address: String,
    verified: Boolean,
    balance: Number,
    publicKey: String,
    encryptedPrivateKey: String, // If custodial
    isCustodial: Boolean
  },

  // Subscription (reference)
  subscription: {
    tier: Enum ['free', 'starter', 'pro', 'elite'],
    subscriptionId: ObjectId (ref: subscriptions),
    isActive: Boolean,
    expiresAt: Date
  },

  // Preferences
  preferences: {
    language: String,
    currency: String,
    timezone: String,
    notifications: {
      email: Boolean,
      push: Boolean,
      sms: Boolean,
      inApp: Boolean
    },
    privacy: {
      profileVisibility: Enum ['public', 'private', 'connections'],
      showEmail: Boolean,
      showWallet: Boolean
    }
  },

  // Activity tracking
  activity: {
    lastLogin: Date,
    lastLoginIp: String,
    lastLoginDevice: String,
    loginCount: Number,
    createdIp: String
  },

  // Metadata
  createdAt: Date,
  updatedAt: Date,
  deletedAt: Date // Soft delete
}
```

### 2. **auth_sessions** Collection (MongoDB + Redis)

```javascript
{
  _id: ObjectId,

  userId: ObjectId (ref: users, indexed),

  // Session tokens
  accessToken: String (hashed, indexed),
  refreshToken: String (hashed, indexed),

  // Token metadata
  accessTokenExpiry: Date,
  refreshTokenExpiry: Date,
  tokenFamily: String, // For refresh token rotation

  // Device information
  device: {
    type: Enum ['web', 'mobile', 'desktop', 'api'],
    os: String,
    browser: String,
    version: String,
    fingerprint: String (indexed),
    userAgent: String
  },

  // Location
  location: {
    ip: String (indexed),
    country: String,
    city: String,
    coordinates: {
      lat: Number,
      lng: Number
    }
  },

  // Authentication context
  authMethod: Enum ['email', 'wallet', 'google', 'github', 'linkedin', 'magic_link'],
  authProvider: String,

  // Session status
  status: Enum ['active', 'expired', 'revoked', 'suspicious'],
  revokedReason: String,
  revokedAt: Date,

  // Security flags
  isTrusted: Boolean,
  requiresMFA: Boolean,
  mfaVerified: Boolean,
  mfaVerifiedAt: Date,

  // Activity
  lastActivity: Date,
  activityCount: Number,

  createdAt: Date,
  expiresAt: Date (indexed for TTL)
}
```

### 3. **auth_methods** Collection

```javascript
{
  _id: ObjectId,

  userId: ObjectId (ref: users, indexed),

  type: Enum [
    'email_password',
    'wallet_signature',
    'oauth_google',
    'oauth_github',
    'oauth_linkedin',
    'magic_link'
  ],

  // Provider-specific data
  provider: {
    name: String,
    id: String, // Provider user ID
    email: String,
    username: String,
    accessToken: String (encrypted),
    refreshToken: String (encrypted),
    expiresAt: Date,
    scope: [String]
  },

  // Wallet-specific data
  wallet: {
    address: String (indexed),
    chain: String,
    publicKey: String,
    lastNonce: String,
    signatureCount: Number
  },

  // Status
  isPrimary: Boolean,
  isActive: Boolean,
  isVerified: Boolean,
  verifiedAt: Date,

  // Security
  lastUsed: Date,
  usageCount: Number,

  // Metadata
  createdAt: Date,
  updatedAt: Date
}
```

### 4. **magic_links** Collection

```javascript
{
  _id: ObjectId,

  email: String (indexed),
  token: String (hashed, unique, indexed),

  purpose: Enum ['login', 'signup', 'verify_email', 'reset_password'],

  // Metadata
  ipAddress: String,
  userAgent: String,

  // Status
  used: Boolean,
  usedAt: Date,
  attempts: Number,

  createdAt: Date,
  expiresAt: Date (indexed for TTL)
}
```

### 5. **verification_codes** Collection

```javascript
{
  _id: ObjectId,

  userId: ObjectId (ref: users, indexed),

  code: String (hashed),
  type: Enum ['email', 'sms', 'totp'],
  purpose: Enum ['login', 'verify', 'reset', 'enable_2fa', 'transaction'],

  // Contact info
  destination: String, // Email or phone

  // Status
  verified: Boolean,
  verifiedAt: Date,
  attempts: Number,
  maxAttempts: Number,

  // Metadata
  ipAddress: String,

  createdAt: Date,
  expiresAt: Date (indexed for TTL)
}
```

### 6. **audit_logs** Collection

```javascript
{
  _id: ObjectId,

  userId: ObjectId (ref: users, indexed),

  action: String (indexed), // 'login', 'logout', 'password_change', etc.
  category: Enum ['auth', 'profile', 'payment', 'security'],

  // Event details
  details: {
    method: String,
    success: Boolean,
    failureReason: String,
    changes: Object, // What changed
    oldValue: Object,
    newValue: Object
  },

  // Context
  context: {
    ip: String (indexed),
    userAgent: String,
    device: String,
    location: Object,
    sessionId: ObjectId
  },

  // Severity
  severity: Enum ['info', 'warning', 'critical'],

  timestamp: Date (indexed),
  createdAt: Date
}
```

---

## 🔑 Authentication Flows

### 1. Email/Password Registration Flow

```
┌─────────┐
│ Client  │
└────┬────┘
     │
     │ POST /api/auth/register
     │ { email, password, role, firstName, lastName }
     ↓
┌────────────────────────────────────────────┐
│          1. Validate Input                 │
│  • Email format                            │
│  • Password strength (min 8 chars)         │
│  • Required fields present                 │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          2. Check Email Exists             │
│  • Query database for email                │
│  • Return error if exists                  │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          3. Hash Password                  │
│  • bcrypt with cost factor 12              │
│  • Generate salt                           │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          4. Create User Record             │
│  • Insert into users collection            │
│  • emailVerified: false                    │
│  • Generate verification token             │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          5. Send Verification Email        │
│  • Generate secure token (32 bytes)        │
│  • Store in magic_links collection         │
│  • Send email with verification link       │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          6. Return Response                │
│  • User ID                                 │
│  • Message: "Check email for verification" │
│  • Do NOT auto-login yet                   │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌─────────┐
│ Client  │ "Success! Check your email"
└─────────┘
```

### 2. Email Verification Flow

```
User clicks link → GET /api/auth/verify-email?token=xxx
                          ↓
                  Verify token in DB
                          ↓
                  Mark email as verified
                          ↓
                  Generate JWT tokens
                          ↓
                  Redirect to app with tokens
```

### 3. Email/Password Login Flow

```
┌─────────┐
│ Client  │
└────┬────┘
     │
     │ POST /api/auth/login
     │ { email, password, deviceFingerprint }
     ↓
┌────────────────────────────────────────────┐
│          1. Rate Limiting Check            │
│  • Check IP-based rate limit (5/min)       │
│  • Check user-based rate limit (10/hour)   │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          2. Find User by Email             │
│  • Query users collection                  │
│  • Return generic error if not found       │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          3. Check Account Status           │
│  • Verify account not suspended            │
│  • Check if locked out                     │
│  • Verify email confirmed                  │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          4. Verify Password                │
│  • bcrypt.compare(password, hash)          │
│  • Increment loginAttempts on failure      │
│  • Lock account after 5 failed attempts    │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          5. Check 2FA Requirement          │
│  • If 2FA enabled, generate TOTP           │
│  • Return tempToken + require2FA flag      │
│  • Wait for 2FA verification               │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          6. Generate Tokens                │
│  • Create JWT access token (15 min)        │
│  • Create JWT refresh token (7 days)       │
│  • Store session in DB + Redis             │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          7. Update User Activity           │
│  • Set lastLogin timestamp                 │
│  • Record IP and device info               │
│  • Reset loginAttempts to 0                │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────┐
│          8. Audit Log                      │
│  • Log successful login                    │
│  • Record device and location              │
└────────────────┬───────────────────────────┘
                 │
                 ↓
┌─────────┐
│ Client  │ { accessToken, refreshToken, user }
└─────────┘
```

### 4. Web3 Wallet Authentication Flow

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ 1. Click "Connect Wallet"
       ↓
┌─────────────────────────────────────────────┐
│  Frontend: Detect Wallet (MetaMask, etc)   │
│  • Check if wallet installed                │
│  • Request account access                   │
└──────┬──────────────────────────────────────┘
       │
       │ 2. Get wallet address
       ↓
┌─────────────────────────────────────────────┐
│  POST /api/auth/wallet/nonce                │
│  { address, chain }                         │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────────────┐
│  Backend: Generate Nonce                    │
│  • Find or create user by wallet address    │
│  • Generate random nonce (UUID)             │
│  • Store nonce in user.wallets[].nonce      │
│  • Return nonce to client                   │
└──────┬──────────────────────────────────────┘
       │
       │ { nonce: "abc123..." }
       ↓
┌─────────────────────────────────────────────┐
│  Frontend: Sign Message                     │
│  • Create message:                          │
│    "Sign this message to authenticate       │
│     with Freelance402.                      │
│     Nonce: abc123..."                       │
│  • Request wallet signature                 │
└──────┬──────────────────────────────────────┘
       │
       │ User approves in wallet
       ↓
┌─────────────────────────────────────────────┐
│  POST /api/auth/wallet/verify               │
│  { address, signature, chain }              │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────────────┐
│  Backend: Verify Signature                  │
│  1. Reconstruct signed message with nonce   │
│  2. Recover signer address from signature   │
│  3. Verify recovered address == provided    │
│  4. Verify nonce hasn't been used           │
│  5. Check signature timestamp (5 min max)   │
└──────┬──────────────────────────────────────┘
       │
       │ Valid?
       ↓
┌─────────────────────────────────────────────┐
│  Process Authentication                     │
│  • Mark nonce as used                       │
│  • Update wallet.verifiedAt                 │
│  • Generate JWT tokens                      │
│  • Create session                           │
│  • Link to X402 wallet if applicable        │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────┐
│   Client    │ { accessToken, refreshToken, user }
└─────────────┘
```

**Wallet Signature Verification (Ethereum Example)**:

```javascript
// Message format
const message = `Sign this message to authenticate with Freelance402.

This request will not trigger any blockchain transaction or cost any gas fees.

Wallet address: ${address}
Nonce: ${nonce}
Timestamp: ${timestamp}`;

// Verification
const signerAddress = ethers.utils.verifyMessage(message, signature);
const isValid = signerAddress.toLowerCase() === address.toLowerCase();
```

### 5. OAuth 2.0 Flow (Google Example)

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ 1. Click "Sign in with Google"
       ↓
┌─────────────────────────────────────────────┐
│  GET /api/auth/google                       │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────────────┐
│  Redirect to Google OAuth                   │
│  • client_id                                │
│  • redirect_uri                             │
│  • scope: email, profile                    │
│  • state: random token (CSRF protection)    │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────────────┐
│  User authenticates with Google             │
│  • Enter Google credentials                 │
│  • Approve permissions                      │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────────────┐
│  Google Redirects Back                      │
│  GET /api/auth/google/callback              │
│  ?code=xxx&state=yyy                        │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────────────┐
│  Backend: Exchange Code for Token           │
│  • Verify state matches                     │
│  • POST to Google token endpoint            │
│  • Receive access_token + id_token          │
│  • Decode id_token (JWT)                    │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────────────┐
│  Find or Create User                        │
│  • Extract email from id_token              │
│  • Check if user exists by email            │
│  • If exists: link Google account           │
│  • If not: create new user                  │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────────────┐
│  Generate JWT Tokens                        │
│  • Create session                           │
│  • Return tokens to client                  │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────┐
│   Client    │ Redirect to app with tokens
└─────────────┘
```

### 6. Magic Link Flow

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ POST /api/auth/magic-link
       │ { email, purpose: 'login' }
       ↓
┌─────────────────────────────────────────────┐
│  Backend: Generate Magic Link              │
│  • Generate secure token (32 bytes)         │
│  • Hash and store in magic_links            │
│  • Set expiry (15 minutes)                  │
│  • Send email with link                     │
└──────┬──────────────────────────────────────┘
       │
       │ Email sent
       ↓
┌─────────────┐
│    User     │ Clicks link in email
└──────┬──────┘
       │
       │ GET /api/auth/magic-link/verify?token=xxx
       ↓
┌─────────────────────────────────────────────┐
│  Backend: Verify Token                      │
│  • Find token in DB                         │
│  • Check not expired                        │
│  • Check not already used                   │
│  • Verify attempts < max                    │
└──────┬──────────────────────────────────────┘
       │
       │ Valid?
       ↓
┌─────────────────────────────────────────────┐
│  Authenticate User                          │
│  • Mark token as used                       │
│  • Generate JWT tokens                      │
│  • Create session                           │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────┐
│   Client    │ Redirect to app with tokens
└─────────────┘
```

### 7. 2FA/MFA Flow

```
After successful password verification:

┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       │ POST /api/auth/2fa/verify
       │ { tempToken, code, method: 'totp' }
       ↓
┌─────────────────────────────────────────────┐
│  Backend: Verify Code                       │
│  • Validate tempToken                       │
│  • Check 2FA method                         │
│                                             │
│  For TOTP:                                  │
│  • Generate TOTP from secret                │
│  • Compare with provided code               │
│  • Allow 30-second window                   │
│                                             │
│  For SMS:                                   │
│  • Lookup code in verification_codes        │
│  • Verify not expired                       │
│  • Compare hashed code                      │
└──────┬──────────────────────────────────────┘
       │
       │ Valid?
       ↓
┌─────────────────────────────────────────────┐
│  Complete Authentication                    │
│  • Mark MFA verified in session             │
│  • Generate full JWT tokens                 │
│  • Return to client                         │
└──────┬──────────────────────────────────────┘
       │
       ↓
┌─────────────┐
│   Client    │ { accessToken, refreshToken }
└─────────────┘
```

---

## 🔐 JWT Token Structure

### Access Token

```javascript
{
  // Standard claims
  iss: "https://freelance402.com",
  sub: "user_id_here",
  aud: "freelance402-api",
  exp: 1234567890, // 15 minutes from issue
  iat: 1234567000,
  jti: "unique_token_id",

  // Custom claims
  email: "user@example.com",
  role: "contractor",
  tier: "pro",
  permissions: ["create_proposal", "submit_review"],

  // Session info
  sid: "session_id",
  device: "web",

  // Security
  type: "access",
  v: 1 // Token version for revocation
}
```

### Refresh Token

```javascript
{
  iss: "https://freelance402.com",
  sub: "user_id_here",
  aud: "freelance402-api",
  exp: 1234567890, // 7 days from issue
  iat: 1234567000,
  jti: "unique_token_id",

  // Minimal claims for security
  type: "refresh",
  family: "token_family_id", // For rotation detection
  v: 1
}
```

---

## 🛡️ Security Implementation

### Password Requirements

```javascript
const passwordRequirements = {
  minLength: 8,
  maxLength: 128,
  requireUppercase: true,
  requireLowercase: true,
  requireNumber: true,
  requireSpecial: true,
  specialChars: "!@#$%^&*()_+-=[]{}|;:,.<>?",
  preventCommon: true, // Check against common password list
  preventUserInfo: true, // Don't allow email, name, etc.
  preventReuse: 5 // Don't allow last 5 passwords
};

function validatePassword(password, user) {
  // Length check
  if (password.length < 8 || password.length > 128) {
    return { valid: false, error: "Password must be 8-128 characters" };
  }

  // Complexity checks
  if (!/[a-z]/.test(password)) {
    return { valid: false, error: "Password must contain lowercase letter" };
  }
  if (!/[A-Z]/.test(password)) {
    return { valid: false, error: "Password must contain uppercase letter" };
  }
  if (!/[0-9]/.test(password)) {
    return { valid: false, error: "Password must contain number" };
  }
  if (!/[!@#$%^&*()_+\-=\[\]{}|;:,.<>?]/.test(password)) {
    return { valid: false, error: "Password must contain special character" };
  }

  // Check against common passwords
  if (isCommonPassword(password)) {
    return { valid: false, error: "Password is too common" };
  }

  // Check against user info
  const userInfo = [user.email, user.firstName, user.lastName].filter(Boolean);
  for (const info of userInfo) {
    if (password.toLowerCase().includes(info.toLowerCase())) {
      return { valid: false, error: "Password cannot contain personal information" };
    }
  }

  return { valid: true };
}
```

### Rate Limiting Strategy

```javascript
// Rate limit configurations
const rateLimits = {
  // Auth endpoints
  login: {
    windowMs: 60 * 1000, // 1 minute
    max: 5, // 5 attempts per minute per IP
    skipSuccessfulRequests: true
  },

  register: {
    windowMs: 60 * 60 * 1000, // 1 hour
    max: 3, // 3 registrations per hour per IP
    skipSuccessfulRequests: false
  },

  passwordReset: {
    windowMs: 60 * 60 * 1000, // 1 hour
    max: 3, // 3 reset requests per hour per email
    skipSuccessfulRequests: false
  },

  magicLink: {
    windowMs: 60 * 60 * 1000, // 1 hour
    max: 5, // 5 magic links per hour per email
    skipSuccessfulRequests: false
  },

  walletNonce: {
    windowMs: 60 * 1000, // 1 minute
    max: 10, // 10 nonce requests per minute per address
    skipSuccessfulRequests: true
  },

  // General API
  api: {
    windowMs: 60 * 1000, // 1 minute
    max: 100, // 100 requests per minute per user
    skipSuccessfulRequests: false
  }
};
```

### Device Fingerprinting

```javascript
// Client-side fingerprinting
async function generateDeviceFingerprint() {
  const components = {
    userAgent: navigator.userAgent,
    language: navigator.language,
    colorDepth: screen.colorDepth,
    deviceMemory: navigator.deviceMemory,
    hardwareConcurrency: navigator.hardwareConcurrency,
    screenResolution: `${screen.width}x${screen.height}`,
    timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
    platform: navigator.platform,
    touchSupport: 'ontouchstart' in window,
    canvas: await getCanvasFingerprint(),
    webgl: await getWebGLFingerprint()
  };

  // Hash components to create fingerprint
  const fingerprint = await sha256(JSON.stringify(components));
  return fingerprint;
}
```

### Session Security

```javascript
// Session validation middleware
async function validateSession(accessToken) {
  // 1. Verify JWT signature
  const decoded = jwt.verify(accessToken, JWT_SECRET);

  // 2. Check token not expired
  if (decoded.exp < Date.now() / 1000) {
    throw new Error('Token expired');
  }

  // 3. Check session exists in database
  const session = await Session.findOne({
    userId: decoded.sub,
    accessToken: hashToken(accessToken),
    status: 'active'
  });

  if (!session) {
    throw new Error('Invalid session');
  }

  // 4. Check device fingerprint matches
  const currentFingerprint = extractFingerprint(request);
  if (session.device.fingerprint !== currentFingerprint) {
    // Suspicious activity - require re-authentication
    await flagSuspiciousActivity(session);
    throw new Error('Device mismatch');
  }

  // 5. Check user still active
  const user = await User.findById(decoded.sub);
  if (!user || user.status !== 'active') {
    throw new Error('User inactive');
  }

  // 6. Update last activity
  await session.updateOne({
    lastActivity: new Date(),
    $inc: { activityCount: 1 }
  });

  return { user, session, decoded };
}
```

### Refresh Token Rotation

```javascript
// Refresh token rotation with breach detection
async function refreshAccessToken(refreshToken) {
  // 1. Verify refresh token
  const decoded = jwt.verify(refreshToken, REFRESH_TOKEN_SECRET);

  // 2. Find session
  const session = await Session.findOne({
    userId: decoded.sub,
    refreshToken: hashToken(refreshToken),
    tokenFamily: decoded.family
  });

  if (!session) {
    // Token reuse detected - possible breach!
    await revokeAllSessionsInFamily(decoded.family);
    await alertUser(decoded.sub, 'Possible token theft detected');
    throw new Error('Token reuse detected');
  }

  // 3. Generate new token pair
  const newAccessToken = generateAccessToken(user);
  const newRefreshToken = generateRefreshToken(user, decoded.family);

  // 4. Update session with new tokens
  await session.updateOne({
    accessToken: hashToken(newAccessToken),
    refreshToken: hashToken(newRefreshToken),
    accessTokenExpiry: new Date(Date.now() + 15 * 60 * 1000),
    refreshTokenExpiry: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
  });

  // 5. Return new tokens
  return {
    accessToken: newAccessToken,
    refreshToken: newRefreshToken
  };
}
```

---

## 🔌 API Endpoints

### Authentication

```
POST   /api/auth/register                    # Register with email/password
POST   /api/auth/login                       # Login with email/password
POST   /api/auth/logout                      # Logout (revoke session)
POST   /api/auth/refresh                     # Refresh access token
POST   /api/auth/verify-email                # Verify email with token

### Password Management
POST   /api/auth/forgot-password             # Request password reset
POST   /api/auth/reset-password              # Reset password with token
POST   /api/auth/change-password             # Change password (authenticated)

### Magic Link
POST   /api/auth/magic-link                  # Request magic link
GET    /api/auth/magic-link/verify           # Verify magic link

### Wallet Authentication
POST   /api/auth/wallet/nonce                # Get nonce for wallet
POST   /api/auth/wallet/verify               # Verify wallet signature
POST   /api/auth/wallet/link                 # Link wallet to existing account
DELETE /api/auth/wallet/:address             # Unlink wallet

### OAuth 2.0
GET    /api/auth/google                      # Initiate Google OAuth
GET    /api/auth/google/callback             # Google OAuth callback
GET    /api/auth/github                      # Initiate GitHub OAuth
GET    /api/auth/github/callback             # GitHub OAuth callback
GET    /api/auth/linkedin                    # Initiate LinkedIn OAuth
GET    /api/auth/linkedin/callback           # LinkedIn OAuth callback
DELETE /api/auth/social/:provider            # Unlink social account

### Two-Factor Authentication
POST   /api/auth/2fa/enable                  # Enable 2FA
POST   /api/auth/2fa/verify                  # Verify 2FA code
POST   /api/auth/2fa/disable                 # Disable 2FA
POST   /api/auth/2fa/regenerate-backup       # Regenerate backup codes
POST   /api/auth/2fa/verify-login            # Verify 2FA during login

### Session Management
GET    /api/auth/sessions                    # List all active sessions
DELETE /api/auth/sessions/:id                # Revoke specific session
DELETE /api/auth/sessions                    # Revoke all sessions except current

### Account Verification
POST   /api/auth/resend-verification         # Resend verification email
POST   /api/auth/verify-phone                # Send phone verification code
POST   /api/auth/confirm-phone               # Confirm phone with code
```

---

## 💻 Implementation Examples

### Backend: Registration Endpoint (Node.js + TypeScript)

```typescript
// src/controllers/auth.controller.ts

import { Request, Response } from 'express';
import bcrypt from 'bcryptjs';
import { z } from 'zod';
import { User } from '../models/User.model';
import { generateVerificationToken, sendVerificationEmail } from '../services/email.service';
import { AuditLog } from '../models/AuditLog.model';

// Validation schema
const registerSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(128),
  firstName: z.string().min(1).max(50),
  lastName: z.string().min(1).max(50),
  role: z.enum(['client', 'contractor']),
  acceptedTerms: z.boolean().refine(val => val === true)
});

export async function register(req: Request, res: Response) {
  try {
    // 1. Validate input
    const data = registerSchema.parse(req.body);

    // 2. Validate password strength
    const passwordValidation = validatePasswordStrength(data.password, data);
    if (!passwordValidation.valid) {
      return res.status(400).json({
        error: 'Invalid password',
        message: passwordValidation.error
      });
    }

    // 3. Check if email already exists
    const existingUser = await User.findOne({ email: data.email.toLowerCase() });
    if (existingUser) {
      // Don't reveal if email exists (security)
      return res.status(400).json({
        error: 'Registration failed',
        message: 'If this email is valid, you will receive a verification link'
      });
    }

    // 4. Hash password
    const salt = await bcrypt.genSalt(12);
    const passwordHash = await bcrypt.hash(data.password, salt);

    // 5. Create user
    const user = await User.create({
      email: data.email.toLowerCase(),
      passwordHash,
      profile: {
        firstName: data.firstName,
        lastName: data.lastName,
        displayName: `${data.firstName} ${data.lastName}`
      },
      role: data.role,
      status: 'pending_verification',
      emailVerified: false,
      subscription: {
        tier: 'free',
        isActive: true
      },
      activity: {
        createdIp: req.ip,
        lastLogin: new Date()
      }
    });

    // 6. Generate verification token
    const verificationToken = await generateVerificationToken(user._id);

    // 7. Send verification email
    await sendVerificationEmail(
      user.email,
      user.profile.firstName,
      verificationToken
    );

    // 8. Audit log
    await AuditLog.create({
      userId: user._id,
      action: 'register',
      category: 'auth',
      details: {
        method: 'email',
        success: true
      },
      context: {
        ip: req.ip,
        userAgent: req.headers['user-agent']
      },
      severity: 'info',
      timestamp: new Date()
    });

    // 9. Return success (don't auto-login)
    return res.status(201).json({
      success: true,
      message: 'Registration successful! Please check your email to verify your account.',
      userId: user._id
    });

  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.errors
      });
    }

    console.error('Registration error:', error);
    return res.status(500).json({
      error: 'Registration failed',
      message: 'An error occurred during registration'
    });
  }
}
```

### Backend: Wallet Authentication (Node.js + TypeScript)

```typescript
// src/controllers/wallet-auth.controller.ts

import { Request, Response } from 'express';
import { ethers } from 'ethers';
import { v4 as uuidv4 } from 'uuid';
import { User } from '../models/User.model';
import { generateTokenPair } from '../services/jwt.service';
import { createSession } from '../services/session.service';

export async function getWalletNonce(req: Request, res: Response) {
  try {
    const { address, chain } = req.body;

    // Validate address format
    if (!ethers.utils.isAddress(address)) {
      return res.status(400).json({
        error: 'Invalid wallet address'
      });
    }

    // Find or create user by wallet address
    let user = await User.findOne({
      'wallets.address': address.toLowerCase()
    });

    if (!user) {
      // New wallet - create pending user
      user = await User.create({
        status: 'pending_verification',
        role: 'contractor', // Default, can be changed
        subscription: {
          tier: 'free',
          isActive: true
        },
        wallets: [{
          address: address.toLowerCase(),
          chain,
          isPrimary: true,
          isVerified: false,
          nonce: uuidv4()
        }]
      });
    } else {
      // Update nonce for existing wallet
      const walletIndex = user.wallets.findIndex(
        w => w.address === address.toLowerCase()
      );

      user.wallets[walletIndex].nonce = uuidv4();
      user.wallets[walletIndex].lastUsed = new Date();
      await user.save();
    }

    // Get the nonce
    const wallet = user.wallets.find(w => w.address === address.toLowerCase());

    return res.json({
      nonce: wallet.nonce,
      message: `Sign this message to authenticate with Freelance402.\n\nThis request will not trigger any blockchain transaction or cost any gas fees.\n\nWallet address: ${address}\nNonce: ${wallet.nonce}\nTimestamp: ${Date.now()}`
    });

  } catch (error) {
    console.error('Get nonce error:', error);
    return res.status(500).json({
      error: 'Failed to generate nonce'
    });
  }
}

export async function verifyWalletSignature(req: Request, res: Response) {
  try {
    const { address, signature, chain } = req.body;

    // Find user by wallet address
    const user = await User.findOne({
      'wallets.address': address.toLowerCase()
    });

    if (!user) {
      return res.status(404).json({
        error: 'Wallet not found'
      });
    }

    // Get wallet and nonce
    const wallet = user.wallets.find(w => w.address === address.toLowerCase());

    if (!wallet || !wallet.nonce) {
      return res.status(400).json({
        error: 'No nonce found for this wallet'
      });
    }

    // Reconstruct message
    const message = `Sign this message to authenticate with Freelance402.\n\nThis request will not trigger any blockchain transaction or cost any gas fees.\n\nWallet address: ${address}\nNonce: ${wallet.nonce}\nTimestamp: ${Date.now()}`;

    // Verify signature
    let signerAddress: string;

    try {
      if (chain === 'ethereum' || chain === 'polygon') {
        signerAddress = ethers.utils.verifyMessage(message, signature);
      } else if (chain === 'solana') {
        // Solana verification (different library)
        signerAddress = await verifySolanaSignature(message, signature, address);
      } else {
        return res.status(400).json({
          error: 'Unsupported chain'
        });
      }
    } catch (error) {
      return res.status(401).json({
        error: 'Invalid signature'
      });
    }

    // Verify signer matches provided address
    if (signerAddress.toLowerCase() !== address.toLowerCase()) {
      return res.status(401).json({
        error: 'Signature verification failed'
      });
    }

    // Mark wallet as verified and clear nonce
    wallet.isVerified = true;
    wallet.verifiedAt = new Date();
    wallet.nonce = null; // Clear nonce after use
    wallet.lastUsed = new Date();

    // Update user status
    if (user.status === 'pending_verification') {
      user.status = 'active';
    }

    await user.save();

    // Check if wallet is linked to X402
    await linkX402WalletIfApplicable(user, address, chain);

    // Generate JWT tokens
    const tokens = await generateTokenPair(user);

    // Create session
    const session = await createSession({
      userId: user._id,
      tokens,
      device: {
        type: 'web',
        fingerprint: req.body.deviceFingerprint,
        userAgent: req.headers['user-agent']
      },
      location: {
        ip: req.ip
      },
      authMethod: 'wallet_signature',
      authProvider: chain
    });

    // Return tokens and user info
    return res.json({
      accessToken: tokens.accessToken,
      refreshToken: tokens.refreshToken,
      user: {
        id: user._id,
        email: user.email,
        role: user.role,
        tier: user.subscription.tier,
        wallet: {
          address: wallet.address,
          chain: wallet.chain
        }
      }
    });

  } catch (error) {
    console.error('Verify signature error:', error);
    return res.status(500).json({
      error: 'Authentication failed'
    });
  }
}
```

### Frontend: Wallet Connection (React + TypeScript)

```typescript
// src/hooks/useWalletAuth.ts

import { useState } from 'react';
import { ethers } from 'ethers';
import { useAuth } from './useAuth';
import api from '../services/api';

export function useWalletAuth() {
  const [isConnecting, setIsConnecting] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const { setTokens, setUser } = useAuth();

  async function connectWallet() {
    setIsConnecting(true);
    setError(null);

    try {
      // 1. Check if MetaMask is installed
      if (!window.ethereum) {
        throw new Error('Please install MetaMask to continue');
      }

      // 2. Request account access
      const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
      });

      const address = accounts[0];
      const chain = 'ethereum';

      // 3. Get nonce from backend
      const nonceResponse = await api.post('/api/auth/wallet/nonce', {
        address,
        chain
      });

      const { nonce, message } = nonceResponse.data;

      // 4. Sign message with wallet
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      const signer = provider.getSigner();
      const signature = await signer.signMessage(message);

      // 5. Verify signature on backend
      const authResponse = await api.post('/api/auth/wallet/verify', {
        address,
        signature,
        chain,
        deviceFingerprint: await generateDeviceFingerprint()
      });

      const { accessToken, refreshToken, user } = authResponse.data;

      // 6. Store tokens and user
      setTokens(accessToken, refreshToken);
      setUser(user);

      return user;

    } catch (err: any) {
      console.error('Wallet connection error:', err);

      if (err.code === 4001) {
        setError('You rejected the connection request');
      } else if (err.code === -32002) {
        setError('Please check MetaMask for a pending connection request');
      } else {
        setError(err.message || 'Failed to connect wallet');
      }

      throw err;
    } finally {
      setIsConnecting(false);
    }
  }

  return {
    connectWallet,
    isConnecting,
    error
  };
}
```

### Frontend: Auth Component

```typescript
// src/components/auth/LoginModal.tsx

import React, { useState } from 'react';
import { useWalletAuth } from '../../hooks/useWalletAuth';
import { useEmailAuth } from '../../hooks/useEmailAuth';

export function LoginModal({ onClose }: { onClose: () => void }) {
  const [mode, setMode] = useState<'email' | 'wallet'>('email');
  const { connectWallet, isConnecting: isConnectingWallet } = useWalletAuth();
  const { login, isLoading: isLoggingIn } = useEmailAuth();

  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  async function handleEmailLogin(e: React.FormEvent) {
    e.preventDefault();

    try {
      await login(email, password);
      onClose();
    } catch (error) {
      console.error('Login failed:', error);
    }
  }

  async function handleWalletConnect() {
    try {
      await connectWallet();
      onClose();
    } catch (error) {
      console.error('Wallet connection failed:', error);
    }
  }

  return (
    <div className="modal">
      <div className="modal-content">
        <h2>Sign In</h2>

        {/* Toggle between email and wallet */}
        <div className="auth-mode-toggle">
          <button
            className={mode === 'email' ? 'active' : ''}
            onClick={() => setMode('email')}
          >
            Email
          </button>
          <button
            className={mode === 'wallet' ? 'active' : ''}
            onClick={() => setMode('wallet')}
          >
            Wallet
          </button>
        </div>

        {mode === 'email' ? (
          <form onSubmit={handleEmailLogin}>
            <input
              type="email"
              placeholder="Email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              required
            />
            <input
              type="password"
              placeholder="Password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              required
            />
            <button type="submit" disabled={isLoggingIn}>
              {isLoggingIn ? 'Signing in...' : 'Sign In'}
            </button>
          </form>
        ) : (
          <div className="wallet-auth">
            <button
              onClick={handleWalletConnect}
              disabled={isConnectingWallet}
              className="wallet-connect-btn"
            >
              {isConnectingWallet ? (
                'Connecting...'
              ) : (
                <>
                  <img src="/metamask-icon.svg" alt="MetaMask" />
                  Connect with MetaMask
                </>
              )}
            </button>

            <p className="wallet-info">
              Sign a message to prove you own this wallet.
              No transaction or gas fees required.
            </p>
          </div>
        )}

        {/* Social login options */}
        <div className="social-login">
          <p>Or continue with</p>
          <div className="social-buttons">
            <button onClick={() => window.location.href = '/api/auth/google'}>
              <img src="/google-icon.svg" alt="Google" />
              Google
            </button>
            <button onClick={() => window.location.href = '/api/auth/github'}>
              <img src="/github-icon.svg" alt="GitHub" />
              GitHub
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}
```

---

## 🔄 Integration with X402 Protocol

### Linking X402 Wallet

When a user authenticates with a Web3 wallet, automatically check if it's X402-compatible and link it for payments:

```typescript
// src/services/x402.service.ts

import { X402Client } from '@x402/sdk';

export async function linkX402WalletIfApplicable(
  user: IUser,
  walletAddress: string,
  chain: string
) {
  // Check if wallet is X402-compatible
  const x402Client = new X402Client();
  const isCompatible = await x402Client.isWalletCompatible(walletAddress, chain);

  if (!isCompatible) {
    return false;
  }

  // Get wallet balance
  const balance = await x402Client.getBalance(walletAddress);

  // Link wallet to user
  user.x402Wallet = {
    address: walletAddress,
    verified: true,
    balance,
    publicKey: await x402Client.getPublicKey(walletAddress),
    isCustodial: false
  };

  await user.save();

  return true;
}
```

### Custodial X402 Wallet Creation

For users who don't have a Web3 wallet, create a custodial X402 wallet:

```typescript
export async function createCustodialX402Wallet(userId: string) {
  const x402Client = new X402Client();

  // Generate new wallet
  const wallet = await x402Client.createWallet();

  // Encrypt private key
  const encryptedPrivateKey = await encryptWithKMS(wallet.privateKey, userId);

  // Update user
  await User.findByIdAndUpdate(userId, {
    x402Wallet: {
      address: wallet.address,
      verified: true,
      balance: 0,
      publicKey: wallet.publicKey,
      encryptedPrivateKey,
      isCustodial: true
    }
  });

  return wallet.address;
}
```

---

## 📱 Multi-Device Support

### Device Management

Users can view and manage all devices with active sessions:

```typescript
// GET /api/auth/sessions

export async function listSessions(req: Request, res: Response) {
  const userId = req.user._id;
  const currentSessionId = req.session._id;

  const sessions = await Session.find({
    userId,
    status: 'active',
    expiresAt: { $gt: new Date() }
  }).sort({ lastActivity: -1 });

  const sessionsWithDetails = sessions.map(session => ({
    id: session._id,
    device: {
      type: session.device.type,
      os: session.device.os,
      browser: session.device.browser
    },
    location: {
      city: session.location.city,
      country: session.location.country
    },
    lastActivity: session.lastActivity,
    isCurrent: session._id.equals(currentSessionId),
    isTrusted: session.isTrusted
  }));

  return res.json({ sessions: sessionsWithDetails });
}
```

---

## ✅ Next Steps

This authentication architecture is now ready to be integrated into the Freelance402 platform. The system supports:

✅ Traditional email/password authentication
✅ Web3 wallet authentication (Ethereum, Solana, X402)
✅ OAuth 2.0 social login (Google, GitHub, LinkedIn)
✅ Magic link passwordless authentication
✅ Two-factor authentication (TOTP, SMS)
✅ Refresh token rotation with breach detection
✅ Device fingerprinting and management
✅ Comprehensive audit logging
✅ X402 protocol integration for payments
✅ Multi-device session management

**Ready to implement! 🚀**
