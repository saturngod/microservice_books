# အခန်း ၅ — gRPC နှင့် Protocol Buffers

## နိဒါန်း

gRPC (Google Remote Procedure Call) သည် Google မှ ၂၀၁၅ ခုနှစ်တွင် open source ထုတ်ပြန်ခဲ့သော high-performance, language-agnostic remote procedure call framework တစ်ခုဖြစ်သည်။ HTTP/2 ကို transport layer အဖြစ် အသုံးပြုပြီး Protocol Buffers (protobuf) ကို serialization format အဖြစ် ကိုင်တွယ်သည်။ Microservices internal communication တွင် REST/HTTP ထက် gRPC ကို ရွေးချယ်ရသောအကြောင်းရင်းမှာ performance, type safety, bidirectional streaming, နှင့် multi-language support တို့ကြောင့်ဖြစ်သည်။ Netflix, Cisco, Cockroach Labs, CoreOS တို့ကဲ့သို့ large-scale organizations များသည် internal service communication တွင် gRPC ကို တွင်ကျယ်စွာ အသုံးပြုနေကြသည်။ ဤအခန်းတွင် gRPC ၏ architecture, Protocol Buffers schema design, streaming modes, production-ready features, နှင့် REST နှင့် ယှဉ်ခြင်းတို့ကို ဆွေးနွေးသွားမည်ဖြစ်သည်။

---

## ၅.၁ ဘာကြောင့် Internal Services တွင် gRPC ကို REST ထက် ရွေးသနည်း

**Performance Comparison**

```
REST/HTTP/1.1 (JSON):
├── Text-based encoding    → Verbose, large payload
├── HTTP/1.1 limitations  → Per-request connection overhead
└── Schema validation      → Runtime only

gRPC (Protocol Buffers):
├── Binary encoding        → 3-10x smaller payload
├── HTTP/2 multiplexing   → Single connection, concurrent streams
├── Code generation        → Compile-time type checking
└── Native streaming       → Built-in streaming support

Benchmark (1000 RPS, simple object):
┌───────────────────┬──────────────┬───────────────┬──────────────┐
│ Metric            │ REST/JSON    │ gRPC/protobuf │ Improvement  │
├───────────────────┼──────────────┼───────────────┼──────────────┤
│ Latency (p99)     │ 45ms         │ 12ms          │ ~3.7x faster │
│ Throughput        │ 2,000 RPS    │ 8,500 RPS     │ ~4.25x more  │
│ Payload size      │ 1,200 bytes  │ 280 bytes     │ ~4.3x smaller│
│ CPU usage         │ 100%         │ 35%           │ ~2.9x less   │
└───────────────────┴──────────────┴───────────────┴──────────────┘
```

**ဘယ်နေရာတွင် gRPC ကောင်းသနည်း**

```
gRPC သင့်တော်သော Use Cases:
✅ Microservice-to-Microservice internal calls
✅ Polyglot environments (Go, Java, Python, Node.js)
✅ Real-time streaming (IoT data, live updates)
✅ Mobile apps ← Backend (Android/iOS grpc client libraries ရှိ)
✅ Low-latency, high-throughput requirements

gRPC မသင့်တော်သော Use Cases:
❌ Browser clients (gRPC-Web middleware လိုသည်)
❌ Simple CRUD APIs for external partners
❌ Human-readable debugging requirement
❌ Legacy systems without HTTP/2 support
```

---

## ၅.၂ Protocol Buffers Schema နှင့် Serialization

Protocol Buffers (protobuf) သည် Google မှ develop လုပ်သော language-neutral, platform-neutral, extensible serialization format တစ်ခုဖြစ်သည်။

**Proto file (.proto) Structure**

```protobuf
// order_service.proto
syntax = "proto3";  // Proto3 syntax သုံးသည် (proto2 မဟုတ်)

package order.v1;  // Package namespace

// Go, Java, Python ကဲ့သို့ languages အတွက် option settings
option go_package = "github.com/company/order-service/gen/order/v1";
option java_package = "com.company.orderservice.v1";

// Import other proto files
import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

// Enum definition — Order status
enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;  // Default value (proto3 convention)
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_CONFIRMED = 2;
    ORDER_STATUS_SHIPPED = 3;
    ORDER_STATUS_DELIVERED = 4;
    ORDER_STATUS_CANCELLED = 5;
}

// Message definition — Value Object
message Money {
    int64 amount_cents = 1;  // Field number (1 = ပထမဆုံး field)
    string currency_code = 2;  // "USD", "MMK"
}

// Message definition — Order Item
message OrderItem {
    string product_id = 1;
    int32 quantity = 2;
    Money unit_price = 3;  // Nested message
}

// Message definition — Order
message Order {
    string order_id = 1;
    string customer_id = 2;
    repeated OrderItem items = 3;  // repeated = List/Array
    OrderStatus status = 4;
    Money total = 5;
    google.protobuf.Timestamp created_at = 6;  // Timestamp type
}

// Request/Response messages
message CreateOrderRequest {
    string customer_id = 1;
    repeated OrderItem items = 2;
    string idempotency_key = 3;  // Optional field
}

message CreateOrderResponse {
    Order order = 1;
}

message GetOrderRequest {
    string order_id = 1;
}

message ListOrdersRequest {
    string customer_id = 1;
    int32 page_size = 2;
    string page_token = 3;  // Cursor-based pagination
}

message ListOrdersResponse {
    repeated Order orders = 1;
    string next_page_token = 2;
}
```

**Serialization ဥပမာ**

```python
# serialization_demo.py

import order_service_pb2  # protoc မှ generate လုပ်သော code

# Protobuf object ဖန်တီးသည်
order = order_service_pb2.Order(
    order_id="ord_abc123",
    customer_id="cust_xyz",
    status=order_service_pb2.ORDER_STATUS_PENDING
)

# Binary serialization
binary_data = order.SerializeToString()
print(f"Protobuf size: {len(binary_data)} bytes")  # ~25 bytes

# JSON comparison
import json
json_data = json.dumps({
    "order_id": "ord_abc123",
    "customer_id": "cust_xyz",
    "status": "ORDER_STATUS_PENDING"
})
print(f"JSON size: {len(json_data)} bytes")  # ~75 bytes

# Deserialization
order2 = order_service_pb2.Order()
order2.ParseFromString(binary_data)
```

---

## ၅.၃ Unary, Server Streaming, Client Streaming, Bidirectional

gRPC ၌ RPC types လေးမျိုးရှိသည်။

### Unary RPC (တစ်ကြိမ် တစ်ကြိမ် ဆက်သွယ်မှု)

```protobuf
// Unary: Request တစ်ခု → Response တစ်ခု
service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
    rpc GetOrder(GetOrderRequest) returns (Order);
}
```

```python
# server_implementation.py
import grpc
import order_service_pb2_grpc
import order_service_pb2

class OrderServiceImpl(order_service_pb2_grpc.OrderServiceServicer):
    async def CreateOrder(self, request, context):
        # Unary RPC Handler
        try:
            order = await self.order_repo.create(
                customer_id=request.customer_id,
                items=request.items
            )
            return order_service_pb2.CreateOrderResponse(order=order)
        except ValueError as e:
            # gRPC error response
            await context.abort(grpc.StatusCode.INVALID_ARGUMENT, str(e))
```

### Server Streaming RPC

```protobuf
// Server Streaming: Request တစ်ခု → Response stream
service OrderService {
    rpc WatchOrderStatus(WatchOrderRequest)
        returns (stream OrderStatusUpdate);
}
```

```python
# server_streaming_handler.py

async def WatchOrderStatus(self, request, context):
    """
    Order status ပြောင်းလဲတိုင်း server မှ client သို့ push လုပ်သည်
    """
    order_id = request.order_id

    async for status_update in self.order_events.subscribe(order_id):
        # Stream ဖြင့် updates ကို client သို့ ပို့သည်
        yield order_service_pb2.OrderStatusUpdate(
            order_id=order_id,
            status=status_update.status,
            message=status_update.message
        )
        # Order terminal state ဆိုလျှင် stream ပိတ်သည်
        if status_update.status in ("DELIVERED", "CANCELLED"):
            break

# Client side
async def watch_order(channel, order_id: str):
    stub = order_service_pb2_grpc.OrderServiceStub(channel)
    async for update in stub.WatchOrderStatus(
        order_service_pb2.WatchOrderRequest(order_id=order_id)
    ):
        print(f"Order {order_id} status: {update.status}")
```

### Client Streaming RPC

```protobuf
// Client Streaming: Request stream → Response တစ်ခု
service InventoryService {
    rpc BulkUpdateInventory(stream InventoryUpdate)
        returns (BulkUpdateResponse);
}
```

```python
# client_streaming_handler.py

async def BulkUpdateInventory(self, request_iterator, context):
    """
    Client မှ inventory updates stream ကို လက်ခံပြီး summary return
    """
    success_count = 0
    failed_count = 0

    async for update in request_iterator:
        # Client မှ တစ်ခုချင်း updates ကို process လုပ်သည်
        try:
            await self.inventory_repo.update(
                product_id=update.product_id,
                quantity_delta=update.quantity_delta
            )
            success_count += 1
        except Exception:
            failed_count += 1

    return inventory_service_pb2.BulkUpdateResponse(
        success_count=success_count,
        failed_count=failed_count
    )
```

### Bidirectional Streaming RPC

```protobuf
// Bidirectional: Request stream → Response stream (တပြိုင်နက်)
service ChatService {
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

```python
# bidirectional_streaming.py

async def Chat(self, request_iterator, context):
    """
    Chat — client နှင့် server နှစ်ဖက်ကနေ တပြိုင်နက် message ပို့နိုင်သည်
    """
    async def send_messages():
        # Server မှ responses generate
        async for msg in self.message_broadcaster.subscribe():
            yield chat_service_pb2.ChatMessage(
                sender="server",
                content=msg,
                timestamp=get_timestamp()
            )

    async def receive_messages():
        # Client မှ messages receive
        async for request in request_iterator:
            await self.message_broadcaster.broadcast(
                f"{request.sender}: {request.content}"
            )

    # Concurrent send and receive
    await asyncio.gather(
        send_messages(),
        receive_messages()
    )
```

---

## ၅.၄ Deadlines, Cancellation, Interceptors

### Deadlines (Timeouts)

```python
# deadlines_demo.py
import grpc
from datetime import timedelta

async def call_with_deadline():
    async with grpc.aio.insecure_channel("order-service:50051") as channel:
        stub = order_service_pb2_grpc.OrderServiceStub(channel)

        try:
            # 5 seconds deadline သတ်မှတ်သည်
            response = await stub.CreateOrder(
                request,
                timeout=5.0  # seconds
            )
            return response
        except grpc.RpcError as e:
            if e.code() == grpc.StatusCode.DEADLINE_EXCEEDED:
                # Timeout ဖြစ်ပါက fallback logic
                print("Service timeout! Fallback ကို သုံးမည်")
                raise TimeoutError("Order service did not respond")
```

### Interceptors (Middleware equivalent)

```python
# interceptors.py
import grpc
import time
import logging

class LoggingInterceptor(grpc.aio.ServerInterceptor):
    """
    gRPC Server Interceptor — request/response logging
    """
    async def intercept_service(self, continuation, handler_call_details):
        start_time = time.time()
        method = handler_call_details.method

        # Pre-processing
        logging.info(f"gRPC call started: {method}")

        try:
            response = await continuation(handler_call_details)
            elapsed = (time.time() - start_time) * 1000
            logging.info(f"gRPC call completed: {method} in {elapsed:.2f}ms")
            return response
        except Exception as e:
            elapsed = (time.time() - start_time) * 1000
            logging.error(f"gRPC call failed: {method} after {elapsed:.2f}ms - {e}")
            raise

class AuthInterceptor(grpc.aio.ServerInterceptor):
    """
    JWT Authentication Interceptor
    """
    async def intercept_service(self, continuation, handler_call_details):
        metadata = dict(handler_call_details.invocation_metadata)
        token = metadata.get("authorization", "").replace("Bearer ", "")

        if not token or not self.validate_jwt(token):
            # Unauthenticated request ကို reject လုပ်သည်
            raise grpc.RpcError(
                grpc.StatusCode.UNAUTHENTICATED,
                "Valid JWT token required"
            )

        return await continuation(handler_call_details)

# Server တွင် interceptors register လုပ်သည်
server = grpc.aio.server(
    interceptors=[LoggingInterceptor(), AuthInterceptor()]
)
```

---

## ၅.၅ gRPC-Web

Browser native support မရှိသောကြောင့် **gRPC-Web** သည် browser ↔ gRPC server ကြားတွင် proxy layer ဖြင့် ဖြေရှင်းသည်။

```
Architecture:
┌──────────────┐  gRPC-Web    ┌──────────────┐  gRPC/HTTP2  ┌──────────────┐
│   Browser    │──(HTTP/1.1)─▶│  Envoy Proxy │─────────────▶│ gRPC Server  │
│  (JS client) │◀─────────────│  (Transcoder)│◀─────────────│              │
└──────────────┘              └──────────────┘              └──────────────┘
```

```javascript
// browser_client.js — gRPC-Web JavaScript client
import { OrderServiceClient } from './generated/order_service_grpc_web_pb';
import { CreateOrderRequest, OrderItem } from './generated/order_service_pb';

const client = new OrderServiceClient('http://localhost:8080');

const request = new CreateOrderRequest();
request.setCustomerId('cust_123');

const item = new OrderItem();
item.setProductId('prod_456');
item.setQuantity(2);
request.addItems(item);

// Unary call
client.createOrder(request, {}, (err, response) => {
    if (err) {
        console.error('Error:', err.message);
        return;
    }
    console.log('Order created:', response.getOrder().getOrderId());
});
```

---

## ၅.၆ gRPC vs REST Comparison

```
Detailed Comparison Table
────────────────────────────────────────────────────────────────────────
Feature              │ gRPC                    │ REST/HTTP
────────────────────────────────────────────────────────────────────────
Protocol             │ HTTP/2                  │ HTTP/1.1 or HTTP/2
Serialization        │ Protocol Buffers        │ JSON (usually)
Schema               │ Required (.proto)       │ Optional (OpenAPI)
Type Safety          │ Strong (compile-time)   │ Weak (runtime)
Code Generation      │ Built-in (protoc)       │ Optional (openapi-gen)
Streaming            │ Native (4 types)        │ Limited (SSE, WS separate)
Browser Support      │ Needs gRPC-Web proxy    │ Native
Human Readable       │ No (binary)             │ Yes (JSON)
Caching              │ Complex                 │ HTTP cache headers
Versioning           │ Proto evolution rules   │ URL/header versioning
Multi-language       │ Excellent (20+ langs)   │ Excellent
Error Handling       │ Status codes + details  │ HTTP status codes
Performance          │ High                    │ Moderate
Debugging            │ Need tooling (grpcurl)  │ curl/browser easy
Learning Curve       │ Moderate-High           │ Low
────────────────────────────────────────────────────────────────────────
```

**ရွေးချယ်မှု လမ်းညွှန်ချက်**

```
gRPC ကို ရွေးချယ်ပါ —
├── Internal microservice-to-microservice calls
├── Polyglot services (languages ကွဲများ)
├── Streaming data (IoT, real-time feeds)
├── High performance, low latency required
└── Strongly typed contracts လိုသောနေရာ

REST ကို ရွေးချယ်ပါ —
├── Public APIs (external developers)
├── Browser-based web apps (direct)
├── Simple CRUD operations
├── Partner/third-party integrations
└── Team REST experience ရှိပြီးသောနေရာ
```

**Proto backward compatibility rules**

```protobuf
// proto_evolution.proto

message OrderV1 {
    string order_id = 1;   // Field 1 — မဖျက်ပါနှင့်
    string customer_id = 2; // Field 2 — မဖျက်ပါနှင့်
    // Field 3 ကို ဖျက်ပြီးပြီဆိုလျှင်:
    reserved 3;            // Reserved ဖြင့် mark လုပ်ပါ
    reserved "old_field";  // Field name ကိုလည်း reserve လုပ်ပါ

    // New fields ကို အဆုံးတွင် ထည့်ပါ
    string new_field = 4;  // Backward compatible
}
```

---

## အဓိကသင်ခန်းစာများ (Key Takeaways)

1. **gRPC** သည် HTTP/2 + Protocol Buffers ဖြင့် REST ထက် 3-4 ဆ performance မြင့်ကာ internal microservice communication အတွက် ကောင်းမွန်သောရွေးချယ်မှုဖြစ်သည်။
2. **Protocol Buffers** သည် strongly-typed, binary serialization format ဖြစ်ပြီး compile-time type safety ပေးသည်; JSON ထက် 3-10 ဆ size လျှော့ချနိုင်သည်။
3. **RPC Types** — Unary (1→1), Server Streaming (1→N), Client Streaming (N→1), Bidirectional (N→N) တို့ကို use case ပေါ်မူတည်ပြီး ရွေးချယ်ပါ။
4. **Deadlines** — gRPC calls တိုင်းတွင် timeout/deadline သတ်မှတ်ပါ; cascading failure ကို ကာကွယ်နိုင်သည်။
5. **Interceptors** သည် logging, authentication, tracing ကဲ့သို့ cross-cutting concerns ကို middleware pattern ဖြင့် ဆောင်ရွက်နိုင်သော gRPC ၏ powerful mechanism ဖြစ်သည်။
6. **gRPC-Web** — Browser clients ကို Envoy proxy မှတဆင့် gRPC backend နှင့် ချိတ်ဆက်နိုင်သည်။
7. **Proto evolution** — Field numbers မဖျက်ပါနှင့်၊ new fields ကို အဆုံးတွင်သာ ထည့်ပါ၊ ဖျက်ထားသော fields ကို reserved ဖြင့် mark လုပ်ပါ — backward compatibility ကို ထိန်းသိမ်းနိုင်မည်ဖြစ်သည်။
