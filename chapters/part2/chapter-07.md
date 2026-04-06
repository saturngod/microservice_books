# အခန်း ၇ — API Gateway နှင့် Service Mesh

## နိဒါန်း

Microservices architecture တွင် services ဆယ်ကျော်ရာနှင့် ရာကျော်ထောင် ဖြစ်လာသောအခါ cross-cutting concerns (authentication, rate limiting, logging, load balancing) ကို service တစ်ခုချင်းတွင် ထပ်ထပ်ရေးနေမည်ဆိုပါက maintenance nightmare ဖြစ်မည်ဖြစ်သည်။ **API Gateway** နှင့် **Service Mesh** တို့သည် ဤ cross-cutting concerns ကို centralized (API Gateway) သို့မဟုတ် distributed (Service Mesh) ဖြင့် ဖြေရှင်းသော infrastructure patterns နှစ်ခုဖြစ်သည်။

ဤအခန်းတွင် API Gateway ၏ responsibilities, gateway patterns, popular tools (Kong, AWS API Gateway, Nginx, Envoy), Service Mesh (Istio, Linkerd), Sidecar Proxy Pattern, နှင့် East-West vs North-South traffic ကွဲပြားချက်တို့ကို အသေးစိတ် ဆွေးနွေးသွားမည်ဖြစ်သည်။

---

## ၇.၁ API Gateway Responsibilities

**API Gateway** ဆိုသည်မှာ external clients နှင့် internal microservices ကြားတွင် single entry point တစ်ခုဖြင့် ဆောင်ရွက်ပေးသော infrastructure component ဖြစ်သည်။ Reverse proxy ကဲ့သို့ အလုပ်လုပ်ပြီး client requests များကို appropriate backend service သို့ route လုပ်ပေးသည်။

```
API Gateway Responsibilities:
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY                            │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │   Auth    │  │   Rate    │  │   Load    │               │
│  │  (JWT/    │  │  Limiting │  │ Balancing │               │
│  │  OAuth)   │  │           │  │           │               │
│  └───────────┘  └───────────┘  └───────────┘               │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │  Request  │  │  Response │  │   SSL     │               │
│  │  Routing  │  │ Transform │  │Termination│               │
│  └───────────┘  └───────────┘  └───────────┘               │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │   Logging │  │  Caching  │  │  Circuit  │               │
│  │  Tracing  │  │           │  │  Breaker  │               │
│  └───────────┘  └───────────┘  └───────────┘               │
└─────────────────────────────────────────────────────────────┘
```

**Core Responsibilities တစ်ခုချင်း ရှင်းလင်းချက်**

| Responsibility | ရှင်းလင်းချက် |
|---|---|
| **Authentication** | JWT/OAuth2 token validation — service တစ်ခုချင်း မလုပ်ရ |
| **Authorization** | Scope-based access control — role/permission check |
| **Rate Limiting** | Client IP/API key ဖြင့် request throttle ပြုလုပ်သည် |
| **Request Routing** | Path/method/header ဖြင့် upstream service ကို route ပြုလုပ်သည် |
| **Load Balancing** | Upstream service instances ကြား traffic ဖြန့်ဝေသည် |
| **SSL Termination** | TLS terminate ပြီး internal HTTP ဖြင့် forward ပြုလုပ်သည် |
| **Response Transform** | Request/Response headers, body ပြင်ဆင်သည် |
| **Caching** | Frequently-requested responses ကို cache ပြုလုပ်သည် |
| **Circuit Breaking** | Upstream failure တွင် requests ဖြတ်တောက်သည် |
| **Observability** | Distributed tracing, access logs စုဆောင်းသည် |

### Authentication & Authorization

Gateway level တွင် authentication ကို first line of defense အဖြစ် centralized ပြုလုပ်နိုင်သော်လည်း backend services များသည် request ကို "already authenticated" ဟု blind trust မလုပ်သင့်ပါ။ Downstream service တစ်ခုချင်းသည် JWT claims, signed headers, mTLS identity, နှင့် authorization rules များကို context အလိုက် verify ပြန်လုပ်ရသည်။

```python
# API Gateway authentication middleware (Python/Flask ဥပမာ)
from functools import wraps
import jwt

# JWT secret key — production တွင် environment variable မှ ယူသင့်သည်
JWT_SECRET = "your-secret-key"

def gateway_auth_middleware(f):
    """Gateway level JWT authentication decorator
    Backend services သို့ authenticated request များသာ ရောက်ရှိစေသည်"""
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get("Authorization", "").replace("Bearer ", "")
        if not token:
            # Token မရှိပါက 401 Unauthorized ပြန်ပို့သည်
            return jsonify({"error": "Token required"}), 401
        try:
            # Token ကို decode ပြုလုပ်ပြီး user info ထုတ်ယူသည်
            payload = jwt.decode(token, JWT_SECRET, algorithms=["HS256"])
            request.user_id = payload["user_id"]
            request.roles = payload.get("roles", [])
        except jwt.ExpiredSignatureError:
            return jsonify({"error": "Token expired"}), 401
        except jwt.InvalidTokenError:
            return jsonify({"error": "Invalid token"}), 401
        return f(*args, **kwargs)
    return decorated
```

### Rate Limiting

Rate limiting သည် DDoS attack ကာကွယ်ခြင်း၊ fair usage enforce ခြင်းတို့အတွက် အရေးကြီးသည်။ Token Bucket algorithm ကို အသုံးများသည်။

```python
import time
from collections import defaultdict

class TokenBucketRateLimiter:
    """Token Bucket algorithm ဖြင့် rate limiting ပြုလုပ်သည်
    bucket တစ်ခုတွင် tokens အရေအတွက် ကန့်သတ်ပြီး
    request တစ်ခုလျှင် token တစ်ခု သုံးသည်"""

    def __init__(self, max_tokens=100, refill_rate=10):
        # bucket တစ်ခုတွင် ထည့်နိုင်သော အများဆုံး token အရေအတွက်
        self.max_tokens = max_tokens
        # တစ်စက္ကန့်လျှင် ပြန်ဖြည့်သော token အရေအတွက်
        self.refill_rate = refill_rate
        self.buckets = defaultdict(lambda: {
            "tokens": max_tokens,
            "last_refill": time.time()
        })

    def allow_request(self, client_id):
        """Client request ကို ခွင့်ပြုမည်/ငြင်းပယ်မည် ဆုံးဖြတ်သည်"""
        bucket = self.buckets[client_id]
        now = time.time()
        # elapsed time အလိုက် tokens ပြန်ဖြည့်သည်
        elapsed = now - bucket["last_refill"]
        bucket["tokens"] = min(
            self.max_tokens,
            bucket["tokens"] + elapsed * self.refill_rate
        )
        bucket["last_refill"] = now

        if bucket["tokens"] >= 1:
            bucket["tokens"] -= 1
            return True  # Request ခွင့်ပြုသည်
        return False  # Rate limit ကျော်လွန်ပြီ — 429 ပြန်ပို့မည်

# အသုံးပြုပုံ
limiter = TokenBucketRateLimiter(max_tokens=100, refill_rate=10)
# Client "user_123" ၏ request ကို စစ်ဆေးသည်
if not limiter.allow_request("user_123"):
    # HTTP 429 Too Many Requests ပြန်ပို့သည်
    print("Rate limit exceeded")
```

### SSL Termination

```
SSL Termination Flow:
┌──────────┐  HTTPS   ┌──────────────┐ HTTPS/mTLS ┌──────────────┐
│  Client  │ ───────> │  API Gateway │ ─────────> │  Service A   │
│ (Browser)│  :443    │  (TLS Term)  │   :8443    │  (Internal)  │
└──────────┘          └──────────────┘         └──────────────┘

Client နှင့် Gateway ကြားတွင် HTTPS (encrypted)
Gateway နှင့် Internal Service ကြားတွင် TLS re-encryption သို့မဟုတ် mTLS ဖြင့် encrypt ပြုလုပ်သည်
Zero-trust production environment တွင် internal network ကို trusted zone ဟု မယူဆသင့်
(Service Mesh သုံးပါက internal traffic ကို auto-mTLS ဖြင့် enforce ပြုလုပ်နိုင်သည်)
```

### Request Routing

```yaml
# Kong declarative config ဥပမာ — route configuration
# Path prefix ဖြင့် appropriate upstream service သို့ route ပြုလုပ်သည်
_format_version: "3.0"

services:
  - name: user-service
    url: http://user-svc:8080
    routes:
      - name: user-routes
        paths:
          - /api/v1/users    # /api/v1/users/* requests အားလုံး user-service သို့
        strip_path: false

  - name: order-service
    url: http://order-svc:8080
    routes:
      - name: order-routes
        paths:
          - /api/v1/orders   # /api/v1/orders/* requests order-service သို့
        strip_path: false

  - name: product-service
    url: http://product-svc:8080
    routes:
      - name: product-routes
        paths:
          - /api/v1/products
        methods:             # GET method သာ ခွင့်ပြုသည် (read-only endpoint)
          - GET
```

---

## ၇.၂ Gateway Patterns

Gateway pattern သုံးမျိုးကို လက်တွေ့တွင် အသုံးများသည် — Edge Gateway, Backend for Frontend (BFF), Micro-Gateway တို့ဖြစ်သည်။

### Edge Gateway Pattern

```
Edge Gateway Pattern:
                    ┌─────────────────┐
  Mobile App ──────>│                 │──────> User Service
  Web App ─────────>│   Edge Gateway  │──────> Order Service
  Third Party ─────>│   (Single)      │──────> Product Service
  IoT Device ──────>│                 │──────> Payment Service
                    └─────────────────┘

Client အားလုံးအတွက် Gateway တစ်ခုတည်း
ရိုးရှင်းသော architecture — small-medium scale အတွက် သင့်တော်သည်
```

Edge Gateway သည် အရိုးအရှင်းဆုံး pattern ဖြစ်ပြီး single gateway တစ်ခုမှတဆင့် traffic အားလုံး ဝင်ရောက်သည်။ Small-to-medium scale applications အတွက် အသင့်တော်ဆုံးဖြစ်သည်။

**အားသာချက်များ:**
- Single point of configuration — ပြင်ဆင်ရလွယ်သည်
- Centralized logging/monitoring
- Simple deployment model

**အားနည်းချက်များ:**
- Single point of failure ဖြစ်နိုင်သည် (HA configuration လိုသည်)
- Client အမျိုးမျိုးအတွက် response format customize ပြုလုပ်ရခက်သည်
- Gateway logic ကြီးထွားလာသည်နှင့်အမျှ complexity တိုးလာသည်

### Backend for Frontend (BFF) Pattern

```
BFF Pattern:
                    ┌───────────────┐
  Mobile App ──────>│  Mobile BFF   │──┐
                    └───────────────┘  │
                    ┌───────────────┐  │     ┌──────────────┐
  Web App ─────────>│   Web BFF     │──┼────>│  Services    │
                    └───────────────┘  │     │  (Backend)   │
                    ┌───────────────┐  │     └──────────────┘
  IoT Device ──────>│   IoT BFF     │──┘
                    └───────────────┘

Client type တစ်ခုချင်းအတွက် dedicated gateway
Mobile BFF: data ကို compress ပြုလုပ်၊ minimal payload ပြန်ပို့
Web BFF: rich data format, aggregated responses
IoT BFF: lightweight protocol (MQTT/CoAP) support
```

BFF pattern သည် client type တစ်ခုစီအတွက် dedicated gateway ထားရှိခြင်းဖြစ်သည်။ Netflix, SoundCloud တို့က ဤ pattern ကို ကျယ်ကျယ်ပြန့်ပြန့် အသုံးပြုသည်။

```javascript
// Mobile BFF — data ကို compress ပြုလုပ်ပြီး minimal payload ပြန်ပို့သည်
// Mobile device တွင် bandwidth limited ဖြစ်သောကြောင့် payload သေးသင့်သည်
app.get("/api/mobile/products/:id", async (req, res) => {
  const product = await productService.getById(req.params.id);
  const reviews = await reviewService.getSummary(req.params.id);

  // Mobile အတွက် essential fields သာ ပြန်ပို့သည်
  res.json({
    id: product.id,
    name: product.name,
    price: product.price,
    rating: reviews.averageRating,
    // Mobile တွင် thumbnail သာ လိုသည်
    image: product.thumbnailUrl,
    inStock: product.inventory > 0,
  });
});

// Web BFF — rich data ပါသော full response ပြန်ပို့သည်
// Desktop browser တွင် bandwidth များသောကြောင့် detailed data ပေးနိုင်သည်
app.get("/api/web/products/:id", async (req, res) => {
  const [product, reviews, related, seller] = await Promise.all([
    productService.getById(req.params.id),
    reviewService.getAll(req.params.id),
    recommendationService.getRelated(req.params.id),
    sellerService.getInfo(product.sellerId),
  ]);

  // Web အတွက် full detail ပြန်ပို့သည်
  res.json({
    ...product,
    reviews: reviews,
    relatedProducts: related,
    seller: seller,
    images: product.allImages,  // Full resolution images အားလုံး
    specifications: product.specs,
  });
});
```

### Micro-Gateway Pattern

```
Micro-Gateway Pattern:
  ┌─────────────────────────────────────────────────────┐
  │                   Edge Gateway                      │
  │              (Auth, Rate Limiting)                   │
  └───────┬──────────────┬──────────────┬───────────────┘
          │              │              │
  ┌───────▼──────┐ ┌────▼────────┐ ┌──▼────────────┐
  │ User Domain  │ │Order Domain │ │Product Domain │
  │  Gateway     │ │  Gateway    │ │   Gateway     │
  │  ┌────────┐  │ │  ┌────────┐ │ │  ┌────────┐  │
  │  │User Svc│  │ │  │Ord Svc │ │ │  │Prod Svc│  │
  │  │Auth Svc│  │ │  │Pay Svc │ │ │  │Inv Svc │  │
  │  └────────┘  │ │  └────────┘ │ │  └────────┘  │
  └──────────────┘ └─────────────┘ └──────────────┘

Domain/team တစ်ခုချင်းအတွက် own gateway ထားရှိသည်
Edge Gateway သည် global concerns (auth, rate limit) ကို handle ပြုလုပ်သည်
Domain Gateway သည် domain-specific routing/transformation ပြုလုပ်သည်
```

Micro-Gateway pattern သည် large organization များအတွက် သင့်တော်ပြီး team autonomy ကို ပေးစွမ်းသည်။ Domain team တစ်ခုချင်းသည် own gateway configuration ကို manage ပြုလုပ်နိုင်သည်။

**Gateway Pattern ရွေးချယ်မှု:**

| Pattern | သင့်တော်သော အခြေအနေ | ဥပမာ |
|---|---|---|
| **Edge Gateway** | Small/medium apps, single client type | Internal tool, single SPA |
| **BFF** | Client types ကွဲပြားသည်, UI team ခွဲထားသည် | Netflix (TV, Mobile, Web) |
| **Micro-Gateway** | Large org, domain teams autonomous | Amazon, Uber |

---

## ၇.၃ Kong vs AWS API Gateway vs Nginx vs Envoy

### Kong

Kong သည် Nginx/OpenResty အပေါ်တွင် built ပြုလုပ်ထားသော open-source API Gateway ဖြစ်သည်။ Plugin architecture ကြောင့် extensible ဖြစ်ပြီး Lua ဖြင့် custom plugins ရေးနိုင်သည်။

```yaml
# Kong plugin configuration ဥပမာ
# Rate limiting plugin enable ပြုလုပ်ခြင်း
plugins:
  - name: rate-limiting
    config:
      minute: 100           # တစ်မိနစ်လျှင် requests 100 ခွင့်ပြုသည်
      policy: redis          # Redis backend ဖြင့် distributed rate limiting
      redis_host: redis-host
      redis_port: 6379

  - name: jwt                # JWT authentication plugin
    config:
      claims_to_verify:
        - exp                # Token expiration စစ်ဆေးသည်

  - name: cors               # Cross-Origin Resource Sharing
    config:
      origins:
        - "https://myapp.com"
      methods:
        - GET
        - POST
      headers:
        - Authorization
        - Content-Type
```

### AWS API Gateway

AWS API Gateway သည် fully managed service ဖြစ်ပြီး AWS ecosystem နှင့် deeply integrated ဖြစ်သည်။ Lambda integration ဖြင့် serverless architecture ကို support ပြုလုပ်သည်။

```yaml
# AWS SAM template — API Gateway + Lambda integration
# Serverless architecture အတွက် AWS API Gateway configuration
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  UserApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      # Cognito authorizer ဖြင့် authentication ပြုလုပ်သည်
      Auth:
        DefaultAuthorizer: CognitoAuth
        Authorizers:
          CognitoAuth:
            UserPoolArn: !GetAtt UserPool.Arn

  # Lambda function — API Gateway မှ trigger ပြုလုပ်သည်
  GetUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: users.get_handler
      Runtime: python3.11
      Events:
        GetUser:
          Type: Api
          Properties:
            RestApiId: !Ref UserApi
            Path: /users/{id}
            Method: GET
      # Usage plan ဖြင့် rate limiting ပြုလုပ်သည်
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref UsersTable
```

### Nginx

Nginx သည် high-performance reverse proxy/load balancer ဖြစ်ပြီး API Gateway အဖြစ်လည်း အသုံးပြုနိုင်သည်။ Configuration-based approach ဖြစ်သည်။

```nginx
# Nginx API Gateway configuration
# Upstream services group ခွဲထားသည်
upstream user_service {
    # Round-robin load balancing (default)
    server user-svc-1:8080 weight=3;   # Weight 3 — traffic 3 ပုံ ပိုရသည်
    server user-svc-2:8080 weight=1;
    keepalive 32;                        # Connection pooling
}

upstream order_service {
    # Least connections algorithm — load အနည်းဆုံး server သို့ route
    least_conn;
    server order-svc-1:8080;
    server order-svc-2:8080;
}

server {
    listen 443 ssl;
    server_name api.myapp.com;

    # SSL termination configuration
    ssl_certificate     /etc/ssl/certs/api.crt;
    ssl_certificate_key /etc/ssl/private/api.key;

    # Rate limiting zone — client IP ဖြင့် 10 req/sec ကန့်သတ်သည်
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

    # User service routing
    location /api/v1/users {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://user_service;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;
    }

    # Order service routing
    location /api/v1/orders {
        limit_req zone=api_limit burst=10;
        proxy_pass http://order_service;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Envoy Proxy

Envoy သည် CNCF project ဖြစ်ပြီး modern cloud-native applications အတွက် ဒီဇိုင်းထုတ်ထားသည်။ Istio service mesh ၏ data plane component အဖြစ် အသုံးပြုသည်။

```yaml
# Envoy proxy configuration ဥပမာ
# Envoy သည် xDS API ဖြင့် dynamic configuration support ပြုလုပ်သည်
static_resources:
  listeners:
    - name: main_listener
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        # /api/users path ကို user_cluster သို့ route
                        - match:
                            prefix: "/api/users"
                          route:
                            cluster: user_cluster
                            # Retry policy — failure တွင် 3 ကြိမ် retry
                            retry_policy:
                              retry_on: "5xx,reset,connect-failure"
                              num_retries: 3
                        - match:
                            prefix: "/api/orders"
                          route:
                            cluster: order_cluster

  clusters:
    - name: user_cluster
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: user_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: user-service
                      port_value: 8080
      # Circuit breaker configuration
      circuit_breakers:
        thresholds:
          - max_connections: 1000
            max_pending_requests: 500
            max_requests: 1000
            max_retries: 3
```

### Comparison Table

| Feature | Kong | AWS API GW | Nginx | Envoy |
|---|---|---|---|---|
| **Type** | Open Source / Enterprise | Managed | Open Source | Open Source (CNCF) |
| **Protocol** | HTTP, gRPC, WebSocket | HTTP, WebSocket | HTTP, TCP, UDP | HTTP/1.1, HTTP/2, gRPC |
| **Plugin System** | Lua plugins (rich) | Lambda + Extensions | Lua/njs modules | C++ filters, WASM |
| **Service Discovery** | DNS, Consul | AWS internal | DNS | xDS API (dynamic) |
| **Rate Limiting** | Built-in plugin | Usage Plans | ngx_http_limit_req | Built-in filter |
| **mTLS** | Plugin | ACM integration | Config | Native support |
| **Observability** | Prometheus, Datadog | CloudWatch | Basic access logs | Prometheus, tracing |
| **Cost** | Free OSS / $$ Enterprise | Pay-per-request | Free | Free |
| **Best For** | Multi-cloud API GW | AWS-native apps | High-perf reverse proxy | Service mesh data plane |
| **Complexity** | Medium | Low (managed) | Low-Medium | High |

---

## ၇.၄ Service Mesh — Istio, Linkerd

### Service Mesh ဆိုသည်မှာ

Service Mesh သည် microservices ကြားရှိ network communication ကို manage ပြုလုပ်သော dedicated infrastructure layer ဖြစ်သည်။ Application code ကို မပြင်ဘဲ service-to-service communication တွင် security (mTLS), observability (tracing, metrics), reliability (retries, circuit breaking) တို့ကို ထည့်သွင်းပေးနိုင်သည်။

```
Service Mesh Architecture:
┌────────────────────────────────────────────────────────────┐
│                     CONTROL PLANE                          │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────┐    │
│  │  Pilot   │  │   Citadel    │  │     Galley        │    │
│  │ (Config) │  │  (Cert/mTLS) │  │  (Config Mgmt)    │    │
│  └──────────┘  └──────────────┘  └───────────────────┘    │
└────────────────────────────────────────────────────────────┘
         │              │                    │
         ▼              ▼                    ▼
┌────────────────────────────────────────────────────────────┐
│                      DATA PLANE                            │
│  ┌─────────────────┐      ┌─────────────────┐             │
│  │  Pod A          │      │  Pod B          │             │
│  │  ┌───────────┐  │      │  ┌───────────┐  │             │
│  │  │  Service  │  │ mTLS │  │  Service  │  │             │
│  │  │     A     │  │◄────►│  │     B     │  │             │
│  │  └─────┬─────┘  │      │  └─────┬─────┘  │             │
│  │  ┌─────▼─────┐  │      │  ┌─────▼─────┐  │             │
│  │  │  Envoy    │  │      │  │  Envoy    │  │             │
│  │  │  Sidecar  │◄─┼──────┼─►│  Sidecar  │  │             │
│  │  └───────────┘  │      │  └───────────┘  │             │
│  └─────────────────┘      └─────────────────┘             │
└────────────────────────────────────────────────────────────┘

Control Plane: configuration, certificates, policies manage ပြုလုပ်သည်
Data Plane: sidecar proxies ဖြင့် actual traffic handle ပြုလုပ်သည်
```

### Istio

Istio သည် Google, IBM, Lyft တို့ ဖန်တီးထားပြီး feature-rich service mesh ဖြစ်သည်။ Envoy proxy ကို sidecar အဖြစ် အသုံးပြုသည်။

```yaml
# Istio VirtualService — traffic routing rules
# Canary deployment: traffic 90% ကို v1, 10% ကို v2 သို့ route
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
    - order-service
  http:
    - route:
        - destination:
            host: order-service
            subset: v1
          weight: 90      # Traffic ၏ 90% ကို v1 သို့ ပို့သည်
        - destination:
            host: order-service
            subset: v2
          weight: 10      # Traffic ၏ 10% ကို v2 (canary) သို့ ပို့သည်
      timeout: 5s         # Request timeout 5 seconds
      retries:
        attempts: 3       # Failure တွင် 3 ကြိမ် retry
        perTryTimeout: 2s
---
# Istio DestinationRule — subset definitions + circuit breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    # Circuit breaker configuration
    outlierDetection:
      consecutive5xxErrors: 5     # 5xx errors 5 ခု ဆက်တိုက်ဖြစ်ပါက
      interval: 30s               # 30 seconds interval ဖြင့် စစ်ဆေးသည်
      baseEjectionTime: 60s       # 60 seconds circuit open ထားသည်
      maxEjectionPercent: 50      # Upstream hosts ၏ 50% ထက်မပိုဘဲ eject
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
---
# Istio PeerAuthentication — mTLS enforce ပြုလုပ်ခြင်း
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT    # namespace အတွင်း service-to-service traffic အားလုံး mTLS required
```

### Linkerd

Linkerd သည် lightweight, Rust-based service mesh ဖြစ်ပြီး simplicity ကို ဦးစားပေးသည်။ Istio ထက် resource consumption နည်းပြီး setup ပိုလွယ်သည်။

```bash
# Linkerd installation — Kubernetes cluster တွင် install ပြုလုပ်ခြင်း
# Step 1: Linkerd CLI install ပြုလုပ်သည်
curl -sL https://run.linkerd.io/install | sh

# Step 2: Pre-check — cluster compatibility စစ်ဆေးသည်
linkerd check --pre

# Step 3: Control plane install ပြုလုပ်သည်
linkerd install | kubectl apply -f -

# Step 4: Installation verify ပြုလုပ်သည်
linkerd check

# Step 5: Application namespace ကို mesh ထဲသို့ inject ပြုလုပ်သည်
# Pod restart ပြုလုပ်သောအခါ sidecar proxy auto-inject ဖြစ်သည်
kubectl annotate namespace production linkerd.io/inject=enabled

# Step 6: Dashboard ဖွင့်ပြီး traffic observe ပြုလုပ်သည်
linkerd dashboard
```

```yaml
# Linkerd ServiceProfile — retry & timeout policies
# Service level ရှိ routes အတွက် fine-grained policies သတ်မှတ်သည်
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: order-service.production.svc.cluster.local
  namespace: production
spec:
  routes:
    - name: "GET /api/orders/{id}"
      condition:
        method: GET
        pathRegex: "/api/orders/[^/]*"
      # GET request ကို retry ပြုလုပ်နိုင်သည် (idempotent)
      isRetryable: true
      timeout: 3s

    - name: "POST /api/orders"
      condition:
        method: POST
        pathRegex: "/api/orders"
      # POST request ကို retry မပြုလုပ်သင့် (non-idempotent)
      isRetryable: false
      timeout: 10s
```

### Istio vs Linkerd Comparison

| Feature | Istio | Linkerd |
|---|---|---|
| **Sidecar Proxy** | Envoy (C++) | linkerd2-proxy (Rust) |
| **Resource Usage** | Higher (Envoy heavy) | Lower (ultra-light proxy) |
| **Setup Complexity** | Complex | Simple (5 minutes) |
| **mTLS** | Full support | Full support (auto) |
| **Traffic Management** | Advanced (weighted, mirror) | Basic (retry, timeout) |
| **Observability** | Kiali, Jaeger, Grafana | Built-in dashboard |
| **Multi-cluster** | Yes | Yes (newer versions) |
| **WASM Extensions** | Yes | No |
| **Best For** | Large enterprise, complex routing | Small-medium, simplicity |

---

## ၇.၅ Sidecar Proxy Pattern

Sidecar Proxy Pattern သည် service mesh ၏ foundational pattern ဖြစ်သည်။ Application container ဘေးတွင် proxy container ကို "sidecar" အဖြစ် deploy ပြုလုပ်ခြင်းဖြစ်သည်။

```
Sidecar Proxy Pattern — Detailed View:
┌──────────────────────────────────────────────────────────────────┐
│  Kubernetes Pod                                                  │
│                                                                  │
│  ┌─────────────────────┐    ┌───────────────────────────────┐   │
│  │  Application         │    │  Sidecar Proxy (Envoy)        │   │
│  │  Container           │    │                               │   │
│  │                      │    │  ┌─────────┐  ┌──────────┐   │   │
│  │  ┌────────────────┐  │    │  │  mTLS   │  │  Retry   │   │   │
│  │  │  Order Service  │  │    │  │  Mgmt   │  │  Logic   │   │   │
│  │  │  (Go/Java/etc)  │ ─┼───►│  └─────────┘  └──────────┘   │   │
│  │  │                  │  │    │  ┌─────────┐  ┌──────────┐   │   │
│  │  │  localhost:8080  │  │    │  │ Circuit │  │ Load     │   │   │
│  │  └────────────────┘  │    │  │ Breaker │  │ Balance  │   │   │
│  │                      │    │  └─────────┘  └──────────┘   │   │
│  └─────────────────────┘    │  ┌─────────┐  ┌──────────┐   │   │
│                              │  │ Metrics │  │ Access   │   │   │
│                              │  │ Export  │  │ Logging  │   │   │
│                              │  └─────────┘  └──────────┘   │   │
│                              └───────────────────────────────┘   │
│                                                                  │
│  Inbound traffic:  External ──► Sidecar ──► App Container        │
│  Outbound traffic: App Container ──► Sidecar ──► External        │
└──────────────────────────────────────────────────────────────────┘

Application code ကို ပြုပြင်ရန် မလိုဘဲ networking concerns ကို handle ပြုလုပ်သည်
iptables rules ဖြင့် Pod ၏ traffic အားလုံးကို sidecar မှတဆင့် redirect ပြုလုပ်သည်
```

### Sidecar Injection

Kubernetes တွင် sidecar injection ကို automatic (mutating webhook) သို့မဟုတ် manual ဖြင့် ပြုလုပ်နိုင်သည်။

```yaml
# Manual sidecar injection ဥပမာ
# Production တွင် auto-inject ကို သုံးသော်လည်း concept ရှင်းရန် manual ပြသည်
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        # Application container — business logic ပါသော main container
        - name: order-service
          image: order-service:v1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"

        # Sidecar proxy container — networking concerns handle ပြုလုပ်သည်
        - name: envoy-sidecar
          image: envoyproxy/envoy:v1.28
          ports:
            - containerPort: 15001   # Outbound traffic port
            - containerPort: 15006   # Inbound traffic port
            - containerPort: 15090   # Prometheus metrics port
          resources:
            requests:
              cpu: "100m"            # Sidecar ၏ CPU usage — lightweight
              memory: "128Mi"
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
            - name: certs
              mountPath: /etc/certs  # mTLS certificates mount
              readOnly: true

      # Init container — iptables rules set up ပြုလုပ်ပြီး traffic redirect
      initContainers:
        - name: istio-init
          image: istio/proxyv2:1.20
          command: ["istio-iptables"]
          args:
            - "-p"
            - "15001"               # Envoy outbound port
            - "-z"
            - "15006"               # Envoy inbound port
            - "-u"
            - "1337"                # Envoy user ID (iptables bypass)
          securityContext:
            capabilities:
              add: ["NET_ADMIN"]    # iptables modify ပြုလုပ်ရန် privilege
```

### Sidecar Pattern ၏ အားသာချက်/အားနည်းချက်

**အားသာချက်များ:**
- **Language agnostic** — Go, Java, Python မည်သည့် language နှင့်မဆို အလုပ်လုပ်သည်
- **Separation of concerns** — networking logic ကို app code နှင့် ခွဲထားသည်
- **Consistent policies** — service အားလုံးတွင် uniform security/observability
- **Zero code changes** — application ကို modify မလိုဘဲ mTLS, tracing ထည့်နိုင်သည်

**အားနည်းချက်များ:**
- **Latency overhead** — request တစ်ခုလျှင် sidecar hop 2 ခု ထပ်ပေါင်းသည် (~1-2ms)
- **Resource overhead** — Pod တစ်ခုလျှင် sidecar container ၏ CPU/memory ပိုကုန်သည်
- **Debugging complexity** — network issues debug ပြုလုပ်ရာတွင် sidecar layer ထပ်ပြီး ရှုပ်ထွေးသည်
- **Upgrade coordination** — sidecar version upgrade ပြုလုပ်သောအခါ Pod restart လိုသည်

---

## ၇.၆ East-West vs North-South Traffic

Microservices architecture တွင် traffic ကို direction ဖြင့် North-South နှင့် East-West ဟု ခွဲခြားသည်။

```
Traffic Direction Diagram:
                        NORTH
                          │
                    ┌─────▼─────┐
                    │    API    │
    External ──────>│  Gateway  │──────> External
    Clients         │           │        Clients
                    └─────┬─────┘
                          │ North-South Traffic
                          │ (Client ↔ Service)
          ════════════════╪══════════════════════
                          │
         ┌────────────────┼────────────────┐
         │                │                │
    ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
    │ Service │◄───►│ Service │◄───►│ Service │
    │    A    │     │    B    │     │    C    │
    └────┬────┘     └────┬────┘     └─────────┘
         │               │
         └───────────────┘
           East-West Traffic
           (Service ↔ Service)

                        SOUTH
```

### North-South Traffic

North-South traffic ဆိုသည်မှာ **external clients** (browsers, mobile apps, third-party systems) နှင့် **internal services** ကြားရှိ traffic ဖြစ်သည်။

**Characteristics:**
- API Gateway မှတဆင့် ဝင်ရောက်သည်
- TLS/SSL encrypted ဖြစ်သည်
- Authentication required ဖြစ်သည်
- Rate limiting apply ပြုလုပ်သည်
- Public internet ကို expose ပြုလုပ်သည်

### East-West Traffic

East-West traffic ဆိုသည်မှာ **internal services** ချင်း communicate ပြုလုပ်သော traffic ဖြစ်သည်။

**Characteristics:**
- Service mesh မှတဆင့် manage ပြုလုပ်သည်
- mTLS ဖြင့် encrypt ပြုလုပ်နိုင်သည်
- Internal service discovery ဖြင့် route ပြုလုပ်သည်
- Higher volume — N-S ထက် volume များစွာ ပိုများသည်
- Latency sensitive — internal communication ဖြစ်သောကြောင့် low latency လိုသည်

### N-S vs E-W Comparison

| Aspect | North-South | East-West |
|---|---|---|
| **Direction** | Client ↔ Service | Service ↔ Service |
| **Gateway** | API Gateway | Service Mesh / Sidecar |
| **Security** | TLS + Auth + Rate Limit | mTLS + AuthZ Policies |
| **Volume** | Lower (external requests) | Higher (internal calls) |
| **Protocol** | HTTP/HTTPS, REST | gRPC, HTTP, message queues |
| **Discovery** | DNS / Public endpoints | Service registry / DNS |
| **Latency** | Higher acceptable | Sub-millisecond preferred |

### Real-World Traffic Flow ဥပမာ

```
E-Commerce Order Flow — N-S နှင့် E-W Traffic:

[Browser] ──HTTPS──> [API Gateway]     ← North-South
                          │
                    ┌─────▼─────┐
                    │  Order    │
                    │  Service  │
                    └──┬──┬──┬──┘
          ┌────────────┘  │  └────────────┐
          ▼ (E-W)        ▼ (E-W)         ▼ (E-W)
   ┌──────────┐   ┌──────────┐    ┌──────────────┐
   │ Inventory│   │ Payment  │    │ Notification │
   │ Service  │   │ Service  │    │   Service    │
   └──────────┘   └────┬─────┘    └──────────────┘
                       │ (E-W)
                  ┌────▼─────┐
                  │  Fraud   │
                  │ Detection│
                  └──────────┘

Browser → API Gateway: North-South (HTTPS, JWT auth)
API Gateway → Order Service: N-S internal leg
Order → Inventory/Payment/Notification: East-West (gRPC/mTLS)
Payment → Fraud Detection: East-West (gRPC/mTLS)

Microservices architecture တွင် E-W traffic volume သည် N-S ထက်
10-100x ပိုများနိုင်သည်။ ထို့ကြောင့် service mesh ၏ E-W optimization
(connection pooling, load balancing, circuit breaking) သည်
system performance အတွက် အရေးကြီးသည်။
```

### API Gateway + Service Mesh ပေါင်းစပ်အသုံးပြုခြင်း

Production environment အများစုတွင် API Gateway (N-S) နှင့် Service Mesh (E-W) ကို ပေါင်းစပ်အသုံးပြုသည်။

```
Combined Architecture:
┌──────────────────────────────────────────────────────────┐
│                    NORTH-SOUTH LAYER                     │
│                                                          │
│  ┌────────────────────────────────────────────────┐      │
│  │              API Gateway (Kong/Nginx)           │      │
│  │  • SSL Termination  • Rate Limiting             │      │
│  │  • Authentication   • Request Routing           │      │
│  │  • Caching          • Response Transform        │      │
│  └────────────────────────────────────────────────┘      │
└──────────────────────────────────────────────────────────┘
                          │
┌──────────────────────────────────────────────────────────┐
│                    EAST-WEST LAYER                        │
│                  (Service Mesh / Istio)                   │
│                                                          │
│  ┌──────┐  mTLS   ┌──────┐  mTLS   ┌──────┐            │
│  │Svc A │◄───────►│Svc B │◄───────►│Svc C │            │
│  │+Envoy│         │+Envoy│         │+Envoy│            │
│  └──────┘         └──────┘         └──────┘            │
│                                                          │
│  • Mutual TLS       • Circuit Breaking                   │
│  • Load Balancing    • Retry Policies                    │
│  • Distributed Tracing • Traffic Shifting                │
└──────────────────────────────────────────────────────────┘
```

---

## အဓိက အချက်များ (Key Takeaways)

1. **API Gateway** သည် North-South traffic အတွက် single entry point ဖြစ်ပြီး authentication, rate limiting, SSL termination, routing တို့ကို centralized ပြုလုပ်သည်

2. **Gateway Patterns** သုံးမျိုး — Edge (simple), BFF (client-specific), Micro-Gateway (domain-specific) — project scale နှင့် team structure အလိုက် ရွေးချယ်သင့်သည်

3. **Kong** (plugin-rich, multi-cloud), **AWS API GW** (managed, serverless), **Nginx** (high-performance), **Envoy** (cloud-native, service mesh) — use case ပေါ်မူတည်ပြီး ရွေးချယ်သင့်သည်

4. **Service Mesh** (Istio/Linkerd) သည် East-West traffic ကို manage ပြုလုပ်ပြီး application code ကို မပြင်ဘဲ mTLS, observability, traffic management ထည့်သွင်းပေးသည်

5. **Sidecar Proxy Pattern** သည် service mesh ၏ foundation ဖြစ်ပြီး language-agnostic networking layer ကို provide ပြုလုပ်သည်

6. **North-South** (client ↔ service) traffic ကို API Gateway ဖြင့် manage ပြုလုပ်ပြီး **East-West** (service ↔ service) traffic ကို Service Mesh ဖြင့် manage ပြုလုပ်သည်

7. Production environment တွင် API Gateway + Service Mesh ပေါင်းစပ်အသုံးပြုခြင်းဖြင့် N-S နှင့် E-W traffic နှစ်မျိုးစလုံးကို effectively manage ပြုလုပ်နိုင်သည်

---

> **နောက်အခန်း Preview:** အခန်း ၈ တွင် Database per Service Pattern, Shared Database Anti-Pattern, Data Consistency strategies (Saga, Event Sourcing), နှင့် CQRS pattern တို့ကို ဆွေးနွေးသွားမည်ဖြစ်သည်။
