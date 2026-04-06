# အခန်း ၂၃: Microservices တွင် Security (လုံခြုံရေး)

## နိဒါန်း

Security သည် microservices architecture ၌ multi-layered challenge ဖြစ်ပါသည်။ Monolith application တွင် single perimeter (castle-and-moat model) ကိုသာ ကာကွယ်ရသော်လည်း microservices တွင် services ပေါင်းများစွာ၊ APIs ပေါင်းများစွာ၊ network communication channels ပေါင်းများစွာကို ကာကွယ်ရပါသည်။ Service တစ်ခု compromise ဖြစ်လျှင် lateral movement ဖြင့် system တစ်ခုလုံးကို ထိခိုက်နိုင်ပါသည်။ ဤအခန်းတွင် authentication, authorization, service-to-service security, secrets management နှင့် Zero Trust Architecture တို့ကို systematically လေ့လာပါမည်။

---

## ၂၃.၁ Authentication vs Authorization

ဤ concepts နှစ်ခုကို ရှင်းလင်းစွာ ခွဲခြားနားလည်ရန် အရေးကြီးပါသည်။

```
+------------------------------------------------------------------+
|  Authentication (AuthN)        |  Authorization (AuthZ)           |
|  "ဘယ်သူလဲ?"                    |  "ဘာလုပ်ခွင့်ရှိလဲ?"              |
|--------------------------------|----------------------------------|
|  Identity verification         |  Permission check                |
|  "Who are you?"                |  "What can you do?"              |
|  Login, JWT, OAuth2            |  RBAC, ABAC, Policies            |
+------------------------------------------------------------------+

Request Flow:
+--------+     +--------+     +---------+     +----------+
| Client | --> | AuthN  | --> | AuthZ   | --> | Resource |
|        |     | (JWT?) |     | (RBAC?) |     | (API)    |
+--------+     +--------+     +---------+     +----------+
                  |                |
              "ဘယ်သူလဲ?"     "ခွင့်ပြုမလား?"
```

```python
# Authentication - JWT token verify လုပ်ခြင်း
import jwt
from functools import wraps
from flask import request, jsonify

SECRET_KEY = "your-secret-key"  # Production တွင် Vault မှ ယူပါ

def authenticate(func):
    """Authentication middleware - JWT token စစ်ဆေးခြင်း"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        token = request.headers.get('Authorization', '').replace('Bearer ', '')
        if not token:
            return jsonify({"error": "Token required"}), 401  # Unauthorized
        
        try:
            # JWT decode လုပ်ခြင်း - signature verify ပါဝင်ခြင်း
            payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
            request.user = payload  # User info ကို request ထဲ ထည့်ခြင်း
        except jwt.ExpiredSignatureError:
            return jsonify({"error": "Token expired"}), 401
        except jwt.InvalidTokenError:
            return jsonify({"error": "Invalid token"}), 401
        
        return func(*args, **kwargs)
    return wrapper

# Authorization - Role-Based Access Control (RBAC)
def authorize(*required_roles):
    """Authorization decorator - role စစ်ဆေးခြင်း"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            user_roles = request.user.get('roles', [])
            # User ၏ role များထဲတွင် required role ပါဝင်ခြင်း ရှိ/မရှိ စစ်ဆေးခြင်း
            if not any(role in user_roles for role in required_roles):
                return jsonify({"error": "Forbidden"}), 403  # Forbidden
            return func(*args, **kwargs)
        return wrapper
    return decorator

# API endpoint - authentication + authorization
@app.route("/api/admin/users", methods=["GET"])
@authenticate                          # ပထမ - ဘယ်သူလဲ စစ်ခြင်း
@authorize("admin", "super_admin")     # ဒုတိယ - admin role ရှိမှ ခွင့်ပြုခြင်း
def get_all_users():
    return jsonify(users)
```

---

## ၂၃.၂ OAuth၂ & OpenID Connect (JWT, Opaque Tokens)

### OAuth2 Authorization Code Flow

```
+--------+                               +---------------+
| User   |                               | Authorization |
| Browser|                               | Server        |
+---+----+                               | (Keycloak/    |
    |                                     |  Auth0)       |
    | 1. Login request                    +-------+-------+
    |-------------------------------------->      |
    |                                             |
    | 2. Redirect to auth server                  |
    |<----------------------------------------------
    |                                             |
    | 3. User login + consent                     |
    |---------------------------------------------->
    |                                             |
    | 4. Authorization code                       |
    |<----------------------------------------------
    |                                             |
    | 5. Exchange code for tokens                 |
    +--------+                                    |
    | Client |------------------------------------>
    | App    | 6. Access Token + Refresh Token     |
    |        |<------------------------------------+
    +--------+
```

### JWT vs Opaque Tokens

```python
# JWT Token ဖွဲ့စည်းပုံ
# Header.Payload.Signature

import jwt
import datetime

# JWT Token ဖန်တီးခြင်း
def create_jwt_token(user_id, roles, expires_hours=1):
    """JWT access token ဖန်တီးခြင်း"""
    payload = {
        "sub": user_id,                    # Subject (user ID)
        "roles": roles,                     # User roles
        "iss": "auth.myapp.com",           # Issuer
        "aud": "api.myapp.com",            # Audience
        "iat": datetime.datetime.utcnow(), # Issued at
        "exp": datetime.datetime.utcnow()  # Expiration
              + datetime.timedelta(hours=expires_hours),
        "jti": str(uuid.uuid4())           # Unique token ID (revocation အတွက်)
    }
    return jwt.encode(payload, PRIVATE_KEY, algorithm='RS256')

# Opaque Token - token ထဲတွင် data မပါ၊ server side lookup လိုအပ်ခြင်း
# JWT: Self-contained - service ကိုယ်တိုင် verify နိုင်ခြင်း (no DB lookup)
# Opaque: Reference token - auth server ကို introspect call လုပ်ရခြင်း

# OpenID Connect - OAuth2 + Identity Layer
# ID Token (JWT) ပါဝင်ပြီး user info (name, email) ကို claims အဖြစ် ယူဆောင်ခြင်း
```

| Feature | JWT | Opaque Token |
|---------|-----|-------------|
| Self-contained | ဟုတ်သည် | မဟုတ် |
| Server lookup | မလို | လိုသည် |
| Revocation | ခက်ခဲသည် | လွယ်ကူသည် |
| Size | ကြီးသည် (~1KB) | သေးသည် (~32 bytes) |
| Offline verify | ရနိုင်သည် | မရနိုင် |

---

## ၂၃.၃ mTLS for Service-to-Service Communication

**Mutual TLS (mTLS)** တွင် client နှင့် server နှစ်ဖက်စလုံး certificate ဖြင့် identity ကို verify လုပ်ပါသည်။ Service mesh (Istio, Linkerd) ဖြင့် automatic mTLS setup ရနိုင်သည်။

```
Regular TLS:                        Mutual TLS (mTLS):
                                    
Client ------> Server               Client <-----> Server
        verify server cert           verify each other's cert
        (one-way)                    (two-way)

mTLS Handshake:
+----------+                        +----------+
| Service A|                        | Service B|
+----+-----+                        +-----+----+
     |  1. ClientHello                     |
     |------------------------------------>|
     |  2. ServerHello + Server Cert       |
     |<------------------------------------|
     |  3. Verify Server Cert              |
     |  4. Client Cert                     |
     |------------------------------------>|
     |  5. Verify Client Cert              |
     |<------------------------------------|
     |  6. Encrypted Communication         |
     |<----------------------------------->|
     +-------------------------------------+
```

```yaml
# Istio Service Mesh ဖြင့် automatic mTLS
# PeerAuthentication - mTLS mode သတ်မှတ်ခြင်း
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT     # mTLS မဖြစ်မနေ အသုံးပြုရမည် (PERMISSIVE = optional)

---
# AuthorizationPolicy - service-to-service access control
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  rules:
    - from:
        - source:
            principals:
              # Order service မှသာ payment service ကို ခေါ်ခွင့်ရှိခြင်း
              - "cluster.local/ns/production/sa/order-service"
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/payments/*"]
```

---

## ၂၃.၄ Secrets Management (Vault, AWS Secrets Manager)

Database passwords, API keys, certificates စသည့် secrets များကို source code, environment variables, config files တွင် hardcode မလုပ်ဘဲ dedicated secrets management tool ဖြင့် manage လုပ်ရပါသည်။

```
+-----------------------------------------------------------+
|                  Secrets Management                        |
+-----------------------------------------------------------+
|                                                           |
|  BAD:                          GOOD:                      |
|  DB_PASS="p@ssw0rd"           +--------+                  |
|  (in .env file)               | Vault  | <-- Encrypted   |
|                                +---+----+     Storage     |
|  AWS_KEY="AKIA..."                |                       |
|  (in docker-compose.yml)          v                       |
|                                Service                    |
|                                (runtime secret fetch)     |
+-----------------------------------------------------------+
```

```python
# HashiCorp Vault ဖြင့် secrets ရယူခြင်း
import hvac

class SecretsManager:
    def __init__(self, vault_url, vault_token):
        """Vault client initialize လုပ်ခြင်း"""
        self.client = hvac.Client(url=vault_url, token=vault_token)

    def get_database_credentials(self, db_role):
        """Dynamic database credentials ရယူခြင်း - Vault မှ ယာယီ credentials ထုတ်ပေးခြင်း"""
        # Vault ၏ database secrets engine မှ dynamic credentials ရယူခြင်း
        response = self.client.secrets.database.generate_credentials(
            name=db_role  # "order-service-readonly" ကဲ့သို့ role
        )
        return {
            "username": response['data']['username'],  # ယာယီ username
            "password": response['data']['password'],  # ယာယီ password
            "lease_duration": response['lease_duration'],  # Credential ၏ သက်တမ်း
            "lease_id": response['lease_id']
        }

    def get_secret(self, path):
        """KV secrets engine မှ secret ရယူခြင်း"""
        response = self.client.secrets.kv.v2.read_secret_version(path=path)
        return response['data']['data']

# အသုံးပြုပုံ
vault = SecretsManager(
    vault_url="https://vault.internal:8200",
    vault_token="s.xxxxx"  # Production တွင် Kubernetes auth method သုံးပါ
)

# Database credentials ရယူခြင်း (auto-rotate ဖြစ်သည်)
db_creds = vault.get_database_credentials("order-service-readwrite")
print(f"DB User: {db_creds['username']}")  # Temporary username

# API key ရယူခြင်း
api_secrets = vault.get_secret("services/payment-gateway")
stripe_key = api_secrets['stripe_api_key']
```

```yaml
# Kubernetes Vault integration (Vault Agent Injector)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        # Vault agent injector annotations
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "order-service"
        # Secret ကို file အဖြစ် inject လုပ်ခြင်း
        vault.hashicorp.com/agent-inject-secret-db: "database/creds/order-service"
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "database/creds/order-service" -}}
          DB_USERNAME={{ .Data.username }}
          DB_PASSWORD={{ .Data.password }}
          {{- end -}}
```

---

## ၂၃.၅ Zero Trust Architecture

**Zero Trust** ၏ အခြေခံ မူဝါဒမှာ "Never trust, always verify" ဖြစ်ပါသည်။ Network perimeter ကို trust boundary အဖြစ် မယူဆဘဲ request တိုင်းကို verify လုပ်ပါသည်။

```
Traditional (Castle-and-Moat):       Zero Trust:

+---------------------------+        Every request is verified:
|  Trusted Internal Network |
|                           |        Service A --[mTLS + JWT]--> Service B
|  Service A --> Service B  |                    |
|  (no auth needed)         |                    v
|                           |        [Identity] [Device] [Context]
+------ Firewall -----------+        [Least Privilege] [Encrypt]
   ^                                 [Continuous Verification]
   |
 Untrusted
 Internet
```

### Zero Trust ၏ အဓိက မူဝါဒများ

1. **Verify Explicitly** - Request တိုင်းကို authenticate + authorize လုပ်ခြင်း
2. **Least Privilege Access** - လိုအပ်သော အနည်းဆုံး permission ကိုသာ ပေးခြင်း
3. **Assume Breach** - Breach ဖြစ်ပြီးဟု ယူဆ၍ blast radius ကို minimize လုပ်ခြင်း

```python
# Zero Trust middleware - request တိုင်းကို verify လုပ်ခြင်း
class ZeroTrustMiddleware:
    def __init__(self, app):
        self.app = app

    def __call__(self, request):
        # 1. Identity verification (mTLS certificate check)
        if not self._verify_mtls_cert(request):
            return Response("Invalid client certificate", status=403)

        # 2. JWT token validation
        if not self._verify_jwt_token(request):
            return Response("Invalid or expired token", status=401)

        # 3. Authorization policy check (OPA - Open Policy Agent)
        if not self._check_policy(request):
            return Response("Access denied by policy", status=403)

        # 4. Rate limiting per service identity
        if not self._check_rate_limit(request):
            return Response("Rate limit exceeded", status=429)

        # 5. Audit log - request တိုင်းကို log ရေးခြင်း
        self._audit_log(request)

        return self.app(request)

    def _check_policy(self, request):
        """OPA (Open Policy Agent) ဖြင့် fine-grained authorization"""
        policy_input = {
            "source_service": request.mtls_identity,
            "target_service": "order-service",
            "method": request.method,
            "path": request.path,
            "time": datetime.utcnow().isoformat()
        }
        # OPA server ကို query လုပ်ခြင်း
        result = requests.post(
            "http://opa:8181/v1/data/authz/allow",
            json={"input": policy_input}
        )
        return result.json().get("result", False)
```

---

## ၂၃.၆ OWASP Top ၁၀ for APIs

OWASP API Security Top 10 (2023) သည် API security threats များ၏ စာရင်းဖြစ်ပြီး microservices APIs များအတွက် အထူး သက်ဆိုင်ပါသည်။

```
+----+----------------------------------+-----------------------------------+
| #  | Vulnerability                    | ကာကွယ်နည်း                       |
+----+----------------------------------+-----------------------------------+
| 1  | Broken Object Level AuthZ (BOLA) | Object-level permission check     |
| 2  | Broken Authentication            | Strong auth, MFA, rate limit      |
| 3  | Broken Object Property Level AuthZ| Field-level access control       |
| 4  | Unrestricted Resource Consumption| Rate limiting, pagination         |
| 5  | Broken Function Level AuthZ      | RBAC, least privilege             |
| 6  | Unrestricted Access to Sensitive | Whitelist allowed endpoints       |
|    | Business Flows                   |                                   |
| 7  | Server Side Request Forgery      | Input validation, allowlists      |
| 8  | Security Misconfiguration        | Hardened defaults, security scan  |
| 9  | Improper Inventory Management    | API catalog, deprecation policy   |
| 10 | Unsafe Consumption of APIs       | Validate 3rd party responses      |
+----+----------------------------------+-----------------------------------+
```

```python
# BOLA (Broken Object Level Authorization) ကာကွယ်ခြင်း ဥပမာ
@app.route("/api/orders/<order_id>", methods=["GET"])
@authenticate
def get_order(order_id):
    order = db.get_order(order_id)
    
    # BOLA check - user သည် ဤ order ကို ကြည့်ခွင့်ရှိမရှိ စစ်ဆေးခြင်း
    if order.customer_id != request.user['sub'] and 'admin' not in request.user['roles']:
        return jsonify({"error": "Access denied"}), 403  # Forbidden
    
    return jsonify(order.to_dict())

# Input validation - injection attacks ကာကွယ်ခြင်း
from pydantic import BaseModel, validator, constr

class CreateOrderRequest(BaseModel):
    product_id: constr(max_length=50, regex=r'^[a-zA-Z0-9-]+$')  # Alphanumeric only
    quantity: int
    
    @validator('quantity')
    def validate_quantity(cls, v):
        if v < 1 or v > 1000:
            raise ValueError('Quantity must be between 1 and 1000')
        return v
```

---

## အဓိက အချက်များ (Key Takeaways)

- **Authentication** (ဘယ်သူလဲ) နှင့် **Authorization** (ဘာလုပ်ခွင့်ရှိလဲ) ကို ရှင်းလင်းစွာ ခွဲခြားပြီး implement လုပ်ပါ
- **JWT** သည် self-contained ဖြစ်ပြီး service ကိုယ်တိုင် verify နိုင်သည်။ Revocation အတွက် short expiry + refresh token pattern ကို အသုံးပြုပါ
- **mTLS** ဖြင့် service-to-service communication ကို encrypt + authenticate လုပ်ပါ။ Service mesh (Istio) ဖြင့် auto-mTLS ရနိုင်သည်
- **Vault** ကဲ့သို့ secrets manager ဖြင့် credentials များကို centrally manage လုပ်ပြီး auto-rotate လုပ်ပါ
- **Zero Trust** - "Never trust, always verify" မူဝါဒဖြင့် request တိုင်းကို verify + authorize + audit log လုပ်ပါ
- **OWASP API Top 10** ကို reference အဖြစ် အသုံးပြုပြီး API security ကို systematically ကာကွယ်ပါ
