# အခန်း ၂၉: Authentication နှင့် Identity စီမံခန့်ခွဲမှုစနစ် (Okta / Auth0)

## နိဒါန်း

Authentication နှင့် Identity Management သည် modern enterprise systems ၏ security foundation ဖြစ်သည်။ Okta, Auth0, Ping Identity ကဲ့သို့သော Identity-as-a-Service (IDaaS) providers များသည် ကုမ္ပဏီများ၏ authentication complexity ကို abstraction ပြုလုပ်ပေးသည်။ ဤအခန်းတွင် SSO (Single Sign-On), MFA (Multi-Factor Authentication), OAuth2, OpenID Connect, JWT, token lifecycle management, နှင့် multi-tenant identity design တို့ကို production-grade microservices perspective မှ လေ့လာမည်ဖြစ်သည်။

Authentication မှာ "ဤ user ဘယ်သူလဲ" ဟု verify ပြုလုပ်ခြင်းဖြစ်ပြီး Authorization မှာ "ဤ user သည် ဘာလုပ်နိုင်သနည်း" ဟု check ပြုလုပ်ခြင်းဖြစ်သည်။ ဤ system သည် သန်းချီသော users ၏ identity lifecycle ကို secure, scalable နှင့် user-friendly ဖြစ်အောင် manage ပြုလုပ်ရမည်ဖြစ်သည်။

---

## ၂၉.၁ Requirements

### Functional Requirements

- **SSO (Single Sign-On)**: User တစ်ကြိမ် login ပြုလုပ်ခြင်းဖြင့် multiple applications ကို access ပြုနိုင်ရမည်
- **MFA (Multi-Factor Authentication)**: TOTP, SMS, Email, Push notification, Hardware keys (FIDO2/WebAuthn)
- **Social Login**: Google, GitHub, Facebook, Apple ID ဖြင့် login ပြုနိုင်ရမည်
- **Machine-to-Machine (M2M)**: Services ချင်း authenticate ပြုနိုင်ရမည် (Client Credentials flow)
- **User Management**: Registration, password reset, profile update
- **Organization/Tenant Management**: Multi-tenant SaaS support

### Non-Functional Requirements

- **Availability**: 99.999% (< 5 minutes downtime/year) — Auth is critical path
- **Latency**: Token validation < 5ms (P99), Login < 500ms
- **Security**: Zero-trust, encrypted tokens, audit logging
- **Compliance**: SOC2, ISO27001, GDPR, HIPAA

### Scale Estimation

```
Users: 100 million total, 10M DAU
Login events: 10M DAU × 3 sessions/day = 30M logins/day
Token validation: 10M users × 100 requests/day = 1B validations/day
  = ~11,574 validations/second (peak: 50,000/second)

Token storage: 
  - 10M active sessions × 1 KB per session = 10 GB
  - Redis with 64 GB RAM → Comfortable headroom

Audit log:
  - 30M events/day × 500 bytes = 15 GB/day = 5.4 TB/year
```

---

## ၂၉.၂ Domain Decomposition

### ၂၉.၂.၁ Identity Service

User identity ၏ source of truth ဖြစ်သည်။

**Responsibilities:**
- User registration, profile management
- Password hashing (bcrypt/Argon2)
- Social identity linking (external IdP)
- Organization/tenant management
- Directory services (LDAP/AD sync)

### ၂၉.၂.၂ Token Service

Authentication tokens ၏ lifecycle ကို manage ပြုသည်။

**Responsibilities:**
- JWT Access Token issuance
- Refresh Token issuance/rotation
- Token revocation
- JWK (JSON Web Key Set) endpoint (/jwks)
- Token introspection (/introspect)

### ၂၉.၂.၃ Session Service

User sessions ကို track ပြုလုပ်သည်။

**Responsibilities:**
- Session creation/termination
- Session binding (device, IP, user-agent)
- Concurrent session limits
- "Remember me" functionality
- Global logout (all sessions)

### ၂၉.၂.၄ MFA Service

Multi-factor authentication challenges ကို handle ပြုသည်။

**Responsibilities:**
- TOTP enrollment/validation (Google Authenticator compatible)
- SMS/Email OTP delivery
- WebAuthn/FIDO2 challenge-response
- Backup codes management
- Risk-based MFA trigger

### ၂၉.၂.၅ Audit Service

Security events ၏ immutable log ကို maintain ပြုသည်။

**Responsibilities:**
- Login/logout events
- Failed authentication attempts
- Token operations
- Admin actions
- Compliance reporting

---

## ၂၉.၃ OAuth2 နှင့် OpenID Connect Flows

### Authorization Code Flow + PKCE (Web/Mobile Apps)

PKCE (Proof Key for Code Exchange) သည် authorization code interception attack ကို prevent ပြုသည်။

```
1. Client generates:
   code_verifier = random 64 bytes (base64url)
   code_challenge = base64url(sha256(code_verifier))

2. Client → Auth Server:
   GET /authorize?
     response_type=code&
     client_id=app123&
     redirect_uri=https://app.com/callback&
     scope=openid profile email&
     state=random_state&
     code_challenge=XYZ...&
     code_challenge_method=S256

3. Auth Server → User: Login page

4. User authenticates (password + MFA)

5. Auth Server → Client:
   HTTP 302 Location: https://app.com/callback?
     code=auth_code_123&
     state=random_state

6. Client → Auth Server:
   POST /token
   {
     "grant_type": "authorization_code",
     "code": "auth_code_123",
     "redirect_uri": "https://app.com/callback",
     "code_verifier": "original_verifier",  // PKCE verification
     "client_id": "app123"
   }

7. Auth Server verifies: sha256(code_verifier) == code_challenge

8. Auth Server → Client:
   {
     "access_token": "eyJ...",  // JWT, 15 min expiry
     "refresh_token": "ref_tok_...",  // Opaque, 30 day expiry
     "id_token": "eyJ...",  // OIDC identity token
     "token_type": "Bearer",
     "expires_in": 900
   }
```

### Client Credentials Flow (Machine-to-Machine)

```
Service A → Auth Server:
  POST /token
  {
    "grant_type": "client_credentials",
    "client_id": "service-a-id",
    "client_secret": "secret-or-JWT-assertion",
    "scope": "service-b:read service-b:write"
  }

Auth Server → Service A:
  {
    "access_token": "eyJ...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "scope": "service-b:read service-b:write"
  }

Service A → Service B:
  Authorization: Bearer eyJ...
```

### Device Flow (Smart TV, CLI tools)

```
1. Device → Auth Server: POST /device/authorize
   Response: {device_code, user_code: "ABCD-1234", 
              verification_uri: "auth.example.com/device"}

2. Device: Display "Go to auth.example.com/device, enter ABCD-1234"

3. User: Opens browser, enters code, authenticates

4. Device: Poll POST /token {grant_type: "device_code", device_code}
   Until: success | expired | denied
```

---

## ၂၉.၄ JWT vs Opaque Token Trade-offs

### JWT (JSON Web Token) Access Token

```
Header.Payload.Signature

Header: {"alg": "RS256", "typ": "JWT", "kid": "key-id-1"}
Payload: {
  "sub": "user-uuid-123",
  "iss": "https://auth.company.com",
  "aud": "https://api.company.com",
  "exp": 1712345678,
  "iat": 1712344778,
  "scope": "read write",
  "org_id": "org-456",
  "roles": ["admin"]
}
Signature: RSASHA256(base64(header) + "." + base64(payload), private_key)
```

**ဂုဏ်သတ္တိများ:**
- **Stateless**: Resource Server သည် database lookup မလုပ်ဘဲ signature verify ပြုသည်
- **Self-contained**: User info embedded ဖြစ်သည်
- **Problem**: Revocation မဆောင်ရွက်နိုင် (expiry ထိ valid ဖြစ်နေသည်)

### Opaque Token

```
Token: "tok_live_abc123xyz..." (random string)
Stored in: Token Service database

Validation:
  Resource Server → Token Service: POST /introspect
  {"token": "tok_live_abc123xyz..."}
  
  Token Service → Resource Server:
  {"active": true, "sub": "user-123", "scope": "read", "exp": 1712345678}
```

**ဂုဏ်သတ္တိများ:**
- **Revocable**: Token delete ပြုလုပ်လျှင် immediately invalid
- **No information leakage**: Token မှ user info မဖတ်နိုင်
- **Problem**: Every request တွင် Token Service call လိုသည် (latency)

### Hybrid Approach (Production Best Practice)

```
Access Token: Short-lived JWT (15 minutes)
  - Validates locally (no round-trip)
  - If compromised: Max 15 min exposure

Refresh Token: Long-lived Opaque (30 days)
  - Stored in Token Service DB
  - Revocable immediately
  - Rotation: Each use generates new refresh token

Token Revocation:
  - Maintain "jti" (JWT ID) blocklist in Redis
  - TTL = token expiry time
  - Check blocklist on critical operations
```

---

## ၂၉.၅ Token Introspection နှင့် Revocation at Scale

### Distributed Token Validation

```
Problem: 50,000 token validations/second at scale

Solution 1 (JWT + Local Validation):
  - Resource services cache JWKS (JSON Web Key Set) locally
  - Validate signature locally: 1ms (no network)
  - Refresh JWKS every 1 hour or on "kid" cache miss

Solution 2 (Token Introspection with Caching):
  - Cache introspection results in local Redis
  - TTL = min(token_remaining_life, 60 seconds)
  - Cache hit: 1ms, Cache miss + introspect: 10ms

JWKS Endpoint:
  GET /jwks
  {
    "keys": [
      {
        "kty": "RSA",
        "kid": "key-id-1",  // Key ID for rotation
        "use": "sig",
        "alg": "RS256",
        "n": "...",  // Public key modulus
        "e": "AQAB"  // Public exponent
      }
    ]
  }
```

### Key Rotation

```
Key rotation strategy:
  1. Generate new key pair (key-id-2)
  2. Publish key-id-2 to JWKS (keep key-id-1)
  3. Start issuing tokens signed with key-id-2
  4. After all key-id-1 tokens expire: Remove key-id-1

Rotation frequency: Every 90 days (or on compromise)
Emergency rotation: Immediate new key, old key revoke
```

---

## ၂၉.၆ Multi-Tenant Identity Design

### Tenant Isolation Models

```
Model 1: Shared Database, Tenant Column
  CREATE TABLE users (
    user_id    UUID PRIMARY KEY,
    tenant_id  UUID NOT NULL,  ← Row-Level Security
    email      TEXT,
    ...
  );
  -- Row Level Security (PostgreSQL):
  ALTER TABLE users ENABLE ROW LEVEL SECURITY;
  CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

Model 2: Schema per Tenant
  tenant_acme.users (table)
  tenant_globex.users (table)
  -- Pros: Better isolation
  -- Cons: Schema management overhead

Model 3: Database per Tenant (Enterprise)
  -- Best isolation, highest cost
  -- Used for highly regulated industries
```

### Tenant-Scoped JWT

```json
{
  "sub": "user-uuid-123",
  "tenant_id": "tenant-acme",
  "tenant_domain": "acme.auth.company.com",
  "org_roles": ["admin"],
  "permissions": ["users:read", "billing:write"],
  "custom_claims": {
    "department": "Engineering",
    "cost_center": "CC-100"
  }
}
```

---

## ၂၉.၇ Session Management — Stateless vs Stateful

### Stateless Session (JWT-based)

```
Pros:
  - No server-side session storage
  - Horizontally scalable
  - Works across microservices

Cons:
  - Cannot forcefully logout (until token expiry)
  - Large token payload (JWT size)
  - Token theft = session hijack until expiry
```

### Stateful Session (Session ID + Redis)

```
Flow:
  1. Login → Server creates session in Redis
     Key: session:{session_id}
     Value: {user_id, ip, user_agent, created_at, last_active}
     TTL: 30 minutes (sliding)
  
  2. Client stores session_id in HttpOnly, Secure cookie
  
  3. Each request: Server validates session_id against Redis
     - IP binding check (optional)
     - Update last_active timestamp

  4. Logout: DEL session:{session_id} → immediate effect
  
  5. Global logout: DEL session:* WHERE user_id = X

Pros:
  - Revocable immediately
  - Fine-grained control

Cons:
  - Redis as critical dependency
  - Cross-service session sharing needed
```

### Risk-Based Authentication

```
Risk signals:
  - New device/browser → Require MFA
  - New country/IP range → Require MFA
  - Unusual time of day → Step-up auth
  - Multiple failed attempts → Account lockout + CAPTCHA
  - Velocity: 10+ logins/hour from same IP → Block

Risk Score calculation:
  risk_score = base_score
    + (new_device ? 30 : 0)
    + (new_country ? 40 : 0)
    + (unusual_time ? 10 : 0)
    + (tor_exit_node ? 80 : 0)
  
  if risk_score > 50: Require MFA
  if risk_score > 80: Block + Alert security team
```

---

## ၂၉.၈ Architecture Diagram နှင့် Data Flow

### System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Client Applications                        │
│         Web App / Mobile App / CLI / IoT Device                  │
└────────┬──────────────────────────────────────────────┬──────────┘
         │                                              │
         │ Auth Requests                                │ Resource API
         │                                              │
┌────────▼──────────────────────────────┐    ┌─────────▼──────────┐
│         Auth Server / AS              │    │   Resource Server   │
│  (OAuth2 + OIDC Authorization Server) │    │   (Microservices)   │
│                                       │    │                     │
│  ┌─────────────┐  ┌───────────────┐   │    │  JWT Validation     │
│  │  Identity   │  │ Token Service │   │    │  (JWKS cached)      │
│  │  Service    │  │ (JWT + Opaque)│   │    │                     │
│  └──────┬──────┘  └───────┬───────┘   │    └─────────────────────┘
│         │                 │           │
│  ┌──────▼──────┐  ┌───────▼───────┐   │
│  │  MFA        │  │  Session      │   │
│  │  Service    │  │  Service      │   │
│  └─────────────┘  └───────┬───────┘   │
│                           │           │
│  ┌────────────────────────▼────────┐  │
│  │          Audit Service          │  │
│  │    (Immutable Event Log)        │  │
│  └─────────────────────────────────┘  │
└───────────────────────────────────────┘
           │                  │
┌──────────▼──────┐  ┌────────▼────────┐
│ PostgreSQL       │  │  Redis Cluster  │
│ (Users, Orgs,    │  │  (Sessions,     │
│  Audit Logs)     │  │   Token Cache,  │
│                  │  │   Revocation)   │
└──────────────────┘  └─────────────────┘
```

### Login Flow (Authorization Code + PKCE)

```
1. User clicks "Login" in web app
2. App generates code_verifier, code_challenge
3. App → Auth Server: GET /authorize?... (with code_challenge)
4. Auth Server → User: Render login page
5. User enters credentials
6. Identity Service: Verify password (bcrypt compare)
7. MFA Service: If MFA required → Send TOTP challenge
8. User enters TOTP code
9. MFA Service: Validate TOTP
10. Auth Server: Issue authorization_code (short-lived, 5 min)
11. Auth Server → App: Redirect with code
12. App → Auth Server: POST /token (code + code_verifier)
13. Auth Server: Verify PKCE (sha256(verifier) == challenge)
14. Token Service: Issue access_token (JWT) + refresh_token
15. Session Service: Create session record
16. Audit Service: Log "successful_login" event
17. Auth Server → App: Return tokens
18. App: Store access_token in memory, refresh_token in HttpOnly cookie

Resource Access:
19. App → Resource Server: API call with Bearer access_token
20. Resource Server: Validate JWT signature with cached JWKS
21. Resource Server: Check token not in revocation blocklist
22. Resource Server: Process request
```

---

## Key Design Decisions နှင့် Trade-offs

| Decision | ရွေးချယ်ချက် | Rationale |
|---|---|---|
| Token format | JWT (short-lived) + Opaque (refresh) | Stateless validation + Revocability |
| PKCE | Required for all flows | Prevent code interception |
| Session storage | Redis | Immediate revocation capability |
| Key algorithm | RS256 (asymmetric) | Resource services need only public key |
| Multi-tenant | Row-Level Security | Balance isolation vs operational simplicity |
| MFA default | Risk-based | UX vs Security balance |

### Security Hardening Checklist

```
Token Security:
  [x] Short-lived access tokens (15 min)
  [x] Refresh token rotation (new token each use)
  [x] Refresh token binding (device fingerprint)
  [x] PKCE for all public clients
  [x] Token revocation list in Redis

Transport Security:
  [x] HTTPS only (HSTS enabled)
  [x] Certificate pinning (mobile apps)
  [x] CORS strict policy

Cookie Security:
  [x] HttpOnly (no JavaScript access)
  [x] Secure (HTTPS only)
  [x] SameSite=Strict (CSRF protection)
  [x] Short Max-Age
```

## အနှစ်ချုပ်

Authentication system ၏ complexity မှာ security, usability, နှင့် scalability ကြားတွင် balance ကို maintain ပြုရခြင်းဖြစ်သည်။ OAuth2 + OIDC standards ကို follow ပြုလုပ်ခြင်းဖြင့် interoperability ကို ရရှိပြီး JWT ၏ stateless nature ဖြင့် 50,000+ validations/second ကို handle ပြုနိုင်သည်။ Multi-tenant design တွင် Row-Level Security ဖြင့် tenant isolation ကို PostgreSQL level တွင် enforce ပြုလုပ်ခြင်းဖြင့် application bugs ကြောင့် data leakage ကို prevent ပြုနိုင်သည်။ Risk-based MFA ဖြင့် user friction ကို minimize ပြုပြီး security level ကို maintain ပြုနိုင်ရသည်မှာ ဤ system ၏ elegant solution ဖြစ်သည်။
