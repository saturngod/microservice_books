# အခန်း ၄ — REST & HTTP

## နိဒါန်း

REST (Representational State Transfer) သည် Roy Fielding ၂၀၀၀ ပြည့်နှစ်တွင် PhD dissertation ဖြင့် ဖော်ပြခဲ့သော architectural style တစ်ခုဖြစ်ပြီး HTTP protocol ကို အခြေခံ၍ distributed systems ကို ဆောက်တည်ရာတွင် ကျယ်ပြန့်စွာ အသုံးပြုနေကြသည်။ ၂၀၂၀ ခုနှစ်များတွင် gRPC, GraphQL တို့ ပေါ်ထွန်းလာသော်လည်း REST/HTTP သည် public API, mobile app backends, web applications တို့တွင် ယနေ့ထိ ထိပ်တန်းနေရာတွင် ရှိနေသေးသည်။ ဤအခန်းတွင် RESTful API ကို professional standard ဖြင့် design လုပ်ရန် လိုအပ်သော principles, versioning strategies, HATEOAS, pagination, HTTP/2  နှင့် HTTP/3 improvements, OpenAPI/Swagger contract-first design, နှင့် idempotency keys တို့ကို အသေးစိတ် ဆွေးနွေးသွားမည်ဖြစ်သည်။

---

## ၄.၁ RESTful API Design Principles

REST ၏ ခြောက်ချက်သော constraints (ကန့်သတ်ချက်) —

```
1. Uniform Interface    → Resources ကို standard ဖြင့် identify/manipulate
2. Stateless            → Server state ကို client session တွင် မထိန်းသိမ်း
3. Client-Server        → Concerns separation
4. Cacheable            → Response တွင် cacheability mark လုပ်ရမည်
5. Layered System       → Client ကို intermediate layer မသိပါ
6. Code on Demand       → (optional) server မှ executable code download
```

**HTTP Methods နှင့် ၎င်းတို့ ကိုက်ညီသော semantic**

| Method | Action | Idempotent | Safe |
|--------|--------|-----------|------|
| GET | Resource ဖတ်သည် | Yes | Yes |
| POST | Resource ဖန်တီးသည် | No | No |
| PUT | Resource ကို အပြည့်ပြောင်းသည် | Yes | No |
| PATCH | Resource ကို တစ်စိတ်တစ်ပိုင်း ပြောင်းသည် | မဆိုနိုင် | No |
| DELETE | Resource ဖျက်သည် | Yes | No |
| HEAD | Metadata ဖတ်သည် (body မပါ) | Yes | Yes |

**HTTP Status Codes Best Practices**

```
2xx Success
├── 200 OK              → GET, PUT, PATCH success
├── 201 Created         → POST - new resource ဖန်တီးသည်
├── 202 Accepted        → Async processing started
├── 204 No Content      → DELETE success (body မပါ)
└── 206 Partial Content → Range request

4xx Client Error
├── 400 Bad Request     → Validation failure
├── 401 Unauthorized    → Authentication required
├── 403 Forbidden       → Authorized but no permission
├── 404 Not Found       → Resource မတွေ့
├── 409 Conflict        → State conflict (duplicate)
├── 422 Unprocessable   → Semantic validation fail
└── 429 Too Many Reqs   → Rate limit exceeded

5xx Server Error
├── 500 Internal Error  → Unexpected server error
├── 502 Bad Gateway     → Upstream service error
├── 503 Unavailable     → Service down/overloaded
└── 504 Gateway Timeout → Upstream timeout
```

**ဥပမာ — Well-designed REST API Response**

```python
# api/orders.py
from fastapi import FastAPI, HTTPException, status
from pydantic import BaseModel
from typing import List, Optional
import uuid

app = FastAPI()

class OrderItem(BaseModel):
    product_id: str
    quantity: int

class CreateOrderRequest(BaseModel):
    customer_id: str
    items: List[OrderItem]

class OrderResponse(BaseModel):
    order_id: str
    customer_id: str
    status: str
    total: float
    created_at: str
    # HATEOAS links
    links: dict

@app.post("/api/v1/orders",
          status_code=status.HTTP_201_CREATED,
          response_model=OrderResponse)
async def create_order(request: CreateOrderRequest):
    order_id = str(uuid.uuid4())

    # Order ဖန်တီးလုပ်ငန်းစဉ်
    order = await order_service.create(request)

    # Response တွင် HATEOAS links ပါဝင်သည်
    return OrderResponse(
        order_id=order.id,
        customer_id=order.customer_id,
        status=order.status,
        total=order.total,
        created_at=order.created_at.isoformat(),
        links={
            "self": f"/api/v1/orders/{order.id}",
            "cancel": f"/api/v1/orders/{order.id}/cancel",
            "payment": f"/api/v1/orders/{order.id}/payment",
            "customer": f"/api/v1/customers/{order.customer_id}"
        }
    )
```

---

## ၄.၂ Resource Naming, Versioning, HATEOAS

### Resource Naming Conventions

```
✅ မှန်ကန်သော Resource Naming:
GET    /api/v1/orders              → Orders collection
GET    /api/v1/orders/{id}         → Specific order
POST   /api/v1/orders              → New order ဖန်တီး
PATCH  /api/v1/orders/{id}         → Order ပြင်ဆင်
DELETE /api/v1/orders/{id}         → Order ဖျက်
GET    /api/v1/orders/{id}/items   → Order items (nested resource)
POST   /api/v1/orders/{id}/cancel  → Action (verb လက်ခံသောနေရာ)

❌ မှားသော Resource Naming:
GET    /api/getOrders              → Verb ထည့်မထားပါနှင့်
GET    /api/order                  → Plural ထည့်ပါ
POST   /api/createNewOrder         → Verb မသုံးပါနှင့်
GET    /API/V1/ORDERS              → Lowercase ကိုသာ သုံးပါ
```

### API Versioning Strategies

```
Strategy 1: URL Path Versioning (ရိုးရှင်းဆုံး — ကျင့်သုံးအများဆုံး)
GET /api/v1/orders
GET /api/v2/orders

Strategy 2: Header Versioning
GET /api/orders
Accept: application/vnd.myapp.v2+json

Strategy 3: Query Parameter Versioning
GET /api/orders?version=2

Strategy 4: Subdomain Versioning
https://v2.api.example.com/orders
```

**Versioning Best Practices**

```python
# versioning/router.py

from fastapi import APIRouter

# Version 1 Router
v1_router = APIRouter(prefix="/api/v1")

@v1_router.get("/orders/{order_id}")
async def get_order_v1(order_id: str):
    # V1 response format (legacy)
    order = await order_service.get(order_id)
    return {
        "id": order.id,
        "status": order.status,
        "amount": order.total  # V1  တွင် "amount" ဟု ခေါ်သည်
    }

# Version 2 Router — breaking changes
v2_router = APIRouter(prefix="/api/v2")

@v2_router.get("/orders/{order_id}")
async def get_order_v2(order_id: str):
    # V2 response format — richer, different structure
    order = await order_service.get(order_id)
    return {
        "order_id": order.id,      # V2  တွင် "order_id" ဟု ပြောင်းသည်
        "status": order.status,
        "total": {                  # V2  တွင် nested object
            "amount": order.total,
            "currency": order.currency
        },
        "links": { ... }           # V2  တွင် HATEOAS ပေါင်းထည့်
    }
```

### HATEOAS (Hypermedia as the Engine of Application State)

**HATEOAS** ဆိုသည်မှာ API response တွင် next possible actions ၏ links ပါဝင်ရမည်ဆိုသောသဘောတရားဖြစ်သည်။ Client သည် API structure ကို hard-code မလုပ်ဘဲ response မှ navigateလုပ်နိုင်သည်။

```json
// GET /api/v1/orders/ord_123

{
  "order_id": "ord_123",
  "status": "PENDING_PAYMENT",
  "total": { "amount": 99.99, "currency": "USD" },
  "_links": {
    "self": { "href": "/api/v1/orders/ord_123" },
    "pay": {
      "href": "/api/v1/orders/ord_123/payment",
      "method": "POST"
    },
    "cancel": {
      "href": "/api/v1/orders/ord_123/cancel",
      "method": "POST"
    },
    "customer": { "href": "/api/v1/customers/cust_456" }
  }
}
```

---

## ၄.၃ Pagination, Filtering, Sorting at Scale

Large dataset ဖြင့် API ဆောက်တည်ရာတွင် pagination မပါပါက performance ပြဿနာ ဖြစ်ပေါ်မည်ဖြစ်သည်။

### Pagination Strategies

```
Strategy 1: Offset-based Pagination (ရိုးရှင်းဆုံး)
GET /api/v1/orders?offset=0&limit=20
GET /api/v1/orders?page=2&per_page=20

Drawbacks:
- Large offset ဆိုလျှင် slow (DB scan)
- Data ပြောင်းလဲနေသောအခါ inconsistent results

Strategy 2: Cursor-based Pagination (Scale ကောင်းဆုံး)
GET /api/v1/orders?cursor=eyJpZCI6MTAwfQ&limit=20

Advantages:
- O(1) — offset မဆိုင်ဘဲ fast
- Consistent results during data mutations
```

**Cursor Pagination Implementation**

```python
# pagination.py
import base64, json
from typing import List, Optional

class CursorPagination:
    def encode_cursor(self, order_id: str, created_at: str) -> str:
        # Cursor ကို Base64 encode လုပ်သည်
        cursor_data = json.dumps({"id": order_id, "created_at": created_at})
        return base64.b64encode(cursor_data.encode()).decode()

    def decode_cursor(self, cursor: str) -> dict:
        decoded = base64.b64decode(cursor.encode()).decode()
        return json.loads(decoded)

@app.get("/api/v1/orders")
async def list_orders(
    limit: int = 20,
    cursor: Optional[str] = None,
    # Filtering parameters
    status: Optional[str] = None,
    customer_id: Optional[str] = None,
    # Sorting parameters
    sort_by: str = "created_at",
    sort_order: str = "desc"
):
    # Cursor ဖြင့် query ပြုလုပ်သည်
    orders = await order_repo.find_with_cursor(
        cursor=cursor,
        limit=limit + 1,  # +1 ဖြင့် next page ရှိမရှိ စစ်သည်
        filters={"status": status, "customer_id": customer_id},
        sort={sort_by: sort_order}
    )

    has_next = len(orders) > limit
    if has_next:
        orders = orders[:limit]  # Extra item ဖြတ်သည်

    # Cursor Pagination Response
    return {
        "data": orders,
        "pagination": {
            "limit": limit,
            "has_next_page": has_next,
            "next_cursor": cursor_pagination.encode_cursor(
                orders[-1].id, orders[-1].created_at
            ) if has_next else None
        }
    }
```

**Filtering & Sorting**

```
Advanced Filtering Examples:
GET /api/v1/orders?status=PENDING&created_after=2024-01-01
GET /api/v1/orders?total_min=100&total_max=500
GET /api/v1/products?category=electronics&price[lt]=100

Sorting:
GET /api/v1/orders?sort=-created_at        → Descending (-prefix)
GET /api/v1/orders?sort=total,-created_at  → Multiple fields

Field Selection (reduce payload):
GET /api/v1/orders?fields=order_id,status,total
```

---

## ၄.၄ HTTP/2 နှင့် HTTP/3

### HTTP/2 (2015)

HTTP/1.1 ၏ performance limitations ကို ဖြေရှင်းရန် HTTP/2 ၂၀၁၅ ခုနှစ်တွင် မိတ်ဆက်ခဲ့သည်။

```
HTTP/1.1 Problems:
├── Head-of-line blocking (ဦးတည်သောအတန်းဆိုင်ပိတ်ဆို့မှု)
├── Multiple TCP connections လိုအပ်
├── Header ကို text format ဖြင့် ပြန်ပြန်ပို့ရသည်
└── Server push မရှိ

HTTP/2 Solutions:
┌──────────────────────────────────────────────────────┐
│  Single TCP Connection                               │
│  ┌──────────────────────────────────────────────┐   │
│  │  Stream 1: GET /orders    → Response         │   │
│  │  Stream 3: GET /products  → Response         │   │
│  │  Stream 5: GET /user      → Response         │   │
│  └──────────────────────────────────────────────┘   │
│  (Multiplexing — တပြိုင်နက် requests များ)            │
└──────────────────────────────────────────────────────┘

HTTP/2 Features:
├── Multiplexing      → Single connection ဖြင့် concurrent requests
├── Header Compression → HPACK format — header size လျှော့ချ
├── Binary Protocol   → Text ထက် efficient
├── Stream Prioritization → Important requests ကို ဦးစားပေး
└── Flow Control      → Slow receiver များအတွက် backpressure ပေးနိုင်
```

Server Push ကို HTTP/2 specification တွင် define လုပ်ထားသော်လည်း modern browser ecosystem တွင် support နှင့် adoption နှစ်မျိုးစလုံး နည်းလာသောကြောင့် API design ၏ primary selling point အဖြစ် မယူသင့်ပါ။

### HTTP/3 (QUIC)

```
HTTP/3 သည် TCP ကို QUIC (UDP-based) ဖြင့် အစားထိုးသည်

Advantages over HTTP/2:
├── 0-RTT connection resumption  → မြန်ဆန်သော reconnect
├── Better packet loss recovery  → Wireless networks ကောင်းသည်
├── Connection migration         → IP ပြောင်းသော်လည်း session မပျက်
└── Eliminates TCP HOL blocking  → Per-stream independent delivery

Performance Impact:
HTTP/1.1  ── 300ms first byte
HTTP/2    ── 150ms first byte
HTTP/3    ── 80ms first byte (QUIC 0-RTT)
```

---

## ၄.၅ OpenAPI/Swagger Contract-First Design

**Contract-First Design** ဆိုသည်မှာ code မရေးမီ API specification (contract) ကို အရင်ရေးသောနည်းလမ်းဖြစ်သည်။

```
Traditional (Code-First):
Code → Auto-generate Docs → Client uses (docs တိကျမှုမရှိ)

Contract-First:
API Spec (OpenAPI) → Server implements → Client generates SDK
                  ↑ Single source of truth
```

**OpenAPI 3.0 Specification ဥပမာ**

```yaml
# openapi.yaml
openapi: 3.0.3
info:
  title: Order Service API
  version: "1.0"
  description: Microservices Order Management API

paths:
  /api/v1/orders:
    post:
      summary: Order အသစ် ဖန်တီးသည်
      operationId: createOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        "201":
          description: Order successfully created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        "400":
          description: Validation error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        "422":
          description: Business rule violation

components:
  schemas:
    CreateOrderRequest:
      type: object
      required: [customer_id, items]
      properties:
        customer_id:
          type: string
          example: "cust_abc123"
        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItem'

    OrderItem:
      type: object
      required: [product_id, quantity]
      properties:
        product_id:
          type: string
        quantity:
          type: integer
          minimum: 1

    OrderResponse:
      type: object
      properties:
        order_id:
          type: string
        status:
          type: string
          enum: [PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED]
        total:
          $ref: '#/components/schemas/Money'

    Money:
      type: object
      properties:
        amount:
          type: number
        currency:
          type: string
          example: "USD"

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

---

## ၄.၆ Idempotency Keys

**Idempotency** ဆိုသည်မှာ တူညီသော operation ကို တစ်ကြိမ် (သို့) ထပ်ခါတလဲလဲ execute လုပ်သော်လည်း result တူညီသည်ဟူသော property ဖြစ်သည်။

**ဘာကြောင့် Idempotency Keys လိုသနည်း?**

```
Problem Scenario:
Client → POST /orders → Network Error → Client မသိ (Order ဖြစ်ပြီးလား မဖြစ်ဘူးလား)
                                      → Order ဖန်တီးပြီးပြီလား?
                                      → Retry လုပ်မလား?

Without Idempotency:        With Idempotency Key:
POST /orders  → order_1     POST /orders (key: abc) → order_1
POST /orders  → order_2     POST /orders (key: abc) → order_1 (same!)
(Duplicate orders!)         (Duplicate prevention ✓)
```

**Implementation**

```python
# idempotency.py
import hashlib
from datetime import datetime, timedelta

class IdempotencyService:
    def __init__(self, cache):  # Redis cache
        self.cache = cache
        self.ttl = timedelta(hours=24)  # 24 နာရီ ထိန်းသိမ်းသည်

    async def check_or_execute(
        self,
        idempotency_key: str,
        operation_fn,
        *args, **kwargs
    ):
        cache_key = f"idempotency:{idempotency_key}"

        # Cache ထဲ ရှိပြီးပြီဆိုလျှင် stored response ကို return
        cached = await self.cache.get(cache_key)
        if cached:
            return json.loads(cached)  # Previous result return

        # Operation execute လုပ်သည်
        result = await operation_fn(*args, **kwargs)

        # Result ကို 24 နာရီ cache တွင် သိမ်းသည်
        await self.cache.setex(
            cache_key,
            int(self.ttl.total_seconds()),
            json.dumps(result)
        )
        return result

# FastAPI endpoint တွင် Idempotency Key ထည့်သုံး
@app.post("/api/v1/orders", status_code=201)
async def create_order(
    request: CreateOrderRequest,
    idempotency_key: str = Header(..., alias="Idempotency-Key")
):
    """
    Header တွင် Idempotency-Key ပေးသောကြောင့်
    Client retry လုပ်ပါက same result ပြန်ရမည်
    """
    return await idempotency_service.check_or_execute(
        idempotency_key=idempotency_key,
        operation_fn=order_service.create_order,
        request=request
    )
```

**Client-side Idempotency Key Generation**

```python
# client.py
import uuid

class OrderApiClient:
    async def create_order_safely(self, order_data: dict) -> dict:
        # Unique idempotency key ဖန်တီးသည်
        idempotency_key = str(uuid.uuid4())

        max_retries = 3
        for attempt in range(max_retries):
            try:
                response = await self.http_client.post(
                    "/api/v1/orders",
                    json=order_data,
                    headers={"Idempotency-Key": idempotency_key}
                    # Same key ဖြင့် retry လုပ်ပါက same result ရမည်
                )
                return response.json()
            except NetworkError:
                if attempt == max_retries - 1:
                    raise
                await asyncio.sleep(2 ** attempt)
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **RESTful Design** — HTTP methods ကို semantic ဖြင့် မှန်ကန်စွာ သုံးပါ (GET ဖတ်ရန်, POST ဖန်တီးရန်, PUT အပြည့်ပြင်ရန်, PATCH တစ်စိတ်ပြင်ရန်, DELETE ဖျက်ရန်)။ Status codes ကို သင့်တော်သောအတိုင်း return ပြုပါ။
2. **Resource Naming** — Nouns (plural) သုံးပါ၊ lowercase-kebab-case ကိုင်ပါ၊ verbs ကို URL တွင် မထည့်ပါ။
3. **Versioning** — URL path versioning (/api/v1/) သည် ရိုးရှင်းပြီး ကျင့်သုံးအများဆုံးဖြစ်သည်၊ breaking changes ဖြစ်မှ version တိုးပါ။
4. **Pagination** — Production scale ဆိုပါက offset-based မဟုတ်ဘဲ cursor-based pagination ကို ကိုင်ပါ။
5. **HTTP/2** — Multiplexing, header compression, flow control တို့ဖြင့် performance နှင့် connection efficiency မြင့်တက်သည်; modern APIs သည် HTTP/2 ကို default ဖြစ်သင့်သည်။
6. **OpenAPI Contract-First** — Code မရေးမီ spec ရေးခြင်းဖြင့် team ကြား contract တိကျမှုရပြီး SDK auto-generation လုပ်နိုင်သည်။
7. **Idempotency Keys** — Payment, order creation ကဲ့သို့ critical operations တွင် Idempotency Key ထည့်ပါ — network retry ကြောင့် duplicate operations ဖြစ်ပေါ်ခြင်းကို ကာကွယ်သည်။
