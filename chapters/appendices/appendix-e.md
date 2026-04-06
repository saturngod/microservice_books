# နောက်ဆက်တွဲ E: API Protocol နှိုင်းယှဉ်ဇယား

---

## E.1 အဓိက API Protocol နှိုင်းယှဉ်ဇယား

| နှိုင်းယှဉ်မှုအတိုင်းအတာ | **REST** | **gRPC** | **GraphQL** | **WebSocket** | **SSE** (Server-Sent Events) |
|------------------------|---------|---------|------------|--------------|---------------------------|
| **Underlying Protocol** | HTTP/1.1 သို့မဟုတ် HTTP/2 | HTTP/2 (မဖြစ်မနေ) | HTTP/1.1 သို့မဟုတ် HTTP/2 | TCP (HTTP Upgrade) | HTTP/1.1 သို့မဟုတ် HTTP/2 |
| **Data Format** | JSON (ပုံမှန်), XML, plain text | Protocol Buffers (binary) | JSON | Binary / Text (custom) | Plain text (UTF-8) |
| **Streaming** | Limited (HTTP chunking) | Bidirectional streaming (4 modes) | Subscription (limited) | Full bidirectional | Server → Client only (one-way) |
| **Browser Native Support** | Yes | Limited (grpc-web proxy လိုအပ်) | Yes (HTTP ဖြင့်) | Yes (WebSocket API) | Yes (EventSource API) |
| **Type Safety** | No (OpenAPI/Swagger ဖြင့် optional) | Yes (`.proto` schema) | Yes (schema + type system) | No | No |
| **Code Generation** | Optional (OpenAPI generator) | Yes (protoc compiler) | Optional (GraphQL codegen) | No | No |
| **Performance** | Medium | High (binary, multiplexed) | Medium (over-fetching ဖြစ်နိုင်) | High (persistent connection) | Medium |
| **Caching** | Easy (HTTP cache headers) | ခက်ခဲ (POST-like semantics) | Complex (query-level) | မဖြစ်နိုင် | HTTP cache ဖြင့် limited |
| **Error Handling** | HTTP status codes (4xx, 5xx) | gRPC status codes (14 types) | HTTP 200 + errors in body | Custom | HTTP status codes |
| **Versioning** | URL path / header | Package namespace in .proto | Schema evolution (additive) | Custom protocol version | N/A |
| **Security** | HTTPS, JWT, OAuth2 | TLS, mTLS | HTTPS, JWT | WSS (WebSocket Secure) | HTTPS |
| **Firewall / Proxy Friendly** | Yes | Partial (HTTP/2 ကို block တတ်) | Yes | Partial (proxy config လိုတတ်) | Yes |
| **Load Balancing** | Easy (stateless) | Complex (L7 load balancer လိုအပ်) | Easy | Sticky session လိုတတ်သည် | Sticky session လိုတတ်သည် |
| **Tooling Maturity** | Very High | High | High | Medium | Medium |
| **Learning Curve** | Low | Medium | Medium-High | Low-Medium | Low |

---

## E.2 Streaming Mode ရှင်းလင်းချက်

### REST — ကန့်သတ်ချက်ရှိသော streaming

```
Client          Server
  │── GET /data ──→│
  │                │  (process)
  │←── 200 OK ────│
  │←── [chunk 1] ─│
  │←── [chunk 2] ─│
  │←── [end]   ───│
(HTTP chunked transfer — one direction only)
```

### gRPC — Bidirectional Streaming (4 Modes)

```
1. Unary RPC
   Client ──request──→ Server ──response──→ Client

2. Server Streaming
   Client ──request──→ Server ──stream───→ Client
                              ──stream───→
                              ──stream───→

3. Client Streaming
   Client ──stream───→ Server ──response──→ Client
          ──stream───→
          ──stream───→

4. Bidirectional Streaming
   Client ⇄──stream───⇄ Server  (همزمان both ways)
```

### WebSocket — Full Duplex

```
Client                     Server
  │── HTTP Upgrade req ──→ │
  │←── 101 Switching ────  │
  │                         │
  │←────── message ─────── │  (server push)
  │──────── message ──────→│  (client send)
  │←────── message ─────── │
  (persistent TCP connection)
```

### SSE — Server Push Only

```
Client                     Server
  │── GET /events ────────→│
  │←── 200 text/event-stream│
  │←── data: event1\n\n ── │
  │←── data: event2\n\n ── │
  │←── data: event3\n\n ── │
  (one-way: server → client only)
  (auto-reconnect built-in)
```

---

## E.3 Protocol ရွေးချယ်မှု — Use Case မူတည်

| Use Case | အကြံပြု Protocol | အကြောင်းပြချက် |
|----------|----------------|--------------|
| Public API (third-party developer) | **REST** | Documentation, ecosystem, tooling mature |
| Internal microservice-to-microservice | **gRPC** | Performance, type safety, code gen |
| Mobile app backend (flexible queries) | **GraphQL** | Client controls data shape, reduce over-fetching |
| Real-time chat / collaborative editing | **WebSocket** | Bidirectional, low latency |
| Live sports score / stock ticker | **SSE** | Simple, server-push only, auto-reconnect |
| File upload / download | **REST** (multipart) | HTTP standard, CDN support |
| Streaming analytics ingestion | **gRPC** (client streaming) | High throughput, binary efficiency |
| Dashboard live updates | **SSE** သို့မဟုတ် **WebSocket** | Push updates without polling |
| BFF (Backend-for-Frontend) | **GraphQL** | Aggregate multiple services, client-driven |
| IoT device telemetry | **gRPC** သို့မဟုတ် MQTT | Binary efficiency, streaming |
| Browser-based game | **WebSocket** | Bidirectional, real-time |
| Webhook notification | **REST** (HTTP POST) | Simple, widely supported |

---

## E.4 gRPC Protocol Buffers ဥပမာ

```protobuf
// user.proto
syntax = "proto3";

package user.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);               // Unary
  rpc ListUsers(ListUsersRequest) returns (stream UserResponse);       // Server streaming
  rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse); // Client streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);           // Bidirectional
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  string user_id = 1;
  string name    = 2;
  string email   = 3;
  int64  created_at = 4;
}
```

---

## E.5 REST vs gRPC — Payload Size နှင့် Speed

| Metric | REST (JSON) | gRPC (Protobuf) | ကွာခြားချက် |
|--------|------------|----------------|-----------|
| Payload size (ပုံမှန် object) | ~200 bytes | ~50 bytes | gRPC ~75% ငယ်သည် |
| Serialization speed | ~1x | ~5-10x faster | Protobuf binary parsing မြန် |
| Network roundtrips | 1 per request | 1 per request (+ streaming) | Streaming ဖြင့် gRPC သာသည် |
| HTTP/2 multiplexing | Optional | မဖြစ်မနေ | gRPC က HOL blocking ဖယ်ရှား |
| Compression | gzip (optional) | Built-in (proto compression) | — |

---

## E.6 GraphQL — Over-fetching / Under-fetching ဖြေရှင်းချက်

### REST ၏ ပြဿနာ

```json
// Client needs: name + email only
// REST returns everything:
GET /users/123
{
  "id": "123",
  "name": "Aung Aung",
  "email": "aung@example.com",
  "address": { ... },       // မလိုဘဲ ပါလာ (over-fetching)
  "orders": [ ... ],        // မလိုဘဲ ပါလာ
  "preferences": { ... }    // မလိုဘဲ ပါလာ
}
```

### GraphQL ဖြေရှင်းချက်

```graphql
# Client သည် သူလိုသောအရာကိုသာ တောင်းဆို
query {
  user(id: "123") {
    name
    email
  }
}

# Response — requested fields only:
{
  "data": {
    "user": {
      "name": "Aung Aung",
      "email": "aung@example.com"
    }
  }
}
```

---

## E.7 Error Handling နှိုင်းယှဉ်

### REST HTTP Status Codes

| Code | အဓိပ္ပာယ် |
|------|---------|
| 200 OK | အောင်မြင်သော request |
| 201 Created | Resource အသစ် ဖန်တီးပြီး |
| 400 Bad Request | Client request မှားယွင်း |
| 401 Unauthorized | Authentication မရှိ |
| 403 Forbidden | Authorization မရ |
| 404 Not Found | Resource မတွေ့ |
| 409 Conflict | State conflict (idempotency) |
| 422 Unprocessable Entity | Validation error |
| 429 Too Many Requests | Rate limit ကျော် |
| 500 Internal Server Error | Server-side error |
| 503 Service Unavailable | Service down / overloaded |

### gRPC Status Codes

| Code | တန်ဖိုး | အဓိပ္ပာယ် |
|------|--------|---------|
| OK | 0 | အောင်မြင် |
| INVALID_ARGUMENT | 3 | Request argument မှားယွင်း |
| NOT_FOUND | 5 | Resource မတွေ့ |
| ALREADY_EXISTS | 6 | Resource ရှိပြီးသား |
| PERMISSION_DENIED | 7 | Authorization မရ |
| RESOURCE_EXHAUSTED | 8 | Rate limit / quota ကျော် |
| FAILED_PRECONDITION | 9 | System state မပြည့်မီ |
| UNAUTHENTICATED | 16 | Authentication မရှိ |
| UNAVAILABLE | 14 | Service ရယူမရ |
| DEADLINE_EXCEEDED | 4 | Timeout |
| INTERNAL | 13 | Server-side error |

---

## E.8 Protocol Selection Decision Tree

```
API ဒီဇိုင်း ဘာသုံးမည်?
│
├── External / Public API ဖြစ်သလား?
│   ├── Client data shape flexible ဖြစ်ဖို့ လိုသလား? → GraphQL
│   └── Standard CRUD / resource-based?              → REST
│
├── Internal microservice-to-microservice ဖြစ်သလား?
│   ├── Performance critical, type safety လိုသလား?  → gRPC
│   └── Simpler, HTTP-friendly?                      → REST
│
└── Real-time / event-driven ဖြစ်သလား?
    ├── Server push only (notifications, feeds)?     → SSE
    └── Bidirectional (chat, gaming, collaboration)? → WebSocket
```

---

## E.9 Security Comparison

| Security Aspect | REST | gRPC | GraphQL | WebSocket | SSE |
|----------------|------|------|---------|-----------|-----|
| **Transport Encryption** | TLS (HTTPS) | TLS (မဖြစ်မနေ) | TLS (HTTPS) | TLS (WSS) | TLS (HTTPS) |
| **Authentication** | JWT, OAuth2, API Key | TLS cert, JWT (metadata) | JWT, OAuth2 | JWT (handshake), cookies | JWT, cookies |
| **mTLS Support** | Optional | Built-in (recommended) | Optional | Partial | Partial |
| **Authorization** | Middleware / decorator | Interceptor | Resolver-level | Middleware | Middleware |
| **CSRF Protection** | CSRF token | N/A (no cookies) | CSRF token | Origin check | CSRF token |
| **Rate Limiting** | API Gateway level | Interceptor | Complexity limiting | Per-connection | Per-connection |
